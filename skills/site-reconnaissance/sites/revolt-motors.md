# Revolt Motors

**Domain:** revoltmotors.com (parent: RattanIndia Enterprises)
**Vertical:** sales-distribution / electric two-wheeler (motorcycle) dealers
**Last verified:** 2026-05-21
**Tier:** 1 (raw HTTP + Next.js RSC reassembly)
**Framework:** Next.js App Router (server components, RSC stream)
**Protection:** Cloudflare (no anti-bot challenge observed for plain Chrome UA)

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| Home | `/` | `https://www.revoltmotors.com/` |
| Locator (canonical) | `/contact-us` | `https://www.revoltmotors.com/contact-us` |
| Locator (claimed in YAML) | `/dealer-locator` | **HTTP 404** — does not exist |
| Per-city showroom page | `/showrooms/{city-slug}` | `/showrooms/visakhapatnam` (174 in the sitemap; SEO landing pages only, no per-dealer data) |

## Hydration payload

- **Location:** Next.js RSC stream — many `self.__next_f.push([1, "<chunk>"])` script blocks in the `/contact-us` page HTML.
- **Source name (as reported by recon):** App-Router RSC flow embedded in HTML.
- **Schema:** the reassembled stream embeds three arrays:
  - `state[]` — 23 states/UTs `{ state_id, state_name, status }`
  - `city[]` — 174 cities `{ city_id, city_name, state_id, status }`
  - **dealer[]** — two variants merged on `hub_id`:
    - Rich variant (179 rows) — `{ hub_id, hub_name, city_id, dealer_number, dealer_address, dealer_area, dealer_pincode, dealer_code, latitude, longitude, dealer_link, status }`
    - Lite variant (163 rows) — `{ hub_id, hub_name, city_id, dealer_number, dealer_address, office_time, latitude, longitude, status }`
  - **Union = 259 unique dealers** as of 2026-05-21.

The same `/contact-us` URL with header `RSC: 1` returns the flow-text directly (no HTML wrapper) — useful for low-bandwidth re-recons.

## Extraction strategy

Tier 1. Single HTTP GET to `/contact-us` with a plain Chrome UA + `Accept-Language: en-IN,en;q=0.9`. Reassemble RSC chunks → parse arrays → project to `RetailStore`.

No browser needed. No warmup needed. Cloudflare passes on first request.

## Field inventory

| Field | Page type | Source | Notes |
|---|---|---|---|
| storeId | locator | `dealer_code` (else `hub_id`) | "SRN2404407"-style |
| name | locator | `hub_name` | All-caps "REVOLT HUB <AREA>" |
| city | locator | join `city_id` → `city.city_name` | Source uses "Bengaluru", not "Bangalore" |
| state | locator | join through `state_id` | |
| postalCode | locator | `dealer_pincode` | Rich variant only |
| line1 | locator | `dealer_address` | |
| locality | locator | `dealer_area` | Rich variant only — e.g. "NORTH DELHI" |
| lat / lng | locator | `latitude` / `longitude` (then fallback to `dealer_link`) | A handful of rows have DMS-encoded values ("255029.7N") or swapped lat/lng — the parser reconciles to the India bounding box |
| phone | locator | `dealer_number` | Comma-separated occasionally (multi-line phones preserved) |
| hours | locator | `office_time` (lite variant only) | Single "10:00-18:00" window — replicated across 7 days |
| storeType | derived | constant | "Revolt Hub" |
| locationType | derived | constant | "dealer" |

## Files in this repo

- **Extractor (pure):** `scrapers/sales-distribution/revolt-motors.ts`
- **Production runner:** `scrapers/revolt-motors-stores.ts` (Phase D: scorecard → snapshot → alerts)
- **Fixture:** `__tests__/fixtures/revolt-motors-contact-us.html`
- **Tests:** `__tests__/revolt-motors-stores.test.ts` (9 tests, 1,860 expects)
- **Results:** `data/results/revolt-motors-stores.jsonl`

## Pagination

None. Single GET returns the entire dealer set.

## Coverage observed (2026-05-21)

- Extracted: **259 dealers**
- States covered: 23 (out of 28 + 8 UTs)
- Cities covered: 170 distinct
- Scorecard: 81/100 (band: **high**) — completeness 70, data quality 92, source authenticity 91, freshness 75

## Gotchas

- **YAML's `/dealer-locator` URL is a 404.** The canonical surface is `/contact-us`. Update the YAML or the recon will fail.
- **YAML's `authoritative_count: 70` is stale** — the live RSC payload returns 259. Press cited the "70+ stores" claim in 2023-2024; Revolt's network has grown ~3.7× since then. The extractor flags this as a completeness "above by 270%" warning (acceptable with `tolerance_pct: 30`).
- **DMS-formatted lat/lng on a handful of rows.** Example: `latitude: "255029.7N"` — meaning 25° 50′ 29.7″ N. The parser falls back to the Google-Maps `dealer_link` field when the structured value fails the India bounding-box check.
- **Two dealer-array variants in the RSC stream** — one carries `dealer_code`/`dealer_pincode`, the other carries `office_time`. The parser unions them on `hub_id` (preferring the row with more keys), so neither set of fields is lost.
- **City names use Indian-language preferred spelling.** "Bengaluru" not "Bangalore", "Belagavi" not "Belgaum". Downstream consumers should match on `city_id` rather than `city_name`.
