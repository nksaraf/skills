# Shree Cement

**Domain:** shreecement.com (locator on `dealers.shreecement.com`)
**Vertical:** Sales & distribution (cement dealers — India's 2nd-largest cement producer)
**Last verified:** 2026-05-21
**Tier:** 1 (state-page HTML sweep — every field surfaces as hidden inputs in the listing markup)
**Framework:** Synup "EnterprisesV2" / `EThemeForMasterGrid` white-label storelocator (customerId `435076`)
**Protection:** none observed (no Cloudflare / Akamai / WAF on the subdomain; default Chrome UA passes)
**Parent:** Shree Cement Ltd (listed)

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| Locator landing | `https://dealers.shreecement.com/` | (lists 12 state cards) |
| State page 1 | `/location/{state-slug}` | https://dealers.shreecement.com/location/rajasthan |
| State page N | `/location/{state-slug}?page=N` | https://dealers.shreecement.com/location/rajasthan?page=11 |
| Dealer detail | `/{long-slug}-{outletId}/Home` | https://dealers.shreecement.com/shree-cement-tanya-traders-itarana-circle-cement-supplier-itarana-circle-alwar-473942/Home |

The brand's primary domain `www.shreecement.com` has a 404 at `/dealer-locator` (the YAML's `official_locator_url` is stale). The actual locator lives on the `dealers.` subdomain; the brand homepage links to it as "Find Shree Cement Shop Near You" → `https://dealers.shreecement.com/`.

## Hydration payload

- **Location:** server-rendered HTML — no XHR / no SPA. Every dealer card on a state listing page ships its key data inline as hidden `<input>` elements:
  ```html
  <input class="outlet-latitude"  value="27.538959">
  <input class="outlet-longitude" value="76.628782">
  <input class="business_city"    value="Alwar">
  <input class="business_name"    value="Shree Cement - Tanya Traders, Itarana Circle">
  <input class="address"          value="Ground Floor, Rajgarh Road, Itarana Circle, Alwar, Rajasthan, 301001">
  <input class="phone"            value="+918071681308">
  <input class="business_email"   value="bansal.saurabh48@gmail.com">
  <input class="state"            value="Rajasthan">
  ```
- **Per-card outlet ID:** the trailing numeric suffix of the dealer's permalink slug (e.g. `…-473942/Home` → `473942`). This is the canonical `storeId`.
- **Per-card eacsi (Synup tracking ID):** lives on the anchor as `data-track-event-eacsi="…"`. Differs from the outletId. Preserved in `_extra.eacsi` for completeness but not used as primary key.
- **Per-card hours:** `id="store_outlet_business_hours_{outletId}"` carries the current open/closed string (e.g. "Open until 06:00 PM"). Preserved as `_extra.hours_now` — full weekly hours only live on the per-dealer detail page (we don't fetch detail pages for the basic locator sweep).
- **JSON-LD on detail pages:** every `/…-{outletId}/Home` page also embeds a full `LocalBusiness` schema.org JSON-LD with weekly `openingHoursSpecification`, `geo`, `paymentAccepted`, `amenityFeature`, etc. Not used in the current extractor but a known enrichment path.

## Extraction strategy

**Tier 1 — state×page sweep.** For each of the 12 state slugs surfaced by the landing page:
1. GET page 1.
2. Read `maxPage` from the `<a href="…?page=N">` pagination links.
3. GET pages 2…maxPage (250 ms between requests).
4. Parse every `<div class="store_outlet_01__item store-info-box">…` block on each page.

The locator is unprotected, so a plain Chrome UA + `Accept: text/html` is sufficient. We do not need to warm cookies, render JS, or impersonate. ~56 total requests for a full national sweep — completes in ~25 seconds with 250 ms pacing.

## Field inventory

| Field | Page type | Source | Notes |
|---|---|---|---|
| `brand` | listing | hardcoded `"Shree Cement"` | parent company; the per-record sub-brand goes to `_extra.sub_brand` |
| `storeId` | listing | URL slug suffix on `home_url` | stable numeric ID (e.g. `473942`); NOT the same as `eacsi` |
| `name` | listing | hidden `business_name` input | full `"{SubBrand} - {Trade Name}, {Sublocality}"` |
| `address.line1` / `line2` / `locality` | listing | parsed from `address` input | comma-split: `"Line1, [Line2,] Locality, City, State, Pincode"` |
| `address.city` | listing | `business_city` input (fallback: parsed `address`) | |
| `address.state` | listing | `state` input (fallback: parsed `address`) | full names, e.g. "Rajasthan" |
| `address.postalCode` | listing | trailing 6-digit token in `address` | every row has a 6-digit pincode |
| `lat` / `lng` | listing | `outlet-latitude` / `outlet-longitude` inputs | always present, never zero |
| `phone` | listing | `phone` input | already canonical `+91XXXXXXXXXX` |
| `email` | listing | `business_email` input | every row has an email |
| `url` | listing | `<a href="…/Home">` | the detail-page permalink |
| `locationType` | hardcoded | `"dealer"` | the source doesn't distinguish stockist vs retailer |
| `_extra.sub_brand` | parsed | first segment of `business_name` before " - " | `"Shree Cement"` (~88%) or `"Rock Strong"` (~12%) |
| `_extra.trade_name` | parsed | second segment of `business_name` after " - " | |
| `_extra.eacsi` | listing | `data-track-event-eacsi="…"` | Synup analytics ID — preserved but not used as primary key |
| `_extra.hours_now` | listing | `id="store_outlet_business_hours_{outletId}"` | current "Open until …" / "Closed" string |

## Pagination

URL param `?page=N` (1-indexed). Page 1 is the bare `/location/{state-slug}` URL; page N omits `?page=1` and uses `?page=N` for N≥2. Pagination terminates when the page's link list no longer contains a higher number.

## Files in this repo

- **Extractor:** `scrapers/sales-distribution/shree-cement.ts` (pure functions)
- **Production runner:** `scrapers/sales-distribution/shree-cement-run.ts` (live HTML sweep + Phase D pipeline)
- **Fixtures:** `__tests__/fixtures/shree-cement-rajasthan-p1.html`, `__tests__/fixtures/shree-cement-delhi-p1.html` (the Delhi fixture covers both Shree Cement + Rock Strong cards)
- **Tests:** `__tests__/shree-cement-stores.test.ts` (17 tests, 285 expects)
- **Output:** `data/results/shree-cement-stores.jsonl` (242 records on 2026-05-21)

## Sub-brand breakdown (2026-05-21)

| Sub-brand | Count | Share | Notes |
|---|---|---|---|
| Shree Cement | 213 | 88.0 % | the premium/parent brand |
| Rock Strong | 29 | 12.0 % | south-India brand surfaced on the same locator |
| (Bangur) | 0 | 0.0 % | NOT on this locator — Bangur dealers live on a separate subdomain (`dealers.bangurcement.com`); future recon job |

## Top states (2026-05-21)

| State | Dealers |
|---|---|
| Rajasthan | 60 |
| Bihar | 56 |
| Uttar Pradesh | 32 |
| Jharkhand | 19 |
| Haryana | 17 |
| Karnataka | 15 |
| West Bengal | 12 |
| Delhi | 11 |
| Punjab | 8 |
| Odisha | 5 |
| Andhra Pradesh | 4 |
| Telangana | 3 |

Total: **242 dealers across 12 states.**

## Gotchas

- The YAML's `official_locator_url: shreecement.com/dealer-locator` is **a 404**. The real locator is on the `dealers.shreecement.com` subdomain (linked from the homepage as "Find Shree Cement Shop Near You").
- The card's `data-track-event-eacsi="…"` is NOT the dealer's outletId — it's a Synup analytics tracking ID. The real outlet ID is the trailing numeric suffix of the dealer's permalink slug (`/…-NNN/Home`). The hours block's `id="store_outlet_business_hours_NNN"` uses the same outletId. We use the URL suffix as the canonical `storeId` and preserve `eacsi` in `_extra` for completeness.
- The `business_name` is a composite `"{Brand} - {Trade Name}[, {Sublocality}]"`. The brand prefix tells us the SUB-brand (Shree Cement / Rock Strong); the top-level `brand` field on every record is hardcoded to the parent "Shree Cement" (matching the YAML / corporate parent).
- **Bangur Cement** is a sibling sub-brand hosted on a completely separate Synup site at `dealers.bangurcement.com` (linked from this locator's footer). The two locators share a UI framework but are independently provisioned (different customerIds). Bangur is NOT included in this extractor; it should be a sibling YAML / recon task.
- Some dealers appear in multiple state listings (likely state-boundary edge cases or Synup's `areaServed` cross-listing). Dedup by `storeId` collapses these — 249 raw rows → 242 unique on the 2026-05-21 sweep.
- Email is surfaced in plain text on the listing page (`business_email` input) — privacy-noteworthy but a published-by-the-source field.
- Full weekly hours are only on the per-dealer detail page's JSON-LD; the listing only carries a "Open until 06:00 PM" snapshot. Detail-page enrichment would be a follow-up Tier 1 pass (242 GETs).
- Public dealer count is unpublished — the YAML's `authoritative_count` is `null`. We score against the locator's own self-claim (242).
