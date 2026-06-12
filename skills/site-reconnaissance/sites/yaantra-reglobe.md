# Yaantra / ReGlobe

**Domain:** reglobe.in (post-acquisition) + yaantra.com (legacy)
**Vertical:** India recommerce (used phone buy-back + repair) — direct Cashify competitor by category
**Last verified:** 2026-05-22
**Tier:** n/a (no public locator; both domains decommissioned in different ways)
**Framework:** n/a (reglobe.in 301-funnels to a third-party site; yaantra.com is an ASP.NET / Microsoft-IIS legacy husk)
**Protection:** none — nothing to protect; reglobe.in serves no content of its own

## Top-line finding

Yaantra was a used-phone recommerce + repair platform; Reliance Retail Ventures acquired it in 2022 and rebranded as ReGlobe. The pre-acquisition store + repair-centre footprint was ~150 sites. **Both brand domains are presently non-functional as locators**:

1. **reglobe.in** — every URL (including `/stores`, `/sitemap.xml`, `/robots.txt`) returns:

   ```
   HTTP/2 301
   location: https://www.cashify.in/?__utmrg=____ref_AAE595960E56471C9F23A98616CEC37B&__utmh=reglobe.in
   ```

   i.e. the entire domain is a single-target 301 funnel pointing at **Cashify** with a stable affiliate / referral tag. **No content is served on reglobe.in itself.** Cashify is an independently-operated competitor (parent: Manak Waste Management Pvt Ltd) — not a Reliance backend share. This is a customer-acquisition arrangement, not a technical handoff.

2. **yaantra.com** — homepage renders ("Sell Your Old Phone & Get Instant Cash"). Every secondary URL — including the 57 `/repair/mobile-repair-service-in-{city}` URLs in `yaantra.com/sitemap.xml` (lastmod 2023-01-24) — returns a 130KB 404 stub canonicalised to `/pagenotfound`. The site is decommissioned in all but the home page.

Conclusion: **parked**. There is no public locator surface to extract from on either domain. Re-check quarterly to catch a Reliance re-platforming.

## Vendor comparison vs Cashify

Cashify is the right comparator — same vertical (used-phone recommerce + repair + buyback), same Indian market, similar pre-acquisition era.

| Aspect              | Cashify                              | Yaantra / ReGlobe                              |
|---------------------|--------------------------------------|------------------------------------------------|
| Parent              | Manak Waste Management Pvt Ltd       | Reliance Retail Ventures (acquired 2022)       |
| Brand domain(s)     | cashify.in                           | reglobe.in (redirect-only) + yaantra.com (husk)|
| Public locator      | `/offline-stores` (Next.js App Router)| **none**                                       |
| Framework           | Next.js 14 + RSC stream              | n/a (nginx 301 on reglobe; ASP.NET husk on yaantra)|
| Tier                | 1 (raw HTTP)                         | n/a                                            |
| Protection          | none observed                        | n/a (nothing reachable to protect)             |
| Stores extracted    | 239 (across 91 cities)               | 0                                              |
| Affiliate-funnel-in | n/a                                  | Receives 100% of reglobe.in/* traffic via 301   |

The funnel direction is striking: a Reliance-owned brand's web property hands every visitor over to an independent competitor. Most likely explanation is that Reliance has not (yet) re-platformed the ReGlobe consumer surface and has parked reglobe.in as an affiliate referrer pending integration into another Reliance retail property (Reliance Digital? Reliance Smart Bazaar? Flipkart `/reset-sell-store` — visible link on yaantra.com home).

## URL patterns

There are no working extractable patterns. For reference, here are the dead patterns probed:

| Page type | Pattern attempted | Result |
|---|---|---|
| Locator (YAML-claimed) | `https://reglobe.in/stores` | 301 to `https://www.cashify.in/` |
| Locator (alternate guesses) | `/locations`, `/find-a-store`, `/store-locator`, `/our-stores`, `/repair-centres`, `/dealers`, `/our-network` | all 301 to Cashify |
| Infrastructure | `/sitemap.xml`, `/robots.txt` | both 301 to Cashify (proving domain serves zero content) |
| Yaantra repair (legacy nav) | `https://yaantra.com/repair/mobile-phone-repair-service-center-pune` (and 56 sibling URLs in sitemap.xml) | all return 130368-byte 404 stub with `canonical=/pagenotfound` |
| Yaantra locator guesses | `/stores`, `/locations`, `/our-presence`, `/repair-centres`, `/branches`, `/dealers` | all 404 stub |

## Hydration payload

n/a — there is no payload to walk. The reglobe.in redirect is a `nginx` `301` with `content-length: 162` and a `text/html` body that is just the HTTP-spec stub. Yaantra's 404 page contains no payload of interest.

## Extraction strategy

**There is no extraction strategy.** The module ships a `extractYaantraReGlobeStores()` placeholder that THROWS — silent empty data is the worst failure mode and would pass downstream pipelines while producing zero records.

If Reliance later restores a locator:

1. Re-run a probe sweep on `reglobe.in` (and `yaantra.com` for paranoia).
2. If the 301 is gone and a real Next.js / Magento / custom page renders at `/stores`: capture HTML, run `extractAllHydrationPayloads`, build extractor.
3. **Do NOT reuse the Cashify extractor against reglobe.in.** Cashify is an unrelated company; double-counting its stores under the Yaantra brand would corrupt cross-brand totals.
4. The most likely shape (given Reliance Digital's existing online stack) is Magento / Adobe Commerce, NOT Next.js (which would be an oddly direct copy of Cashify).

## Field inventory

n/a (zero records).

## Files in this repo

- **Extractor stub:** `scrapers/retail/yaantra-reglobe.ts` — exports the parked-state constants + a throwing `extractYaantraReGlobeStores()` placeholder
- **Phase D runner:** `scripts/run-yaantra-reglobe-snapshot.ts` — scores the empty JSONL, writes snapshot, evaluates alerts
- **Tests:** `__tests__/yaantra-reglobe-stores.test.ts` (10 schema-locked assertions including locationType breakdown of the JSONL)
- **Output:** `data/results/yaantra-reglobe-stores.jsonl` (empty by design)
- **Scorecard:** `data/reports/yaantra-reglobe-scorecard.md`

## Pagination

n/a (no listing page reachable to paginate).

## locationType breakdown

| locationType | count | source |
|---|---|---|
| store | 0 | No public locator on either domain. |
| pickup-point | 0 | No public locator on either domain. |

## Gotchas

- **The 301 carries an affiliate tag** (`__utmrg=____ref_AAE595960E56471C9F23A98616CEC37B&__utmh=reglobe.in`). Proves the funnel is intentional, not accidental DNS / hosting bleed-through. If this tag ever disappears from the redirect, that's a strong signal Reliance is re-platforming the domain — re-recon.
- **Curl with default `Host` header on `www.reglobe.in` and `reglobe.in` returns identical 301 behaviour** — there is no www-vs-apex difference (the `https://www.reglobe.in/stores` URL in the YAML resolves identically to `https://reglobe.in/stores`).
- **Yaantra's sitemap.xml is a fossil** — 57 URLs from January 2023, all of which 404 today. Don't trust the sitemap alone; HEAD-check at least one URL before treating it as alive.
- **The Cashify extractor MUST NOT be reused for this brand.** Cashify is an unrelated company. The 301 from reglobe.in to cashify.in is a referral / affiliate funnel, not a backend share. Reusing Cashify's records under the Yaantra brand would double-count and corrupt cross-brand totals (this is an exact mirror of the Westside / Zudio sibling-brand finding, but in the OPPOSITE direction: there the same backend was hiding a sibling brand; here a different brand uses an external competitor as a placeholder).
- **DNS quirk during recon:** in sandboxed environments the host `www.reglobe.in` may fail to resolve via getent while the apex `reglobe.in` resolves fine — use the apex hostname during automated probes.
