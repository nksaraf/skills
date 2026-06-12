# HDFC Bank — branch + ATM locator

**Domain:** hdfcbank.com (locator at `near-me.hdfcbank.com` → 302 → `www.hdfc.bank.in/branch-locator`)
**Vertical:** Indian banking (largest private-sector bank; ~8,700 branches + ~18,000 ATMs post HDFC Ltd merger)
**Last verified:** 2026-05-21
**Tier:** 2 (direct AEM JSON API — no browser needed)
**Framework:** Adobe Experience Manager (AEM) — selector-suffix `.json` endpoints on the `www.hdfc.bank.in` publisher
**Protection:** mild — CloudFront fronts everything, but the JSON endpoints accept a plain Chrome UA + en-IN headers. No 403/Cloudflare challenges observed on a ~12-call sweep.

## URL patterns

| Page type | URL | Method |
|---|---|---|
| Locator landing | `https://www.hdfc.bank.in/branch-locator` | GET (HTML) |
| **State → city directory** | `/content/hdfcbankpws/api/city.json/content-fragments/branch-locator` | GET (JSON) |
| **Per state + city listing** | `/content/hdfcbankpws/api/branchlocator.{State}.{City}.json` | GET (JSON) |
| GraphQL persisted (alternative) | `/graphql/execute.json/hdfcbankpws/getLocatorDetailsByState` | GET |
| GraphQL persisted (by checkbox/type) | `/graphql/execute.json/hdfcbankpws/getLocatorDetailByType` | GET |

**The endpoint we use:** `/content/hdfcbankpws/api/branchlocator.{State}.{City}.json`. It returns BOTH branches AND ATMs in one payload, distinguished by the `type` field (`HDFC_Bank`, `HDFC_Bank_Home_Loan`, `HDFC_Bank_ATM`). The selector-suffix is AEM's standard pattern — state and city are passed as URL selectors, no query-string.

The `near-me.hdfcbank.com` subdomain is a stub redirector that 302s to the main www site. There is no Singleinterface / Synup SaaS locator here despite the subdomain naming.

## Endpoint discovery

The HTML at `/branch-locator` is a hand-rolled AEM template — no `window.__INITIAL_STATE__`, no `__NEXT_DATA__`. The branch-locator JS bundle (`/etc.clientlibs/hdfcbankpws/clientlibs/clientlib-branchLocator.min.js`) is pure UI. Data-fetching lives in a separate bundle: `/etc.clientlibs/hdfcbankpws/clientlibs/clientlib-api.min.js`. Inside it is an `apiEndpoints` map enumerating every locator endpoint — including the per-state-city selector pattern (`apiEndpoints[BRANCH_LOCATOR_BY_STATECITY] = "/content/hdfcbankpws/api/branchlocator"` plus a URL builder that appends `"." + state + "." + city + ".json"`).

## Payload shape (per row)

```jsonc
{
  "branchCode": "7012",                          // primary key — unique per row
  "branchName": "Navroze,_Pali_Hill,_Bandra_(W)", // underscore-encoded
  "type": "HDFC_Bank",                            // → locationType "branch"
  "typeOfBranch": "",
  "districtAsPerCensusData2011": "",
  "city": "Mumbai",
  "revisedCity": "Mumbai",
  "geographicalState": "Maharashtra",
  "ifsc": "HDFC0007012",     // populated only for full-service branches
  "micr": "",                // rarely populated in the public feed
  "branchAddress": "SN 2 Pali Hill Navroze Premises CHSL Pali Hill Road Mumbai Maharashtra",
  "pincode": "400050",
  "pincodeArea": "400050",
  "branchTiming": "09:30 AM - 03:30 PM",   // ATMs are "12:00 AM - 11:59 PM"
  "latitude": "19.0680382",
  "longitude": "72.825611",
  "contactNumber": "+919409767967",
  "branchManager": "Rachna Khera",
  "branchBankingHeadBbh": "Tarun Chaudhry",
  "zonalHeadEbfsName": "Pankaj Seth",
  "clusterstateHead": "Sandeep Kochar",
  "bbhRegion": "West",
  "lockers25": "",                                            // "Yes"/"" — locker availability flag
  "aadhaarSevaKendra16thJun25": "",                           // "Yes"/"" — flag
  "handicapAccessRampsProvision31stMay25": "",                // "Yes"/"" — flag
  "pagePath": "/content/hdfcbankpws/in/branch-locator/maharashtra/mumbai/hdfcbank-branch-navrozepalihillbandraw.html"
}
```

## Branch vs ATM split

A **single endpoint** carries both — there are NOT two separate finders. The `type` field is the discriminator:

| `type` value | Maps to `locationType` | IFSC? | branchCode shape |
|---|---|---|---|
| `HDFC_Bank` | `branch` | YES (`HDFC0NNNNNN`) | numeric (`7012`) |
| `HDFC_Bank_Home_Loan` | `branch` | usually no | `HLNNNN` |
| `HDFC_Bank_ATM` | `atm` | no | `TNNN`, `IMNNN`, `ATMNNNN` (varied) |

Mumbai sample: **247 ATMs + 203 branches + 13 home-loan offices = 463 rows.** That ratio is consistent with HDFC's national mix (~2× ATMs vs branches).

## Coverage

- The state→city directory enumerates **39 states/UTs and 1,135 cities**. The cities are NOT a flat list — they include district-level aliases, sub-areas, and a few phantoms (e.g. `Goa.North_Goa.json` returned 53 empty objects on one capture day). Always validate non-emptyness and dedupe.
- A polite full-national sweep (one call per city) would be ~1,135 HTTP calls. CloudFront serves these from cache so it should not be rate-limited, but pace anyway. **For per-state recon: one state with ~50–100 cities is sufficient to validate.**
- This recon scraped **Goa only** (12 cities) and observed **167 unique locations after dedup** (80 branches + 87 ATMs). The national authoritative number is ~8,700 branches + ~18,000 ATMs.

## Field inventory (canonical projection)

| Field | Source (JSON path) | Notes |
|---|---|---|
| `storeId` | `ifsc` ?? `branchCode` (branch); `branchCode` (ATM) | IFSC is the canonical Indian-banking key |
| `name` | `branchName` (underscores → spaces) | source uses underscore-as-space convention |
| `status` | (hard-coded `"open"`) | source has no closed-store flag |
| `address.line1` | `branchAddress` | unstructured single-line address |
| `address.city` | `city` ?? `revisedCity` | |
| `address.state` | `geographicalState` (underscores → spaces) | |
| `address.postalCode` | `pincode` ?? `pincodeArea` | |
| `address.country` | (hard-coded `"IN"`) | |
| `lat` / `lng` | `latitude` / `longitude` | both arrive as numeric strings — coerced |
| `phone` | `contactNumber` | usually a call-centre 1860 number for ATMs |
| `url` | derived from `pagePath` (strip `/content/hdfcbankpws/in` prefix) | per-branch detail HTML |
| `hours.all` | `branchTiming` (normalised) | ATMs always 24/7 |
| `locationType` | derived from `type` (`*ATM*` → `atm`, else `branch`) | |
| `storeType` | `type` (underscores → spaces) | "HDFC Bank" / "HDFC Bank ATM" / "HDFC Bank Home Loan" |
| `features` | derived from `lockers25` / `aadhaarSevaKendra16thJun25` / `handicapAccessRampsProvision31stMay25` | "Yes" → feature label |
| `_extra.ifsc` | `ifsc` | **always preserved verbatim** — banking primary key |
| `_extra.micr` | `micr` | preserved when present (often blank in public feed) |
| `_extra.branchCode` | `branchCode` | **NOT promoted out** — kept in _extra as dedupe key |

## Files in this repo

- **Extractor library:** `scrapers/banking/hdfc-bank.ts` (per-row mapper, type classifier, URL builder, dedupe)
- **Production runner:** `scrapers/banking/hdfc-bank-run.ts` (Goa-state sweep, scorecard, snapshot, alerts)
- **Fixture:** `__tests__/fixtures/hdfc-bank-maharashtra-mumbai.json` (463 rows from Maharashtra/Mumbai)
- **Tests:** `__tests__/hdfc-bank-stores.test.ts` (14 schema-locked tests)

## Pagination

None — each city URL is a complete list for that city. There's no `?page=N`, no cursor. The challenge is **dedupe across the city hierarchy**, not pagination: the same branch appears under `City=Mumbai`, `City=Maharashtra`, and any sub-locality that mentions it.

## Gotchas

- **Underscore-as-space encoding.** The source emits `branchName`, `geographicalState`, `type` with underscores standing in for spaces (`"Navroze,_Pali_Hill,_Bandra_(W)"`). Decode before display.
- **IFSC is NOT unique across rows.** Mumbai exhibits one shared IFSC (`HDFC0000240`) across two distinct branchCodes (240 and 560) — a main branch + co-located annex. Dedupe by `branchCode` (the true primary key), not by IFSC or storeId.
- **branchCode kept in `_extra` on purpose.** It's the source's primary key and the dedup key — we intentionally don't promote it out so dedupe can find it.
- **City list contains district aliases and phantoms.** `Goa.North_Goa.json` returns 53 empty `{}` objects (probably an AEM content-fragment placeholder for a district that isn't a real city). Always filter empty rows and dedupe by `branchCode`.
- **City-level overlap is heavy.** Goa: 185 raw rows across 12 cities → 38–167 unique branchCodes depending on cache state. Always `extractHdfcLocations(...) → dedupeHdfcLocations(...)`.
- **CloudFront edge-cache variance.** Two identical fetches a minute apart can return different row counts for the same city URL (especially the district-level aliases like `Goa.Goa.json`). This is CDN cache age, not anti-bot — the response is just a stale variant. Schedule scrapes weekly and rely on the snapshot diff to surface real drift.
- **The `near-me.hdfcbank.com` subdomain is a redirector**, NOT a separate SaaS locator. It 302s to `www.hdfc.bank.in/branch-locator`. The yaml's `official_locator_url` is honoured by browsers but bots should follow the redirect to the real source.
- **`branchTiming` is the same for every row in a category.** Branches all say `"09:30 AM - 03:30 PM"`; ATMs all say `"12:00 AM - 11:59 PM"`. There are no per-store hour overrides in the public feed.
- **Three "yes/no" feature flags carry dates in their field names** (`lockers25`, `aadhaarSevaKendra16thJun25`, `handicapAccessRampsProvision31stMay25`). The dates encode when the data point was last refreshed. Treat as opaque-name booleans; do NOT parse the date out of the field name.
- **Two-way state/city normalisation needed.** State names in the directory use underscores (`Tamil_Nadu`) but lat/lng dedupe needs the spaced form. The extractor handles this; consumers do not need to.

## Endpoint strategy

Sweep states one at a time → for each state walk every city in the directory → call `branchlocator.{State}.{City}.json` → dedup by `_extra.branchCode`. For Goa that's 13 HTTP calls (12 cities + the directory). For a national sweep that's ~1,136 calls — well within polite limits at 600ms pacing (~12 minutes wall clock).

## Why Tier 2 (not Tier 1 raw HTML, not Tier 3 browser)

- Tier 1 fails because the HTML at `/branch-locator` carries no embedded payload — the locator hydrates after JS loads.
- Tier 3 (browser) is unnecessary — the AEM JSON endpoints accept a plain Chrome UA with no warmup, no cookies, no CSRF token. CloudFront serves them cleanly.
- Tier 2 is strictly better: one HTTP call per city, ~10ms response, no JS engine.
