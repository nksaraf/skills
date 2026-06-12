# Hero Vida

**Domain:** vidaworld.com (the YAML's `official_domain` `hero-vida.com` is a parked GoDaddy lander)
**Vertical:** Sales / distribution — e2W dealer network (Hero MotoCorp's electric two-wheeler sub-brand)
**Last verified:** 2026-05-21
**Tier:** 2 (direct JSON API — AEM JCR selector endpoints)
**Framework:** Adobe Experience Manager (AEM) — `clientlibs` are `/etc.clientlibs/vida/...`, content is at `/content/vida/in/en/...`
**Protection:** none — plain Chrome UA + `en-IN` Accept-Language passes; no Akamai/Cloudflare/Imperva signals on either the locator HTML or the JSON endpoints

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| Locator landing | `https://www.vidaworld.com/dealers-locator.html` | (HTML carries configured API URLs in an `<configurations>` blob) |
| State/city catalog | `/content/vida/in/en/sf-master/jcr:content.allstateandcity.json` | 28 states, 613 cities |
| Vida dealer list | `/content/vida/in/en/sf-master/jcr:content.dealerlistwithoutsku.<STATECODE>.<CITYCODE>.json` | `.DL.DELHI.json`, `.KA.BENGALURU.json` |
| Nearby branches (Hero + Vida mixed) | `/content/vida/in/en/sf-master/jcr:content.nearbybranches.<CITY>.json` | `.DELHI.json` — returns Hero ICE dealers, service centres AND Vida Experience Centres |
| Dealer detail by id | `/content/vida/in/en/sf-master/jcr:content.dealerDetails.<id>.json` | empty `[]` in our probes — looks deprecated |

The `<configurations>` blob in `dealers-locator.html` lists every API URL the SPA uses. It is the authoritative source for discovering endpoints — start there.

## Locator gotcha — the YAML-listed domain is parked

The brand assignment YAML lists `official_domain: hero-vida.com` and `official_locator_url: https://www.hero-vida.com/find-a-store`. Both serve a GoDaddy "parked-lander" page (`<script src="https://img1.wsimg.com/parking-lander/..."/>`). The brand's REAL customer-facing site is `vidaworld.com`, embedded in the Hero MotoCorp product stack but with its own AEM template. **Hero MotoCorp's main site `heromotocorp.com/vida` is also distinct from vidaworld.com.**

## Hydration / payload schema

`dealerlistwithoutsku` response shape:

```json
{ "data": { "dealers": {
  "total_count": 1,
  "page_info": { "current_page": 1, "page_size": 40, "total_pages": 1 },
  "items": [ { /* see VidaDealerItem in scrapers/sales-distribution/hero-vida.ts */ } ]
}}}
```

A "no dealer in this city" response uses GraphQL-style errors:

```json
{ "errors": [ { "message": "Dealer not found with the provided inputs: State: KA, City: BANGALORE" } ],
  "data": { "dealers": null } }
```

`VidaDealerItem` schema (top-level keys observed 2026-05-21):
- `id`           — AEM record id (e.g. "15")
- `code`         — Hero internal dealer code (e.g. "17015") — we use this as `storeId`
- `name`         — display name (e.g. "Himgiri Automobiles - Premia", "Experience Centre Bengaluru - RT Krishna Autos")
- `email`        — dealer contact (typically `<prefix>.<dealer>.<city>@heromotocorp.biz`)
- `phone`        — direct dialer
- `status`       — "1" = open, "0" = closed
- `address_line_1` / `address_line_2`
- `state` / `city` — two-letter state code + uppercase city code (matches catalog keys)
- `zip_code`
- `gstin_number`
- `latitude` / `longitude` — both as strings

## Extraction strategy

Tier 2 dual-endpoint union (revised 2026-05-21):

**Pass 1 — `dealerlistwithoutsku`** (clean but sparse): Loop over the 613 `(state, city)` pairs from `allstateandcity`. For each pair, call `dealerlistwithoutsku.<STATE>.<CITY>.json`. Treat the "Dealer not found" error envelope as the canonical "no Vida outlet here" signal. Yields 8 outlets across 5 cities.

**Pass 2 — `nearbybranches`** (mixed, must filter): For each unique city code, call `nearbybranches.<CITY>.json` and run each row through `isVidaBranch` (see filter rules below). Yields 70 additional outlets across 58 cities.

**Union**: dedupe by `vidaDedupeKey` — a normalised "name + city" string. The two endpoints use disjoint ID spaces (numeric Hero dealer code vs Salesforce-style branch id like `a0KOX00000F6qD22AJ`), so we cannot dedupe by ID. Pass-1 records win on conflict (they carry the Hero dealer code + GSTIN + open/closed status, which pass-2 lacks). 9 of the 79 Vida hits in pass 2 collide with pass 1 → 78 unique outlets total.

Each record carries `_extra.source_endpoint` ∈ {`dealerlistwithoutsku`, `nearbybranches`} so downstream tooling can tell which source it came from.

### Vida-vs-Hero ICE filter (`isVidaBranch`)

`nearbybranches` returns a mixed bag: Vida Experience Centres + Hero ICE Authorised Dealers + ServiceBranch (workshops) + Mobile Service Branch (vans) + Warehouse (inventory). The Vida-only filter:

1. `type === "Experience Centre"` (eliminates ServiceBranch, Mobile Service Branch, Warehouse — those are operational infrastructure, not Vida storefronts).
2. AND one of:
   - Name contains `PREMIA` → Vida Hub (Hero's premium Vida channel).
   - Name contains `VIDA` (e.g. "Max Motors - VIDA HUB") → explicit.
   - Name starts with `EXPERIENCE CENTRE` or `EC-` → Vida Experience Centre.
   - `branchTypeCategory === "Hub"` or `branchTypeCategory === "Experience Center"` (US spelling — diverges from the British "Experience Centre" used in `type`).

Hero ICE Authorised Dealers like `AMAN MOTORS`, `JMD AUTOWORLD`, `KHANNA AUTOMOBILES` all appear under `type === "Experience Centre"` with `branchTypeCategory === "Authorised Dealers"` and no Vida markers in the name — these are rejected. The locator FAQ confirms the distinction: "Walk into any VIDA Hub, Experience Center, or select Hero dealership" — we keep the first two channels, not the third.

Dedupe by `vidaDedupeKey(record)` (name + city, normalised). No browser, no warmup needed — gentle pacing (`rate-ms 80`, `concurrency 12`) is enough.

## Location type discrimination

The assignment requires:
- Vida Hub → `locationType: "store"`
- service → `locationType: "service-centre"`

`classifyLocationType(name)` maps the name:
- contains "Premia" / "Experience Centre" / "Hub" / "Showroom" / "Sales" → `store`
- contains "Service" without sales markers → `service-centre`

Currently 0/8 records map to `service-centre` (the Vida dealer list does not surface service-only outlets via this endpoint). The check is kept defensive for FY25 footprint expansion.

`storeType` (free-text sub-type) is derived from the name: `Premia`, `Experience Centre`, or `null`.

## Field inventory

| Field | Source (JSON path) | Notes |
|---|---|---|
| storeId | `items[].code` | Hero internal numeric (e.g. "17015") |
| name | `items[].name` | encodes sub-type (Premia / Experience Centre) |
| phone | `items[].phone` | direct dial |
| email | `items[].email` | `<prefix>.<dealer>.<city>@heromotocorp.biz` |
| address.line1 | `items[].address_line_1` | one long descriptor; not normalised |
| address.city | `items[].city` | uppercase code in API; we Title-Case |
| address.state | `STATE_CODE_TO_NAME[items[].state]` | two-letter → full name |
| address.postalCode | `items[].zip_code` | 6-digit Indian PIN |
| lat / lng | `items[].latitude` / `.longitude` | strings in API; parsed to numbers |
| status | `items[].status` | "1"→open, "0"→closed |
| storeType | derived | "Premia" / "Experience Centre" / null |
| locationType | derived | store / service-centre |
| _extra | `gstin_number`, `status` (raw), plus any new fields | preservation |

## Files in this repo

- **Extractor (pure functions):** `scrapers/sales-distribution/hero-vida.ts`
- **Production scraper:** `scrapers/hero-vida-stores.ts`
- **Fixtures:**
  - `__tests__/fixtures/hero-vida-allstatecity.json` — 28 states / 613 cities
  - `__tests__/fixtures/hero-vida-dealers-DL-DELHI.json` — 1 Premia in Delhi
  - `__tests__/fixtures/hero-vida-dealers-KA-BENGALURU.json` — 3 outlets in Bengaluru (mix of Premia + Experience Centre)
  - `__tests__/fixtures/hero-vida-dealers-MH-MUMBAI.json` — AEM "Dealer not found" error envelope (the negative case)
  - `__tests__/fixtures/hero-vida-nearbybranches-DELHI.json` — 37 rows; 1 Vida-branded after `isVidaBranch` (HIMGIRI AUTOMOBILES - PREMIA), 36 Hero ICE / service / warehouse rows filtered out
  - `__tests__/fixtures/hero-vida-nearbybranches-BENGALURU.json` — 51 rows; 9+ Vida-branded after filter, demonstrates the gap-closing power (3 via `dealerlistwithoutsku` vs 9+ via `nearbybranches`)
  - `__tests__/fixtures/hero-vida-nearbybranches-MUMBAI.json` — 30 rows; 3 Vida-branded after filter (FORTPOINT/RAJ AUTO/TRADEMARKK — all PREMIA). Mumbai returns "Dealer not found" via `dealerlistwithoutsku`, so this is pure gain.
- **Tests:** `__tests__/hero-vida-stores.test.ts`
- **Phase D pipeline:** `scripts/run-hero-vida-snapshot.ts`
- **Snapshot:** `data/snapshots/sales-distribution/hero-vida/<TIMESTAMP>Z.jsonl`
- **Scorecard:** `data/reports/hero-vida-scorecard.md` + `.json`

## Pagination

`dealerlistwithoutsku` returns `page_info: { current_page: 1, page_size: 40, total_pages: 1 }` and items are always a single page (max observed: 3 dealers in Bengaluru). `nearbybranches` returns an object-keyed flat array (`{ "0": {...}, "1": {...}, ... }`) — no pagination metadata, but the largest observed payload (Bengaluru, 51 rows) is well under any practical page limit. No multi-page handling required at the city level for either endpoint. The **outer** "pagination" is the (state,city) catalog (or unique city list for `nearbybranches`), walked exhaustively.

## `nearbybranches` payload schema (Salesforce-derived)

```json
{ "0": { /* NearbyBranchItem */ }, "1": { /* ... */ }, ... }
// Empty city → []
```

`NearbyBranchItem` fields (observed 2026-05-21):
- `id`                   — Salesforce-style id (e.g. "a0KOX00000F6qD22AJ") — distinct from the Hero dealer code used by `dealerlistwithoutsku`
- `experienceCenterName` — display name (UPPERCASE in source)
- `type`                 — one of `Experience Centre` | `ServiceBranch` | `Mobile Service Branch` | `Warehouse`
- `branchTypeCategory`   — one of `Authorised Dealers` | `Experience Center` (US spelling) | `Hub` | `""` (warehouses, some service rows)
- `address`              — single string ending `, CITY,STATE, India-PIN` (parsed via `splitNearbyAddress`)
- `postalCode`           — 6-digit Indian PIN
- `latitude` / `longitude` — strings; sometimes absent on warehouse / service rows (filtered out anyway)
- `phonenumber` / `servicephonenumber`
- `email` / `serviceemail`
- `testRideAvailable`    — boolean; true on customer-facing Vida outlets
- `accountpartnerId`     — Salesforce partner id; same value across the dealer's Experience Centre + Warehouse rows (a useful cross-row identity if we ever need it)
- `branchImage`          — contains literal `$[env:BRANCH_IMAGE_PATH]` prefix — never resolves; ignored

## Gotchas

- `hero-vida.com` is a parked GoDaddy domain — DO NOT crawl it. Use `vidaworld.com`.
- City codes are not always the obvious spelling: the API rejects `MH.MUMBAI` and `KA.BANGALORE` with "Dealer not found", because Vida's footprint uses neither code (note: "BANGALORE" is also not a city code in the catalog, only "BENGALURU"). Always source codes from `allstateandcity.json`, never invent them.
- `total_count` is the dealer count for one (state,city) page — it is NOT a brand-wide total. There is no brand-wide count endpoint.
- The authoritative count is ~100 (Hero MotoCorp investor materials reference "Vida Hub expansion to ~100 cities by FY24-end"). On 2026-05-21 the original `dealerlistwithoutsku`-only extractor returned 8 outlets — a substantial gap. The dual-endpoint strategy (above) closes the gap to **78** (Premia 57, VIDA Hub 14, Experience Centre 7) — within the ±30% authenticity tolerance. The remaining gap (78 vs ~100) is plausibly explained by (a) the FY24 target slipping into FY25, (b) outlets in the announce pipeline that aren't yet live in either AEM endpoint. The two endpoints overlap (9 of the 70+ pass-2 Vida hits also appear in pass 1), which is a useful internal consistency check — Bengaluru's `RT KRISHNA AUTOS - PREMIA` shows up in both.
- The locator UI exposes other endpoints (`pickuplocation`, `pincodecity`) that ARE tactical (used by the Buy flow) not strategic for a brand-wide store census. `nearbybranches` was historically dismissed for the same reason, but it turns out to be the primary gap-closer for the Vida footprint — the SPA uses it to power the locator map and it contains many Vida outlets that aren't in `dealerlistwithoutsku`. Use both `dealerlistwithoutsku` (clean) AND `nearbybranches` (filtered via `isVidaBranch`).
- `branchImage` URLs in `nearbybranches` contain a literal `$[env:BRANCH_IMAGE_PATH]` template prefix — never resolves; ignore.
