# Parle Agro

**Domain:** parleagro.com
**Vertical:** sales-distribution / FMCG bottler (Frooti, Appy, Appy Fizz, Bailley, Smoodh, B Fizz)
**Last verified:** 2026-05-21
**Tier:** 1 (raw HTTP + regex parse of server-rendered HTML)
**Framework:** Custom static site (NOT WordPress)
**Protection:** none

## Context

Parle Agro Pvt Ltd is the privately-held parent that runs Frooti, Appy,
Appy Fizz, Bailley packaged water, Smoodh, and B Fizz. It is **distinct
from listed Parle Products** (Parle-G biscuits — the families split
decades ago). Like VBL the business is **B2B-dominant** — there is no
consumer store locator. Unlike VBL the site is NOT WordPress so the
`wp-json/wp/v2/pages` enumeration trick does not apply.

The full beverage-manufacturing footprint **is published directly on the
public `/contact-us` page** under a tabbed state-keyed list under the
heading `Beverage manufacturing facilities`. 15 plants: 10 company-owned
(PAPL) and 5 franchise, spanning 12 Indian states + Nepal.

## URL patterns

| Page | Pattern | Example |
|---|---|---|
| Contact / plants | `/contact-us` | https://www.parleagro.com/contact-us |
| Sitemap | (none — 404) | |

The page hosts THREE address sections under `.contact-details`:

1. **Sales offices** — 15 city-keyed sales offices with phone/email
2. **Beverage manufacturing facilities** — state-keyed list of plants (this is what we extract)
3. **Legacy city-keyed copy of the same plants** — same `<div class="tab-content">` wrapper, ids like `Mysore`, `Patalganga`, `Khurda`, `Ambala1`, `Ambala2`, etc. (filtered out by the allowlist)

## Hydration payload

None — server-rendered HTML. Each plant is one `<li class="location">`:

```html
<div class="tab-pane" id="Maharashtra">
  <ul>
    <li class="location">
      <p>Parle Agro Pvt Ltd<br>Village Vanivali, (Patalganga) Taluka Khalapur, District Raigad, Maharashtra - 410220</p>
      <p>(PAPL)</p>
    </li>
  </ul>
</div>
```

The `<br>` separates the operator (line 1) from the address (line 2).
The trailing `<p>(PAPL)</p>` or `<p>(Franchise)</p>` flags ownership.
Nepal ships no ownership tag — we infer Franchise from the non-Parle
operator.

## Extraction strategy

Tier 1. One fetch. ~50KB HTML. Anchor on the
`<h5>Beverage manufacturing facilities</h5>` heading, capture the
adjacent `<div class="tab-content">` wrapper, then iterate
`<div class="tab-pane" id="<StateKey>">`. Filter by an explicit
state-key allowlist (`Maharashtra`, `UttarPradesh`, `Madhyapradesh`,
`Tamilnadu`, etc.) to ignore the legacy city-keyed tabs that share the
same wrapper. First occurrence wins (handles a duplicate `id="Nepal"`
stub).

Records emit:

- `locationType: "warehouse"` (canonical schema lacks "bottling-plant")
- `storeType: "bottling-plant"` (free-text discriminator)
- `storeId: "<StateKey>:<seq>"` (positional — source ships no plant codes)
- `features: ["ownership:PAPL"]` or `["ownership:Franchise"]` + optional
  `["operator:<franchise-name>"]` for non-PAPL rows
- `_extra.stateKey`, `_extra.rawAddress`, `_extra.operator` preserved

## Field inventory

| Field | Source | Notes |
|---|---|---|
| brand | hard-coded "Parle Agro" | |
| storeId | `"<StateKey>:<seq>"` | Positional — no stable plant code in source |
| name | `"Parle Agro — <city> (<ownership>)"` | |
| address.line1 | second segment of `<p>` (post-`<br>`) | |
| address.city | per-state lookup with multi-plant disambiguation (Haryana → Ambala (Saha) / Ambala (Mohra); UP → Ghaziabad / Varanasi) | |
| address.state | from tab id via STATE_KEY_MAP | "Orissa" → "Odisha", "Tamilnadu" → "Tamil Nadu", "Madhyapradesh" → "Madhya Pradesh" |
| address.postalCode | 6-digit extract from address, handles "410220" and "410 220" forms | |
| address.country | "IN" (or "NP" for Nepal) | |
| features | `ownership:<PAPL\|Franchise>` + `operator:<name>` when franchise | |
| lat / lng / phone / email / hours / url | n/a | source doesn't ship these |

## Files in this repo

- **Extractor:** `scrapers/sales-distribution/parle-agro.ts`
- **Run script:** `scrapers/sales-distribution/parle-agro-run.ts` (writes JSONL + scorecard + snapshot + alerts)
- **Fixture:** `__tests__/fixtures/parle-agro-contact.html`
- **Tests:** `__tests__/parle-agro-stores.test.ts` (11 tests, 86 assertions)
- **Output:** `data/results/parle-agro-stores.jsonl` (15 records)
- **Snapshot:** `data/snapshots/fmcg/parle-agro/<ts>.jsonl`

## Pagination

None — single page, single section.

## Gotchas

- **Not WordPress.** The VBL `wp-json` enumeration trick does NOT apply
  here. Parle Agro runs a custom static site (no `/sitemap.xml`, no
  `/wp-json/*`). Recon would have wasted time probing those if not
  flagged up front.
- **Duplicate tab-panes in one tab-content.** The "Beverage manufacturing
  facilities" tab-content holds BOTH the canonical state-keyed tabs
  ("Maharashtra", "Karnataka", ...) AND a legacy city-keyed copy
  ("Mysore", "Patalganga", ...) of the same plants. Filter by a fixed
  state-key allowlist; do not try to "take the first tab-content block
  only" because they share a wrapper.
- **Second `id="Nepal"` stub.** The legacy copy includes a placeholder
  `<li><p>NA</p></li>`. First-occurrence-wins on `stateKey` drops it;
  the parser also independently filters `<li>` bodies equal to "NA".
- **"Haryana1" and "UttarPradesh1" tabs are commented out** of the
  visible dropdown (`<!--<option value="Haryana1">...-->`) but their
  `<div class="tab-pane">` markup is still in the DOM and duplicates a
  row from the visible tab. Explicit blocklist.
- **Nepal row has no ownership tag.** The Nepal entry (`South Asian
  Beverages Ltd`) ships email/phone where the `(Franchise)` tag normally
  sits. Inferred from operator-not-Parle-Agro.
- **Multi-plant states need disambiguation.** Haryana has Ambala (Mohra)
  + Ambala (Saha). Uttar Pradesh has Ghaziabad + Varanasi. Disambiguated
  by content matching in the address line, not by tab id.
- **Address has both "410220" and "410 220" (spaced) PIN forms.** The
  pincode regex tries the contiguous form first, falls back to a "NNN
  NNN" form with an internal space.

## What I tried that didn't work

- `/our-business`, `/business`, `/company`, `/plants`, `/factories`,
  `/our-plants`, `/manufacturing` — all 404.
- `/sitemap.xml` — 404.
- `/wp-json/wp/v2/pages` — N/A (site is not WordPress; the path 404s).
- `/our-business` (the VBL playbook's first probe) — 404 on Parle Agro.

The eventual hit was the navigation crawl that pointed at `/contact-us`
— the page name belied its content (it is effectively the corporate
locations page, not just a contact form).

## Scorecard (2026-05-21)

- **Overall:** 73 (medium)
- **Completeness:** 87 (15/15 vs source self-claim, full sibling probe N/A — Parle Agro is the parent, not a sibling)
- **Data quality:** 40 (no lat/lng, no phone/hours — expected for plant addresses)
- **Source authenticity:** 91 (parleagro.com matches official domain, known brand, Wikipedia article)
- **Freshness:** 75

## Lesson

> Treat `/contact-us` as a first-class probe for FMCG B2B brands. The
> name suggests "form only" but in practice many privately-held FMCG
> parents (Parle Agro, similar Indian conglomerates) put their full
> manufacturing footprint there because that is the page their
> compliance / consumer-affairs / FSSAI audit teams point regulators
> at. Always grep the raw HTML for `<li class="location">` (or similar
> address-list markup) before declaring a contact page "form only".
