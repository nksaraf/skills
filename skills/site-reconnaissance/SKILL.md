---
name: site-reconnaissance
description: Observe a website, figure out how to scrape it, write a Playwright/HTTP scraper using defineScraper(). Use when asked to scrape a site, extract data from a website, or build a scraper for listings/catalogs/stats.
triggers:
  - scrape
  - scraper
  - extract data from
  - reconnaissance
  - recon
  - crawl
  - site investigation
  - website data
  - listings scraper
  - catalog scraper
---

# Site Reconnaissance

**Goal:** go from "here's a URL" to "here's a working `scraper.ts` + fixture + tests" — systematically.

## Setup

This skill drives the [`@vinxi/scraper`](https://www.npmjs.com/package/@vinxi/scraper) runtime (Bun). If the project doesn't have it yet:

```bash
bun add @vinxi/scraper
bun add -d playwright && bunx playwright install chromium   # for browser-tier sites
```

Conventions in your project: scrapers live in `scrapers/<site>.ts`, output streams to `data/results/<site>/latest.jsonl` (each run is timestamped; `latest.jsonl` points at the newest successful run). Run with `bunx vinxi-scraper run <site>`, inspect with `bunx vinxi-scraper export <site> --format json`. The recon helpers (`walkSchema`, `startXhrCapture`, `auditPage`, `scoreCard`, …) import from `@vinxi/scraper/recon`.

## First: check the case library

Before running anything, look up the site in `sites/INDEX.md`. If we've reconned it before, the per-site addendum has the URL patterns, hydration payload paths, protection signals, and links to the existing extractor + fixture. **Always read the addendum if it exists — don't re-discover what we already know.**

If the site is new, run the recon below, then **write a new addendum** at the end. This is how the skill learns.

## ⚠ Critical: the recon-extract loop

**This is the single most important framing.** Recon and extraction are NOT a linear pipeline. They are a tight loop that runs 3-4 times before you ship.

Pure recon — "looking at one page" — is incomplete by construction. Variation, missed fields, and bad heuristics surface ONLY when you write the extractor and run it on multiple samples. Don't promise a "complete extractor by phase 5d". Promise a draft. **Iterate.**

See `reference/recon-extract-loop.md` for the framing in detail. The phases below describe a single turn of the loop, not the whole journey.

---

## Methodology — six phases with quality gates

### Phase 0a — Enumerate page types

List every page type the scraper will need to touch. Most data sites are built from a few recurring archetypes — identify which ones your target has:

- **Collection / index** (search results, category, `?page=N`, `/products.json`) — summary fields; the entry point for enumeration
- **Detail** (path with a stable per-item ID, e.g. `/products/<handle>`, `/p/`, `spid-`) — the full record
- **Locator** (store / branch / dealer finder — usually a map plus a geo or bbox API) — addresses, hours, coordinates
- **Profile** (seller, brand, host) — inventory count, other items by the same entity

Not every site has all four: a catalog may be just collection + detail; a store finder may be just a locator.

**Rule:** if collection URLs expose a stable per-item ID, **there's a detail page — visit it**.

Then run Phases 0-5 once per page type.

### Phase 0 — HTTP assessment

One `curl` per page type with realistic Chrome headers (UA, an `Accept-Language` matching the target's region, `Sec-Fetch-*`). Record:
- HTTP status, server, set-cookie
- Framework signature (see `reference/framework-signatures.md`)
- Whether all target data is in the raw HTML (search for known values)
- Protection signals (403/503, Cloudflare/Akamai/Imperva headers, challenge HTML)

> **Gate A:** All target data points in raw HTML AND no protection signals? → skip to Phase 3.

### Phase 1 — Browser recon (if Gate A failed)

Launch stealth Playwright (`ctx.browser.newPage()` is stealth-by-default; locale defaults to `en-IN` / `Asia/Kolkata` — pass `{ locale, timezoneId }` to match the target's region). Capture:
- Rendered DOM after JS executes
- Network XHR traffic — JSON APIs at `/api/`, `/graphql`, `/_next/data/`
- Auth headers visible on those calls

### Phase 1c — XHR / network capture (if no payload in HTML)

If `extractAllHydrationPayloads()` returns nothing useful AND data only appears after page load, the site is XHR-only. Use `startXhrCapture()` from `@vinxi/scraper/recon` to intercept JSON (and HTML-partial) responses BEFORE navigating.

```typescript
import { startXhrCapture, capturedToPayloads } from "@vinxi/scraper/recon"

const capture = await startXhrCapture(page, { urlMatcher: /\/api\// })
await page.goto(targetUrl, { waitUntil: "networkidle" })
await ctx.human.pause(2000, 4000)
const responses = capture.stop()
for (const p of capturedToPayloads(responses)) {
  // p.data — the parsed JSON (feed it to walkSchema)
  // p.url  — the FULL request URL incl. api_key + params. KEEP THIS: it's what
  //          lets you replay the call with ctx.fetch and drop the browser
  //          (the Tier 3 → Tier 2 downgrade). p.source is truncated for display.
}
```

> **Tip:** filter out third-party map/tile vendors (`mapbox`, `googleapis`, `tile`, `cdn`) before ranking endpoints — they carry `address`-shaped keys (sprite names, layer labels) and produce false positives in the geo/addr heuristics.

See `patterns/xhr-and-partial-html.md` for the full recipe, the protected-site warmup pattern, and how to downgrade from Tier 3 (browser) to Tier 2 (direct API call) once you've identified the canonical endpoint.

> **Gate B:** All data points now covered (raw HTML + rendered DOM + traffic)? → skip to Phase 3.

### Phase 2 — Deep scan (only for missing data points)

Trigger dynamic loads (scroll, click filters, dropdowns). Watch new traffic. Test pagination.

### Phase 3 — Validate every extraction method

For each claimed selector / JSON path / API endpoint, run an actual extraction test against the captured HTML. If a selector doesn't match the expected count, you don't have the field — go back.

### Phase 4 — Protection profile (conditional)

Run only if you saw protection signals, or the site runs a known anti-bot stack (Akamai, Imperva, Cloudflare, DataDome, PerimeterX). See `patterns/protection-profiles.md` for the stacks we've profiled and `sites/INDEX.md` for specific sites flagged high-protection.

Escalate: Raw HTTP → Stealth browser → Residential proxy → TLS fingerprint spoofing. Stop when access is confirmed.

### Phase 5 — Build + validate

**5a — Reconnaissance report** with the findings table.

**5b — Self-critique** — gaps, skipped steps, unvalidated claims, staleness risks.

**5c — Write `scrapers/<site>-test.ts`** using `defineScraper()`. Single-page smoke test only. Then write `scrapers/<site>.ts` for the full crawl.

**5d — Run the test scraper.** Inspect output via `bunx vinxi-scraper export <site>-test --format json`. Iterate until correct.

**5e — Visual ground truth** (this is the step that catches what 5d missed). See `Phase 5e recipe` below.

**5f — Schema walk** (this is the step that catches what 5e missed). See `Phase 5f recipe` below.

**5g — Save a fixture + write schema-locked tests** so a future schema change fails loudly. Then write the site addendum.

> **Gate D:** After 5e + 5f + 5g, do you have (1) per-site extractor, (2) HTML fixture, (3) schema-locked tests, (4) updated `sites/<site>.md`? If yes — proceed to Phase 6.

### Phase 6 — Validation scorecard (MANDATORY before declaring done)

A scrape that ran without errors is not the same as a scrape with good data. Phase 6 produces a numeric scorecard across four dimensions: **Completeness**, **Data Quality**, **Source Authenticity**, **Freshness**. Each is 0-100 with weighted evidence.

> **Scope:** `scoreCard()` is for **store-locator / geo** cohorts — it scores coordinate coverage, address completeness, metro spread. Build inputs with `makeRetailStore()` from `@vinxi/scraper/retail`. A **product catalog** (SKUs, prices, variants) has no geographic scorecard — validate it instead by sampling field coverage (% with price, % with barcode, in-stock %) and reconciling the count against the source's own total. Phase 6 / Gate E still applies in spirit (don't ship un-validated data) — just with catalog-appropriate checks.

See `patterns/validation-scorecard.md` for the full pattern and `reference/recon-extract-loop.md` for why this isn't optional. Specific things the scorecard has caught in past sessions:

- A multi-brand parent corporation hiding a sibling brand at the same API (Westside scored 100 completeness but the scorecard flagged "no sibling probe done" — Zudio with 578 stores was hiding behind `?type=zudio`)
- 0% lat/lng on 555 records that visual inspection missed
- A flaky API run that silently dropped 34 Karnataka stores (300 → 266)
- A "Tamilnadu" vs "Tamil Nadu" data quality bug in source data

How to run it: write a small validator scraper in your project (e.g. `scrapers/validate-<domain>.ts`) that loads your scraped records, calls `scoreCard()` from `@vinxi/scraper/recon` per brand/site, and writes `formatScoreCard()` output to `data/reports/`:

```bash
bunx vinxi-scraper run validate-<domain>
# → produces data/reports/<brand>-scorecard.md per brand
# → produces data/reports/<domain>-summary.md cross-brand
```

When you add a new brand or site, register a `BrandContext` (or equivalent) with:
- `sourceClaimedCount` — the source's own published count, if any
- `authoritativeCount` + `authoritativeCitation` — public reference (Wikipedia / press / annual report)
- `expectedCities` — places the brand definitely has stores
- `siblingBrandsProbed` — multi-brand parents (see `patterns/sibling-brand-probing.md`)
- `authenticitySignals` — manually-asserted public facts

> **Gate E:** Scorecard produced and **overall band is "high" (≥80)** OR the gaps are documented and accepted. If band is "medium" or "low", iterate on Phase 5d/5e/5f and re-run.

## Recipe — Phase 5e (visual ground truth)

The recon-scaffolding tools live in `@vinxi/scraper/recon` — they help you SEE the page. They are noisy on purpose. The agent reads the output + screenshots and writes the per-site extractor.

```typescript
import { auditPage, findMissingFields } from "@vinxi/scraper/recon"

const page = await ctx.browser.newPage({
  viewport: { width: 1280, height: 900 },
  deviceScaleFactor: 1, // keep PNGs under the 2576px image-viewer cap
})
await page.goto(url, { waitUntil: "domcontentloaded" })
await ctx.human.moveMouse(page)
await ctx.human.pause(2000, 3000)

const audit = await auditPage(page, { screenshotPrefix: "/tmp/recon-<site>-" })
ctx.log.info("audit", {
  headings: audit.headings.length,
  pairs: audit.labelValuePairs.length,
  badges: audit.statusBadges,
  detailLinks: audit.detailLinks.slice(0, 5),
})
```

Then **read each screenshot** (`/tmp/recon-<site>-1.png`, `-2.png`, `-3.png`) and compare to the scraped record. Anything visible but not extracted goes into the next iteration.

## Recipe — Phase 5f (schema walk)

When a hydration payload exists, walk every leaf before committing to a few:

```typescript
import { extractAllHydrationPayloads, walkSchema, filterLeaves } from "@vinxi/scraper/recon"

const all = extractAllHydrationPayloads(html)
// Each entry has { source, data } — could be window.X, JSON-LD, __NEXT_DATA__, etc.
for (const p of all) {
  const leaves = walkSchema(p.data, { arraySampleSize: 2, maxDepth: 8 })
  // Bucket by the field families YOUR target should expose — adapt these keywords
  // to the domain (catalog, locator, listings, …).
  ctx.log.info(p.source, {
    total: leaves.length,
    pricing: filterLeaves(leaves, ["price","amount","cost","currency"]).length,
    geo: filterLeaves(leaves, ["lat","lng","latitude","longitude","coordinate"]).length,
    identity: filterLeaves(leaves, ["id","sku","name","title","handle"]).length,
    meta: filterLeaves(leaves, ["available","status","stock","rating","updated"]).length,
  })
}
```

Use the bucket counts as a navigation tool: if a bucket you expect is 0 you missed a field type; if `pricing` is 30 you should pick the canonical one.

## What goes in the site addendum

When you finish recon on a site, write `sites/<site>.md` using the template in `sites/INDEX.md`. Future recon (yours or someone else's) starts there.

Minimum to capture:
- URL patterns (collection, detail, locator, others)
- Framework + tier
- Hydration payload location + key paths
- Protection profile
- Field inventory (what's extractable)
- Pointers to extractor + fixture + tests in this repo
- Gotchas (the specific things that bit you)

## Tier selection — quick reference

| Tier | Transport | When to use |
|---|---|---|
| 1 | `ctx.fetch` + cheerio | All data in raw HTML, no protection |
| 2 | `ctx.fetch` (JSON API) | Clean API in traffic, data cleaner than HTML |
| 3 | `ctx.browser` (Playwright) | SPA, anti-bot, JS-only data |

Prefer Tier 1 > 2 > 3. Never browser if HTTP works. **But:** sites with Akamai/Imperva/Cloudflare protection require Tier 3 even when the data is in the SSR'd HTML.

## Cross-references

- **`reference/recon-extract-loop.md`** — **READ FIRST** — how to think about recon and extraction as a tight loop, not a pipeline
- **`sites/INDEX.md`** — every site we've reconned, with framework + tier + last-verified date
- **`patterns/`** — abstracted observations (hydration shapes, label/value DOM conventions, protection profiles, XHR-driven SPAs, sibling-brand probing, embed-coord recovery, validation scorecard)
- **`reference/framework-signatures.md`** — header + HTML signatures by framework
- **`reference/domain-hints-india-realestate.md`** — site-specific starting points for Indian real estate
- **`reference/retail-store-locator-schema.md`** — canonical retail-store schema (cross-brand)
- **`reference/tier-selection.md`** — Tier 1/2/3 decision rules
- **`reference/common-pitfalls.md`** — gotchas we've actually hit

## Core principle

The scaffolding tools (`@vinxi/scraper/recon`) help you understand a site fast. They will never be perfect on every site — and they don't have to be. **The deliverable is a per-site extractor + fixture + tests + addendum**, not a generic extractor that works everywhere.

If you find yourself adding another branch to `visualAudit.ts` to handle one more site, stop. Write a 50-line site-specific extractor instead.
