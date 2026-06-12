# Savana — business intelligence

Companion to [savana.md](./savana.md). The first doc covers HOW to scrape Savana; this one covers WHO they are, WHAT they sell, and HOW they run promos. Captured 2026-06-10/11 via the recon scrapers in this repo.

---

## TL;DR

- **Savana is Urbanic, rebranded.** Same backend (`@ub/*` packages, `mfrcdn.com` CDN, Urbanic Buyer Component framework, Urbanic legal team). Savana is the consumer-facing brand for **India + Iraq**; the **UK, US, Canada, Mexico, Brazil, South Africa** still operate under `urbanic.com`.
- **Catalog scale: 42,172 product detail pages** in the India sitemap. Women's fast-fashion (≈85% clothing, ≈10% jewellery + accessories, ≈5% beauty).
- **1,097 active "activity" feeds** running concurrently. Only 25 (2.3%) surface on the homepage; the other 1,072 are deep-link landing pages (date-scheduled flash sales, mood collections, sub-category drilldowns, retargeting campaigns).
- **Pricing sweet-spot: ₹500–2,000** (95% of dresses; median ₹1,390, mean ₹1,450). Range ₹489–₹7,590. Sub-₹500 inventory is essentially absent in the Dresses activity (only 1 of 2,707).
- **Discount mechanics — three-layer:** (1) the strike-through `salePrice` → `promotePrice` % off; (2) the "Get it for ₹X" stacked further promo (86% of dresses, ~₹114 extra savings); (3) authenticated coupons gated behind login (`popResource` returns ret=100 anonymously).
- **Logistics — 3-5 business day fast delivery on 89% of products** (per Savana's own shipping policy), shipped from a domestic warehouse. Free shipping when cart ≥ ₹990, else ₹99 fee. ~5% of newly-dropped items aren't fast-delivery eligible yet.
- **Stock state observable: 100% in stock.** Out-of-stock items are dropped from listings entirely (the only OOS signal is the per-size `isInStock` flag inside the per-product detail API).
- **No best-seller tag exists.** Closest popularity proxies: `Sort:Recommended` (algorithmic), `totalCommentSize` from detail API (review count), or simply repeated colour-variant SKUs of the same base design (e.g. "A-Line Dress" appears as 198 different colourway SKUs).
- **AI chatbot integrated** at `api-aichat-in.savana.com` (`api-aichat-in.urbanic.com` legacy still hardcoded — transitional).

---

## 1. Parent company

| | |
|---|---|
| Brand (consumer-facing) | **Savana** (India, Iraq) / **Urbanic** (UK, US, CA, MX, BR, ZA) |
| Tagline | "Savana – Fashion from London." |
| Legal entity (per bundle hits) | Urbanic (`legal@urbanic.com`, `legal.india@urbanic.com`, `prensa@urbanic.com`) |
| Founding / model | London-headquartered (per Urbanic's positioning); design sourcing via Urbanic's existing supply chain; **domestic Indian warehouse for fulfilment** — 3-5 day fast-delivery confirmed in their own shipping policy |
| Tech-stack identity | "Urbanic Buyer Component" — bundle prefix `@ub/ubc-rwd-*` (rwd = Responsive Web Design) |
| App framework | `rwd-guide` v6.34.0, `rwd-home` v7.40.3, served via `static.mfrcdn.com` |
| React version | 17.0.2 |
| AI chatbot vendor | Custom — `api-aichat-in.savana.com`, token `KBwbSBiXn7LSZR5xdYmcVfeC` |
| Mobile apps | iOS `id6476902707` (Savana IN), `id6630376768` (Savana ME), plus legacy Urbanic IDs `id1466063983`, `id1571882009`. Android: `com.savana.in`, `com.savana.me`, legacy `com.urbanic`, `com.urbanic.multis`. |
| Attribution SDK | Adjust (deeplinks `linkin.urbanic.com/u/newuserdownload`, `linkmx.urbanic.com/u/newuserpopupmx`) |
| Analytics fingerprints | Facebook domain-verification (`2bddrgkymb8tvaligwsjqu97xknbjx`), Pinterest verification, async-loaded GTM + facebook.net, **TikTok pixels (6 IDs spotted in homepage XHRs)** |
| Payments | Paytm logo asset only — no SDK fingerprint (server-side checkout) |
| Shipping providers | None hardcoded — server-side fulfillment |

### Geographic footprint (per i18n bundles loaded from `i18n-o3.mfrcdn.com`)

| Locale | Brand | Site | Currency notes |
|---|---|---|---|
| en-IN | Savana | savana.com | ₹ INR — sweet spot ₹500-2k |
| ar-IQ | Savana | iq.savana.com (Arabic RTL) | IQD (Iraqi Dinar) — `country-language:en-IQ` for API |
| en-GB | Urbanic | urbanic.com | £ |
| en-US | Urbanic | urbanic.com | $ |
| en-CA | Urbanic | urbanic.com | $ CAD |
| en-ZA | Urbanic | urbanic.com | R (South Africa) — newer market |
| es-MX | Urbanic | urbanic.com (Spanish) | $ MXN |
| pt-BR | Urbanic | urbanic.com (Portuguese) | R$ BRL |

`urbanic.com` → 302 → `in.urbanic.com` → 500 (= India migrated off `urbanic.com` to `savana.com`).

---

## 2. The activity universe — 1,097 running activities

Sampled `flowId` 10000-14000 against the `pageList` API. Real-activity threshold: `ret==200 AND data.title != null AND data.totalSize > 0`.

| | |
|---|---|
| Total live activities | **1,097** |
| Activity ID range | 10006 – 13001 (3-year accumulation; ramp-up around 12100) |
| Surfaced on homepage | 25 (2.3%) |
| Hidden (deep-link only) | 1,072 (97.7%) |
| Unique titles | 920 (so ~177 duplicates — A/B variants or staged rollouts) |
| Median totalSize (first-page advertised) | 500 |
| Activities with first-page totalSize=500 (= capped feeds — actual catalog flows in via recommendation pagination) | 602 / 1097 (55%) |
| Activities with first-page totalSize 1-99 (small curated drops) | 153 |

### Activity classification (regex on titles)

| Pattern | Count | Notes |
|---|---:|---|
| **Flash sale (dated)** | 104 | Daily / sub-daily cadence — `Flash sale 14/06`, `Flash sale 21/06`, ..., spanning the calendar year. Programmatic deal engine. |
| **Mood / aesthetic** | 73 | "Soft Girl", "Office Siren", "Sunset Mode", "Stadium vibe.", "Vintage era.", "Afterparty glow.", "Festival energy." — Pinterest-style aesthetic edits |
| **Up to N% off** | 71 | Discount-themed (20-80% off ranges) |
| **New arrivals / drops** | 40 | "NEW DROPS", "Just landed", "Fresh in" |
| **Color theme** | 32 | "Pastel Colors", "Yellow dress", "Black edits", "White edits 🤍", "Metals" |
| **Under ₹N** | 27 | "Under ₹500", "Under ₹990", "Starting at ₹90" — entry-price funnels |
| **Clearance / urgency** | 24 | "Last Chance", "Final Call", "Almost Gone", "Selling Fast" |
| **Monsoon (India seasonal)** | 9 | "Monsoon Style Edit" (×6, A/B variants), "Monsoon-Ready", etc. India-only psychology |
| **BOGO / buy-X-get-Y** | 7 | "Buy 1 get 1 free" (×5 IDs), "2 for ₹990" (×2 IDs) |
| **Restock / back-in-stock** | 7 | "Back in Stock", "Comeback" |
| **Members / VIP / loyalty** | 5 | **"Members First: New Arrivals, 24h Early Access"** — confirms loyalty tier exists |
| **Birthday / occasion** | 4 | "Birthday Dresses", "Birthday looks" |
| **Anniversary / milestone** | 4 | "DAY 30", "1111" (Singles' Day reference), "Anniversary" |
| **Wedding / occasion** | 1 | "Wedding Guest", etc. |
| **Festive / Kurta / Diwali / Eid** | 1 | "Festive Kurtas" — India-only |
| Other / pure category | 724 | Sub-category drilldowns ("Tube Dresses", "Embroidered Jeans", "Bag Charms", "Maxi dresses", "Cropped", "Co-ords & Jumpsuits", etc.) |

### Read of the playbook

1. **Daily flash sales** are a core conversion engine. 104 dated activities → roughly 2 flash sales/week running year-round.
2. **Mood / aesthetic edits** (73) replace the older Pinterest-board model — Savana curates by *vibe* ("Office Siren", "Soft Girl") instead of just demographics or categories.
3. **Urgency framing** (24 clearance + 9 monsoon + 7 restock = 40) is omnipresent — every Savana page has at least one "running out" signal.
4. **Members tier** is real but anonymous-API-gated. The "24h Early Access" hint implies the membership funnel is deeper than what we can probe without a session.
5. **A/B testing on titles** is heavy — "Monsoon Style Edit" exists at 6 distinct activity IDs, "Buy 1 get 1 free" at 5. Same content, different ranking buckets.

### Activity IDs to walk for an "everything-running" sweep

```
data/savana-all-activities.json          — list of all 1,097 {id, title, status, totalSize}
data/savana-activities-labeled.json      — same + regex-classified category tags
```

Pass any of these IDs to `savana-activity --activity=<id>` to deep-walk one.

For walking many activities at once (with cross-activity dedup + membership map), use:

```bash
# Strategic 12-activity sample (homepage + BOGO + Beauty + smaller moods)
bunx vinxi-scraper run savana-walk-many --ids=sample

# All 25 homepage activities
bunx vinxi-scraper run savana-walk-many --ids=homepage

# All 1,097 activities (with skip-subsumed bailout — runs faster than naïve N×walk)
bunx vinxi-scraper run savana-walk-many --ids=all

# A specific subset by category
bunx vinxi-scraper run savana-walk-many --ids=bogo        # 7 BOGO activities
bunx vinxi-scraper run savana-walk-many --ids=flashsale   # 104 dated flash sales
bunx vinxi-scraper run savana-walk-many --ids=members     # 5 members-only activities
bunx vinxi-scraper run savana-walk-many --ids=clearance   # 24 clearance activities

# Custom list
bunx vinxi-scraper run savana-walk-many --ids=12769,11351,12526
bunx vinxi-scraper run savana-walk-many --ids-file=/tmp/ids.txt
```

### Sample-walk result (12 activities)

Captured 2026-06-11. Strategic preset = `--ids=sample`:

| ID | Title | Pages | Returned | New | Prior | Reason |
|---:|---|---:|---:|---:|---:|---|
| 12769 | Dresses | 137 | 2,735 | 2,735 | 0 | exhausted |
| 12024 | T-Shirts | 76 | 1,502 | 1,502 | 0 | exhausted |
| 12028 | Bottoms | 66 | 1,315 | 1,315 | 0 | exhausted |
| 12064 | Lingerie | 60 | 1,183 | 1,183 | 0 | exhausted |
| 11351 | Buy 1 get 1 free | 20 | 385 | 385 | 0 | exhausted |
| 12526 | Under ₹500 | 42 | 838 | 229 | 609 | exhausted (73% subsumed by T-Shirts + Dresses) |
| 12307 | NEW DROPS | 46 | 912 | 574 | 338 | exhausted |
| 12815 | "Throw it on. Mean it." | 57 | 1,130 | 580 | 550 | exhausted |
| 12823 | Office Siren | 11 | 203 | 197 | 6 | exhausted |
| 12034 | Beauty | 59 | 1,164 | 1,113 | 51 | exhausted |
| 12029 | Jewelry | 73 | 1,447 | 1,205 | 242 | exhausted |
| 12031 | Accessories | 146 | 2,916 | 2,590 | 326 | exhausted |
| **Total** | | | **15,730** | **13,608 unique** | **2,122 redundant** | 13.5% dedup hit |

### Membership map — how often does a product appear in multiple activities?

From the 12-activity sample:
- **85.4% of products live in EXACTLY ONE activity** (11,624 of 13,608)
- 13.6% are in 2 activities (1,846)
- 1.0% are in 3 activities (138)

So Savana's activity feeds are mostly DISJOINT — each category/mood/promo activity surfaces its own product set. The overlap mostly happens between (1) the canonical category and (2) cross-promoting moods/price-tiers, e.g. T-Shirts × Under ₹500 × "Throw it on. Mean it." cluster.

### Most-cross-promoted products (sample)

The products that appear in the MOST activities are mid-tier T-shirts under ₹500 — they're surfaced in the canonical T-Shirts feed AND the "Under ₹500" budget feed AND the "Throw it on. Mean it." mood feed simultaneously. Savana clearly tags its "value champion" SKUs for triple-surface promotion.

```
2221092  ₹490  Pullover T-Shirt                 in [12024, 12526, 12815]
2252932  ₹490  Pullover T-Shirt                 in [12024, 12526, 12815]
2119072  ₹343  Printed Pullover T-Shirt         in [12024, 12526, 12815]
1777662  ₹392  "TRIPLE FUNCTION" SILKY HANDFEEL in [12024, 12526, 12815]
2227372  ₹690  Drawstring Pullover T-Shirt      in [12024, 12307, 12815]   ← also in NEW DROPS
2248102  ₹690  Rhinestone Pullover T-Shirt      in [12024, 12307, 12815]
```

### Output files

```
data/savana-sample12-catalog.json        — 13,608 products with activityIds[] each (sample run)
data/results/savana-walk-many/*.jsonl    — raw walker output: one record per unique product + one per activity-summary
```

---

## 3. Catalog mix — 42,172 product pages

Source: `https://www.savana.com/sitemap.in.xml.gz` (8.2 MB gzipped, 45,005 URLs).

| URL prefix | Count | Notes |
|---|---:|---|
| `/details/<slug>-<goodsId>` | 42,172 | Product detail pages |
| `/search/result/<query>` | 2,824 | SEO-canonical search landing pages (`tops under 500`, `dresses under 500`, `phone cover`, `bracelet`, ...) |
| `/category/<slug>` | 8 | Top-level cats: new-in, clothing, dresses, knitwear, denim, sports, swimwear, special-prices |
| `/` | 1 | Homepage |

### Top product-name keywords (across 42,172 slugs)

| Keyword | Count | Keyword | Count |
|---|---:|---|---:|
| dress | 6,741 | knit | 1,932 |
| top | 5,204 | jewellery | 1,872 |
| set | 4,125 | straight | 1,768 |
| line (A-line) | 3,477 | ord (co-ord) | 1,765 |
| pullover | 3,111 | bodycon | 1,715 |
| shirt | 3,085 | pants | 1,635 |
| leg | 2,702 | lace | 1,624 |
| bag | 2,414 | size | 1,515 (mostly "plus size") |
| bow | 2,396 | plus | 1,511 |
| artificial | 2,307 (artificial jewellery) | printed | 2,065 |

**Read:** women's clothing first (dresses, tops, A-line/bodycon/straight cuts, knits, co-ords, denim), with bags, jewellery, "artificial" (= imitation/costume jewellery), bows, and plus-size as second-tier breadth.

### Dresses activity deep-cut analytics — `dresses-12769` (2,707 SKUs walked)

| | |
|---|---|
| Unique product names | **501** (= 501 distinct dress designs) |
| Single-variant products | 304 (60% of designs sold in one colour only) |
| Multi-variant products | 197 (designs offered in 2-198 colours) |
| Top variant counts | "A-Line Dress" → 198 SKUs · "Bow A-Line Dress" → 148 · "Backless A-Line Dress" → 127 · "Bodycon Dress" → 92 · "Ruffle A-Line Dress" → 79 |

### Price distribution (Dresses, ₹)

```
₹    0-500   :    1   (<0.1%)
₹  500-1000  :  301   (11.1%)
₹ 1000-1500  : 1443   (53.3%)  ← bulk
₹ 1500-2000  :  835   (30.8%)
₹ 2000-3000  :  100   ( 3.7%)
₹ 3000-5000  :   24   ( 0.9%)
₹ 5000-10000 :    3   ( 0.1%)
```
Median ₹1,390 · Mean ₹1,450 · Min ₹489 · Max ₹7,590.

### Discount mechanics (per-SKU on Dresses)

| Layer | Coverage | Median value |
|---|---:|---|
| `discountValueText` ("17%OFF" red badge) | 809 / 2,707 (29.9%) | 10-20% |
| `bigSalesPrice` ("Get it for ₹X" green text) | 2,332 / 2,707 (86.1%) | ~₹114 extra savings |
| Both layers stacked | majority of discounted SKUs |
| `hitBigSaleEvent` (site-wide sale flag) | 2,707 / 2,707 (100%) |

### Logistics

| | |
|---|---|
| In stock (`supportPurchase`) | 2,707 / 2,707 (100%) — listings filter OOS items |
| `fastDelivery` tag (3-5 business days, domestic warehouse) | 2,407 / 2,707 (89%) |
| Warm-up (pre-launch) | 0 |
| Image CDN | All 2,707 images on `img105.savana.com` (legacy `img101.urbanic.com` etc. still resolve) |

---

## 4. Tag taxonomy (full universe)

Observed across 25+ activities (page-1 sample of each):

| `tagType` | Form | Display | Where seen |
|---|---|---|---|
| `fastDelivery` | img | Badge image (resolves to "⚡ Fast delivery" UI) = **3-5 business days** per Savana's own shipping policy (`/help-center/shipping-delivery`); free shipping when cart ≥ ₹990 | ~89% of products across all activities |
| `marketing` | text | Label STRING (`"Buy 1 get 1 free"`, red on pink) | Only on BOGO activities (IDs 11351, 12002, 12077, 12204, 12265, 11377, 12078) |

**`bestSeller` does NOT exist** in Savana's data model. The bundle has zero references to a best-seller tag-type. Closest proxies for popularity:

1. `Sort:Recommended` — Savana's algorithmic relevance sort (mixes CTR/CVR signals server-side)
2. `totalCommentSize` from per-product detail API (review count as popularity proxy — but requires per-product round-trips)
3. Colour-variant repetition — designs like "A-Line Dress" with 198 SKUs are bestsellers structurally; one-variant designs are tails

---

## 5. Filter taxonomy — Savana's product attribute model

Lives on every activity's `data.screenList`. Up to 14 filter groups, ~25-100 option values per group. Used internally to classify every SKU even when the per-SKU rows don't carry the tags directly.

| Filter | Possible values |
|---|---|
| **Sort** | Recommended, New Arrivals, Price: Low to High, Price: High to Low |
| **Price** | (continuous, ₹489-₹7,590 observed) |
| **Color** | Black, Red, Blue, Pink, Multicolored, Brown, Neutral, White, Yellow, Green, Purple, Grey, Orange, Metallic, Silver |
| **Size** | XXS, XS, S, M, L, XL, 2XL, XXL, One-Size (+ plus sizes via slug analysis) |
| **Service** | Fast Delivery |
| **Style** | Elegant, Sexy, Vacation, Sweet, Casual, Extravagant, French, Retro |
| **Discount** | 10-30%OFF, 30-50%OFF |
| **Occasion** | Date, Date Night, Cocktail, Birthday, Resort / Vacation, Brunch, Club, Dinner party, Girl's Night out, Ballroom, Wedding, Wedding Guest, Daily/Everyday, Party/Gala, New Year's Eve, Holiday Party, weekend getaways, Picnic, Formal, Beach, Carnival, Convocation/Graduation, Red carpet, Valentine, travel |
| **Promotion** | activity-specific |
| **Category** | activity-specific sub-categories |
| **Length** | Short, Long |
| **Waistline** | Regular waist, High Waist, None |
| **Pattern** | activity-specific (florals, polka, abstract, etc.) |
| **More** | "New" |

Per-product detail API additionally surfaces (via `aboutProduct.aboutProductDetail`):

- Style (sub-style: "Cottagecore, French")
- Occasion (more granular than the filter)
- Length (Maxi, Midi, Mini)
- Waistline (Regular, High Waist, None)
- Main Fabric (Flat Woven, Knit, Crepe, etc.)
- All Over Print (Small Floral, Polka, Abstract, etc.)
- Fit Type (Slim, Regular, Loose)
- Stretchability (Stretchable, Not Stretchable)
- Design Detail (Tie Up, Lace-trimmed, Backless, etc.)
- Fabric weight (Light, Medium, Heavy)

These attributes give you a per-SKU taxonomy WITHOUT needing the listing tagList — at the cost of one round-trip per product.

---

## 6. Promotional + coupon system (anonymous-API-gated)

`POST /n/api/trade/intention/popResource` with `{"position":"0","addTimes":true}` returns the **"Welcome Offer 20% off"** modal when called from a session WITH browser fingerprint cookies — but returns `ret:100, msg:"wrong user"` when called from a plain anonymous probe. Coupons are **session-gated**.

What we DO know (from one browser-captured response):

- **Welcome Offer:** 20% off, threshold ≥ ₹500, on selected items, applies to first-time visitors
- Coupon objects ship `{id, popupMsgList, thresholdAmount, validity}` — the SPA renders a popup modal on first session

Positions:
- `position=0` (with `addTimes:true`) — welcome offer popup
- `position=1` — second-trigger popup (likely cart-add or exit-intent)
- `position=2` — third-trigger
- `position=3+` — invalid (`wrong position param`)

To enumerate live coupons exhaustively you'd need to authenticate. As of this recon, the only publicly-observable coupon is the welcome 20%-off.

### Other promo signals on the homepage

- Homepage hero (`type:IMAGE_ONLY_BANNER`) — full-bleed editorial banners (5 currently rotating)
- "App-download" yellow popup (4.8 rating) — pushes to iOS/Android apps with the Adjust deeplink
- TikTok pixel (6 IDs `CLS2KHRC77U2H4CKK5BG`, `C7IJKMISLUCN3VE6H72G`, `CLSK8QBC77U441RFG8SG`, `CU6TVCBC77UBK8D3LGD0`, `CUKS31RC77U6K9SHTGR0`, `CVLO5C3C77UEAE54R1P0`) — heavy social-attribution fingerprinting

---

## 7. Search & trends

`GET /n/api/buyer/sr/hotword/list/v2` returns trending search terms.

**Curated trend section** (each links to an activity):
- Pastel Colors
- Summer 2026 Edit
- Birthday Dresses
- Sun-proof add-ons
- Polka Dots

**Algorithmic auto-complete** (from user search behavior):
- Dresses western, Co ord set, Phone case, Nails, Bra, Yellow dress, Jeans, Skirt, Bags for ladies, Full sleeve top

**Sitemap-encoded search keywords (2,824 SEO landing pages):**
- High-volume: women, dress, shirt, top, jeans, tops, pants, black, dresses, jacket, party, white, long, skirt, sweater, ladies, denim
- Price-anchored: "tops under 500", "tops under 300", "dresses under 500", "dresses under 1000", "dress under 300", "kurti under 300"

**Read:** the price-anchored search SEO is heavy — Savana clearly competes on Google for "X under ₹N" queries. The Indian market is highly price-sensitive and Savana is positioned as the affordable London-brand alternative to local fast-fashion.

---

## 8. Iraq market (savana.com/iq) snapshot

One probe: `GET https://api-shop-iq.savana.com/n/api/buyer/guide/user/module/pageDetail?sceneType=INDIVIDUATION_H5_HOME&defaultData=0&pageSize=-1` with `country-language: en-IQ`.

- **23 homepage modules** (vs India's 22) — slightly different layout
- **Zero module-ID overlap** with India — completely separate merchandising
- **IQD currency hints** present in 12 places (vs zero `INR` mentions in India payload — currency rendered client-side via locale)
- Iraq `jumpUrl`s use absolute URLs (`https://iq.savana.com/help-center/home`); India uses relative paths
- RTL Arabic strings loaded via `i18n/prod/fe/@ub/ubc-rwd-*/ar-IQ/*.json` bundles

To deep-scrape Iraq: swap the API host to `api-shop-iq.savana.com` and the `country-language` header to `en-IQ`. The pageList API shape is identical.

---

## 9. SEO & growth notes

- **`robots.txt` is restrictive:** `Disallow: /` for `User-agent: *`. Only ~30 bots are allowlisted (Googlebot, Bingbot, AdsBot, Mediapartners, Semrush, Ahrefs, Meta, Twitter, Apple, etc.). Savana is NOT crawler-friendly for unknown agents — combine with the SPA model and you get an indirect anti-bot posture.
- **SEO is NOT category-tuned:** every `/category/<slug>` returns the same boilerplate `<title>` = "Category" and identical meta description. This is a notable gap — they likely rely on the 2,824 SEO landing pages + per-product detail pages for organic acquisition rather than category indexing.
- **Per-product titles ARE indexable** — sitemap exposes 42,172 detail URLs and the `tdk` API gives per-product H1/title/description for each.
- **No newsletter / email-capture form** found in the homepage HTML (the welcome offer is the equivalent funnel).

---

## 10. What we still can't see (without auth)

| Data | Why not visible |
|---|---|
| Live coupon catalog (beyond welcome offer) | `popResource` returns `wrong user` without session |
| Member tier benefits | "Members First" exists but the loyalty API is auth-gated |
| Cart/checkout flow | Out of scope; would need to add to cart with a real session |
| Per-SKU sales / inventory velocity | Backend metrics — never exposed client-side |
| Per-product review TEXT | Listing has `totalCommentSize` count but full reviews require additional `comment/*` calls |
| True activity creation dates | `flowId` is the only timestamp proxy (linearly assigned 10006→13001 over ~1 year) |
| Sister-brand products (Urbanic UK/US/CA/ZA/MX/BR) | Different API hosts — would need parallel scraper per market |

---

## 11. Suggested next probes

If you want to push this further:

1. **Iraq deep-walk** — same scraper, swap `api-shop-iq.savana.com` + `country-language:en-IQ`. Should yield a sister catalog of ~similar size with IQD pricing.
2. **Urbanic UK/US deep-walk** — extend `discovery.ts` with the urbanic.com region hosts and walk those markets for cross-market product comparison.
3. **Detail-batch on a sample** — pick 100 random products and run `fetchSavanaDetail` to capture the rich per-SKU taxonomy (style, occasion, fabric, fit, sizes-in-stock, gallery). 100 round-trips ≈ 25 s.
4. **Activity longitudinal probe** — re-run the activity ID sweep weekly and diff to detect new flash sales, dead activities, and inventory drift.
5. **Authenticated probe** — create a test account, capture the session token, and re-call `popResource` + member endpoints to map the full loyalty + coupon surface.

---

*Captured via the scrapers in this repo. Reproduce with: `bunx vinxi-scraper run savana-discover-activities` (homepage 25), `bunx vinxi-scraper run savana-activity --activity=<id>` (deep walk any of 1,097 activities). Snapshot data files: `data/savana-all-activities.json`, `data/savana-activities-labeled.json`, `data/savana-dress-catalog-analytics.json`.*
