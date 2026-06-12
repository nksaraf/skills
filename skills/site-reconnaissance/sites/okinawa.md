# Okinawa Autotech (okinawascooters.com)

**Domain:** okinawascooters.com
**Vertical:** Sales-distribution — E2W (electric two-wheeler) dealer network, India
**Last verified:** 2026-05-21
**Tier:** 1 (single HTML GET, no JS, no XHR)
**Framework:** ASP.NET WebForms 4.0.30319 on IIS (Sterco Digitex CMS)
**Protection:** none (default Chrome UA gets a clean 200)

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| Dealer locator (full list) | `/find-us` | https://okinawascooters.com/find-us |
| Locator (redirected) | `/location`, `/location.aspx?mpgid=31&pgidtrail=31` | both 301→302 to `/find-us` |
| Galaxy store (flagship — DIFFERENT from dealer) | `/galaxy-store/<city>` | `/galaxy-store/noida` |
| Galaxy store via aspx | `/store.aspx?mpgid=43&pgidtrail=43&storeid=<n>` | `?storeid=1..7` |
| Sitemap | `/sitemap.xml` | only lists `/galaxy-store/*` (7 entries) — NOT the dealers |

The original YAML's `/dealer-locator` 404s — the actual locator path is `/find-us`. **Update the YAML's `official_locator_url`** if you copy this addendum into a future scrape config.

## Hydration payload

- **Location:** server-side rendered HTML directly inside the `<ul>` of the locator block. No JSON, no `__NEXT_DATA__`, no JSON-LD on this page.
- **Source name:** the `aspnetForm` (action="/find-us") owns the markup; the dealer list is a `<asp:Repeater id="rptstores">` rendered to a flat `<li>` list.

```html
<li><a>
  <h4>Perfect Sales</h4>
  <p>Field Road, Kurwar Naka, District-Sultanpur Uttar Pradesh- 228001</p>
</a>
  <div class="block_box">
    <p><img …> <a href="http://www.google.com/maps/place/," …>Get Direction</a></p>
    <div class="view-more-btn">
      <a id="ctl00_ContentPlaceHolder1_rptstores_ctl000_lnk" href="javascript:__doPostBack(…)">View More</a>
    </div>
  </div>
</li>
```

## Extraction strategy

Tier 1. One `fetch` against `/find-us`. The regex anchor is the `rptstores_ctl<NN>_lnk` id — present on every real dealer row and **only** on dealer rows (filters out the nav and mobile-menu `<li><a><h4>` blocks elsewhere in the markup).

Per dealer we recover:
- **name** — h4 text
- **addressLine** — p text
- **postalCode** — first 6-digit token in the address
- **state** — match against the India states list (handles "Tamilnadu" → "Tamil Nadu", "Orissa" → "Odisha", "Pondicherry" → "Puducherry"); falls back to PIN-prefix → state when the row omits state
- **city** — segment after the last comma before the state anchor, with stripping of "District-" prefixes, parenthesised state mentions like "(Odisha)", and noise prefixes (P.S., P.O., Dist., Ward No., …)

## Field inventory

| Field | Source | Coverage in fixture | Notes |
|---|---|---|---|
| name | `<h4>` | 315/315 (100%) | unique enough — used to spot duplicates if any |
| addressLine | `<p>` | 315/315 (100%) | full street, ends with the state + PIN |
| state | address regex + PIN-prefix fallback | 313/315 (99.4%) | only 2 dealers have unparseable state |
| postalCode | `\b(\d{6})\b` against address | 312/315 (99.0%) | 3 rows have no PIN |
| city | last-comma segment | 311/315 (98.7%) | a few rows have district-only or single-segment addresses |
| lat / lng | — | 0% | source has none; "Get Direction" is hardcoded `http://www.google.com/maps/place/,` (no comma'd coords) |
| phone | — | 0% | not present on any dealer row |
| hours | — | 0% | not present |
| email | — | 0% | not present |

## Files in this repo

- **Extractor:** `scrapers/sales-distribution/okinawa.ts`
- **Production crawler:** `scrapers/sales-distribution/okinawa-dealers.ts`
- **Fixture:** `__tests__/fixtures/okinawa-find-us.html` (572 KB, captured 2026-05-21)
- **Tests:** `__tests__/okinawa-stores.test.ts` (10 tests, all green)
- **JSONL output:** `data/results/okinawa-stores.jsonl`
- **Phase D snapshot script:** `scripts/run-okinawa-snapshot.ts`

## Pagination

None. **The whole dealer network is on a single page.** State/city dropdowns trigger `__doPostBack` (full-page form post) and filter the same list client-side; we don't need them because the unfiltered list is exactly what we want.

## Gotchas

- **`/dealer-locator` 404s** even though the YAML names it. The real path is `/find-us`. `/location` and `/location.aspx?mpgid=31&pgidtrail=31` both redirect there.
- **www. subdomain is the 404'er** — `https://www.okinawascooters.com/find-us` returns 404; `https://okinawascooters.com/find-us` works. The www. → apex redirect is misconfigured for some subpaths.
- **"Get Direction" carries no coordinates** — every dealer's map link is literally `http://www.google.com/maps/place/,` (trailing comma, no lat/lng). So there's no way to recover geocoords from the index. A future iteration could batch-geocode the PIN + city + state, but that's a different pipeline.
- **"Galaxy Stores" are NOT dealers** — they are 7 Okinawa-owned flagship stores rendered via `/galaxy-store/<city>` and `/store.aspx?storeid=...`. They appear in the sitemap.xml; the dealer network does not. **Do not confuse them.**
- **Authoritative-count gap is REAL** — press still cites "350+" dealers from the 2023 era; the locator shows 315 today. That's a ~10% contraction. Given FAME-II subsidy clawback issues hit the company in 2023, **this is consistent with a real network shrink**, not an extractor bug. Documented under `notes` in the YAML and in the snapshot's scorecard.
- **Address pincode-prefix fallback covers ~20% of rows** that omit the state name (e.g. "Sn Electra Motors | …Chennai-600024"). The two unmapped rows are pincodes that fall in zone boundaries the PIN-prefix table doesn't cover yet.

## Scorecard summary (2026-05-21)

- Overall: **65/100 (medium)**
- Completeness: **72** — extracted 315 vs authoritative 350+ (10% below, within 30% tolerance)
- Data quality: **35** — capped because the source has no coords/phone/hours
- Source authenticity: **91** — domain matches brand, Wikipedia-known, well-known E2W brand
- Freshness: **75** — first snapshot today; alert "no-prior-snapshot" raised
