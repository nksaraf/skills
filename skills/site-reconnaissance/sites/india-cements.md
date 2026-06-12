# India Cements

**Domain:** indiacements.co.in
**Vertical:** Sales & distribution (cement) — South India major
**Last verified:** 2026-05-22
**Tier:** 1 (single GET of a JS asset that inlines all data)
**Framework:** Custom legacy PHP + jQuery (Bootstrap 3, owl-carousel, mapit.js) — Microsoft IIS server (`X-Frame-Options: SAMEORIGIN`, ASP-like 404 page)
**Protection:** none (default Chrome UA passes; no Akamai / Cloudflare / Imperva)
**Parent:** The India Cements Ltd — **being acquired by UltraTech Cement** (announced 2024, deal pending FY26)

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| Plants landing | `/plants-locations.html` | https://www.indiacements.co.in/plants-locations.html |
| **Plants data (inline JS)** | `/js/jquery.mapit.js` | https://www.indiacements.co.in/js/jquery.mapit.js — contains `locations: [...]` literal with all 9 plants |
| Dealer locator (parked) | `/dealers-locations.html` | renders "We are in the process of updating the dealer location to serve you better." — the AJAX endpoint `/dealers_action.php` is **404** |
| YAML's documented dealer-locator URL | `/dealer-locator` | **404** — the canonical (still-parked) path is `/dealers-locations.html` |

## Hydration payload

- **Location:** XHR-free — `js/jquery.mapit.js` is fetched by the page and contains the data inline as a JavaScript array literal.
- **Source name:** the variable initialiser `locations: [...]` inside the `$.fn.mapit` plugin's `defaults` block.
- **Schema (positional rows):**
  ```js
  locations: [
    [lat, lng, 'images/marker_red.png', '', '<div class="plantimg ..."><img src="..."/></div><h2>NAME</h2><h5>OPERATOR</h5><p>ADDRESS, Pincode:XXXXXX</p><p>Tel.: <phone><br />Fax: <fax></p><p><strong>CAPACITY</strong> <br />X.XX Million Tonnes (P/A)</p>'],
    ...
  ]
  ```
- **Total rows on 2026-05-22:** 9 plants across 4 states (Tamil Nadu × 4, Andhra Pradesh × 2, Telangana × 2, Rajasthan × 1)
- **Marker icons encode state grouping** — `marker_red.png` = Tamil Nadu, `marker1.png` = Andhra Pradesh, `marker2.png` = Telangana, `marker4.png` = Rajasthan (preserved in `_extra.iconMarker`)

## UltraTech-acquisition signal

The mobile-only header embeds an UltraTech logo at `images/mob-ultratech-logo.jpg` linking to `https://www.ultratechcement.com/corporate` — explicit acquisition recognition. **But the underlying stack has NOT migrated:**

- India Cements still runs the legacy PHP / jQuery stack (IIS, Bootstrap 3) versus UltraTech's Adobe AEM (`/bin/exceltojsonservlet`).
- The dealer locator is **parked at source** ("process of updating") — strong signal of operational pause pending deal closure. The visible state dropdown only lists 3 states (Tamil Nadu, Kerala, Maharashtra) with Chennai as the only city — a stub from the pre-pause UI.
- No India Cements rows have been observed in UltraTech's `storelocator.xlsx` dump as of the 2026-05-21 UltraTech recon.

**Implication for ops:** the public dealer estate will likely collapse into UltraTech's locator on or after deal closure. Until then this brand has plants-only coverage.

## Vendor comparison vs cement peers

| Brand | Vendor | Dealer locator | Plants disclosed publicly |
|---|---|---|---|
| **India Cements** | Custom PHP + jQuery (legacy) | Parked ("updating") | ✅ 9 plants (lat/lng + capacity + phone/fax in jQuery mapit.js) |
| UltraTech | Adobe AEM `exceltojsonservlet` | ✅ 3,956 dealers (xlsx blob) | none in locator |
| Ambuja | Sitecore + Next.js + custom REST (`ambujahelp.in`) | ✅ 10,353 dealers (state API sweep) | none in locator |
| ACC | n/a (no locator) | ❌ stripped (post-Adani) | none |
| Shree Cement | Synup white-label | ✅ 242 dealers (HTML sweep) | none in locator |
| Dalmia Cement | custom pincode API | ✅ pincode-keyed | none in locator |

India Cements is the **only** Indian cement major surfacing plant lat/lng + capacity together publicly. UltraTech and Ambuja both keep plant data investor-side; their public locators are dealer-only.

## Extraction strategy

**Tier 1 — single fetch of `js/jquery.mapit.js`.** The asset is small (~12 KB) and the data is inline. We parse the JS literal directly (not via `eval` — the row contents are single-quoted HTML so we walk the source char-by-char with a single-quote-aware bracket scanner; see `locateLocationsLiteral` + `parseSingleRow`).

The 5th positional cell is HTML used for the marker info-window — we parse it with cheap regexes for `<h2>` (name), `<h5>` (operator), free-text `<p>` (address with pincode), `Tel.:` / `Fax:` `<p>`, and the `<strong>CAPACITY</strong> ... Million Tonnes` `<p>`.

**Why not just visit `/plants-locations.html`?** The page has nothing but `<div id="map_canvas">`. The data lives in `/js/jquery.mapit.js`. Skipping the HTML page saves a request and one round of HTML noise.

**Why not Tier 3 (Playwright)?** No JS execution needed — the source is a static JS file. The page ALSO runs Google Maps which would require an API key; we'd never use the rendered map anyway.

## Field inventory

| Field | Source | Notes |
|---|---|---|
| `brand` | hardcoded "India Cements" | |
| `storeId` | slug of `<h2>` heading | e.g. `sankarnagar-tirunelveli`, `banswara-rajasthan`. Source ships no stable plant code. |
| `name` | `<h2>` text | e.g. "Sankarnagar, Tirunelveli" |
| `address.line1` | first non-Tel/Fax/Capacity `<p>` | Free-text. Banswara is empty (capacity-only marker). |
| `address.city` | first comma-token of name | "Sankarnagar", "Vallur", "Banswara" |
| `address.state` | derived from title + address scan (see `deriveState`) | Title can be "Plant, District" (Tamil Nadu plants) or "Plant, State" (AP/TG/RJ) — district lookup map covers Tirunelveli/Salem/Perambalur/Tiruvallur/Cuddapah/Rangareddy/Nalgonda. |
| `address.postalCode` | regex `Pincode:XXXXXX` (handles stray spaces, e.g. `600 120`) | All 6-digit |
| `lat` / `lng` | positional cells 0 and 1 | India bounds verified |
| `phone` | `Tel.:` `<p>` cell | Landlines, normalised to `+91`-prefixed digit-only |
| `features` | `["capacity:X.XX-mtpa", "operator:The India Cements Limited"]` | Capacity preserved as feature + as numeric `_extra.capacityMtpa` |
| `_extra` | `iconMarker` (state-group hint), `rawAddress`, `capacityMtpa`, `fax`, `titleHint` | Fax kept; not promoted because canonical schema has no fax slot |
| `locationType` | hardcoded `"warehouse"` | Canonical schema lacks `"cement-plant"` — same convention as Parle Agro / VBL |
| `storeType` | `"cement-plant"` | Free-text sub-type |

## Files in this repo

- **Extractor:** `scrapers/sales-distribution/india-cements.ts` (pure functions)
- **Production runner:** `scrapers/sales-distribution/india-cements-run.ts` (live fetch + Phase D pipeline)
- **Fixture:** `__tests__/fixtures/india-cements-mapit.js` (the entire ~12 KB jquery.mapit.js file)
- **Tests:** `__tests__/india-cements-stores.test.ts` (15 tests, 139 expects)
- **Output:** `data/results/india-cements-stores.jsonl` (9 records)

## Pagination

None. Single asset returns everything in one body (~12 KB).

## Gotchas

- The YAML's `official_locator_url: /dealer-locator` is **a 404**. The legacy `/dealers-locations.html` page exists (in the sitemap) but is itself **parked**: the table renders "We are in the process of updating the dealer location to serve you better." and the AJAX endpoint `dealers_action.php` is 404. Likely pause is acquisition-driven. Do not bother sweeping the dealer surface until the deal closes.
- The plants page (`/plants-locations.html`) has no inline data — just `<div id="map_canvas">`. The data lives in `/js/jquery.mapit.js`. Fetch the JS, not the HTML.
- Single-quoted strings inside the JS literal carry embedded `<div>...</div>` markup — `JSON.parse` cannot parse it. Parse manually with a single-quote-aware bracket scanner (`locateLocationsLiteral`).
- Pincode "600 120" (Vallur) has a stray space. `extractPincode` collapses it to "600120".
- Banswara row has no address line — the marker HTML jumps straight from `<h2>` to `<p><strong>CAPACITY</strong>...`. Our extractor produces `address.line1: ""` and still emits the record with state + lat/lng + capacity. Tests assert this.
- Marker icon paths encode state grouping (`marker_red.png` = TN, `marker1.png` = AP, `marker2.png` = TG, `marker4.png` = RJ). We surface this as `_extra.iconMarker` so future ops can cross-check state inference.
- Title field (`<h2>`) is sometimes "City, District" (Tamil Nadu) and sometimes "City, State" (AP/TG/RJ). `deriveState` checks for explicit state names first, then falls back to a district-to-state lookup. **If India Cements adds a plant in a district we haven't seeded, `state` may be `null`** — add the district to `districtMap` in `deriveState`.
- The site's mobile header embeds the UltraTech logo (acquisition signal). This is the ONLY visible migration artefact at recon time. The technical stack is otherwise still the legacy custom site.
- Total disclosed capacity across the 9 plants is 14.45 mtpa as of 2026-05-22 (test asserts this). India Cements' annual report cites a wider ~15.5 mtpa nameplate including grinding units — the 1+ mtpa delta is grinding-only or RMC sites that don't appear on the plants map.
