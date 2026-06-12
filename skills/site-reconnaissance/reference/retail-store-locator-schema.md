# Reference: Retail store locator schema

Canonical data model + preservation principle for store-locator scrapes across brands. Used by all `scrapers/<brand>-stores.ts` extractors so cross-brand analysis works on a single schema.

## Core principle: capture everything, normalise some

Scraping sessions are expensive. The scraper must preserve **every field a brand emits**, even if it doesn't fit the canonical model. The canonical fields exist for downstream joining and dashboards; the `_extra` bag exists so we never re-scrape just because we discovered a useful field later.

**Rule:** if a field is in the source, it goes into the output — either into a canonical slot OR into `_extra`. No silent drops.

## Schema

```typescript
interface RetailStore {
  // ---- identity ----
  brand: string                     // "Zara", "H&M", "Mango", ...
  storeId: string | null            // brand's own ID if exposed
  name: string                      // "Mango DLF Promenade"
  status: "open" | "coming-soon" | "closed" | "unknown" | null

  // ---- location ----
  mallOrLocation: string | null     // "DLF Promenade", "Ambience Mall"
  address: {
    line1: string                   // street address
    line2: string | null
    locality: string | null         // neighbourhood / sub-area
    city: string
    state: string | null
    postalCode: string | null
    country: string                 // ISO-3166 alpha-2: "IN", "US", ...
  }
  lat: number | null
  lng: number | null

  // ---- contact ----
  phone: string | null
  email: string | null
  url: string | null                // brand's per-store page if any

  // ---- operating ----
  /**
   * Day-of-week → hours window string. Keys lowercase ("mon"|"tue"|...).
   * Value examples: "10:00-22:00", "10:00-13:00, 16:00-21:00" (split shifts),
   * "closed", "24h".
   */
  hours: Record<string, string> | null

  // ---- segmentation ----
  storeType: string | null          // "Flagship" | "Outlet" | "Express" | "Online-only" | ...
  segments: readonly string[]       // ["WOMEN", "MEN", "KIDS"] — what the store sells
  features: readonly string[]       // ["Click & Collect", "Online Returns", "Tailoring", "Wheelchair"]

  // ---- provenance ----
  sourceUrl: string                 // canonical detail URL OR the page we extracted from
  scrapedAt: string                 // ISO timestamp

  // ---- preservation ----
  /**
   * EVERY field from the source that didn't fit the canonical slots above.
   * Keys preserve the source's naming (do NOT normalise). Values preserve
   * the source's types. Future schema upgrades can promote a frequently-used
   * key into a canonical slot.
   *
   * This is non-negotiable: never drop data just because we don't have a
   * canonical home for it.
   */
  _extra: Record<string, unknown>
}
```

## Mapping conventions

When converting a source record to `RetailStore`:

1. **Promote known fields to canonical slots.** If the source has `latitude`/`Latitude`/`lat`, fill `lat`. Same for `lng`, `address`, `phone`, etc.
2. **Normalise the canonical values:**
   - `country` → ISO-3166 alpha-2 (`"IN"` not `"India"`)
   - `hours` keys → lowercase 3-letter (`"mon"`, `"tue"`, ...)
   - `lat`/`lng` → `number`, not `string`
   - `name` → trimmed, single-spaced
   - `segments` / `features` → array of trimmed strings, deduplicated
3. **Everything else goes into `_extra`.** Use the source's original key naming. If `storeRecord.openingYear === 2018`, write `_extra.openingYear = 2018`. Never drop.
4. **Don't compute derived fields** during scraping. `nearestCompetitor`, `distanceToCity`, etc. belong in a downstream pass — keep the scrape lean.

## Identifying these fields during recon

When walking a schema (via `walkSchema()`), bucket leaves into the canonical slots. The recon scaffolding pattern:

```typescript
const buckets = {
  identity: filterLeaves(leaves, ["id", "code", "uid", "storenumber", "name", "title"]),
  status: filterLeaves(leaves, ["status", "open", "closed", "active", "coming"]),
  geo: filterLeaves(leaves, ["latitude", "longitude", "lat", "lng", "coord", "geo"]),
  address: filterLeaves(leaves, ["address", "street", "city", "state", "postal", "zip", "country"]),
  contact: filterLeaves(leaves, ["phone", "tel", "email", "url", "website"]),
  hours: filterLeaves(leaves, ["hours", "open", "close", "schedule", "timing", "monday", "weekday"]),
  segmentation: filterLeaves(leaves, ["segment", "department", "category", "type", "kind", "feature"]),
  // Whatever isn't here — log it too. Don't decide it's noise until you've looked.
}
```

The `_extra` bag catches everything that doesn't match a bucket, plus anything that's in `_extra` because the canonical slot already has a higher-priority source value.

## Per-brand customisation

Every brand has fields the others don't:

- **Mango**: `segments: ["WOMEN", "MEN", "KIDS", "TEEN"]` (Mango segments stores explicitly)
- **Uniqlo**: `services: ["alterations", "online-pickup"]`
- **Westside**: `region: "South"|"West"|"North"|"East"`
- **Zara**: `phoneSecondary`, `customerCareEmail`

Push these into `_extra`. If a field shows up across ≥3 brands, promote it to canonical in a future schema revision.

## Cross-brand analysis enablers

With this schema, downstream queries like:

- **"Show me all retail stores within 2 km of {lat,lng}"** — geo index over `lat`/`lng`
- **"Which mall has the most Zara competitors?"** — group by `mallOrLocation`, count brands
- **"Compare Zara coverage vs. H&M in Tier-2 cities"** — `city` filter + brand pivot
- **"Brands offering Click & Collect"** — `features` filter

...all work without per-brand special cases.

## Schema-locked tests

Each brand's extractor should have at least:

- `extract<Brand>Stores(payload, sourceUrl) → RetailStore[]` — pure function on a captured payload
- Tests against an HTML/JSON fixture asserting:
  - All canonical fields populated where the source provided them
  - `_extra` is non-empty (preservation working)
  - Specific values (`lat` close to known coordinate, `phone` matches expected, etc.)
  - Country normalised to alpha-2
  - Hours keys lowercase

When the brand's payload schema changes silently, these tests fail loud.

## Coordinate recovery from Google Maps embeds

If the source provides a `google_emmbeded_code` (or similar) URL in addition to lat/lng fields, the URL itself carries the coordinates as `!2d{lng}!3d{lat}`. See [`../patterns/google-maps-embed-recovery.md`](../patterns/google-maps-embed-recovery.md). Real impact: Zudio went from 0% to 99% lat/lng coverage by adding this fallback to the extractor.

## Validation scorecard

Every brand's extraction is followed by a numeric scorecard across 4 dimensions: Completeness, Data Quality, Source Authenticity, Freshness. See [`../patterns/validation-scorecard.md`](../patterns/validation-scorecard.md). Bands: high (≥80) ship, medium (60-79) caveat, low (<60) re-recon.

## Output convention

Each brand writes to `data/results/<brand>-stores.jsonl`. One JSON line per store. The unified cross-brand view is just:

```bash
cat data/results/*-stores.jsonl > data/all-retail-stores.jsonl
```

That's the deliverable. Geospatial / pivot analysis from there.
