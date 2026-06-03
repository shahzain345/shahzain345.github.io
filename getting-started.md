# Getting started

## Install

```bash
pip install pyhttpz
```

The distribution is `pyhttpz` on PyPI, but the import is just `httpz`:

```python
import httpz
```

The wheel ships with the compiled Go bridge for your platform (a `.dll` on Windows, `.so` on Linux, `.dylib` on macOS) and the preset data baked straight into the Python source, so there's nothing else to fetch and no data files to bundle if you later freeze your app with PyInstaller or similar.

If you're working from a checkout instead of a wheel, you'll need to build the bridge first — `cd go && build.bat` on Windows, `make` on Linux/macOS. You'll know it isn't built because the first request raises a `TransportError` telling you exactly that.

## Your first request

```python
import httpz

r = httpz.get("https://httpbin.org/get")
print(r.status_code)
print(r.json())
```

That's the one-shot form. It spins up a client, makes the request, and tears the client down again. Fine for a script or a quick poke at an API. If you're making more than one request, don't do this in a loop — make a client and reuse it.

## Use a session

```python
import httpz

with httpz.Client() as client:
    client.get("https://httpbin.org/cookies/set?token=abc")
    r = client.get("https://httpbin.org/cookies")
    print(r.json())   # the cookie set on the first call is sent on the second
```

A `Client` keeps connections alive between requests and carries a cookie jar, the same way `requests.Session` or `httpx.Client` does. The `with` block matters: it closes the underlying Go session and releases the sockets when you're done. You *can* call `client.close()` by hand instead, but the context manager is harder to get wrong.

## Now the actual reason you're here

```python
with httpz.Client(impersonate="chrome131") as client:
    r = client.get("https://tls.peet.ws/api/all")
    print(r.json()["tls"]["ja3_hash"])
```

`impersonate="chrome131"` is shorthand. In one argument it sets the JA3 fingerprint, the HTTP/2 (Akamai) fingerprint, the User-Agent, and the browser navigator — and it also sends the default request headers that Chrome 131 actually sends (client hints, `Accept`, `Sec-Fetch-*`, and so on, in the order Chrome sends them). So the request looks like Chrome at the TLS layer *and* the HTTP layer, not just one of them.

There are 170 presets to choose from. List them at runtime:

```python
print(httpz.list_impersonate_targets())
```

Or, better, let your editor list them for you — the `impersonate` argument is typed against a `Literal` of every valid name, so you get autocomplete and a red squiggle if you typo one. See [Impersonate presets](impersonate.md).

## Setting a raw JA3

If you've captured a fingerprint from somewhere and want to replay it exactly:

```python
with httpz.Client(
    ja3="771,4865-4866-4867-49195-49199-...,29-23-24,0",
    browser="chrome",
) as client:
    client.get("https://example.com")
```

The one rule: **if you pass `ja3`, you must also pass `browser`.** The browser tells the engine which navigator profile to build the rest of the handshake around. Forget it and you get a clear `HTTPZError` saying so, not a mysterious failure.

## Three things that trip people up

1. **Reuse the client.** Creating one per request throws away your connection pool and cookie jar, and it's slower. Make one, keep it, close it at the end.

2. **A JA3 hash you read back won't be identical every time.** Chrome shuffles its TLS extension order on every connection, and httpz faithfully reproduces that. So `ja3_hash` changes run to run. The order-independent fingerprints (JA4, peetprint) stay stable — those are the ones to compare if you're checking that impersonation "worked." More on this in [Fingerprinting](fingerprinting.md).

3. **`browser=` (and `impersonate=`) now sends real browser headers for you.** If you were relying on httpz sending a bare request, that changed — emulating a browser means looking like one all the way up. You can turn it off with `browser_headers=False`. See [Client](client.md).

That's enough to be dangerous. The rest of the docs are reference.
