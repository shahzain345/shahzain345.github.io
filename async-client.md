# AsyncClient

`httpz.AsyncClient` is the `asyncio` counterpart to [`Client`](client.md). Same API, same arguments, same behaviour — except every method that does I/O is an `async def` you `await`.

```python
import asyncio
import httpz

async def main():
    async with httpz.AsyncClient(impersonate="chrome131") as client:
        r = await client.get("https://example.com")
        print(r.status_code)

asyncio.run(main())
```

## How it actually works (and why you should care)

`AsyncClient` is a thin wrapper. Under the hood it builds a real synchronous `Client` and dispatches each request into a worker thread with `asyncio.to_thread`. The blocking call into the Go bridge happens on that thread, so it never stalls your event loop.

That design has one consequence worth being upfront about: **your async throughput is bounded by the default thread pool**, which is roughly `min(32, os.cpu_count() + 4)` threads. For the overwhelming majority of workloads — a few dozen requests in flight, scraping with fingerprint control, talking to an API concurrently — this is completely fine and you'll never notice the ceiling.

But if you're firing *hundreds* of concurrent requests and you don't need TLS-fingerprint control, a natively-async client like `aiohttp` or `httpx`'s async client will out-throughput this, because their concurrency lives in the event loop rather than a thread pool. Pick the right tool. httpz's value is the fingerprinting; if you don't need it, you have lighter options.

## Constructor

```python
AsyncClient(**kwargs)
```

Every keyword argument is forwarded verbatim to `Client`. There's no separate list to learn — `headers`, `cookies`, `proxy`, `ja3`, `h2_fingerprint`, `browser`, `user_agent`, `impersonate`, `timeout`, `max_redirects`, `verify`, and `browser_headers` all mean exactly what they mean on [`Client`](client.md).

## Attributes

### `client.headers` → `Headers`
### `client.cookies` → `Cookies`

Both are shared with the underlying sync client — they're the same objects, not copies. Mutating `client.headers` or `client.cookies` on the async client changes the session for subsequent requests, same as the sync side. You can also assign a plain dict to either; it gets coerced.

## Requests

All awaitable, all returning a [`Response`](response.md):

```python
await client.request(method, url, **kwargs)
await client.get(url, **kwargs)
await client.post(url, **kwargs)
await client.put(url, **kwargs)
await client.patch(url, **kwargs)
await client.delete(url, **kwargs)
await client.head(url, **kwargs)
await client.options(url, **kwargs)
```

The keyword arguments are identical to the sync client's — `headers`, `content`, `data`, `json`, `params`, `timeout`, `follow_redirects`, `max_redirects`. See [Client](client.md#making-requests) for the details rather than repeating them here.

## Reconfiguring a live session

The same in-place reconfiguration methods, awaitable:

```python
await client.apply_ja3(ja3, browser)
await client.apply_h2_fingerprint(fingerprint)
await client.set_proxy(proxy_url)
await client.clear_proxy()
```

## Lifecycle

### `await client.close()`

Closes the underlying session and releases its sockets.

### Async context manager

The clean way to use it:

```python
async with httpz.AsyncClient() as client:
    ...
# closed on exit
```

`__aenter__` returns the client; `__aexit__` awaits `close()`.

## Fanning out concurrent requests

The thing you'll actually use it for — a batch of requests with one shared fingerprint and cookie jar:

```python
import asyncio
import httpz

async def main():
    async with httpz.AsyncClient(impersonate="chrome131") as c:
        urls = [f"https://httpbin.org/anything/{i}" for i in range(20)]
        responses = await asyncio.gather(*(c.get(u) for u in urls))
        for r in responses:
            print(r.status_code, r.url)

asyncio.run(main())
```

`asyncio.gather` kicks them all off; the thread pool runs as many in parallel as it has threads, and the rest queue. The shared client means all twenty requests reuse the same connection pool and see the same cookies.
