# Realme India (Offline Retail Stores)

**Domain:** realme.com (locator on `www.realme.com/in/support/store-address`)
**Vertical:** Sales / distribution — consumer electronics retail network
**Last verified:** 2026-05-22
**Tier:** 2 (direct JSON API)
**Framework:** Nuxt 3 storefront (`static2.realme.net/static/storeAddress.*.js`) calling Realme's own REST gateway
**Protection:** none on `api.realme.com`; plain Chrome UA + Referer to support page passes

## URL patterns

| Page type | Method + URL | Body / params |
|---|---|---|
| Locator page | GET `www.realme.com/in/support/store-address` | — |
| Store list (all in one) | GET `api.realme.com/in/offlineStore/page/list?page=1&limit=10000` | — |
| Province catalogue | GET `api.realme.com/in/offlineStore/province/list` | — |
| Province → city availability tree | GET `api.realme.com/in/offlineStore/list/available` | — |
| Service-centre (RESERVED) | GET `api.realme.com/in/service/center/list?provinceName=X&cityName=Y` | requires both params; ALWAYS returns 0 records in observed data |
| Store detail (unused) | GET `api.realme.com/in/offlineStore/detail?id=N` | — |

The Nuxt runtime config exposes `loginApi: "https://api.realme.com"` and `siteCode: "in"` — gateway base URL is `{loginApi}/{siteCode}` per the JS class `ft` in `storeAddress.a0a4d2b0.js`.

## Sibling-brand backend comparison vs OPPO

**Same parent, different stack.** The YAML hinted that Realme might share OPPO's GCSM platform. **It does not.**

| Aspect | OPPO India | Realme India |
|---|---|---|
| Gateway host | `ind-sow-cms-app.oppo.com` (GCSM) | `api.realme.com` (Realme proprietary) |
| API root | `/oppo-api/siteInfo/v1/*` | `/in/offlineStore/*` |
| Method | POST with JSON body | GET with query params |
| Pagination model | Province sweep (28 calls) | Single request returns all 131 |
| Discriminator | `siteType` string (`SERVICE_SITE` / `COLLECTION_POINT`) | `storeType` numeric (12-16) |
| Records returned | 596 — ALL service centres, 0 stores | 131 — ALL retail stores, 0 service centres |
| canonicalised `locationType` | `service-centre` (100%) | `store` (100%) |
| Frontend | Static SPA (jimu webpack) | Nuxt 3 |
| Stable PK | `siteCode` (`OISxxxxxxx`) | `id` (numeric) |
| Auth tokens visible | none | none |

**Why the divergence:** OPPO's GCSM is a global service-network platform shared with OPPO's other regions (the JS chunk name is `serviceCenterAsia`). Vivo's recon already showed a similar story — **same BBK parent, different stacks** (Vivo serves the same data straight from `vivo.com/in/*`). Realme follows Vivo's pattern: keep retail data on a brand-owned `api.realme.com` subdomain rather than route through any shared BBK infrastructure.

Net: OPPO is the OUTLIER in the BBK family in choosing the GCSM platform; Vivo and now Realme keep their locator data first-party. We confirmed this by probing for `ind-sow-cms-app.realme.com`, `sow-cms-app.realme.com`, `realme-sow-cms-app.realme.com` (all 000 / unreachable).

## Hydration payload

- **Location:** XHR JSON at `api.realme.com`
- **Envelope shape:** `{code: 200, msg: "success", data: {records: [...], recordTotal, pageNum, pageSize}, error: null, argsI18n: null}`
- **Store row key paths:**
  - `id` → Realme's stable PK (numeric)
  - `siteCode` → always `"in"` for India
  - `storeName` → free-text name
  - `storeType` → numeric (12 Coco, 13 Flagship, 14 Experience, 15 Smart, 16 Realme Store / Mini)
  - `state`, `city`, `pincode` → admin fields
  - `addressLine` → one-line concatenated street
  - `contact` → phone (rarely multiple, comma-separated)
  - `openTime` → "HH:MM-HH:MM" daily window
  - `closeTimeWeek` → comma-separated CLOSED weekdays (1=Sun…7=Sat); empty = open all 7
  - `status` → 1 = open
  - `longitude`, `latitude` → WGS-84 string
  - `serviceType` → always null in observed payload (reserved for future)

## Extraction strategy

Tier 2 direct API. **One GET** with `limit=10000` returns the full 131-record corpus. The endpoint accepts `page` + `limit` but the production sweep needs neither pagination nor province scoping. Headers required:

```
Accept: application/json,text/plain,*/*
Accept-Language: en-IN,en;q=0.9
Origin: https://www.realme.com
Referer: https://www.realme.com/in/support/store-address
User-Agent: <recent Chrome desktop>
```

No warmup, no cookies. Wall time ~250 ms.

## locationType breakdown (live sweep, 2026-05-22)

| storeType (raw) | Label | LocationType | Count |
|---|---|---|---|
| 12 | Coco Store | `store` | 1 |
| 13 | Flagship Store | `store` | 1 |
| 14 | Experience Store | `store` | 7 |
| 15 | Smart Store | `store` | 59 |
| 16 | Realme Store (incl. Mini) | `store` | 63 |
| (service-centre feed) | n/a | — | **0 (feed empty)** |
| **Total** | | | **131** |

**Why no service centres:** The `api.realme.com/in/service/center/list` endpoint validates `provinceName` + `cityName` but returns `{records: [], recordTotal: 0}` for every province/city we probed (Delhi, Mumbai, Bangalore, Chennai, Hyderabad). The "Authorized Service Center", "Exclusive Service Center", "Customer Service Center" labels on `/in/support/services` are **support channels** (WhatsApp, phone, email, chat) — not physical locations. Realme appears to dispatch repairs via the same retail store network or via a dealer back-end not exposed publicly.

This is materially different from sibling OPPO India whose entire 596-record locator is service-centre with zero retail stores. **The two brands have made opposite product choices about what to expose publicly** — Realme shows you where to BUY, OPPO shows you where to get serviced.

## State coverage (live sweep)

23 states / UTs covered. Top by store count:

| Rank | State | Stores |
|---|---|---|
| 1 | Bihar | 14 |
| 2 | Tamil Nadu | 13 |
| 3 | Andhra Pradesh / Chattisgarh | 11 each |
| 4 | West Bengal | 8 |

The footprint skews to Tier-2 markets — a deliberate Realme strategy (per company statements, the brand prioritises non-metro penetration through its sub-Rs-20,000 catalogue).

## Field inventory

| Field | Source path | Notes |
|---|---|---|
| `brand` | (constant) | "Realme India" |
| `storeId` | `id` | Numeric stable PK, stringified |
| `name` | `storeName` | All start with "realme " (mixed casing) |
| `status` | derived from `status` | 1 → open; else unknown |
| `address.line1` | `addressLine` | Sometimes prefixed "Address:" — preserved verbatim |
| `address.city` | `city` | E.g. "Coimbatore", "Kolkata" |
| `address.state` | `state` | "Tamil Nadu" — occasionally typo'd "Chattisgarh"/"kerela" |
| `address.postalCode` | `pincode` | 6-digit India PIN |
| `lat`, `lng` | `latitude`, `longitude` | String → number; out-of-bounds nulled + flagged |
| `phone` | `contact` | Mostly 10-digit, no country prefix |
| `hours` | `openTime` × `closeTimeWeek` | Projected to weekday → "HH:MM-HH:MM"/"closed" map |
| `locationType` | derived from `storeType` | always `store` for this feed |
| `storeType` | derived label | "Coco Store" / "Flagship Store" / "Experience Store" / "Smart Store" / "Realme Store" |
| `_extra.source_store_type` | `storeType` | Numeric mirror for downstream filtering |
| `_extra.coords_flagged` | (derived) | Set to "out-of-bounds" when source coords lie outside India (2/131 observed) |

## Sample record (Coimbatore Smart Store — out-of-bounds coords)

```json
{
  "brand": "Realme India",
  "storeId": "15420",
  "name": "realme smart store-R S Puram-Coimbatore -Tamil Nadu",
  "status": "open",
  "address": {
    "line1": "Address:New no 97 ( Old no 101-B), Gypsy corner, R. S Puram, DB road,641602",
    "city": "Coimbatore",
    "state": "Tamil Nadu",
    "postalCode": "641002",
    "country": "IN"
  },
  "lat": null,
  "lng": null,
  "phone": "7604823330",
  "hours": {"sunday": "closed", "monday": "10:00-22:00", ..., "saturday": "closed"},
  "locationType": "store",
  "storeType": "Smart Store",
  "_extra": {"source_store_type": 15, "isRest": false, "coords_flagged": "out-of-bounds", "source_lat_raw": 8.89404, "source_lng_raw": 44.76783, ...}
}
```

## Sample record (Kolkata Smart Store — well-formed)

```json
{
  "brand": "Realme India",
  "storeId": "16114",
  "name": "Realme Smart Store -Emall- Kolkata",
  "address": {"city": "Kolkata", "state": "West Bengal", "postalCode": "700072", "country": "IN"},
  "lat": 22.5722,
  "lng": 88.3557,
  "phone": "9903866000",
  "hours": {"sunday": "11:00-20:30", "monday": "11:00-20:30", ..., "saturday": "11:00-20:30"},
  "locationType": "store",
  "storeType": "Smart Store"
}
```

## Scorecard (2026-05-22 live sweep)

- **Overall:** 76 / 100 (medium)
- Completeness 52, Data Quality 99, Source Authenticity 91, Freshness 75
- The medium band is dragged by `authoritativeCount: null` (no published Realme India retail count). Source-claim cross-check is informational — the API ships `recordTotal=131` which matches what we extract (zero loss).
- Data quality 99 reflects ~99% phone + coord + hours + pincode coverage. The 1% miss is the 2 out-of-bounds coordinate records that we null + flag rather than swap.

## Gotchas

1. **`page` + `limit`, NOT `pageNo` + `pageSize`**: the JS calls the endpoint with `getOfflineStoreApi({page:1, limit:1e4})`. Other params silently no-op. Recon initially probed `pageNo`/`pageSize` and got stuck on 20 records (default page size) before I read the JS bundle.
2. **No `stateId` filter**: passing `stateId=IN-AP` returns the same full corpus — the endpoint ignores it. State filtering happens client-side.
3. **OPPO's GCSM is NOT shared** with Realme. Don't waste recon time probing `*-sow-cms-app.realme.com` subdomains (they don't exist). Realme runs its own `api.realme.com` REST gateway.
4. **`closeTimeWeek` is 1-indexed CLOSED days (1=Sun..7=Sat)** — the opposite polarity of the typical "open on these days". Empty string means open 7 days. One Bihar store had `closeTimeWeek="7"` + `isRest: true` simultaneously — that's a closed-on-Saturday store currently on its rest day.
5. **2/131 records have garbage coordinates** (Coimbatore at lat 8.9/lng 44.8 — actually somewhere in the Indian Ocean; Durg at lat 4.3/lng 50.7 — also off the map). We null them + retain raw values in `_extra` rather than auto-swap (no clean way to disambiguate honest data-entry errors from systematic axis swaps).
6. **State name typos at source**: "Chattisgarh" (should be "Chhattisgarh"), "kerela" (should be "Kerala"), "TamilNadu" (no space). Preserved verbatim — fixing them would mask source quality issues from the scorecard.
7. **Service-centre feed exists but is empty**. The `api.realme.com/in/service/center/list` endpoint validates `provinceName` + `cityName` but returns 0 records for every probed combo. Reserved hook (`classifyByServiceType`) kept in the extractor for forward-compat.

## Files in this repo

- Extractor: `scrapers/sales-distribution/realme-india.ts`
- Runner: `scrapers/sales-distribution/realme-india-run.ts`
- Tests: `__tests__/realme-india-stores.test.ts` (16 tests, ~795 expects)
- Fixtures:
  - `__tests__/fixtures/realme-india-stores.json` (full 131-store envelope)
  - `__tests__/fixtures/realme-india-provinces.json` (province catalogue)
  - `__tests__/fixtures/realme-india-available.json` (province → city tree)
- JSONL: `data/results/realme-india-stores.jsonl` (131 records)
- Summary: `data/results/realme-india-stores-summary.json`
- Snapshot: `data/snapshots/sales-distribution/realme-india/<runAt>.jsonl`
