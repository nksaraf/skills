# 99acres

**Domain:** 99acres.com
**Vertical:** India real estate (residential + commercial rental + sale)
**Last verified:** 2026-05-22
**Tier:** 3 (Playwright required)
**Framework:** Custom React SSR — `server-timing: react;dur=...` in response headers
**Protection:** Akamai Bot Manager (AkamaiGHost) — `AkamaiGHost` server header

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| SRP | `https://www.99acres.com/{category}-for-rent-in-{city}-ffid?page=N` | `/warehouse-for-rent-in-delhi-ffid?page=2` |
| Detail | URL with `-spid-XXXXXXXX` suffix | `/warehouse-for-rent-lease-in-kirti-nagar-...-spid-F82453766` |

### Residential category slug table (verified 2026-05-22)

| Slug | URL | Inventory (Delhi) | Notes |
|---|---|---|---|
| `flats` | `flats-for-rent-in-{city}-ffid` | 17,720+ | Pluralized — NOT `flat-for-rent` |
| `house` | `house-for-rent-in-{city}-ffid` | 1,412+ | Also works as `independent-house` |
| `independent-house` | `independent-house-for-rent-in-{city}-ffid` | same pool | Alias for `house` |
| `builder-floor` | `builder-floor-for-rent-in-{city}-ffid` | 10,213+ | |
| `studio-apartments` | `studio-apartments-for-rent-in-{city}-ffid` | 806+ | Plural — NOT `studio-apartment` |
| `farm-house` | `farm-house-for-rent-in-{city}-ffid` | small (~25 per page) | |

### Commercial category slug table (verified 2026-05-19)

| Slug | URL | Notes |
|---|---|---|
| `warehouse` | `warehouse-for-rent-in-{city}-ffid` | |
| `showroom` | `showroom-for-rent-in-{city}-ffid` | |
| `office-space` | `office-space-for-rent-in-{city}-ffid` | |
| `commercial-shops` | `commercial-shops-for-rent-in-{city}-ffid` | |

- **City names:** lowercase, dash-separated (delhi, mumbai, gurgaon, noida, ...)

## Hydration payload

- **Location:** `window.__initialData__` (inline `<script>`, ~69 KB on detail page)
- **Source name from `extractAllHydrationPayloads`:** `window.__initialData__`
- **Schema:** root → `pd.pageData`
  - **`propertyDetails.prop_data`** — the main extraction target (~30 useful fields)
    - `Latitude`, `Longitude`
    - `Price`, `Price_Text`, `Min_Price`, `Max_Price`, `Is_Price_Negotiable`
    - `Is_Price_All_Inclusive`, `electricityWaterCharges`
    - `Super_Area`, `Superarea_Unit`, `displaySuperArea`, `displaySuperAreaUnit`, `displaySuperAreaSqft`
    - `Description`, `AddressTupple`
    - `Posted_On_Label`, `Register_Date`, `isPaidOwnerListing`
    - `Verified` (Y/N), `isPremiumListing` (bool)
    - `Map_Status` (MAPPED/...)
    - `washroom_Number`, `washroomsLabel`
    - `topUsp` (array of `{id, label}` — facing, nearby)
    - `uspV2` (Vaastu, flooring)
    - `nearByPlacesOfInterest` (array of 47 POIs with `category, text, roadDistance`)
    - `ContactDetails.{userClass, company_label, only_name_label, Phone, address1_label, ...}`
    - `specification.amenities.{homeAmenities, societyAmenities}` ← **BOTH lists needed**
    - `specification.flooringLabel`, `specification.furnishStatusLabel`
  - **`AdvertiserDetails`** — dealer info, `dealer_localities`
- **SRP also has JSON-LD `ItemList`** at `<script type="application/ld+json">` with 27 listing URLs per page

## Extraction strategy

**Tier 3 mandatory.** First HTTP request slips through Akamai with realistic Chrome headers; the *next* request from the same IP gets 403. Production scraping needs Playwright Chromium with stealth + Indian locale + Client Hints. The runtime's default `ctx.browser.newPage()` already includes everything needed.

## Field inventory

The 99acres extractor is the most complete in the repo. See `extractDetail()` in `scrapers/99acres-detail-test.ts` for the canonical mapping. Covers ~65 fields per listing.

## Files in this repo

- **SRP extractor:** `scrapers/99acres-test.ts` (single page) + `scrapers/99acres.ts` (production: 9 categories × 10 cities × up to 30 pages)
- **Detail extractor:** `scrapers/99acres-detail-test.ts` (single URL) + `scrapers/99acres-detail.ts` (lazy enrichment from SRP results JSONL)
- **Verify scraper (for manual cross-check):** `scrapers/99acres-verify.ts`
- **Fixtures:** `__tests__/fixtures/99acres-detail-M90237068.html` (398 KB, Mayapuri warehouse) + `__tests__/fixtures/99acres-srp-delhi-residential.html` (1.6 MB, Delhi flats SRP page 1)
- **Tests:** `__tests__/99acres.test.ts` (9 unit parsers), `__tests__/99acres-detail.test.ts` (11 schema-locked), `__tests__/99acres-residential.test.ts` (9 schema-locked residential)

## Pagination

- **Pattern:** `?page=N`, 1-indexed
- **Per page:** 27 listings
- **Terminator:** `.tupleNew__outerTupleWrap` count == 0
- **Hard ceiling:** Production scraper caps at 30 pages per (category, city)

## Gotchas

- **Residential URL pattern is PLURAL, not singular.** `flats-for-rent-in-{city}-ffid` (with `s`) returns 17,720+ listings. `flat-for-rent-in-{city}-ffid` (singular) silently redirects to the generic `property-for-rent-in-{city}-ffid` page. Same trap for `studio-apartments` (plural) vs `studio-apartment` (singular). The previous failed attempt used singular forms and got 0 residential results.
- **`pg-for-rent-in-{city}-ffid` is dead.** 99acres removed PG/paying-guest as a top-level category. The URL returns "The Page is gone". There is no residential PG category via this URL pattern.
- **`penthouse-for-rent-in-{city}-ffid` 404s.** Not a valid category on 99acres.
- **`villa-for-rent-in-{city}-ffid` redirects** to the generic property-for-rent page. Villas appear under `flats` or `house` SRPs as mixed results.
- **Akamai session rate-limiting:** After running multiple Playwright sessions in quick succession (recon, smoke test, commercial verify, main sweep), all within the same hour, Akamai blocks the IP entirely — even page 1 with warmup returns 0 tuples. Wait ~5 minutes between burst sessions.
- **Akamai single-shot allowance:** First request with realistic browser headers succeeds; further requests in the same session get 403. Stealth Chromium + session reuse + humanized pacing is required.
- **POI badge wrapper false positives:** The 47 nearby places are inside `<div class="tags-and-chips__badgeParent">` — without the `badgeText()` direct-text rule in `visualAudit.ts`, they all match as status badges. Real badges: `Verified`, `Featured`, `STATUS`, `NOT AVAILABLE`, `NEW`, `FREE`.
- **`NOT AVAILABLE` is DOM-only:** Not in the payload. Look in `<span class="component__status"><b>NOT AVAILABLE</b></span>`. Critical for filtering stale inventory — without this filter, rent benchmarks include withdrawn listings.
- **`specification.amenities` has TWO lists:** `homeAmenities` (e.g. Vaastu Compliant) AND `societyAmenities` (e.g. Water Storage). Easy to miss the second.
- **Area is in sq.yd by default** for warehouses. `displaySuperAreaSqft` may be missing — derive from `Super_Area * 9` if unit is `sqyd`, or `displaySuperArea * 10.7639` if `sq.m.`.
- **Phone numbers are masked** in payload (`98***543**`). Reveal requires login + CAPTCHA — not in scope.
- **`Posted_On_Label` is relative** (`"Updated 1m ago by dealer"`). `Register_Date` is absolute (`"Apr 10, 2026"`).

## Verification screenshots (last run)

`/tmp/99acres-verify-{1,2,3}-{top,mid,bot}.png` — for visual diff against scrape.
