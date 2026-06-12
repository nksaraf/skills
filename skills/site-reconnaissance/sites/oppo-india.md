# OPPO India (Authorised Service Centres)

**Domain:** oppo.com (in) — locator on `support.oppo.com/in/service-center/`
**Vertical:** Sales / distribution — consumer electronics service network
**Last verified:** 2026-05-21
**Tier:** 2 (direct JSON API)
**Framework:** SPA (`jimu/oppo-global-service` webpack bundle, `serviceCenterAsia` chunk) calling OPPO's GCSM gateway
**Protection:** none on `ind-sow-cms-app.oppo.com`; plain Chrome UA + Referer to support page passes the gateway

## URL patterns

| Page type | Method + URL | Body |
|---|---|---|
| Locator page | GET `support.oppo.com/in/service-center/` | — |
| Administrative divisions (state/district/city tree) | POST `ind-sow-cms-app.oppo.com/oppo-api/siteInfo/v1/administrationDivisions` | `{"regionIsoCode2":"IN"}` |
| Sites by area (province-scoped) | POST `ind-sow-cms-app.oppo.com/oppo-api/siteInfo/v1/querySiteInfoByArea` | `{"regionIsoCode2":"IN","provinceCode":"IND0201"}` |
| Business-time detail | POST `…/siteInfo/v1/queryBusinessTime` | (per-site; not needed — hours are inline in the area response) |

The gateway base URL `https://ind-sow-cms-app.oppo.com/oppo-api` is wired into the page HTML as `GCSMAPIPATH`. The chunk name `serviceCenterAsia` indicates OPPO routes Indian users through the Asia variant of their service-centre UI.

## Hydration payload

- **Location:** XHR JSON at `ind-sow-cms-app.oppo.com`
- **Envelope shape:** `{code: "1", msg: "ok", data: [...]}`
- **Site row key paths:**
  - `siteCode` → OPPO's stable PK (e.g. `OIS019011`)
  - `siteName` → free-text site name
  - `siteType` → `"SERVICE_SITE"` | `"COLLECTION_POINT"` (only two tokens observed for India)
  - `operationStatus` → `"NORMAL"` | suspended states
  - `latitude`, `longitude` → numeric WGS-84
  - `detailAddress` → one-line concatenated street
  - `sitePhoneNumber` → +91 format
  - `operationTime.workTimes[]` → array of `{workDays: ["MONDAY", …], workHour: {start, end}, lunchBreakHour: {…}}`
  - `provinceName`, `provinceCode`, `cityName`, `cityCode`, `districtName`, `districtCode` — admin hierarchy (note: OPPO's `districtName` corresponds to what Indians call the *city*; `cityName` corresponds to the *district*)

## Extraction strategy

Tier 2 direct API. The XHR endpoint is open to plain Chrome UA + `Referer: https://support.oppo.com/in/service-center/` and `Origin: https://support.oppo.com` headers. No cookies, no warmup, no JS render needed.

Discovery flow:

1. **Fetch the division tree** once — returns 28 state/UT province codes (`INDxxxxx`).
2. **For each province** POST `querySiteInfoByArea` with `{regionIsoCode2: "IN", provinceCode}`.
3. **Project + dedupe** by `siteCode`. Province responses are non-overlapping (OPPO partitions by state), so 596 raw rows → 596 unique sites with 0 duplicates.

Polite pacing: 350ms between province requests. Total wall time ~12s.

## locationType breakdown (live sweep, 2026-05-21)

| siteType (raw) | LocationType | storeType label | Count |
|---|---|---|---|
| `SERVICE_SITE` | `service-centre` | Authorized Service Centre | 596 |
| `COLLECTION_POINT` | `service-centre` | Collection Point | 0 (observed) |
| (brand / experience stores) | — | — | **NOT in this feed** |
| **Total** | | | **596** |

**Why no brand stores:** OPPO India sells direct online at `oppo.com/in/store/` and through multi-brand retailers — they do NOT operate a branded retail estate comparable to Samsung Experience Stores. The corporate Asia locator surfaces only service touchpoints. Sister brands in BBK (Vivo, Realme, OnePlus) follow the same model — verified during recon by examining the `serviceCenterAsia` chunk's allowed siteType tokens.

## State coverage (live sweep)

All 28 Indian state/UT provinces returned at least 1 site. Top states by count:

| Rank | State | Sites |
|---|---|---|
| 1 | Uttar Pradesh | 70 |
| 2 | Maharashtra | 56 |
| 3 | Gujarat | 41 |
| 4 | Tamil Nadu | 40 |
| 5 | Madhya Pradesh / Rajasthan | 39 each |
| 6 | Karnataka | 35 |
| 7 | Bihar | 33 |

Smallest: Chandigarh (1), Meghalaya (1), Sikkim (1), Goa (2), Puducherry (2), Tripura (2).

## Field inventory

| Field | Source path | Notes |
|---|---|---|
| `brand` | (constant) | "OPPO India" |
| `storeId` | `siteCode` | Stable PK; format `OIS\d+` |
| `name` | `siteName` | All start with "OPPO Authorized Service Center -" |
| `status` | derived from `operationStatus` | `NORMAL`→open; other→unknown |
| `address.line1` | `detailAddress` | One-line concatenated street + city + state + pin |
| `address.city` | `districtName` (OPPO's district = Indian city) | Use districtName, not cityName |
| `address.state` | `provinceName` | e.g. "Andhra Pradesh" |
| `address.postalCode` | `postCode` | Mostly null in payload — pin embedded in `detailAddress` |
| `address.country` | (derived) | "IN" |
| `lat`, `lng` | `latitude`, `longitude` | Numeric (no string parsing needed) |
| `phone` | `sitePhoneNumber` | Mostly `+91` prefixed; some without |
| `hours` | `operationTime.workTimes[]` | Projected to weekday→`HH:MM-HH:MM` map |
| `locationType` | derived from `siteType` | `SERVICE_SITE`/`COLLECTION_POINT` → `service-centre` |
| `storeType` | derived label | "Authorized Service Centre" / "Collection Point" |
| `_extra.source_store_type` | `siteType` | Mirror for downstream filtering |

## Gotchas

1. **Naming inversion**: OPPO's `cityName` is the *district* (administrative division above the city), and `districtName` is the actual *city*. We extract `city` from `districtName`.
2. **Two webpack chunks**: `serviceCenter` (Western) and `serviceCenterAsia` — India uses the Asia chunk. The Asia chunk advertises `COLLECTION_POINT` as a recognised siteType but no Indian province returns one in production today. Forward-compat preserved.
3. **No global count endpoint**: there's no `siteCount` field. Scorecard's `authoritativeCount` is null and the source-claim check is a warn.
4. **No postCode**: most rows have `postCode: null`; the postal code is embedded inside `detailAddress` (regex extraction not implemented here — `_extra` retains the source row).
5. **Region routing**: there's a `GCSMAPIPATH_OE` variable for an "overseas" variant. Plain `GCSMAPIPATH` is the IN-direct path; we hard-code `ind-sow-cms-app.oppo.com` (the resolved value for IN).
6. **Sister-brand siblings**: Vivo, Realme, OnePlus all run on the same BBK GCSM platform. The serviceCenterAsia chunk is shared — a Vivo/Realme/OnePlus India extractor would need only the corresponding `ind-sow-cms-app` subdomain (or whatever variant their site references).

## Sample record (Vijayawada AP service centre)

```json
{
  "brand": "OPPO India",
  "storeId": "OIS019011",
  "name": "OPPO Authorized Service Center - Vijayawada",
  "status": "open",
  "address": {
    "line1": "R K Plaza, Door No 39-3-2, Behind Anjaneya Jewellery, Beside CMR Shopping Mall, M G Road, Labbipet, Venkateswara Puram, Vijayawada, Andhra Pradesh-520010",
    "city": "Vijayawada",
    "state": "Andhra Pradesh",
    "postalCode": null,
    "country": "IN"
  },
  "lat": 16.50204,
  "lng": 80.641515,
  "phone": "+91 8977642325",
  "hours": {"monday": "10:00-18:00", "tuesday": "10:00-18:00", "wednesday": "10:00-18:00", "thursday": "10:00-18:00", "friday": "10:00-18:00", "saturday": "10:00-18:00"},
  "locationType": "service-centre",
  "storeType": "Authorized Service Centre",
  "_extra": {"siteScore": "0.00", "operationStatus": "NORMAL", "source_store_type": "SERVICE_SITE", ...}
}
```

## Scorecard (2026-05-21 live sweep)

- **Overall:** 76 / 100 (medium)
- Completeness 55, Data Quality 94, Source Authenticity 91, Freshness 75
- The medium band is driven by `authoritativeCount: null` (no public OPPO India service-centre count to cross-check against) and a mid-tier completeness signal. Data quality and source authenticity are both excellent — OPPO-owned subdomain, ~99% of records have full coordinates + phone + hours.

## Files

- Extractor: `scrapers/sales-distribution/oppo-india.ts`
- Runner: `scrapers/sales-distribution/oppo-india-run.ts`
- Tests: `__tests__/oppo-india-stores.test.ts` (15 tests, 176 expects)
- Fixtures: `__tests__/fixtures/oppo-india-andhra-pradesh.json`, `__tests__/fixtures/oppo-india-administrative-divisions.json`
- JSONL: `data/results/oppo-india-stores.jsonl` (596 records)
- Summary: `data/results/oppo-india-stores-summary.json`
