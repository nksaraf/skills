# JK Cement

**Domain:** jkcement.com
**Vertical:** India sales-distribution — building materials (grey cement, white cement, wall putty)
**Last verified:** 2026-05-22
**Tier:** 2 (single JSON XHR POST)
**Framework:** WordPress (`wp-content/themes/jkcement`) + `admin-ajax.php` AJAX gateway
**Protection:** none (Imperva-Incapsula `__uzm*` edge cookies issued on apex but not enforced on `admin-ajax.php`)

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| YAML's claimed locator (404) | `/dealer-locator/` | `https://www.jkcement.com/dealer-locator/` |
| Real locator landing (200) | `/store-locator` | `https://www.jkcement.com/store-locator` |
| SEO-canonical (200, same widget) | `/grey-cement/find-a-dealer/` | `https://www.jkcement.com/grey-cement/find-a-dealer/` |
| State page | `/grey-cement/find-a-dealer/{state-slug}/` | `.../delhi/`, `.../uttar-pradesh/` |
| City page | `/grey-cement/find-a-dealer/{state-slug}/{city-slug}/` | `.../delhi/new-delhi/` |
| Backing AJAX | `/wp-admin/admin-ajax.php` (POST) | see below |
| Locator JS | `/wp-content/themes/jkcement/js/store-locator.js` (or rocket-cached `/wp-content/cache/min/1/...`) | source-of-truth for the action names |

**Note:** The dealer-card HTML is NEVER server-rendered. State / city / pincode pages all render the same empty `<div id="addressesHolder">` shell; the locator JS injects results after the AJAX POST returns.

## The four `admin-ajax.php` actions

| Action | Body | Returns |
|---|---|---|
| `get_store_cities` | `state=<state>` | list of cities for a state |
| `get_pincodes` | `city=<city>` | list of pincodes for a city |
| `get_stores` | `state=<...>&city=<...>&pincode=<...>` | up to 100 dealers nearest the pincode |
| **`get_all_stores`** | (none) | **all 8,909 dealers in one ~4 MB JSON envelope — what we use** |

No CSRF / nonce required — `wp_ajax_nopriv_get_all_stores` is exposed to logged-out users. Default Chrome UA + `X-Requested-With: XMLHttpRequest` passes.

## Hydration payload

- **Location:** JSON response body from `POST /wp-admin/admin-ajax.php` with `action=get_all_stores`
- **Envelope:** `{ type: "success", data: [...row] }`
- **Row keys (16):** `s_no`, `sap_customer_code`, `customer_name`, `customer_phone`, `address_line_1` through `address_line_4` (line_4 is a messy auto-concatenation of 1-3 with trailing-comma artefacts), `pin_code`, `taluka`, `district_name`, `city_name`, `state_name`, `latitude`, `longitude`, `tier`
- **`latitude` / `longitude` sentinel values:** `"-"` (most common) or `"0"` / `"0.00"` (~28% of rows have no real coords → null in canonical output, never zero)
- **`tier` values observed:** `No Tier`, `Bronze`, `Silver`, `Gold`, `Platinum`, `Titanium`, `Temp-Bronze`, `Temp-No Tier`, `""` (blank). Promoted to `storeType`.

## Extraction strategy

Tier 2 — one POST. Zero pagination, zero session bootstrap, zero CSRF. Run-time ~1 second to pull the 4 MB national dump.

## Field inventory

| Field | Source | Notes |
|---|---|---|
| brand | constant | `"JK Cement"` |
| storeId | `sap_customer_code` | always populated; 6-digit SAP code |
| name | `customer_name` | SHOUTY UPPERCASE in source |
| address.line1 | rebuilt from `address_line_1..3` | line_4 is unusable (double-commas) |
| address.city | `city_name` | title-cased in extractor |
| address.locality | `taluka` | sub-district |
| address.state | `state_name` | `Uttaranchal` collapsed into `Uttarakhand` |
| address.postalCode | `pin_code` | always 6-digit |
| lat / lng | `latitude` / `longitude` | sentinel-aware (`"-"` → null) |
| phone | `customer_phone` | bare 10-digit → `+91XXXXXXXXXX` |
| storeType | `tier` | dealer-tier classification (Bronze/Silver/Gold/Platinum/Titanium) |
| locationType | constant | `"dealer"` (cross-industry channel-partner sense) |
| _extra | `s_no`, `sap_customer_code`, `district_name`, `taluka`, `tier` | preserved verbatim |

## Vendor comparison — JK Cement vs the four reconned cement stacks

| Brand | Backend | Transport | Auth | Coverage | Has lat/lng | Has phone | Has tier |
|---|---|---|---|---|---|---|---|
| **JK Cement** | **WordPress + `admin-ajax.php`** | **Single XHR POST, `get_all_stores` action** | **None** | **8,909 dealers / 19 states** | **72% (sentinel "-" for ~28%)** | **100%** | **Yes (8 tiers)** |
| UltraTech (Aditya Birla) | AEM + `/bin/exceltojsonservlet` | Single XHR POST, returns Excel-converted dump | None | 3,956 / 25 states | No | ~70% | No |
| Ambuja (Adani) | Sitecore + Next.js + custom REST | State-sweep `/dealersapi/dealers/{state}` | None | 10,353 / 25 states | No (sister ACC has it) | ~95% | No |
| Shree | Synup `EThemeForMasterGrid` white-label | State×page HTML sweep, hidden inputs per card | None | 242 / 12 states | Yes (inline) | ~90% | No |
| Dalmia (Dalmia Bharat) | CodeIgniter `/welcome/getDealerByPincode` | POST per pincode, ≤5 nearest | Radware Bot-Manager session bootstrap mandatory | ~hundreds via pincode grid | Yes | Yes | No |

**JK Cement is a fifth distinct stack.** Closest cousin is UltraTech (also a single-call national dump) but JK ships lat/lng (UltraTech doesn't), uses snake_case row-of-objects shape (UltraTech: Excel-named envelope key), and stays on WordPress rather than AEM. JK and Ambuja are the only two reconned cements with >8k records; UltraTech and Shree are 1-2 orders of magnitude smaller. JK is the only one with a published tier classification (Bronze through Titanium).

## Coverage shape — north-India dominance

| State | Count | Notes |
|---|---|---|
| Uttar Pradesh | 1,444 | largest bucket |
| Rajasthan | 1,358 | core market |
| Karnataka | 1,152 | only south-Indian state with material presence |
| Madhya Pradesh | 1,086 | |
| Gujarat | 949 | |
| Bihar | 726 | |
| Haryana | 697 | |
| Maharashtra | 604 | west-central |
| Punjab | 338 | |
| Delhi | 168 | |
| Uttarakhand | 160 | includes 2 source rows tagged `Uttaranchal` (pre-2007 name) merged into canonical bucket |
| Goa | 103 | |
| Kerala | 94 | only other southern foothold |
| Himachal Pradesh | 12 | |
| Chandigarh | 6 | |
| Dadra and Nagar Haveli | 4 | |
| Daman and Diu | 3 | |
| Assam | 1 | sole NE footprint — likely a single warehouse / distributor anomaly |
| **(absent)** | — | West Bengal, Odisha, Jharkhand, Telangana, AP, TN, all of NE: zero dealers in public locator |

Top 7 (UP / Rajasthan / Karnataka / MP / Gujarat / Bihar / Haryana) ≈ 85% of the network. Matches YAML claim "north India dominant".

## Files in this repo

- **Extractor:** `scrapers/sales-distribution/jk-cement.ts`
- **Production runner (Phase D):** `scrapers/sales-distribution/jk-cement-run.ts`
- **Fixture:** `__tests__/fixtures/jk-cement-locator.json` (156-row sample, 19 states)
- **Tests:** `__tests__/jk-cement-stores.test.ts` (16 tests, 2,122 assertions)
- **JSONL output:** `data/results/jk-cement-stores.jsonl` (8,905 records)

## Pagination

Not applicable — `get_all_stores` is a single national dump. No `?page=N`, no cursor, no offset.

## Gotchas

- The YAML's `official_locator_url` is **`/dealer-locator/` → 404**. Real locator path is `/store-locator` (alias) → SEO canonical `/grey-cement/find-a-dealer/`.
- State + city + pincode pages all render the same empty `<div id="addressesHolder">` shell. Dealer cards are JS-injected; scraping the HTML directly yields zero records. Hit `admin-ajax.php` instead.
- `address_line_4` is a server-side auto-concatenation of lines 1-3 with double-comma + trailing-comma artefacts (`"Shop No.11, Vandna Cinema Road,, Khatri Kuva, Vijapur-382 870.,"`) — do not use; rebuild from lines 1-3.
- `latitude` / `longitude` can be the **string `"-"`** (most common sentinel) OR `"0"` / `"0.00"`. Naive `parseFloat("-")` returns `NaN`; naive `parseFloat("0")` returns 0. Both must coerce to `null`. ~28% of rows have no real coords.
- `state_name` has **two buckets for Uttarakhand** — `Uttaranchal` (pre-2007 name, 158 rows) and `Uttarakhand` (2 rows). Same state. Collapsed in `normaliseState`.
- `tier` field has empty string `""` for some rows (not `null`); `_extra.tier` preserves verbatim but `storeType` becomes `null` for those.
- The `myAjax.key` exposed in `wp-localize-script` (`YXtu PHtJ TYxy 1ZOI iSZ8 o2Cd`) is NOT required for `get_all_stores` — it's used by the Gravity Forms ("contact dealer") submission flow, not the locator. Don't send it; it's irrelevant.
