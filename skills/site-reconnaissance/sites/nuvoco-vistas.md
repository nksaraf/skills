# Nuvoco Vistas

**Domain:** nuvoco.com
**Vertical:** Indian cement — Nirma Group's cement arm (post-Lafarge India 2016)
**Last verified:** 2026-05-22
**Tier:** 1 (HTTP + bundle mining; no browser, no protection)
**Framework:** Create-React-App SPA (`/static/js/main.<hash>.js`, single-bundle build)
**Protection:** none (default Chrome UA passes; no Cloudflare/Akamai/Imperva)

## TL;DR

Nuvoco does NOT have a public dealer locator. The YAML's `/dealer-locator`
URL renders the homepage SPA shell (every unmapped route does — `/find-dealer`,
`/store-locator`, `/our-presence`, `/dealers`, `/cement-plants/<slug>` all
fall through to the same 12 KB shell). The React Router defined in the
bundle exposes ~25 routes — none of them is a locator. `/contact-us` is a
"Submit your enquiry" form; `/nuvoco-zero-m-partner` is a "Become a Dealer"
*application* form (inbound recruitment, not a directory).

What the site DOES publish is an internal `greatPlaces` array embedded in
the React bundle — the data source for an "Our Presence" Google-Maps widget.
72 records covering plants + offices, with full lat/lng, address, phone,
and a marketing description per facility.

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| SPA shell | any route | https://www.nuvoco.com/dealer-locator |
| JS bundle | `/static/js/main.<hash>.js` | https://www.nuvoco.com/static/js/main.3142f4ae.js |

## Hydration payload

- **Location:** inside the minified JS bundle, as a `defaultProps={...,greatPlaces:[...]}` literal
- **Source name:** `greatPlaces` (internal Google-Maps wrapper component default-prop)
- **Schema (one row):**
  - `id` → numeric, mostly 1..74 with a few gaps
  - `lat` / `lng` → decimal degrees
  - `title` → "Arasmeta Cement Plant" / "Faridabad" / "Corporate & Registered Office"
  - `company` → optional, usually "Nuvoco Vistas Corp. Ltd."
  - `address` → free-text
  - `phone` → free-text (multi-extension common)
  - `IsCompany` → minified boolean (`!0` / `!1`)
  - `ctid` → 1=cement plant, 2=RMX plant, 3=office, 4=innovation centre
  - `location` → state name (set on ctid:1 cement plants; usually blank elsewhere)
  - `description` → marketing HTML blob (cement plants only)
  - `link` → goo.gl/maps short link
  - `url` → canonical Google Maps URL
  - `bannerImg` → file name of a banner image (cement plants only)

Verified breakdown on 2026-05-22:

| ctid | category           | count |
|------|--------------------|-------|
| 1    | cement plants      | 11    |
| 2    | RMX plants         | 44    |
| 3    | corporate offices  | 16    |
| 4    | innovation centre  |  1    |
| —    | **total**          | **72** |

## Extraction strategy

Tier 1 — three GETs:

1. `GET https://www.nuvoco.com/dealer-locator` returns the 2.4 KB SPA shell HTML.
2. Regex out `/static/js/main.[a-f0-9]+\.js` (bundle hash rotates on
   redeploy; do NOT hard-code).
3. `GET <bundleUrl>` returns ≈ 2 MB of minified JS.
4. Locate `greatPlaces:[…]` via a bracket-balanced scan that respects JS
   string literals (single + double quoted) — descriptions contain `[…]`
   substrings (`<ul><li>…</li></ul>`) that naive bracket counting would
   trip over.
5. Convert JS-object-literal to JSON via three string-aware transforms:
   - `IsCompany:!0` / `!1` → `true` / `false`
   - single-quoted JS strings → double-quoted JSON (escape embedded `"`)
   - bareword keys → `"key":` (only at object/array member positions; in-string
     identifiers are untouched)
6. `JSON.parse` the result.

No `eval` / dynamic-function evaluation; the string-aware munge round-trips
72/72 rows deterministically (security-hook clean).

## Field inventory

| Field | Source path (in raw row) | Mapped to | Notes |
|---|---|---|---|
| name | `title` | `name` | "Arasmeta Cement Plant" / "Faridabad" / "Corporate & Registered Office" |
| storeId | `id` | `nv-<id>` | Bundle-assigned numeric, stable across deploys (verified by hash diff) |
| lat / lng | `lat`, `lng` | top-level | 100% populated; every record inside India bounding box |
| address.line1 | `address` | line1 | unparsed free-text (kept verbatim) |
| address.state | `location` ∪ address mining ∪ title mining | state | source's `location` is set on ctid:1 only; we mine state from address (31-pattern allowlist) and fall back to the title (RMX rows ship the city as `title`); 72/72 coverage on 2026-05-22 |
| address.postalCode | regex over `address` | postalCode | `\b(\d{3}\s?\d{3})\b`; 64/72 (89%) |
| phone | `phone` | phone | mix of 10-digit mobiles and multi-extension landlines; we +91-prefix bare 10-digit only |
| url | `url` ∪ `link` | url | canonical Google Maps URL preferred over goo.gl short link |
| locationType | derived | `"depot"` | canonical schema lacks "plant" / "office"; depot is the closest fit ("warehouse / distribution hub") |
| storeType | `ctid` → label | "cement-plant" / "rmx-plant" / "office" / "innovation-centre" | the four Nuvoco categories ride here as free text |
| features | `[type:<storeType>]` | features | enables downstream filtering |
| _extra.description_plain | strip-HTML of `description` | _extra | plant-only marketing copy with `<b>capacity</b>` etc. flattened |
| _extra.map_link / map_url | `link` / `url` | _extra | both short + canonical preserved for verification |

## Files in this repo

- **Extractor:** `scrapers/sales-distribution/nuvoco-vistas.ts`
- **Production runner:** `scrapers/sales-distribution/nuvoco-vistas-run.ts`
- **Recon driver:** `scrapers/recon-nuvoco-vistas.ts`
- **Fixture (SPA shell):** `__tests__/fixtures/nuvoco-vistas-shell.html`
- **Fixture (greatPlaces literal):** `__tests__/fixtures/nuvoco-vistas-great-places.js`
- **Tests:** `__tests__/nuvoco-vistas-stores.test.ts` (16 tests / 528 assertions)
- **JSONL output:** `data/results/nuvoco-vistas-stores.jsonl` (72 records)

## Pagination

None — the entire dataset ships in one bundle GET.

## Gotchas

1. **The YAML's `/dealer-locator` is a SPA fallback, not a real route.**
   It returns 200 with the homepage shell. Don't trust the YAML
   `official_locator_url` field at face value for SPAs — verify the React
   Router actually defines the path.

2. **Bundle hash rotates on redeploy.** The current `3142f4ae` hash will
   change. Always regex it out of the shell HTML — never hard-code.

3. **`greatPlaces` is preceded by `defaultProps={center:[…lat,lng…],zoom:5,greatPlaces:[…]}`.**
   The bracket-balanced scan MUST respect string literals — plant
   descriptions embed `<ul><li>…</li></ul>` which contains square brackets
   in fragment shapes (`[…]` substrings). A naive depth counter that
   doesn't skim quoted strings will close the array early.

4. **`IsCompany:!0` / `IsCompany:!1` are the only minified booleans we
   see** — substituting just these two is enough; no general `!0/!1`
   regex needed (which could mis-trigger on numeric expressions).

5. **State is `location` on ctid:1 only.** RMX plants (ctid:2) and
   offices (ctid:3) ship blank `location` — must be mined from the
   address blob via an allowlist + city-fallback map. Spelling variants
   in the source: "Chattisgarh" (Arasmeta record — typo), "Uttrakhand"
   (Rudrapur record — typo), "Telanagana" (Miyapur record — typo).

6. **No dealer / channel-partner data anywhere on nuvoco.com.** The Q1
   FY26 corporate disclosures cite ~25,000 channel partners but none of
   that surfaces on the public web. Authoritative count for this
   extractor's scope is the published facility footprint (72), NOT the
   national dealer aggregate.

7. **Two records describe "RMX Project" sites** — Runwal Bliss Project +
   Oberoi Skycity Mall — both Mumbai captive batching plants for specific
   construction projects. They have lat/lng but no postalCode. Filter on
   `_extra.ctid_label === "rmx-plant"` AND `name.endsWith("Project")` if
   you want to exclude these from a pure-facility view.

## Sibling-brand probe

Nuvoco's three published consumer brands are **Concreto**, **Duraguard**,
**Premium Slag Cement / Double Bull** (a regional Concreto sub-brand).
There is no separate consumer-store/dealer surface for any of these — all
three roll up under the same nuvoco.com SPA. Probe: NEGATIVE — no shared
backend with sibling cement majors (UltraTech / Shree / Ambuja-ACC). Each
of those publishes its own locator; Nuvoco does not.
