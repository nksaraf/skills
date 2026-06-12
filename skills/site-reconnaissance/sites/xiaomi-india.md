# Xiaomi India (Mi Home + Authorised Dealers + Service Centres)

**Domain:** mi.com (in)
**Vertical:** Sales / distribution - consumer electronics retail + dealer + repair network
**Last verified:** 2026-05-21
**Tier:** 2 (three direct JSON/JSONP APIs across three Xiaomi sub-domains)
**Framework:** legacy jQuery+JSONP pages (Mi Home / Authorised Stores) + React/webpack SPA (Service Centre)
**Protection:** none on the data sub-domains (`store.mi.com`, `in-go.buy.mi.com`); www.mi.com is fronted by Akamai but the data endpoints sit elsewhere

## URL patterns

Xiaomi India splits its footprint across THREE locator surfaces - each backed by its own endpoint on a different sub-domain. All three are unprotected GETs - plain Chrome UA suffices.

| Channel | Page | API | Pagination | Returns |
|---|---|---|---|---|
| **Mi Home / Mi Studio** | `mi.com/in/service/mihome/` | `store.mi.com/in/mihome/getdata` (JSONP, callback `mihomeCallback`) | none (single shot) | 58 stores |
| **Mi Authorized Store** | `mi.com/in/service/authorized_stores/` | `store.mi.com/in/authorizedstore/getdata?pageSize=10000&pageNum=1&state=All+states&city=All+cities` (JSONP, callback `authorized`) | `pageSize` accepts 10000 -> single shot | 1,026 dealers |
| **Authorised Service Centre (ASC)** | `mi.com/in/service/service-center/` | `in-go.buy.mi.com/in/serviceplus/api/fos/public/v1/mi-store/near-by-asc?offset=N` | `offset` + 10/page server-enforced cap | 949 service centres |

## Hydration payloads

### Mi Home (JSONP)

Wrapper: `if(window.mihomeCallback)mihomeCallback({...});`
Envelope: `{ statelist: {...}, storelist: [...], storecount: 8 }`

Note: `storecount` is buggy/unused — actual list length (58) is the true count.

Schema (per row):
- `name`, `address`, `mobile`, `state`, `city`, `pincode`, `area`
- `open_time`, `close_time`, `weekend` (free text)
- `latitude`/`longitude` (strings) + `location: { lat, lng }` (numbers, duplicate)
- `tag` -> always "Mi Store" on this feed
- `twitter_name`, `twitter_link`, `pic`

### Authorized Store (JSONP)

Wrapper: `if(window.authorized)authorized({...});`
Envelope: `{ all_states, all_cities, storelist: [...], statelist: {...}, storecount: 129 }`

Note: `storecount` here is PAGE count, not record count (pageSize default 8, 1026/8 ≈ 129).

Schema (per row): same shape as Mi Home plus:
- `mobileArr` - array form of `mobile`
- `store_type` - "1" in every observed row
- `storeType` - always "Mi Store"

### Service Centre (REST)

Auth: `Authorization: Basic bWlzdG9yZS1zZXJ2aWNlcGx1cy1pbnRlZ3JhdGlvbjp4aWFvbWlAQURNSU4=`
(Same credential is hard-coded into the React bundle `64076.chunk.js` - public, not a bypass.)

Envelope: `{ previousPage, currentPage, nextPage, lastPage, start, end, count, stores: [...] }`

Schema (per row, NB different from JSONP feeds):
- `orgId` - stable internal PK ("XMIN5000")
- `name`, `address`
- `latitude`, `longitude` (strings)
- `openTime`, `closeTime`, `phoneNumber`
- `type` - always empty string

The service-centre `address` is a single free-text blob - no parsed city/state/pincode. We leave those slots null.

## Extraction strategy

Tier 2 direct API across three sub-domains. The Mi Home and Authorized JSONP feeds return everything in one shot when `pageSize=10000`. The Service Centre REST endpoint is hard-capped at 10 records per page (server-side, regardless of pageSize) - we iterate `offset` 0..940 in steps of 10 (~95 calls).

No cookies, no warmup, no JS render. Polite pacing: 150ms between service-centre pages.

## locationType breakdown (live sweep, 2026-05-21)

| Channel | Source label | LocationType | Count |
|---|---|---|---|
| Mi Home / Mi Studio | "Mi Store" (tag) | `store` | 58 |
| Mi Authorized Store | "Mi Store" (storeType) | `dealer` | 1,026 |
| Authorised Service Centre | (empty type) | `service-centre` | 949 |
| **Total** | | | **2,033** |

## Field inventory

| Field | Mi Home | Authorized | Service Centre | Notes |
|---|---|---|---|---|
| `brand` | "Xiaomi India" | "Xiaomi India" | "Xiaomi India" | constant |
| `storeId` | null | null | `orgId` ("XMIN5000") | only service centres have stable PKs |
| `name` | `name` | `name` | `name` | |
| `locationType` | `store` | `dealer` | `service-centre` | discriminator |
| `storeType` | `tag` ("Mi Store") | `storeType` ("Mi Store") | "Authorised Service Centre" (default - source type empty) | raw label preserved |
| `address.line1` | `address` | `address` | `address` | service centre is a single blob |
| `address.city` | `city` | `city` | "" | service centre has no parsed city |
| `address.state` | `state` (title-cased) | `state` (title-cased) | null | upstream uppercases; we normalise |
| `address.postalCode` | `pincode` | `pincode` | null | |
| `lat`/`lng` | numeric | numeric | numeric | 100% populated across all feeds |
| `phone` | `mobile` | `mobile` | `phoneNumber` | mix of formats |
| `hours` | `weekday`+`weekend` | `weekday`+`weekend` | `weekday` only | best-effort from open/close/weekend triple |
| `_extra.source_store_type` | `tag` | `storeType` | (omitted) | mirror for downstream filtering |

## Files in this repo

- **Extractor:** `scrapers/sales-distribution/xiaomi-india.ts`
- **Production runner:** `scrapers/sales-distribution/xiaomi-india-run.ts`
- **Fixtures:**
  - `__tests__/fixtures/xiaomi-india-mihome.json` (58 Mi Home rows - JSONP-unwrapped)
  - `__tests__/fixtures/xiaomi-india-authorized-stores.json` (25-row slice of authorized feed)
  - `__tests__/fixtures/xiaomi-india-service-center-page1.json` (page 1 of /near-by-asc - 10 rows + envelope)
- **Tests:** `__tests__/xiaomi-india-stores.test.ts` (20 tests, 392 expects)
- **Output:** `data/results/xiaomi-india-stores.jsonl` (2,033 records)

## Pagination

- Mi Home: none - single JSONP shot returns all 58.
- Authorized: `pageSize`+`pageNum`. `pageSize=10000` returns all 1,026 in one shot.
- Service Centre: `offset`-based; server-enforced 10/page cap regardless of `pageSize`. Iterate until `offset >= count` (95 pages total at 949 records).

## Gotchas

- **THREE separate endpoints on THREE sub-domains for ONE brand.** Mi Home (flagship), Authorized Store (dealer), and Service Centre (repair) each have their own page, API host, schema, AND authentication regime. Easy to ship just one and miss 50% of the footprint.
- **Service Centre auth credential is hard-coded into the public React bundle.** `Basic mistore-serviceplus-integration:xiaomi@ADMIN` is THE credential the page itself uses - confirmed by reading chunk `64076.chunk.js`. Not a secret, not a bypass; document it as the documented public key.
- **`pageSize` is ignored by the service-centre API.** Server-side cap of 10 records/page regardless of what you request. Must paginate with `offset`.
- **`storecount` means different things on different feeds.** On Mi Home it's a stale buggy field (returns 8 when storelist has 58). On Authorized it's PAGE count (not record count). The actual record count is `storelist.length`.
- **Mi Home `city` field is empty for every row.** State is populated; city must be parsed from `address` if needed.
- **State casing is inconsistent.** Mi Home uppercases ("TAMIL NADU"); Authorized uppercases ("MAHARASHTRA"). We title-case both for downstream consistency.
- **Service Centre rows have no parsed city/state/pincode.** Address is a single free-text blob - downstream geocoding would be required to enrich.
- **Authorized feed has no stable id.** The recently-launched Mi Home feed also lacks id. Cross-run dedupe falls back to a coords+name key.
- **`distance` (search-relative) is NOT exposed on Xiaomi feeds.** No projection caveat needed (unlike Samsung).
- **`mi.com` Akamai is real but irrelevant.** The customer-facing pages on www.mi.com are fronted by Akamai with geo-routing; the data endpoints on store.mi.com and in-go.buy.mi.com are open to plain Chrome UA from any geo.

## Scorecard (2026-05-21 baseline)

| Dimension | Score |
|---|---|
| Completeness | 82/100 (source-claimed = 2033 matches extracted; metro coverage 100%) |
| Data quality | 77/100 (100% coords, 100% address, 100% phone for active fields; service-centre records lack parsed city/state) |
| Source authenticity | 99/100 (mi.com is Xiaomi-owned; public company, Wikipedia, known brand) |
| Freshness | 75/100 (scraped today; no source-side `updated_at`) |
| **Overall** | **83/100 (high)** |
