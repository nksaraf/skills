# Cashify

**Domain:** cashify.in
**Vertical:** India recommerce (used phone buy-back / repair / refurbished sale)
**Last verified:** 2026-05-21
**Tier:** 1 (raw HTTP + RSC payload parse, no browser needed)
**Framework:** Next.js App Router (React Server Components stream)
**Protection:** none observed — plain Chrome UA suffices, no Cloudflare/Akamai/Imperva

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| Locator (listing) | `/offline-stores` | https://www.cashify.in/offline-stores |
| City aggregator | `/offline-stores/cashify-store-{city}` | `/offline-stores/cashify-store-delhi` |
| Store detail | `/offline-stores/cashify-store-{slug}-in-{city}` | `/offline-stores/cashify-store-vishrantwadi-in-pune` |

City slugs occasionally carry a numeric disambiguation suffix (`cashify-store-lucknow-2`). The `cityFromUrl` helper strips that.

## Hydration payload

Cashify's locator is a Next.js App Router page — the RSC stream is delivered as concatenated `self.__next_f.push([1, "<chunk>"])` calls inside the HTML. **`reassembleRSCStream` is reused from `scrapers/retail/mango.ts`** verbatim (this is the same RSC encoding Mango ships).

- **Source name:** RSC stream (no `__NEXT_DATA__`, no JSON-LD)
- **Listing payload root:** `data.kioskStores.postCount` + `data.kioskStores.posts[]`
- **Detail payload:** the wrapping `post` object with `type: "offline_store"`, full data under `post.custom`

### Key paths

**Listing / city pages** (post-shape):

```
posts[i].title.rendered          → name
posts[i].url                     → /offline-stores/cashify-store-{slug}
posts[i].offlineStoreAddress     → address line1 (prefixed with "Shop Address: " sometimes)
posts[i].offlineStorePincode     → postal code (sometimes number, sometimes string)
posts[i].offlineStoreContact     → phone
posts[i].offlineStoreLatlong     → "lat,lng" (LOWERCASE l; usually empty in SSR stub)
posts[i].storeDirection          → Google Maps URL with `@lat,lng,17z` (RELIABLE lat/lng source)
posts[i].storeTimingList         → [{ selectDays: ["Monday", ...], openingTime: "10:00 AM", closingTime: "09:00 PM" }]
posts[i].featuredMediaUrl        → primary image
```

**City directory** rows inside the listing payload are also in `kioskStores.posts[]`:
- Have `count > 0`
- Have a `/offline-stores/cashify-store-{city}` URL (no `-in-` infix)
- Have empty `offlineStoreAddress`

**Detail pages** (custom-shape):

```
post.title.rendered                       → name (e.g. "Cashify Mobile Phone Store Vishrantwadi Pune")
post.breadcrumbs[].link / .title          → city (the breadcrumb whose link ends `/offline-stores/cashify-store-{city}`)
post.custom.offlineStoreLatLong           → "lat,lng" (CAPITAL L)
post.custom.offlineStoreAddress           → full street address
post.custom.offlineStorePincode           → postal code
post.custom.offlineStoreContact           → phone
post.custom.offlineStoreServices          → ["bb", "rep", "acc", "ref"]   (codes — map to features)
post.custom.offlineStoreRating            → "5"  (string)
post.custom.offlineStoreReviewCount       → "2"  (string)
post.custom.storeTimingList               → same shape as listing
post.custom.storeDirection                → Google Maps URL
post.dateModified                         → updated_at (preserved in _extra)
```

### Service code mapping

```
bb   → Buy-Back
rep  → Repair
acc  → Accessories
ref  → Refurbished Sale
```

## Extraction strategy

Three-layer fetch, 6-8 concurrency, ~80-220ms jitter between requests. No warmup needed.

1. **Listing** — fetch `/offline-stores` once, parse the RSC stream:
   - `kioskStores.postCount` is the source's authoritative claim (243 at capture).
   - `extractCityDirectory()` returns 91 city tuples `{name, url, count}`.
   - 12 "featured" stores are populated; the other 231 are stubs.
2. **City fan-out (with pagination)** — fetch each of the 91 city pages, plus the SSR-paginated tail (`/page/N`) for any city with >12 stores:
   - Each city page surfaces up to 12 fully populated `post`-shape store records.
   - Cashify's CMS caps SSR'd posts at 12 per request. `OfflineStoreListDynamic` is the component that surfaces the rest — but it is **not** an XHR loader. It's a thin lazy-loaded React wrapper around the same SSR page route, navigated via `/page/N` URL suffixes (see "Pagination" below). The runner fans out those tail pages.
   - With pagination: Delhi (37 → 4 pages: 12+12+12+1), Bengaluru (19 → 2 pages: 12+7), every other city ≤12 fits in 1 page.
   - `extractStoreDetailUrls` returns the discovered store slugs (`-in-{city}` URLs) — these become the detail-page queue.
3. **Detail fan-out** — fetch every discovered detail URL:
   - These carry lat/lng (capital-L `offlineStoreLatLong`), services, rating, review count.
   - Records are deduped against the city-page payload by `url`; detail wins.

In the current capture this yields **239 of 243** claimed stores (~98% coverage). The remaining 4-record gap is structural: Cashify's CMS reports `postCount: 243` but the sum of per-city `count` fields is only 237 — a handful of stores cross-list across cities and the global counter double-counts them. There is no extraction path that reaches 243 without inventing duplicates.

### Pagination discovery (2026-05-21 follow-up)

The mystery component was `OfflineStoreListDynamic` from the page bundle (`/_next/static/chunks/app/gpro/offline-store/landing/[slug]/page-2000ca5a162f6256.js`). It lazy-loads webpack chunk `5639` (module `58020`) — full URL: `https://www.cashify.in/gpro_/_next/static/chunks/5639.f3340aacbed5d4de.js`.

That chunk contains **no XHR**. The component is a `lazy(() => s.e(5639).then(s.bind(s,58020)))` wrapper around a `<Pagination>` UI that builds prev/next anchor URLs via:

```js
let u=(e,l)=>{
  if(!e)return;
  e=e.replace(/^\/+/,"");
  e=(e=("/").concat(e)).replace("/page/".concat(l),"");
  e.endsWith("/")||(e=e.concat("/"));
  return l<=1 ? e : e.concat("page/".concat(l))
};
```

So tail pages are simply the same SSR route at `/{citySlug}/page/{N}` — the same RSC payload shape, just sliced. The runner fans those out via `cityPageUrl(city.url, n)` for `n ∈ [1, ceil(count / 12)]`. Tested empirically:

| City | Claim | Pages | Per-page populated counts |
|---|---|---|---|
| Delhi | 37 | 4 | 12 + 12 + 12 + 1 |
| Bengaluru | 19 | 2 | 12 + 7 |
| Kolkata | 11 | 1 | 11 |

Out-of-bounds pages (e.g. Delhi `/page/5`) return a generic 404-style HTML stub with zero `-in-{city}` detail URLs; the runner detects and skips that.

## Field inventory

| Field | Page type | Source path | Notes |
|---|---|---|---|
| brand | all | constant | "Cashify" |
| storeId | detail | `post.id` | Listing stubs zero this — fall back to URL. |
| name | all | `title.rendered` | "Cashify Mobile Phone Store {locality} {city}" |
| address.line1 | all | `offlineStoreAddress` | Strip "Shop Address: " prefix. |
| address.city | all | breadcrumb link OR URL slug | `offlineStoreCity` is usually empty. |
| address.postalCode | all | `offlineStorePincode` | Coerce number → string. |
| lat / lng | detail (cap L) → listing (lower l) → `storeDirection` `@x,y,17z` | `offlineStoreLatLong` / `offlineStoreLatlong` / maps URL | Listing posts have empty Latlong but populated `storeDirection` — the regex extracts lat/lng from the maps URL. |
| phone | all | `offlineStoreContact` | |
| hours | all | `storeTimingList` | Fan single timing across `selectDays`. |
| features | detail | `offlineStoreServices` codes | Mapped to canonical labels. |
| _extra.rating | detail | `offlineStoreRating` | Coerced to number. |
| _extra.review_count | detail | `offlineStoreReviewCount` | Coerced to number. |
| _extra.services | detail | raw codes | `["bb", "rep", "acc", "ref"]` |
| _extra.maps_url | detail | `storeDirection` | Preserved verbatim. |
| _extra.featured_image | detail | `featuredMediaUrl` | First image URL. |
| _extra.updated_at | detail | `dateModified` | Source-side last-modified — feeds the freshness scorecard. |
| _extra.source_type | all | derived (default `offline_store`) | Always `offline_store` — locator never surfaces other types. |
| _extra.source_id | all | `post.id` | Preserved separately from `storeId`. |

## Pagination

**City pages paginate via path suffix `/page/N`** — no query string, no XHR.

| URL | Slice |
|---|---|
| `/offline-stores/cashify-store-delhi` | Posts 1–12 |
| `/offline-stores/cashify-store-delhi/page/2` | Posts 13–24 |
| `/offline-stores/cashify-store-delhi/page/3` | Posts 25–36 |
| `/offline-stores/cashify-store-delhi/page/4` | Post 37 (last) |
| `/offline-stores/cashify-store-delhi/page/5` | 404 stub |

The listing page (`/offline-stores`) does **not** paginate — it is a city directory only, and the 12 featured stores at the top are a curated slice, not page 1 of a sequence. The 91 cities + their tail pages are the only fetch fan-out needed.

Page-size is 12 (defined in extractor as `CASHIFY_SSR_PAGE_SIZE`). Pages are computed up-front from each city's `count`: `ceil(count / 12)`. Cities reporting `count ≤ 12` get a single fetch.

## locationType breakdown

| locationType | count | source |
|---|---|---|
| store | 206 | All records — Cashify's public locator only surfaces owned/branded outlets. |
| pickup-point | 0 | Cashify's "Pickup Points" mentioned in marketing copy are **not exposed via the locator**. They're booked through the buy-back funnel (where a courier picks up a phone from the customer's address) — there's no public directory of partner kiosks. |

If a future scrape reveals a `type: "kiosk"` or `category: "pickup"` value on the source records, update `extractCashifyStorePost` / `extractCashifyStoreDetail` to classify accordingly. The `_extra.source_type` discriminator is preserved verbatim on every record so a re-classification can be done in-place.

## Files in this repo

- **Extractor:** `scrapers/retail/cashify.ts`
- **Runner (full crawl):** `scripts/run-cashify-extract.ts`
- **Snapshot/scorecard runner:** `scripts/run-cashify-snapshot.ts`
- **Fixture:** `__tests__/fixtures/cashify-stores.json` (132 listing posts + 1 full detail record)
- **Tests:** `__tests__/cashify-stores.test.ts` (14 schema-locked assertions)
- **Output:** `data/results/cashify-stores.jsonl`
- **Scorecard:** `data/reports/cashify-scorecard.md`

## Gotchas

- **Two distinct key spellings:** listing pages use lowercase `offlineStoreLatlong`, detail pages use capital `offlineStoreLatLong`. The extractor handles both.
- **`offlineStoreLatlong` is almost always empty on listing/city pages** — even when address/phone/hours are populated. Recover lat/lng from `storeDirection` (Google Maps `@lat,lng,17z`).
- **`post.id` is zeroed on SSR stubs** (the lazy-loaded posts on the listing page). Fall back to the URL as the stable identity.
- **`offlineStoreCity` is empty everywhere.** Use breadcrumbs (on detail pages) or parse from the URL slug (`-in-{city}` suffix).
- **`offlineStorePincode` is sometimes a number, sometimes a string.** Coerce via `String(...)`.
- **City slug disambiguation:** `cashify-store-lucknow-2` is the same city as Lucknow — the `-N` suffix is a WordPress permalink collision. `cityFromUrl` strips it.
- **The 12-per-SSR cap.** Each `kioskStores.posts[]` response — on the listing AND on every city page — is capped at exactly 12 populated records. The runner compensates by paginating any city whose `count > 12` via `/page/N` URLs. `OfflineStoreListDynamic` looks like an XHR component name but is actually a `lazy()`-wrapped `<Pagination>` UI; the data still comes from the SSR'd page route, just sliced.
- **Address noise:** records sometimes carry a "Shop Address: " prefix on `offlineStoreAddress`. Strip it.
- **No public pickup-point directory.** Cashify's "Pickup Points" mentioned in their marketing copy are buyback-flow kiosks, not enumerated on the site — every record from this locator is `locationType: "store"`.
