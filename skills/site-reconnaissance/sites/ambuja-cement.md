# Ambuja Cement

**Domain:** ambujacement.com (corporate; no locator) → **ambujahelp.in** (real locator)
**Vertical:** Sales & distribution (cement channel partners — Adani Cement, India's #2 cement producer)
**Last verified:** 2026-05-21
**Tier:** 2 (state-sweep over a vendor REST API)
**Framework:** Sitecore (CMS) + Next.js (front-end) + custom `/dealersapi/*` REST backend
**Protection:** none observed (default Chrome UA passes; no Akamai/Imperva/Cloudflare challenge)
**Parent:** Adani Group → Adani Cement (with sibling ACC Limited)

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| Corporate IR site (no locator) | `ambujacement.com/*` | https://www.ambujacement.com/ |
| Customer microsite landing | `ambujahelp.in/` | https://www.ambujahelp.in/ |
| Locator page | `/find-a-dealer` | https://www.ambujahelp.in/find-a-dealer |
| Sitecore JSS layout (ships state→city→area tree) | `/sitecore/api/layout/render/jss?item=/sitecore/content/Ambuja/Home/find-a-dealer&sc_apikey=<GUID>&sc_site=ambuja` | (returns nested `options[]` with all 25 states + their cities + areas) |
| **Dealer-list XHR (GET)** | `/dealersapi/dealers/{state-label-url-encoded}` | https://www.ambujahelp.in/dealersapi/dealers/Gujarat |
| Dealer-list XHR (extra segments — same payload) | `/dealersapi/dealers/{state}/{city}/{area}` | (server ignores city + area; returns same state-wide list) |

The YAML's `official_locator_url: /dealer-locator` is **stale (404)** on `ambujacement.com`. The corporate site is an investor/CSR portal only. The actual dealer locator lives on the customer-facing Help microsite `ambujahelp.in`.

## Hydration payload

- **Location:** XHR — `GET /dealersapi/dealers/{state}` (one call per state — no national bulk endpoint)
- **Body:** none — plain `GET`
- **Response:**
  ```json
  {
    "labels": { "nameLabel": "Name:", "mobileNoLabel": "Mobile No.:", ... },
    "details": [
      {
        "icon": "location",
        "imageAlt": "",
        "name": "Dinesh Kumar Aggarwal",
        "contact": "9872842323",
        "pincode": "160014",
        "organisation": "Aggarwal Sales Corporation",
        "stateId": "Ch", "state": "Chandigarh",
        "cityId": "Ch03", "city": "Chandigarh",
        "areaId": "22P", "area": "Chandigarh",
        "address": "Khasra No-176,Shop No-Hb 15,Chandigarh,Chandigarh-160014"
      },
      ...
    ]
  }
  ```
- **Live national total (2026-05-21):** **10,389 raw rows → 10,353 unique** across 25 states. Top 5: Gujarat (2,058), Maharashtra (1,098), West Bengal (1,015), Rajasthan (996), Uttar Pradesh (839).
- **No native unique row ID.** Ambuja's payload omits the `DealerID` field that ACC's payload exposes. We synthesise a composite `stateId:cityId:areaId:organisation` key.
- **No lat/lng.** Ambuja's payload omits coords entirely (in contrast to ACC's sibling endpoint which DOES include `latitude` + `longitude`).

## Vendor / backend comparison vs UltraTech

This is the **direct head-to-head**: UltraTech (Aditya Birla / Birla Cement) vs Ambuja (Adani Cement). They are India's #1 and #2 cement producers and both expose a public dealer locator — but the backends are **completely different vendors**:

| Aspect | UltraTech Cement | Ambuja Cement |
|---|---|---|
| Locator domain | `ultratechcement.com` (one domain) | `ambujahelp.in` (separate microsite from corporate `ambujacement.com`) |
| Stack | Adobe Experience Manager (AEM) | Sitecore CMS + Next.js front-end |
| Data backend | Static Excel sheet baked into AEM dispatcher | Live DB-backed REST API |
| API contract | `POST /bin/exceltojsonservlet?filePath=...storelocator.xlsx` | `GET /dealersapi/dealers/{state}` |
| Calls needed | **1** (single national dump, ~890 KB) | **25** (one per state) |
| Payload envelope | `{ "Store Locator Data UBS 24th Feb": [...] }` (key named after the Excel sheet) | `{ "labels": {...}, "details": [...] }` |
| Row schema | `Allied Code, Name, Address, City, State, Pincode, Phone1, Phone2, Id, Index` | `name, contact, pincode, organisation, stateId, state, cityId, city, areaId, area, address` |
| Unique row ID | `Allied Code` (native, ~265/3,956 blanks) | none (we synthesise a composite key) |
| Lat/lng | absent | absent (but **present on ACC's sibling endpoint** — see below) |
| Public-locator size | 3,956 across 25 states | **10,353** across 25 states |
| Protection | none (CloudFront-fronted, plain UA) | none |
| State spellings | inconsistent — `MP`, `UP`, `HP`, `J&K`, `UTTARANCHAL`, `TRIPURA` (canonicalised in extractor) | mostly clean — one misspelling at source (`Himanchal Pradesh`) silently corrected |

**Sibling-brand corroboration (Adani Cement intra-group):**

The locator at `acchelp.in/find-a-dealer` uses the **identical endpoint contract** as Ambuja: `GET /dealersapi/dealers/{state}` returning `{labels, details[]}` with the same row schema. Probing `https://www.acchelp.in/dealersapi/dealers/Chandigarh` confirms this — ACC's payload is byte-shape-identical to Ambuja's *plus* three extra fields (`DealerID`, `latitude`, `longitude`). **This is the same vendor build, same Sitecore content tree, with Ambuja's payload deliberately stripped of coordinates and the dealer-ID — likely an intentional config decision per brand, not a different deployment.** Practical implication: the Ambuja extractor template is one-to-one reusable for ACC; only the host changes (`ambujahelp.in` → `acchelp.in`) and ACC will additionally populate `lat/lng` + a native `storeId`.

UltraTech's stack shares nothing with Adani's — different vendor, different CMS, different data substrate. No code can be shared between the two extractors; only the canonical `RetailStore` projection.

## Extraction strategy

**Tier 2, state-sweep.** The UI is a 3-level state→city→area cascade, but the API only honours the state segment — appending `/{city}` and `/{city}/{area}` returns the identical state-wide payload. So we sweep 25 GETs with polite pacing (~250 ms between states) and concatenate.

State list is sourced from the Sitecore JSS layout render of the find-a-dealer page (`options[]` array under the state dropdown component). We hard-code the 25 state labels in `STATE_IDS` — if the source ever adds/removes a state, the snapshot diff will flag a count change and the addendum should be re-verified.

## Field inventory

| Field | Source | Notes |
|---|---|---|
| `brand` | hardcoded "Ambuja Cement" | |
| `storeId` | synthesised `stateId:cityId:areaId:organisation` | No native ID in Ambuja's payload (ACC has `DealerID`; Ambuja does not) |
| `name` | `organisation` (with `name` fallback) | Trading name; proprietor `name` kept in `_extra` |
| `address.line1` | `address` | Comma-delimited free text |
| `address.locality` | `area` | Sub-city neighbourhood |
| `address.city` | `city` (light normalisation) | |
| `address.state` | `state` (with one misspelling correction) | Source ships `Himanchal Pradesh` (sic); normalised to `Himachal Pradesh` |
| `address.postalCode` | `pincode` | Always 6 digits |
| `phone` | `contact` | 10-digit Indian mobile → `+91XXXXXXXXXX` |
| `lat` / `lng` | **null** — Ambuja's payload omits coords (ACC's does not) | |
| `locationType` | hardcoded `"dealer"` | |
| `_extra` | preserves `name` (proprietor), `stateId`, `cityId`, `areaId` | |

## Files in this repo

- **Extractor:** `scrapers/sales-distribution/ambuja-cement.ts` (pure functions)
- **Production runner:** `scrapers/sales-distribution/ambuja-cement-run.ts` (state-sweep + Phase D pipeline)
- **Fixture:** `__tests__/fixtures/ambuja-cement-locator.json` (75-row representative sample across all 25 states)
- **Tests:** `__tests__/ambuja-cement-stores.test.ts` (17 tests, 665 expects)
- **Output:** `data/results/ambuja-cement-stores.jsonl` (10,353 records)

## Pagination

None. Each state's full dealer list ships in one response. The largest single response is Gujarat (~2,078 rows). Sweep granularity is fixed at state-level (the API ignores city/area refinements).

## Gotchas

- The YAML's `official_locator_url: /dealer-locator` is **a 404 on `ambujacement.com`**. The real locator is on the customer-facing microsite `ambujahelp.in/find-a-dealer`. The corporate site hosts no public dealer/distributor data.
- The state label `Himanchal Pradesh` is misspelled at source (sic). `STATE_IDS` mirrors that spelling for the URL path; `normaliseState` corrects the row-level `state` field to `Himachal Pradesh`.
- The API returns **the same state-wide payload at any URL depth** — `/state`, `/state/city`, `/state/city/area` all return identical bodies. Do not be tempted to sweep at city/area level — it's wasteful duplication.
- Ambuja's payload **does not include `DealerID` or coordinates** even though the same-vendor ACC endpoint exposes both. Treat lat/lng as absent and dedupe via a synthesised composite key (`stateId:cityId:areaId:organisation`).
- `organisation` (the trading entity name) and `name` (the proprietor's personal name) are distinct fields. We promote `organisation` to `.name` because user-intent for "what's the dealer called" is the trading name, and we preserve the proprietor in `_extra.name`.
- ~10% of rows have a blank `contact` (1,051 of 10,389) — no phone available.
- No authoritative dealer-count figure is published in Ambuja's investor-relations materials (unlike UltraTech's "100,000+" claim that lives in corporate disclosures). The scorecard scores self-consistency against the live sweep size; YAML's `authoritative_count` stays `null`.
- Sibling brand ACC (same Adani Cement parent, same backend vendor) returns identical-shape data plus `DealerID/lat/lng` on `acchelp.in` — direct reuse opportunity for the next extractor.
