# Hindustan Coca-Cola Beverages (HCCB)

**Domain:** hccb.in
**Vertical:** sales-distribution / FMCG bottler (Coca-Cola India franchise)
**Last verified:** 2026-05-22
**Tier:** 1 (raw HTTP + HTML accordion / infocard parse)
**Framework:** Custom static site (NOT WordPress - locale prefixes `/en`, `/hi`, `/tel`; sitemap.xml served by xml-sitemaps.com generator)
**Protection:** none

## Sibling comparison vs VBL

HCCB is the direct structural sibling to **Varun Beverages (VBL)** - the
two are the largest bottling franchisees in India:

| Dimension | VBL (PepsiCo) | HCCB (Coca-Cola) |
|---|---|---|
| Parent | Listed (NSE: VBL) | Privately held (being divested to Jubilant Bhartia consortium 2024) |
| Plant count published | 39 (FY24 disclosure matches) | 16 (FY24 disclosure matches) |
| Page that carries the list | `/plants/` (only reachable via wp-json enumeration) | `/contact-us` (linked from main nav) |
| Source structure | One `<table>` with 5 columns (SL / Plant Name / Plant Code / Address / LIC) | THREE redundant layers (desktop accordion + map markers + mobile infocards) over a "Where Our Beverages Come to Life" section |
| Stable identifier | 1-2 letter plant code ("AA", "AG") | `dm-index` integer in mobile infocards |
| Address detail | Full FSSAI Form-B address + 6-digit pincode | Plant name + state only (no street, no pincode) |
| FSSAI licence | yes - 14-digit, kept in `features` as `fssai-lic:...` | not published |
| Lat / lng | no | no |
| Hidden-but-not-deleted entries | one (Plant Tour menu item routes to empty stub) | FOUR plants commented out of desktop layers but live in mobile infocards (Aryana, Kanchenkanya, Sanand, Raninagar - divestment-in-flight pattern) |

The two extractors emit the SAME canonical shape (`locationType:
"warehouse"`, `storeType: "bottling-plant"`) so a downstream union of
"all Indian beverage bottling plants" is a single filter.

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| Plant locator | `/contact-us` | https://www.hccb.in/contact-us |
| Sitemap | `/sitemap.xml` | https://www.hccb.in/sitemap.xml |

The site soft-404s with HTTP 200 on every guessed path
(`/our-plants`, `/plants`, `/factories`, `/manufacturing`, `/about`,
`/our-locations` all return the same `<title>404 | HCCB</title>` page).
The sitemap.xml is exhaustive - reading it is the right discovery step.
The plant data lives ONLY on `/contact-us`.

## Hydration payload

None - server-rendered HTML. THREE layers carry the same plant list:

1. **Desktop accordion** (`<div class="section-4-accordion-desktop">`):
   `<div class="accordion-item" data-marker-index="N">` cards with `<h4>`
   plant name + `<h6>` state.
2. **Desktop map markers** (`<span class="marker" data-marker-index="N">`):
   each carries `<h1>` plant name + `<h2>` state over an SVG India map.
3. **Mobile infocards** (`<div class="section-4-infocard-container" dm-index="N">`):
   the carousel layer - two `<p>` lines per card.

**The three layers use DIFFERENT numbering schemes.** Bidadi is
`dm-index=7` in the infocard but `data-marker-index=17` in the accordion.
You cannot index-join the three layers - the extractor name-joins them.

The infocard layer is the most complete (16 plants once Bengaluru is
dropped). The accordion + markers ship only 12 - the divestment-in-flight
plants (Aryana, Kanchenkanya, Sanand, Raninagar) are present in
infocards but commented out of the desktop layers.

## Extraction strategy

Tier 1, single fetch (~64KB HTML, no JS / protection).

1. `stripHtmlComments()` removes HCCB's hide-but-don't-delete entries.
2. `parseInfocards()` recovers the 16 active mobile infocards as
   `{ dmIndex, line1, line2 }`.
3. `pickNameAndState()` swaps the two lines when the FIRST `<p>` is a
   recognised Indian state (Bidadi quirk: source emits "Karnataka" /
   "Bidadi Factory" in that order).
4. `applyOverrides()` repairs the four well-known mis-labels (Ameenpur
   marker says Maharashtra; Kanchenkanya / Raninagar carry city /
   factory-tag in the state slot).
5. `parseAccordionNames()` + `parseMarkerNames()` populate the
   `accordion-active` / `map-marker-active` feature flags by NAME (the
   index-schemes don't match between layers).

Each record emits:
- `locationType: "warehouse"` (canonical schema lacks "bottling-plant")
- `storeType: "bottling-plant"`
- `storeId: dm-<N>` (`dm-1` ... `dm-17`, with `dm-17` being Bengaluru and dropped)
- `features: ["accordion-active:yes|no", "map-marker-active:yes|no"]`
- `_extra: { plantName, plantState, accordionActive, mapMarkerActive }`

## Field inventory

| Field | Source | Notes |
|---|---|---|
| brand | hard-coded "Hindustan Coca-Cola Beverages" | |
| storeId | `dm-` + infocard `dm-index` | |
| name | "HCCB - " + plant name | |
| address.line1 | "<name>, <state>" | source ships no street address |
| address.city | plant name | source mixes state/city - we take the plant name |
| address.state | resolved via `INDIAN_STATES` + `PLANT_STATE_OVERRIDES` | |
| address.postalCode | n/a | source ships no pincode |
| address.country | "IN" | |
| features | accordion-active + map-marker-active flags | divestment visibility |
| lat / lng / phone / email / hours / url | n/a | source ships none |

## Plant footprint

16 plants across 9 states. Desktop-visible (accordion + map) = 12;
infocard-only = 4 (divestment-in-flight).

| State | Plants |
|---|---|
| Telangana | Ameenpur, Bandathimmapur, Aryana* |
| Andhra Pradesh | Srikalahasti, Atmakuru |
| Karnataka | Bidadi, Aranya |
| Maharashtra | Wada, Pirangut |
| Gujarat | Goblej, Sanand* |
| West Bengal | Kanchenkanya*, Raninagar* |
| Tamil Nadu | Nemam |
| Goa | Verna |
| Madhya Pradesh | Pilukedi |

\* infocard-only (not on desktop accordion / map - divestment-in-flight)

## Files in this repo

- **Extractor:** `scrapers/sales-distribution/hccb.ts`
- **Production runner:** `scrapers/sales-distribution/hccb-run.ts`
- **Fixture:** `__tests__/fixtures/hccb-contact-us.html`
- **Tests:** `__tests__/hccb-stores.test.ts` (9 tests, 98 assertions)
- **Output:** `data/results/hccb-stores.jsonl` (16 records)

## Scorecard

`overall=71 band=medium (completeness=87 dataQuality=35 auth=91 freshness=75)`

The medium band is data-quality capped - the source ships no
coordinates, no phone, no hours, no street address. Completeness 87
because we recovered 16/16 matching the FY24 authoritative count
exactly. Auth 91 because the domain matches the brand, brand is on
Wikipedia, and the page is brand-controlled.

## Pagination

None - one fetch covers the country.

## Gotchas

- **Soft-404 returns HTTP 200.** Every guessed path
  (`/our-plants`, `/plants`, `/factories`, `/manufacturing`, `/about`,
  `/our-locations`) returns a 95896-byte `<title>404 | HCCB</title>` page
  with HTTP 200. Use sitemap.xml or the homepage's main nav for ground
  truth, not status codes.
- **Locator lives on `/contact-us`.** None of the obvious
  "plant" / "factory" / "manufacturing" paths exists; the data is
  embedded in the contact-us page's "Where Our Beverages Come to Life"
  section. This is the same false-trail-on-nav pattern as VBL (where the
  visible menu item "Plant Tour" routes to `/plan-tour/`, an empty stub).
- **NOT WordPress.** `/wp-json/wp/v2/pages` returns the same 95896-byte
  404 page (HTTP 200). The VBL wp-json enumeration trick does NOT apply
  here. Site is a custom static / CMS - locale-prefixed (`/en`, `/hi`,
  `/tel`).
- **THREE redundant layers with DIFFERENT numbering schemes.** Mobile
  infocards use `dm-index`; desktop accordion + markers use
  `data-marker-index`. Bidadi is `dm-index=7` in the infocard but
  `data-marker-index=17` in the accordion. You cannot index-join the
  layers - name-join them.
- **Source mis-labels some states.** Ameenpur marker says "Maharashtra"
  (real: Telangana); Bidadi infocard reverses lines (state first); a
  few cards put the city or factory tag in the state slot. The
  `PLANT_STATE_OVERRIDES` table fixes the known cases.
- **Four plants are infocard-only.** Aryana, Kanchenkanya, Sanand,
  Raninagar are present in mobile infocards but commented out of the
  desktop accordion + map markers - likely the divestment-in-flight
  posture (HCCB is being sold to the Jubilant Bhartia consortium 2024).
  The extractor reports them with `accordion-active:no` +
  `map-marker-active:no` so a downstream consumer can filter them out
  if they only want "publicly-flaunted" plants.
- **One plant fully commented out.** Bengaluru (`dm-index=17` /
  `data-marker-index=17` - the SAME index in both schemes) - the
  in-city Bengaluru factory was decommissioned and replaced by Bidadi
  in the same metro. The extractor's `DROP_PLANT_NAMES` set guarantees
  we still drop it if a future edit un-comments it.

## What I tried that didn't work

In order: probed `/our-plants`, `/plants`, `/factories`,
`/our-locations`, `/manufacturing`, `/about` (all soft-404). Tried
`/wp-json/wp/v2/pages` (returns the 404 page - NOT WordPress).
Read the sitemap.xml (no plant-list URL in it). Read
`/business`, `/investor-relations`, `/about-us`, `/community` (prose
only). Finally found 16 `<h4>Factory</h4>` accordion / `dm-index`
infocard pairs on `/contact-us` - one fetch covers everything.
