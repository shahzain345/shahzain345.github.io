# Fingerprinting

This is the reason httpz exists. Everything else — the sessions, the cookie jar, the httpx-shaped API — you can get elsewhere. What you can't easily get in Python is the ability to hand a client a JA3 string and have the actual TLS handshake change to match. So this page covers the two fingerprints you can control, how to set them, and one piece of behaviour that confuses everyone the first time they see it.

## The two fingerprints

A server fingerprinting your client looks at two layers, and httpz lets you set both.

**JA3 — the TLS layer.** When a client opens a TLS connection it sends a ClientHello listing its supported cipher suites, extensions, elliptic curves, and point formats. JA3 is a compact string summarizing all of that. Two clients with the same JA3 present an indistinguishable TLS handshake. A Python script using the standard `ssl` module has a very Python-looking JA3; real Chrome has a Chrome-looking one. httpz lets you send Chrome's.

**Akamai / HTTP/2 — the protocol layer.** Once TLS is up and HTTP/2 starts, the client sends SETTINGS, a WINDOW_UPDATE, and orders its pseudo-headers (`:method`, `:authority`, `:scheme`, `:path`) in a particular way. The "Akamai fingerprint" captures that. Browsers each have a recognizable one.

Get both right and your client looks like a browser at both layers. Get only the TLS right and you've left an HTTP/2 tell. httpz handles both, and the [presets](impersonate.md) set them together.

## Setting fingerprints at construction

The raw way, when you have specific fingerprint strings to replay:

```python
import httpz

client = httpz.Client(
    ja3="771,4865-4866-4867-49195-49199-...,29-23-24,0",
    h2_fingerprint="1:65536;2:0;4:6291456;6:262144|15663105|0|m,s,a,p",
    browser="chrome",
)
```

`ja3` rewrites the ClientHello. `h2_fingerprint` controls the HTTP/2 frames. `browser` is mandatory whenever you set `ja3` — it tells the engine which navigator family to base everything else on (header casing, default behaviours, the bits JA3 doesn't capture). Leave it out and you get an `HTTPZError` at construction, not a vague failure later.

The `browser` value is one of the navigator constants:

```python
httpz.CHROME    # "chrome"
httpz.FIREFOX   # "firefox"
httpz.SAFARI    # "safari"
httpz.EDGE      # "edge"
httpz.OPERA     # "opera"
httpz.IOS       # "ios"
```

Most of the time you won't hand-write fingerprint strings at all — you'll use `impersonate="chrome131"` and let a [preset](impersonate.md) fill them in. The raw form is for when you've captured a fingerprint yourself and want it reproduced exactly.

## The HTTP/2 fingerprint format

The `h2_fingerprint` string is the Akamai format, four sections joined by `|`:

```
SETTINGS | WINDOW_UPDATE | PRIORITY | PSEUDO_HEADER_ORDER
```

- **SETTINGS** — `id:value` pairs joined by `;`, in send order. E.g. `1:65536;2:0;4:6291456;6:262144`. The ids are the HTTP/2 setting numbers (1 = HEADER_TABLE_SIZE, 2 = ENABLE_PUSH, 4 = INITIAL_WINDOW_SIZE, 6 = MAX_HEADER_LIST_SIZE, 8 = ENABLE_CONNECT_PROTOCOL, 9 = NO_RFC7540_PRIORITIES, and so on).
- **WINDOW_UPDATE** — the increment, e.g. `15663105`.
- **PRIORITY** — priority frames, or `0` if none.
- **PSEUDO_HEADER_ORDER** — the order of `:method`, `:authority`, `:scheme`, `:path`, abbreviated as `m,a,s,p` (in whatever order the browser uses).

You rarely build these by hand. They come from the presets, which were captured from real browsers.

## Changing fingerprints on a live client

You don't have to rebuild a client to change its fingerprint. Two methods mutate the running session in place — your connections and cookies survive the change.

```python
client.apply_ja3("771,4865-4867-...", browser="chrome")
client.apply_h2_fingerprint("1:65536;2:0;4:6291456;6:262144|15663105|0|m,a,s,p")
```

Rotating the JA3 between requests on one client is a real technique — it makes a sequence of requests look less like one fixed automated client. Same `browser` rule applies: it's required on `apply_ja3`.

On the async client these are awaitable: `await client.apply_ja3(...)`.

## Why your JA3 hash changes every request

Here's the thing that looks like a bug and isn't.

You impersonate Chrome, make a request, read back the JA3 hash the server saw. You make the same request again and the hash is *different*. You conclude impersonation is broken. It isn't.

Modern Chrome (and anything faithfully imitating it) **randomizes the order of its TLS extensions on every connection**. This is real Chrome behaviour, added years ago specifically to stop servers from relying on a fixed extension order. JA3 is computed over the extension list *in order*, so when the order shuffles, the JA3 hash changes. Two genuine Chrome requests produce two different JA3 hashes for exactly this reason. httpz reproduces the shuffle, so it does too.

What stays stable are the order-independent fingerprints:

- **JA4** sorts the extensions before hashing, so it's constant across requests from the same profile.
- **peetprint** (what `tls.peet.ws` reports) likewise.

So if you're verifying that impersonation worked, **compare JA4 or peetprint, not JA3.** A matching JA4 with a differing JA3 means the handshakes are equivalent and the JA3 difference is just the shuffle. If you genuinely need a frozen JA3, that's a different requirement — but for blending in, the shuffle is a feature, not a problem.

## A practical gotcha: extension 41 (pre_shared_key)

If you capture a JA3 from a live browser session and it ends with extension `41`, strip it before using it as a preset. TLS 1.3's `pre_shared_key` is only valid in a ClientHello when it's accompanied by real session-resumption material — without that, the server rejects the handshake outright. Browsers and the underlying engines drop it on the way out, so a correct on-the-wire JA3 never actually contains 41. The scraper that builds the presets doesn't strip extensions for you, so this is something to watch if you're adding a preset by hand.

## Verifying it works

The quickest check is to ask a fingerprinting endpoint what it saw:

```python
with httpz.Client(impersonate="chrome131") as c:
    j = c.get("https://tls.peet.ws/api/all").json()

print(j["tls"]["ja4"])                       # compare this run-to-run, not ja3
print(j["http2"]["akamai_fingerprint"])      # the HTTP/2 fingerprint on the wire
print(j["user_agent"])
```

For how the built-in profiles map to real browsers and how the names resolve, head to [Impersonate presets](impersonate.md).
