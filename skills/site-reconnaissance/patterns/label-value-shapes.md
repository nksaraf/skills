# Pattern: Label/value DOM shapes

The single most-varying part of HTML extraction. Six conventions in active use across the six sites we've reconned. The `auditPage` helper in `@vinxi/scraper/recon` (`visualAudit`) runs an extractor for each.

## The six shapes

### 1. Colon-text (in a single text node)

```html
<span>Electricity &amp; Water Charges : Charges not included</span>
```

- **Extractor:** `colonPairs`
- **Sites:** 99acres (detail "facts" rows), CommonFloor (SRP card lines)
- **Recognise:** any `<li>`/`<p>`/`<span>`/`<div>` whose text matches `^([^:]{1,40})\s*:\s*(.{1,150})$`

### 2. Table rows (`<table><tr><th>label</th><td>value</td></tr>`)

```html
<table>
  <tr><th>Product / Goods Details</th><td>Warehouse</td></tr>
  <tr><th>Service Location</th><td>Pan India</td></tr>
</table>
```

- **Extractor:** `tableRows`
- **Sites:** **IndiaMART** (canonical use — B2B spec tables)
- **Recognise:** any `<tr>` with ≥2 `<th>`/`<td>` cells

### 3. Definition lists (`<dl><dt>term</dt><dd>def</dd></dl>`)

```html
<dl>
  <dt>GST Registration Date</dt>
  <dd>15-Jan-2018</dd>
</dl>
```

- **Extractor:** `definitionList`
- **Sites:** IndiaMART (company metadata: GST, legal status, annual turnover)
- **Recognise:** any `<dl>` with paired `<dt>`/`<dd>` children

### 4. Sibling spans with class hints

```html
<div>
  <span class="mb-srp__card__label">Total Units</span>
  <span class="mb-srp__card__value">140</span>
</div>
```

- **Extractor:** `siblingPairs`
- **Sites:** MagicBricks (canonical), 99acres (some sections)
- **Recognise:** any element with class matching `/label|key|heading|caption/i` whose next sibling carries the value
- **Limitation:** dies on CSS-in-JS sites (Housing.com) where class names are hashed

### 5. Stacked layout (label-above-value)

```html
<div class="some-flex-cell">
  <div>Carpet Area</div>
  <div>630 sq ft</div>
</div>
```

- **Extractor:** `stackedPairs`
- **Sites:** CommonFloor (canonical — Features grid), some MagicBricks cards
- **Recognise:** parent with exactly two text-bearing children, first ≤40 chars, the two children render on different visual lines (`getBoundingClientRect().top` differs by more than half the smaller height)
- **Strength:** works without class names — purely structural

### 6. `data-*` attribute hints

```html
<div data-aut-id="itemPrice">₹ 36,000</div>
<div data-aut-id="item-location">Kerala</div>
<div data-testid="property-card-price">₹3.47 Cr</div>
```

- **Extractor:** `attrPairs`
- **Sites:** **OLX** (canonical — `data-aut-id`), Housing (`data-q`), modern React-stack sites generally
- **Recognise:** any element with `data-testid`, `data-aut-id`, `data-qa`, `data-q`, `data-test`, or `itemprop`
- **Strength:** works on CSS-in-JS sites where class names are useless; conventions from Cypress / Playwright / React Testing Library are universal

## Reliability ranking

When the same field shows up in multiple shapes, prefer in this order:

1. **`attrPairs`** (data-testid is a test contract; rarely changes silently)
2. **`tableRows`** (semantic HTML; rendered explicitly)
3. **`definitionList`** (semantic HTML; rare but explicit)
4. **`stackedPairs`** (structural; no class deps; lower noise than sibling-class)
5. **`siblingPairs`** (class-name based; fragile on CSS-in-JS sites)
6. **`colonPairs`** (text regex; lots of false positives on long text bodies)

`labelValuePairs` in the audit result merges all six in this order. The individual fields remain available for provenance-aware code.

## Common false-positive sources

These trip the extractors and need filtering. The current `visualAudit.ts` handles the first three; the last is a known limit.

| Source | Symptom | Filter |
|---|---|---|
| Sidebar navigation lists | "Commercial property in X: Commercial property in Y" pairs | `isInNav()` excludes elements inside nav/aside/footer/header/breadcrumb/sidebar |
| List-of-similar-items | "Sq yard to Cent: Sq yard to Marla" | `looksLikeListPair()` — 15+ char common prefix → drop |
| Phone numbers as "values" | "IndiaMART Contact Number: 91XXXXXXXXX" | `isNoiseValue()` — `/^\+?\d[\d\s()-]{8,}$/` |
| URLs as "values" | "Website: https://example.com/page" | `isNoiseValue()` — `/^https?:/` |
| Multi-fact concatenations | "Parking NoListed by KavitaProperty on Ground..." | Not currently filtered — happens when stacked-pair parent has more than two text children |

## Decision flow for new sites

1. Run `auditPage` and inspect each of the six extractors' output independently
2. If `tableRows` or `definitionList` is non-empty, the site probably uses semantic HTML — extract from those
3. If `attrPairs` is non-empty, the site uses test-id conventions — extract from those (most stable)
4. If only `stackedPairs` + `colonPairs` produce output, the site is older / plainer HTML — DOM extraction is fine but write a per-site extractor (don't rely on the merged `labelValuePairs`)
5. If ALL six extractors are empty but the page has data, the site is either XHR-only (see `hydration-payloads.md` Tier 4) or uses an exotic convention worth adding to this doc

## When to add a new shape

Add a shape here when:
- Two or more sites use the same convention not already covered
- The convention isn't reducible to one of the existing six

Recent candidates considered but rejected (single-site oddities, kept in the site addendum instead):

- 99acres `tags-and-chips__badgeParent` POI wrapper — filtered via `badgeText()` direct-text rule, doesn't deserve a top-level extractor
- IndiaMART `Basic_Information[]` array — already covered by the payload schema walk, not a DOM extraction
