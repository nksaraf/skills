# Dalmia Cement

**Domain:** dalmiacement.com
**Vertical:** Sales & distribution (cement channel partners — India's #4 cement producer, south + east stronghold)
**Last verified:** 2026-05-21
**Tier:** 2 (single XHR POST to a CodeIgniter PHP endpoint, session-bootstrap required)
**Framework:** CodeIgniter PHP (`/welcome/...` controller routes, `dalmia_csrf_*` cookies, `ci_session`)
**Protection:** **Radware Bot Manager** ("Unauthorized Activity Detected") — blocks naive POSTs; passes once a clean session + CSRF hash are established by GETting the bootstrap page
**Parent:** Dalmia Bharat Ltd (publicly listed)

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| YAML's `official_locator_url` | `/dealer-locator` | **HTTP 500** (dead route — do NOT use) |
| Live locator page | `/knowledge-centre/#findDealerSection` | https://www.dalmiacement.com/knowledge-centre/#findDealerSection |
| Bootstrap GET (mints cookies + CSRF) | `/knowledge-centre/` | https://www.dalmiacement.com/knowledge-centre/ |
| **Dealer-by-pincode XHR (POST)** | `/welcome/getDealerByPincode` | body `pincode=600001&dalmia_csrf_token=<hash>` |

The YAML's `/dealer-locator` path is a CodeIgniter 500 dead-end. The live widget is one section down on `/knowledge-centre/` — discovered via the homepage's "Dalmia Dealers → Find A Dealer" menu, which links to `#findDealerSection`.

## Session bootstrap (mandatory)

The XHR endpoint will not respond cleanly without all three of:

1. **`ci_session` cookie** — minted by ANY GET on the site (e.g. `/knowledge-centre/`).
2. **`dalmia_csrf_cookie`** — minted in the same response. Value must MATCH the body's `dalmia_csrf_token` field.
3. **`dalmia_csrf_token` body field** — the hash inside `<meta name="csrf-token-hash" content="…">` on the bootstrap page's HTML. Same value as `dalmia_csrf_cookie` (CodeIgniter's standard double-submit cookie pattern).

Without these:
- `POST` with no cookies → `HTTP 500` empty body.
- `POST` with cookies but no/wrong CSRF → `HTTP 200` BUT body is the Radware "Unauthorized Activity Detected" HTML.
- `POST` with the WRONG `Referer` header (e.g. nothing or a foreign origin) → Radware likewise.

What works (verified 2026-05-21): plain Chrome UA + `Referer: https://www.dalmiacement.com/knowledge-centre/` + `X-Requested-With: XMLHttpRequest` + the cookie pair + the CSRF body field. No JS execution, no Playwright, no proxy.

## Hydration payload

- **Endpoint:** `POST /welcome/getDealerByPincode`
- **Body (`application/x-www-form-urlencoded`):**
  ```
  pincode=<6-digit>&dalmia_csrf_token=<hash>
  ```
- **Response:** `Content-Type: text/html; charset=UTF-8` (despite the JSON-array body — this is a CodeIgniter `echo json_encode(...)` shape). Two response shapes:

  - The string literal `notfound` — zero dealers within radius.
  - A JSON array of up to **5** dealer rows, sorted ASC by `distanceKm`:

    ```json
    [{
      "businessName": "MURUGAN AGENCIES",
      "address": "70, Mint St, Peddanaickenpet, George Town, Chennai, Tamil Nadu 600001, India, Chennai",
      "distanceKm": "1.1",
      "location": {
        "geoCode": { "latitude": 13.103722, "longitude": 80.279922 },
        "pincode": "600001"
      },
      "contactPerson": "P MURUGAN .",
      "contactNo": "9382330779",
      "directionLink": "https://www.google.com/maps/dir/?api=1&origin=…&destination=…",
      "classification": null
    }, …]
    ```

- **Capped at 5 dealers per pincode by distance.** No way to override (no `?radius=…` or `?limit=…` knob exists — the body fields are validated server-side).
- **`distanceKm` is request-scoped** (km from the queried pincode, not the dealer's location). It is intentionally NOT preserved in `_extra` — preserving it would defeat dedupe (same dealer would carry different distances across overlapping pincode sweeps).
- **`classification` is always `null`** in current responses — preserved verbatim if it ever populates.
- **`contactPerson`** ends with a stray ` .` in every observed row (data-entry quirk) — normalised by `normaliseContactPerson`.

## Extraction strategy (Ola-style two-phase sweep)

Identical idea to the Ola Electric extractor:

1. **Phase 1 — coarse pincode sweep across India.** Stride 500 by default (`buildPincodeSweep(500)` → ~1,490 pincodes). Each pincode returns ≤5 nearby dealers; we dedupe by `(lat,lng,phone)` since Dalmia ships no stable id field.
2. **Phase 2 — zoom around every found dealer's own pincode** (±100 stride 5 = 41 calls per seed), up to 3 rounds. Each round adds fewer outlets until <1% new per round → saturation.

Why a pincode sweep and not a state/district enumeration? **Because none of those endpoints exist.** The widget is a pincode-only input; there is no `getDealersByState` / `getDealersByDistrict` route on `/welcome/*` (probed and 404s).

**Scope of "everything".** Dalmia Bharat publishes no channel-partner count in its investor materials. The public locator surfaces only the dealers indexed by Dalmia's geocoder for pincode-proximity search. Coverage is regional: south + east India (TN, AP, Telangana, Karnataka, Kerala, WB, Odisha, Bihar, Jharkhand, NE corridor) returns dealers; north India (Delhi, UP, Punjab, Haryana, Rajasthan) consistently returns `notfound`. This matches Dalmia's known market footprint.

## Field inventory

| Field | Source | Notes |
|---|---|---|
| `brand` | hardcoded "Dalmia Cement" | |
| `storeId` | **null** — source ships no stable id | Dedupe relies on (lat,lng,phone) tuple |
| `name` | `businessName` | Mixed case (proper) |
| `address.line1` | `address` (free text, verbatim) | Trailing-comma format: `<line>, <City>, <State> <pin>, India, <City>` |
| `address.city` | trailing comma chunk of `address` | Repeated as suffix after `, India,` — we prefer that |
| `address.state` | `address` segment before `<6-digit pin>, India` | `normaliseState` canonicalises shorthand |
| `address.postalCode` | embedded 6-digit pin in `address` | NOT `location.pincode` (which is the REQUESTED pincode, not the dealer's) |
| `lat` / `lng` | `location.geoCode.latitude/longitude` | 100% coverage on observed rows |
| `phone` | `contactNo` | 10-digit Indian mobile, normalised to `+91XXXXXXXXXX` |
| `url` | `directionLink` (Google Maps direction URL) | Useful as a "view on map" link |
| `locationType` | hardcoded `"dealer"` | Channel-partner network — no service-centre tier surfaced |
| `_extra.contactPerson` | `contactPerson` (normalised: trailing `.` stripped) | Useful field source-only; not in canonical schema |
| `_extra.classification` | `classification` | Always `null` today; preserved for forward-compat |

## Files in this repo

- **Extractor:** `scrapers/sales-distribution/dalmia-cement.ts` (pure functions)
- **Production runner:** `scrapers/sales-distribution/dalmia-cement-run.ts` (session-bootstrap + sweep + Phase D pipeline)
- **Fixtures:** `__tests__/fixtures/dalmia-cement-pincode-600001.json` (Chennai, 5 rows) + `dalmia-cement-pincode-500001.json` (Hyderabad, 5 rows)
- **Tests:** `__tests__/dalmia-cement-stores.test.ts` (17 tests)
- **Output:** `data/results/dalmia-cement-stores.jsonl`

## Pagination

None. Each pincode POST returns up to 5 dealers, no cursor, no offset. Coverage emerges from sweeping the PIN range and deduping.

## Gotchas

- **YAML's `official_locator_url: /dealer-locator` is a HTTP 500 dead-end.** The live widget is at `/knowledge-centre/#findDealerSection`. Updated in YAML alongside this addendum.
- **Radware Bot Manager fronts the whole site.** A bare POST returns either 500 (no cookies) or 200-with-HTML (cookies but no/wrong CSRF). Always bootstrap by GETting `/knowledge-centre/` first and reusing the cookie jar + CSRF hash.
- **5-dealer hard cap per pincode.** No radius / limit override. Sweep density drives coverage.
- **`distanceKm` is request-scoped.** Don't preserve it in `_extra` — the SAME dealer will carry a different `distanceKm` across overlapping pincode queries, breaking dedupe.
- **`location.pincode` is the requested pincode, not the dealer's.** Use the 6-digit pin embedded in the `address` free-text field instead.
- **`contactPerson` has a trailing `.`** in every observed row (e.g. `"P MURUGAN ."`). Normalised by `normaliseContactPerson`.
- **No stable id**. Source has no `id` / `dealerCode` / `Allied Code` analog. Dedupe is forced to use `(lat,lng,phone)` — works because Dalmia's geocoder returns identical coordinates for the same outlet across queries.
- **Regional coverage.** Northern pincodes (Delhi, UP, Punjab, Rajasthan) consistently return `notfound`. This is real (Dalmia is south + east dominant), not a bug.
- **No `Content-Type: application/json` header** on the response (CodeIgniter ships `text/html`). Don't trust the header; parse the body as JSON if it isn't the literal `notfound`.
