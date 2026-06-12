# Birla Corporation (MP Birla Cement)

**Domain:** birlacorporation.com (corporate, no locator) → **mpbirlacement.com** (real locator)
**Vertical:** Sales & distribution (cement channel partners — M.P. Birla Group cement arm; mid-tier player ~12 MTPA FY24 capacity)
**Last verified:** 2026-05-22
**Tier:** 1 (single HTML XHR endpoint on WordPress admin-ajax)
**Framework:** WordPress 6.8.5 + custom `dealers` CPT (registered via CPT-UI) + a small jQuery widget on `/dealer-network/`
**Protection:** none observed (plain Chrome UA passes; no Cloudflare/Akamai challenge; no cookies/CSRF)
**Parent:** Birla Corporation Limited (M.P. Birla Group; listed; CIN L01132WB1919PLC003334)

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| Corporate IR site (no locator) | `birlacorporation.com/*` | https://www.birlacorporation.com/ |
| Corporate `/dealer-locator` | **HTTP 404** | https://www.birlacorporation.com/dealer-locator |
| Consumer microsite landing | `mpbirlacement.com/` | https://www.mpbirlacement.com/ |
| Locator page (pincode widget) | `/dealer-network/` | https://www.mpbirlacement.com/dealer-network/ |
| **Dealer-search XHR (POST)** | `/wp-admin/admin-ajax.php` with `action=search_dealers&pincode=<6digit>` | (HTML response — see below) |
| Authoritative-count oracle (REST) | `/wp-json/wp/v2/dealers?per_page=1` | ships `x-wp-total: 6415` on 2026-05-22 |
| Per-dealer post (empty body) | `/dealers/{slug}/` | https://www.mpbirlacement.com/dealers/dealer-name-mr-md-rezaullah/ |

The YAML's `official_locator_url` (`birlacorporation.com/dealer-locator`) is **a 404**. The corporate site is investor-comms only and contains no consumer journey. The real locator lives on the consumer microsite `mpbirlacement.com` (linked from the corporate homepage as `http://www.mpbirlacement.com/iwp/`).

## Hydration payload

The locator widget POSTs the pincode to `admin-ajax.php` and substitutes the response HTML directly into `<div id="dealer-results">`. Response shapes:

- **No-match sentinel**:
  ```html
  <p class="error-message">No dealers found</p>
  ```

- **Match payload** (1+ dealer cards wrapped in a grid):
  ```html
  <div class="dealer-grid">
    <div class="dealer-card">
      <h4 class="sub-heading-dealer mb-50 text-center">JEEVANJYOTI AGENCIES (P) LTD</h4>
      <p><strong>Member Name:</strong> NAND KISHORE BAGRODIA</p>
      <p><strong>Mobile Number:</strong> 9831753206</p>
      <p><strong>State:</strong> WB</p>
      <p><strong>Pin Code:</strong> 700001</p>
    </div>
    ...
  </div>
  ```

**Fields per card:** business name (h4), member name, mobile number, state (shorthand for the 8 nominal-footprint states; full names otherwise), pincode. **No coordinates, no email, no street address, no sub-brand, no business hours.**

**Authoritative total.** The `dealers` Custom Post Type ships `x-wp-total: 6415` on the REST endpoint `/wp-json/wp/v2/dealers?per_page=1`. This is the canonical universe size. REST only surfaces titles + IDs (no city/pincode/phone — those fields live in post-meta that isn't exposed via REST), so REST is a totals-oracle, not a data path.

## Extraction strategy (Tier 1 — pincode sweep)

The API is **strict pincode equality** — not nearest-N (Dalmia / Ola Electric), not state-filtered (Ambuja), not bulk-dump (UltraTech). Prefix inputs (`462`, `4620`, `46200`), 7-digit inputs, and empty/wildcard inputs all return the "No dealers found" sentinel. So coverage requires actually iterating Indian pincodes.

**Two-phase sweep** (mirrors Dalmia's pattern):

1. **Phase 1 — coarse stride-10 sweep across India.** Iterate `100000…855999` at stride 10 = ~75k requests. Each request returns 0–N exact-match dealers. Concurrency 25, ~30 ms inter-launch gate = ~33 req/s = ~38 min for phase 1.
2. **Phase 2 — zoom around dealer-bearing pincodes.** For each pincode that surfaced ≥1 card in phase 1, sweep `seed ± 50` at stride 1. Repeat up to 3 rounds, terminating when <1% new dealers per round.

**India-wide, not footprint-only.** The page's "Become a Dealer" form is restricted to 8 states (West Bengal, Bihar, UP, MP, Rajasthan, Haryana, Gujarat, Maharashtra), but the live API surfaces dealers OUTSIDE that subset — e.g. Telangana 500001 returns 2 cards (MOOSANI MARKETING, BAWA STEEL AND CEMENT). This is empirically confirmed in the fixture `birla-corp-pincode-500001.html`. The dataset reflects historical / cross-border channel partners that the dealer-application form no longer recruits for. Footprint-only would silently miss those rows.

## Vendor / backend comparison vs other cement majors

This is the head-to-head across the cement-major recon set. Birla's stack is the **simplest** of the five — a single WP admin-ajax endpoint with pincode-equality semantics.

| Aspect | UltraTech | Shree Cement | Ambuja | ACC | Dalmia | **Birla Corp** |
|---|---|---|---|---|---|---|
| Locator host | ultratechcement.com | dealers.shreecement.com (Synup) | ambujahelp.in (separate microsite) | none (acclimited.com 404s) | dalmiacement.com | **mpbirlacement.com (separate microsite)** |
| Stack | AEM | Synup white-label | Sitecore + Next.js | (none) | CodeIgniter + Radware | **WordPress + admin-ajax** |
| Backend semantics | static Excel dump | server-rendered HTML pages per state | per-state REST | n/a | nearest-5 by pincode | **exact-pincode equality** |
| Calls needed | 1 | ~56 (state × page) | 25 (state) | 0 | sweep + zoom | **sweep + zoom (~75k coarse + zoom)** |
| Authoritative count surface | none | none | none | n/a | none | **`x-wp-total` on REST CPT (6,415)** |
| Sub-brand per dealer | none | yes (Shree Cement / Rock Strong) | no | n/a | no | **no** (composite SKU bag — see below) |
| Lat/lng | absent | present | absent | n/a | present | **absent** |
| Phone | yes (10-digit) | yes (E.164) | yes | n/a | yes | **yes (10-digit, +91 normalised)** |
| Address detail | comma-delimited line | full structured | comma-delimited | n/a | full structured | **none — just state + pincode** |
| Protection | none | none | none | n/a | Radware (cookies + CSRF required) | **none** |
| Public-locator size | 3,956 | 242 | 10,353 | 0 | ~thousands (south + east) | **≥ (to be filled by live run; ≤ 6,415 universe ceiling)** |

**Key takeaway**: every cement major exposes their locator via a different vendor stack. There is no shared infra to exploit (in contrast to BBK Electronics' Vivo/OPPO siblings or Adani's Ambuja/ACC). The Birla extractor is from-scratch.

## Sub-brand handling

Birla markets its cement under multiple sub-brands (Perfect Plus, Unique Plus, Ultimate, Ultimate Ultra, Rakshak, Samrat Advanced, Concrecem, Multicem, Chetak, PSC, SBR), but the **public dealer locator surfaces every dealer as a composite "MP Birla Cement" outlet** — there's no per-dealer sub-brand discrimination in the payload. This is **structurally different from Shree Cement**, whose dealer cards encode the sub-brand in the business-name prefix (e.g. "Shree Cement - Tanya Traders" vs "Rock Strong - …"), letting us split 213 vs 29.

We preserve `_extra.sub_brand = null` on every record to signal "looked, none found" rather than letting the field be absent. If Birla ever surfaces per-dealer sub-brand (e.g. via a separate Samrat- or Multicem-specific locator microsite), the extractor only needs the projection updated; the wire-level parser will already capture the new field if it shows up in a `<p><strong>...</strong></p>` row.

**Sibling-microsite probe.** The corporate homepage links to ten product microsites at `mpbirlacement.com/mp-birla-cement-{perfect-plus,unique,ultimate,ultimate-ultra,samrat,multicem,concrecem,psc,sbr,chetak}/`. Each is a product info page (one HTML, no widget). No per-sub-brand locator exists. The `mp-birla-cement-chetak/` slug is a brand legacy reference, not a separate company.

## Field inventory

| Field | Page type | Source | Notes |
|---|---|---|---|
| `brand` | hardcoded | `"Birla Corporation"` | parent / listed entity name |
| `storeId` | listing | **null** (source ships none) | dedupe uses (phone, name, pincode) |
| `name` | listing | `<h4>` text | business / trading name (SHOUTY in ~95% of rows) |
| `address.line1` | listing | synthesised from state + pincode | source ships NO street address |
| `address.city` | listing | **empty string** | source ships NO city |
| `address.state` | listing | `<p>State:</p>` | normalised from shorthand (WB → West Bengal etc.) |
| `address.postalCode` | listing | `<p>Pin Code:</p>` | always 6-digit when populated |
| `phone` | listing | `<p>Mobile Number:</p>` | normalised to `+91XXXXXXXXXX` |
| `lat` / `lng` | n/a | **null** — source ships none | |
| `email` | n/a | **null** — source ships none | |
| `url` | n/a | **null** — per-dealer page has no useful data | |
| `hours` | n/a | **null** — source ships none | |
| `locationType` | hardcoded | `"dealer"` | source doesn't surface stockist vs retailer |
| `_extra.member_name` | listing | proprietor/partner name (honorific-stripped) | |
| `_extra.member_name_raw` | listing | original verbatim (preserves "MR." prefix) | |
| `_extra.state_raw` | listing | original shorthand (e.g. "WB", "MP") | |
| `_extra.query_pincode` | provenance | the pincode that surfaced this card | |
| `_extra.sub_brand` | hardcoded | **null** | signals "looked, none surfaced" |

## Pagination

None. Each pincode request returns the complete set of dealers at that exact pincode. No `?page=N` semantics on the AJAX endpoint.

## Files in this repo

- **Extractor:** `scrapers/sales-distribution/birla-corp.ts` (pure functions)
- **Production runner:** `scrapers/sales-distribution/birla-corp-run.ts` (sweep + Phase D pipeline)
- **Fixtures:** `__tests__/fixtures/birla-corp-pincode-700001.html` (Kolkata, 3 dealers), `birla-corp-pincode-462001.html` (Bhopal, 7 dealers), `birla-corp-pincode-500001.html` (Hyderabad / out-of-footprint, 2 dealers), `birla-corp-pincode-empty.html` (sentinel)
- **Tests:** `__tests__/birla-corp-stores.test.ts` (19 tests, 535 expects)
- **Output:** `data/results/birla-corp-stores.jsonl`

## Gotchas

- The YAML's `official_locator_url: birlacorporation.com/dealer-locator` is **a 404**. The corporate site has no consumer surface at all. The real locator is on the consumer microsite `mpbirlacement.com/dealer-network/`. (Same gotcha hit UltraTech, Shree, Ambuja, Okinawa, Hero Vida, Revolt, Ola Electric — corporate yaml URLs are routinely stale.)
- The API is **strict pincode equality**, not nearest-N or prefix-match. Empty / 1-5 digit / 7+ digit / wildcard inputs all return the "No dealers found" sentinel. There's NO way to do a bulk dump — the only enumeration path is pincode sweep.
- The "Become a Dealer" form on `/dealer-network/` has 8 states in its dropdown — DO NOT use that subset as a coverage proxy. The live API surfaces dealers in Telangana (500001), and likely other "non-application" states. The realistic sweep is India-wide (100000–855999).
- Pincode 9xxxxx is reserved for Army APO / Navy FPO and confirmed-empty on Birla. The sweep windows exclude circle 9.
- The `dealers` CPT REST endpoint exposes only post titles (= member names) + post IDs, NOT the pincode/state/mobile fields (those live in WP post-meta the REST whitelist doesn't include). REST is a totals oracle (`x-wp-total: 6415`), not a data source.
- Each pincode returns at MOST whatever dealers are registered there — there's no per-page cap. Largest single response we've seen is 7 cards at 462001 Bhopal; most pincodes that have dealers have 1–3.
- ~95% of business names are SHOUTY UPPERCASE; member names have a ~80% "MR." / "MR " prefix that the extractor strips into `_extra.member_name`.
- State field is inconsistent: footprint states use SHOUTY shorthand ("WB", "MP", "UP"); non-footprint states use proper-cased full names ("Telangana"). `normaliseState` canonicalises both.
- 6,415 dealer-CPT posts vs N pincode-bearing dealers — the gap is dealers without a populated `pincode` post-meta (orphan posts the CMS team hasn't backfilled). This is the same "X% of CPT carries usable locator data" pattern Dalmia and Hero Vida exhibit. The completeness score uses 6,415 as the ceiling — gap is documented, not an extractor bug.
- The pincode space sweep is the primary cost. Stride-10 coarse + ±50 zoom is the working tradeoff; tighter strides would catch more pincode-aligned dealers but the gain saturates beyond stride 5.

## Re-probe / re-recon triggers

- WP REST `x-wp-total` shifts more than 5% (canary for CPT churn).
- A new sub-brand-specific locator microsite launches (e.g. `dealers.samrat-advanced.com`).
- The "Become a Dealer" form's state dropdown expands or contracts.
- The locator widget switches transport (e.g. drops admin-ajax for a /wp-json/birla/v1/dealers REST route).
