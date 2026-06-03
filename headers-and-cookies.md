# Headers and Cookies

These are the two container types httpz uses for request/response metadata. Both behave like dictionaries, so most of the time you won't think about them. But each has one design decision behind it that's worth understanding, because it explains behaviour you'd otherwise find surprising.

---

# Headers

`httpz.Headers` is a case-insensitive, order-preserving, multi-value mapping. It's a `MutableMapping`, so it quacks like a dict — `h["Accept"]`, `h.get("accept")`, `in`, `len()`, iteration, all there.

```python
from httpz import Headers

h = Headers({"Content-Type": "application/json", "X-Request-Id": "abc"})
h["Authorization"] = "Bearer token"
print(h["content-type"])   # case-insensitive: "application/json"
```

## Why it isn't just a dict

Two reasons, and they're the whole point of the class:

**Order is preserved.** Header order is part of a fingerprint. Real browsers send headers in a consistent, recognizable sequence, and anti-bot systems look at that order. A plain Python dict would preserve insertion order too, but `Headers` guarantees the order survives the trip across the bridge to the Go side via `to_ordered_pairs()`. This is why httpz goes to the trouble of having its own type instead of passing a dict around.

**Case is preserved on the wire but ignored on lookup.** You can store `"X-My-Header"` and look it up as `"x-my-header"`. The stored casing is what gets sent; the lookup just doesn't care.

## Constructing one

You can build a `Headers` from a few things:

```python
Headers()                                          # empty
Headers({"Accept": "*/*"})                         # from a dict
Headers([("Accept", "*/*"), ("Accept", "text/*")]) # from pairs — keeps both values
Headers(other_headers)                             # copy of another Headers
```

That third form is the multi-value case: a list of pairs can carry the same key twice, and both are kept.

## Methods

### `h[key]` / `h[key] = value` / `del h[key]`

Standard mapping access, all case-insensitive. Setting a key removes any existing entries with that name (case-insensitively) and appends the new one — so a `__setitem__` moves the key to the end if it was already present.

### `get(key, default=None)`

The forgiving lookup. Returns `default` instead of raising `KeyError` when the header isn't there.

### `get_list(key)` → `list[str]`

All values for a header, for the genuinely multi-valued ones. Where `h["set-cookie"]` gives you one value, `h.get_list("set-cookie")` gives you every one.

```python
h = Headers([("Set-Cookie", "a=1"), ("Set-Cookie", "b=2")])
h.get_list("set-cookie")   # ["a=1", "b=2"]
```

### `multi_items()` → `list[tuple[str, str]]`

Every `(key, value)` pair, duplicates included, in order. This is the "give me everything, flat" accessor — iterating the mapping normally collapses duplicate keys, this doesn't.

### `to_ordered_pairs()` → `list[list[str]]`

The serialization the Go bridge wants: a list of `[key, value]` pairs in order. You generally won't call this yourself — the client does, when it builds a request — but it's public if you need to see exactly what's going over the wire.

### `copy()` → `Headers`

A shallow copy with its own internal list, so mutating the copy doesn't touch the original.

## Iteration

Iterating a `Headers` yields each header name once, even if it appears multiple times, with the first casing seen. `len()` likewise counts unique (case-folded) names, not total entries. If you want the raw, possibly-duplicated truth, that's what `multi_items()` is for.

---

# Cookies

`httpz.Cookies` is also a `MutableMapping` of name → value. The twist here isn't about ordering — it's that it quietly tracks *who set each cookie*, and that changes how cookies get sent.

```python
from httpz import Cookies

c = Cookies({"session": "abc123"})
c["theme"] = "dark"
print(c["session"])
```

## The user-set vs. server-set distinction

This is the one thing to understand about cookies in httpz.

When you set a cookie yourself — through the constructor or `client.cookies["x"] = "y"` — it's marked as **user-set**. User-set cookies are serialized into a `Cookie:` header on each request, because *you* are responsible for sending them.

When a *server* sets a cookie (via `Set-Cookie` on a response), it lands in the same container so you can see it, but it's **not** marked user-set. The Go session has its own cookie jar that already handles re-sending server cookies automatically. If httpz also stuffed them into a `Cookie:` header, they'd be sent twice.

So the split exists to avoid duplicating server cookies while still letting you inject your own. In day-to-day use you don't think about it — set what you want, read what came back, and the right thing happens. It only matters if you're inspecting exactly which cookies leave on a given request.

## Methods

### `c[name]` / `c[name] = value` / `del c[name]`

Standard mapping access. Setting a cookie marks it user-set; deleting one also drops it from the user-set tracking.

### `update_from_response(cookies_dict)`

How the client folds a response's cookies back into the jar. These come in *without* the user-set mark, exactly as described above. You won't normally call this — the client does it for you after each request — but it's there.

### `user_set_items()` → `dict[str, str]`

Just the cookies you set explicitly, as a plain dict. This is precisely the set that gets turned into the `Cookie:` header on outgoing requests. Handy for debugging "wait, why is this cookie being sent."

```python
client.cookies["auth"] = "token123"
# later, on the next request, the Cookie header will include auth=token123
print(client.cookies.user_set_items())   # {"auth": "token123"}
```

## Construction

```python
Cookies()                       # empty
Cookies({"session": "abc"})     # from a dict — all marked user-set
Cookies(other_cookies)          # copy, preserving the user-set marks
```
