# RAI — Retailers Association of India (Members Directory)

**Domain:** rai.net.in
**Vertical:** India retail industry association — member directory
**Last verified:** 2026-05-22
**Tier:** 1 (plain HTTP + cheerio)
**Framework:** PHP (PHPSESSID cookie, `.php` URLs, nginx/1.23.4)
**Protection:** Imperva (x-cdn: Imperva, `visid_incap_*` + `incap_ses_*` cookies) — present but NOT blocking raw HTTP with Chrome UA

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| Members list (default) | `/our-members.php` | https://rai.net.in/our-members.php |
| Members list (paginated) | `/our-members.php?page={N}` | https://rai.net.in/our-members.php?page=50 |
| Members list (search) | `/our-members.php?company_name={text}` | https://rai.net.in/our-members.php?company_name=reliance |

## Data model

Single page type only — a flat HTML table. No detail pages, no JSON API, no hydration payloads.

| Field | Source | Notes |
|---|---|---|
| companyName | `<tr class=item> td:eq(0)` (colspan=2) | Text, trimmed |
| membershipCategory | `<tr class=item> td:eq(1)` | Core, Associate, Real Estate, Affiliate Association, Founder, Academic |
| city | `<tr class=item> td:eq(2)` | Sometimes trailing comma; mixed case (`lucknow` vs `Lucknow`) |

## Extraction strategy

Tier 1: plain `fetch()` + cheerio. All data is in the SSR'd HTML table — no JavaScript required.

- 20 rows per page
- Pages 1–107 as of 2026-05-22 (page 108 is empty)
- **2,127 total members**
- Pagination via `<ul class="link-btn">` with `?page=N` links
- The `?page=0` link in the pagination arrows is equivalent to page 1
- Search form submits `?company_name=` — server-side filtering, still paginated

Pacing: 500–1000ms between requests. Total crawl time ~90 seconds.

Imperva cookies are set on first response but no challenge/block observed. Standard Chrome UA headers suffice.

## Pagination

- Query param: `?page=N` (1-indexed)
- 20 rows per page
- Last page with data: page 107 (7 rows)
- Page 108+ returns empty table
- Terminator: 2 consecutive empty pages

## Membership categories (as of 2026-05-22)

| Category | Count | Share |
|---|---|---|
| Core | 1,851 | 87.0% |
| Associate | 185 | 8.7% |
| Real Estate | 35 | 1.6% |
| Affiliate Association | 30 | 1.4% |
| Founder | 14 | 0.7% |
| Academic | 12 | 0.6% |

## Top cities (as of 2026-05-22)

Mumbai (379), Chennai (267), Kolkata (195), Bengaluru (174), Hyderabad (129), Coimbatore (86), New Delhi (78), Delhi (77), Gurgaon (54), Kochi (34). Total 232 unique cities.

## Files in this repo

- **Extractor:** `scrapers/rai-members.ts` (production), `scrapers/rai-members-test.ts` (smoke test — page 1 only)
- **Fixtures:** `__tests__/fixtures/rai-members-page1.html`, `__tests__/fixtures/rai-members-page107.html`
- **Tests:** `__tests__/rai-members.test.ts` (19 tests, 131 assertions)
- **Runner:** `scripts/run-rai-members.ts`
- **Output:** `data/results/rai-members.jsonl` (2,127 records)

## Gotchas

- **Page 0 = Page 1.** The left-arrow pagination link points to `?page=0`, which returns the same content as page 1 / default. The scraper starts from page 1 and ignores page 0.
- **City data quality is inconsistent.** Some entries have trailing commas (`Nathdwara, Rajsamand,`), some are lowercase (`lucknow`), some have `New Delhi` vs `Delhi` as separate values. The extractor trims trailing commas but does NOT normalise case or merge Delhi/New Delhi — that's a consumer concern.
- **Imperva cookies are passive.** Despite `x-cdn: Imperva` and `visid_incap_*` cookies, no challenge page or 403 was observed across 108 sequential requests. The WAF may escalate under higher volume — untested above ~2 req/sec.
- **No detail pages.** The members list is the ONLY data surface. There are no per-company detail pages, no `/member/{id}` URL, no company logos or contact info. What you see in the table is all that exists.
- **Category dropdown is empty.** The search form has a commented-out category `<select>` with no `<option>` values populated. Server-side category filtering via the form is non-functional. Only `company_name` text search works.
- **Last member is "Reliance Retail Ltd." (Founder, Mumbai)** on page 107, row 6 — alphabetical ordering appears to be case-insensitive but with some anomalies.
