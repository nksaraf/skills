# Bisleri (parked)

**Domain:** bisleri.com
**Vertical:** FMCG / packaged water (India)
**Last verified:** 2026-05-21
**Status:** PARKED — no public locator of any kind
**Framework:** Salesforce Commerce Cloud (Demandware) — D2C e-commerce storefront

## Outcome

Bisleri's public domain is a **direct-to-consumer e-commerce storefront**
("Bisleri@Doorstep"), not a corporate / locator site. Plant list, distributor
finder, café outlets — none exist on `bisleri.com`. Setting `status: parked`
with a documented reason; re-check on any corporate-site redesign.

## What we tried

1. **Homepage scrape** (`https://www.bisleri.com/`, 200, 126 KB) — surfaced
   `/book-plant-visit`, `/contact-us`, `/our-journey`, `/leadershipTeam.html`,
   `/90-quality-tests.html`, `/what-makes-us-stand-apart.html`, brand pages
   (`/bisleri-water-bottles`, `/vedica-himalayan-spring-water`, `/pop.html`,
   `/limonataFooterPage.html`). No nav entry for plants, distributors, or
   outlets. Site is Demandware (`/on/demandware.static/Sites-Bis-Site/...`),
   not WordPress, so VBL's `wp-json/wp/v2/pages` enumeration is N/A.

2. **`/book-plant-visit`** (200) — appointment-booking form; the only
   structured data on the page is a `<select>` of **5 booking cities**:
   Mumbai, Bangalore, Delhi, Chennai, Hyderabad. These are the metros where
   Bisleri runs guided tours; they are **not** the production-plant list.
   No table, no list, no embedded JSON, no per-plant pages.

3. **`/contact-us`** (200) — corporate HO address + `1800 121 1007` + email
   `wecare@bisleri.co.in`. Single record; not a multi-location list.

4. **`/our-journey`** (200) — brand timeline (1949–2021); the embedded
   `img_data` JSON is a year-by-year milestones array (e.g. "2018: world's
   first vertical manufacturing plant"). Mentions plants only narratively —
   no count, no addresses.

5. **`sitemap.xml`** → 404 (robots.txt advertises it as
   `Sitemap: https://www.bisleri.com/sitemap.xml`, but the file is missing).
   `sitemap_index.xml` → 404 also.

6. **`wp-json/wp/v2/pages?per_page=100`** → 404 (site is not WordPress).

7. **Probed paths** — all 404 except where noted:
   - `/plants`, `/our-plants`, `/factories`, `/plant-locations` → 404
   - `/distributors`, `/distributor-locator`, `/find-a-distributor`,
     `/dealer-locator` → 404
   - `/store-locator`, `/storelocator`, `/where-to-buy`, `/find-store` → 404
   - `/cafe-bisleri`, `/bisleri-drink-station` → 404
   - `/about-us` → **500** (Demandware misconfig; `/our-journey` is the
     canonical About page)

8. **Demandware locator pipelines** —
   `/on/demandware.store/Sites-Bis-Site/default/Locator-Show` returns 500.
   The `LocationSelector-*` endpoints (`SetCityLocation`,
   `AddGlobalAddress`, etc.) are **delivery-zone serviceability** for the
   storefront cart — they govern whether Bisleri@Doorstep will ship to a
   pincode. They are not a list of retail outlets and do not expose one.

9. **robots.txt** — only blocks cart / checkout / dashboard / search-detail
   paths. No clue toward a locator endpoint.

## Why this is genuinely B2B-invisible (and unlike VBL)

VBL is an SEBI-listed bottler whose investor-relations website lists every
plant by name and location. Bisleri is **privately held by the Chauhan
family** — no SEBI filings, no investor-relations site, and the consumer
domain is run by a separate e-commerce stack (Demandware) with no
corporate-locator subsection. The plants + Bisleri@Doorstep delivery hubs
+ general-trade distributors exist offline only.

## Re-check triggers

Park this one and revisit when:

- Corporate-site redesign adds an investor-relations or sustainability
  section (would likely surface plant counts to support ESG narratives).
- News reports a Bisleri IPO (rumoured periodically) — DRHP would contain
  the full plant inventory.
- Secondary-source path: scrape **FSSAI's packaged-water licensee
  registry** by brand name "Bisleri" to derive plant addresses
  authoritatively. Out of scope for the bisleri.com recon track.

## Files in this repo

- **YAML:** `targets/industries/fmcg/bisleri.yaml` (`status: parked`)
- **Addendum:** this file (`sites/bisleri.md`)
- **No extractor, no fixture, no tests, no JSONL** — parked before any
  extraction work.
