# Response

Every request hands you back a `Response`. It's a plain data object — built once from what the Go bridge returns, then read-only as far as you're concerned. The shape follows `httpx.Response`, so `.text`, `.json()`, `.status_code`, and `raise_for_status()` are all where you'd expect.

```python
r = client.get("https://httpbin.org/get")
print(r.status_code)      # 200
print(r.headers["content-type"])
data = r.json()
```

## Attributes

These are set when the response is constructed and just sit there for you to read.

### `status_code` → `int`

The HTTP status. `200`, `404`, `503`, whatever the server said.

### `url` → `str`

The final URL. If the request followed redirects, this is where it ended up, not where it started.

### `content` → `bytes`

The raw response body, as bytes. This is the source of truth — `.text` is derived from it. Empty body comes back as `b""`, not `None`.

### `headers` → `Headers`

The response headers, as a [`Headers`](headers-and-cookies.md) object. Case-insensitive lookup, so `r.headers["Content-Type"]` and `r.headers["content-type"]` are the same.

### `cookies` → `Cookies`

The cookies the server set on this response, as a [`Cookies`](headers-and-cookies.md) object. (The session jar handles re-sending them on later requests; this is here so you can see what came back.)

### `http_version` → `str`

The negotiated protocol — `"HTTP/2.0"`, `"HTTP/1.1"`, and so on. Useful when you want to confirm you actually got HTTP/2.

## Body accessors

### `text` → `str` *(property)*

The body decoded to a string, using the response's `encoding`.

```python
print(r.text)
```

### `json(**kwargs)` → `Any`

Parse the body as JSON. Any keyword arguments are passed straight through to `json.loads`, so things like `parse_float=` work.

```python
data = r.json()
```

This doesn't check the `Content-Type` first — if the body isn't JSON, you get a `json.JSONDecodeError`. That's deliberate; it's the same behaviour as the standard library.

### `encoding` → `str` *(property)*

The character encoding used to decode `.text`. It's pulled from the `charset=` parameter of the `Content-Type` header if there is one, and falls back to `utf-8` otherwise. This is a pragmatic default rather than full content-sniffing — if a server lies about its charset or omits it, `.text` assumes UTF-8.

## Status helpers

A handful of booleans so you don't have to memorize status-code ranges.

### `ok` → `bool`

`True` for anything below 400. The loosest "did this basically work" check.

### `is_success` → `bool`

`True` for 2xx (200–299).

### `is_redirect` → `bool`

`True` for the redirect codes specifically — 301, 302, 303, 307, 308. Note this checks the exact set, not just the 3xx range, so an oddball like 304 Not Modified is *not* a redirect by this measure.

### `is_client_error` → `bool`

`True` for 4xx (400–499).

### `is_server_error` → `bool`

`True` for 5xx (500–599).

## `raise_for_status()`

Raises [`HTTPStatusError`](exceptions.md) if the status is 400 or above; does nothing otherwise. The exception carries the response on `.response`, so you can dig into it in the handler.

```python
r = client.get("https://httpbin.org/status/404")
r.raise_for_status()   # raises HTTPStatusError: 404 for url ...
```

```python
try:
    r.raise_for_status()
except httpz.HTTPStatusError as e:
    print("failed:", e.response.status_code)
    print("body:", e.response.text)
```

It returns `None` on success, so the common pattern is to call it for its side effect right after the request and carry on if it doesn't raise.

## A note on what's missing

There's no streaming/iterator interface here — `content` is the whole body, read eagerly. For the request sizes httpz is typically used on (API calls, page fetches) that's fine and simpler. If you're pulling down something genuinely large, be aware the whole thing lands in memory.
