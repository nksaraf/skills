# Pattern: Post-extraction validation scorecard

Every scrape ends with a **scorecard** that produces numeric confidence across four dimensions. Without this step, you don't actually know if you've shipped good data — you just know the code ran.

See `@vinxi/scraper/recon` (`scrapeScoreCard`) for the implementation and `scrapers/validate-retail.ts` for the runner.

## The four dimensions

Each is 0-100 with weighted evidence. The composite "overall" weights them: 40% Completeness, 30% Data Quality, 20% Authenticity, 10% Freshness.

### 1. Completeness — "did we get all the records?"

Evidence:
- **Source claim** (50%) — Does the source's own published count match what we extracted? (`Westside API returns store_count: 300; we extracted 300 → 100`)
- **Authoritative cross-check** (30%) — Does our count agree with public reference (Wikipedia, annual report, press)? Tolerance configurable (default 15%).
- **Sibling-brand exhaustiveness** (10%) — For multi-brand parents, did we probe every sibling? See [`sibling-brand-probing.md`](./sibling-brand-probing.md).
- **Major-metro coverage** (10%) — A national brand with N≥50 stores must appear in all 6 major Indian metros. (Tier-2 brands with fewer stores get a softer threshold.)

### 2. Data Quality — "how complete is each record?"

Evidence:
- `has-coords` (30%) — lat/lng populated
- `has-address` (20%) — line1 + city
- `has-phone` (15%)
- `has-hours` (15%)
- `has-storeId` (10%)
- `has-postal` (5%)
- `has-extra` (5%) — `_extra` bag non-empty (preservation working)
- **Penalty:** out-of-India coordinates reduce the score directly (up to -20).

### 3. Source Authenticity — "is this a real source?"

Asserted manually by the agent (not programmatically inferable):
- `domain-brand-match` (30%) — Source domain matches the brand's official domain
- `known-brand` (25%) — The brand is well-known (manual)
- `public-company` (20%) — Publicly listed → audited disclosures
- `wikipedia` (15%) — Has a Wikipedia article (third-party notability)
- `https-source` (10%) — HTTPS confirmed

These are written into the brand's `BrandContext` based on public research before validation runs.

### 4. Freshness — "how recent is the data?"

Evidence:
- `scrape-timestamp` (50%) — How recently we ran the scrape
- `source-updated-at` (50%) — Median `updated_at` across all records (if the source exposes one)

Source-side `updated_at` is the more important signal — a scrape today of data the brand last refreshed in 2022 should score lower than a scrape today of fresh data.

## How to add a new brand to the validation runner

1. Extract the brand to `data/results/<brand>-stores.jsonl` using the canonical schema
2. Open `scrapers/validate-retail.ts` and add a `BrandContext` to the `CONTEXTS` object:

```typescript
NewBrand: {
  brand: "NewBrand",
  sourceClaimedCount: 123,           // if source emits one
  authoritativeCount: 100,           // from public research
  authoritativeCitation: "...",
  tolerancePct: 15,
  expectedCities: ["Mumbai", "Delhi", ...],
  sourceDomain: "api.newbrand.com",
  officialDomain: "newbrand.com",
  siblingBrandsProbed: [
    { name: "RelatedBrand", expectedCount: 0, extracted: false },
  ],
  authenticitySignals: {
    hasWikipediaArticle: true,
    publicCompany: false,
    knownBrand: true,
    domainMatchesBrand: true,
  },
},
```

3. Add a row to `BRAND_FILES`:
   ```typescript
   NewBrand: "newbrand-stores.jsonl"
   ```

4. Run: `bunx vinxi-scraper run validate-retail`

Output: `data/reports/newbrand-scorecard.md` + entry in `retail-summary.md`.

## When to update the `authoritativeCount`

This is a manually-curated number from public sources (Wikipedia infobox, annual report, news). It MUST be updated when:

- The brand publishes a quarterly/annual report with a fresh store count
- Press reports significant openings or closures
- The brand crosses a milestone (e.g. "Zudio crosses 600 stores")

A stale authoritative count produces false positives ("scrape is wrong, count is low!") when actually the scrape is fine and the authoritative number is the one out of date. Cite the source date in `authoritativeCitation`.

## What the bands mean

- **High (≥80)** — Ship. Run as a scheduled job. Trust the data for decisions.
- **Medium (60-79)** — Use with caution. Cite the gaps when sharing. Don't compute averages across all brands at this band.
- **Low (<60)** — Don't ship. Re-recon. The scorecard's recommendations are the prioritised work list.

## Real-world catches this session

Concrete examples of the validator finding things we couldn't have caught manually:

1. **Zudio sibling-brand missing** — Westside completeness scored 100, but the "no sibling-brand probe performed" warning surfaced. One curl revealed Zudio with 578 stores. Without the validator, we'd have shipped Trent's portfolio at 30% of its actual coverage.

2. **Zudio 0% lat/lng** — Data quality flagged `has-coords: 1/555 (0%)`. Visually inspecting 555 records to notice 99% are missing geo would have taken hours. The validator found it in milliseconds. The recommendation pointed us at `google_emmbeded_code` recovery, which restored 99%.

3. **Westside 1 store out of India bounds** — `coords-in-bounds` flag caught a single GPS-error record. Not material for analysis but matters for downstream geo joins.

4. **Westside data quality regressed mid-test** — A flaky API response dropped Karnataka, taking the count from 300 to 266. The validator immediately scored completeness 89 (down from 100). Re-running the scraper restored Karnataka and the score.

5. **Westside "Tamilnadu" vs "Tamil Nadu" inconsistency** — Per-state distribution audit (a manual offshoot of the metro-coverage check) found 1 store labelled with the no-space spelling. Source data quality issue that would otherwise have silently fragmented the Tamil Nadu state aggregate.

## Anti-patterns

- **Skipping the scorecard because "the scrape ran fine"** — Running != correct. The scrape that produced 555 Zudio records was running fine the whole time it was extracting 0% lat/lng.
- **Adjusting `tolerancePct` until the score passes** — If a 15% tolerance fails, the answer is to investigate, not to widen tolerance to 30%. The exception: small-N brands (Mango at 9-15 stores) genuinely warrant higher relative tolerance.
- **Treating authoritative count as ground truth** — It's also a measurement, with its own staleness. When source and authoritative disagree by 30%, both could be wrong. Cross-reference with the brand's social media or recent press.
- **Running validation only once** — Run it after every re-scrape. Drift signals: yesterday's scrape was 300, today's was 266 → API is flaky, not the scraper.
