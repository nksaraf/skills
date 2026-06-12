# Ampere Electric

**Domain:** amperevehicles.com (public) / ampere.greaveselectricmobility.com (canonical app) / ampere-showroom.greaveselectricmobility.com (Singleinterface dealer-locator microsite)
**Vertical:** E2W dealer network (sales + service touchpoints), India
**Parent:** Greaves Cotton Ltd (BSE/NSE-listed)
**Last verified:** 2026-05-21
**Tier:** 1 (sitemap + per-page JSON-LD)
**Framework:** Singleinterface enterprise store-locator (same vendor as Bata India; this microsite uses customerId/enterpriseId `537585`)
**Protection:** Cloudflare WAF blocks `www.amperevehicles.com` (403 to plain HTTP and even stealth Playwright); `amperevehicles.com` (non-www) and `ampere-showroom.greaveselectricmobility.com` are unprotected.

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| Public landing | `amperevehicles.com/dealer-locator` (non-www) | redirects to JS pincode-enquiry form, no dealer list inline |
| Locator microsite | `ampere-showroom.greaveselectricmobility.com/` | Singleinterface SPA - data not in HTML directly |
| Master sitemap | `ampere-showroom.greaveselectricmobility.com/sitemap.xml` | sitemap index of 295 per-city `.xml.gz` shards |
| Per-city sitemap | `/files/enterprise/sitemap/google/537585/{state}/{city}.xml.gz` | 4 URLs per dealer (Home/Map/Contact-Us/Photos-Videos) |
| Dealer detail (canonical) | `/{slug}-{outletId}/Home` | `/ampere-electric-scooter-kaushalpura-agra-electric-motor-scooter-dealer-kaushalpura-agra-555704/Home` |
| Dealer map | `.../{outletId}/Map` | same JSON-LD body |
| Dealer contact | `.../{outletId}/Contact-Us` | same JSON-LD body |
| Dealer photos | `.../{outletId}/Photos-Videos` | same JSON-LD body |

## Hydration payload

- **Location:** `<script type="application/ld+json">` block (one per page, contains an array of nodes)
- **Schema (key paths on the LocalBusiness node):**
  - `name` -> dealer name
  - `address[0].streetAddress` -> line1
  - `address[0].addressLocality` -> neighbourhood
  - `address[0].addressRegion` -> city
  - `address[0].postalCode` -> PIN
  - `address[0].telephone[0]` -> phone
  - `address[0].email` -> dealer email
  - `geo.latitude` / `geo.longitude` -> coords (strings)
  - `openingHoursSpecification[].dayOfWeek/opens/closes` -> hours
  - `paymentAccepted` -> comma-separated string ("Cash, Credit Card, Debit Card, Online Payment")
  - `amenityFeature[].value` -> array of amenity values (Free parking on site, ...)
- **Store node** carries `hasMap`, `contactPoint` (brand customer-support hotline + email).
- **Organization node** is brand-level boilerplate (Greaves Cotton EV arm).

URL outletId is the source's stable primary key.

## Extraction strategy

Tier 1. Direct HTTP fetches with plain Chrome UA succeed against the
Singleinterface subdomain. No warmup, no stealth browser needed.

1. GET `/sitemap.xml` -> list of 295 city shard URLs.
2. GET each shard `.xml.gz` -> gunzip -> filter `/Home` URLs.
3. GET each dealer `/Home` URL -> cheerio-parse JSON-LD -> project to `RetailStore`.
4. Dedup by outletId.

Pacing: 150ms between shard fetches, 300ms between dealer fetches. Polite,
no observed throttling.

The public `www.amperevehicles.com` host is Cloudflare WAF-blocked even with
stealth Playwright (returns 403 + Cloudflare beacon for all paths, including
`/robots.txt`). Stick to the non-www `amperevehicles.com` or the
Singleinterface subdomain.

## Field inventory

| Field | Source | Notes |
|---|---|---|
| `storeId` (outletId) | URL path `-{digits}/Home` tail | Stable primary key |
| `name` | LocalBusiness.name OR Store.name | "Ampere Electric Scooter - {Locality} - {City}" |
| `address.line1` | LocalBusiness.address[0].streetAddress | |
| `address.locality` | LocalBusiness.address[0].addressLocality | Neighbourhood |
| `address.city` | LocalBusiness.address[0].addressRegion | (Singleinterface puts CITY in addressRegion, not state) |
| `address.state` | LocalBusiness.address[0].addressRegion | Same as city in this feed - state is not separated |
| `address.postalCode` | LocalBusiness.address[0].postalCode | India PIN |
| `phone` | LocalBusiness.address[0].telephone[0] | E.164 |
| `email` | LocalBusiness.address[0].email | Dealer email |
| `lat`/`lng` | LocalBusiness.geo | Strings in JSON-LD; we parseFloat |
| `hours` | LocalBusiness.openingHoursSpecification | Per-day "10:00 AM"/"07:00 PM" -> "10:00-19:00" |
| `features` | LocalBusiness.amenityFeature[].value + paymentAccepted | "Free parking on site", "Payment: Cash", ... |
| `locationType` | constant "dealer" | E2W authorized dealers (sales + service combined) |

**Caveat:** `addressRegion` carries the CITY, not the state - Singleinterface's
own field, not a bug in our extractor. The dealer's true state can be
recovered from the sitemap shard path (`/files/enterprise/sitemap/google/537585/{state}/{city}.xml.gz`)
if we ever need it. We currently surface region-as-city in the canonical
`city` slot and copy it into `state` so downstream consumers don't see
nulls.

## Pagination

None - the master sitemap enumerates the full network. No cursors.

## Gotchas

- `www.amperevehicles.com` is Cloudflare 403-blocked for ALL clients including
  stealth Playwright. Use the non-www or the Singleinterface subdomain.
- Sub-sitemap files are gzipped (`.xml.gz`). Use `node:zlib` `gunzipSync`.
- Singleinterface emits 6+ JSON-LD nodes per page - we want the
  `LocalBusiness` one for full data; fall back to `Store` if absent.
- Some dealers omit Saturday from `openingHoursSpecification` (real data,
  not a parser bug) - tests assert >=6 days, not 7.
- The brand description embedded in every page claims "400+ dealer and
  service touchpoints", while older Greaves FY24 disclosures claim 600+.
  Tolerance is set to 25% to absorb both. Authoritative count in the
  scorecard is set to 400 (the live source claim).

## Files in this repo

- **Extractor:** `scrapers/sales-distribution/ampere-electric.ts`
- **Runner:** `scrapers/sales-distribution/ampere-electric-run.ts`
- **Fixtures:**
  - `__tests__/fixtures/ampere-electric-sitemap-index.xml` (full master index)
  - `__tests__/fixtures/ampere-electric-sitemap-agra.xml` (one city shard)
  - `__tests__/fixtures/ampere-electric-dealer-detail-kaushalpura-agra.html`
- **Tests:** `__tests__/ampere-electric-stores.test.ts` (10 tests)
- **Output:** `data/results/ampere-electric-stores.jsonl`
