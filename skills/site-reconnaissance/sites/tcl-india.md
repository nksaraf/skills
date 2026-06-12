# TCL India (Authorised Service)

**Domain:** tcl.com (consumer-facing); **locator data API** at **cseleinind.tcl.com:8081**
**Vertical:** Sales / distribution — consumer electronics (TCL Technology, HK-listed)
**Last verified:** 2026-05-22
**Tier:** 2 (direct JSON API — ABP / ASP.NET CRM backend)
**Framework:** ABP-style ASP.NET CRM ("cselein in d" — TCL India's customer-service ELE backend) behind a Microsoft-Azure-fronted IP
**Protection:** none observed; default Chrome UA + Origin + Referer pass

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| Canonical locator page | `tcl.com/in/en/support/maps` | https://www.tcl.com/in/en/support/maps |
| Locator config (embedded in page) | `<div id="maps-root" property='{"crmApi":"https://cseleinind.tcl.com:8081", ...}'>` | (parsed from SSR HTML) |
| CRM Country list | `/api/Country/GetList?searchText={text}` | https://cseleinind.tcl.com:8081/api/Country/GetList?searchText=India |
| CRM Province list | `/api/Province/GetList?country={countryId}` | …?country=1d8fbdce-5e45-e911-a816-000d3af04c74 |
| CRM City list | `/api/City/GetList?country={cid}&province={pid}` | …?country={IN}&province={DELHI} |
| Find Nearest ASP | `/api/Account/FindNearestASPService?country={cid}&state={pid}&city={cityId}` | required path — no broader query supported |
| **Stale paths (DO NOT USE)** | `tcl.com/in/en/service/find-service-center` (yaml's), `…/store-locator`, `…/where-to-buy`, `…/dealer-locator` | all **404** |
| Sibling locators (worldwide) | `tcl.com/{id\|ph\|pk\|tr}/{lang}/support/maps` | same widget; needs the country's own CRM root (not `cseleinind`) |

## Hydration payload

- **Location:** the locator widget reads a JSON `property` attribute on `<div id="maps-root">` from the SSR'd `/in/en/support/maps` page, then calls four XHRs against `cseleinind.tcl.com:8081`.
- **Response envelope:** standard ABP wrap — `{result, targetUrl, success, error, unAuthorizedRequest, __abp}`.
- **Service-centre row shape (`FindNearestASPService.result[i]`):**
  - `id` → stable CRM GUID (primary key used for dedupe)
  - `name`
  - `phone` (free-text, 10-digit Indian mobile in practice)
  - `addr` → comma-joined free-text: `"<line>,<locality>,<state>,<postal>,India"`
  - `latitude` / `longitude` — numeric (truncated to 2 decimals; many ASPs share coords because the CRM only resolves city-level centroids)

## Extraction strategy

**Tier 2 multi-step sweep.** The CRM rejects requests without a `city` GUID — there is NO state-only or country-wide "list all" endpoint. The widget always:
1. Picks India's `countryId` once.
2. Picks a `provinceId` (state) from a dropdown.
3. Picks a `cityId` (sub-locality) from a dropdown.
4. Calls `FindNearestASPService(country, state, city)` which returns ≤ 2-5 of the NEAREST authorised service partners.

Since each city call returns only the K-nearest, we must sweep across cities to surface every ASP. Empirically (Goa, 30 cities sampled out of 270): the dataset converges in fewer than ~30 city calls per state — most cities surface the same nearest neighbours. The production script caps at **50 cities per province** (8-way parallelism), which suffices to surface every ASP in dense-but-small states. We sweep **all 38 real provinces** (40 raw - 2 sandbox/test entries — `IN-TEST`, `Unique Tech`).

**Critical findings:**
1. **No retail / dealer surface exists in any TCL India locator.** Only Authorised Service Partners. Consumer-facing `tcl.com/in/en/support` exposes only a hotline (1800-102-0622) and the online service-request form. All YAML-implied paths (`/service/find-service-center`, `/store-locator`, `/dealer-locator`, `/where-to-buy`) are 404.
2. **Country/Province/City params are GUIDs, not names.** `country=India` returns a validation error; only the GUID `1d8fbdce-5e45-e911-a816-000d3af04c74` works. Province param name is **lowercase `country`** + lowercase `state` + lowercase `city` for FindNearest, but `country` for Province/City. Capitalisation mismatch (`Country=`/`CountryId=`) silently returns `[]`.
3. **Backend is SLOW.** Each `FindNearestASPService` call took 10-14s in our run (likely a synthetic-distance MS-SQL spatial query against a large customer-care table). We use 8-way parallel `Promise.all` batches — TCL's ABP backend accepts the concurrency cleanly (no 429/503). City and Province lists are fast (<1s).
4. **`(lat, lng)` truncated to 2 decimals** in the source. Many ASPs share the exact same coords — they reflect the city centroid, not the shop. Useful for "approximate map placement" only.
5. **Province list contains test/dupes** — `IN-TEST` and `Unique Tech` are CRM sandbox entries; `MAHARASHTRA` appears twice (legitimate duplicate in CRM); `CHATTISGARH` and `Chhattisgarh` both exist as separate provinces (we keep both — they index different ASP cohorts).

## locationType

All records → `locationType: "service-centre"`, `storeType: "Authorised Service Centre"`. TCL India has no public retail locator.

## Field inventory

| Field | Source path | Notes |
|---|---|---|
| `brand` | (constant) | "TCL India" |
| `storeId` | `id` | CRM GUID — primary key for dedupe |
| `name` | `name` | |
| `locationType` | (constant) | "service-centre" |
| `storeType` | (constant) | "Authorised Service Centre" |
| `phone` | `phone` | 10-digit Indian mobile (no country code) |
| `address.line1` | parsed from `addr` (head) | comma-split, everything BEFORE the last 4 tokens |
| `address.locality` / `.city` | parsed from `addr` (-4 token) | also used as `address.city` (no separate city in source) |
| `address.state` | provinceName from sweep | source-recorded state; matches the province loop |
| `address.postalCode` | parsed from `addr` (5-6 digit token) | |
| `address.country` | parsed | "IN" |
| `lat` / `lng` | `latitude` / `longitude` | truncated to 2dp; (0,0) → null |
| `email` | n/a | not surfaced |
| `url` | n/a | not surfaced |
| `hours` | n/a | not surfaced |
| `_extra.sourceProvince` | provinceName | audit trail of which sweep surfaced this row |

## Sample record

```json
{"brand":"TCL India","storeId":"d577031b-2b6c-f011-aac4-6045bda59e28","name":"Time wheel","phone":"9130824459","locationType":"service-centre","storeType":"Authorised Service Centre","address":{"line1":"Chimbel","city":"Chimbel","state":"GOA","postalCode":"403006","country":"IN"},"lat":15.49,"lng":73.87}
```

## Files

- Extractor: `scrapers/sales-distribution/tcl-india.ts`
- Run script: `scrapers/sales-distribution/tcl-india-run.ts`
- Tests: `__tests__/tcl-india-stores.test.ts` (17 tests)
- Fixtures:
  - `__tests__/fixtures/tcl-india-country.json` — India countryId lookup
  - `__tests__/fixtures/tcl-india-provinces.json` — 40 raw provinces (38 real + 2 sandbox)
  - `__tests__/fixtures/tcl-india-cities-goa.json` — 270 Goa cities
  - `__tests__/fixtures/tcl-india-asp-goa-sample.json` — 30 captured FindNearest envelopes
  - `__tests__/fixtures/tcl-india-asp-goa-unique.json` — 8 unique Goa ASPs after dedupe
- Production JSONL: `data/results/tcl-india-stores.jsonl`
- Snapshot: `data/snapshots/sales-distribution/TCL India/<stamp>.jsonl`

## Gotchas

- **Do NOT trust the YAML's locator URL.** `tcl.com/in/en/service/find-service-center` is 404. The real canonical is `tcl.com/in/en/support/maps`. Update the YAML to reflect this (we've left `notes` as a hint).
- **Do NOT use country/state names for the API.** GUIDs only. The string-name variants silently return `[]` (success: true), which is a debugging trap.
- **Do NOT filter cities by name.** TCL's "cities" are actually sub-localities — Delhi alone has 551, and most are real but tiny administrative bins. The list IS exhaustive within each state.
- **Beware backend slowness.** Single calls take 10-15s; parallelism is required for any India-wide sweep to finish in under an hour. 8-way `Promise.all` works; 16-way may exceed the backend's tolerance — keep at 8.
- **(lat, lng) is city-centroid, not shop-precise.** Many ASPs share identical coords. Don't use these for "nearest shop on map" UX — use postal code instead.
- **No retail surface.** Future re-recon should probe `cseleinind` for other ABP controllers (DistributorService, ExclusiveStoreService etc) — none surfaced in `maps.min.js` today.

## Re-recon triggers

- `Country/GetList?searchText=India` returns `[]` → CRM was re-pointed; reread `<div id="maps-root">` `property` JSON.
- `FindNearestASPService` 401 / 403 → ABP started enforcing auth; check for `X-Lge-…` style header injection in newer `maps.min.js` bundles.
- `extracted < 100` (national sweep) → backend lost records; alert via snapshot count-delta.
- New ABP controller appears in `static-obg.tcl.com/.../crm/maps.min.js` → a sibling channel (dealer / store) became surfaceable.
