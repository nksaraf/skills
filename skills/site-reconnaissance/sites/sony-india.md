# Sony India (Retail + Authorised Service)

**Domain:** sony.co.in (consumer-facing); **locator** at **locator.sony/en_IN/**
**Vertical:** Sales / distribution — consumer electronics (Sony Group)
**Last verified:** 2026-05-21
**Tier:** 2 (direct JSON API)
**Framework:** Standalone React SPA (`__dealer-locator-root`) hosted on `locator.sony`, behind AkamaiGHost, proxying AWS API Gateway behind it
**Protection:** **AkamaiGHost** (requires the full Chrome `sec-ch-ua*` + `Sec-Fetch-*` + `Accept-Encoding: gzip` fingerprint; no IP geo-gate)

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| Canonical locator (retail) | `locator.sony/en_IN/retailshops/` | https://locator.sony/en_IN/retailshops/ |
| Canonical locator (service) | `locator.sony/en_IN/servicecenters/` | https://locator.sony/en_IN/servicecenters/ |
| Locator API | `locator.sony/api/v1/dealers?orgLevel2=consumer&orgLevel3={smk\|customer_support}&minLat={}&maxLat={}&minLng={}&maxLng={}&limit={}&locale=en_IN` | see `scrapers/sales-distribution/sony-india.ts` |
| SPA static config | `locator.sony/api/v1/configs?profile=en_IN_retailshops` | `{defaultCenter, defaultZoom, addressSearchMaxZoomLevel}` only |
| Stale paths (DO NOT USE) | `sony.co.in/electronics/store-locator`, `sony.co.in/storefinder/`, `sony.co.in/microsite/retailshops/` | all **403 / 404** from any client — the yaml's original `sony.co.in/electronics/store-locator` is dead |

## Hydration payload

- **Location:** XHR JSON at `locator.sony/api/v1/dealers`
- **Envelope shape:** `{filters: {...}, items: [...]}`
- **Schema (key paths):**
  - `id` → Sony's internal UUID (stable PK across queries) — used for dedupe
  - `externalId` → locale-prefixed legacy id (e.g. `"en_IN_1016704"`)
  - `name.en`, `address.streetAddress.en`, `address.addressLocality.en`, `address.addressRegion.en`, `address.postalCode`
  - `latitude`, `longitude` (numbers)
  - `telephone: string[]` (service-centre rows only)
  - `openingHoursText.en` (free-text, e.g. `"10:30 am To 9:00 pm"`)
  - `types: string[]` (one-element token, see breakdown below)
  - `orgLevel1..orgLevel4` (`sony` / `consumer` / `smk|customer_support` / `ap`)
  - `categories: string[]` (iaCodes — surfacing PlayStation / camera / TV product lines stocked)
  - `countryCode: "IN"`, `translations: ["en"]`
  - `distance` (search-relative — **stripped during projection**, not preserved)

## Extraction strategy

**Tier 2 direct API in TWO calls total** (one per channel). The locator's React SPA loads two static configs + then issues one bounding-box `/v1/dealers` query per channel. We bypass the SPA and hit the API directly.

**Critical findings:**
1. **`orgLevel4` MUST be omitted.** The SPA encodes `orgLevel4=sid` for India (from the `regionLocales.consumer.sid = ["en_IN"]` map in the JS bundle), but **forwarding it to the API yields an empty `items` array**. The API ignores `sid`/regional codes — only `orgLevel2` (consumer) + `orgLevel3` (smk | customer_support) gate the dataset. Empirically confirmed: `orgLevel4=ap` works, `orgLevel4=sid` returns 0.
2. **One India-wide bounding box `{6,37,68,98}` returns the FULL channel** in a single call (no pagination — the `limit=10000` cap is well above the live total of ~250 per channel).
3. **AkamaiGHost gates everything on `locator.sony`** (and on every `sony.co.in/**` path). Default curl UA gets 403. The minimum passing fingerprint is the full Chrome `sec-ch-ua` triad + `Sec-Fetch-*` + `Accept-Encoding: gzip, deflate, br` — see `HEADERS` in `sony-india-run.ts`. No cookies, no warmup, no JS render needed once headers are right.
4. The locator API **is NOT IP-geo-gated** (unlike Samsung's service-centre AEM endpoint, which only returns India data to India-geolocated clients). Sony's `locator.sony` returns all India records from any IP.

Polite pacing: 400ms between the two channel calls.

## locationType breakdown (live, 2026-05-21)

| `types[0]` (raw) | LocationType | storeType label | Count |
|---|---|---|---|
| `sony_centers` | `store` | Sony Center | **112** |
| `alpha_flagship_store` | `store` | Alpha Flagship Store | **87** |
| `exclusive_stores` | `store` | Sony Exclusive | **16** |
| `sony_camera_lounge` | `store` | Sony Camera Lounge | **2** |
| `authorised_service_centre` | `service-centre` | Authorised Service Centre | **251** |
| **Total** | | | **468** |

**Network posture:** Post-smartphone-exit (Sony India retreated from mobile handsets ~2020), the retail footprint has contracted versus press-cited "300+ Sony Centers" peak. The 4-sub-type retail layer + dedicated service network is the current shape. **There is NO dedicated "Authorised Dealer" multi-brand channel in the locator** — that historical brochure category does not exist as a separate feed.

## Field inventory

| Field | Source path | Notes |
|---|---|---|
| `brand` | (constant) | "Sony India" |
| `storeId` | `id` | Sony's UUID — used for dedupe; falls back to `externalId` |
| `name` | `name.en` | Localised text block — `.en` picked, `.hi` etc. preserved in `_extra` |
| `locationType` | derived from `types[0]` | Token map — see breakdown above |
| `storeType` | derived from `types[0]` | Human label ("Sony Center" etc.) |
| `_extra.source_store_type` | `types[0]` | Raw token — for downstream filtering |
| `_extra.types` | `types` | Full array preserved (always one element in practice) |
| `_extra.externalId` | `externalId` | Legacy id — kept for cross-referencing older Sony data |
| `_extra.orgLevel1..4` | same | Hierarchy preserved for audit |
| `_extra.categories` | `categories` | Product-line iaCodes (TV, camera, PlayStation, audio…) |
| `_extra.note` | `note.en` | Service-centre-only — usually "Product Handled: All Categories" |
| `address.line1` | `address.streetAddress.en` | |
| `address.city` | `address.addressLocality.en` | |
| `address.state` | `address.addressRegion.en` | Full state name (e.g. "Chhattisgarh") |
| `address.postalCode` | `address.postalCode` | |
| `address.country` | `countryCode` | "IN" |
| `lat` / `lng` | `latitude` / `longitude` | Numbers — coords on ~100% of records |
| `phone` | `telephone[0]` | Array in source — first value taken; null for retail rows |
| `email` | n/a | Source does not surface emails |
| `url` | n/a | Source does not surface per-store URLs |
| `hours.text` | `openingHoursText.en` | Free-text only — Sony does not publish weekday granularity |
| `sourceUrl` | (channel API URL) | One of the two bounding-box URLs |
| `scrapedAt` | new Date() | ISO-8601 UTC |

Stripped (NOT preserved): `distance` (search-relative).

## Sample records

**Sony Center (Bilaspur):**
```json
{"brand":"Sony India","storeId":"dc782a69-1e04-40c9-b8fd-8dcb37011fcd","name":"Sony Center - Fairdeal","locationType":"store","storeType":"Sony Center","address":{"line1":"Srikanth Verma Marg Near Rama Magneto Mall","city":"Bilaspur","state":"Chhattisgarh","postalCode":"495001","country":"IN"},"lat":22.07324,"lng":82.152436,"hours":{"text":"10:30 am To 9:00 pm"}}
```

**Alpha Flagship Store (Durg):**
```json
{"brand":"Sony India","storeId":"c22bcb27-aabc-4658-95ac-7b1e0184d565","name":"Alpha Flagship Store - Laxmi studio & color lab","locationType":"store","storeType":"Alpha Flagship Store","address":{"city":"Durg","state":"Chhattisgarh","postalCode":"491001"},"lat":21.188623,"lng":81.278577}
```

**Authorised Service Centre (Bilaspur):**
```json
{"brand":"Sony India","storeId":"fa7af050-f56e-4538-8d23-6772aea86324","name":"Cyber Services","locationType":"service-centre","storeType":"Authorised Service Centre","phone":"+917752407466","address":{"line1":"H.No- 59, Rohini Vihar, Green Park Colony, Raipur Road","city":"Bilaspur","state":"Chhattisgarh","postalCode":"495004"},"lat":22.075557,"lng":82.142492}
```

## Gotchas

- **DO NOT touch `sony.co.in/**`.** Every consumer path (homepage, store-locator, microsite/*) is **403 Access Denied** on the Akamai edge. The canonical retail surface is `locator.sony` (different TLD, same Akamai config, but actually serves content with the right Chrome fingerprint).
- **DO NOT pass `orgLevel4`** in API requests. The SPA does, and it produces an empty response from the API. (The discrepancy is likely a server-side validator mismatch — `sid` is a SPA-only regional config that the API doesn't recognise as a content partition.)
- **`distance` is per-query-relative.** Strip it during projection or it will mislead downstream consumers (a record's "distance" depends entirely on the query center used to fetch it).
- **`hours` is free-text only.** Sony does not publish weekday-granular hours. We expose the raw string under `hours.text` rather than fabricating a weekday map.
- **No phone for retail rows.** The retail channel omits phone numbers entirely; only service centres carry `telephone[]`.

## Scorecard (live, 2026-05-21)

| Dimension | Score | Notes |
|---|---|---|
| Completeness | 57 | Brings the overall down — no source-claimed total, no authoritative reference. The 468 figure matches what the live locator surfaces. |
| Data quality | 94 | All records have UUIDs, coords, addresses; retail-channel rows lack phone (by source design — not an extraction gap). |
| Source authenticity | 99 | `locator.sony` is the Sony-Group-controlled canonical locator (linked from `sony.co.in/electronics/support/service-center-india` and `sony.co.in/microsite/retailshops/`). |
| Freshness | 75 | Captured this run — fresh. |
| **Overall** | **78** (band: **medium**) | |

## What we tested

- 20 tests in `__tests__/sony-india-stores.test.ts` (1663 expect calls):
  - fixture envelope shape lock (both channels)
  - URL builder structure (channel + bbox + locale)
  - **assertion that `orgLevel4` is NOT forwarded** (regression guard)
  - classifySonyLocationType across all five known tokens + channel-default fallback
  - labelSonyType for human labels
  - pickLocalised for the `.en` / first-locale extraction
  - All four retail sub-types appear in the fixture (sony_centers, alpha_flagship_store, exclusive_stores, sony_camera_lounge)
  - Service channel surfaces exactly ONE type (authorised_service_centre)
  - Retail rows ALL → `store`; service rows ALL → `service-centre`
  - storeType + `_extra.source_store_type` preservation
  - Stable storeId uniqueness across the captured set
  - India bounding-box coverage of lat/lng
  - `_extra` discipline — promoted fields stripped, source-only fields preserved
  - Deterministic snapshot for the first Sony Center (Bilaspur, id `dc782a69…`)
  - Deterministic snapshot for the first service centre (Cyber Services, Bilaspur, phone `+917752407466`)
  - API host + locator-page constants
  - INDIA_BBOX correctness
  - Optional: production JSONL breakdown (≥150 store + ≥150 service-centre)

## Files

- Extractor: `scrapers/sales-distribution/sony-india.ts`
- Run script: `scrapers/sales-distribution/sony-india-run.ts`
- Tests: `__tests__/sony-india-stores.test.ts`
- Fixtures:
  - `__tests__/fixtures/sony-india-retailshops-all.json` (217 retail rows, India-wide)
  - `__tests__/fixtures/sony-india-servicecenters-all.json` (251 service rows, India-wide)
- Production JSONL: `data/results/sony-india-stores.jsonl` (468 records)
- Snapshot: `data/snapshots/sales-distribution/sony-india/20260521T102321Z.jsonl`

## Re-recon triggers

- 403 from `locator.sony/api/v1/dealers` with default Chrome fingerprint → Akamai stiffened; needs JA3/TLS fingerprint review.
- `filters.items` empty with `orgLevel3=smk` → the bbox or `orgLevel4` rule changed; re-check by hitting `/api/v1/configs?profile=en_IN_retailshops` first.
- Total drops below ~400 → retail or service contraction event (e.g. further Sony India retreat).
- New `types[]` token appears → classification needs adding to `classifySonyLocationType`. The extractor falls back to the channel default but logs `_extra.source_store_type` so audit catches it.
