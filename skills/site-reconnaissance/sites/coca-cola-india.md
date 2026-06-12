# Coca-Cola India

**Domain:** coca-cola.com/in (also coca-colaindia.com, which serves the same site)
**Vertical:** FMCG / beverages / brand-owner corporate site
**Last verified:** 2026-05-21
**Status:** parked
**Tier:** n/a (no locator exists)
**Framework:** Adobe Experience Manager — "OneXP" (One Experience Platform), served via CloudFront (server header: `CEP`)
**Protection:** none observed (200 OK on default Chrome UA)

## Context

Coca-Cola India is **the brand owner** — The Coca-Cola Company's India
operating entity. The Indian bottling network is owned by separate
entities: **HCCB** (Hindustan Coca-Cola Beverages — Coke-owned, ~16 plants)
and franchisees like Moon Beverages, SLMG, Kandhari, etc. The brand-owner
site at coca-cola.com/in does not publish a locator, a plant list, a
distributor map, or any structured geographic data.

This is the same pattern observed in the wider beverage-brand-owner
category: global FMCG parents push location data down to the bottling /
franchise tier and keep the brand site focused on marketing + product
storytelling. Compare to:

- **Varun Beverages (PepsiCo bottler)** — publishes plants list at
  `/plants/` because VBL is itself a listed company with FSSAI-driven
  disclosure obligations. The brand-owner PepsiCo India equivalent would
  almost certainly be parked for the same reason.
- **HCCB** (Coke's bottler) — likely has a structured plant list and
  should be a separate brand slug.

## What was tried

1. **HEAD/GET on candidate paths** (all returned 404 unless noted):
   - `/in/en` — 200 (homepage)
   - `/in/en/about-us` — 200
   - `/in/en/brands` — 200
   - `/in/en/about-us/contact-us` — 200 (consumer-feedback AEM form, no locator)
   - `/in/en/our-plants` — 404
   - `/in/en/plants` — 404
   - `/in/en/find-us` — 404
   - `/in/en/store-locator` — 404
   - `/in/en/where-to-buy` — 404
   - `/in/en/locator` — 404
   - `/in/en/manufacturing` — 404
   - `/in/en/our-business` — 404
   - `/in/en/our-company` — 404
   - `/in/en/our-india-operations` — 404

2. **Sitemap probes** (all 404):
   - `/sitemap.xml`
   - `/in/sitemap.xml`
   - `/in/en/sitemap-index.xml`
   - `/in/en/query-index.json` (Helix/Franklin convention)
   - `/in/query-index.json`

3. **Human-readable sitemap** at `/in/en/sitemap` returned 24 internal links.
   The full inventory is:
   - `/in/en` (home)
   - `/in/en/about-us`, `/in/en/about-us/contact-us`, `/in/en/about-us/faq`,
     `/in/en/about-us/history`, `/in/en/about-us/leadership`
   - `/in/en/brands` + 12 brand sub-pages (coca-cola, thums-up, sprite,
     fanta, limca, maaza, kinley, minute-maid, schweppes, smartwater,
     georgia, costa-coffee, charged-by-thums-up, rimzim)
   - `/in/en/sustainability`, `/in/en/water-stewardship`,
     `/in/en/fruit-circular-economy`, `/in/en/world-without-waste`,
     `/in/en/value-chain-transformation`, `/in/en/responsible-business`
   - `/in/en/media-center`, `/in/en/media-center/limca-book-of-records`
   - `/in/en/legal/*` (privacy, ToS, supplier T&Cs, financial policy docs)

   **No locator, plant list, factory list, distributor list, or "where to
   buy" page.** The contact-us page is a JCR-defined AEM Forms widget
   (`/content/forms/af/onexp/in/en/contact-us`) with name/email/phone/
   address/city/state fields — these are FORM INPUTS for consumer feedback,
   not data about Coca-Cola facilities.

4. **Alt-domain check**:
   - `cocacolaindia.com`, `www.cocacolaindia.com` → DNS-resolves to nothing useful (000)
   - `coca-colaindia.com` → 200, but title is "Home | Coca-Cola" and
     content fingerprints identical to coca-cola.com/in/en (Sprite WhatsApp
     CTA, same OneXP assets). Same site under an alias domain.

5. **VBL playbook check (wp-json enumeration)** — **deliberately skipped**.
   The site is AEM, not WordPress. The OneXP resource paths
   (`/content/onexp/in/en/...`) and the `helix-rum` instrumentation tag
   confirm AEM. WordPress REST conventions don't apply.

6. **HCCB**? — **explicitly out of scope.** HCCB is the bottler, a sister
   brand with its own slug. Keeping brand boundaries clean was an explicit
   recon-brief constraint.

## Why parked

Global beverage brand-owner site with no public locator. The full sitemap
is 24 corporate/marketing/legal pages and no path returns a structured
list of plants, distributors, retail outlets, or partner locations. This
is the canonical "PARKED" outcome from the recon brief.

## Manifest signal

**Global beverage brand-owners (and most global-FMCG brand owners) have
no public locator on their consumer-facing corporate site.** Location
data is pushed down to the bottling / franchise / distributor tier (HCCB,
VBL, Moon Beverages, SLMG, etc.). When the brand has a separately listed
bottler with FSSAI disclosure obligations (VBL), that bottler publishes
plants. When the bottler is private (HCCB), the data is in annual reports
and press releases, not on a website.

**Recon implication:** for `industry: fmcg, subcategory: beverages`
entries where the brand IS the brand-owner (not the bottler), set
`status: parked` immediately unless the brand has an unusual D2C presence
(e.g. branded Coca-Cola museum gift shops, which don't exist in India).
The actionable signal is at the bottler / franchisee tier.

## Files in this repo

- **Extractor:** none (parked)
- **Fixture:** none (parked)
- **Tests:** none (parked)
- **YAML:** `targets/industries/fmcg/coca-cola-india.yaml` (status: parked)

## If we ever un-park

The realistic path to coverage is the **bottler tier**:

1. Build extractors for HCCB (likely a separate brand slug with its own
   YAML — confirm before starting). HCCB is private and may have only a
   sparse static site.
2. Mine annual reports / press releases / FSSAI Form-B registry for the
   plant footprint. This is **not** a "scraper-engine" outcome — it's an
   intelligence outcome.
3. The "Coca-Cola India" slug itself stays parked.
