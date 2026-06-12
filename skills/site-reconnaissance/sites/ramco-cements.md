# Ramco Cements

**Domain:** ramcocements.in
**Vertical:** Sales & distribution (cement channel partners — Madras-HQ, south India's largest cement player by volume)
**Last verified:** 2026-05-22
**Tier:** 3 (stealth Playwright — domain-locked reCAPTCHA v3 gate forces real-browser execution)
**Framework:** Next.js SSR (nginx 1.24) + Strapi v4 CMS (`/cms/graphql` endpoint) + `react-google-recaptcha` invisible widget
**Protection:** **Imperva** (`x-cdn: Imperva`, `visid_incap_*` / `incap_ses_*` cookies; non-aggressive on HTML pages) + **reCAPTCHA v3** (site key `6Le3Qq0cAAAAABwUe034uZQbK6UTVwA6Qt3dsmeG`, action `submit`, domain-locked to `ramcocements.in`) — captcha gates every locator GraphQL query
**Parent:** The Ramco Cements Ltd (publicly listed; part of the Ramco Group conglomerate)

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| YAML's `official_locator_url` | `/dealer-locator` | **HTTP 404 (Next.js page-not-found)** |
| **Live locator page** | `/locator` | https://www.ramcocements.in/locator |
| Strapi GraphQL endpoint | `/cms/graphql` | https://www.ramcocements.in/cms/graphql |
| Sample / demo store list | `query { stores { ... } }` (Strapi entity) | 4 demo rows only — NOT the real channel network |

The YAML's `/dealer-locator` is a Next.js 404 — the live widget is at `/locator` (linked from the global footer's "Store Locator" entry).

## Hydration payload — five GraphQL queries

All five live behind reCAPTCHA v3 and are wired in `/_next/static/chunks/pages/locator-*.js`:

| Query | Args | Returns |
|---|---|---|
| `loadStates` | `captcha:String` | `string` (JSON-encoded array of `{Text, Value}` state labels) |
| `loadDistricts` | `state:String, captcha:String` | `string` (JSON-encoded array of districts) |
| `loadLocations` | `state:String, district:String, captcha:String` | `string` (JSON-encoded array of locations) |
| **`getStoresFromZIP`** | `zipcode:String, captcha:String` | `string` (JSON-encoded array of dealer rows — primary extraction surface) |
| `getStoresFromLatLon` | `lat:String, lon:String, captcha:String` | `string` (JSON-encoded array, same shape as `getStoresFromZIP`) |

**Critical wire quirk:** every query returns a *JSON-encoded STRING* inside its data field. To parse: `JSON.parse(j.data.getStoresFromZIP)`. The empty-result sentinel is the empty string `""` (NOT `null`, NOT `[]`).

Anonymous calls (no/wrong captcha) → `{"data":{"<queryName>":""}}` — i.e. the resolver hard-validates the token server-side, silently returning the empty-result sentinel rather than an error.

### `getStoresFromZIP` row shape

```json
{
  "Cust_No": "230012043",
  "Cust_Name": "TILE HOUSE",
  "addr_addrline1": "No.97 & 98 BASIN BRIDGE ROAD",
  "addr_addrline2": "MINT",
  "addr_addrline3": "CHENNAI",
  "addr_city": "MINT",
  "addr_state": "",
  "addr_country": "",
  "PinCode": "600001",
  "Clo_Mobile": "9600048360",
  "Clo_EMail": "tilehousemint@gmail.com",
  "lat": "13.106134",
  "lng": "80.279466"
}
```

- `Cust_No` is **per-product-line, NOT per-location** — the same outlet recurs across `Cust_No`s (Hyderabad pincode 500001 returns 3 rows for "TRIMURTHI CEMENT DISTRIBUTORS" at the identical address/phone, under codes `151110595`, `191110595`, `111110595`). Treat it as the dealer's ProductLine×PostingCode key. **Dedup model is `(lat,lng,phone)`; merged `Cust_No`s preserved in `_extra.custNos`.**
- `addr_state` and `addr_country` are **empty strings in every observed row** — state must be RECOVERED from `PinCode` via the India Post PIN-prefix → state map (see `pincodeToState` in the extractor).
- `addr_city` is frequently the sub-locality (e.g. `MINT`, `PARRYS`, `MANNADY`) rather than the metro name. Preserved verbatim (title-cased) — the `PinCode` + state pair is the authoritative geo key.
- `lat`/`lng` ship as **strings**, even though numeric.

## Extraction strategy

**Why Tier 3 is unavoidable.** The reCAPTCHA v3 site key is domain-locked to `ramcocements.in`. Three attempts at obtaining a token bypass-the-page failed:
1. Direct `grecaptcha.execute(siteKey, {action:"submit"})` from a foreign Playwright origin → "Invalid site key or not loaded in api.js".
2. Calling the same from the `/locator` page's own context but BEFORE the page's React app initialised the widget → same error.
3. Sending an arbitrary token to the GraphQL endpoint → resolver returns the empty-string sentinel `{"data":{"<query>":""}}`.

Only path that worked: **drive the locator page itself.** Fill `input[name="findStoreZipcode"]` + click `button:has-text("Find Stores")` and the page's own `react-google-recaptcha` widget mints a valid token and dispatches the GraphQL call. Intercept the response via `page.waitForResponse`.

**Sweep strategy (Ola-style two-phase, south + east bias).** Ramco's locator returns 0 for northern pincodes (110001 Delhi, 400001 Mumbai, 411001 Pune) — its channel network is south + east only. The runner defaults to `--start 500000 --end 855000` (covers TN/AP/TS/KA/KL/WB/OR/BR/AS); `--full` extends to the all-India range.

1. **Phase 1 — coarse pincode sweep** (default stride 1000, ≈355 pincodes over the south+east range). Each `getStoresFromZIP` POST returns up to ~13 dealers (no fixed cap observed — much more permissive than Dalmia's 5-cap).
2. **Phase 2 — zoom around every found dealer's pincode** (±100, stride 5, up to 3 rounds; saturation = <1% new outlets per round).
3. Dedupe by `(lat,lng,phone)`; merge variant `Cust_No`s into `_extra.custNos`.

**Rate-limit reality.** Single page, sequential pincodes, ~500–700 ms apart. Each query needs a fresh captcha-token mint via the in-page widget, and grecaptcha aggressively rate-limits concurrent invocations from the same page. Parallelising the browser context costs more than it saves at this volume.

## Field inventory

| Field | Source (raw) | Notes |
|---|---|---|
| `brand` | hardcoded "Ramco Cements" | |
| `storeId` | `Cust_No` | Per-product-line code — kept as primary id for traceability; variant codes for the same outlet in `_extra.custNos` |
| `name` | `Cust_Name` | Trailing comma stripped (a handful of rows end in ` ,`) |
| `address.line1` | `addr_addrline1` + `addr_addrline2` + `addr_addrline3` (comma-joined, blanks dropped) | Three CRM address columns; each ≤30 chars typically |
| `address.city` | `addr_city` (title-cased) | Often sub-locality not metro; `.` / `-` placeholders nulled |
| `address.state` | `pincodeToState(PinCode)` fallback (source ships `""`) | India Post PIN-prefix lookup; 100% recovery on south+east PINs |
| `address.postalCode` | `PinCode` | Verbatim 6-digit |
| `lat` / `lng` | `lat` / `lng` (string→number) | 100% coverage on observed rows |
| `phone` | `Clo_Mobile` | Normalised to `+91XXXXXXXXXX` |
| `email` | `Clo_EMail` | Lower-cased; null when blank |
| `url` | — | Source ships no per-dealer URL |
| `hours` | — | Source ships no hours field |
| `locationType` | hardcoded `"dealer"` | Channel-partner network |
| `_extra.custNos` | `[Cust_No, ...]` | Variant per-product-line codes for the same physical outlet |

## Files in this repo

- **Extractor:** `scrapers/sales-distribution/ramco-cements.ts` (pure functions — parsers, normalisers, dedupe, sweep helper)
- **Production runner:** `scrapers/sales-distribution/ramco-cements-run.ts` (Playwright driver + Phase D pipeline: scorecard + snapshot + alerts)
- **Fixtures:** `__tests__/fixtures/ramco-cements-pincode-600001.json` (Chennai, 13 rows) + `__tests__/fixtures/ramco-cements-pincode-500001.json` (Hyderabad — 3 rows, all the same outlet under different `Cust_No`)
- **Tests:** `__tests__/ramco-cements-stores.test.ts` (20 assertions over parser / normaliser / dedupe / PIN-to-state / projection)
- **Output:** `data/results/ramco-cements-stores.jsonl`

## Pagination

None. Each `getStoresFromZIP` call returns "all dealers within Ramco's geocoded radius around the queried PIN". No cursor, no offset, no per-state enumeration on this endpoint. Coverage emerges from the pincode-sweep grid + zoom rounds.

The `loadStates → loadDistricts → loadLocations` chain is an alternate enumeration, but each terminal `loadLocations` returns a list of LOCATION LABELS (not dealers) and the actual dealer fetch still has to round-trip through a different API call. The pincode sweep is the cleaner shape.

## Vendor comparison vs the other Indian cement majors

| Brand | Tier | Locator stack | Auth gate | Scope (recon-observed) | Notes |
|---|---|---|---|---|---|
| **UltraTech Cement** (Aditya Birla) | 2 | AEM `/bin/exceltojsonservlet` | none | 3,956 across 25 states | One XHR POST returns the entire Excel-backed dataset |
| **Ambuja Cement** (Adani) | 2 | `ambujahelp.in` REST (Next.js + Sitecore + custom REST) | none | 10,353 across 25 states | Per-state sweep on `/dealersapi/dealers/{state}` |
| **Shree Cement** | 1 | Synup white-label HTML grid (`EThemeForMasterGrid`) | none | 242 across 12 states | State×page HTML sweep; hidden inputs per card |
| **Dalmia Cement** | 2 | CodeIgniter `/welcome/getDealerByPincode` | Radware + CSRF (cookie-jar bootstrap) | south + east only | Pincode sweep; ≤5 dealers/pin hard cap; per-pincode JSON |
| **Ramco Cements** | **3** | **Strapi v4 GraphQL + reCAPTCHA v3** | **Domain-locked invisible captcha** | south + east only | **Only major requiring stealth Playwright**; per-pincode GraphQL; no cap; rich row shape |
| **ACC Limited** (Adani) | n/a | (no public locator) | n/a | 0 | Stripped consumer surfaces post-acquisition |

Ramco is the **only Indian cement major with a real anti-bot captcha gate.** The rest publish their dealer networks via either unprotected REST endpoints (UltraTech, Ambuja, Shree) or behind a recoverable CSRF cookie pair (Dalmia). The captcha makes pure-HTTP impossible — Tier 3 (Playwright) is mandatory.

## Gotchas

- **YAML's `/dealer-locator` is a 404.** The live widget is `/locator`. Updated in YAML alongside this addendum.
- **Domain-locked reCAPTCHA v3.** Cannot mint tokens from a foreign Playwright origin or by calling `grecaptcha.execute(siteKey, ...)` directly even on the right page. Must drive the form (`input[name="findStoreZipcode"]` + `button:has-text("Find Stores")`) and let `react-google-recaptcha` mint each token.
- **Empty-string sentinel.** Bad captcha → `{"data":{"<query>":""}}` (silent fail, no GraphQL `errors`). Must inspect the data field before parsing.
- **Doubly-encoded JSON.** Every locator query returns a JSON-encoded STRING in its data field — `JSON.parse(j.data.getStoresFromZIP)` to get rows.
- **`addr_state` and `addr_country` are always blank.** State recovery from PIN-prefix is mandatory.
- **`addr_city` is often the sub-locality**, not the metro (e.g. `MINT`/`PARRYS`/`MANNADY` for Chennai 600001 rows). Don't rely on it for geo aggregation — use `PinCode` + recovered state instead.
- **`Cust_No` is NOT a per-location key.** Hyderabad 500001 returns three identical-address rows for "TRIMURTHI CEMENT DISTRIBUTORS" with three different `Cust_No`s. Dedupe on `(lat,lng,phone)`.
- **`lat`/`lng` are strings.** `parseFloat` before using; some rows ship as `"17.3956"` (5 digits) vs `"13.106134"` (6 digits) — both parse fine but watch precision for dedupe key.
- **North-India pincodes return empty.** Don't waste sweep budget on PIN 110000–499999; default `--start 500000` skips the dead range. Use `--full` to verify if needed.
- **Imperva on the front-end** is mild — a plain Chrome UA + the stealth init script gets through. Captcha is the real gate.
