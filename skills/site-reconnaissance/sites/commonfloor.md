# CommonFloor

**Domain:** commonfloor.com
**Vertical:** India real estate (residential focus; commercial filter is ignored)
**Last verified:** 2026-05-21
**Status:** ✅ extractor + fixture + tests + live sweep 2026-05-21
**Tier:** 3 used; Tier 1 likely workable
**Framework:** Old SSR (jQuery-era pattern) — `window.stateFromServer = {}; window.stateFromServer.X = ...;` incremental construction
**Protection:** none observed

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| SRP | `/listing-search?city={city}&search_intent={rent\|sale}&property_type={type}` | `/listing-search?city=delhi&search_intent=rent&property_type=commercial-property` (filter ignored — returns residential) |
| Detail | `/listing/{descriptive-slug}/{id}` | `/listing/2-bhk-builderfloor-for-rent-in-rohini-sector-8-delhi-at-rental-/wlf70stez2qzk9zf` |

## Hydration payload

CommonFloor has **two complementary payloads**:

1. **`window.stateFromServer`** — built incrementally via `window.stateFromServer = {}; window.stateFromServer.pageType = 'SerpPage'; window.stateFromServer.city = 'Delhi'; ...`. The strict `extractAnyHydrationPayload` returns `{}` (the empty object literal). Our `synthesizeIncrementalAssignments` aggregates all the `.foo = ...` assignments into a synthetic payload.
   - **Source name:** `window.stateFromServer (synthesized)`
   - **SRP:** 22 leaves (pageType, city, localityId, ...)
   - **Detail:** 18 leaves
2. **`<script type="application/ld+json">`** — schema.org Product
   - **Source name:** `ld+json[0]`
   - **Detail payload (31 leaves) includes:**
     - `offers.price = 25000`, `offers.priceCurrency = "INR"`
     - `floorSize.value = 630`, `floorSize.unitCode = "FTK"` (FTK = sq.ft per UN/CEFACT), `floorSize.name = "Carpet Area"`
     - `address.streetAddress`, `address.addressLocality`, `address.addressRegion = "Delhi"`, `address.addressCountry = "IN"`
   - This single payload covers price + area + full address — JSON-LD is the canonical channel.
3. **Microdata** — `BreadcrumbList` with 7 items giving hierarchical location.

## Extraction strategy

Tier 1 should work. Strategy: prefer JSON-LD Product on detail, fall back to synthesized `stateFromServer` for fields not in LD.

Per-card facts on the SRP (Owner, Posted N days ago, Listed by, Property on, etc.) need `colonPairs` (inline `:` text) + `stackedPairs` (label above value in grid cells).

## Files in this repo

- **Recon demo:** `scrapers/recon-commonfloor.ts`
- **Extractor:** ✅ `scrapers/commonfloor.ts` — production scraper with `extractListings`, `parsePrice`, `parseArea`
- **Smoke test:** ✅ `scrapers/commonfloor-test.ts` — fixture capture + single-page extraction
- **Fixture:** ✅ `__tests__/fixtures/commonfloor-srp-delhi.html` — Delhi apartments SRP (2026-05-21, 257 KB)
- **Tests:** ✅ `__tests__/commonfloor.test.ts` — 31 assertions (parser unit tests + fixture extraction)

## Production output

**Sweep date:** 2026-05-21
**Command:** `bunx vinxi-scraper run commonfloor --cities=delhi --categories=apartment,independent-house,villa,builder-floor,studio,pg --out=delhi-rentals-2026-05-21`
**Output file:** `data/results/commonfloor/delhi-rentals-2026-05-21.jsonl`
**Total rows:** 78 (13 listings × 6 categories)
**Duration:** ~202s
**Category breakdown:** apartment: 13, independent-house: 13, villa: 13, builder-floor: 13, studio: 13, pg: 13

**Note on `property_type` filter:** The filter appears to be silently ignored server-side — all 6 category URLs return the same 13 default residential Delhi listings. This matches the prior recon note about `commercial-property` being ignored. CommonFloor's SRP appears to show the same "featured" listings regardless of the type filter. Total unique listings: 13 (78 rows with 6 category labels assigned).

## Pagination

`?page=N` (appended as `&page=N` since the base URL already has query params). Validated in production: page 2 returns 0 listings, confirming depth = 1 page for Delhi residential.

## Gotchas

- **ALL `property_type` filters are silently ignored** — confirmed in production: `apartment`, `independent-house`, `villa`, `builder-floor`, `studio`, `pg` all return the same 13 default residential Delhi listings. The filter param exists in the URL but is not applied server-side. Consider using a different URL pattern (e.g. `/list/apartments-for-rent-in-delhi`) if category-separated data is needed.
- **`window.stateFromServer` is built incrementally.** This is the canonical site that motivated the `synthesizeIncrementalAssignments` recovery in `schemaWalker.ts`. Without it, the SRP and detail pages yield 0 schema leaves.
- **Currency is rendered as a font-icon glyph**, NOT as the Unicode `₹` codepoint. The default audit `currencyMentions` returns 0. To detect, look for `<i class='cf-rupee'>` (or similar) immediately followed by a number. Parked enhancement.
- **Detail-page facts are in stacked-pair layout** (label above value in flex/grid cells, no class hints). The `stackedPairs` extractor catches them. Without it, the pairs count is 0 on detail.
- **Status badges visible:** none observed. `propertyStatus` and `tagSearch`-style classes exist on SRP cards but are wrappers — the `badgeText()` direct-text rule rejects them (correctly, since they're not status indicators).

## Recon output (2026-05-19)

- SRP: `window.stateFromServer (synthesized)` 22 leaves, 36 label-value pairs (13 detail-links — 1:1 with cards)
- Detail: `window.stateFromServer (synthesized)` 18 leaves, `ld+json[0]` 31 leaves, 38 label-value pairs (mostly `stackedPairs`)

## What this site teaches us

CommonFloor is the canonical example of an **older SSR site that builds its state object incrementally** rather than emitting one JSON blob. The `synthesizeIncrementalAssignments` helper in `schemaWalker.ts` exists for this class of site. Likely also applies to other long-lived Indian portals (PropertyWala, older Naukri pages, etc.) — worth a quick check if the recon shows `window.X = {}` followed by many `window.X.foo = ...` lines.
