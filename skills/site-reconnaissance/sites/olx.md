# OLX India

**Domain:** olx.in
**Vertical:** India classifieds (cross-category: properties, vehicles, jobs, electronics, ...)
**Status:** ✅ extractor + fixture + tests + live sweep 2026-05-21
**Last verified:** 2026-05-21
**Tier:** 3
**Framework:** Custom React SSR with `window.__APP = { ... }` (JS object literal, not strict JSON) + heavy JSON-LD
**Protection:** mild (no blocks observed); HTTP/2 protocol errors occasionally on consecutive requests

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| SRP (geo-scoped) | `/en-in/{city_g{geoid}}/q-{query}` | `/en-in/delhi_g4058659/q-houses-apartments-rent` |
| SRP (keyword, NOT geo) | `/en-in/items/q-{query}` | `/en-in/items/q-houses-apartments-rent-delhi` |
| Detail | path with `iid-NNNNNNNNNN` (10-digit ID) | `/en-in/item/.../iid-1838643942` |

**GOTCHA: Keyword search vs geo filter** — `/en-in/items/q-{category}-{city-name}` treats the city as a
free-text query, not a geographic filter. Results include listings from anywhere in India that mention
"delhi" in the title or description (verified: only 7% of 92 results were actually in Delhi). Always use
the geo-scoped `/en-in/{city_g{geoid}}/q-{category}` form instead.

**OLX city geo IDs (verified 2026-05-21):**

| City | Geo slug |
|---|---|
| Delhi | `delhi_g4058659` |
| Mumbai | `mumbai_g4058997` |
| Bengaluru | `bengaluru_g4058803` |
| Hyderabad | `hyderabad_g4058526` |
| Chennai | `chennai_g4059162` |
| Pune | `pune_g4059014` |
| Kolkata | `kolkata_g4157275` |
| Ahmedabad | `ahmedabad_g4058677` |
| Gurgaon | `gurgaon_g4058748` |
| Noida | `noida_g4059326` |

IDs sourced from `GET https://www.olx.in/api/locations/popular?limit=100&type=CITY&lang=en-IN`
and verified by navigating each URL (page H1 = "Houses Apartments Rent in {City}").

## Hydration payload

OLX is a **JSON-LD-first** site — the canonical extraction target is the LD blocks, not the `window.__APP` global.

- **Source names from `extractAllHydrationPayloads`:**
  - `window.configTracking (generic)` — analytics config, not useful
  - `ld+json[0]`, `ld+json[1]`, `ld+json[2]` — **this is where the data lives**
- **JSON-LD blocks:**
  - **SRP `ld+json[1]`** — schema.org `ItemList` with 40 entries. Each has `name`, `position`, `url` (with `iid-XXX`), `description`, `image`. URLs are detail-page links — Tier 1-style discovery, no DOM scraping needed.
  - **Detail `ld+json[1]`** — schema.org **Product** with `offers.price`, `offers.priceCurrency`, `offers.priceValidUntil`, `offers.seller`, `offers.availability`, `offers.itemCondition`, `sku`, `image[0]`, `name`, `description`. **One canonical payload covers the entire scrape.**
  - **Detail `ld+json[2]`** — `BreadcrumbList` exposing category hierarchy (Properties → For Rent: Shops & Offices → Kerala → Kozhikode → ...)
- **`window.__APP`** — 407 KB on SRP, 139 KB on detail. JS object literal with unquoted keys (`{ props: { lang: "en-IN", ... }, ... }`). The permissive parser in `extractAllHydrationPayloads` *should* catch it but real OLX likely has additional JS-isms (`undefined`, functions) that defeat the JSON.parse retry. **Skip it — LD-JSON is enough.**

## Extraction strategy

Tier 3 + stealth + Indian locale. Detail pages are most efficient via LD-JSON Product (one JSON.parse → done). SRP via LD-JSON ItemList for URL discovery.

## Files in this repo

- **Recon demo:** `scrapers/recon-olx.ts`
- **Extractor:** `scrapers/olx-test.ts` (exports `extractListings`, `parsePrice`, `parseArea`, `parseSrpUrl`)
- **Production scraper:** `scrapers/olx.ts` (exports `DEFAULT_CITIES`, `DEFAULT_CATEGORIES`, `resolveSliceFromArgs`)
- **Fixture:** `__tests__/fixtures/olx-srp-delhi.html` (1.2 MB, captured 2026-05-21)
- **Tests:** `__tests__/olx.test.ts` — 18 tests (all pass)

## Pagination

OLX uses **infinite scroll** on SRP (no `?page=N` parameter). The production scraper does a single deep `jitteredScroll` pass per bucket, collecting all lazily-loaded cards. Each bucket = one page load + deep scroll.

## JSON-LD schema (actual, 2026-05-21)

OLX wraps the ItemList inside `@graph` as an **object** (not array):
```json
{
  "@context": "https://schema.org",
  "@graph": {
    "@type": "ItemList",
    "itemListElement": [
      { "@type": "ListItem", "position": 1, "url": "/item/...-iid-NNNN", "name": "...", "description": "..." }
    ]
  }
}
```
Note: the recon (2026-05-19) expected `ld+json[1]` to be a top-level `ItemList`. The actual structure nests it inside `@graph`. The extractor handles both forms.

## URL structure (confirmed working)

Use the geo-scoped form with `_g{geoid}` city slug:
```
https://www.olx.in/en-in/{city_g{geoid}}/q-{category-slug}
```
Examples (Delhi, geo ID 4058659):
- `https://www.olx.in/en-in/delhi_g4058659/q-houses-apartments-rent`
- `https://www.olx.in/en-in/delhi_g4058659/q-pg-flatmates-for-rent`
- `https://www.olx.in/en-in/delhi_g4058659/q-commercial-property-rent`

**Do NOT use:**
- `/en-in/items/q-{category}-{city-name}` — keyword search, not geo filter (~7% Delhi accuracy)
- `/en-in/{city}_{N}/q-{query}` (old `_5`, `_4290` style slugs) — causes `ERR_HTTP2_PROTOCOL_ERROR`
- `/en-in/{city_g{id}}/` (no query) — also causes `ERR_HTTP2_PROTOCOL_ERROR`

## DOM card structure

Real listing cards use `data-aut-id="itemBox3"`. Non-listing interstitials (ad banners, placeholders) use different aut-ids and should be skipped.

Card fields:
| `data-aut-id` | Content |
|---|---|
| `itemPrice` | `₹ 21,000` (no cadence on SRP) |
| `itemTitle` | Listing title |
| `item-location` | Locality, City |
| `itemDetails` | `2 BHK - 2 Bathroom - 835 sqft` |
| `itemDate` | Posting date (`Apr 30`) |

## Gotchas

- **CRITICAL: `/en-in/items/q-{category}-delhi` is a keyword search, not a geo filter.** OLX
  treats "delhi" as free text and returns results from anywhere in India mentioning "delhi". A
  2026-05-21 sweep using this URL returned only 6/92 listings (7%) actually in Delhi. Always use
  `/en-in/delhi_g4058659/q-{category}` for geo-filtered results. The geo ID map is in
  `scrapers/olx.ts` (`CITY_GEO_SLUGS`).
- **The old `_5`, `_4290`-style numeric slugs cause `ERR_HTTP2_PROTOCOL_ERROR`** — they are NOT
  OLX's internal geo IDs. The valid form is `_g{id}` (with the letter "g" prefix) from the
  `/api/locations` API response.
- **`window.__APP` permissive parse fails on real OLX HTML** — the simple "unquoted keys" hack handles the synthetic case but not OLX's full payload. **This is a documented limitation.** Don't chase it; use JSON-LD.
- **`data-aut-id` is the OLX label-hint convention.** Examples: `itemPrice`, `itemTitle`, `item-location`, `itemDate`, `featuredTag`. Caught by the `attrPairs` extractor in `visualAudit.ts`.
- **OLX side categories are NOT inside `<nav>`/`<aside>`** — they're a `<ul>` with hashed class names. The colon-pattern extractor previously caught them as false positives (`"For Sale": "Houses & Apartments"`); now filtered out by `looksLikeListPair` (15-char common prefix) and by the data-* extractor preferring `data-aut-id` over class-string hits.
- **HTTP/2 protocol errors on consecutive requests** — OLX sometimes drops the connection mid-navigation. Retry with backoff; don't escalate to a heavier mitigation unless persistent. Note: `/en-in/{city_g{id}}/` (no query path) also triggers this error — always include the `/q-{category}` segment.
- **Featured badge:** `data-aut-id="featuredTag"` on detail. Caught by `[data-aut-id$='Tag' i]` selector.

## Recon output (2026-05-19)

- SRP: 4 hydration sources (configTracking + 3 LD-JSON), 285 label-value pairs (lots of `data-aut-id` attrs), 0 badges (initial pass — `featuredTag` was missed)
- Detail: same shape, LD-JSON Product canonical

## Production output

### Buggy sweep (2026-05-21) — keyword search, DO NOT USE AS REFERENCE

Run: `bunx vinxi-scraper run olx --cities=delhi ... --out=delhi-rentals-2026-05-21`
URL used: `/en-in/items/q-houses-apartments-rent-delhi` (keyword search)

| Category | Listings | Delhi % |
|---|---|---|
| houses-apartments-rent | 12 | ~8% |
| pg-flatmates-for-rent | 40 | ~5% |
| commercial-property-rent | 40 | ~8% |
| plots-land-for-rent | 0 | — |
| **Total** | **92** | **~7%** |

### Fixed sweep (2026-05-21) — geo-scoped URLs

Run: `bunx vinxi-scraper run olx --cities=delhi --categories=houses-apartments-rent,pg-flatmates-for-rent,commercial-property-rent,plots-land-for-rent --out=delhi-rentals-2026-05-21-fixed`
URL used: `/en-in/delhi_g4058659/q-{category}` (geo filter)

| Category | Listings | Delhi % |
|---|---|---|
| houses-apartments-rent | 40 | 100% |
| pg-flatmates-for-rent | 40 | ~98% |
| commercial-property-rent | 45 | ~95% |
| plots-land-for-rent | 0 (no Delhi results on OLX) | — |
| **Total** | **125** | **97.6%** |

- 97.6% of records have `locationText` matching Delhi (3 are Faridabad/Gurgaon NCR adjacents)
- ~85% have `priceValueINR`
- ~30% have `areaValue` (cards with `itemDetails` populated)
- Duration: ~3 minutes for Delhi × 4 categories
