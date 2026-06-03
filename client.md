# Client

`httpz.Client` is the synchronous client and the centre of the library. It owns a session — a live connection pool, a cookie jar, and a TLS/HTTP2 configuration — inside the Go runtime. Everything you do flows through one of these.

It's modeled on `httpx.Client` closely enough that most code written against httpx ports over by changing the import. The differences are all on the constructor, where the fingerprinting arguments live.

```python
import httpz

with httpz.Client(impersonate="chrome131") as client:
    r = client.get("https://example.com")
```

## Constructor

```python
Client(
    *,
    headers=None,
    cookies=None,
    proxy=None,
    ja3=None,
    h2_fingerprint=None,
    browser=None,
    user_agent=None,
    impersonate=None,
    timeout=None,
    max_redirects=10,
    verify=True,
    browser_headers=True,
)
```

Every argument is keyword-only — there are no positional arguments, on purpose, because a positional list this long is impossible to read at the call site.

| Argument | Type | What it does |
|---|---|---|
| `headers` | dict / `Headers` / list of pairs | Default headers merged into every request. A per-request `headers=` wins key by key. |
| `cookies` | dict / `Cookies` | Cookies to start the jar with. Anything the server sets later is added automatically. |
| `proxy` | str | Upstream proxy URL. `http`, `https`, and `socks5` schemes are supported. |
| `ja3` | str | A JA3 client fingerprint. Rewrites the TLS ClientHello to match. **Requires `browser`.** |
| `h2_fingerprint` | str | An Akamai-format HTTP/2 fingerprint — controls SETTINGS, WINDOW_UPDATE, the pseudo-header order, and priority. |
| `browser` | str | The navigator family backing the handshake. Use the constants: `httpz.CHROME`, `FIREFOX`, `SAFARI`, `EDGE`, `OPERA`, `IOS`. |
| `user_agent` | str | Overrides the `User-Agent`. |
| `impersonate` | str (a `BrowserTypeLiteral`) | A preset name that expands to `ja3` + `h2_fingerprint` + `browser` + `user_agent` at once. |
| `timeout` | float | Default per-request timeout, in seconds. A per-request `timeout=` overrides it. |
| `max_redirects` | int | How long a redirect chain is allowed to get before `TooManyRedirects`. Default 10. |
| `verify` | bool | Set `False` to skip TLS certificate verification. Handy for self-signed certs in dev; never ship it `True`-less to production. |
| `browser_headers` | bool | When `True` (the default) and a browser navigator is active, send that browser's default request headers too. See below. |

### How the arguments interact

`impersonate` is just a convenience. Internally it resolves the preset and fills in `ja3`, `h2_fingerprint`, `browser`, and `user_agent` — but only for the ones you didn't pass yourself. Explicit arguments always win. So this pins the Chrome 131 fingerprint but swaps in your own User-Agent:

```python
httpz.Client(impersonate="chrome131", user_agent="MyBot/1.0")
```

The `ja3` + `browser` pairing is the one hard rule. A JA3 string describes cipher suites and extensions, but the engine still needs to know which browser family to model the rest of the behaviour on. Pass `ja3` without `browser` and you get an `HTTPZError` immediately, at construction time, rather than a confusing handshake failure later.

### Browser default headers (`browser_headers`)

This is the part that surprises people coming from older versions. A real browser request isn't just a TLS handshake — Chrome sends `sec-ch-ua`, `Accept`, `Sec-Fetch-*`, `Accept-Language`, `priority`, and so on, in a particular order. A request with a flawless JA3 but none of those headers is still trivially flagged as a bot.

So whenever a browser navigator is active — through `impersonate=` or a bare `browser=` — httpz fills in that browser's real default headers, and emits them in the browser's natural order (Chrome and friends put `User-Agent` fifth, after `upgrade-insecure-requests`; Firefox and Safari lead with it). The values track the preset: the `sec-ch-ua` version and `Accept-Encoding` follow the Chrome version, the platform hint follows the User-Agent.

Your own headers still win. Anything you pass in `headers=` overrides a default value while keeping the browser's position for that header; headers the browser doesn't send are appended at the end.

```python
# Keep all of Chrome's headers, but change Accept-Language and add one of your own
httpz.Client(impersonate="chrome131", headers={"accept-language": "fr-FR,fr;q=0.9"})
```

If you'd rather send only what you specify and nothing else, turn it off:

```python
httpz.Client(impersonate="chrome131", browser_headers=False)
```

There's a fuller treatment in [Impersonate presets](impersonate.md).

## Attributes

### `client.headers` → `Headers`

The session's default headers, as a mutable [`Headers`](headers-and-cookies.md) object. Read it, or change it after construction and the change applies to every request from then on:

```python
client.headers["X-Api-Key"] = "secret"
del client.headers["x-api-key"]   # lookup is case-insensitive
```

You can also assign a plain dict to it (`client.headers = {...}`) — it gets coerced to a `Headers` so ordering still works.

### `client.cookies` → `Cookies`

The session cookie jar, as a [`Cookies`](headers-and-cookies.md) object. Inspect what the server set, or inject your own:

```python
client.cookies["session"] = "abc123"
print(client.cookies["csrftoken"])
```

Cookies you set this way are re-sent on later requests via a `Cookie` header; cookies the server sets are remembered by the Go session's own jar and sent automatically. (The distinction matters in one edge case — see [Cookies](headers-and-cookies.md).)

## Making requests

### `request(method, url, *, ...)`

The general method every verb helper delegates to.

```python
request(
    method,
    url,
    *,
    headers=None,
    content=None,
    data=None,
    json=None,
    params=None,
    timeout=None,
    follow_redirects=True,
    max_redirects=None,
) -> Response
```

| Argument | What it does |
|---|---|
| `method` | HTTP verb. Cased up for you, so `"get"` and `"GET"` are the same. |
| `url` | The target URL. |
| `headers` | Per-request headers, merged over the session defaults key by key. |
| `content` | Raw `bytes` for the body. Sent as-is. |
| `data` | A form body. A dict is URL-encoded (`application/x-www-form-urlencoded`); a str is sent verbatim; bytes are sent raw. |
| `json` | Any JSON-serializable object. Serialized and sent with `Content-Type: application/json`. |
| `params` | Query parameters, merged into the URL's existing query string. |
| `timeout` | Overrides the session timeout for this one request. |
| `follow_redirects` | `True` by default. Set `False` to get the 3xx back instead of chasing it. |
| `max_redirects` | Overrides the session's redirect cap for this request. |

A word on the body arguments: `content`, `data`, and `json` are mutually exclusive in practice — pass one. They're checked in the order `json`, then `data`, then `content`, so if you somehow pass more than one, that's the precedence.

### The verb helpers

```python
client.get(url, **kwargs)
client.post(url, **kwargs)
client.put(url, **kwargs)
client.patch(url, **kwargs)
client.delete(url, **kwargs)
client.head(url, **kwargs)
client.options(url, **kwargs)
```

Each is a one-line wrapper over `request()` with the method filled in. Every keyword `request()` accepts works on all of them, so `client.post(url, json={...}, timeout=5)` does what you'd expect.

```python
r = client.post("https://api.example.com/items", json={"name": "widget"})
r.raise_for_status()
```

All of them return a [`Response`](response.md).

## Reconfiguring a live session

You don't have to throw away a client to change its fingerprint or proxy. These methods mutate the existing Go session in place, so your connection pool and cookies survive.

### `apply_ja3(ja3, browser)`

Swap the TLS fingerprint mid-session.

```python
client.apply_ja3("771,4865-4867-...", browser="chrome")
```

Rotating JA3 between requests on the same client is a legitimate trick for blending in. `browser` is required here for the same reason it is on the constructor.

### `apply_h2_fingerprint(fingerprint)`

Swap the HTTP/2 (Akamai) fingerprint.

```python
client.apply_h2_fingerprint("1:65536;2:0;4:6291456;6:262144|15663105|0|m,a,s,p")
```

### `set_proxy(proxy_url)` and `clear_proxy()`

Point the session at a proxy, or stop using one, without rebuilding it.

```python
client.set_proxy("socks5://user:pass@127.0.0.1:1080")
# ... requests now go through the proxy ...
client.clear_proxy()
```

## Lifecycle

### `close()`

Tears down the Go session and releases its sockets. After this, any request raises `HTTPZError("Client is closed")`. Calling `close()` twice is harmless.

### Context manager

`Client` is a context manager, and this is the recommended way to use it:

```python
with httpz.Client(impersonate="chrome131") as client:
    ...
# closed automatically here, even if the block raised
```

### `__del__`

There's a finalizer that closes the session if you forgot to. It's a safety net, not a strategy — garbage-collection timing isn't something to depend on, especially for releasing sockets. Use `with` or call `close()`.

## A full example

Everything together — preset, proxy, a starting cookie, per-request timeout, and a mid-session fingerprint rotation:

```python
with httpz.Client(
    impersonate="chrome131",
    proxy="socks5://user:pass@127.0.0.1:1080",
    timeout=30.0,
    headers={"Accept-Language": "en-US,en;q=0.9"},
) as client:
    client.cookies["session"] = "abc123"

    r = client.get("https://api.example.com/me")
    r.raise_for_status()

    client.apply_ja3("771,4865-4867-...", browser="chrome")
    r2 = client.post("https://api.example.com/items", json={"name": "x"})
```

For the async version of all this, see [AsyncClient](async-client.md).
