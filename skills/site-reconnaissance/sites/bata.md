# Bata India — store locator

**Domain:** stores.bata.com (locator microsite; the brand site `www.bata.com/in/` and the legacy `www.bata.in` redirect to it)
**Vertical:** India retail (footwear — Bata India Ltd., the parent)
**Last verified:** 2026-05-21
**Tier:** 1 (direct HTML fetch — every store has a static URL in `sitemap.xml`; LocalBusiness JSON-LD on each detail page)
**Framework:** Singleinterface / Promanage CMS (server-rendered HTML; `domainid=59`)
**Protection:** none on `stores.bata.com`; `www.bata.com/in/` is Akamai-blackholed (every fetch with realistic Chrome headers times out — the locator is functionally moved to the microsite)

## URL patterns

| Page type | URL | Method |
|---|---|---|
| Sitemap | `https://stores.bata.com/sitemap.xml` | GET |
| Store detail | `https://stores.bata.com/bata-shoes-sandals-store-in-{locality}-{city}` | GET (HTML) |
| Brand homepage | `https://www.bata.com/in/store-locator` | GET (Akamai-blocked — do not use) |

**Key insight:** the brand URL `bata.in/store-locator` 301s to `bata.com/in/store-locator` which is behind Akamai and returns no body to a clean Chrome UA (every `curl`/`fetch` hangs the full 25s and times out). But the actual customer-facing locator is on the `stores.bata.com` subdomain — a Singleinterface/Promanage microsite that ships completely clean HTML with no protection. The Akamai-protected page is a marketing shell; the data lives on the unprotected microsite.

## Hydration payload

- **Listing source:** `sitemap.xml` — a flat `<urlset>` with `<loc>` per store. 1,620 entries on 2026-05-21 (matches Bata India FY24 ~1,600 authoritative count almost exactly).
- **Detail source:** one `<script type="application/ld+json">` block per page, with a Schema.org `LocalBusiness` record.

### Detail JSON-LD schema

```jsonc
{
  "@context": "https://schema.org",
  "@type": "LocalBusiness",
  "url": "https://stores.bata.com/bata-shoes-sandals-store-in-civil-lines-agra",
  "name": "Bata in Civil Lines, Agra",
  "telephone": "08860835983            ",    // right-padded with 12 spaces — always trim
  "latitude": "27.2067691            ",       // string, padded
  "longitude": "78.0031208            ",
  "address": {
    "@type": "PostalAddress",
    "name": "Bata Shoe Store",
    "addressLocality": "Civil Lines            ",
    "addressCountry": "IN",
    "postalCode": "282002            ",
    "streetAddress": "No. 2/205, Vinayak Mall, Mahatma Gandhi Road, Civil Lines, Near Canara Bank ATM, Civil Lines, Agra - 282002            "
  },
  "image": "https://stores.bata.com/staticfiles/sites/templates/images/logo/batalogo.webp",
  "aggregateRating": {
    "@type": "AggregateRating",
    "ratingValue": "4.9",
    "reviewCount": "809"
  },
  "review": [ ... ]
}
```

JSON-LD does **not** contain:

- `addressRegion` (state) — we don't get state from the source. Possible to backfill from pincode if needed.
- Opening hours — they live in the DOM `<section id="dvaMenuBizHrs">`.
- Payment methods, parking/wifi/washroom attributes — DOM-only.

### DOM extractions

The `<section id="dvaMenuBizHrs">` block carries:

- One line like `"10:30 AM - 09:30 PM"` (most common — one window for all 7 days)
- OR a `<li>` list with `"Sunday - Saturday: 10:00 AM - 8:00 PM"` (range)
- OR per-day entries (rare)

Two labelled `<h3>` sections render bullet lists:

- **Attributes** — typical values: "Free Parking", "WIFI", "Washroom", "Instore Pickup", "Delivery"
- **Payments** — "Cash Payment", "Cheque Payment", "Debit Card", "Master Card", "Online Payment", "Visa"

## Field inventory

| Canonical field | Source | Notes |
|---|---|---|
| `storeId` | URL slug tail (`civil-lines-agra`) | unique + stable across crawls |
| `brand` | hard-coded `"Bata India"` | parent corp is Bata Corporation; this is the India ltd subsidiary |
| `name` | JSON-LD `name` | "Bata in {Locality}, {City}" |
| `status` | hard-coded `"open"` | Bata only lists live stores in the sitemap |
| `mallOrLocation` | regex on `streetAddress` for "...Mall/Plaza/Centre/Complex/Arcade/Tower" | best-effort |
| `address.line1` | JSON-LD `streetAddress` (trimmed) | always set |
| `address.locality` | JSON-LD `addressLocality` (title-cased) | always set |
| `address.city` | parsed from `streetAddress` tail ("..., {City} - {pin}") | fall back to URL slug tail |
| `address.state` | (not in source) | `null` — could backfill from postalCode later |
| `address.postalCode` | JSON-LD `postalCode` (trimmed) | always set, 6 digits |
| `address.country` | JSON-LD `addressCountry` (always `"IN"`) | `normaliseCountry()` |
| `lat` / `lng` | JSON-LD `latitude` / `longitude` (string -> number) | always set, all inside India bounds |
| `phone` | JSON-LD `telephone` (trimmed) | usually 10-digit mobile prefixed `0` |
| `email` | (not in source) | `null` |
| `url` | the detail URL itself | |
| `hours` | DOM `dvaMenuBizHrs` -> `parseHours()` -> 7-day record | every entry normalised to `HH:MM-HH:MM` |
| `locationType` | hard-coded `"store"` | |
| `features` | DOM "Attributes" `<li>` items | "Free Parking", "WIFI", "Washroom", "Instore Pickup", "Delivery" |
| `_extra.payments` | DOM "Payments" `<li>` items | preserved (not promoted) |
| `_extra.ratingValue` | JSON-LD `aggregateRating.ratingValue` | preserved |
| `_extra.reviewCount` | JSON-LD `aggregateRating.reviewCount` | preserved |
| `_extra.image` | JSON-LD `image` | preserved |

## Extraction strategy

Tier 1, concurrent. Per run:

1. GET `sitemap.xml` once -> `parseSitemap(xml)` -> 1,620 URLs.
2. Worker pool (concurrency 8-10) fetches each detail page with realistic Chrome UA + en-IN Accept-Language.
3. `parseBataDetail(html, url)` -> `BataStoreRaw` (JSON-LD + DOM extractions).
4. `extractBataStore(raw, url)` -> canonical `RetailStore` with `locationType: "store"`, `_extra` non-empty.

The full run takes ~3 minutes at concurrency 10 (~9 stores/sec). No anti-bot, no warm-up needed.

## Coverage

On 2026-05-21:

- **1,620 store URLs in sitemap** — matches Bata India FY24 annual report (~1,600 stores incl. franchise) almost exactly.
- Expect all 1,620 to extract cleanly — JSON-LD format is consistent across the four cities probed (Agra UP, Ahmedabad GJ, Yadgir KA, Adilabad TS), spanning 4 states and both metro + tier-3 cities.

## Files in this repo

- **Extractor library:** `@vinxi/scraper/retail` (canonical schema) + `scrapers/retail/bata.ts` (sitemap + JSON-LD parser + record mapper)
- **Production scraper:** `scrapers/bata-stores.ts`
- **One-shot extractor script:** `scripts/run-bata-extract.ts`
- **Fixtures:** `__tests__/fixtures/bata-store-detail-civil-lines-agra.html`, `__tests__/fixtures/bata-stores-raws.json`, `__tests__/fixtures/bata-sitemap-sample.xml`
- **Tests:** `__tests__/bata-stores.test.ts` (8 schema-locked tests)
- **Output:** `data/results/bata-stores.jsonl`

## Pagination

None — sitemap is flat (no nested index) and contains every store URL.

## Gotchas

- **www.bata.com is Akamai-blackholed.** Don't waste time trying to scrape the brand URL — every fetch times out after 25s. `bata.in` redirects to it. `stores.bata.com` is the working subdomain.
- **JSON-LD strings are right-padded with whitespace.** Every string value in the LocalBusiness block has trailing spaces (typically 12). Always trim. Our extractor uses `strOrNull` which trims + returns `null` on empty.
- **No state field in source.** JSON-LD has `addressCountry` and `postalCode` but not `addressRegion`. If state is needed downstream, derive from pincode (first 2 digits = state region in India Post numbering).
- **City must be parsed from `streetAddress`.** The URL slug tail (`-agra`, `-new-delhi`) is unreliable when the city is multi-word ("West Tripura", "West Nimar") — there's no boundary marker between locality words and city words. We anchor on the postal code: `..., {City} - {pin}` is consistent.
- **Hours: source emits ONE window for most stores.** We replicate it across all 7 days. A handful of stores emit "Sunday - Saturday: 10:00 AM - 8:00 PM" as a range — we handle that too.
- **Vendor is Singleinterface / Promanage.** Other Indian brands on the same vendor: Lifestyle (`stores.lifestylestores.com`). The HTML conventions (`dvaMenuBizHrs`, `dvaMenuContact`, `/Process/listajaxdatabata`, `domainid=N`, `staticfiles/sites/templates/`) are reusable signatures for vendor identification. We did NOT use the `/Process/listajaxdatabata` AJAX endpoint — it requires session-like params (`lat`, `longi`, `franchise`) and returns HTML fragments; the per-page JSON-LD path is simpler and more robust.
- **`extractMall` is heuristic.** Regex on `Mall|Plaza|Centre|Center|Market|Arcade|Complex|Tower` — occasionally matches landmark refs like "Near Kaveri Saree Centre" (false positive). Treat `mallOrLocation` as best-effort; the canonical address is `address.line1`.

## Sibling brand notes

`siblings: [hush-puppies-india]` in the YAML. Hush Puppies in India is distributed by Bata India but operates on its own locator (not stores.bata.com). Not extracted in this run; a follow-up recon turn should probe `hushpuppies.in/store-locator` and report the vendor.

## Why this is Tier 1 (and not Tier 3 via the brand URL)

We probed `www.bata.com/in/store-locator` first (the YAML's `official_locator_url` redirect target). It is fully Akamai-protected — three curl retries timed out at 25s each with zero bytes. A Tier 3 stealth-browser path would work but is wasted effort: `stores.bata.com` ships clean SSR HTML with full LocalBusiness JSON-LD and zero protection. We always prefer the unprotected sibling subdomain when one exists.

## Candidates for promotion to canonical (future schema revision)

- `_extra.payments` — list of accepted payment methods. If three brands ship a similar list, promote to canonical `paymentMethods: string[]`.
- `_extra.ratingValue` / `_extra.reviewCount` — Bata is the first brand in this corpus to ship Google-style aggregate ratings on every store. If two more brands match, promote to canonical `rating?: { value: number, count: number }`.
