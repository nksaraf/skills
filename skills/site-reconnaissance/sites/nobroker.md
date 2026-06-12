# NoBroker

**Domain:** nobroker.in
**Vertical:** India real estate (owner-direct, no-brokerage residential + commercial rentals)
**Last verified:** 2026-05-21
**Tier:** 3 (stealth Playwright browser — needed to execute React SPA + get full SSR state)
**Framework:** Custom React/Redux SPA with server-side rendering via Express
**Protection:** mild (nginx; Imperva/Cloudflare NOT observed; requires homepage warmup for cookie seeding)

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| SRP (canonical city — SSR data) | `/flats-for-rent-in-{city}-{city}?pageNo={n}` | `/flats-for-rent-in-delhi-delhi?pageNo=2` |
| PG SRP (canonical city — SSR data) | `/pg-in-{city}-{city}?pageNo={n}` | `/pg-in-bangalore-bangalore` |
| SRP (locality-filtered, thin shell) | `/flats-for-rent-in-{locality}-{city}` | `/flats-for-rent-in-rohini-delhi` |
| Detail | `/property/{slug}/{id}/detail` | `/property/2-bhk-apartment-for-rent-in-rohini.../8a9f.../detail` |
| BHK-filtered (thin SPA shell only) | `/2-bhk-flats-for-rent-in-{city}` | (**NOT usable for scraping**) |

**Critical:** the `{city}-{city}` duplicated-suffix pattern is the canonical SSR form. Single-suffix URLs (`/flats-for-rent-in-delhi`) load a thin SPA shell with no server-rendered data.

## City coverage

NoBroker operates in **8 Indian cities only**. Kolkata and Ahmedabad return HTTP 410.

| City slug | Total flats listings (2026-05-21) |
|---|---|
| delhi | 17,682 |
| bangalore | 58,076 |
| hyderabad | 41,888 |
| pune | 27,444 |
| noida | 10,730 |
| chennai | 15,104 |
| mumbai | 4,758 |
| gurgaon | 10,730 |

## Hydration payload

- **Location:** Inline `<script>` tag: `nb.appState = { ... }`
- **Source name:** `nb.appState`
- **Top-level key path:** `nb.appState.rentListPage.listPageProperties[]`
- **Schema (key paths per listing object):**

| Field | JSON key | Type | Notes |
|---|---|---|---|
| Listing ID | `id` | string | 32-char hex UUID |
| Title | `title` | string | Full human title |
| BHK type | `typeDesc` | string | "1 RK", "2 BHK", etc. |
| Locality | `locality` | string | Locality name (may include street address) |
| Address | `address` | string | Full street address (usually more complete than `locality`) |
| City | `city` | string | Always lowercase city slug (e.g. "delhi") |
| Monthly rent | `rent` | number | INR/month (integer) |
| Formatted price | `formattedPrice` | string | "22,000" (no ₹ prefix) |
| Deposit | `deposit` | number | INR (integer) |
| Area | `propertySize` | number | sqft (no unit suffix) |
| Floor | `floor` | number | Current floor number |
| Total floors | `totalFloor` | number | Building total floors |
| Bathrooms | `bathroom` | number | Count |
| Furnishing | `furnishing` | string | FULLY_FURNISHED / SEMI_FURNISHED / NOT_FURNISHED |
| Detail URL | `detailUrl` | string | Relative path starting with `/property/...` |

- **Pagination params:** `nb.appState.rentListPage.listPageOtherParams`
  - `count`: properties on this page (25)
  - `total_count`: total across all pages (e.g. 17682 for Delhi)
  - `searchToken`: null for canonical pages (pagination purely via `?pageNo=N`)

## Detail-page payload

- **Location:** Same inline `<script>` tag: `nb.appState = { ... }` (also available as live Redux store in browser)
- **Key path:** `nb.appState.propertyDetails.detailsData`
- **Notes:** The detail page populates a different Redux reducer (`propertyDetails`, not `rentListPage`). The full `detailsData` object is embedded in the SSR'd HTML — parse it the same way as the SRP state.

| Field | JSON key | Type | Notes |
|---|---|---|---|
| Listing ID | `id` | string | 32-char hex UUID |
| Latitude | `latitude` | number | Per-listing (or locality-level when `accurateLocation=false`) |
| Longitude | `longitude` | number | Per-listing (or locality-level) |
| Accurate location | `accurateLocation` | boolean | true = building-precise; false = locality-level pin |
| Floor | `floor` | number | Current floor |
| Total floors | `totalFloor` | number | Building total |
| Area (sqft) | `propertySize` | number | sqft (no unit suffix) |
| Bathrooms | `bathroom` | number | Count |
| Balconies | `balconies` | number | Count |
| Furnishing | `furnishing` | string | FULLY_FURNISHED / SEMI_FURNISHED / NOT_FURNISHED |
| Furnishing short desc | `furnishingDesc` | string | "Full" / "Semi" |
| Facing | `facing` | string | "S", "N", "NE", etc. |
| Property age (years) | `propertyAge` | number | Age in years |
| Rent | `rent` | number | INR/month |
| Deposit | `deposit` | number | INR |
| Maintenance amount | `maintenanceAmount` | number or null | INR/month (null when not set) |
| Negotiable | `negotiable` | boolean | |
| Available from | `availableFrom` | number | Unix epoch ms |
| Lease type | `leaseTypeNew` | string[] | ANYONE / FAMILY / BACHELOR_MALE / BACHELOR_FEMALE / COMPANY |
| Amenities (map) | `amenitiesMap` | object | Record of code → boolean (preferred over `amenities` string) |
| Amenities (string) | `amenities` | string | JSON-encoded flags (fallback if `amenitiesMap` absent) |
| Verified | `listingVerified` | boolean | |
| Owner description | `ownerDes` | string | User-written portion of description |
| Combined description | `description` | string | Owner + NoBroker system-generated text |
| Photos | `photos` | array | Array of photo objects; `photos.length` = image count |
| Locality | `locality` | string | Locality/address string |
| Sub-locality | `nbLocality` | string | Human-readable locality name |
| City | `city` | string | Always lowercase slug |
| Address | `address` | string | Full street address |
| BHK type code | `type` | string | BHK1 / BHK2 / BHK3 / BHK4 / RK1 |
| BHK type human | `typeDesc` | string | "1 BHK", "2 BHK", etc. |
| Building type | `buildingType` | string | IH = Independent House, AP = Apartment |
| Property code | `propertyCode` | string | "NB-xxxxxxx" (null for some listings) |
| Nearby places | `seoDescription` | string | JSON array of `{ name, placeType, latitude, longitude }` |
| Water supply | `waterSupply` | string | "CORP_BORE", "BOREWELL", etc. |
| Power backup | `powerBackup` | string | "None", "Partial", "Full" |

### Detail-page production output (2026-05-21 Delhi sweep)

- **Input:** 750 Delhi rental listings from SRP
- **Lat/lng coverage:** 100% (all 750 expected to have coordinates)
- **latLngSource breakdown (sample of 20):** 14 "exact", 3 "locality-centroid"
- **Lat/lng verification:** Per-listing, NOT shared centroids. Two Rohini listings in same locality differ by ~0.029° (~3.2 km). Confirmed per-property geocoding.
- **Key finding:** `accurateLocation=true` = building-precise pin; `false` = locality-level but still unique per listing

Sample record (LIMIT=20 output, listing 8a9f8e03...):
```json
{
  "source": "nobroker.in",
  "listingId": "8a9f8e038fb42506018fb4964f2f3a8f",
  "latitude": 28.65003475452938,
  "longitude": 77.164014,
  "latLngSource": "exact",
  "propertyType": "2 BHK",
  "priceValueINR": 32000,
  "securityDeposit": 32000,
  "furnishing": "Furnished",
  "amenities": ["Lift", "Air Conditioning", "Security", "Power Backup"],
  "posterType": "Owner",
  "verified": null,
  "imageCount": 6
}
```

## Extraction strategy

Tier 3 (stealth Playwright) required because:
1. The canonical SSR URL only renders full data when the browser presents a proper `nbccc` + `nbDevice` cookie pair from the homepage visit (warmup)
2. Without warmup, the page may return thin or incomplete state
3. Curl-based raw HTTP retrieval works for the canonical URL (verified) but may degrade over time as NoBroker tightens protection

**Pacing:**
- PAGE_DELAY: 2,500–5,500ms between pages
- 8,000–20,000ms between category×city buckets

**Anti-bot signals observed:** None (nginx, no Akamai/Cloudflare headers). However, a homepage warmup is strongly recommended to seed the `nbccc` session cookie.

## Field inventory

| Field | Extractable | Source | Notes |
|---|---|---|---|
| Listing ID | ✅ | `id` | 32-char hex |
| Title | ✅ | `title` | Full human title |
| Location text | ✅ | `address` (preferred) or `locality` | Address is usually the more complete string |
| City | ✅ | `city` | Always lowercase slug |
| Monthly rent (INR) | ✅ | `rent` | Integer — no Lac/Cr in API, always absolute INR |
| Deposit | ✅ | `deposit` | Integer INR |
| Area (sqft) | ✅ | `propertySize` | Integer sqft |
| Detail URL | ✅ | `detailUrl` (relative) | Prefix `https://www.nobroker.in` |
| BHK type | ✅ | `typeDesc` / `type` | 1 RK, 1 BHK, 2 BHK, 3 BHK, 4 BHK |
| Lat/Lon | ✅ | `latitude`, `longitude` | float |
| Furnishing | ✅ | `furnishing` | enum |
| Owner name | ✅ | `ownerName` | string |
| Floor | ✅ | `floor`, `totalFloor` | integers |
| Sponsored flag | ✅ | `sponsored` | boolean |
| Verified flag | ✅ | `listingVerified` | boolean |

## Files in this repo

### SRP (search-result page)

- **Extractor:** `scrapers/nobroker-test.ts` (smoke), `scrapers/nobroker.ts` (production)
- **Fixture:** `__tests__/fixtures/nobroker-srp-delhi.html`
- **Tests:** `__tests__/nobroker.test.ts` (18 tests, all passing)
- **Recon demo:** `scrapers/recon-nobroker.ts`

### Detail page

- **Extractor:** `scrapers/nobroker-detail-test.ts` (pure extractor + smoke), `scrapers/nobroker-detail.ts` (production enricher)
- **Fixture:** `__tests__/fixtures/nobroker-detail-delhi.html` (listing 8a9f9384... — 2BHK Paschim Vihar, Delhi)
- **Tests:** `__tests__/nobroker-detail.test.ts` (35 tests, all passing)
- **Recon scrapers:** `scrapers/recon-nobroker-detail.ts`, `scrapers/recon-nobroker-detail2.ts`

## Pagination

`?pageNo=N` (1-indexed) on the canonical `{city}-{city}` URL. 25 items per page. No cursor — pure integer page number. Total available from `listPageOtherParams.total_count`. Delhi flats: 17,682 listings → ~707 pages at 25/page.

For the production scraper `MAX_PAGES_PER_BUCKET = 30` (750 listings per city), which covers the highest-priority listings without exhausting the full catalog.

## Gotchas

1. **The canonical URL is `{city}-{city}` not just `{city}`**: `/flats-for-rent-in-delhi-delhi` has full SSR data; `/flats-for-rent-in-delhi` returns thin SPA shell (no server-rendered listings).

2. **Kolkata and Ahmedabad are 410 Gone**: NoBroker operates in 8 cities only. Do not include these in `DEFAULT_CITIES`.

3. **Category-specific URLs (2-BHK, office, etc.) use a different flow**: They launch the "smart search" session-based flow via `resultScreenReducer`, not `rentListPage`. XHR session creation is needed to retrieve their data. The production scraper covers all BHK types via the `flats` canonical URL (which aggregates all residential rental types).

4. **BHK types observed in the "flats" canonical feed**: BHK1, BHK2, BHK3, BHK4, RK1 — all mixed together, which is by design. Category filtering in the extractor can be done post-extraction via the `typeDesc` field.

5. **Homepage warmup required**: Without a homepage visit first, the SRP may return a login-wall or incomplete state. Always warm up via `https://www.nobroker.in/` before hitting the SRP.

6. **Villas URL has SSR data but shows mixed types**: `/villas-for-rent-in-delhi-delhi` returns 200 with `nb.appState` populated but the `listPageProperties` items are mixed residential types (not exclusively villas). Not a reliable filter.

7. **Price is always integer INR in the JSON**: `rent` field is always an integer (e.g. 22000), never a Lac/Cr formatted string. The extractor does not need Lac/Cr parsing for the primary price field. `parsePrice()` exists for display strings but the raw `rent` field is preferred.

8. **`address` field is more complete than `locality`**: `locality` may be just "Rohini" while `address` contains the full street. The extractor prefers `address` as `locationText` when it's longer than `locality`.

9. **`detailUrl` is relative** — always prefix with `https://www.nobroker.in`.

10. **Rate limiting**: No observed hard rate limiting, but aggressive requests (< 2s/page) may trigger temporary soft-blocks. The 2.5–5.5s paging delay + 8–20s bucket delay is conservative and should be maintained.

11. **Detail page reducer is `propertyDetails`, NOT `rentListPage`**: The detail page populates `nb.appState.propertyDetails.detailsData`, while the SRP populates `nb.appState.rentListPage.listPageProperties`. Do not confuse the two.

12. **`extractAllHydrationPayloads` misses `nb.appState` on detail pages** because NoBroker uses `nb.appState = {...}` (not `window.nb.appState = {...}`). Use the balanced-brace parser on the `nb.appState` marker directly, or use `page.evaluate()` in the browser.

13. **Owner name is NOT in the detail page state**: The `ownerId` is present but the owner's name is behind the "Request Contact" button flow (requires user authentication). NoBroker positions itself as owner-direct, so `posterType: "Owner"` is always correct.

14. **`amenitiesMap` is preferred over `amenities`**: `amenitiesMap` is a pre-parsed object of code → boolean. The `amenities` field is a JSON-encoded string (same data). Parse `amenitiesMap` first; fall back to parsing the `amenities` string.

15. **`accurateLocation=false` is common (especially older listings)**: Even with `accurateLocation=false`, coordinates are still per-listing unique (not shared locality centroids). Classification: `true` → "exact", `false` → "locality-centroid".
