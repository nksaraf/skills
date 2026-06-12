# Mango India — store locator

**Domain:** shop.mango.com (locator at `/in/en/stores`)
**Vertical:** India retail (Spanish fast fashion — Zara competitor)
**Last verified:** 2026-05-20
**Tier:** 1 (direct HTML fetch — store data is fully SSR'd inside the Next.js RSC stream)
**Framework:** Next.js App Router (React Server Components)
**Protection:** Akamai Bot Manager (set-cookie `_abck`, `bm_sz`, `x-akamai-transformed`). Passes a single curl with realistic Chrome UA + en-IN locale; fast reruns can hit "Access Denied" — pace gently.

## URL patterns

| Page type | URL | Method |
|---|---|---|
| Store listing | `https://shop.mango.com/in/en/stores` | GET (HTML) |
| Store detail | `https://shop.mango.com/in/en/stores/{relativeUrl}` (e.g. `gurgaon/ambience-mall/2048`) | GET (HTML) |

**Key insight:** The earlier recon turn saw an XHR to `api.shop.mango.com/e/api.woosmap.com/localities/geocode` and assumed Woosmap was the canonical store source. **It is not.** Woosmap is only used for the geocoder-typeahead in the search box. The store data itself ships inside the page HTML as a React Server Components stream — no API call, no browser needed.

## Hydration payload

- **Location:** Next.js App Router RSC stream — a sequence of `self.__next_f.push([1, "<chunk>"])` calls interleaved through the HTML. Concatenating every chunk reproduces the line-prefixed RSC stream (`<hex>:<json>`).
- **Listing canonical path:** any RSC line whose payload nests `{ storesFromLite: MangoStoreLite[] }` (observed at line `4f` in the 2026-05-20 capture but the exact index changes per build — the extractor walks the tree, doesn't pin the index).
- **Detail canonical path:** any RSC line whose payload nests `{ initialState: { stores: [MangoStoreFull] } }` (observed at line `55`).

### Listing schema (`MangoStoreLite`)

```jsonc
{
  "id": "2048",
  "collections": ["Women"],
  "clickAndCollectExpressActive": false,
  "url": "gurgaon/ambience-mall/2048",
  "addresses": {
    "address": "G-36 AMBIENCE MALL",
    "city": "Gurgaon",            // mixed case across stores: "Gurgaon"|"NEW DELHI"|"MUMBAI"|"Mumbai"
    "postalCode": "122001",
    "shoppingCenter": "AMBIENCE MALL",
    "coordinates": { "lat": 28.5053725, "lng": 77.096411499999, "distance": -1 }
  },
  "todaySchedule": { "dayOfWeek": "Wednesday", "timeList": [{...}, {...}] }
}
```

### Detail schema (`MangoStoreFull`)

```jsonc
{
  "id": "2048",
  "collections": ["Women"],
  "details": {
    "timeSchedule": [{ "dayOfWeek": "Monday", "timeList": [{"openHour":"11:00","closeHour":"22:00"},{"openHour":"-","closeHour":"-"}] }, ...],
    "phone": "01244665318",
    "isCorner": false,
    "urlVisitStore": null,
    "showVisitStore": false,
    "sizesIds": [],
    "sizes": [],
    "sapId": null,
    "tiposVenta": ["ONLINE","ONLINE_TELEFONICA","ONLINE_CLIENTE","VOTF","FRANQUICIAS","TIENDAS_PROPIAS","EMPLEADO","EMPLEADO_CENTRAL","EMPLEADO_NO_CENTRAL"],
    "url": "gurgaon/ambience-mall/2048"
  },
  "addresses": { ...same as Lite }
}
```

## Field inventory

| Canonical field | Source (lite → detail) | Notes |
|---|---|---|
| `storeId` | `id` | string of digits ("2048", "8678") |
| `name` | `addresses.shoppingCenter` | mall/centre name — Mango doesn't ship a separate "store name" |
| `status` | (none) | Mango lists only live stores — hard-coded to `"open"` |
| `mallOrLocation` | `addresses.shoppingCenter` | same as name |
| `address.line1` | `addresses.address` | sometimes upper-case ("G-36 AMBIENCE MALL") |
| `address.city` | `addresses.city` (Title-cased) | source emits mixed case — normalised to Title Case |
| `address.postalCode` | `addresses.postalCode` | always set |
| `address.country` | (hard-coded) | always "IN" — the locator is `/in/en/` |
| `lat` / `lng` | `addresses.coordinates.lat` / `.lng` | always numbers (not strings) |
| `phone` | detail `details.phone` | not present on lite |
| `url` | derived from `url` field | listing emits relative path; we prepend `https://shop.mango.com/in/en/stores/` |
| `hours` | detail `details.timeSchedule` (7-day) or lite `todaySchedule` (1-day) | each day has two slots; the second is `{-,-}` filler when there is no split shift |
| `storeType` | derived from `details.isCorner` | `"Corner"` when `isCorner: true`, otherwise null |
| `segments` | `collections` (UPPER-cased) | always `["WOMEN"]` for the India dataset |
| `features` | derived | `"Click & Collect Express"` when `clickAndCollectExpressActive`; `"Visit Store"`; `"Corner"` |

Preserved in `_extra` (detail extractor flattens `details.*` non-promoted sub-fields):

- `clickAndCollectExpressActive` (boolean — every Indian store currently false)
- `isCorner` (boolean)
- `showVisitStore`, `urlVisitStore` (boolean + nullable URL — every Indian store currently false/null)
- `sizesIds`, `sizes` (sized-stock channel info — empty arrays in India)
- `sapId` (SAP system id — null in India; populated in EU stores per the field name)
- `tiposVenta` (Spanish "sales channels" enum — the most informative `_extra` field: identifies online/franchise/employee channels)

## Extraction strategy

Tier 1: two `ctx.fetch` calls per store.

1. Listing fetch → `extractStoresFromListingHtml(html)` returns 9 `MangoStoreLite`.
2. For each lite, fetch the detail URL → `extractStoreFromDetailHtml(html)` returns `MangoStoreFull`; `extractMangoStoreFull` projects to `RetailStore`.
3. If a detail fetch fails (HTTP error or parse failure), fall back to `extractMangoStoreLite` so we never lose the store entirely — log a warn so we notice.

## Coverage

- **9 stores across 7 cities** (Mumbai 2, New Delhi 2; Ahmedabad, Bangalore, Gurgaon, Kolkata, Pune one each)
- **All 9 have city + storeId + lat/lng + 7-day hours + phone** populated. Mango India is small but the data is uniformly complete.

## Files in this repo

- **Extractor library:** `@vinxi/scraper/retail` (canonical schema) + `scrapers/retail/mango.ts` (RSC parser + per-record mapper)
- **Production scraper:** `scrapers/mango-stores.ts`
- **Fixture:** `__tests__/fixtures/mango-stores-india.json` (9 lite records + 1 full detail)
- **Tests:** `__tests__/mango-stores.test.ts` (11 schema-locked tests)
- **Recon scratch:** `scrapers/recon-mango-stores.ts`

## Pagination

None — Mango India is one page with all 9 stores embedded. There is no `?page=N`, no infinite scroll, no city filter that requires re-fetching.

## Gotchas

- **The XHR rabbit hole is a red herring.** The Woosmap geocode call at `api.shop.mango.com/e/api.woosmap.com/localities/geocode` returns *geocoder typeahead* for the search box. The store list is **not** behind any API — it is in the page HTML. The previous recon turn missed this because `extractAllHydrationPayloads()` does not understand Next.js App Router's RSC stream format (it looks for `window.X`, `__NEXT_DATA__`, JSON-LD — none of which Mango uses).
- **`storesFromLite` only contains today's schedule.** The full 7-day schedule + phone is on the per-store detail page. Skipping the detail fetch costs you both.
- **Mango's `timeList` always has TWO slots per day.** The second is `{"openHour":"-","closeHour":"-"}` filler when there is no split shift. The extractor drops the filler.
- **City casing is inconsistent across rows** (`"Gurgaon"` vs `"NEW DELHI"` vs `"Mumbai"` vs `"MUMBAI"`). We normalise to Title Case (`normaliseCity`). If you don't, downstream group-by-city will double-count Mumbai.
- **Akamai will rate-limit fast reruns.** Previous recon attempts got "Access Denied" mid-session. 1.5–3.5s between detail fetches is sufficient — the full run takes ~40s.
- **The RSC line index is unstable across builds.** We saw `storesFromLite` at line `4f` and detail at line `55` in this capture; do NOT pin the indices. The extractor walks the JSON tree looking for the shape, not the index.

## Candidates for promotion to canonical (future schema revision)

- `tiposVenta` — Mango is the first brand we have that exposes "sales channels" (online vs franchise vs employee). If Zara/H&M expose similar, promote to canonical `salesChannels: string[]`.
- `clickAndCollectExpressActive` — already represented in `features` when true, but the explicit boolean (including `false`) is in `_extra`. If three brands ship the same flag, promote to canonical `clickAndCollect: boolean`.

## Why this is Tier 1 (and not Tier 2 via Woosmap)

We probed for a Woosmap stores endpoint, but only saw `localities/geocode` (the typeahead). A Tier 2 hit on Woosmap would require either:

1. **Knowing the Woosmap private key** for Mango's account — exposed in the geocode URL as a `key=...` query param. But Mango proxies through `api.shop.mango.com/e/...`, so the proxy may strip/append the key server-side and not expose the upstream credentials.
2. **A direct `/stores/search` call** — never observed in our captures, because the page already has the data SSR'd.

Tier 1 was strictly better here: one HTTP call per page, no key management, no proxy semantics to reverse-engineer.

## Cross-brand notes

Mango is the second Indian retail brand fully extracted (Westside being the first). It validates that the `_extra` preservation pattern works for an entirely different stack (Next.js RSC vs Shopify + custom API) without changing the canonical schema. The `reassembleRSCStream()` helper in `scrapers/retail/mango.ts` is generic and could be moved to a shared utility if/when we see another Next.js App Router brand (e.g. Uniqlo, Zara — both Next.js properties).
