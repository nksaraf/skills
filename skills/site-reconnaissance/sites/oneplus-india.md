# OnePlus India (Experience Stores + Authorized Service Centres)

**Domain:** `oneplus.in` (retail) + `service.oneplus.com` (service) + `ind-sow-cms.oneplus.com` (BBK API)
**Vertical:** Sales / distribution — consumer electronics (premium Android)
**Last verified:** 2026-05-22
**Tier:** mixed — **Tier 2** for service-centre API + **Tier 1** for retail (HTML)
**Framework:** SPA (`jimu/global-store` webpack bundle, sibling of OPPO's `jimu/oppo-global-service`) for service-centre locator; Vue SPA with inline `var _store_data` for the experience-and-retail page
**Protection:** none on either source — plain Chrome UA passes both endpoints; no warmup, no cookies

## Sibling-brand backend comparison (BBK Electronics family)

| Sibling | Service-centre API | Retail / Experience |
|---|---|---|
| **OPPO** | `ind-sow-cms-app.oppo.com/oppo-api/siteInfo/v1/...` | none (online-only direct sales) |
| **OnePlus** | `ind-sow-cms.oneplus.com/oppo-api/siteInfo/v1/...` ← **shared BBK stack** | `oneplus.in/experience-and-retail` inline `_store_data` ← unique to OnePlus |
| **Vivo** | `vivo.com/in/support/queryServiceCenterByCondition` ← **in-house, NOT BBK** | `vivo.com/in/retailers/queryByCondition` ← in-house |

**Headline finding.** OnePlus's service-centre locator is **the same OPPO stack** — same `oppo-api` URL path, same envelope shape `{code, msg, data: SiteRaw[]}`, same `siteCode` / `siteName` / `siteType` / `operationTime` schema, same OPPO `districtName=city` naming inversion. Only the gateway host differs (`ind-sow-cms.oneplus.com` vs OPPO's `ind-sow-cms-app.oppo.com`). This confirms the OPPO-recon hypothesis: BBK runs ONE GCSM platform across sub-brands.

Vivo, despite the same parent corp, **does NOT** share this backend — it serves both retail and service directly from `vivo.com/in/*`.

OnePlus's retail Experience Stores are NOT on the BBK service-centre API — they live in a completely separate Vue SPA on `www.oneplus.in/experience-and-retail` as an inline JS literal.

## URL patterns

| Page type | Method + URL | Body |
|---|---|---|
| Service locator UI | GET `service.oneplus.com/in/service-center` | — |
| Service: administrative divisions | POST `ind-sow-cms.oneplus.com/oppo-api/siteInfo/v1/administrationDivisions` | `{"regionIsoCode2":"IN"}` |
| Service: sites by province | POST `ind-sow-cms.oneplus.com/oppo-api/siteInfo/v1/querySiteInfoByArea` | `{"regionIsoCode2":"IN","provinceCode":"INDxxxxx"}` |
| Retail locator (Experience & Partner stores) | GET `www.oneplus.in/experience-and-retail` | — (inline `var _store_data`) |

**Yaml's `oneplus.in/service/store-locator` is a 404** — the actual store locator lives at `service.oneplus.com/in/service-center` (BBK pattern, mirror of `support.oppo.com/in/service-center/`).

## Hydration payload

### Source A — Service centres (XHR JSON, same shape as OPPO)

- **Envelope:** `{code:"1", msg:"ok", data: SiteRaw[]}`
- **Site row keys:**
  - `siteCode` → stable PK (`IPS\d+`, e.g. `IPS001026002`)
  - `siteName` → free-text label (always contains "OnePlus Authorized" / "OnePlus Authorized Multi-brand" / "OnePlus Exclusive Service Centre")
  - `siteType` → token (see below)
  - `operationStatus` → "NORMAL" / suspended states
  - `latitude` / `longitude` → numeric
  - `detailAddress` → one-line concatenated street
  - `sitePhoneNumber` → +91 or local-format
  - `operationTime.workTimes[]` → array of `{workDays, workHour, lunchBreakHour}`
  - `provinceName`/`districtName`/`cityName` — admin hierarchy (OPPO's naming inversion preserved)

#### siteType tokens observed (national sweep, 905 records)

| siteType | LocationType | storeType label | Count | Notes |
|---|---|---|---|---|
| `SERVICE_SITE` | `service-centre` | Authorized Service Centre | 620 | OnePlus-only authorized centres |
| `OUTSOURCE_POINT` | `service-centre` | Authorized Multi-brand Service Centre | 275 | Third-party multi-brand outlets that also service OnePlus |
| `SALE_AND_SERVICE_SITE` | `service-centre` | Exclusive Service Centre | 10 | OnePlus Exclusive Service Centres (also sell — distinct from Experience Stores) |
| **Total** | | | **905** | |

### Source B — Retail Experience & Partner stores (inline JS)

- **Location:** `var _store_data = {...}` JS object literal at HTML offset ~130 KB.
- **Format:** JS object literal with single-quoted keys/values (NOT JSON). Contains: JS line comments (Chinese maintainer notes `// 这两个link有问题`), trailing commas, embedded non-breaking spaces (`U+00A0`), and one DQ string with an apostrophe inside (`No'40` in a Reliance Digital map URL). The repo's `singleQuoteLiteralToJson` converter handles all of these.
- **Six buckets** (138 records total):

| Bucket | Source key | type code | LocationType | storeType label | Count |
|---|---|---|---|---|---|
| OnePlus Experience Zone (owned) | `exclusive` | 1 | `store` | OnePlus Experience Store | 38 |
| Authorized Experience Store (partner-operated, OnePlus-branded) | `authorized` | 2 | `store` | OnePlus Authorized Experience Store | 33 |
| OnePlus Kiosk | `kiosks` | 4 | `store` | OnePlus Kiosk | 18 |
| Croma (multi-brand electronics retail) | `croma` | 3 | `dealer` | Croma | 21 |
| Reliance Digital (multi-brand electronics retail) | `reliance-digital` | 5 | `dealer` | Reliance Digital | 21 |
| OnePlus Mini Store | `mini-store` | 6 | `store` | OnePlus Mini Store | 7 |
| **Total** | | | | | **138** |

**Per-row schema:** `{state, city, title, type, pincode, address, tel, time, points, link}` — `points` is `"lat,lng[,zoom]"` string.

## Extraction strategy

Hybrid: **Tier 2** for service centres (the BBK API is fast, complete, and returns a province-partitioned dataset in 29 calls), **Tier 1** for retail (one GET on the experience page yields all 138 records). One run sweeps both sources concurrently and merges.

Pacing: 350ms between province requests for source A. Total wall time ~12s (29 provinces + 1 HTML fetch).

Dedupe: by `storeId` — service-centre IDs are `IPS\d+`, retail IDs are synthetic `oneplus-<bucket>-NNN`. No overlap observed; dedupe is defensive.

## Field inventory

| Field | Source path | Source A (svc) | Source B (retail) |
|---|---|---|---|
| `brand` | constant | "OnePlus India" | "OnePlus India" |
| `storeId` | `siteCode` / synthetic | `IPS\d+` | `oneplus-<bucket>-NNN` |
| `name` | `siteName` / `title \|\| bucketLabel` | ✓ | ✓ |
| `status` | derived | `NORMAL`→open | always "open" |
| `address.line1` | `detailAddress` / `address` | ✓ | ✓ |
| `address.city` | `districtName` / `city` | ✓ (OPPO inversion) | ✓ |
| `address.state` | `provinceName` / `state` | ✓ | ✓ (but **kiosks bucket sometimes puts city in state field** — preserved verbatim) |
| `address.postalCode` | `postCode` / `pincode` | mostly null | ✓ |
| `lat`/`lng` | numeric | ✓ | parsed from `points` string |
| `phone` | `sitePhoneNumber` / `tel` | ✓ | ✓ |
| `url` | — | null | Google Maps `link` |
| `hours` | `operationTime.workTimes[]` | ✓ projected | parsed from "Mon-Sun HH:MMam-HH:MMpm" |
| `locationType` | derived | `service-centre` | `store` (owned) / `dealer` (Croma + Reliance) |
| `storeType` | derived | tokenised label | bucket label |
| `_extra.source_store_type` | `siteType` / bucket | mirror | mirror |
| `_extra.source_bucket` | — | — | bucket key |
| `_extra.source_type_code` | — | — | integer `type` |

## Gotchas

1. **Yaml's `/service/store-locator` is a 404.** Canonical service-centre UI is at `service.oneplus.com/in/service-center` (mirrors OPPO's `support.oppo.com/in/service-center/`). Yaml has been updated.
2. **`oneplus.in/experience-and-retail#/` is a SPA hash route** — the HTML at the bare path (no hash) is the same and contains the full `_store_data` literal; no need to drive the SPA. Plain GET.
3. **`_store_data` is JS, not JSON.** Single-quoted, trailing commas, `//` line comments, NBSPs between tokens, one stray apostrophe inside a double-quoted URL (`No'40` in a Reliance Digital map link). The `singleQuoteLiteralToJson` converter normalises all of this and is covered by three unit tests.
4. **`pointsArr` is added by Vue at runtime — NOT in the HTML.** Only the comma-separated `points` string is in the raw HTML. The extractor parses `points` directly; `pointsArr` support is forward-compat.
5. **`kiosks` bucket has a source-side data-quality bug**: `state` sometimes holds the city name (e.g. `{state:"Amritsar", city:"Amritsar"}` or `{state:"AURANGABAD",...}`). Preserved verbatim — fix at projection time would mask the upstream bug.
6. **OPPO naming inversion preserved**: OnePlus's `districtName` is the Indian-sense city; `cityName` is the district. The extractor reads `city` from `districtName` (matching the OPPO extractor).
7. **`SALE_AND_SERVICE_SITE` is NOT an Experience Store** — these are 10 "OnePlus Exclusive Service Centres" that also sell devices but the BBK locator classifies them as service. The retail Experience Stores are in Source B with NO overlap.
8. **Croma + Reliance Digital are `dealer`s, not `store`s.** They're multi-brand electronics chains stocking OnePlus alongside competitors. Calling them stores would inflate the brand's owned-retail footprint. OnePlus-branded outlets (`exclusive`, `authorized`, `kiosks`, `mini-store`) are stores.
9. **No authoritative count.** OnePlus does not publish a service-centre or retail-store count. The YAML's `~50 Experience Stores + several hundred Authorised Service Centres` proves to be roughly correct (96 store + 42 dealer + 905 service-centre = 1,043 total).
10. **BBK family pattern locked.** OnePlus service-centre API ⊇ OPPO API. Future Realme-India recon should start by probing `ind-sow-cms.realme.com` or similar — same `oppo-api` URL path is very likely.

## Sample records

### Service centre (OUTSOURCE_POINT)

```json
{
  "brand": "OnePlus India",
  "storeId": "IPS001026002",
  "name": "OnePlus Authorized Multi-brand Service Center (Visakhapatnam LK MAX MOBILES)",
  "status": "open",
  "address": {
    "line1": "58-1-278/1 , 1st Floor,NAD Junction beside rasoi restaurant , Gopalapatnam Road , visakhapatnam- 530027",
    "city": "Visakhapatnam",
    "state": "Andhra Pradesh",
    "country": "IN"
  },
  "lat": 17.742766,
  "lng": 83.234623,
  "phone": "9505356688",
  "hours": {"monday": "10:00-19:00", ..., "saturday": "10:00-19:00"},
  "locationType": "service-centre",
  "storeType": "Authorized Multi-brand Service Centre",
  "_extra": {"source_store_type": "OUTSOURCE_POINT", ...}
}
```

### Retail (Experience Zone)

```json
{
  "brand": "OnePlus India",
  "storeId": "oneplus-exclusive-001",
  "name": "OnePlus Experience Zone",
  "status": "open",
  "address": {
    "line1": "OnePlus Boulevard, UN 101/105, 201/204, 301, Forum Rex Walk WN 176, Municipal, 169, Brigade Rd, Bengaluru, Karnataka 560001",
    "city": "Bangalore",
    "state": "Karnataka",
    "postalCode": "560001",
    "country": "IN"
  },
  "lat": 12.972543,
  "lng": 77.6067563,
  "phone": "7760511527",
  "url": "https://maps.app.goo.gl/LoyCiY6Cn7Np78FQ6",
  "hours": {"monday": "11:00-21:30", ..., "sunday": "11:00-21:30"},
  "locationType": "store",
  "storeType": "OnePlus Experience Store",
  "_extra": {"source_bucket": "exclusive", "source_type_code": 1, ...}
}
```

## Scorecard (2026-05-22 live sweep)

- **Overall:** 76 / 100 (medium)
- Completeness 55, Data Quality 94, Source Authenticity 91, Freshness 75
- The medium band is driven by `authoritativeCount: null` (no public OnePlus India retail-or-service count to cross-check against). Data quality and source authenticity are both excellent — OnePlus-owned subdomain for service + OnePlus.in for retail, ~99% of records have full coordinates + phone + hours.

## Files

- Extractor: `scrapers/sales-distribution/oneplus-india.ts`
- Runner: `scrapers/sales-distribution/oneplus-india-run.ts`
- Recon helper: `scrapers/sales-distribution/oneplus-india-recon.mjs` (Playwright DOM probe — captured `window._store_data`)
- Tests: `__tests__/oneplus-india-stores.test.ts` (27 tests, 444 expects)
- Fixtures:
  - `__tests__/fixtures/oneplus-india-andhra-pradesh.json` (37 service sites)
  - `__tests__/fixtures/oneplus-india-administrative-divisions.json` (29 IN provinces)
  - `__tests__/fixtures/oneplus-india-experience-and-retail.html` (full SPA page with inline literal)
  - `__tests__/fixtures/oneplus-india-experience-stores.json` (parsed `_store_data` dump for recon)
- JSONL: `data/results/oneplus-india-stores.jsonl` (1,043 records)
- Summary: `data/results/oneplus-india-stores-summary.json`
- Snapshot: `data/snapshots/sales-distribution/oneplus-india/<runAt>.jsonl`
