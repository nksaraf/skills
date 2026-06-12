# Sangeetha Mobiles

**Domain:** sangeethamobiles.com
**Vertical:** India retail — mobile / consumer electronics specialist (Apple + Android, new & refurbished)
**Last verified:** 2026-05-21
**Tier:** 2 (single direct JSON POST)
**Framework:** Next.js 12-ish (pages router) SPA over a Django backend (`sangeetha_deal_page.urls`)
**Protection:** none — endpoint refuses GET (405) and empty lat/lng (400) but accepts the same hardcoded `client_id`+`secret_key` the SPA ships in plain text

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| Locator page (SPA) | `/store-locate` | `https://www.sangeethamobiles.com/store-locate` |
| Locator API (POST) | `/b/api/store-geography` | `https://www.sangeethamobiles.com/b/api/store-geography` |
| Per-pincode nearby (POST) | `/b/api/search-store-location` | same host, used by "search by pincode" |

The yaml's `https://www.sangeethamobiles.com/store-locator` is a **404**. The
correct page is `/store-locate` — recovered from
`/_next/static/{buildId}/_buildManifest.js`. The page is otherwise an empty
SPA shell; the actual store data is fetched via XHR.

## Hydration payload

- **Location:** XHR-only — no `__NEXT_DATA__` payload, no JSON-LD listing.
  The SPA dispatches a Redux thunk that POSTs to `/b/api/store-geography`
  with `{client_id, secret_key, latitude, longitude}`.
- **Source name (as reported by `extractAllHydrationPayloads`):** N/A
- **Schema (key paths):**
  - `total_store` — string like `"924 stores in India"`
  - `data.{StateLabel}[]` — array of store rows
  - `data.{StateLabel}[].id` — stable per-store integer
  - `data.{StateLabel}[].store_name` — display name (suffixed `(SMPL)`)
  - `data.{StateLabel}[].store_code` — 3-4 letter abbreviation
  - `data.{StateLabel}[].address` — multi-line, ends with `..., {City} - {pincode}.`
  - `data.{StateLabel}[].mobile` — 10-digit Indian phone
  - `data.{StateLabel}[].lattitude` (sic — two t's), `longitude`
  - `data.{StateLabel}[].timing` — `"10:00 am - 10:00 pm"` OR `" - "` (placeholder for new stores)
  - `data.{StateLabel}[].geography` — district/city as labelled by the source (e.g. "Bengaluru Urban")
  - `data.{StateLabel}[].distance` — computed against the request lat/lng (we drop it)

## Credentials

The Redux thunk in `pages/_app-*.js` ships these in clear text:

```
client_id  = "SangeethaMobilesSA"
secret_key = "cb1838d3197d3a625ed8d139550f6579df20cf55cc9fe21ae06e590a1fe6a04d"
```

The endpoint refuses GET, empty body, and empty lat/lng. Any India-shaped
lat/lng works — the lat/lng only affects the `distance` field that the
backend computes per row. **Total returned is unaffected by lat/lng.**

## Extraction strategy

**Tier 2.** One POST returns every store in India in ~388 KB. No
pagination, no per-state probe, no per-store detail fetch. We sweep all
state buckets and project each row through `extractSangeethaStore` →
canonical `RetailStore`.

The locator page is otherwise XHR-only — there is no SSR fallback for the
store data, and `/store-locator` (yaml's URL) 404s. Recon recovered the
correct path from the Next.js build manifest. The Django backend has
`DEBUG = True` enabled in production (the 404 page leaks `URLconf` and
file paths) — not exploited, but worth noting for the site's overall
hygiene.

## Field inventory

| Field | Page type | Source (JSON path) | Notes |
|---|---|---|---|
| storeId | API | `id` (number → string) | stable across requests |
| name | API | `store_name` with `(SMPL)` suffix stripped | preserve `(SMPL)` in `_extra` |
| storeType | API | `store_code` | 3-4 letter code (e.g. "BRI", "KGR2") |
| address.line1 | API | `address` (cleaned) | multi-line normalized to comma list |
| address.city | API | parsed from `address` "..., {City} - {pin}." tail | falls back to `geography` |
| address.state | API | object key — **canonicalized** (Tamilnadu → Tamil Nadu, etc.) | 7 dirty spellings observed |
| address.postalCode | API | last `\d{6}` in address | |
| lat, lng | API | `lattitude` + `longitude` | reject `0.00000000` sentinel + outside India box |
| phone | API | `mobile` | |
| hours | API | `timing` | `"10:00 am - 10:00 pm"` fanned 7d; `" - "` → null (new store) |
| `_extra.store_code` | API | promoted from raw | |
| `_extra.source_geography` | API | preserves "Bengaluru Urban" / "Krishnagiri" district label | |
| `_extra.source_state` | API | preserves dirty raw key for debugging | |
| `_extra.state_was_dirty` | API | true when canonical ≠ raw | |

## Pagination

None. One request → entire footprint.

## Sibling brands

None known. Sangeetha Mobiles Pvt Ltd is single-brand. The site mentions
"Sangeetha Gadgets" as an `alternateName` on the corporate JSON-LD but it
appears to be the same retail entity.

## Files in this repo

- **Extractor:** `scrapers/retail/sangeetha-mobiles.ts`
- **Run scripts:** `scripts/run-sangeetha-mobiles-extract.ts`, `scripts/run-sangeetha-mobiles-snapshot.ts`
- **Fixture:** `__tests__/fixtures/sangeetha-mobiles-store-geography.json` (388 KB; 924 records as of 2026-05-21)
- **Tests:** `__tests__/sangeetha-mobiles-stores.test.ts` (14 tests, 88 expect calls)
- **Output:** `data/results/sangeetha-mobiles-stores.jsonl`
- **Scorecard:** `data/reports/sangeetha-mobiles-scorecard.md`, `.json`
- **Snapshot:** `data/snapshots/retail/sangeetha-mobiles/{stamp}.jsonl`

## Verified counts (2026-05-21)

- **Extracted:** 924 stores
- **Source `total_store` claim:** "924 stores in India" — exact match
- **By canonical state:** Karnataka 345, Telangana 195, Tamil Nadu 185,
  Andhra Pradesh 154, Gujarat 18, Maharashtra 8, Goa 6, Delhi 5,
  Puducherry 3, Uttar Pradesh 3, Kerala 1, West Bengal 1
- **Authoritative reference:** Mint 2024 cites ~750 stores; Business
  Standard 2023 cites 700+. Brand is private (no audited count). 924 is
  ~23% above the press-cited 750 — likely reflects FY24-25 expansion plus
  inclusion of pilot/franchise stores. Tolerance set to 25%.
- **Scorecard:** **91/100 (high)** — completeness 95, data-quality 94,
  authenticity 85, freshness 75.

## Sample record

```json
{
  "brand": "Sangeetha Mobiles",
  "storeId": "64",
  "name": "Brigade Road",
  "status": "open",
  "address": {
    "line1": "#126/1, Ground Floor, Brigade Road, Opp: Brigade Towers, Bengaluru - 560001",
    "city": "Bengaluru",
    "state": "Karnataka",
    "postalCode": "560001",
    "country": "IN"
  },
  "lat": 12.967033,
  "lng": 77.6066271,
  "phone": "7019452259",
  "hours": { "mon": "10:00-22:00", "tue": "10:00-22:00", "wed": "10:00-22:00", "thu": "10:00-22:00", "fri": "10:00-22:00", "sat": "10:00-22:00", "sun": "10:00-22:00" },
  "locationType": "store",
  "storeType": "BRI",
  "sourceUrl": "https://www.sangeethamobiles.com/store-locate",
  "_extra": { "store_code": "BRI", "source_geography": "Bengaluru Urban", "source_state": "Karnataka", "distance": "2 km away" }
}
```

## Gotchas

- **The yaml's `store-locator` URL is wrong.** The correct path is
  `/store-locate`. Recover paths from the Next.js build manifest.
- **`lattitude` is misspelled in the source.** Two t's. Keep the
  misspelling in the raw type; project to `lat`.
- **State labels are dirty.** Seven distinct misspellings of Tamil Nadu
  ("Tamilnadu", "TAMILNADUÂ " with literal mojibake), Karnataka
  ("Karnatak", "Karanataka"), Maharashtra ("Maharastra"), Puducherry
  ("Pudicherry"), Uttar Pradesh ("Uttarpradesh"). Canonicalize via
  `canonicalState()` — every dirty spelling is mapped.
- **Coordinate sentinel: `0.00000000`.** One Maharashtra store
  ("Honey Well") ships `"lattitude": "0.00000000", "longitude": "0.00000000"`.
  Treat as missing.
- **`(SMPL)` suffix.** Strip from display name; preserve raw in `_extra`.
- **Hours placeholder: `" - "`.** ~43% of stores ship this — typically
  new/pilot outlets. Treat as null hours, not as the literal " - ".
- **Phone field key: `mobile`, not `phone`.**
- **`distance` field is per-request.** Computed against the request
  lat/lng. Don't promote it to canonical — keep in `_extra` if at all.
- **Django DEBUG mode.** The site leaks URLconf module names in 404
  pages. Not used by the extractor but a hygiene flag for the site.
