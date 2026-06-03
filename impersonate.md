# Impersonate presets

Writing JA3 and HTTP/2 fingerprint strings by hand is miserable. So httpz ships 170 of them, captured from real browsers, behind friendly names. Pass `impersonate="chrome131"` and you get that browser's TLS fingerprint, HTTP/2 fingerprint, User-Agent, navigator, *and* its default request headers — the whole package.

```python
with httpz.Client(impersonate="chrome131") as c:
    c.get("https://tls.peet.ws/api/all")
```

## What a preset actually is

Each preset is a small record: a JA3 string, an Akamai HTTP/2 fingerprint, the matching User-Agent, and the browser family. When you pass `impersonate=`, the client looks that record up and uses it to fill in `ja3`, `h2_fingerprint`, `user_agent`, and `browser` — but only the ones you didn't set yourself. Your explicit arguments always win, so you can pin a preset and override one field:

```python
httpz.Client(impersonate="chrome131", user_agent="MyBot/1.0")
```

The preset data lives directly inside `httpz/presets.py` as a Python dict — not a separate JSON file. That's deliberate: a `.py` module gets bundled automatically when you freeze your app (PyInstaller, py2exe, zipapp), so there's no data file to remember to include.

## Listing and resolving presets

Two functions on the package let you work with presets programmatically.

### `httpz.list_impersonate_targets()` → `list[str]`

Every name you can pass to `impersonate=`, sorted. That's all 170 canonical profiles plus their aliases — 219 names in total.

```python
import httpz
print(httpz.list_impersonate_targets())
```

### `httpz.resolve_impersonate(name)` → `dict`

The preset record for a name, after alias resolution. Raises `HTTPZError` if the name isn't known.

```python
p = httpz.resolve_impersonate("chrome131")
print(p["ja3_hash"], p["browser"], p["user_agent"])
```

## The `BrowserTypeLiteral` type

You don't have to memorize preset names or keep `list_impersonate_targets()` open in another window. The `impersonate` argument is typed against `httpz.BrowserTypeLiteral`, a `typing.Literal` of every valid name. Two things fall out of that:

- Your editor autocompletes the names as you type.
- A type checker (mypy, pyright, your IDE's built-in) flags a typo'd or non-existent name before you ever run the code.

```python
import httpz

httpz.Client(impersonate="chrome131")   # fine
httpz.Client(impersonate="chrome1311")  # type checker: not a valid BrowserTypeLiteral
```

It's the same idea as `curl_cffi`'s `BrowserType` — a closed set of known-good strings rather than a bare `str`. You can import it to annotate your own code too:

```python
from httpz import BrowserTypeLiteral

def fetch(profile: BrowserTypeLiteral) -> None:
    ...
```

The literal is generated from the preset data by the scraper, so it can never drift out of sync with the actual presets — regenerate one and the other regenerates with it.

## How name resolution works

Browser presets came from several sources, and the same browser version sometimes shows up under slightly different spellings. Resolution smooths that over.

Names are normalized before lookup — underscores and dots are stripped and everything is lowercased — so all of these land on the same profile:

```
chrome131    chrome_131    Chrome131    CHROME131
```

When a browser version genuinely has more than one distinct handshake on the wire (a different TLS extension order, say), each variant keeps its own name rather than being collapsed. You'll see sibling names like `chrome131` and `chrome_131`, or a lettered variant like `chrome133a`. Those aren't duplicates — they're real, distinct profiles, and which one you want depends on exactly what you're trying to match. Most of the time any of them is fine.

## The 170 profiles

Grouped by browser. Any of these works as an `impersonate=` value.

### Chrome (78)
`chrome99`, `chrome99_android`, `chrome100`, `chrome101`, `chrome104`, `chrome106a`, `chrome107`, `chrome107a`, `chrome108a`, `chrome109a`, `chrome110`, `chrome110a`, `chrome114a`, `chrome116`, `chrome116a`, `chrome117a`, `chrome118a`, `chrome119`, `chrome119a`, `chrome120`, `chrome120a`, `chrome123`, `chrome123a`, `chrome124`, `chrome124a`, `chrome126a`, `chrome127a`, `chrome128a`, `chrome129a`, `chrome130a`, `chrome131`, `chrome131_android`, `chrome131a`, `chrome132`, `chrome133a`, `chrome133b`, `chrome134`, `chrome135`, `chrome136`, `chrome136a`, `chrome137`, `chrome138`, `chrome139`, `chrome140`, `chrome141`, `chrome142`, `chrome142a`, `chrome143`, `chrome144`, `chrome145`, `chrome145a`, `chrome146`, `chrome146a`, `chrome147`, `chrome148`, `chrome_100`, `chrome_101`, `chrome_104`, `chrome_105`, `chrome_106`, `chrome_107`, `chrome_108`, `chrome_109`, `chrome_114`, `chrome_116`, `chrome_117`, `chrome_118`, `chrome_119`, `chrome_120`, `chrome_123`, `chrome_124`, `chrome_126`, `chrome_127`, `chrome_128`, `chrome_129`, `chrome_130`, `chrome_131`, `chrome_133`

### Firefox (20)
`firefox133`, `firefox135`, `firefox136`, `firefox139`, `firefox142`, `firefox143`, `firefox144`, `firefox145`, `firefox146`, `firefox147`, `firefox148`, `firefox149`, `firefox151`, `firefox_109`, `firefox_117`, `firefox_128`, `firefoxandroid135`, `firefoxprivate135`, `firefoxprivate136`, `tor145`

### Safari (34)
`safari26`, `safari153`, `safari155`, `safari170`, `safari172_ios`, `safari180`, `safari180_ios`, `safari183`, `safari184`, `safari184_ios`, `safari185`, `safari260`, `safari260_ios`, `safari261`, `safari262`, `safari1831`, `safari2601`, `safari_15.3`, `safari_15.5`, `safari_15.6.1`, `safari_16`, `safari_16.5`, `safari_17.2.1`, `safari_17.4.1`, `safari_17.5`, `safari_18.2`, `safari_ios_16.5`, `safari_ios_17.4.1`, `safari_ios_18.1.1`, `safari_ipad_18`, `safariios26`, `safariios262`, `safariipad26`, `safariipad262`

### Edge (23)
`edge99`, `edge101`, `edge122a`, `edge127a`, `edge131a`, `edge134`, `edge135`, `edge136`, `edge137`, `edge138`, `edge139`, `edge140`, `edge141`, `edge142`, `edge143`, `edge144`, `edge145`, `edge146`, `edge147`, `edge_101`, `edge_122`, `edge_127`, `edge_131`

### Opera (15)
`opera116`, `opera117`, `opera118`, `opera119`, `opera120`, `opera121`, `opera122`, `opera123`, `opera124`, `opera125`, `opera126`, `opera127`, `opera128`, `opera129`, `opera130`

## Browser default headers

A preset doesn't just set the TLS and HTTP/2 fingerprints — it also makes the request *carry the headers that browser actually sends*. This matters more than it sounds. A request with Chrome's exact JA3 but missing `sec-ch-ua`, `Accept`, and the `Sec-Fetch-*` headers is still easy to spot as automated, because no real Chrome ever sends a request that bare.

So whenever a browser navigator is active — via `impersonate=` or a plain `browser=` — httpz adds that browser's default headers, in the browser's natural order, with values derived from the preset:

- **Chrome / Edge / Opera** send the client hints (`sec-ch-ua`, `sec-ch-ua-mobile`, `sec-ch-ua-platform`), `upgrade-insecure-requests`, `User-Agent`, `Accept`, the four `Sec-Fetch-*` headers, `Accept-Encoding`, `Accept-Language`, and `priority`. The `sec-ch-ua` version and whether `Accept-Encoding` includes `zstd` track the Chrome version; the platform and mobile hints track the User-Agent. `User-Agent` sits fifth, right after `upgrade-insecure-requests`, exactly where Chrome puts it.
- **Firefox** sends no client hints (it doesn't implement them) and leads with `User-Agent`, followed by `Accept`, `Accept-Language`, `Accept-Encoding`, `upgrade-insecure-requests`, the `Sec-Fetch-*` set, `priority`, and `te: trailers`.
- **Safari / iOS** send a leaner set led by `User-Agent`, then `Accept`, `Accept-Language`, `Accept-Encoding`.

### Overriding and opting out

Your headers win. Anything in `headers=` overrides the browser default for that key while keeping the browser's position for it; headers the browser doesn't normally send get appended.

```python
# Replace one header, add another — the rest of Chrome's headers stay intact and in order
httpz.Client(impersonate="chrome131", headers={"accept-language": "fr-FR,fr;q=0.9"})
```

If you want httpz to send only the headers you specify and nothing else, switch it off:

```python
httpz.Client(impersonate="chrome131", browser_headers=False)
```

### One honest caveat

The header templates match the common case for each family very closely — Chrome, Edge, Opera, and Firefox line up with what `curl_cffi` sends, header-for-header and in order. Safari uses the classic Safari layout (lead with `User-Agent`), which is right for Safari 15 through 18. The very newest Safari (26) sends a richer, oddly-interleaved set with extra `Sec-Fetch-*` and `priority` headers that the lean template doesn't fully reproduce. If you're specifically targeting Safari 26 fingerprinting at the header level, that's the one spot where there's still a gap.

## Refreshing the presets

The presets are generated by `scripts/scrape_fingerprints.py`, which drives each source library (curl_cffi, primp, wreq) through real requests to a fingerprinting endpoint and records what it sees.

```bash
# Full rebuild from every source (this drops any manual entries)
pip install -U curl_cffi primp wreq
python scripts/scrape_fingerprints.py

# Or add only NEW profiles from one source on top of the current set,
# skipping any whose fingerprint already exists (keeps manual entries)
python scripts/scrape_fingerprints.py --augment --only wreq
```

The scraper dedupes profiles that produce byte-identical handshakes, regenerates `httpz/presets.py`, and regenerates `httpz/_types.py` (the `BrowserTypeLiteral`) alongside it so the two never disagree. Manually-added profiles survive `--augment` but are wiped by a full rebuild, so keep a backup if you've curated any.
