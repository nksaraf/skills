# Ola Electric

**Domain:** olaelectric.com
**Vertical:** Indian E2W (electric-two-wheeler) brand — sales / distribution
**Last verified:** 2026-05-21
**Tier:** 1 (raw HTTP, no browser)
**Framework:** Magento (storefront) + Next.js / Node service (locator API)
**Protection:** Akamai fronts the storefront but the locator API host
  (`ola-ev-ui.olaelectric.com`) is unprotected for plain Chrome UA. Concurrent
  requests >24 trip soft 403s; back off + retry recovers.

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| Locator landing | `/experience-centres` | https://www.olaelectric.com/experience-centres |
| SEO city pages (no real data) | `/showroom/electric-scooters-in-<City>` | https://www.olaelectric.com/showroom/electric-scooters-in-Ambedkar-Nagar |
| SEO mandal pages (no real data) | `/find-ola-dealer/electric-scooters-in-<city>/<Mandal>` | https://www.olaelectric.com/find-ola-dealer/electric-scooters-in-anakapalle/Atchutapuram |
| **Locator XHR** | `https://ola-ev-ui.olaelectric.com/api/getExperienceCenterDetails?pincode=<6digit>` | …?pincode=560001 |

The `/stores` path advertised in many places (including the YAML) is a **404**.
The live page is `/experience-centres`.

The sitemap exposes thousands of SEO landing pages (`sitemap_dealer_pages.xml`
has 2843 entries, `sitemap_showroom_pages.xml` has 735) — these are mandal /
city wrappers, NOT stores. They hit the same pincode XHR client-side.

## Locator endpoint

- **Discovery:** the page's inline JS contains
  `const CONFIG = { BASE_URL: 'https://ola-ev-ui.olaelectric.com', ... }` and
  fetches `${CONFIG.BASE_URL}/api/getExperienceCenterDetails?pincode=<pin>` on
  pincode entry. There is **no global enumeration** — every shape that omits
  pincode (`?lat=&lng=`, `?city=`, no params) returns `400 Pincode is required`.
- **Response shape:**
  ```json
  {
    "status": "SUCCESSFUL",
    "data": {
      "expList": [
        {
          "id": 104843,
          "pincode": "560001",
          "businessUnitId": "XC-Bangalore-Rajajinagar2",
          "distance": 4.265,
          "cityCode": "BLR",
          "expName": "Ola Electric Store, Rajajinagar New",
          "expPhoneNumber": "8033113311",
          "expStatus": "ACTIVE",
          "address": "#1699, Dr Rajkumar Road, Prakash Nagar, Rajajinagar, Bengaluru, Karnataka 560021",
          "latitude": 12.991891,
          "longitude": 77.557421,
          "directionUrl": "http://olaca.bs/P9V3NjP",
          "ecAttributes": { "vehicle_inventory": ["SCOOTER","BIKE","AUTO"] }
        },
        ...
      ]
    }
  }
  ```
- Each call returns **0-6 rows** within ~30 km of the requested pincode, sorted
  ASC by `distance` (km).
- **Service centres are not exposed** — `getServiceCenterDetails` 404s, no
  parallel endpoint exists.

## Extraction strategy

Two-phase pincode sweep:

1. **Coarse:** stride-100 across the entire valid Indian PIN range
   (110000-855000) — 7,451 calls.
2. **Adaptive zoom:** for each outlet found in phase 1, sweep ±50 around the
   outlet's own postal code at stride-10 — restricts the dense crawl to
   geographically-productive regions.

The dedup key is **`businessUnitId`** (e.g. `XC-Vadodara-Jetalpur`), NOT the
integer `id`. The source ships multiple `id` values per physical outlet (one
per inventory SKU bundle); deduping by `id` leaves 2-4× duplicate rows.

## Field inventory

| Field | Source path | Notes |
|---|---|---|
| `storeId` | `id` (string-cast) | Per-SKU-bundle, NOT per-outlet — kept for traceability |
| `name` | `expName` | Always "Ola Electric Store, *" |
| `lat/lng` | `latitude`, `longitude` | Always present; 100% inside India bbox |
| `phone` | `expPhoneNumber` | National 0803 311 3311 line; per-outlet number not exposed |
| `address.line1` | `address` (verbatim) | Full address |
| `address.city` | last comma chunk before state | Parsed from address tail |
| `address.state` | second-to-last chunk after stripping pincode | Parsed from address tail |
| `address.postalCode` | trailing 6-digit | Outlet's own pincode |
| `segments` | `ecAttributes.vehicle_inventory` | SCOOTER / BIKE / AUTO |
| `storeType` | derived from `businessUnitId` prefix | "Experience Centre" if `XC-*` |
| `url` | `directionUrl` | `olaca.bs/*` short-link → Google Maps |
| `_extra.businessUnitId` | (preserved) | True per-outlet primary key |
| `_extra.cityCode` | `cityCode` | 3-letter metro code |

## Pagination

None. One round-trip per pincode.

## Gotchas

- `/stores` is a 404; the canonical path is `/experience-centres`.
- The `id` field is NOT unique per outlet — dedup by `businessUnitId`.
- The `distance` and `pincode` fields are request-scoped (not properties of
  the outlet); they're intentionally dropped from `_extra` so the same outlet
  produces the SAME extracted record regardless of which sweep pincode
  surfaced it.
- ~88% of rows parse a city cleanly; the ~12% that don't have unstructured
  addresses (lines like "Champaben Ravjibhai Tank Sukhanatha Nagar
  Bhagvatpara Village Gondal M+ OG Taluka Gongal District Rajkot 360001")
  that don't follow the trailing `, <City>, <State> <PIN>` pattern.
- The API soft-blocks bursty concurrency with 403s. Backoff + retry at
  concurrency ≤16, rate-ms ≥40 recovers cleanly.
- Service centres are NOT surfaced by this endpoint and Ola's
  `getServiceCenterDetails` is a 404 — the published locator is sales-only.

## Files in this repo

- **Extractor:** `scrapers/sales-distribution/ola-electric.ts`
- **Runner:** `scrapers/sales-distribution/ola-electric-run.ts`
- **Tests:** `__tests__/ola-electric-stores.test.ts`
- **Fixtures:** `__tests__/fixtures/ola-electric-pincode-{560001,110001}.json`
- **JSONL output:** `data/results/ola-electric-stores.jsonl`
