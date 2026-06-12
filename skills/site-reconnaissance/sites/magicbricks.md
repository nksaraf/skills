# MagicBricks

**Domain:** magicbricks.com
**Vertical:** India real estate
**Last verified:** 2026-05-21
**Tier:** 3 (Playwright)
**Framework:** Custom React SSR — uses `window.SERVER_PRELOADED_STATE_` (not a known framework convention)
**Protection:** mild — no observed blocks during recon

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| SRP | `/property-for-rent/{type}?proptype={Type}&cityName={City}` | `/property-for-rent/commercial-real-estate?proptype=Office-Space&cityName=New-Delhi` |
| Detail | Path with `-pdpid-` token (hex-encoded MB project ID) | `/mayank-apartments-dwarka-new-delhi-pdpid-4d4235303334303235` |

## Hydration payload

- **Location:** `window.SERVER_PRELOADED_STATE_` (inline script, ~485 KB on SRP, ~1.1 MB on detail)
- **Source name from `extractAllHydrationPayloads`:** `window.SERVER_PRELOADED_STATE_ (generic)`

### SRP schema
```
root → searchResult[]
  searchResult[i]:
    encId, pBuc, possStatusD, pmtSource, cntvl, ctArea, url,
    projectSocietyLink    → slug containing pdpid-<hex> for detail URL
    seoURL                → fallback URL path
    propertyTitle         → listing title
    lmtDName              → locality name
    scdloc                → "locality, city"
    price                 → numeric INR
    priceD                → display string ("35,000")
    carpetArea            → sqft (string)
    caSqFt                → covered area (number)
    carpAreaUnit          → area unit ("Sq-ft")
```

### Detail-page schema (PROJECT page architecture)
`pdpid-` URLs resolve to **project/society pages**, not individual unit pages.
The initial hydration exposes two main branches:

```
root:
  isCsr, userInfoData, searchBean, searchAdditionalDataBean
  projectDetailData
    prjMobileBean
      infoBean
        psmLatitude/psmLongitude  → project centroid lat/lng (string, numeric-parseable)
        hasLatLon                 → "Y"/"N"
        minRentPrice/maxRentPrice → project rent price range (often null)
        price                     → project price display string
      amenitiesList[]
        amenityName               → "Power Back Up", "Lift", "Gymnasium", ...
        amenityImg, isExists, amenityId
      detailBean
        numOfFloors               → total floors in building
        bedroom                   → BHK type
      ratingBean
        overallRt, prjAmenityRt, securityRt, maintenanceRt
      contact
        contactType, contactButton, userType
      rentCountProperty           → count of rental listings in project
    contactDetails
      userType                    → "agent" / "owner"
    title, seoPropertyType
  projectPageSeoStaticData
    bhkDetailsDTO
      bhkProjectDetailsMap
        ALL
          groupedRentResult[]      ← rental unit listings (cg="r") — PREFERRED for rental data
            bookingAmtExact        → security deposit (numeric string, e.g. "70000")
            possStatusD            → availability ("Immediately" or date string)
            bachelor               → tenant pref: "Y"=bachelor OK, "D"=family only, "N"=not specified
            nriPref                → "Y" if NRI tenants preferred
            (all other fields same as groupedResult entries below)
          groupedResult[]          ← individual unit listings (typically sale, sometimes rent)
            encId, enc_Id          → base64 encoded IDs
            ltcoordGeo             → "lat,lng" per-listing coordinate (EXACT, not centroid)
            pmtLat, pmtLong        → project centroid (number)
            price                  → numeric INR
            priceD                 → display string ("3.35 Cr")
            cg                     → "b" (buy/sale) or "r" (rent)
            carpetArea             → sqft (string)
            caSqFt                 → covered area (number)
            coveredArea, ca        → covered area strings
            carpAreaUnit           → "Sq-ft"
            bedroomD               → "4" (bedrooms)
            bathD                  → "3" (bathrooms)
            balconiesD             → "2"
            floorNo                → floor number string
            floors                 → total floors string
            floorD                 → "3 of 9 Floor" (display)
            facingD                → "East", "North - East", etc.
            furnishedD             → "Semi-Furnished", "Furnished", "Unfurnished"
            furnished              → numeric enum (11901=Unfurnished, 11902=Semi-Furnished, 11903=Furnished)
            maintenanceCharges     → "3000" (string)
            maintenanceD           → "Monthly"
            acD                    → property age ("15 to 20 years", "Above 20 years")
            userType               → "Agent" / "Owner"
            contName               → contact person name
            oname                  → owner/company name
            companyname            → company name
            caCompNameD            → company display name
            ctVerifd               → "Y"/"N" (verified badge)
            propTypeD              → "Multistorey Apartment"
            possStatusD            → "Ready to Move"
            transactionTypeD       → "Resale" / "New Property"
            transType              → "Sale" / "Rent"
            prjname                → project name
            psmid                  → MB project ID (e.g., "5034025")
            lmtDName               → locality name
            locSeoName             → more specific locality
            ctName                 → city name
            psmAdd                 → full address text
            landmarkDetails[]      → ["19210|Dwarka Sector-6 & 10 Market Bus Stop", ...]
            luxAmenitiesD          → comma-separated amenity names
            amenities              → space-separated amenity IDs
            imgCt                  → image count
            dtldesc / plgdtldesc   → agent description
            auto_desc              → auto-generated description
            seoDesc                → SEO description
            pfAgCg                 → "s" (sale) - usually present for sale listings
```

### Architecture: two listing arrays in the initial SSR
The initial hydration contains **two** listing arrays in `bhkProjectDetailsMap.ALL`:
- `groupedRentResult[]` — rental listings (`cg: "r"`). Contains all rental-specific fields.
- `groupedResult[]` — sale listings (`cg: "b"`). Contains structural/price data for sale.

**The `#rentTab` click does NOT trigger an XHR.** It only toggles pre-rendered DOM. Rental data is fully available in the initial SSR payload under `groupedRentResult[]`.

**Extraction strategy (as of 2026-05-21 v2):**
- `bestRentalListing` = `groupedRentResult[0]` — drives `securityDeposit`, `availableFrom`, `tenantPreferred`
- `bestListing` = `bestRentalListing ?? groupedResult[0]` — drives all structural fields (price, area, geo, floor, poster)
- `ltcoordGeo` = per-listing exact coordinate (NOT centroid) — use this when present
- `psmLatitude/psmLongitude` = project centroid fallback
- Project amenities from `amenitiesList[]` are available regardless
- `bachelor` field decoding: "Y" → Bachelor, "D" → Family, "N" → not specified; `nriPref: "Y"` adds NRI

## Extraction strategy

Tier 3 + stealth + Indian locale. No special pacing seen during recon — likely tolerates faster crawling than 99acres.

## Files in this repo

- **Recon demo:** `scrapers/recon-magicbricks.ts`
- **SRP extractor:** `scrapers/magicbricks.ts` + `scrapers/magicbricks-test.ts`
- **Detail extractor:** `scrapers/magicbricks-detail.ts` (production) + `scrapers/magicbricks-detail-test.ts` (extractor + smoke test)
- **Fixture:** `__tests__/fixtures/magicbricks-detail-delhi.html` (Mayank Apartments, Dwarka, ~1.1 MB)
- **Tests:** `__tests__/magicbricks.test.ts` (25 tests), `__tests__/magicbricks-detail.test.ts` (26 tests)

## Pagination

URL param (`?page=N`) for SRP, confirmed working up to 100 pages per category/city bucket (cap raised from 30 → 100 on 2026-05-22).

## proptype values — VERIFIED 2026-05-22

MB uses **numeric IDs** internally (`searchBean.propertyType`). The `proptype` URL param is parsed server-side and silently falls back to the generic residential set (`["10002","10003","10021","10022","10020"]`) if it doesn't match a known string. This produces DUPLICATE records identical to builder-floor-apartment data with wrong category labels — undetectable without checking the numeric IDs.

Verification method: inspect `window.SERVER_PRELOADED_STATE_.searchBean.propertyType` in the SSR payload after each URL navigation.

| Category slug | Correct `proptype=` value | Internal ID(s) | Notes |
|---|---|---|---|
| `multistorey-apartment` | `Multistorey-Apartment` | `10002,10003,10021,10022,10020` | Includes penthouse, SA, serviced appt |
| `residential-house` | `Residential-House` | — | |
| `villa` | `Villa` | — | |
| `builder-floor-apartment` | `Builder-Floor-Apartment` | `10002,10003,10021,10022,10020` | Same IDs as multistorey — MB buckets them |
| `studio-apartment` | `Studio-Apartment` | — | |
| `office-space` | `Office-Space` | `10007,10018` | Also works as `Commercial-Office-Space` |
| `commercial-shop` | `Commercial-Shop` | `10008,10009` | NOT `Commercial-Shops` (pluralised = fallback) |
| `commercial-showroom` | `Commercial-Showroom` | `10008,10009` | NOT `Commercial-Showrooms`; same IDs as shop |
| `warehouse-godown` | `Warehouse-Godown` | `10011` | NOT `Warehouses`; `Warehouse/Godown` fails URL encoding |
| `industrial-building` | `Industrial-Building` | `10013` | NOT `Industrial-Buildings` |
| `industrial-shed` | `Industrial-Shed` | `10014` | |

### Silently-fallback proptypes (DO NOT USE)

The following values look valid but MB silently falls back to the generic residential set:

| Wrong value | Observed behaviour |
|---|---|
| `Commercial-Shops` | Falls back → 900 rows duplicating builder-floor data |
| `Commercial-Showrooms` | Same |
| `Warehouses` | Same |
| `Industrial-Buildings` | Same |
| `1-RK-Flat` | Same — MB has no standalone proptype for 1-RK |

### Unsupported categories

- **`1-rk-flat`**: MB does not expose a proptype for 1-RK units. The BHK filter is client-side only; the SSR payload ignores `bhk=1RK` in the URL. Removed from DEFAULT_CATEGORIES.
- **`paying-guest`**: `proptype=Paying-Guest` correctly routes to the PG-specific page (`propertyType: ["10050","10053"]`) but that page renders PG listings client-side — `searchResult` is `[]` in the SSR payload. The `mb-srp__card` selector finds 0 elements. Would need a separate PG extractor. Removed from DEFAULT_CATEGORIES.

## Gotchas

- **`SERVER_PRELOADED_STATE_` is NOT in any framework's canonical list.** Our `extractAllHydrationPayloads` catches it only via the generic `window.<IDENT>` matcher. Without that matcher, the entire payload is invisible. This is the canonical example of why the generic matcher exists.
- **Label/value DOM is sibling-spans.** No `<table>`, no `<dl>`, no `Label: Value` colon strings. The audit's `siblingPairs` heuristic (`[class*='label']` + adjacent value element) catches MagicBricks-style facts.
- **Listing cards** use `[class*='mb-srp__card']` and `[class*='mb-srp__card__summary']` — not `factsRow`/`factsTable`/`tupleNew`.
- **Status badges visible:** `NEW`, `New`, `Free` — caught by the tightened badge matcher.
- **`pdpid-` is the detail-page token** — added to default `DEFAULT_DETAIL_LINK_PATTERNS_SRC` in `visualAudit.ts`.
- **Project-page architecture:** `pdpid-` detail URLs resolve to project/society pages, not individual unit pages. The initial SSR hydration contains BOTH `groupedResult[]` (sale) and `groupedRentResult[]` (rental) — no lazy API call is needed.
- **`ltcoordGeo` vs `pmtLat/pmtLong`:** `ltcoordGeo` is a per-listing "lat,lng" string (EXACT coordinate). `pmtLat/pmtLong` are the project centroid. Always prefer `ltcoordGeo`.
- **`pdpid` hex decoding:** `pdpid-4d4235303334303235` = hex → `MB5034025` (the MB project ID). Simple hex decode.
- **Silent fallback proptype bug (2026-05-22):** Invalid proptype strings produce generic residential results with the wrong category label. Detection: check `searchBean.propertyType` numeric IDs — generic fallback always gives `["10002","10003","10021","10022","10020"]`. `categoryToProptype()` + unit tests guard against regression.

## Recon output (2026-05-19 SRP, 2026-05-21 detail)

- SRP: 1189 schema leaves, 99 label-value pairs, 3 badges, 4 detail links
- Detail: 1692 schema leaves, `projectDetailData` + `projectPageSeoStaticData.bhkDetailsDTO`
- Detail individual listing: ~192 schema leaves per `groupedResult[]` entry; `groupedRentResult[]` entries share same schema plus `bookingAmtExact`, `bachelor`, `nriPref`

## Detail-page production output (2026-05-21, v2 with rental fields)

- **Input:** 1560 Delhi rental listings from SRP (297 unique pdpid project pages)
- **LIMIT=5 smoke:** 5/5 with `securityDeposit`, `availableFrom`, `tenantPreferred` all populated
- **Lat/lng accuracy:** per-listing `ltcoordGeo` from `groupedRentResult[0]`
- **Sample record (Mayank Apartments, 2026-05-21 smoke):**
  ```json
  {
    "source": "magicbricks.com",
    "listingId": "pdpid-4d4235303334303235",
    "priceCadence": "month",
    "priceValueINR": 35000,
    "securityDeposit": 70000,
    "availableFrom": "Immediately",
    "tenantPreferred": ["Family"],
    "furnishing": "Semi-Furnished",
    "localityName": "Dwarka",
    "cityName": "New Delhi",
    "propertyType": "Multistorey Apartment",
    "bedrooms": 4,
    "bathrooms": 3,
    "amenities": ["Power Back Up", "Lift", "Security", "Swimming Pool", "Gymnasium", ...],
    "posterType": "Agent",
    "projectName": "Mayank Apartments",
    "projectId": "MB5034025"
  }
  ```

## Rental-specific fields — fully captured (2026-05-21)

`securityDeposit`, `availableFrom`, and `tenantPreferred` are now populated from `groupedRentResult[]` in the initial SSR payload. No XHR interception needed. The `#rentTab` click does NOT trigger a network request.

**Field mapping:**
| DetailRecord field | SSR source | Example |
|---|---|---|
| `securityDeposit` | `groupedRentResult[0].bookingAmtExact` | `"70000"` → 70000 |
| `availableFrom` | `groupedRentResult[0].possStatusD` | `"Immediately"` |
| `tenantPreferred` | `bachelor` + `nriPref` | `"D"+"N"` → `["Family"]` |

## What's still missing

- Full URL pattern coverage for commercial categories (only residential tested in detail enrichment)
- PG (Paying Guest) extractor — needs a separate page architecture to handle client-rendered PG listings
- 1-RK scraping — no valid MB proptype; would need a client-side JS approach or intercepting XHR after filter click
