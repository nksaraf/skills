# Indian Oil Corporation (IOCL)

**Domain:** iocl.com (corporate) — locator backend is `sdms.indianoil.in`
**Vertical:** Sales & distribution — fuel retail (petrol pumps, LPG, KSK)
**Last verified:** 2026-05-21
**Tier:** 2 (direct JSON-ish API — pipe/CSV payload)
**Framework:** Java/JSP backend (Oracle-style WebCenter on the consumer portal; standalone Tomcat on the locator)
**Protection:** mild — Sucuri WAF in front of `iocl.com` (blackholes default UA on the corporate site); the locator backend `sdms.indianoil.in/PumpLocator/` is unprotected but session-scoped (requires JSESSIONID warmup)
**Status (2026-05-21):** ⚠ **PARKED — backend in scheduled maintenance**. Extractor + parser + tests + Phase D pipeline are in place; ready to plug in live data the moment SDMS restores service.

## Background — THREE distinct location networks

Per IOCL Annual Report FY24:

| Network | Count | locationType | Endpoint (where discovered) |
|---|---|---|---|
| Retail Outlets (petrol pumps) | ~37,000 | `store` (sub-type "Retail Outlet") | `sdms.indianoil.in/PumpLocator/MapLocations` |
| Indane LPG Distributors | ~12,500 | `distributor` | `indane.co.in` (separate site; was returning HTTP 000 during recon) |
| Kisan Seva Kendras (KSK) | ~6,500 | `store` + `_extra.is_ksk: true` (sub-type "Kisan Seva Kendra") | Same `MapLocations` endpoint — distinguished by name/RO-code prefix |

Total ~56,000 nationwide. **Recon scope is "extractor works on one slice" — Goa state only (~0.6% of national).**

## URL patterns

| Page type | URL | Status |
|---|---|---|
| Corporate landing — RO locator promo | `https://iocl.com/our-petroleum-products/retail-outlet-locator` | Sucuri 307 on curl; browser-renders fine — points users to `sdms.indianoil.in/PumpLocator/` |
| PumpLocator app root | `https://sdms.indianoil.in/PumpLocator/` | 200 — Java app, sets JSESSIONID |
| State-wise UI | `…/PumpLocator/stateWiseRO.jsp` | 200 — but **inline `<script>window.location.href='exception.jsp';</script>` indicates maintenance mode** |
| District-wise UI | `…/PumpLocator/districtWiseRO.jsp` | Same maintenance fallback |
| Route source-to-destination | `…/PumpLocator/routeROs.jsp` | **200 — fully working** — exposes the `MapLocations` AJAX endpoint |
| Networked-RO (highway pumps) | `…/PumpLocator/networkedRO.jsp` | Maintenance fallback |
| AJAX — state-wise listing | `…/PumpLocator/StateWiseLocator` (POST `state=X`) | 302 → `exception.jsp` during maintenance |
| AJAX — radius-based listing | `…/PumpLocator/MapLocations` (POST `latitude=…&longitude=…&lat=…&long=…`) | **Working when route page is the referrer** |
| LPG distributor locator | `https://indane.co.in/distributor.php` (probable) | Connection failed (HTTP 000) during recon — endpoint documented, not extracted |

## Hydration / payload

The `MapLocations` POST returns a **pipe-delimited string of comma-separated rows** (no JSON). Each row has ~45 columns. Column mapping reverse-engineered from `routeROs.jsp`'s table-build code (`locations[i].split(",")[j]` assignments):

| Col | Field |
|---|---|
| 0 | Outlet name |
| 1 | Latitude |
| 2 | Longitude |
| 3 | Address (often contains 6-digit pincode at end) |
| 25 | MS price (petrol, Rs/L) |
| 26 | HSD price (diesel, Rs/L) |
| 27 | XtraPremium price |
| 28 | XtraMile price |
| 29 | Sales officer contact |
| 30 | Dealer / Partner / Operator name |
| 31 | State office |
| 32 | Divisional office |
| 33 | Sales area |
| 34 | District |
| 35 | State |
| 36 | Contact number |
| 37 | Distance from source (km) |
| 38 | RO code (primary key — e.g. `GA0021`, `KSK0007`) |
| 39 | COVID-relief flag (Y/N, legacy) |
| 40 | COVID-relief contact (when 39 = Y) |
| 42 | XP100 price |
| 43 | XP95 price |
| 44 | XG (Xtra Green) price |
| 45 | E100 price |

KSK outlets carry "KSK" or "Kisan Seva Kendra" in the name OR the RO code starts with `KSK`.

## Extraction strategy

**Tier 2 — direct POST.** Because `MapLocations` is a session-scoped endpoint:

1. GET `https://sdms.indianoil.in/PumpLocator/routeROs.jsp` first to obtain `JSESSIONID` + `TS01799c4f` cookies.
2. POST `MapLocations` with `Referer: …/routeROs.jsp` + `X-Requested-With: XMLHttpRequest` + form `latitude=X&longitude=Y&lat=X&long=Y` — the (lat, lng) anchor selects the radius around which up to ~50 nearest outlets are returned.
3. Sweep a lat/lng grid across the target state's bounding box (4×4 = 16 calls is enough for Goa). Dedupe by RO code.
4. Project to canonical `RetailStore`. Classify KSK via name/RO-code regex.

For LPG distributors: separate extractor against `indane.co.in` to be built when that site is reachable. Use `locationType: "distributor"`.

## Field inventory

| Field | Source | Notes |
|---|---|---|
| `brand` | constant — `"Indian Oil"` | |
| `storeId` | col 38 (RO code) | Source's primary key |
| `name` | col 0 | Outlet trade name |
| `address.line1` | col 3 | Full address string |
| `address.city` | col 34 | District (closest match for "city") |
| `address.state` | col 35 | |
| `address.postalCode` | regex `\b\d{6}\b` from address | Inline in the address string |
| `lat` / `lng` | col 1 / 2 | |
| `phone` | col 36 | Outlet contact number |
| `locationType` | derived | "store" for all retail + KSK |
| `storeType` | derived | "Retail Outlet" vs "Kisan Seva Kendra" |
| `_extra.outlet_type` | constant | "retail" |
| `_extra.is_ksk` | derived | true for KSK outlets |
| `_extra.ro_code` | col 38 | Preserved verbatim for dedupe |
| `_extra.dealer_operator` | col 30 | |
| `_extra.sales_officer_contact` | col 29 | |
| `_extra.state_office` / `divisional_office` / `sales_area` | col 31/32/33 | IOCL's internal sales hierarchy |
| `_extra.prices` | cols 25-28, 42-45 | Dict of fuel grade → Rs/L (N/A values dropped) |
| `_extra.covid_relief_contact` | col 40 | Only when col 39 = Y |

## Files in this repo

- **Extractor:** `scrapers/sales-distribution/iocl-indian-oil.ts` (parser + projection + dedup + grid builder)
- **Run script:** `scrapers/sales-distribution/iocl-indian-oil-run.ts` (Phase D pipeline — live sweep with fixture fallback when SDMS is in maintenance)
- **Fixture:** `__tests__/fixtures/iocl-indian-oil-goa-sample.txt` (synthetic 8-row Goa fixture matching the reverse-engineered column shape)
- **Tests:** `__tests__/iocl-indian-oil-stores.test.ts` — 14 tests, 149 assertions (all green)
- **Output:** `data/results/iocl-indian-oil-stores.jsonl`

## Pagination

None. Radius-based per call (~50 results / call). **Grid sweep** is the production pattern — 4×4 grid over Goa; larger states need a denser grid.

## Scope-down strategy

National extraction (~56,000 records across 3 networks) is explicitly out of scope. Recon proves the pattern works against ONE state. To scale up later:

1. Reuse `STATE_BBOXES` + `buildLatLngGrid` per state — add new states by adding their bbox.
2. For each state, calibrate grid density. Goa is ~3,700 km² → 4×4 works. Tamil Nadu is ~130,000 km² → ~10×10 grid.
3. For LPG distributors: build a separate extractor against `indane.co.in` once that domain is reachable.
4. For KSK-only sweeps: same endpoint, post-filter by `is_ksk` — no separate API needed.

## Gotchas

- **`iocl.com` is Sucuri-protected** for raw curl with default UA — returns HTTP 307 with no Location header. Use a full Chrome UA + browser context, or skip the corporate site entirely (the locator backend `sdms.indianoil.in` is unprotected).
- **The state-wise / district-wise JSP pages return 200 even in maintenance mode.** The page body contains `<script>window.location.href = 'exception.jsp';</script>` injected into the empty state-list `<select>`. Health-probe must scan the body for `"exception.jsp"`, not rely on status code.
- **`MapLocations` requires session warmup.** Direct curl POST returns 302 → exception.jsp. The `JSESSIONID` from a GET to `routeROs.jsp` (or any working JSP) is mandatory.
- **Payload is pipe/CSV — not JSON.** Don't try to JSON.parse it.
- **Trailing pipe** is the source's own convention — `parseMapLocationsResponse` filters empty splits.
- **Column 41** appears reserved/unused; cols 42-45 carry the newer fuel grades. Don't assume contiguous indices.
- **COVID-relief columns (39-40) are legacy** (pandemic-era) but still in every payload — keep them.
- **KSK detection is name-based** because the source feed doesn't carry a typed flag — the convention is "KSK" in the trade name or RO-code prefix. Verify against `indianoil.com/pages/ro-ksk-dealerships-overview` if a sample turns up tagged inconsistently.
