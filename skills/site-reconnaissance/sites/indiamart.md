# IndiaMART

**Domain:** dir.indiamart.com (also indiamart.com)
**Vertical:** India B2B catalog — warehouses, services, products, manufacturers
**Last verified:** 2026-05-19
**Tier:** 3 (Playwright) but Tier 1 likely workable
**Framework:** Next.js — `<script id="__NEXT_DATA__">` present + `window.__INITIAL_STATE__`
**Protection:** mild

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| Index (catalog) | `/impcat/{slug}.html` | `/impcat/warehouse-on-rent.html` |
| Detail (product/service) | `/proddetail/...-NNNNNNNNNN.html` (10-digit numeric ID at the end) | `/proddetail/warehouse-rental-services-1234567890.html` |
| Seller / company | `/{seller-slug}-NNNN.html` | (similar) |

## Hydration payload

- **Two payloads on detail page:**
  - **`<script id="__NEXT_DATA__">`** — Next.js convention. **194 leaves on detail page.**
  - **`<script type="application/ld+json">`** × 2 — `BreadcrumbList` and `ImageGallery`
- **Source names from `extractAllHydrationPayloads`:** `<script id="__NEXT_DATA__">`, `ld+json[0]`, `ld+json[1]`
- **Schema:** root → `props.pageProps.serviceRes.Data[0]`
  - `Data[0].Basic_Information[]` — 19 entries, each `{TITLE, DATA}` — the canonical spec table (Nature of Business, Additional Business, ...)
  - `Data[0].FOB_PRICE`, `Data[0].BIZ`, `Data[0].ADDRESS`, `Data[0].BRD_CAT_NAME`, `Data[0].BRD_MCAT_NAME`
  - `Data[0].0[i].IMAGE_{125x125|250x250|500X500|1000x1000|ORIGINAL}` — product photos at multiple sizes

## Extraction strategy

Tier 3 likely overkill — IndiaMART HTML is mostly SSR'd. Could try Tier 1 with proper Chrome headers; if 403s appear, escalate. The recon used Tier 3 to be safe.

## Files in this repo

- **Recon demo:** `scrapers/recon-indiamart.ts`
- **Extractor:** ❌ not built (recon-only round)
- **Fixture:** ❌ not captured
- **Tests:** ❌ none

## Pagination

Not explored in detail. Catalog pages have a `?page=N` pattern; index pages with many sellers typically infinite-scroll OR paginate.

## Gotchas

- **Spec data is in TWO places** — `Basic_Information[]` array in `__NEXT_DATA__` (most reliable) and `<table><tr><th>...</th><td>...</td></tr>` rows in the DOM (extracted by `visualAudit.tableRows`). They overlap; prefer the JSON.
- **`<dl>` blocks carry company metadata** (GST date, legal status, annual turnover, IndiaMART member since). Different from real-estate sites that put dealer info in payload.
- **Index card classes:** `.cardlinks` (20 hits), `[class*='product']` (20 hits). Not `factsRow`/`tupleNew`.
- **Detail URLs end in `-NNNNNNNNNN.html`** (10-digit numeric ID). The `/proddetail/` pattern was added to the default detail-link regex.
- **JSON-LD is per-page-type** — only `BreadcrumbList` + `ImageGallery` on detail; SRP and catalog pages may have different LD blocks.
- **Phone numbers visible** (less aggressive masking than real-estate sites): some `IndiaMART Contact Number` strings appear in the page. Validate before extracting.

## Recon output (2026-05-19)

- Index: 1057 schema leaves (Next.js `props.pageProps`)
- Detail: 194 schema leaves
- Detail label-value pairs: 10 (from `<table>` rows, e.g. `Product / Goods Details: Warehouse`, `Service Location: ...`)

## What's missing for production

- Full schema mapping of `Basic_Information[]` (19 entries vary by service type)
- Per-listing extractor + fixture + tests
- Pagination strategy across category indexes
- Confirm that Tier 1 (raw HTTP) is viable for sustained scraping
