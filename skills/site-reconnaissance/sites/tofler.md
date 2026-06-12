# Tofler.in

**Domain:** tofler.in
**Vertical:** Indian company filings / MCA data (registration, directors, financials, charges)
**Last verified:** 2026-05-22
**Tier:** 1 (SSR HTML — direct fetch + cheerio, no browser needed)
**Framework:** Custom server-rendered HTML (no React/Next.js hydration payloads)
**Protection:** None — plain Chrome UA + Accept-Language passes. CloudFront CDN for static assets only.

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| Company list (first page) | `/companylist/{state}` | `https://www.tofler.in/companylist/delhi` |
| Company list (paginated) | `/companylist/{state}/pg-{N}` | `https://www.tofler.in/companylist/delhi/pg-2` |
| Company list (filtered by industry) | `/companylist/{state}/{industry-slug}/pg-{N}` | `https://www.tofler.in/companylist/delhi/business-services/pg-155` |
| Company detail | `/{company-slug}/company/{CIN}` | `https://www.tofler.in/nestle-india-limited/company/L15202DL1959PLC003786` |
| Director profile | `/{director-slug}/director/{DIN}` | `https://www.tofler.in/prathivadibhayankara-rajagopalan-ramesh/director/01915274` |
| Browse all | `/browsecompanies` | Alphabetical index |

## Pagination

- **First page:** `/companylist/{state}` (no pg suffix) — this is NOT `pg-0`. `pg-0` is empty/invalid.
- **Second page:** `pg-2` — note there's no `pg-1`.
- **Subsequent pages:** `pg-3`, `pg-4`, ... sequential.
- **30 companies per page.**
- **Termination:** Empty table body (no `<tr>` inside `#table-body`). Pages go up to ~9,000-9,500 for Delhi.
- **Total Delhi companies:** ~270K-285K.
- **Navigation:** `<a id="next-btn" href="...">Next</a>` and `<a id="prev-btn">Back</a>`.

## Hydration payload

None. This is a server-rendered HTML page with no client-side hydration. Data is embedded directly in the HTML DOM.

One exception: **charges data** is in a JS variable:
```javascript
const chargesTableData = [{"charge_amount": "390.0  cr", "charge_date": "28 Mar, 2017", "charge_holder": "CitiBank N.A."}];
```

## Extraction strategy

**Two-phase crawl:**

1. **List phase** (`tofler-companies`): Crawl paginated list pages, extract company name + CIN + slug + year + industry + status. Output: JSONL with detail URLs.

2. **Detail phase** (`tofler-company-detail`): Read list JSONL, fetch each detail page, extract full company record. Concurrent workers (4) with jitter pacing.

**Why Tier 1:** All data is in the SSR HTML. No JS execution needed. No protection (no Akamai/Cloudflare/Imperva challenges). A plain Chrome UA with `Accept-Language: en-IN` gets full responses.

**Pacing:** 1.5-4s jitter per list page, 2-5s per detail page. Full Delhi crawl takes ~8-13 hours (list) + detail enrichment time depends on how many companies you need.

## Field inventory

### List page fields

| Field | Source | Notes |
|---|---|---|
| name | `#table-body tr td:nth(0) a` text | Uppercase |
| cin | `#table-body tr td:nth(0) a` href tail | CIN from URL |
| slug | `#table-body tr td:nth(0) a` href path | URL slug |
| incorporationYear | `#table-body tr td:nth(1)` text | 4-digit year |
| industry | `#table-body tr td:nth(2)` text | Tofler-inferred, may be wrong for diversified companies |
| status | `#table-body tr td:nth(3)` text | Active, Inactive, Strike Off, etc. |

### Detail page fields

| Field | Section | Source | Notes |
|---|---|---|---|
| CIN | Registered Details | `h3:CIN` sibling `span.font-semibold` | Primary key |
| PAN | Registered Details | `h3:PAN` sibling `span.font-semibold` | May be "-" if not filed |
| incorporationYear | Registered Details | `h3:Incorporation` sibling | "1959, 67.1 years" format |
| incorporationDate | FAQ accordion | `"What is the incorporation date..."` answer | "28 March, 1959" |
| status | `.success.font-semibold` or `.error.font-semibold` | Badge text |
| companyType | Registered Details | `.badge` elements | ["Public", "Non-government Company"] |
| email | Registered Details | `h3:Company Email` sibling | Obfuscated: `[at]` and `[dot]` |
| authorizedCapital | Registered Details | `h3:Authorised Capital` sibling | "₹ 200.0 Cr" |
| paidUpCapital | Registered Details | `h3:Paid up Capital` sibling | "₹ 192.8 Cr" |
| revenueRange | `.stats_highlights` | `.key_metrics_wrapper:Revenue` sibling value | "₹ >5000 cr" or "Not Available" |
| lastAgm | Registered Details | `h3:AGM` sibling | "Jun 2025" |
| industry | `.company_tag_item` | Badge text(s) | Array of classifications |
| products | `.stats_left ul li` | List items | Products/services description |
| registeredAddress | FAQ accordion | `"What is the registered address..."` answer | Full address string |
| description | `.company_description` | First paragraph text | Overview paragraph |
| directors | `#people-module tbody tr` | `td[0]`: designation, `td[1] a`: name + profile URL, `td[2]`: DIN, `td[3]`: tenure | Array of directors |
| charges | `chargesTableData` JS var | JSON array of `{charge_amount, charge_date, charge_holder}` | Array of charges |

### Paywalled fields (NOT extracted)

| Field | Notes |
|---|---|
| Exact revenue/EBITDA/Net profit | Behind "GET PRO" blur |
| Balance sheet items | Behind paywall |
| Ratio analysis | Behind paywall |
| Profit & Loss statement | Behind paywall |
| Location details table | Loaded via client-side JS (empty tbody in SSR) |

## Files in this repo

- **Extractor:** `scrapers/tofler/extract.ts` (pure functions: `parseListPage`, `parseDetailPage`)
- **List scraper:** `scrapers/tofler-companies.ts` (paginated list crawl)
- **Detail scraper:** `scrapers/tofler-company-detail.ts` (detail enrichment)
- **Fixture (list):** `__tests__/fixtures/tofler-list-delhi.html`
- **Fixture (detail):** `__tests__/fixtures/tofler-detail-nestle.html`
- **Tests:** `__tests__/tofler.test.ts` (34 tests)

## Gotchas

1. **`pg-0` is empty.** The first page is `/companylist/{state}` (bare URL). `pg-0` returns a valid page shell but with no table rows. `pg-1` also doesn't exist. Pagination starts at `pg-2` for the second page.

2. **Email obfuscation.** Tofler replaces `@` with `[at]` and `.` with `[dot]` in email fields. Reverse with a simple regex: `email.replace(/\[at\]/g, '@').replace(/\[dot\]/g, '.')`.

3. **Industry classification is inferred.** Tofler warns: "If the company has changed line of business without intimating the Registrar or is a diversified business, classification may be incorrect." Don't trust it blindly.

4. **PAN may be "-".** Some companies have PAN as a dash string rather than null.

5. **Charges are in a JS variable**, not in a `<table>`. Parse `const chargesTableData = [...]` from a `<script>` block. If the company has no charges, the variable may be absent or an empty array.

6. **Revenue is a range, not a number.** Free tier shows ranges like "₹ >5000 cr", "₹ 1 cr - 100 cr", or "Not Available". Exact numbers are paywalled.

7. **Location table is client-rendered.** The `#locations-table-body` element is empty in SSR HTML — it's populated by JS. We don't extract locations (use the registered address from FAQ instead).

8. **Director DIN may be hidden.** KMP (Key Managerial Personnel) entries sometimes show `<HIDDEN>` instead of a DIN, with no profile link.

9. **Scale is massive.** Delhi alone has ~270K-285K companies across ~9,000+ pages. A full crawl takes 8-13 hours for the list phase. Plan for resumability via checkpointing.

10. **Company name casing.** Tofler renders company names in all-caps (`<h1>` and table cells). The `name` field preserves the original casing from the HTML.

## Zauba Corp comparison

Zauba Corp (`zaubacorp.com`) was also evaluated but returns **403 on all list pages** (even with Chrome UA). It appears to have WAF protection that blocks automated access. Tofler was chosen because:
- SSR HTML accessible with plain HTTP
- No protection signals
- Rich free-tier data (CIN, PAN, directors, address, capital, charges)
- Clean pagination structure
- Same MCA data source as Zauba Corp
