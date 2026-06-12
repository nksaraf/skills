# LG India

**Domain:** lg.com (locator at `/in/support/contact-us/locate-repair-center/`)
**Vertical:** Sales / distribution — electronics retail + service network
**Last verified:** 2026-05-21
**Tier:** 2 (direct JSON APIs — no JS execution required)
**Framework:** Next.js front-end on `www.lg.com`, AEM `ncms` JSON proxy backend
**Protection:** none — public `X-Lge-Application-Key` + `X-Lge-System-ID` headers are embedded in the HTML and required for the API; plain Chrome UA + `Origin: https://www.lg.com` + `Referer: <locator-page>` suffices

## Discovery path

Static probe of `/in` discovered the locator landing at
`/in/support/contact-us/locate-repair-center/`. The page is a Next.js shell
that loads two AEM `ncms` proxy endpoints. Both endpoints were extracted from
the bundled JS at `_ncms/goc/packages/support.f5677176d6c1b3aa1e9d.js`:

```
window.supportAPIConfigs.CSTAPIS.FINDSERVICECENTERLISTURL
  = "/ncms/api/v1/support/proxy/retrieveFindServiceCenterList?locale=IN"
```

The brand-shop endpoint surfaced separately on `/in/lg-best-shop/` page data
(`storeBrandShopDetailsApi: "/ncms/api/v1/proxy/brandshop-wtbDistributorData"`).

Both endpoints require the **public** `X-Lge-Application-Key` header (visible
as `window.AccessKey` in every page) and the static
`X-Lge-System-ID: LGCOM_AEM` header. These are NOT secret — they are served to
every visitor in plain HTML.

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| Service centre list (API) | `POST /ncms/api/v1/support/proxy/retrieveFindServiceCenterList?locale=IN` | body: `{searchLatitudeValue, searchLongitudeValue, ascDistance, state, "map-search":"Y", multiCountry:"Y"}` |
| Brand shop list (API) | `POST /ncms/api/v1/proxy/brandshop-wtbDistributorData?locale=IN` | body: `{lat, lng, distance, localeCode:"IN"}` |
| Service centre detail | `GET /in/support/locate-repair-center/{ascId}` | `/in/support/locate-repair-center/IN0471310001S` |
| Locator landing | `/in/support/contact-us/locate-repair-center/` | (Next.js shell) |
| Best Shop landing | `/in/lg-best-shop/` | (Next.js shell) |

## Two payload shapes

The locator surfaces **THREE** location types in the canonical schema, but
sourced from **TWO** APIs with different row shapes:

### A. Service-centre payload (`retrieveFindServiceCenterList`)

```json
{
  "retrieveFindServiceCenterList": {
    "data": [
      {
        "ascId": "IN0471310001S",
        "ascName": "LG Bangalore KH Road Service Center (LG Owned Center)",
        "stateName": "KA",              // ISO state code
        "cityName": "BANGALORE",
        "ascPostalCode": "560027",
        "ascPhoneNo": "8041709631",
        "address1": "93 TKN Mansion KH Road, Double Rd",
        "address2": "opp. Ksrtc Central Office",
        "address3": "Shanti Nagar, Bengaluru",
        "ascLatitudeValue": "12.9568364",
        "ascLongitudeValue": "77.59327859999999",
        "productCodes": "Air Care, Air Conditioners, ...",
        "operationTimeDesc": "09:00 Am to 06:00 Pm",
        "pscDiv": "ASC",                 // ASC = Authorized; DSC = LG-owned
        "corporationCode": "LGEIL",
        "ascUrl": "/in/support/locate-repair-center/IN0471310001S",
        "emailAddr": "sg.asokan@lge.com",
        ...
      }
    ]
  }
}
```

- **One nationwide call** anchored at India centroid (22.0, 79.0) with
  `ascDistance: 5000` returns ~640 unique rows — the server caps the result
  set; you cannot retrieve more than ~640 in any single call.
- A per-state sweep (32 state centroids) was added as redundancy but yields
  zero new rows over the nationwide call. Kept in the runner for documentation
  + future safety net.
- `pscDiv` discriminates `ASC` (Authorised, customer-billed) vs `DSC`
  (LG-owned Direct Service Center). Both map to `locationType = "service-centre"`
  — the ownership distinction is preserved in `_extra.pscDiv` and the
  human-readable label is mirrored in `_extra.source_store_type` + `storeType`.
- `distance` / `calcDistance` are SEARCH-RELATIVE — they vary per query and
  are dropped entirely from the canonical record.
- "." is a missing-value sentinel for `ascPhoneNo`, `ascFaxNo`, `distance`,
  etc. — the extractor coerces it to `null`.

### B. Brand-shop payload (`brandshop-wtbDistributorData`)

```json
{
  "code": 200,
  "wtbDistributorData": [
    {
      "wtbId": "DS05612612",
      "wtbName": "LG BEST SHOP  - AMIT ENTERPRISES",
      "wtbLatitudeValue": 28.5761687,
      "wtbLongitudeValue": 77.1961601,
      "address": "Shop No 70, Sarojini Nagar Market, Sarojini Nagar, New Delhi",
      "pinCode": "110023",
      "wtbPostalCode": "110023",
      "wtbPhoneNo": "9811804188",
      "wtbTime": "11:00 AM ~ 9:00 PM",
      "category": "Air Conditioners,Audio,Dishwashers,...",
      "superCategory": "Air Solutions,Home Appliances,...",
      "distributorTypeCode": "Best Shop",     // see classification below
      "stateName": "Delhi",
      "cityName": "New Delhi",
      "wtbCeo": "Amit Singhal",
      ...
    }
  ]
}
```

- The radius cap is much tighter (~250 stores per call vs nationwide for the
  service-centre API). A per-state sweep across 32 state centroids is required;
  it yields ~190 unique outlets after dedupe by `wtbId`.
- `distributorTypeCode` discriminates LG-branded retail (`Best Shop`,
  `LG Best Shop`, `LG Shoppe`, `LG Exclusive`, `BS`) → `store` vs
  multi-brand authorised retailers (`DEALER`) → `dealer`.
- `category` populates `segments`; `superCategory` populates `features`.
- `stateName` is the LONG state name ("Tamilnadu" with no space) — NOT the
  ISO code the service-centre API uses.

## Extraction strategy

**Tier 2.** Direct JSON POSTs, no warmup, no cookies, no JS execution. Pacing
at 300 ms between requests is more than enough — the endpoints have no rate
limit observed across 60+ requests in a single sweep.

Both endpoints are sequential per the production runner:

1. **Service centres:** one wide nationwide call (centroid 22.0, 79.0,
   `ascDistance: 5000`) for the bulk capture, then a 32-state redundancy
   sweep with `state=<ISO code>` + `ascDistance: 3000`.
2. **Brand shops:** 32-state sweep with `distance: 3000` per state. Dedupe
   by `wtbId`.

Mixed list deduplication uses `storeId` (= `ascId` for service centres,
`wtbId` for brand shops). No collisions observed.

## Field inventory

| Canonical field | Source A (`retrieveFindServiceCenterList`) | Source B (`brandshop-wtbDistributorData`) |
|---|---|---|
| `brand` | "LG India" (constant) | "LG India" (constant) |
| `storeId` | `ascId` | `wtbId` |
| `name` | `ascName` | `wtbName` (whitespace-collapsed) |
| `lat` | `ascLatitudeValue` (string → number) | `wtbLatitudeValue` (number) |
| `lng` | `ascLongitudeValue` (string → number) | `wtbLongitudeValue` (number) |
| `address.line1` | `address1` | `addressLine1Info` ?? `address` |
| `address.line2` | `address2` | (null) |
| `address.locality` | `address3` | (null) |
| `address.city` | `cityName` | `cityName` |
| `address.state` | `stateName` (ISO code: "KA", "DL") | `stateName` (long: "Karnataka") |
| `address.postalCode` | `ascPostalCode` | `wtbPostalCode` ?? `pinCode` |
| `address.country` | "IN" (from `countryCode`) | "IN" (from `countryName`) |
| `phone` | `ascPhoneNo` (`.` → null) | `wtbPhoneNo` |
| `email` | `emailAddr` (lowercased) | `wtbEmail` |
| `url` | `https://www.lg.com{ascUrl}` | `wtbUrl` (or null) |
| `hours.all` | `operationTimeDesc` | `wtbTime` |
| `locationType` | "service-centre" (always) | classify(`distributorTypeCode`) → "store" or "dealer" |
| `storeType` | `_extra.source_store_type` (mirror) | `distributorTypeCode` |
| `segments` | `productCodes` split on `,` | `category` split on `,` |
| `features` | `[]` | `superCategory` split on `,` |
| `_extra.source_store_type` | "LG Owned Service Center" / "Authorized Service Center" | `distributorTypeCode` |
| `_extra.pscDiv` | preserved (`ASC` \| `DSC`) | n/a |
| `_extra.distributorTypeCode` | n/a | preserved |

## Files in this repo

- **Extractor:** `scrapers/sales-distribution/lg-india.ts`
- **Production runner:** `scrapers/sales-distribution/lg-india-run.ts`
- **Tests:** `__tests__/lg-india-stores.test.ts` (19 tests)
- **Fixtures:**
  - `__tests__/fixtures/lg-india-service-centers-bangalore.json` (14 rows: 3 ASC + 11 DSC near Bengaluru)
  - `__tests__/fixtures/lg-india-brand-shops-delhi.json` (67 rows: Best Shop / LG Shoppe / LG Best Shop / LG Exclusive / BS / DEALER)
- **Production output:** `data/results/lg-india-stores.jsonl`

## Output (2026-05-21)

| locationType | Count |
|---|---|
| service-centre | 637 |
| store | 185 |
| dealer | 2 |
| **Total** | **824** |

**Scorecard:** overall=79 / band=medium (completeness=58, dq=95, auth=99, freshness=75).

## Pagination

None. Both APIs are non-paginated. The radius parameter is partially honoured
but server caps the result set at ~640 (service centres) and ~250 (brand shops)
regardless of `ascDistance` / `distance` value. A per-state sweep is the
canonical strategy for the brand-shop API.

## Gotchas

- The service-centre API's `pscDiv` does NOT match the row name. Row 0 of the
  Bengaluru capture is named "LG Bangalore KH Road Service Center (**LG Owned
  Center**)" but its source `pscDiv: ASC` — the API has its own internal
  ownership classification that doesn't always line up with the display name.
  Tests assert against `pscDiv`, not the name.
- Authoritative count discrepancy is large: LG India press materials cite
  ~25,000 service touchpoints; the public locator surfaces 824 (~3.3%). The
  remaining ~24,000 are unlisted DSP (Distribution Service Partner) in-store
  channel partners not exposed via the locator. The YAML records this as a
  known scope gap with `tolerance_pct: 95`.
- The brand-shop API requires lat + lng keyed as `lat`/`lng` (NOT
  `latitudeValue`/`longitudeValue` — that returns HTTP 200 with code 411
  "Missing required parameters:lat,lng").
- "." is a missing-value sentinel in the service-centre API (for distance,
  phone, fax, etc.). Treat as `null`.
- `distance` + `calcDistance` are query-anchor-relative and must NOT be
  promoted into the canonical schema (they would mislead downstream consumers).
- The state code in the service-centre API is the LG-internal ISO code
  ("KA", "DL", "MH", "TG") — NOT the long state name. The brand-shop API
  uses the long name. Both are preserved in `address.state` as the source
  reports them.
- Service-centre `state=<code>` filter alone (without coords) returns 0 rows
  — you MUST also send `searchLatitudeValue` + `searchLongitudeValue` even
  when filtering by state.
- A nationwide service-centre call at India centroid (22, 79) with
  `ascDistance: 5000` returns the entire 637-row dataset. The per-state sweep
  is kept in the runner for documentation only — adds zero new rows.

## Earlier investigation notes (resumed run)

Investigation initially probed `/in/storelocator`, `/in/store-locator`,
`/in/brand-shop`, `/in/best-shop` — all 404. The canonical locator path is
`/in/support/contact-us/locate-repair-center/`. The `FINDSERVICECENTER`
constant + `retrieveFindServiceCenterList` / `findServiceCenterPage` API
names were discovered by grepping the locator HTML for fetch-call patterns;
the AEM `ncms` proxy path was reconstructed from `window.supportAPIConfigs`
in the `support.<hash>.js` bundle.
