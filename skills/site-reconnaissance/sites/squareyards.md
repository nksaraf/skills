# Square Yards

**Domain:** squareyards.com
**Vertical:** India real estate (residential + commercial rentals)
**Last verified:** 2026-05-22
**Tier:** 3 (stealth browser — Akamai present; data is in SSR HTML)
**Framework:** Custom Node.js SSR (no Next.js or React hydration payloads; inline CSS + vanilla JS)
**Protection:** Akamai Bot Manager (present, `akamai-grn` response header; homepage + SRP served 200 with plain curl; stealth browser recommended for production)

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| SRP page 1 | `https://www.squareyards.com/rent/{category}-for-rent-in-{city}` | `https://www.squareyards.com/rent/apartments-for-rent-in-delhi` |
| SRP page N | `https://www.squareyards.com/rent/{category}-for-rent-in-{city}?page=N` | `.../delhi?page=2` |
| Detail | `https://www.squareyards.com/rental-{slug}/{propertyid}` | `.../rental-3-bhk-1650-sq-ft-apartment-in-sector-11-dwarka/10375543` |

**Confirmed category slugs (rental):**
- Residential: `apartments`, `independent-houses`, `villas`, `builder-floors`, `pg`
- Commercial: `office-spaces`, `shops`, `warehouses`, `commercial-properties`, `co-working-spaces`

**Confirmed city slugs:** `delhi`, `mumbai`, `bangalore`, `gurgaon`, `noida`, `pune`, `hyderabad`, `chennai`, `kolkata`, `ahmedabad` (all return HTTP 200)

**Pagination:** `?page=N` — 25 listings per page; total shown in `<div class="total-property">Showing X Listings</div>`. Delhi apartments: 5,996 listings / 240 pages (as of 2026-05-22).

## Hydration payload

- **Location:** No `window.X` hydration payload. No `__NEXT_DATA__`. No JSON-LD listing data (58 JSON-LD blocks but all are BreadcrumbList/FAQPage/meta, no listing arrays).
- **Extraction method:** Parse `data-*` attributes from shortlist div elements in SSR'd HTML.

**Primary element per listing:**
```html
<div class="favorite-btn shortlistcontainerlink{N}"
  data-propertyid="10375543"
  data-price="50000"
  data-totalPrice="₹ 50,000"
  data-priceDurationName="Per Month"
  data-city="Delhi"
  data-locality="Sector 11 Dwarka, Delhi"
  data-sublocalityname="Sector 11 Dwarka"
  data-name="3 BHK Apartment For Rent in Sector 11 Dwarka"
  data-propname="3 BHK Apartment for Rent in Sector 11 Dwarka"
  data-property-title="3 BHK Apartment For Rent in Sector 11 Dwarka"
  data-area="1650 Sq.Ft."
  data-propertylink="https://www.squareyards.com/rental-3-bhk-..."
  data-propertytype="Apartment"
  data-unittype="3 BHK"
>
```

## Extraction strategy

Tier 3: stealth Playwright. Homepage warmup seeds Akamai trust signals. SRP pages are SSR'd with full listing data in `data-*` attributes — the regex pattern `/<div\s+class="favorite-btn\s+shortlistcontainerlink\d+"([^>]*)>/g` matches all 25 listing entries per page. No XHR needed.

Warmup: `ctx.human.warmup(page, "https://www.squareyards.com/")` before any SRP.

## Field inventory

| Field | Source | Notes |
|---|---|---|
| listingId | `data-propertyid` | Numeric string, e.g. "10375543" |
| url | `data-propertylink` | Full canonical detail URL |
| title | `data-name` (preferred) / `data-property-title` / `data-propname` | Long form has config details |
| locationText | `data-locality` (preferred) / `data-sublocalityname` | "Locality, City" format |
| priceValueINR | `data-price` | Exact integer INR — no Lac/Cr parsing needed |
| priceText | `data-totalPrice` | Formatted: "₹ 50,000" or "₹ 2.25 L" |
| priceCadence | `data-priceDurationName` | "Per Month" (default "month" when empty) |
| areaText | `data-area` | "1650 Sq.Ft." / "133 Sq.Yd." |
| areaValue | parsed from `data-area` | Numeric value |
| areaUnit | parsed from `data-area` | "sqft" / "sqyd" / "sqmt" |
| city | `data-city` | City name ("Delhi", "Mumbai", ...) |

## Files in this repo

- **Extractor:** `scrapers/squareyards-test.ts` (smoke), `scrapers/squareyards.ts` (production)
- **Fixture:** `__tests__/fixtures/squareyards-srp-delhi.html` (731 KB, page 1 of Delhi apartments)
- **Tests:** `__tests__/squareyards.test.ts` (21 tests, 0 failures)

## Pagination

`?page=N` on the SRP URL. 25 listings per page. Last page shown in the last pagination link (`<a title="Page 240" ...>`). Empty page = no `shortlistcontainerlink` divs → stop.

## Geographic accuracy

Verified 100% Delhi accuracy on pages 1 and 2 for `apartments-for-rent-in-delhi` (50/50 listings had `data-city="Delhi"`). The URL is a true city filter, not a keyword search.

## Gotchas

- `data-priceDurationName=""` (empty string) appears on ~12% of listings. The site implies "Per Month" but the attribute is blank. Extractor defaults to "month" when empty.
- Area can be in `Sq.Ft.` or `Sq.Yd.` — both appear in Delhi apartments page 1.
- `data-propertytype` has a space before `=` (`data-propertytype = "Apartment"`) — regex must accommodate `\s*=\s*`.
- Akamai `akamai-grn` header present but plain curl returns 200 on SRPs. For production use stealth browser to avoid long-term rate limiting.
- `co-working-spaces` category exists but was not included in DEFAULT_CATEGORIES (smaller inventory, overlap with office-spaces).
