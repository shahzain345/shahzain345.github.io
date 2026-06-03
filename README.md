# httpz documentation

httpz is an HTTP client for Python that lets you set a TLS fingerprint as easily as you set a header. If you've used `httpx` or `requests`, the request/response side will feel like home. The difference is what happens underneath: you can hand it a JA3 string (or an Akamai HTTP/2 fingerprint, or a browser preset name) and the actual bytes on the wire change to match.

These docs cover the whole public surface — every class, every method, and the handful of things that aren't obvious until they bite you.

## Where to start

If you've never touched the library, read [Getting started](getting-started.md). It's short, and it'll save you from the two or three mistakes everyone makes the first day.

After that, the rest is reference. Jump to whatever you're actually using:

- **[Client](client.md)** — the synchronous client. This is the one you'll reach for most. Sessions, cookies, proxies, per-request overrides, the lot.
- **[AsyncClient](async-client.md)** — the `asyncio` version. Same API, every method is awaitable. There's an honest note in there about its throughput ceiling, because it matters.
- **[Response](response.md)** — what you get back. `.text`, `.json()`, status helpers, `raise_for_status()`.
- **[Headers and Cookies](headers-and-cookies.md)** — the two container types. Both are dict-like but each has a quirk worth knowing (header ordering for one, user-set vs. server-set tracking for the other).
- **[Fingerprinting](fingerprinting.md)** — the actual point of the library. JA3, HTTP/2, applying a fingerprint mid-session, and a few words on why a JA3 hash you capture won't match byte-for-byte on the next request (that's expected — it's explained there).
- **[Impersonate presets](impersonate.md)** — the 170 built-in browser profiles, how name resolution works, the `BrowserTypeLiteral` type, and the browser default headers that ship with each preset.
- **[Exceptions](exceptions.md)** — the error hierarchy and how to catch the right thing.
- **[Module-level functions](module-functions.md)** — `httpz.get(...)` and friends, for when spinning up a `Client` is overkill.

## The mental model

It helps to know what's actually running. httpz is a thin Python layer over [azuretls-client](https://github.com/Noooste/azuretls-client), which is written in Go and compiled to a shared library. When you create a `Client`, you're creating a session inside that Go runtime — it owns the TLS spec, the HTTP/2 spec, the connection pool, and the cookie jar. Your Python `Client` object is a handle to it.

A few consequences fall out of that, and they explain most of the "why does it work like this" questions:

- **You should close your clients.** The Go session holds real sockets. Use a `with` block, or call `.close()`. There's a `__del__` safety net, but don't lean on it.
- **The bridge is a process-wide singleton.** One shared library, loaded once, shared across every client. It's mutex-guarded on the Go side, so it's thread-safe, but it's not something you instantiate per client.
- **Headers and cookies cross the boundary as ordered data.** That's deliberate — header *order* is part of a fingerprint, so the library goes out of its way to preserve it rather than hand the Go side a plain map.

If you keep "my Client is a handle to a Go session" in your head, the rest reads easily.

## A note on style

These docs try to tell you the truth, including the parts that are imperfect. Where something is a known quirk (the way `tls.peet.ws` renders one HTTP/2 setting, say, or where Safari 26's header order diverges from the template), it's written down rather than glossed over. If something here is wrong or stale, the code is the source of truth — but we try hard to keep these in step with it.
