# PropertyWala

**Domain:** propertywala.com
**Vertical:** India real estate (residential + commercial rental — NCR focus)
**Last verified:** 2026-05-22
**Tier:** 1 (pure SSR HTML — no XHR, no hydration state, no JSON-LD)
**Framework:** Custom ASP.NET / jQuery SSR (Bootstrap 5 + pw.min.js for UI interactions; search results are server-rendered HTML)
**Protection:** None observed (no Akamai/Cloudflare/Imperva; homepage warmup helps but not strictly required)

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| SRP (page 1) | `/properties/type-{type}/for-rent/location-{city}` | `/properties/type-residential_apartment_flat/for-rent/location-new_delhi` |
| SRP (page N) | `/properties/type-{type}/for-rent/location-{city}?page={N}` | `/properties/type-residential_apartment_flat/for-rent/location-new_delhi?page=2` |
| Detail | `/{id}` | `/P944779781` |
| City landing | `/{city}` | `/new_delhi` |

### Property type slugs

| Category | Slug |
|---|---|
| Apartment | `residential_apartment_flat` |
| Independent house | `residential_independent_house` |
| Paying guest / hostel | `residential_paying_guest` |
| Office space | `commercial_office_space` |
| Shop / showroom | `commercial_shop` |
| Warehouse / godown | `commercial_warehouse_godown` |
| Multipurpose building | `commercial_multipurpose_building` |

### City slugs (Delhi NCR)

| City | Slug |
|---|---|
| New Delhi | `new_delhi` |
| Delhi NCR (broader) | `delhi_ncr` |
| Noida | `noida_uttar_pradesh` |
| Gurgaon | `gurgaon_haryana` |
| Faridabad | `faridabad_haryana` |
| Ghaziabad | `ghaziabad_uttar_pradesh` |

## Hydration payload

- **Location:** None — the page is fully server-rendered HTML.
- **No `window.__` variables**, no `<script id="__NEXT_DATA__">`, no JSON-LD.
- Listings are rendered as `<article id="P{id}" class="listing live property-listing ...">` elements in the DOM.

## Extraction strategy

**Tier 1 — pure HTML parse.** No JavaScript execution required. The full SRP is rendered server-side; the browser can extract listings from `page.content()` after `waitUntil: "domcontentloaded"`.

A homepage warmup is done to seed cookies and avoid cold-start fingerprinting, but is not strictly required (the site has no active bot protection).

30 listings per page. Pagination via `?page=N`. The total listing count is shown in the page (`"96 Apartments for Rent in New Delhi"`).

## Field inventory

| Field | Source | Notes |
|---|---|---|
| listingId | `article[id]` | Format: `P{digits}` (e.g. `P944779781`) |
| url | `figure > a[href]` relative path prefixed with domain | `/P{id}` → `https://www.propertywala.com/P{id}` |
| title | `h3 > a` text (strip inner `<b>` tags) | e.g. "3BHK Flat for rent in Chanakyapuri" |
| locationText | `h3 a[title]` — "X for rent in **Location** - P{id}" | Includes locality + city, e.g. "Chanakyapuri, New Delhi" |
| priceText | `div.property-price` | e.g. "₹ 3.15 L" (Lakh format); empty for "price on request" listings |
| priceValueINR | Parsed from priceText | Lakh → ×100,000; always monthly for rentals |
| priceCadence | Hardcoded "month" | PropertyWala rental SRPs are always monthly |
| areaText | `span.areaUnit` | e.g. "375 SqYards", "3,000 SqFeet", "200 SqMeters" |
| areaValue | Parsed from areaText | Commas stripped; float |
| areaUnit | Parsed from areaText | "sqft" / "sqyd" / "sqm" |

## Files in this repo

- **Extractor:** `scrapers/propertywala.ts` (production scraper)
- **Smoke test:** `scrapers/propertywala-test.ts`
- **Fixture:** `__tests__/fixtures/propertywala-srp-delhi.html` (Delhi apartments SRP, 2026-05-22, 30 listings)
- **Tests:** `__tests__/propertywala.test.ts` (25 assertions)

## Pagination

`?page=N` appended to the SRP URL. Page 1 has no page parameter. 30 listings per page. The page shows total count text (e.g. "96 Apartments for Rent in New Delhi"). Pages beyond the last return an empty listing container (0 articles).

## Inventory size (Delhi NCR, 2026-05-22)

| Category | City | Pages | Approx listings |
|---|---|---|---|
| Apartments | new_delhi | 4 | 96 |
| Apartments | delhi_ncr | ~varies | smaller |
| Independent house | new_delhi | ~varies | smaller |

PropertyWala is a niche NCR-focused portal with lower inventory than 99acres / MagicBricks (~100-500 listings per category vs thousands). Crawls complete quickly.

## Gotchas

- **City dropdown is AJAX-loaded** on the homepage (initially empty `<select>`) — navigating directly to the SRP URL avoids this entirely.
- **driver.js overlay** on homepage (guided tour) intercepts pointer events — not relevant for direct URL navigation, but a problem if you try to interact with the search form.
- **~10% of listings have no price** (`div.property-price` is empty) — these are "price on request" listings. Extract them anyway; `priceValueINR` will be `null`.
- **Area units vary**: SqFeet (most common), SqYards, SqMeters — all observed in fixture.
- **The `/properties/` page with query params** (e.g. `?transaction_type=R&city=new_delhi`) returns a 70KB shell with NO listings. The correct URL is the path-based format: `/properties/type-{type}/for-rent/location-{city}`.
- **API exists** at `/api/documentation/` but is key-gated; not needed since SSR HTML is easier to extract.
