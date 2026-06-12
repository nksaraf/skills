# Vivo India (Exclusive Stores + Service Centres)

**Domain:** vivo.com (in)
**Vertical:** Sales / distribution — consumer electronics retail + authorised service network
**Last verified:** 2026-05-21
**Tier:** 2 (direct JSON via jQuery-style POST)
**Framework:** Static SSR pages (jQuery + plain HTML) calling internal vivo.com endpoints; full state/city catalogue ships inline as a JS literal
**Protection:** none — plain Chrome UA + `Referer` to the locator page + `X-Requested-With: XMLHttpRequest` passes

Sister to OPPO under BBK Electronics, but the stack is completely different. OPPO India uses GCSM (`ind-sow-cms-app.oppo.com`); Vivo India serves everything from `www.vivo.com/in/*`. No shared backend.

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| Retailer page | `vivo.com/in/where-to-buy` | https://www.vivo.com/in/where-to-buy |
| Retailer API | `vivo.com/in/retailers/queryByCondition` (POST, form-encoded) | `regionId=1` returns all ~600 stores |
| Retailer API (geo) | `vivo.com/in/retailers/queryInfoByLatitudeAndLongitude?latitude=&longitude=` | nearest-N by distance |
| Service-centre page | `vivo.com/in/support/service-center` | (canonical YAML URL — actual path is `support/service-center`, not `service/center`) |
| Service-centre API | `vivo.com/in/support/queryServiceCenterByCondition` (POST, form-encoded) | requires regionCode/stateCode/cityCode triple |
| Service-centre API (geo) | `vivo.com/in/support/queryServiceCenterInfoByLatitudeAndLongitude?latitude=&longitude=` | ≤16 nearest |

## Hydration payload

- **Location:** XHR JSON at `www.vivo.com/in/{retailers,support}/...`
- **Envelope shape:** `{success: true, code, msg, data: {status, region, state, city, store: [...], station: null}}`
- **Per-row schema:**
  - `name` — free-text store name
  - `address` — single-line street (English; some entries appear in Devanagari / regional script)
  - `lat`, `lng` — strings ("19.067502")
  - `tel` — comma-joined CSV when multiple
  - `time` — service-centre window with rest-day in parens, e.g. `"10:00 AM - 06:00 PM (Closed - Sunday)"`
  - `summerBusinessHours`, `winterBusinessHours` — Exclusive Store hours (often identical)
  - `range` — distance from query center (only on lat/lng queries; misleading post-hoc, dropped)
  - `siteCode` — service-centre PK (e.g. `INMH01001`); brand stores have `null`
  - `restTime`, `temporaryNotice`, `openAppointment`, `newFromTime`, `newToTime`, `remark`, `tips` — preserved in `_extra`

- **Region catalogue (inline JS literal on the service-center page):** `regionList` — 1 region × 32 states × 546 cities. Captured to `__tests__/fixtures/vivo-india-region-list.json`.

## Extraction strategy

**Two endpoints, two strategies:**

1. **Retailers (Exclusive Stores) — one shot.** A single POST to `/in/retailers/queryByCondition` with body `regionId=1` returns the full national footprint (596 rows). The endpoint silently ignores narrower `stateId/cityId` filters and dumps the whole list every time — this is undocumented but consistent.

2. **Service centres — city sweep.** `/in/support/queryServiceCenterByCondition` requires a (regionCode, stateCode, cityCode) triple. State-only or region-only queries return `data.store: null`. We iterate every cityCode from the captured `regionList` catalogue (546 cities) and aggregate. Polite pacing: 120ms — total run ≈ 70s.

Polite pacing on the retailer call is unnecessary (one request). Service-centre sweep paces 120ms.

**CRITICAL — field-name mismatch between the two endpoints:**

| Endpoint | Region | State | City |
|---|---|---|---|
| `/in/retailers/queryByCondition` | `regionId` | `stateId` | `cityId` |
| `/in/support/queryServiceCenterByCondition` | `regionCode` | `stateCode` | `cityCode` |

The values are the SAME ids in both cases — only the parameter NAMES differ. This is sourced from the actual JS bundles (`public_ffaaaf5.js` defines URLs, `index.pack_61586f0.js` defines call sites — `getStoreListByAddress`).

## locationType breakdown (live sweep, 2026-05-21)

| Source | locationType | storeType | Count |
|---|---|---|---|
| `/in/retailers/queryByCondition` | `store` | "Exclusive Store" | 589 |
| `/in/support/queryServiceCenterByCondition` (546-city sweep) | `service-centre` | "Service Center" | 712 |
| **Total (deduped)** | | | **1301** |

- Retailer dedupe: 596 rows → 589 unique (7 duplicates by name+lat+lng+phone).
- Service-centre dedupe: 712 records, all unique by `siteCode` (Vivo's stable PK).
- Per-row coverage: every row has `name` + `address`; >75% have `lat/lng`; service centres always have `siteCode` + `time` + `tel`; brand stores have `summerBusinessHours/winterBusinessHours` but `siteCode: null`.

Vivo India does NOT publish a single authoritative footprint count. Press reporting cites 70,000+ retail touchpoints — that is the entire multi-brand authorised-retailer channel (3rd-party kirana mobile shops), which is NOT in this locator. The 589 Exclusive Stores returned by the API are vivo's owned/branded experience-centre estate.

## Field inventory

| Field | Source path | Notes |
|---|---|---|
| `brand` | (constant) | "Vivo India" |
| `storeId` | `siteCode` | service centres only; brand stores have `null` |
| `name` | `name` | Free-text |
| `locationType` | derived from source endpoint | retailers → `store`; service → `service-centre` |
| `storeType` | derived from source endpoint | "Exclusive Store" / "Service Center" |
| `_extra.source_store_type` | mirror of `storeType` | for cross-brand filtering |
| `address.line1` | `address` | mixed English + script |
| `address.city`, `address.state` | (from regionList iteration) | only populated on service centres — retailers don't get state/city back from the API |
| `lat`, `lng` | `lat`, `lng` (string-coerced) | mostly populated; rare outliers (lat<1) on stale rows |
| `phone` | `tel` | comma-joined CSV preserved verbatim |
| `hours` | `time` → `{all: ...}`; `summerBusinessHours/winterBusinessHours` → `{summer, winter}` | mutually exclusive |
| `email`, `url` | n/a | API doesn't surface either |

`range` is search-query-relative — dropped by design (would mislead).

## Files in this repo

- **Extractor:** `scrapers/sales-distribution/vivo-india.ts`
- **Production runner:** `scrapers/sales-distribution/vivo-india-run.ts`
- **Fixtures:**
  - `__tests__/fixtures/vivo-india-retailers-all.json` (596 retailers)
  - `__tests__/fixtures/vivo-india-service-mumbai.json` (10 Mumbai service centres)
  - `__tests__/fixtures/vivo-india-region-list.json` (catalogue: 32 states × 546 cities)
- **Tests:** `__tests__/vivo-india-stores.test.ts` (17 tests)
- **Output:** `data/results/vivo-india-stores.jsonl` (1301 records)
- **YAML:** `targets/industries/sales-distribution/vivo-india.yaml`

## Pagination

No pagination — both endpoints return the full result set in one shot (per query). The retailer endpoint silently ignores filter narrowness; the service-centre endpoint REQUIRES city filtering and only returns hits for that city.

## Gotchas

- **YAML's documented locator URL (`vivo.com/in/service/center`) is a 404.** Canonical is `vivo.com/in/support/service-center`. We updated the YAML.
- **State-only or region-only queries to `/in/support/queryServiceCenterByCondition` silently return `data.store: null`.** No error code, no message — just nulls. You MUST drill down to cityCode.
- **Parameter-name mismatch between the two endpoints.** Retailers use `{regionId, stateId, cityId}`; service centres use `{regionCode, stateCode, cityCode}`. Same IDs, different keys. Mixing them yields the same silent-null response.
- **`regionList` catalogue ships inline in the HTML.** Lines 1692–3528 of the service-center page. We persist it as a fixture so the scraper does not re-scrape on every run. Re-capture if the catalogue changes (states added/renamed).
- **Retailer endpoint silently ignores narrower filters.** `regionId=1` returns the same 596 rows as `regionId=1&stateId=00221&cityId=0022117`. We use the broad query.
- **`tel` is CSV-joined when multiple numbers exist.** "7304433600,8097450892" — preserved verbatim per project convention; downstream consumers can split.
- **`summerBusinessHours` is usually identical to `winterBusinessHours` for Exclusive Stores.** They never split seasonally in observed data — but the schema supports it.
- **Some retailer rows have addresses in Devanagari / Marathi script.** Preserved verbatim; sortable by ASCII fields like `name`.
- **Service-centre `restTime` ≠ business closure.** It's the weekly day-off (e.g. `"Sun"`, `"Thu"`, `"No day off"`). Preserved in `_extra`.
- **No Wikipedia-style authoritative count is publishable.** Vivo (BBK) is privately held; the 70k+ touchpoint figure in press is the multi-brand wholesale channel, not the owned-Exclusive-Store estate. Scorecard sets `authoritativeCount: undefined`.
- **Sister brand OPPO uses a DIFFERENT stack.** Despite shared parent (BBK), OPPO India routes through `ind-sow-cms-app.oppo.com` GCSM; Vivo India keeps everything in-house at `www.vivo.com/in/*`. No reusable client; siblings cross-validate footprint magnitude only.

## Scorecard (live extraction, 2026-05-21)

| Dimension | Score | Notes |
|---|---|---|
| Completeness | 57 | no authoritative count (Vivo doesn't publish); all 6 expected cities present |
| Data quality | 82 | name/address 100%, lat/lng >75%, phone ~100% on service centres |
| Source authenticity | 91 | vivo.com self-served; known brand; not public; Wikipedia article exists |
| Freshness | 75 | first run — no prior snapshot to compare |
| **Overall** | **73 (medium)** | |
