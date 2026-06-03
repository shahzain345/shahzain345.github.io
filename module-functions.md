# Module-level functions

For one-off requests, spinning up a `Client` and remembering to close it is more ceremony than the job needs. So the package exposes the HTTP verbs directly:

```python
import httpz

r = httpz.get("https://httpbin.org/get")
print(r.json())
```

## The verb functions

```python
httpz.get(url, **kwargs)
httpz.post(url, **kwargs)
httpz.put(url, **kwargs)
httpz.patch(url, **kwargs)
httpz.delete(url, **kwargs)
httpz.head(url, **kwargs)
httpz.options(url, **kwargs)
```

Each one creates a `Client`, runs the request inside a `with` block, and returns the [`Response`](response.md). The client is closed before the function returns, so there's nothing for you to clean up.

The `**kwargs` are forwarded to the corresponding `Client` request method, so everything `request()` accepts works here too:

```python
httpz.post(
    "https://api.example.com/items",
    json={"name": "widget"},
    headers={"Authorization": "Bearer token"},
    timeout=5,
)
```

### When *not* to use these

There's a real cost hiding in the convenience: each call builds and tears down a whole session. That means a fresh connection pool every time (no keep-alive reuse) and **no shared cookies between calls**. So this:

```python
httpz.get("https://site/login")    # sets a cookie
httpz.get("https://site/account")  # ...which is gone — different session
```

won't carry the login cookie to the second request. The moment you're making more than one related request, switch to a `Client`:

```python
with httpz.Client() as client:
    client.get("https://site/login")
    client.get("https://site/account")   # cookie carried over
```

Rule of thumb: module functions for a single isolated request, `Client` for anything with state or more than one call.

A note on fingerprinting: the module functions don't take fingerprinting arguments because they construct a bare default client. If you want `impersonate=` or a custom JA3 for a one-off, just use a short `with httpz.Client(impersonate=...)` block — it's barely longer and gives you the control.

## Preset functions

These two are also on the package, covered in full in [Impersonate presets](impersonate.md) but listed here for completeness:

### `httpz.list_impersonate_targets()` → `list[str]`

Every valid `impersonate=` name, sorted.

### `httpz.resolve_impersonate(name)` → `dict`

The full preset record for a name (after alias resolution). Raises `HTTPZError` for an unknown name.

```python
print(len(httpz.list_impersonate_targets()))   # 219
print(httpz.resolve_impersonate("chrome131")["ja3_hash"])
```

## Constants and types

For reference, the package also exports:

- The navigator constants — `httpz.CHROME`, `FIREFOX`, `SAFARI`, `EDGE`, `OPERA`, `IOS` — for the `browser=` argument.
- `httpz.BrowserTypeLiteral` — the `Literal` type of every valid `impersonate=` name, for type-checking and autocomplete. See [Impersonate presets](impersonate.md#the-browsertypeliteral-type).
