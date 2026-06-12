# Ather Energy

**Domain:** atherenergy.com
**Vertical:** India electric two-wheeler manufacturer — dealer + service-centre locator
**Last verified:** 2026-05-21
**Tier:** 3 (stealth Playwright — Cloudflare WAF blocks all direct HTTP)
**Framework:** Next.js (page is `__NEXT_DATA__`-shaped but the dealer data is NOT in __NEXT_DATA__; it's inlined further down as a Mapbox-seed GeoJSON)
**Protection:** Cloudflare WAF — blackholes plain `curl` / `bun fetch` (403 regardless of UA, robots-allowed bot UAs, even with `Sec-Fetch-*` headers). Stealth Playwright passes.

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| Locator (single page — every touchpoint inline) | `/locate-ather-dealer` | `https://www.atherenergy.com/locate-ather-dealer` |
| Charging-network home | `/charging` | (also rendered with the same Grid GeoJSON inlined) |

There is **no per-city URL** and **no JSON API** behind the locator. The page renders one giant FeatureCollection covering every dealer in the country. City selection is purely a client-side filter over the inlined data.

## Hydration payload

The rendered HTML contains TWO `"features":[` arrays (in document order):

1. **First** — Ather Grid charging stations (~4,393 features). Properties: `{ Name, Area, Type }` where `Type ∈ {"Ather Space", "Hotel", "Cafe", "Retail", "Mall", ...}`. **Not extracted** (charging infrastructure is a separate domain; tracked here in `_extra` only).

2. **Second** — Dealer touchpoints (~1,115 features). Properties:
   - `branchId` (string, NOT unique — ~64% unique rate; ATM-style "branch code" for the dealer's POS system)
   - `branchType` ∈ `"SALES"` / `"SERVICE"` / `"SALES_AND_SERVICE"`
   - `name`, `address`, `pincode`, `contact_no`, optional `email`
   - `map_location` — Google Maps deep link
   - `city` — full Strapi city object with `cityName`, `state`, `latitude`, `longitude` (city-centroid, not the dealer's)
   - `timings.days[]` — per-day `{startTime, endTime, closed}` records (occasionally a day appears twice — first-wins)
   - `isGoldServiceCenter`, `book` (test-ride bookable flag)
   - `geometry.coordinates: [lng, lat]` — the dealer's actual coordinates (NOT the city centroid above)
   - `id` (top-level GeoJSON feature id) — **the actual unique key**, 1,115/1,115 unique

## Extraction strategy

Tier 3 only — Cloudflare WAF blackholes plain HTTP including `Accept: */*` + Chrome UA + `Sec-Ch-Ua-*` headers + Googlebot UA. Stealth Playwright passes; the FeatureCollection is fully populated client-side immediately after `domcontentloaded`, so we DON'T wait for `networkidle` (the page has long-poll trackers that prevent idle).

Once rendered, slice the SECOND `"features":[` JSON array out of the HTML using bracket-balanced parsing (we ignore brackets inside strings). One page render gives every touchpoint in the country.

## Field inventory

| Field | Source | Notes |
|---|---|---|
| storeId | feature `id` (top level) | unique across all 1,115 records |
| branchId | properties.branchId | **NOT unique** (~64%) — preserved in `_extra` as the dealer's POS code |
| locationType | derived from branchType | SALES → store; SERVICE → service-centre; SALES_AND_SERVICE → store + storeType="Experience & Service Centre" |
| name | properties.name | usually a road/locality, occasionally "Ather Service {City}" |
| address.line1 | properties.address | comma-separated full address |
| address.city | properties.city.cityName | populated 100% |
| address.state | properties.city.state — normalised | source has whitespace/case variants ("Tamilnadu", "Maharahstra", " Maharashtra") — normalised to canonical |
| address.postalCode | properties.pincode | 99% |
| address.country | derived from state | Nepal provinces (Bagmati, Gandaki) → NP; Sri Lanka (Western) → LK; else IN |
| lat / lng | geometry.coordinates [1] / [0] | 100% populated; 100% inside India bounds for IN-tagged records |
| phone | properties.contact_no | 100% populated |
| email | properties.email | ~rare (a handful of international service centres) |
| url | properties.map_location | Google Maps deep link; 99.9% populated |
| hours | properties.timings.days | per-day; supports `closed: true` |
| features | derived | "Gold Service Center" if isGoldServiceCenter; "Test Ride Booking" if book=="true" |
| `_extra.branchId` / `branchType` / `city` / `timings` / `isGoldServiceCenter` / `book` | preserved verbatim | dedupe-by-branchId would be wrong (many SERVICE rows share branchId "0"); use storeId |

## Files in this repo

- **Extractor:** `scrapers/sales-distribution/ather-energy.ts` (pure-function library)
- **Production runner:** `scrapers/sales-distribution/ather-energy-run.ts` (stealth Playwright + scorecard + snapshot + alerts)
- **Fixtures:**
  - `__tests__/fixtures/ather-locator-sample.html` — synthetic page with two `features:[` arrays
  - `__tests__/fixtures/ather-dealers-sample.json` — 6 raw records (2 SALES, 2 SERVICE, 1 SALES_AND_SERVICE, 1 Nepal)
- **Tests:** `__tests__/ather-energy-stores.test.ts` (10 tests, 94 expect() — all pass)
- **JSONL output:** `data/results/ather-energy-stores.jsonl` (1,115 records, 100% city/state/phone/coords)

## Pagination

None. Single rendered page covers the entire country.

## Counts (2026-05-21 capture)

| Bucket | Count |
|---|---|
| SALES (experience centre) | 629 |
| SERVICE (service centre) | 478 |
| SALES_AND_SERVICE (dual-purpose) | 8 |
| **Total dealer touchpoints** | **1,115** |
| Ather Grid charging features (NOT extracted) | 4,393 (per page) — site headline says 5,900+ |

The YAML authoritative count (250) is from the FY24 DRHP — known stale by ~4×. The site itself shows headline counters: "7007+ Ather Touchpoints / 629+ / 478+ / 5900+" — these match exactly.

## Gotchas

- **Cloudflare WAF on every path**, including `robots.txt`-allowed bot UAs (`ClaudeBot`, `GPTBot`, `Googlebot`). Only stealth Playwright passes. `robots.txt` and `llms.txt` ARE retrievable; everything else 403s.
- `waitUntil: "networkidle"` times out (analytics long-poll). Use `domcontentloaded` + poll for `match(/"features":\[/g).length >= 2`.
- TWO `features:[` arrays in the HTML — first is Grid charging, second is dealers. Order has been stable across captures but the extractor falls back to a scan-until-branchType-present strategy if order ever shifts.
- `branchId` is NOT unique (many service-centre rows share "0" or duplicate the sales-twin's code). Use the GeoJSON `id` field for dedupe.
- State names have whitespace/case/spelling variants — Maharashtra appears as "Maharashtra", " Maharashtra", "Maharashtra ", "Maharahstra", "Maarashtra". We normalise.
- 4 records are non-Indian (Nepal x3, Sri Lanka x1) — they have `lat/lng` outside India bounds and country derives correctly from state. Don't filter on India-bounds before checking country.
- `timings.days` sometimes has a day-row with `day: null` — guard.
- One record in the Nov-2026 capture had `phone` populated but `branchId="0"` (Nepal/SL touchpoints).

## What we did NOT extract

- **Ather Grid charging stations** (~5,900+ per site headline, 4,393 in the GeoJSON we captured) — different domain (public fast-charging infra, not dealer touchpoints). The YAML notes allow recording these as `pickup-point` but the task asked to keep them in `_extra` notes only; we left them out.
