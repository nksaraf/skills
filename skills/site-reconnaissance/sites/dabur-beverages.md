# Dabur (Real / Hajmola)

**Domain:** dabur.com
**Vertical:** FMCG / beverages + ayurvedic-wellness B2B contact + factory list
**Last verified:** 2026-05-21
**Tier:** 1 (raw HTTP + HTML structured-div parse)
**Framework:** Drupal / Pimcore (Drupal "views" templates emit `views-row`/`views-field-*` blocks)
**Protection:** none (CloudFront, plain Chrome UA passes; only SEO-tool UAs are robots-blocked)

## Context

Brief said "Dabur Beverages" with siblings `[real, hajmola, vatika, dabur-honey, fem]`.
Dabur does NOT operate the beverage line as a separate subsidiary with its
own factories or distributor finder. Real / Hajmola / Vatika / Chyawanprash
all share a unified Dabur India manufacturing footprint, and the corporate
site publishes the full B2B locator at one URL: `/contact-information`.

That page is the only structured locator Dabur ships. There is NO consumer
store finder (typical for an FMCG selling through general trade). The page
covers 58 structured rows across six tabs and is the canonical answer to
"where does Dabur operate."

> **Lesson:** for an FMCG brand with no consumer locator, the parent
> company's `/contact-information` or `/contact-us` is almost always where
> the B2B footprint hides — factories, regional sales offices, overseas
> entities. Probe corporate before declaring "parked".

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| Contact / locator | `/contact-information` | https://www.dabur.com/contact-information |
| Sitemap parent | `/sitemap.xml` | https://www.dabur.com/sitemap.xml |
| Sitemap (contact) | `/sitemap_contact.xml` | https://www.dabur.com/sitemap_contact.xml |

## Hydration payload

None — server-rendered HTML. The page contains a `quicktabs` widget with
six tab panels:

| Tab | Index | Layout | Rows |
|---|---|---|---|
| Addresses | 0 | `wtu-right` blocks with `<h3>Office</h3>` + `wtu-list`/`wtu-item`/`wtu-label`/`wtu-val` pairs | 2 (Corporate + Registered) |
| Branch Offices | 1 | Drupal view: `views-row` * N | 14 |
| Factories | 2 | Drupal view | 29 |
| Overseas Offices | 3 | Drupal view | 10 |
| Subsidiaries | 4 | Drupal view | 2 (DRDC + H&B Stores) |
| Contact head - Private labels | 5 | Drupal view | 1 |

Inside each `views-row`:

```html
<div class="views-row">
  <div class="views-field views-field-title"><span class="field-content">Sahibabad Unit I &amp; II</h2></span></div>
  <div class="views-field views-field-body"><div class="field-content"><p>Plot No.22, Site-IV<br />Sahibabad, Ghaziabad</p></div></div>
  <div class="views-field views-field-field-mobile"><div class="field-content">0120 - 4378400</div></div>
  <div class="views-field views-field-field-phone"><div class="field-content"></div></div>
  <div class="views-field views-field-field-fax"><div class="field-content">0120 - 4376924</div></div>
  <div class="views-field views-field-field-email"><div class="field-content"></div></div>
</div>
```

(Note the stray `</h2>` without an opening tag — a known Drupal view-template
quirk. The title regex anchors on `</span>` and ignores the dangling h2.)

## Extraction strategy

Tier 1. One fetch on `/contact-information` (~244 KB). Parse:

1. `sliceTabs(html)` — split the document into the six tab sections by
   anchor on `id="quicktabs-tabpage-contact_information_tab-N"`.
2. Tab 0 → `parseAddressesTab` — `<h3>Office</h3>` + `wtu-label`/`wtu-val` pairs.
3. Tabs 1-5 → `parseTabRows` — `<div class="views-row">` * N, with
   `views-field-{title,body,field-mobile,field-phone,field-fax,field-email}`.

Per the brief: "If plant list → warehouse. If distributor finder →
distributor." Dabur has BOTH on one page; we assign per tab:

| Tab | locationType | storeType |
|---|---|---|
| Factories | warehouse | factory |
| Branch Offices | distributor | branch-office |
| Overseas Offices | distributor | overseas-office |
| Subsidiaries | distributor | subsidiary |
| Addresses | distributor | corporate-office |
| Private Labels | distributor | private-label |

`storeId` is synthesised: `dabur-{tabIndex}-{slug(title)}` — the source
emits no UUIDs. This keeps records stable across runs so long as Dabur
doesn't reorder rows.

## Field inventory

| Field | Source | Notes |
|---|---|---|
| brand | hard-coded "Dabur" | |
| storeId | synthesised from title | not source-provided |
| name | views-field-title | trailing `</h2>` quirk handled |
| address.line1 | first line of views-field-body | |
| address.line2 | remaining lines joined | |
| address.city | first segment of title (cut at " - " / "Unit" / "Office") | "Corporate Office" / "Registered Office" become city="Corporate"/"Registered" — known noise on 2 rows |
| address.state | regex over body — recognises "(HP)", "(MP)", "(Dadra & Nagar Havelli)", "Bari Brahmana (Jammu)" | many factory bodies omit state (Sahibabad / Pant Nagar / Newai) → null is the correct answer for those |
| address.postalCode | 6-digit regex over body | source uses both "454774" (compact) and "700 029" (space-separated) — only compact form is matched, since the spaced form collides with phone numbers |
| address.country | "IN" by default; tab 3 (Overseas) resolved to ISO-3166 via title/body match | EG/NG/US/NP recognised, easily extended |
| phone | first non-empty among views-field-field-{mobile,phone} | source mixes both fields |
| email | regex match in views-field-field-email | |
| fax | preserved in `features: ["fax:<num>"]` (no canonical fax slot) | |
| lat / lng / hours | n/a | source ships neither |

## Pagination

None — single page, all six tabs in one HTML response.

## Files in this repo

- **Extractor + runner:** `scrapers/fmcg/dabur-beverages.ts` (one file, mirrors the VBL pattern)
- **Fixture:** `__tests__/fixtures/dabur-beverages-contact.html` (244 KB, captured 2026-05-21)
- **Tests:** `__tests__/dabur-beverages-stores.test.ts` (14 tests, 100 assertions)
- **Output:** `data/results/dabur-beverages-stores.jsonl` (58 records)
- **Snapshot:** `data/snapshots/fmcg/dabur/{TS}.jsonl` (Phase D pipeline)

## Scorecard (Phase D)

Overall **79/100 (medium)**:

- Completeness 95 — source publishes no count, treat extracted ≈ source; all 6 major metros covered.
- Data quality 47 — no lat/lng (-30), no opening hours (-15), no native storeId (-10), some Sahibabad/Pant Nagar bodies lack explicit state. Source does not emit these.
- Source authenticity 99 — Dabur India Ltd (BSE/NSE listed), Wikipedia, domain matches brand.
- Freshness 75 — scraped today; sitemap_contact.xml lastmod is 2025-11-10.

The medium band is fully driven by the source not publishing coords/hours
— there's no extractor change that improves this without an external geocode pass.

## Gotchas

- **wp-json fails.** Per the VBL pattern, we tried `/wp-json/wp/v2/pages`
  first — Dabur is Drupal, not WordPress, so it 404s. The
  `sitemap_contact.xml` led to `/contact-information` directly.
- **Probe-by-guess URLs all 404.** `/our-presence`, `/manufacturing`,
  `/plants`, `/locations` all 404 (and the site serves a 168 KB pretty
  404 page that LOOKS like a real response if you only check
  `HTTP < 400`). Always check the URL is on a sitemap before parsing.
- **Stray `</h2>` in views-field-title.** The Drupal template emits the
  closing `</h2>` without an opening tag. The title regex anchors on
  `</span>` (the wrapping span) and ignores the dangling close.
- **Tab 0 layout differs from Tabs 1-5.** Corporate + Registered offices
  use a `wtu-list` / `wtu-label` / `wtu-val` structure, not `views-row`.
  Special-cased in `parseAddressesTab`.
- **Tab 0 office capture must not terminate on `</div></div></div></div>`.**
  An earlier draft used that as the office terminator and lost the Email
  ID block (the last `wtu-item` ends with a series of close-divs that
  matched the terminator early). Solved by using only `<h3>` and end-of-section
  as terminators.
- **Sahibabad / Pant Nagar / Newai bodies omit the state.** The body for
  Sahibabad reads `"Plot No.22, Site-IV\nSahibabad, Ghaziabad"` — no
  "Uttar Pradesh" token. `extractState` correctly returns null; we don't
  fabricate the state.
- **Spaced PIN codes ("Calcutta – 700 029") are NOT matched.** A regex
  that accepts spaces inside the 6-digit run collides with phone numbers
  ("(0120) 3962100"). Compact PINs are matched (`454774`); spaced PINs
  are sacrificed to keep phone numbers out of postalCode. Affects ~5 rows.
- **"Corporate Office" / "Registered Office" → city="Corporate"/"Registered".**
  `deriveCity` cuts at " Office", so single-token-named offices lose the
  city. Known noise on 2 rows; the address line1 still has the full
  content. Acceptable.
- **Page row count > site count.** Dabur's annual report cites ~20
  manufacturing sites; the page lists 29 factory units because Sahibabad
  is split into I&II + III, Silvassa into I&II × 2, and the Baddi campus
  is enumerated as 12 product-line units (Tooth Powder, Honey,
  Chyawanprash, Glucose, Honitus, Skin Care, etc.). The page granularity
  is per-unit, not per-site — this is faithful, not duplicated.

## What I tried that didn't work

1. `/our-presence`, `/manufacturing`, `/plants`, `/locations` — all 404
   (with 168 KB pretty error pages).
2. `/wp-json/wp/v2/pages` — Dabur is Drupal, 404s.
3. Considered scraping only the factories tab to honour the "Dabur
   Beverages" framing — rejected because Real isn't manufactured at a
   beverage-specific subset of factories; every Dabur unit is shared
   across product lines. Capturing the full page is the honest scrape.
