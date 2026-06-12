# Apple India Authorised (Resellers + Service Providers + Retail)

**Domain:** apple.com (locator at `locate.apple.com/in/en/`)
**Vertical:** Sales / distribution — consumer electronics retail + dealer + service-centre network
**Last verified:** 2026-05-21
**Tier:** 2 (direct JSON API on same origin as the SPA)
**Framework:** React/Vite SPA on Spring backend (`locate.apple.com` — `server: Apple`)
**Protection:** none. Plain Chrome UA + Referer is sufficient. No cookies, no warmup, no JS-rendered tokens.

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| Locator landing | `locate.apple.com/in/en/` | https://locate.apple.com/in/en/ |
| Sales locator (SPA) | `locate.apple.com/in/en/sales` | https://locate.apple.com/in/en/sales |
| Service locator (SPA) | `locate.apple.com/in/en/service` | https://locate.apple.com/in/en/service |
| Channel resources API | `locate.apple.com/api/v1/grlui/{COUNTRY}/{LANG}/globalmatrix` | `/api/v1/grlui/IN/en/globalmatrix` |
| Search API | `locate.apple.com/api/v1/grlui/{country}/{lang}/{channel}?pt=&lat=&lon=&carrier=&maxrad=&maxResult=&repairType=` | see extractor |

Notable: the YAML's `official_locator_url` `https://locate.apple.com/in/en/` is the landing page; the actual sales channel for India is the `/sales` sub-route. Both surface the SAME backend.

## Hydration payload

- **Location:** XHR JSON at `locate.apple.com/api/v1/grlui/in/en/sales`
- **Envelope shape:** `{results: {stores: [...], badgesArray: [...], activeFilters: [...], productMatrixList: [...]}}`
- **Schema (key paths):**
  - `id` → numeric internal record id (e.g. `"29338776"`)
  - `storeId` → stable PK — alphanumeric (`R756` for Apple BKC etc., `"8577563"` for an APR)
  - `title` → free-text outlet name
  - `latitude`, `longitude` → 100% populated
  - `city`, `state` (code), `district` (full state name on retail records), `street1`, `street2`, `postalCode`
  - `phone` → mix of Apple toll-free (`000800-…`) and outlet local numbers
  - `salesProducts: number[]` — product ids the outlet sells (empty on service-only AASPs)
  - `serviceProducts: number[]` — repair-class ids the outlet services
  - `storeBadges: number[]`, `storeTypes: number[]` — badge ids (see classifier)
  - `storeWeb`, `storeEmail` — Apple-retail only (resellers usually have empty)

## Extraction strategy

Tier 2 direct API. The XHR is on the same origin as the SPA, served by a Spring backend (`server: Apple` header). A plain Chrome UA + `Referer: https://locate.apple.com/in/en/sales` + `Origin: https://locate.apple.com` works first-try. POST is rejected at the LB with 405 ("Request method 'POST' is not supported.") — the SPA's TypeScript source claims `ke.post("/search", ...)` but the implementation in the bundled JS builds a **GET** URL `${country}/${lang}/${channel}?pt=&lat=&lon=&carrier=&maxrad=&maxResult=&repairType=`. The `post` string is leftover code that never runs against the IN endpoint. **Use GET.**

Empirically the `sales` and `service` channels return the SAME store list for India — both surface every outlet, and the per-record `salesProducts` / `serviceProducts` arrays let us discriminate the kind of outlet. We sweep only `sales` to halve the request budget.

**Hard cap of 99 stores per query** (declared in `globalmatrix.channels.sales.maxResults`). We sweep 85 centres (82 cities + 3 explicit Apple-owned anchors) and dedupe by `storeId`. 6874 raw rows → 3998 unique locations.

Polite pacing: 400ms between requests. No throttling observed across an 85-request sweep.

## locationType breakdown (live sweep, 2026-05-21)

| storeTypes badge id | Label (from `badgesArray`) | LocationType | Count |
|---|---|---|---|
| 5 | Apple Retail Store | `store` | 5 |
| 1 | Apple Premium Reseller | `dealer` | (subset of dealer) |
| 2 | Apple Shop (in Croma / Reliance) | `dealer` | (subset of dealer) |
| [] + salesProducts non-empty | Apple Authorised Reseller (no badge) | `dealer` | 3806 (total dealer) |
| [] + serviceProducts only | Apple Authorised Service Provider (AASP) | `service-centre` | 187 |
| **Total** | | | **3998** |

The 5 Apple-owned retail stores are **Apple BKC** (Mumbai, R744), **Apple Borivali** (Mumbai, R757), **Apple Saket** (New Delhi, R756), **Apple Noida** (Noida, R787), and **Apple Koregaon Park** (Pune, R788). The press at time of writing (May 2026) cites "BKC + Saket" as the first two — Noida, Borivali and Koregaon Park appear to be more recent openings or about-to-open outlets surfaced by the locator. The API does not publish authoritative-public counts for resellers or AASPs.

## Field inventory

| Field | Source path | Notes |
|---|---|---|
| `brand` | (constant) | "Apple India Authorised" |
| `storeId` | `storeId` | Stable PK — `R{nnn}` for Apple-owned, numeric string for resellers/AASPs |
| `name` | `title` | Verbatim Apple label |
| `locationType` | derived | see classifier above |
| `storeType` | derived | One of 5 human-readable labels; mirrored to `_extra.source_store_type` |
| `address.line1` | `street1` | |
| `address.line2` | `street2` | nullable |
| `address.city` | `city` | |
| `address.state` | `district` ∥ `state` | Prefer full name (`district`); fall back to 2-3 letter code (`state`) |
| `address.postalCode` | `postalCode` | |
| `lat`, `lng` | `latitude`, `longitude` | 100% populated, all inside India bounds |
| `phone` | `phone` | Verbatim. Mix of toll-free and local. |
| `email` | `storeEmail` | Populated only on Apple-owned retail |
| `url` | `storeWeb` | Populated only on Apple-owned retail |
| `_extra.salesProducts` | `salesProducts` | Preserved for discriminating sales vs service tilt |
| `_extra.serviceProducts` | `serviceProducts` | Preserved |
| `_extra.storeBadges` | `storeBadges` | Preserved for classification audit |
| `_extra.storeTypes` | `storeTypes` | Preserved |

Intentionally dropped (search-query-relative or UI-only):
- `distance` — relative to the query lat/lng
- `orderNumber` — relative to the query lat/lng
- `infoBubbleOpen` — UI flag

## Files in this repo

- **Extractor:** `scrapers/sales-distribution/apple-india-authorised.ts`
- **Production runner:** `scrapers/sales-distribution/apple-india-authorised-run.ts`
- **Fixtures:**
  - `__tests__/fixtures/apple-india-mumbai-sales.json` (98 stores — broad mix of dealer + AASP)
  - `__tests__/fixtures/apple-india-saket-sales.json` (96 stores — includes the Apple-owned retail records)
- **Tests:** `__tests__/apple-india-authorised-stores.test.ts` (16 tests)
- **Output:** `data/results/apple-india-authorised-stores.jsonl` (3998 records)

## Pagination

No pagination. The only knob is `maxResult` (server cap 99 in IN feed). Workaround: lat/lng sweep across India and dedupe by `storeId`. Apple's `globalmatrix.channels.sales.searchRadius=100` and `maxResults=99` are the declared service limits; the actual server obeys `maxrad` up to ~200km but truncates at 99 results.

## Gotchas

- **POST is rejected with 405 at the LB.** The SPA's TypeScript source declares `ke.post("/search", body)` but the bundled JS implementation builds a GET URL. Use GET. Don't follow the source.
- **Sales and service channels return the SAME results for India.** Don't sweep both — it would double the request budget for zero new data. Discriminate by `salesProducts`/`serviceProducts` per record instead.
- **Apple Saket isn't in a 200km query from New Delhi centre** (it sometimes isn't — depends on result-ordering — and the 99-cap can bury it). Add explicit anchor coordinates for each Apple-owned retail store to the sweep so the radius doesn't push them off.
- **5 Apple-owned outlets, not 2.** Press from early 2024 cites BKC + Saket as the first two. The locator now lists Apple Borivali, Apple Noida, Apple Koregaon Park as well — likely newer or pre-opening listings. The YAML's `notes` should not hard-code the "2-stores" assumption.
- **State field is inconsistent.** On Apple-owned records `district` carries the full state name ("Delhi"); on dealers `state` carries a 2-3 letter code ("DEL", "UP") and `district` is empty. We promote whichever is non-empty.
- **`distance` and `orderNumber` are search-relative.** Drop them on extraction — they would change meaning if re-queried.
- **The 99-result cap is undocumented in the response itself.** No flag says "truncated". Plan for a sweep from day one.
- **Empty `storeBadges` for plain Authorised Resellers.** The badgesArray declares 3 badges (APR, Apple Shop, Apple Retail) but the majority of outlets (Reliance Digital, Vijay Sales, Croma without "Apple Shop" sub-section, etc.) have empty badges. We classify them as `dealer` if they have `salesProducts`.

## Scorecard (2026-05-21 baseline)

| Dimension | Score |
|---|---|
| Completeness | 57/100 (no authoritative count to compare against; metro coverage 100%) |
| Data quality | 84/100 (100% coords, ~100% address, most have phone, no hours field) |
| Source authenticity | 99/100 (Apple-owned subdomain, public company, Wikipedia, known brand) |
| Freshness | 75/100 (scraped today; no source-side `updated_at`) |
| **Overall** | **75/100 (medium)** |

The "medium" band is driven by the missing authoritative reference (Apple does not publish reseller / AASP counts). Improving requires either Apple publishing a footprint count, or curating one from Apple's India annual filings / SEC disclosures.
