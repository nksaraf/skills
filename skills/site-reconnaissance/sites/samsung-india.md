# Samsung India (Retail + Authorised Dealers)

**Domain:** samsung.com (in)
**Vertical:** Sales / distribution — consumer electronics retail + dealer network
**Last verified:** 2026-05-21
**Tier:** 2 (direct JSON API)
**Framework:** AEM SPA (`pd-g-store-locator` component) calling Samsung's global searchapi
**Protection:** none on `searchapi.samsung.com`; `www.samsung.com` AEM endpoints behind Akamai with geolocation gating

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| Locator page | `samsung.com/in/storelocator/` | https://www.samsung.com/in/storelocator/ |
| Locator API (retail + dealer) | `searchapi.samsung.com/v6/front/b2c/storelocator/list?siteCode=in&distanceUnit=6371&latitude={lat}&longitude={lng}&nRadius={km}&categoryList=&shopTypeList=` | see `scrapers/sales-distribution/samsung-india.ts` |
| Service-centre page | `samsung.com/in/support/service-center/` | (separate AEM endpoint — see Gotchas) |
| Service-centre API | `samsung.com/aemapi/v6/support/ds.do/front/in/support/servicelocation/list` | gated by Akamai geo — empty body from outside India |

## Hydration payload

- **Location:** XHR JSON at `searchapi.samsung.com`
- **Envelope shape:** `{response: {statusCode, resultData: {common: {storesCount}, stores: [...]}}}`
- **Schema (key paths):**
  - `brandType` → free-text Samsung label ("Experience Store" | "Brand Store" | "Others Store")
  - `brandTypeCode` → one-letter code: `E` | `B` | `O`
  - `id` → Samsung's internal store id (stable PK across queries)
  - `name`, `latitude`, `longitude`, `cityName`, `address`, `postalCode`, `area`, `phone`, `email`, `homepageUrl`, `tvExpZoneFlag`

## Extraction strategy

Tier 2 direct API. The XHR endpoint is open to plain Chrome UA + a `Referer: https://www.samsung.com/in/storelocator/` header — no cookies, no warmup, no JS render needed.

**Hard cap of ~50 stores per query** regardless of `nRadius`. We sweep 82 city centers (metros + state capitals + tier-2 cities — `INDIA_CITY_SWEEP` in the extractor) and dedupe by `id`. 1509 raw rows → 1173 unique locations.

Polite pacing: 400ms between requests.

## locationType breakdown (live sweep, 2026-05-21)

| brandTypeCode | brandType (raw) | LocationType | Count |
|---|---|---|---|
| E | Experience Store | `store` | 875 |
| B | Brand Store (SmartPlaza) | `store` | 246 |
| O | Others Store | `dealer` | 52 |
| — | (service centres) | `service-centre` | **0 — endpoint geo-gated** |
| **Total** | | | **1173** |

Note: the API does NOT publish a single authoritative count — `storesCount` in the envelope is per-query (within 50km of the query center), not global. The scorecard's `authoritative_count` is null and the source-claim check is a warn (50%).

## Field inventory

| Field | Source path | Notes |
|---|---|---|
| `brand` | (constant) | "Samsung India" |
| `storeId` | `id` | Samsung's internal PK — used for dedupe |
| `name` | `name` | Free-text store name |
| `locationType` | derived from `brandTypeCode` | E,B → `store`; O → `dealer` |
| `storeType` | `brandType` | Verbatim Samsung label, preserved |
| `_extra.source_store_type` | `brandType` | Mirror for downstream filtering |
| `_extra.brandTypeCode` | `brandTypeCode` | Preserved for classification audit |
| `address.line1` | `address` | One-line concatenated street |
| `address.city` | `cityName` | |
| `address.state` | `area` | Samsung uses `area` for state in IN feed |
| `address.postalCode` | `postalCode` | |
| `lat`, `lng` | `latitude`, `longitude` | 100% populated |
| `phone` | `phone` | mix of `+91`-prefixed and bare 10-digit |
| `email` | `email` | populated for franchise contact |
| `url` | `homepageUrl` | usually null |

`distance` (search-relative) is intentionally NOT preserved — would mislead downstream.

## Files in this repo

- **Extractor:** `scrapers/sales-distribution/samsung-india.ts`
- **Production runner:** `scrapers/sales-distribution/samsung-india-run.ts`
- **Fixture:** `__tests__/fixtures/samsung-india-delhi-50km.json` (Delhi 50km capture: 50 stores)
- **Tests:** `__tests__/samsung-india-stores.test.ts` (15 tests)
- **Output:** `data/results/samsung-india-stores.jsonl` (1173 records)

## Pagination

No pagination on this API. The 50-cap is the only knob. Workaround: pincode/lat-lng sweep + dedupe by `id`.

## Gotchas

- **50-store cap, silent.** No flag in the response says "truncated". You only notice if you assume `nRadius` widens the result — it doesn't. Plan for a sweep from day one.
- **Service centres live on a different API.** `samsung.com/aemapi/v6/support/ds.do/front/in/support/servicelocation/list` is gated by Akamai geolocation — from outside India it returns HTTP 200 with `content-length: 0` for every request. The `country_region` cookie Akamai sets (e.g. `CA-ON` for North-American clients) is informational; the body is dropped at edge. To capture service centres, this sweep must run from an IN-located node (or via an IN residential proxy).
- **`brandTypeCode=O` ("Others Store") often has "Samsung SmartPlaza" in the name.** Samsung's own classification distinguishes franchised brand stores (B) from authorised retailers (O) — both may sell only Samsung product. We trust the source classification and label them `dealer` per the cross-industry schema, but the `_extra.source_store_type` + `storeType` slots preserve the raw label.
- **`distance` field is search-relative.** Drop it on extraction — it would change meaning if anyone re-queries.
- **Phone numbers are inconsistently formatted.** Some `+91`-prefixed, some bare 10-digit. We preserve verbatim (no normalisation).
- **`area` is the state, NOT a sub-locality.** Don't be misled by the field name.

## Scorecard (2026-05-21 baseline)

| Dimension | Score |
|---|---|
| Completeness | 57/100 (no authoritative count to compare against; metro coverage 100%) |
| Data quality | 85/100 (100% coords, 100% address, 100% phone; no hours field on this API) |
| Source authenticity | 99/100 (samsung-owned subdomain, public company, Wikipedia, known brand) |
| Freshness | 75/100 (scraped today; no source-side `updated_at`) |
| **Overall** | **76/100 (medium)** |

The "medium" band is driven entirely by the missing authoritative reference, NOT by extraction quality. Improving requires either (a) Samsung publishing a footprint count, or (b) curating one from quarterly investor decks.
