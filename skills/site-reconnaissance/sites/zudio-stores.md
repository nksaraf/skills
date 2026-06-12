# Zudio (Trent / Tata) — store locator

**Domain:** zudio.com (locator app: customapp.trent-tata.com)
**Vertical:** India retail (Tata Group / Trent — value-fashion, Zara-volume competitor)
**Last verified:** 2026-05-20
**Tier:** 2 (direct JSON API — no browser)
**Framework:** same Trent-tata.com locator API as Westside (`type=zudio` parameter)
**Protection:** none observed

## Discovery story

Zudio was discovered via **sibling-brand probing** on the Westside API. Westside extraction was complete (300 stores). The score card flagged: "no sibling-brand probe performed". One curl with `?type=zudio` (instead of `?type=westside`) returned **`store_count: 578`**. This is the canonical example of why every multi-brand parent corporation needs a sibling probe — see `../patterns/sibling-brand-probing.md`.

## URL patterns

| Page type | URL | Method |
|---|---|---|
| Brand state list | `https://customapp.trent-tata.com/custom/stores?type=zudio` | GET |
| Stores by state | `https://customapp.trent-tata.com/api/custom/getstatestore-name?type=zudio` | POST (`store_state=<State>`) |
| Stores by city | `https://customapp.trent-tata.com/api/custom/getcitystore-data?type=zudio` | POST (`store_city=<City>`) |
| Locator UI | `https://www.zudio.com/pages/store-locator` | GET (HTML) |

Same wire shape as Westside; only the `type=` and headers differ.

## Required headers

```
Accept: application/json
Origin: https://www.zudio.com
Referer: https://www.zudio.com/pages/store-locator
```

## Schema

Same as Westside (same API, same `WestsideStoreRaw` interface). 42 fields per store. The mapper at `scrapers/retail/westside.ts` is **brand-parameterised** — pass `"Zudio"` as the brand argument.

## Files in this repo

- **Extractor:** reuses `scrapers/retail/westside.ts` with `brand: "Zudio"`
- **Production scraper:** `scrapers/zudio-stores.ts`
- **Fixture:** no separate fixture (shape identical to Westside — same tests cover both)
- **Tests:** the Westside test file at `__tests__/westside-stores.test.ts` includes a Zudio-brand test case

## Coverage

- **555 extracted vs 578 claimed** — 96% (4% drift, well within tolerance)
- **27 states**
- **Lat/lng:** initially 0% (API returns blank `store_latitude`/`store_longitude` for Zudio rows) → **99% after `google_emmbeded_code` recovery** (see `../patterns/google-maps-embed-recovery.md`)
- **All other fields:** phone 30% (much lower than Westside's 95%), hours 100%, address 100%, storeId 100%, postalCode 100%

## Pagination

Same as Westside — per-state response is the full list, no cursors.

## Gotchas

- **Source emits blank lat/lng** for all Zudio rows (not Westside's behaviour). Recovery from `google_emmbeded_code` (`!2d{lng}!3d{lat}` pattern) restores 99%.
- **Phone coverage is much lower than Westside** (30% vs 95%). Zudio is a value brand — fewer dedicated phone lines per store.
- **Source-side `updated_at` median: Feb 2024** (~10 months old at time of scrape). Younger than Westside's median Sep 2022, but still flag-worthy for freshness.
- **4% drift between claimed (578) and extracted (555)** — likely a handful of states returned slightly fewer stores than the brand-level claim. Not a hard cap.

## Cross-references

- [`westside-stores.md`](./westside-stores.md) — sibling at the same API host
- [`../patterns/sibling-brand-probing.md`](../patterns/sibling-brand-probing.md) — how Zudio was found
- [`../patterns/google-maps-embed-recovery.md`](../patterns/google-maps-embed-recovery.md) — how Zudio's coordinates were recovered
- [`../patterns/validation-scorecard.md`](../patterns/validation-scorecard.md) — the scorecard that flagged the missing sibling

## What other Trent brands to probe next

The `?type=` parameter suggests more brands at the same API:

- **Westside Light** — smaller-format Westside (probe `?type=westside-light`, `?type=westsidelight`)
- **Star Bazaar / Trent Hypermarket** — different vertical
- **Misbu** — Trent's beauty brand
- **Utsa** — Trent's ethnic-wear brand

A 30-second probe per name. The pattern is documented at `../patterns/sibling-brand-probing.md`.
