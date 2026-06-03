# Exceptions

Everything httpz raises descends from a single base, so you can catch broad or narrow depending on how much you care about the distinction.

```python
import httpz

try:
    with httpz.Client() as c:
        r = c.get("https://example.com", timeout=2)
        r.raise_for_status()
except httpz.TimeoutError:
    print("too slow")
except httpz.HTTPZError as e:
    print("something else went wrong:", e)
```

## The hierarchy

```
HTTPZError                  (base — catch this to catch everything)
├── TransportError          problem talking to the Go bridge itself
├── BridgeError             the Go library returned an error
├── TimeoutError            the request timed out
├── ProxyError             proxy config or connection failed
├── TooManyRedirects        redirect chain exceeded the limit
├── ConnectionError         couldn't establish the connection
└── HTTPStatusError         a 4xx/5xx, raised by raise_for_status()
```

All of them live on the top-level package, so it's `httpz.TimeoutError`, `httpz.HTTPStatusError`, and so on.

## `HTTPZError`

The base class for every error the library raises. Catch this when you just want "did httpz fail" and don't need to distinguish why. It's also what you'll get for a couple of plain misuse cases — passing `ja3` without `browser`, or making a request on a client you've already closed both raise `HTTPZError` directly.

## `TransportError`

Something went wrong communicating with the Go shared library — it returned a null pointer, or handed back a response that wasn't valid JSON. This also covers the "the bridge isn't there" case: if the shared library can't be found (you're on a checkout and haven't built it), the error you get is a `TransportError` whose message tells you how to build it.

In normal operation against a real server you shouldn't see this. If you do, it's usually an install/build problem rather than a network one.

## `BridgeError`

The Go side ran but returned an error string. This is the catch-all for engine-level failures that don't map onto one of the more specific types below. The message is passed through from Go, so it'll usually tell you what happened — a malformed fingerprint, for instance, surfaces here.

## `TimeoutError`

The request didn't finish inside the timeout. Note this is `httpz.TimeoutError`, not Python's built-in `TimeoutError` — if you're catching it, import it from httpz or reference it as `httpz.TimeoutError` so you catch the right one.

## `ProxyError`

The proxy is misconfigured or unreachable. If you set a `socks5://`/`http://`/`https://` proxy and it can't be used, this is what you get.

## `TooManyRedirects`

The redirect chain ran past `max_redirects` (10 by default, or whatever you set on the client or the request). Raised instead of following redirects forever.

## `ConnectionError`

The connection couldn't be established at all — no such host, connection refused, a dial failure. Again, this is `httpz.ConnectionError`, distinct from the built-in of the same name.

## `HTTPStatusError`

The one you raise on purpose. `Response.raise_for_status()` raises this for any status ≥ 400. Unlike the others, it carries the response:

```python
try:
    r.raise_for_status()
except httpz.HTTPStatusError as e:
    print(e.response.status_code)   # the Response is on .response
    print(e.response.text)
```

The constructor signature is `HTTPStatusError(message, *, response)` — the `response` is keyword-only — but you'll almost never construct it yourself; `raise_for_status()` does.

## How errors get classified

When the Go bridge returns an error, httpz looks at the message and routes it to the most specific type it can — "timeout"/"deadline" becomes `TimeoutError`, "proxy" becomes `ProxyError`, redirect-limit messages become `TooManyRedirects`, dial/host/refused messages become `ConnectionError`, and anything it can't pin down stays a `BridgeError`. It's string-matching on the underlying error, so it's good but not infallible; if you need to be certain, catch `HTTPZError` and inspect the message.
