# UltraTech Cement

**Domain:** ultratechcement.com
**Vertical:** Sales & distribution (cement channel partners — India's largest cement producer)
**Last verified:** 2026-05-21
**Tier:** 2 (single XHR POST to an AEM servlet)
**Framework:** Adobe Experience Manager (AEM) with `/bin/*Servlet` JSON endpoints
**Protection:** none observed on the locator endpoint (CloudFront-fronted, plain Chrome UA passes; no Akamai/Imperva challenge)
**Parent:** Aditya Birla Group

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| Locator landing (redirect target) | `/for-homebuilders/homebuilding-explained/home-planning-tools/stores-locator` | https://www.ultratechcement.com/for-homebuilders/homebuilding-explained/home-planning-tools/stores-locator |
| Short alias (301 → above) | `/store-locator` | https://www.ultratechcement.com/store-locator |
| State list XHR (GET) | `/content/dam/ultratechcement/store-locator-excel/states-list.infinity.json` | (used for dropdown labels only — main data is the next one) |
| **Channel-partner data XHR (POST)** | `/bin/exceltojsonservlet?filePath=/content/dam/ultratechcement/store-locator-excel/storelocator.xlsx` | returns the entire ~3,956-row dataset in one call |

The YAML's `official_locator_url` (`/contact-us/find-a-store`) is **stale** — that path returns 404. The current locator lives where listed above.

## Hydration payload

- **Location:** XHR — `POST /bin/exceltojsonservlet?filePath=...storelocator.xlsx`
- **Body:** any JSON (the servlet ignores it; we send `{}`)
- **Response:** `Content-Type` is blank but the body is JSON:
  ```json
  {
    "Store Locator Data UBS 24th Feb": [
      {
        "Allied Code": "731145S001",
        "Name": "Shri Chandarana Sales Corporation",
        "Address": "CHANDARANA CHAMBERS.,15 PANCHNATH PLOT,RAKJOT",
        "City": "RAJKOT",
        "State": "Gujarat",
        "Pincode": "360001",
        "Phone1": "9824210501",
        "Phone2": "9824210501",
        "Id": "",
        "Index": ""
      },
      ...
    ]
  }
  ```
- **Total rows on 2026-05-21:** 3,956 across 25 states
- **`Id` and `Index` columns are always blank** in the current dump — preserved in `_extra` if they ever populate

## Extraction strategy (scope-down)

**Tier 2 — single XHR POST.** No state-by-state sweep is needed. The locator dropdowns are pure client-side filters over the in-memory full list. One request returns everything UltraTech publishes.

**Scope of "everything"** matters here. The YAML's `authoritative_count: 100000` references corporate disclosures about UltraTech's wholesale channel-partner network. The PUBLIC locator surfaces only ~3,956 — about 4% of that. The remaining ~96k is the wholesale stockist tier that isn't published online. We capture the entire public dataset; the gap is documented in the scorecard (which uses the published-locator size as the authoritative count for self-consistency, not the 100k figure).

**Why not sweep harder?** Because there's no second endpoint to sweep. There's no per-state, per-district, or pincode API — the dropdowns filter a static blob. UltraTech doesn't expose deeper data publicly.

## Field inventory

| Field | Source | Notes |
|---|---|---|
| `brand` | hardcoded "UltraTech Cement" | |
| `storeId` | `Allied Code` | Source's unique partner code; ~265/3,956 rows have blank codes |
| `name` | `Name` | Mixed case (proper) |
| `address.line1` | `Address` | Comma-delimited free text, SHOUTY uppercase in ~half the rows |
| `address.city` | `City` (title-cased) | Source is SHOUTY uppercase — we title-case |
| `address.state` | `State` (normalised) | Source mixes "MP" / "UP" / "HP" / "Himachal" / "UTTARANCHAL" / "J&K" — see `normaliseState` |
| `address.postalCode` | `Pincode` | All 6-digit |
| `phone` | `Phone1` → `Phone2` | 10-digit Indian mobile, normalised to `+91XXXXXXXXXX` |
| `lat` / `lng` | **null** — source ships no coordinates | |
| `locationType` | hardcoded `"dealer"` | The source doesn't surface stockist vs retailer; Allied Code's middle letter likely encodes segment (S/P/G/R/M/A/B/...) but mapping isn't published |
| `_extra` | preserves `Allied Code`, `Id`, `Index` | Allied Code stays in `_extra` for dedupe even though it's also `storeId` |

## Files in this repo

- **Extractor:** `scrapers/sales-distribution/ultratech-cement.ts` (pure functions)
- **Production runner:** `scrapers/sales-distribution/ultratech-cement-run.ts` (live XHR + Phase D pipeline)
- **Fixture:** `__tests__/fixtures/ultratech-cement-locator.json` (70-row representative sample, 8 states + 1 blank-Allied-Code row)
- **Tests:** `__tests__/ultratech-cement-stores.test.ts` (13 tests, 683 expects)
- **Output:** `data/results/ultratech-cement-stores.jsonl` (3,956 records)

## Pagination

None. Single XHR returns everything in one body (~890 KB).

## Gotchas

- The YAML's `official_locator_url: /contact-us/find-a-store` is **a 404**. The real locator is at `/store-locator` (which 301-redirects to `/for-homebuilders/.../stores-locator`).
- State spellings are inconsistent: `MP`, `UP`, `HP`, `Himachal`, `UTTARANCHAL`, `TRIPURA`, `J&K`. `normaliseState` canonicalises these. If a downstream tool expects ISO state codes, an additional mapping pass would be needed.
- Cities are mostly SHOUTY — `normaliseCity` title-cases them while leaving rows that came in mixed-case alone.
- The dataset's top-level key is literally `"Store Locator Data UBS 24th Feb"` — clearly named after the Excel spreadsheet last updated by UltraTech ops on 24 Feb. If ops renames the sheet, the envelope key will change and the extractor needs an update. The extractor reads `payload[PAYLOAD_KEY]` with a defensive non-array guard.
- 265 of 3,956 rows have a blank Allied Code. Dedup falls back to `name|pincode|phone` for these so we don't collapse distinct unidentified partners.
- No lat/lng anywhere — the locator is text-only. Scorecard's `has-coords` will score 0 by design. Geocoding pincodes is a downstream enrichment, not a recon job.
- ~5% of rows have no phone at all.
- Don't trust the YAML's `authoritative_count: 100000` to score the public locator — it's the wholesale tier corporate disclosure, not the public locator size. The Phase D scorecard uses the public locator size as the self-consistent reference; the 100k gap is documented in the YAML/addendum and is a known scope-of-source limitation, not an extractor bug.
