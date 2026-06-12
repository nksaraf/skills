# Uniqlo India — store locator

**Domain:** map.uniqlo.com (locator subdomain; main shop is `www.uniqlo.com/in/en/`)
**Vertical:** India retail (Fast Retailing — Japan; global fast-fashion competitor to Zara/H&M)
**Last verified:** 2026-05-20
**Tier:** 2 (direct JSON API — no browser needed)
**Framework:** React SPA (CRA) at `map.uniqlo.com/<country>/en/` shared globally; backed by `/in/api/storelocator/v1/` JSON API
**Protection:** mild Akamai (homepage serves Akamai sensor `/UR9zY5/...`) but the JSON API itself returns 200 with just `Origin` + `Referer` headers — no cookie warmup needed.

## URL patterns

| Page type | URL | Method |
|---|---|---|
| Store-locator SPA | `https://map.uniqlo.com/in/en/` | GET (HTML, JS bundle, no useful data in DOM) |
| Country state list | `https://map.uniqlo.com/in/api/storelocator/v1/en/states` | GET |
| State stores (paginated by 5) | `https://map.uniqlo.com/in/api/storelocator/v1/en/stores?stateCode=<N>&offset=<N>&RESET=true&lang=english&r=storelocator` | GET |
| CMS / static config | `https://map.uniqlo.com/in/api/storelocator/v1/en/cms?path=/storelocator/top/` | GET (not needed for extraction) |
| Per-store detail | `https://map.uniqlo.com/in/en/detail/<storeId>` | GET (SPA, not used) |

**Key insight:** the `map.uniqlo.com/<country>/api/storelocator/v1/en/` path pattern is uniform across Fast Retailing's global locator (Uniqlo Spain, France, EU, etc. — all served from the same SPA at `map.uniqlo.com/<cc>/<lang>/`). Swapping `/in/` for another two-letter country code likely gives the same shape for other markets — worth probing for cross-market analysis.

## Required headers

```
Accept: application/json
Origin: https://map.uniqlo.com
Referer: https://map.uniqlo.com/in/en/
User-Agent: <any Chrome string>
```

No cookies, no API key, no auth required.

## Pagination

- `pagination.total`, `pagination.count`, `pagination.offset`.
- `count` caps at 5 per request — so a state with 6 stores needs 2 calls (offset 0, offset 5).
- States with `total: 0` are returned with an empty `stores: []` and no error.

## Schema (per store)

35 fields per record. Promoted to canonical:

| Canonical field | Source key |
|---|---|
| `storeId` | `id` (e.g. `"12400017"`) |
| `name` | `name` (e.g. `"UNIQLO Inorbit Mall Malad Mumbai"`) |
| `status` | `businessStatus` (`"OPEN"` → `"open"`) |
| `address.line1` | `address` (free-form, comma-separated) |
| `address.city` | inferred from address (last segment before state) |
| `address.state` | resolved via `stateCode` → `stateName` from states endpoint |
| `address.postalCode` | **not exposed** — `null` always |
| `address.country` | hard-coded `"IN"` (locator is `/in/`) |
| `lat` / `lng` | `latitude` / `longitude` (strings — parsed to number) |
| `phone` | `phone` (e.g. `"+91-9319094409"`) |
| `url` | derived: `https://map.uniqlo.com/in/en/detail/<id>` |
| `hours` | `wdOpenAt`/`wdCloseAt` (Mon–Fri) + `weHolOpenAt`/`weHolCloseAt` (Sat–Sun) |
| `storeType` | `storeTypeName` (e.g. `"LARGE-SCALE STORES"`, `"STANDARD STORES"`, `"SMALL STORES"`) |
| `segments` | `productTypeList[].type` (e.g. `["WOMEN", "MEN", "KIDS", "BABY", "MATERNITY"]`) |
| `features` | merged from `clickAndCollectFlagName`, `parkingFlagName`, `payAtStoreFlagName`, `accessibilityFlags[].name`, `storeServiceFlags[].name` |

Preserved in `_extra`:

- `locationId` — Uniqlo's internal location ID (different from `id`)
- `announcement` — banner text like `"Opened Now"` shown in UI
- `recruitFlag` / `recruitLinkUrl` / `regionalRecruitFlag` / `regionalRecruitLinkUrl` — careers links
- `irregularOpenTimeComment` / `irregularOpenTimeShortComment` — holiday-hours notes
- `temporaryCloseDateComment` — closure-period notes
- `pickupLocation` — click-and-collect pickup desk info
- `businessStatusShortComment` — short-form status banner
- `distance` — empty in the unfiltered API response (would be populated if querying by lat/lng)

## Files in this repo

- **Extractor:** `@vinxi/scraper/retail` (canonical schema) + `scrapers/retail/uniqlo.ts` (per-record mapper)
- **Production scraper:** `scrapers/uniqlo-stores.ts` (walks 36 state codes)
- **Recon demo:** `scrapers/recon-uniqlo-stores.ts`
- **Fixtures:** `__tests__/fixtures/uniqlo-stores-maharashtra-page0.json` (5 stores), `…-page1.json` (1 store)
- **Tests:** `__tests__/uniqlo-stores.test.ts` (11 schema-locked tests)

## Coverage

- **20 stores across 7 states** (Delhi 5, Maharashtra 6, Karnataka 3, Haryana 2, Uttar Pradesh 2, Punjab 1, Chandigarh 1). Matches Uniqlo's known India footprint — they entered in 2019 and stayed concentrated in NCR / Mumbai / Bengaluru.
- **All 20 stores have lat/lng + storeId + phone + hours.**
- **0 stores have email** (Uniqlo doesn't expose per-store email anywhere in their global API).
- **0 stores have postalCode** (not in API; would need to geocode addresses).

Fetched in ~89s across 36 states (most return zero stores immediately).

## Gotchas

- **Lat/lng are strings**, not numbers. Parser coerces.
- **Pagination is 5/page** — easy to miss the second page if you just request offset=0. The Inorbit Mall Malad store sits at offset 5 in Maharashtra.
- **`stateCode` is the API's own numeric code** (1–36) from the `/states` endpoint — NOT a standard Indian state code. Always pre-fetch `/states` to get the mapping.
- **City inference is heuristic** — Uniqlo's `address` is one long comma-separated string with the state at the tail. The parser strips the state then takes the last surviving segment, which is usually a locality/area (e.g. "Sector 24" instead of "Gurgaon" for the Ambience Mall store). The raw address is preserved verbatim in `address.line1`.
- **`businessStatus` is sparsely documented** — we've only observed `"OPEN"`. Other values (`"CLOSED"`, `"COMING_SOON"`, `"TEMPORARILY_CLOSED"`) are mapped defensively.
- **`announcement` field is interesting** — for new stores it says `"Opened Now"`. Could be useful as a "newly-opened" signal in time-series scrapes.
- **The API is unauthenticated and fast.** Be polite anyway — 0.5–1.5s between calls is enough; 89s for the full walk.

## Cross-market opportunity

Path pattern `map.uniqlo.com/<cc>/api/storelocator/v1/en/...` likely works for every Uniqlo country listed in the SPA (`/de/`, `/fr/`, `/eu/`, `/sg/`, `/my/`, `/ph/`, `/au/`, `/us/`, `/ca/`, etc.). Same extractor with `BRAND = "Uniqlo"` and a per-country `country` parameter to override the hard-coded `"IN"` would generalise instantly. Parking this for now — the brief was India only.

## Cross-brand notes

Uniqlo is the third Indian retail brand fully extracted (after Westside and Mango). Pattern reuse:

- `RetailStore` interface, `extraOf()`, `normaliseCountry()` — reused
- Same per-day hours convention (`{"mon":"11:00-22:00",…}`)
- Same status enum (`open` | `closed` | `coming-soon` | `unknown`)
- Same `_extra` preservation rule — every non-promoted field with a non-empty value survives

The fact that Uniqlo addresses have no separate postal-code field is a real cross-brand data-quality gap; future cross-brand joins should not assume postal codes are populated.
