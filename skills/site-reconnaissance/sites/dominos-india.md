# Domino's Pizza India — store locator

**Domain:** dominos.co.in (locator at `/store-locations`)
**Vertical:** India retail / QSR (Quick Service Restaurant)
**Last verified:** 2026-05-21
**Tier:** 1 (direct HTTP fetch — every field is server-rendered in the per-store HTML)
**Framework:** PHP / Apache behind CloudFront (no JS execution required)
**Protection:** None observed. CloudFront caches HTML aggressively (`max-age=2592000`); the site sets a `PHPSESSID` cookie that is **not** enforced for subsequent fetches — concurrent requests succeed without a session warmup.

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| Sitemap index | `https://www.dominos.co.in/sitemap.xml` | XML sitemapindex |
| Stores sitemap | `https://www.dominos.co.in/store-locations/sitemap_store.xml` | flat `<urlset>` of every per-store URL |
| Store detail | `https://www.dominos.co.in/store-locations/<city-slug>/<store-slug>` | `/store-locations/hissar/02-eminent-mall-hisar-125001-haryana` |

The store-slug almost always begins with a numeric token that is the Domino's internal store id (`02-eminent-mall-…`, `04-12-10-05-siya-talab-…`). When the slug is fully alphabetic (e.g. `sector-cue-ii`) the slug itself is the only stable identifier.

## Discovery strategy (full-list discovery via sitemap)

**Found the sitemap.** This is the canonical approach for Domino's India — no pincode or lat-lng sweep needed.

1. Fetch `/sitemap.xml` → returns a sitemapindex.
2. Fetch `/store-locations/sitemap.xml` → returns a second sitemapindex pointing at `sitemap_store.xml` (and several locality/area/city sitemaps).
3. Fetch `/store-locations/sitemap_store.xml` → flat list of **1,722 per-store URLs** (as of 2026-05-21).
4. Fetch each per-store URL, parse the HTML.

The locator UI itself is a pincode/lat-lng SPA that we deliberately bypass — there is no public list endpoint, but the sitemap supplies what would otherwise require enumerating ~30,000 Indian pincodes.

## Per-store HTML schema

Every canonical field lives in plain HTML. No `__NEXT_DATA__`, no `window.X`, no JSON-LD for the store record (there IS a JSON-LD `ItemList` of *nearby* stores — ignore it, it's a carousel of the wrong stores).

| Canonical field | Source location | Notes |
|---|---|---|
| `name` | `<h1 class="… store-desktop-main-title">…</h1>` | Includes "Domino's Pizza" prefix |
| `address.line1` | `<h2 class="grey-text store-page-address">…</h2>` | Full unstructured address — comma-separated tail usually contains `<city>, <state>` or `<state> - <PIN>` |
| `lat` / `lng` | `var position = new google.maps.LatLng(<lat>, <lng>)` inside `function initialize_0() { … }` | First LatLng in the init block is the store; subsequent LatLng calls (e.g. `markerPosition_0`) repeat the same coords. There are also LatLng-like coords on the nearby-stores carousel as `data-lat`/`data-lng` — we ignore those |
| `phone` | First `<a href="tel:…">` in the overview block | Almost always `1146564906` or `1800 208 1234` (national hotline — Domino's India routes per-store via Caller ID upstream, not a per-store number) |
| `hours` | `<h2>Opening hours</h2><p class="…">10.57.00 AM to 11:59:00 PM</p>` | Single window; we replicate across all seven days (the source has NO per-day variation). When the source shows " to " (empty), we leave `hours: null` and preserve the raw string in `_extra.opening_hours_raw` |
| `segments` | `<h2>Cuisines</h2><p>Pizza, Fast Food</p>` | Always `Pizza, Fast Food` for Domino's India |
| `features` | `<h2>More info</h2><i class="fa-check"></i> dinein-n-delivery-n-takeaway` | We split on `-n-`, comma, `|`, etc. |
| `address.city` | breadcrumb position 3 (`Home > Store Locations > <City> > <store>`) | Preferred over an address-line-derived city — the breadcrumb is canonical |
| `address.state` | trailing comma-segment of `address.line1` matched against a known-Indian-states whitelist, OR fallback to a word-bounded scan of the full address line | ~81% coverage |
| `address.postalCode` | 6-digit Indian PIN matched against `address.line1` (or URL as fallback) | ~78% coverage — Domino's doesn't always include PIN in the address |
| `storeId` | leading numeric token of the URL slug (or full slug when alphabetic) | Not globally unique — collides across cities; URL is the unique key |

## Coverage

| Metric | Value |
|---|---|
| Sitemap URLs | 1,722 |
| Successfully extracted | **1,669** |
| HTTP failures (404/500) | 53 (3.1%) — stale sitemap entries / temporarily-broken pages |
| Authoritative reference | ~1,900 (Jubilant FoodWorks FY24 AR) — 12% gap (within 15% tolerance) |
| Cities covered | 347 distinct |
| Lat/lng populated | 99.5% (1,661/1,669) |
| Lat/lng inside India bounding box | 99.8% (1,658/1,661) — 3 records have lat==lng (source data bug) or a Sri Lanka coord |
| City populated | 100% |
| State populated | 80.7% (via name-whitelist) |
| Phone populated | 100% (mostly the national hotline) |
| Hours populated | 73.5% — many stores ship " to " (empty open/close) |
| **Scorecard overall** | **84/100 (high)** — completeness 70, data quality 95, source authenticity 99, freshness 75 |

## Extraction strategy

- **Tier 1** — direct `fetch()` with realistic Chrome UA + `Accept-Language: en-IN`.
- Concurrency 16-24 is safe; no rate limiting observed at 30 ms/request spacing.
- 53/1,722 (3%) pages return 404 or 500 — handled by per-URL try/catch, never blocks the run.
- Full crawl: ~2.5 minutes on a single host.

## Files in this repo

- **Extractor:** `scrapers/retail/dominos-india.ts` (pure functions over sitemap XML + per-store HTML)
- **Production crawler:** `scrapers/dominos-india-stores.ts` (sitemap fan-out + concurrent fetch + JSONL emit)
- **Fixtures:** `__tests__/fixtures/dominos-india-store-sector-cue-ii.html`, `__tests__/fixtures/dominos-india-store-eminent-mall.html`, `__tests__/fixtures/dominos-india-sitemap_store.xml`
- **Tests:** `__tests__/dominos-india-stores.test.ts` (13 tests, including a full-record schema lock + a population check against the production JSONL)
- **Results:** `data/results/dominos-india-stores.jsonl` (1,669 records)
- **Scorecard:** `data/reports/dominos-india-scorecard.json`
- **Snapshot:** `data/snapshots/retail/domino-s-pizza/20260521T*.jsonl`

## Pagination

None — the sitemap is one flat file. No `?page=N`, no infinite scroll, no city-paged endpoint.

## Gotchas

- **Phone is almost always the national hotline.** `1146564906` and `1800 208 1234` repeat across the entire estate. Don't be surprised that 100% phone coverage doesn't mean 100% _useful_ phone coverage — Domino's India only publishes the central CRM number.
- **Opening hours is a single window.** No per-day variation. Many stores ship `<p>  to </p>` (an empty open/close pair the templating system rendered around missing data). Our extractor returns `hours: null` in that case and preserves the raw `" to "` in `_extra.opening_hours_raw` so we don't silently lose the signal.
- **`storeId` is not globally unique.** Different cities reuse the same numeric prefix (Mumbai-01, Pune-01, Delhi-01 all exist). Dedupe by URL, not by storeId.
- **lat == lng appears in 2 records.** Source data bug — both axes get the same number. Caught by the India-bounds check on the scorecard.
- **JSON-LD on the page is for nearby stores, not this store.** The `<script type="application/ld+json">` is a carousel `ItemList` of stores near this one — looks tempting but always one-store offset from what you want. Use the H1 + map-init-block path instead.
- **Breadcrumb regex is order-sensitive.** The `<meta property="position" content="3">` follows its `<span property="name">…</span>` in the source HTML; our extractor matches both orders defensively.
- **3% of sitemap URLs are stale.** 404s on `dadra-and-nagar-haveli/pramukh-vihar-naroli-road-silvassa`, `mumbai/powai`, etc. — pages that were valid when the sitemap was generated but have since been removed. Tolerate; don't fail the whole run.

## Candidates for promotion to canonical (future schema revision)

- `average_cost` (text "₹400 for two people (approx.)") — Domino's is the first brand we've reconned that ships an average-cost-for-two field. If two more brands ship it, promote to a `pricing.averageCostForTwo` slot.
- "Service tags" enum (Dinein / Delivery / Takeaway) — already in `features`, but more structured than the free-text we collect today.

## Why this is Tier 1 (not Tier 2 via an API)

The store-locator UI at `/store-locator` is a pincode-input SPA that hits an XHR endpoint per pincode. Sweeping that endpoint would need a curated set of ~200 pincodes to cover all cities AND dedup across overlapping coverage — and would still miss tier-3 towns where no curator-supplied pincode hits. The sitemap is the better source: complete, canonical, cacheable, and HTTP-only.

## Cross-brand notes

This is the first **QSR** brand in the retail addendum library. The schema worked unchanged — `locationType: "store"` is correct for restaurants too. The `_extra.average_cost` field is QSR-specific (fast fashion stores don't surface this). Sibling QSR brands at Jubilant FoodWorks (Dunkin India, Popeyes India, Hong's Kitchen) are NOT served from `dominos.co.in` — they have separate domains and will need their own recon.
