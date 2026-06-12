# Savana (savana.com)

Indian fast-fashion e-commerce (Mafengwo-stack — "rwd-guide" React 17 SPA hosted on `mfrcdn.com` CDN). Operates in India (`.in`) and Iraq (`.iq`). Cloudflare front, **no anti-bot challenge**; the API host accepts plain Chrome `User-Agent` once the required tenant headers are present.

- **Last verified:** 2026-06-10 (extended: activity discovery + product detail API + filter taxonomy)
- **Vertical:** Fast-fashion product listings (dresses, tops, denim, swimwear, knitwear, sports, bags, accessories, etc.)
- **Tier:** **2 — direct JSON API** (no browser needed once the API host + headers are known)
- **Framework:** React 17 SPA, app name `rwd-guide` v6.34.0, bundle on `static.mfrcdn.com`
- **Protection:** Cloudflare (`__cf_bm`, no challenge); `www.savana.com` is the web origin, `api-shop-in.savana.com` is the API origin (mirrors `.iq` for Iraq)
- **Status:** ✅ 3 extractor modules (activity / discovery / detail) + 4 fixtures + **31 schema-locked tests** + production scraper (full catalog walk) + discovery scraper (all activities)
- **Catalog scale captured:** 25 activities discovered on home; 2,707-product dress-activity walk landed
- **Business intel:** see [savana-business-intel.md](./savana-business-intel.md) — **Savana is Urbanic rebranded** (India+Iraq); 1,097 total activities running (only 25 on homepage); 42,172 product pages; daily flash-sale engine; members-tier exists; full attribute taxonomy mapped

## Page types

| Type | URL pattern | What's there | Source |
|---|---|---|---|
| Home | `/` | SPA shell, no SSR product data | — |
| Category | `/category/<slug>` | Curated category page | Sitemap lists 8: new-in, clothing, dresses, knitwear, denim, sports, swimwear, special-prices |
| **Activity** | `/activity/<slug>-<id>` | Curated collection (NOT in sitemap — surfaced via marketing links) | e.g. `/activity/dresses-12769` |
| Search | `/search/result/<query>` | Search results | 2,824 in sitemap |
| **Detail** | `/details/<slug>-<goodsId>` | Per-product page | 47,665 in sitemap (`sitemap.in.xml.gz`) |

## URL → activity id

The trailing numeric segment of `/activity/<slug>-<id>` is the `flowId`. Use `parseActivityId()` from the extractor.

```
/activity/dresses-12769 → activityId = "12769"
```

## Canonical API endpoint

```
POST https://api-shop-in.savana.com/n/api/buyer/guide/user/goods-flow/pageList
```

`pageList` is the unified surface for activity / category / search listings (`flowType` discriminates). Two sibling SSR-only endpoints (`pagelist-activity`, `pagelist-category`) only return data when called from a server-rendered context; for client-side scraping use `pageList`.

⚠ **`www.savana.com` is NOT the API host.** Calling this same path on the web origin returns `{ret:404,msg:"Connect Timeout Error"}`. Use `api-shop-in.savana.com` (India) or `api-shop-iq.savana.com` (Iraq, untested).

### Required headers

The API expects an "h5 client" identity. Without these the backend returns `Connect Timeout Error` (200 OK at HTTP, ret 404 at the body level):

```
Content-Type: application/json;charset=UTF-8
Accept:       application/json, text/plain, */*
Accept-Language: en-IN,en;q=0.9
User-Agent:   Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36
Origin:       https://www.savana.com
Referer:      https://www.savana.com/
client_type:  h5
country-language: en-IN
h5-version:   6.34.0
app_version:  6.34.0
x-platform:   web
x-source:     h5
```

Cookies are NOT required for this endpoint. `uuid`, `vtoken`, `token` headers are emitted by the SPA but the backend doesn't enforce them for read-only listing calls.

### Request body

```json
{
  "flowId": "12769",
  "flowType": "ACTIVITY",
  "quickOptionId": null,
  "screenParam": {},
  "pgType": null,
  "multiTopGoodsIdList": [],
  "supportedFeatures": {},
  "pageIndex": 1,
  "visitedGoodsIdList": [],
  "pageSize": 20
}
```

`flowType` enum: `"ACTIVITY"` | `"CATEGORY"` | `"SEARCH"`.

### ⚠ Pagination — `visitedGoodsIdList` is the cursor

The server is **not page-offset-driven**. Holding `pageIndex` constant and `visitedGoodsIdList` empty returns the SAME 20 items forever. The client must echo every `goodsId` it has seen back to the server on each subsequent request. `pageIndex` is incremented for analytics but plays no role in deduplication.

`data.totalSize` is a **moving estimate** — it grows during the walk (started at 500 on page 1, climbed to 2,710 around page 121, finally stabilised at 2,707 when `hasNextPage` flipped false). Do NOT use it as a fixed ground-truth count. Stop conditions:

1. `data.hasNextPage === false` (canonical)
2. Three consecutive pages with zero new items (safety break)

The dresses-12769 collection walks in ~136 pages × pageSize 20 → 2,707 unique products, ~315 s with 150–350 ms jitter pacing.

## Field inventory (response per goods row)

| Source | Type | Promoted as | Notes |
|---|---|---|---|
| `goodsId` | number | `id` | e.g. 2233122 |
| `goodsName` | string | `name` | e.g. "Tie Up A-Line Dress" — one logical dress repeats N times in the listing, once per colour variant (each gets its own goodsId) |
| — | — | `url` | constructed: `https://www.savana.com/details/<slug-of-goodsName>-<goodsId>` |
| `salePrice` | string `"1490"` | `originalPrice` | rupees, numeric string (NOT a number) |
| `promotePrice` | string `"1241"` | `currentPrice` | post-discount price |
| `salePriceText` | string `"₹1,490"` | (currency parsed from prefix) | for display |
| `discountValue` | number\|null | `discountPercent` | e.g. 17 (means 17% off); null on non-discounted items |
| `discountValueText` | string\|null | `discountLabel` | e.g. "17%OFF" (red badge top-left) |
| `bigSalesPrice.priceSymbolText` | string\|null | `bigSalePrice` | e.g. "₹1,390" — the "Get it for ₹X" green text on the card (a STACKABLE further promo, NOT the same as discountValue) |
| `supportPurchase` | boolean | `inStock` | false ⇒ out of stock / cannot purchase |
| `warmUpProduct` | boolean | `isWarmUp` | true ⇒ pre-launch / coming-soon |
| `hitBigSaleEvent` | boolean | `isBigSaleEvent` | true ⇒ part of an ongoing site-wide sale event |
| `tagList[].tagType` | string[] | `tags` | observed types in dresses-12769: only `"fastDelivery"` |
| `tagList → fastDelivery` | — | `hasFastDelivery` | The "⚡ Fast delivery" badge on cards. Per Savana's shipping policy: **3-5 business days**, free shipping when cart ≥ ₹990. |
| `tagList → bestSeller`/etc | — | `isBestSeller` | reserved — no items in dress activity carry it; tag-type taxonomy is open-ended on the server |
| `imageList[].goodsThumb` | string | `images[].url` | one entry per colour variant (full size, `_w540_h720_q85`) |
| `imageList[].colorPic` | string | `images[].swatchUrl` | small swatch (`_w96_q85`) |
| `imageList[].select` | boolean | `images[].selected` | true on the primary variant |
| `skcPGs` | string | `skcPGs` | `<skcId>-<pgId>` variant identifier |
| `itemTrack` | JSON string | (unused) | analytics blob — contains `activityId`, `sceneId`, `cid` (numeric category id), `mainColorId`, `subColorId`, `bucketId`, `cvr`, `ctr`, etc.; preserved in `_raw` |

## Activity discovery — `pageDetail` on the homepage

The full catalog of curated activities is surfaced by ONE call:

```
GET https://api-shop-in.savana.com/n/api/buyer/guide/user/module/pageDetail
    ?sceneType=INDIVIDUATION_H5_HOME&defaultData=0&pageSize=-1
```

Same headers as `pageList`. Returns `data.moduleList` (22 modules in a single capture) — each module is a homepage section (`IMAGE_ONLY_BANNER`, `HORIZONTAL_GALLERY`, `JUST_N`) with `jumpUrl` strings pointing to `/activity/<id>`. DFS-scan every string field for `/activity/<digits>` matches.

⚠ The homepage uses A/B testing — different sessions may surface different activity sets. For exhaustive discovery, run multiple times and union the results.

### Activities observed in a single capture (June 2026)

25 unique activities, all with `status: NORMAL`:

| ID | Title | First-page totalSize | Notable |
|---:|---|---:|---|
| 11351 | Buy 1 get 1 free | 384 | **carries the `marketing` text-tag** ("Buy 1 get 1 free", red on pink) |
| 12024 | T-Shirts | 500 | |
| 12025 | Tops & Blouses | 500 | |
| 12026 | Denim | 500 | |
| 12027 | Co-ords | 500 | |
| 12028 | Bottoms | 500 | |
| 12029 | Jewelry | 500 | |
| 12031 | Accessories | 500 | |
| 12033 | Activewear | 500 | |
| 12034 | Beauty | 500 | 95% of items carry a `discountValueText` |
| 12063 | Loungewear | 500 | |
| 12064 | Lingerie | 500 | |
| 12307 | NEW DROPS | 500 | ⚠ only ~5% have `fastDelivery` — newly-dropped items aren't fast-delivery eligible yet (likely still being stocked into the domestic warehouse) |
| 12526 | Under ₹500 | 500 | |
| 12643 | TOPS | 500 | |
| **12769** | **Dresses** | 500 | **our deep-walked example — 2,707 products** |
| 12815 | Throw it on. Mean it. | 500 | |
| 12820 | Soft Girl | 500 | |
| 12821 | Game Day Glam | 210 | smaller curated drop |
| 12823 | Office Siren | 203 | smaller curated drop |
| 12824 | Sunset Mode | 500 | |
| 12981 | Stadium vibe. | 350 | |
| 12982 | Festival energy. | 285 | |
| 12983 | Afterparty glow. | 335 | |
| 12984 | Vintage era. | 384 | |

(`initialTotalSize` from page 1; the real walk-to-completion size is usually 5-7× larger because the activity feed folds in recommendations — same pattern as the dresses-12769 walk: 500 → 2,707.)

## Activity-page metadata (top-level fields we now surface)

These live alongside `data.goodsList` and we initially ignored them. The enhanced extractor (`extractSavanaActivityMeta`) lifts:

| Field | Source | Meaning |
|---|---|---|
| `id` | `data.id` | Activity id (number in source; we stringify) |
| `title` | `data.title` | Display title, e.g. "Dresses", "NEW DROPS" |
| `status` | `data.status` | Lifecycle — `"NORMAL"` observed across all 25 activities; SPA also handles `"WARM_UP"` / `"ENDED"` |
| `backgroundColor`, `fontColor` | `data.themeColor` | Hex theme — `"#FFFFFF"` / `"#000000"` for Dresses |
| `initialTotalSize` | `data.totalSize` | Page-1 advertised size (drifts during pagination — see Gotchas) |
| `hasNextPage` | `data.hasNextPage` | Initial pagination signal |
| `showRecommendToast` | `data.showRecommendToast` | True ⇒ "Recommended for you" toast shown after first scroll |
| `filters[]` | `data.screenList` | **Full filter taxonomy** (see below) |

Also present but currently NOT surfaced (defensive — may be null/unused): `imgTextBanner`, `topBanner`, `newCustomerExclusiveZone`, `ext`, `goodsAtmosphere`, `quickOptionList`, `supportPartialRefresh`, `emptyText`.

## Filter taxonomy — `data.screenList`

The biggest field we initially missed. **The filter dropdowns above the product grid are surfaced as a `screenList` array of filter groups, each with an `optionList`.** This is the catalog's full attribute taxonomy.

Filter groups observed across 25 activities:

| Filter | In how many activities | Example option labels |
|---|---:|---|
| **Sort** | 25/25 | Recommended, New Arrivals, Price: Low to High, Price: High to Low |
| **Price** | 25/25 | (range start, e.g. ₹489 minimum) |
| **Color** | 25/25 | Black, Red, Blue, Pink, Multicolored, Brown, Neutral, White, Yellow, Green, Purple, Grey, Orange, Metallic, Silver |
| **Size** | 25/25 | XS, S, M, L, XL, 2XL, XXL, XXS, One-Size |
| **Service** | 25/25 | "Fast Delivery" |
| **Style** | 25/25 | Elegant, Sexy, Vacation, Sweet, Casual, Extravagant, French, Retro |
| **Discount** | 24/25 | "10-30%OFF", "30-50%OFF" |
| **Occasion** | 22/25 | Date, Cocktail, Brunch, Wedding, Wedding Guest, Daily/Everyday essentials, ... (25 distinct values) |
| **Promotion** | 21/25 | activity-specific |
| **More** | 19/25 | "New" |
| **Category** | 17/25 | activity-specific sub-categories |
| **Length** | 14/25 | Short, Long |
| **Waistline** | 12/25 | Regular waist, High Waist, None |
| **Pattern** | 9/25 | activity-specific patterns |

These give you a backdoor into "type of stuff" classifications without hitting the per-product detail API.

## Per-product detail API — much richer than the listing

```
POST https://api-shop-in.savana.com/n/api/intention/item/v4/detail
     body: { "goodsId": <number> }
```

Same headers. ~15-30 KB per product. **The listing-row `tagList` is NOT replicated here** (`data.tagList` is always null) — fetch tags from `pageList`, not `detail`.

### Fields the detail call adds (lifted by `extractSavanaDetail`)

| Field | Source | Meaning |
|---|---|---|
| **Style** | `aboutProduct.aboutProductDetail[key=Style]` | e.g. `"Cottagecore, French"` — concrete style classifications on the SKU |
| **Occasion** | same | e.g. `"Date"` |
| **Length** | `Dress/Skirt Length` row | e.g. `"Maxi"`, `"Mini"`, `"Midi"` |
| **Waistline** | same | `"High Waist"` / `"Regular waist"` / etc. |
| **Fabric** | `Main Fabric` row | e.g. `"Flat Woven"` |
| **Print** | `All Over Print` row | e.g. `"Small Floral"` |
| **Fit Type** | same | `"Slim Fit"` |
| **Stretchability** | same | `"Not Stretchable"` |
| **Design Detail** | same | e.g. `"Tie Up, Lace-trimmed"` |
| **reviewCount** | `totalCommentSize` | numeric review count |
| **categoryId** | `level4CatId` | level-4 (most-granular) category id |
| **sizes / inStockSizes / outOfStockSizes** | `skus[]` filtered on `isInStock` | per-size stock state |
| **sizeStandards** | `sizeStandardList` | regional charts available (IN/EU/UK/US/BR/MEX) |
| **defaultSizeUnit** | `defaultSizeUnit` | `"IN"` or `"CM"` |
| **gallery[]** | `images[].originalPic\|picThumb` | full-resolution image URLs (1440px, vs the 540px listing thumbs) |
| **colors / colorCount** | `specs[Color].specificationValues` | available colors |
| **sizeGuideUrl** | `sizeRecommend.sizeGuideUrl` | link to detailed size-guide modal |
| **mrpLabel / taxLabel** | `priceMrpText`, `priceTaxText` | "MRP" / "Inclusive of all taxes" |
| **vipPrice** | `vipPrice` | VIP-tier pricing if applicable |

Plus `attributes` is an open-ended `Record<string,string>` capturing EVERY key→value row from `aboutProductDetail` — so anything Savana surfaces in the future is preserved without code changes.

### Cost note

The detail call is 1 round-trip per product. For 2,707 dresses that's ~5 min serial. Only fetch detail for the subset you actually want to drill into (e.g. filter first via the listing on `discountPercent > 30` or `tags.includes("marketing")`).

## Tag taxonomy across all 25 activities

Observed tag types in the listing API (`data.goodsList[].tagList[].tagType`):

| `tagType` | Count (across 500 sampled goods) | Tag form | Display content |
|---|---:|---|---|
| `fastDelivery` | 417 | img | Badge image URL → renders as "⚡ Fast delivery". Per Savana's own shipping-policy page: **3-5 business days**, free shipping when cart ≥ ₹990, ₹99 below. The product-detail page also says "1-3 days in most area, enter a pincode to check" — likely the metro tier-1 case. |
| `marketing` | 20 | text | The label STRING — e.g. `"Buy 1 get 1 free"` (red on pink background per `fontColor`/`backgroundColor` fields) |

**No `bestSeller` tag exists in Savana's tag taxonomy.** The bundle code does not reference one either. The closest analogue is sorting by `Sort: Recommended` (the algorithmic relevance sort, which mixes purchase signals) — but there's no explicit best-seller flag on a per-item basis.

The extractor's `isBestSeller` boolean is reserved (matches `bestSeller` / `best_seller` / `bestseller` / `topSeller` / `hot` in `tagType`) — defensive in case Savana adds them later.

## Sibling endpoints discovered (not yet wired through scraper)

| Endpoint | Method | Purpose |
|---|---|---|
| `/n/api/buyer/guide/user/goods-flow/pagelist-category` | POST | Category page variant (SSR-style pre-data) |
| `/n/api/buyer/guide/user/goods-flow/partial-refresh` | POST | Filter / sort updates within a listing |
| `/n/api/buyer/guide/user/module/pageDetail` | GET | **Implemented in `discovery.ts`** — activity discovery + (with different sceneType) activity-page banner metadata |
| `/n/api/buyer/sr/recommend/index` | POST | Recommendation feed (`{"targetId":552,"pageIndex":N,"visitedGoodsIdList":[...]}`); same `visitedGoodsIdList` cursor pattern as `pageList` |
| `/n/api/intention/item/v4/detail` | POST | **Implemented in `detail.ts`** — per-product detail (sizes, materials, full image gallery, reviews) |
| `/n/api/buyer/sr/hotword/list/v2` | GET | Trending search terms |
| `/n/api/trade/intention/popResource` | POST `{"position":"0","addTimes":true}` | Welcome-offer / coupon popup config |
| `/n/api/node/share/tdk?path=/` | GET | SEO title / description / keywords for any path |

## Field promotions — what we did NOT promote (yet)

- `itemTrack.cid` — numeric category id (would let us cross-classify items by Savana's internal taxonomy)
- `itemTrack.sceneId` — recommendation surface label (e.g. `savana_in_activity_rec` — appears only when the SPA passes session context)
- `itemTrack.mainColorId` / `subColorId` — colour taxonomy
- `imageList[].videoThumb` — short-form video showcase (often null in listings, populated on detail pages)
- `data.quickOptionList` / `data.screenList` — filter chips & sort options shown above the grid

## Files

### Extractor modules (pure functions, reusable)
- `scrapers/savana/activity.ts` — pageList request + `extractSavanaProduct(s)` + `extractSavanaActivityMeta` (title/filters/status/theme) + tag-content map
- `scrapers/savana/discovery.ts` — `fetchSavanaHomeModules()` + `discoverActivitiesFromHome()` (homepage → every `/activity/<id>`)
- `scrapers/savana/detail.ts` — `fetchSavanaDetail()` + `extractSavanaDetail()` (style, occasion, sizes, reviews, gallery, size charts)

### Production / smoke / recon scrapers
- `scrapers/savana-activity.ts` — production scraper, full activity walk
- `scrapers/savana-activity-test.ts` — single-page smoke test
- `scrapers/savana-discover-activities.ts` — homepage → per-activity summary (titles, filters, page-1 stats)
- `scrapers/recon-savana.ts` — Phase-1 XHR-capture recon (used to find the API host)
- `scrapers/recon-savana-discovery.ts` — Phase-1 recon for activity discovery
- `scrapers/recon-savana-homepage.ts` — Phase-1 deep XHR capture for homepage endpoint detection

### Fixtures
- `__tests__/fixtures/savana-activity-dresses-12769-page1.json` (20 dresses, fastDelivery tags, 11 filter groups)
- `__tests__/fixtures/savana-activity-bogo-11351-page1.json` (20 BOGO items, marketing text-tags)
- `__tests__/fixtures/savana-home-modules.json` (22 homepage modules → 25 activities)
- `__tests__/fixtures/savana-detail-2218022.json` (full v4/detail response — "Tie Up A-Line Dress")

### Tests (31 passing, 576 assertions)
- `__tests__/savana-activity.test.ts` (13 — pricing, stock, discount, tags, request shape)
- `__tests__/savana-discovery.test.ts` (8 — homepage parse, activity metadata, filter taxonomy, tag-content surface)
- `__tests__/savana-detail.test.ts` (10 — aboutProduct attributes, sizes/stock, gallery, MRP, review count)

## Run

```bash
# 1. Discover every activity on the homepage + summarise each
bunx vinxi-scraper run savana-discover-activities
bunx vinxi-scraper export savana-discover-activities --format json
# → 25 activities with titles + filter taxonomy + page-1 stats

# 2. Smoke test a single activity page (~0.5 s)
bunx vinxi-scraper run savana-activity-test --activity=12769

# 3. Full activity walk — 2,707 dresses, ~5 min
bunx vinxi-scraper run savana-activity --activity=12769

# Variations
bunx vinxi-scraper run savana-activity --activity=https://www.savana.com/activity/dresses-12769
bunx vinxi-scraper run savana-activity --activity=11351   # BOGO collection
bunx vinxi-scraper run savana-activity --activity=12526   # Under ₹500
bunx vinxi-scraper run savana-activity --activity=12769 --max-pages=5

# Export
bunx vinxi-scraper export savana-activity --format json
```

### Fetching rich per-product details (Tier-2 + 1)

Once you've narrowed a subset via the listing, fetch product details one-by-one:

```typescript
import { fetchSavanaDetail, extractSavanaDetail } from "./scrapers/savana/detail.js"
const resp = await fetchSavanaDetail(2218022)
const detail = extractSavanaDetail(resp)
// → { style: "Cottagecore, French", occasion: "Date", length: "Maxi",
//     sizes: ["XS","S","M","L"], inStockSizes: [...], reviewCount: 20,
//     gallery: [5 × 1440px URLs], colors: [...], sizeGuideUrl, ... }
```

## Gotchas

1. **API host swap is mandatory.** `www.savana.com` accepts the path but returns ret 404 "Connect Timeout Error". Use `api-shop-in.savana.com`.
2. **`visitedGoodsIdList` is the cursor, not `pageIndex`.** Skipping it ⇒ same 20 items in an infinite loop.
3. **`data.totalSize` is a moving estimate** — do NOT use it as a fixed corpus size. Trust `hasNextPage` + a streak-of-zero-new safety break.
4. **One `goodsId` per colour variant.** The same logical dress (`goodsName: "A-Line Dress"`) appears multiple times in the listing — once per colour. Dedup-by-name will collapse colour variants; dedup-by-`goodsId` keeps them.
5. **Tag taxonomy is server-controlled.** Across 25 activities we observed exactly TWO tag types: `fastDelivery` (img) and `marketing` (text). Treat `tagList[].tagType` as open-ended — surface every value you see and let downstream code interpret it. Tag CONTENT lives in `tagList[].content` (URL for img-tags; label text for text-tags) — surfaced via `tagContent[tagType]` in the extractor.
6. **"Fast delivery" = 3-5 business days from a domestic warehouse.** Source: Savana's own shipping-policy FAQ at `/help-center/shipping-delivery` (fetched via `/n/api/buyer/mk/about/query`, content id 17): *"you can place an order for fast delivery products as the delivery time for those is 3-5 business days."* The product-detail-page notice for fast-delivery SKUs says "1-3 days delivery in most area, enter a pincode to check" — likely tier-1 metro case. Free shipping when cart ≥ ₹990 else ₹99 fee. Note: the fast-delivery promise applies only when the order is "fast-only" — mixing in a standard-delivery item delays the whole order. Standard-delivery day-count is PIN-dependent and not stated in the policy.
7. **`promotePrice` ≠ "Get it for" price.** The card shows TWO discounts: `discountValue` is the headline % off (drives the red "20%OFF" badge); `bigSalesPrice.priceSymbolText` is a STACKABLE further promo ("Get it for ₹X" in green). Most items have `bigSalePrice = currentPrice − 100`.
8. **Activity vs recommendations.** Beyond the strict curated set (likely ~100 items per activity), the API seamlessly folds in recommendations under the same `flowId`. There is no field to discriminate — every item we observe carries `itemTrack.activityId == "12769"` even when sourced from a recommendation bucket. If "strict collection only" matters, you would need to compare against `data.totalSize` from page 1 BEFORE pagination drifts (it started at 500 here, very different from the actual 2,707 walked).
9. **No best-seller tag.** Savana's tag taxonomy does NOT include a best-seller marker. The bundle source code references no such tagType. Closest proxies (none of them ideal): `Sort: Recommended` (algorithmic), `totalCommentSize` from the per-product detail API (review count as popularity signal), or `discountPercent` magnitude (deep discounts often correlate with end-of-life inventory, not best-sellers).
10. **Homepage is A/B-tested.** `pageDetail?sceneType=INDIVIDUATION_H5_HOME` returns different module sets across sessions. For exhaustive activity discovery, run the discovery scraper repeatedly (with fresh `__cf_bm` cookie / new IP) and union the resulting id sets.
11. **`status: "NORMAL"` is the only state observed.** The bundle handles `WARM_UP` and `ENDED` lifecycle states for activities (e.g. countdown vs ended messages). Watch for these on time-limited promo activities — the extractor's `meta.status` field surfaces whatever the source emits.
12. **`screenList` is the BACKDOOR catalog.** Every filter group surfaces the taxonomy values used for that activity — useful for cross-classifying products WITHOUT having to fetch per-product detail. e.g. a dress activity's `Style` filter has [Elegant, Sexy, Vacation, Sweet, Casual, Extravagant, French, Retro], and your listing scraper can keep these as the canonical style universe even though individual `goodsList` rows don't carry the style tag.
