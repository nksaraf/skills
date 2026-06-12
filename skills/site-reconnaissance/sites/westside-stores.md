# Westside (Tata) — store locator

**Domain:** westside.com (locator app: customapp.trent-tata.com)
**Vertical:** India retail (Tata Group / Trent — fast fashion, Zara competitor)
**Last verified:** 2026-05-20
**Tier:** 2 (direct JSON API — no browser needed)
**Framework:** Shopify storefront + custom Trent-tata.com locator API
**Protection:** none observed

## URL patterns

| Page type | URL | Method |
|---|---|---|
| Store locator page (UI) | `https://www.westside.com/pages/store-locator` | GET (HTML) |
| Brand state list | `https://customapp.trent-tata.com/custom/stores?type=westside` | GET |
| Stores by state | `https://customapp.trent-tata.com/api/custom/getstatestore-name?type=westside` | POST (`store_state=<State>`) |
| Stores by city | `https://customapp.trent-tata.com/api/custom/getcitystore-data?type=westside` | POST (`store_city=<City>`) |

**Key insight:** Westside's website is Shopify-rendered HTML but the store locator JS calls a separate Tata-owned API at `customapp.trent-tata.com`. The API is unprotected and returns clean JSON.

`type=` query param suggests other Trent brands (Zudio sub-brand, Westside Light, etc.) are at the same API — worth probing.

## Required headers

```
Accept: application/json
Origin: https://www.westside.com
Referer: https://www.westside.com/pages/store-locator
```

(User-Agent + Accept-Language are nice-to-have but not required — API accepts any.)

## Schema (per store)

42 fields per record. Promoted to canonical:

| Canonical field | Source key |
|---|---|
| `storeId` | `store_code` (e.g. "W134") |
| `name` | `store_name` (e.g. "FC Road, Deccan Gymkhana") |
| `status` | `is_live` (1 → "open", 0 → "closed") |
| `address.line1` | `store_address` |
| `address.city` | `store_city` |
| `address.state` | `store_state` |
| `address.postalCode` | `store_pin_code` |
| `address.country` | `store_country` (normalised to ISO-2) |
| `lat` / `lng` | `store_latitude` / `store_longitude` (strings — parsed to number) |
| `phone` | `store_mobile_number` |
| `email` | `store_email` |
| `url` | derived from `slug` |
| `hours` | per-day `{day}_time` + `{day}_open` |
| `storeType` | `store_type` |
| `segments` | `categories` |

Preserved in `_extra`:

- `store_zone` (e.g. "South", "Center")
- `whatsapp_number`
- `google_emmbeded_code` (full Google Maps embed URL — visual asset)
- `google_link` (Google Place link)
- `youtube_link` (only 5 stores have this)
- `categories` (when not all "All")
- `warehouse_priority`
- `is_return_accept`, `return_store`, `open_order`, `is_location_created` — also surfaced in canonical `features` array
- `created_at` / `updated_at` — record-management timestamps

## Files in this repo

- **Extractor library:** `@vinxi/scraper/retail` (canonical schema) + `scrapers/retail/westside.ts` (per-record mapper)
- **Production scraper:** `scrapers/westside-stores.ts` (walks 27 states)
- **Fixture:** `__tests__/fixtures/westside-stores-maharashtra.json` (48 Maharashtra stores)
- **Tests:** `__tests__/westside-stores.test.ts` (9 schema-locked tests)

## Coverage

- **300 stores across 27 states** (matches `store_count` from list endpoint)
- **102 unique cities**; top: Bengaluru (25), Hyderabad (21), Mumbai (19), Chennai (12), Ahmedabad (11), Pune (11), Surat (8), Jaipur (8)
- **~70% of stores have lat/lng** — 30% missing (real data quality, not parser bug)
- **All 300 have city + storeId + phone + email** populated
- **Day-by-day hours** present on most stores; some have `store_time` fallback only

## Pagination

None needed — each state's response is the full list (no cursors, no limits observed). 300 stores total across 27 states fetched in 85s with 0.8-2.0s polite pacing.

## Gotchas

- **Lat/lng are strings**, not numbers. Parser coerces.
- **`google_emmbeded_code` has a typo** in the source (double `m`). Preserve as-is in `_extra`.
- **Some stores marked `is_live: 0`** — these are closed. Filter when computing coverage stats.
- **`categories` field shape varies** — sometimes string, sometimes array. Coercion lives in `extractSegments()`.
- **Per-day hours are independent of `is_open`** — a day can have `tuesday_time: "11:00 to 21:00"` AND `tuesday_open: 0` (closed but record still has nominal hours). The extractor honours the open flag.

## Other Trent brands at the same API

Same API host (`customapp.trent-tata.com`) serves the whole Trent portfolio. See [`../patterns/sibling-brand-probing.md`](../patterns/sibling-brand-probing.md) for the discovery workflow.

| Brand | Status | Notes |
|---|---|---|
| **Westside** | ✅ extracted (this addendum, 300 stores) | |
| **Zudio** | ✅ extracted ([zudio-stores.md](./zudio-stores.md), 555 stores) | discovered via sibling probe |
| Westside Light | ⏸ not probed | smaller-format Westside |
| Misbu | ⏸ not probed | Trent's beauty brand |
| Utsa | ⏸ not probed | Trent's ethnic-wear brand |
| Star Bazaar / Trent Hypermarket | ⏸ not probed | different vertical |

## Cross-brand notes

Westside is the first Indian retail brand fully extracted. The canonical schema at `@vinxi/scraper/retail` and the `_extra` preservation pattern were validated here. Other Zara-competitor extractors should reuse:

- `RetailStore` interface
- `extraOf()` / `normaliseHours()` / `normaliseCountry()` helpers
- Same per-day hours convention (`{"mon":"11:00-21:00",...}`)
- Same status enum (`open` | `closed` | `coming-soon` | `unknown`)
