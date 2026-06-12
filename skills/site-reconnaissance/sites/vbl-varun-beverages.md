# Varun Beverages (VBL)

**Domain:** varunbeverages.com
**Vertical:** sales-distribution / FMCG bottler (PepsiCo franchise)
**Last verified:** 2026-05-21
**Tier:** 1 (raw HTTP + HTML table parse)
**Framework:** WordPress (default theme, exposes `/wp-json/wp/v2/pages` REST)
**Protection:** none

## Context

VBL is PepsiCo's largest franchise bottler. **Purely B2B** — no consumer
store locator. The starting hypothesis (per the recon brief) was that this
would be a "parked" outcome. **The hypothesis was wrong.** A structured plant
list is published at `/plants/`, and the page is NOT linked from the main
navigation (the visible "Plant Tour" menu item points to `/plan-tour/`,
which is an empty stub). The plants page was only discovered via the
WordPress REST API page listing.

> **Lesson:** for WordPress-hosted brand sites, always enumerate
> `wp-json/wp/v2/pages` before declaring "no locator". A small fraction of
> brands publish locator pages that are not in the visible nav.

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| Plants list | `/plants/` | https://www.varunbeverages.com/plants/ |
| Page enumeration | `/wp-json/wp/v2/pages` | https://www.varunbeverages.com/wp-json/wp/v2/pages?per_page=100 |

## Hydration payload

None — server-rendered HTML. One `<table>` with `<thead>` + `<tbody>`. First
body row is an empty layout spacer (skip it).

Columns:

| Col | Field | Notes |
|---|---|---|
| 1 | SL. | Positional; not stable across edits |
| 2 | Plant Name | City + optional unit suffix ("Bangalore - Nelamangla", "Jainpur - Pepsi India", "Varun Beverages Limited Unit-1") |
| 3 | Plant Code | 1-2 letter code (e.g. "AA", "B", "AG", "DT"). **Stable identifier.** |
| 4 | Address | Free-form FSSAI Form-B style. Includes 6-digit PIN. |
| 5 | LIC | 14-digit FSSAI licence number |

## Extraction strategy

Tier 1. One fetch. ~60KB HTML. The address column is the only messy field —
PIN is reliably 6 digits; state matches a fixed Indian-states list; city is
derived from the first segment of `Plant Name` (everything before " - ").

Records emit:
- `locationType: "warehouse"` (canonical schema lacks a "bottling-plant" type)
- `storeType: "bottling-plant"` (free-text discriminator for downstream filters)
- `storeId: <plant code>` (e.g. "AA")
- `features: ["fssai-lic:<14-digit>"]` (traceability across renames)
- `_extra.plantName`, `_extra.rawAddress` (preserved for diffs)

## Field inventory

| Field | Source | Notes |
|---|---|---|
| brand | hard-coded "Varun Beverages" | |
| storeId | cell[2] (Plant Code) | |
| name | "Varun Beverages — " + cell[1] | |
| address.line1 | cell[3] (cleaned) | |
| address.city | first segment of cell[1] | Fallback: `Jamshedpur` for Unit-1/Unit-2 rows |
| address.state | regex over cell[3] against Indian states list | Handles "M.P.", "U.P.", "ORISSA" legacy spellings |
| address.postalCode | 6-digit extract from cell[3] | |
| address.country | "IN" | |
| features | "fssai-lic:" + cell[4] | |
| lat / lng / phone / email / hours / url | n/a | source doesn't ship these |

## Files in this repo

- **Extractor + runner:** `scrapers/sales-distribution/vbl-varun-beverages.ts`
- **Fixture:** `__tests__/fixtures/vbl-varun-beverages-plants.html`
- **Tests:** `__tests__/vbl-varun-beverages-stores.test.ts` (8 tests, 54 assertions)
- **Output:** `data/results/vbl-varun-beverages-stores.jsonl` (39 records)

## Pagination

None — single page, single table.

## Gotchas

- **Main nav misleads.** The site menu has "Plant Tour" pointing to `/plan-tour/`
  (note the typo — `plan-tour` not `plant-tour`), which is an empty stub. The
  real plants page is `/plants/`, only reachable via direct URL or the
  `wp-json` page enumeration.
- **First body row is empty.** The `<tbody>` opens with a `<tr>` of five
  empty `<td>` cells (a layout spacer). The parser drops cells-all-empty
  rows so this is handled.
- **Plant Code "T" and "U" both label "Varun Beverages Limited Unit-1/2"** —
  the plant-name column doesn't include the city for these rows. The city
  fallback uses the address line (both happen to be in Jamshedpur).
- **State spellings are inconsistent.** The source uses "M.P.", "U.P.",
  "ORISSA", "MAHARASHTRA" all mixed in. `extractState()` handles each.
- **Pincode regex must reject phone numbers.** Use a lookbehind for a
  separator (space / comma / dash / colon) and a lookahead `(?![\d])` so a
  10-digit phone string can't be mis-parsed as a PIN.

## What I tried that didn't work

Probed in order: `/contact-us/` (HQ only), `/our-business/` (prose only),
`/plan-tour/` (empty stub), `/vbl-at-a-glance/` (prose, mentions some
cities by name). Was about to mark parked, then enumerated `wp-json` and
found `/plants/`.
