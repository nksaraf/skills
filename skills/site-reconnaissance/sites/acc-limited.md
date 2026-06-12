# ACC Limited (Adani Cement)

**Domain:** acclimited.com
**Vertical:** Sales & distribution — cement channel partners (India)
**Parent:** Adani Group (Adani Cement); sibling = Ambuja Cement
**Last verified:** 2026-05-21
**Tier:** N/A — no public locator surface
**Framework:** Next.js (App Router, RSC) on a Sitecore-backed `/-/media/` asset host
**Protection:** none observed (plain Chrome UA passes everywhere — but nothing to protect)
**Status:** ❌ **BLOCKED / PARKED** — no public dealer-locator surface on either ACC or sibling Ambuja

## TL;DR

ACC has no public dealer locator. Neither does its sister brand Ambuja. The
sibling-brand probe — the whole reason this recon was a priority — found
that BOTH Adani Cement properties have stripped consumer-facing channel
discovery in favour of investor-comms-only websites post-acquisition.
There is no shared backend to extract because there is no backend on either
side. The would-be UltraTech-equivalent "single XHR returns all channel
partners" does not exist for Adani Cement in 2026.

## Why this is a structural blocker, not a transient one

Three independent signals all point to "the locator is gone on purpose":

1. **YAML's `official_locator_url` returns 404.** `/contact-us/dealer-locator`
   is a Next.js 404 page (the site's catchall). Not a "page moved" — the
   path is simply not part of the deployed routing tree.
2. **`sitemap.xml` is empty on ACC and locator-less on Ambuja.** ACC's
   `sitemap.xml` is a literal `<urlset/>` with zero `<loc>` entries. Ambuja's
   `sitemap.xml` lists 47 URLs covering Investors, Newsroom, Sustainability,
   About, Contact — NONE of them are dealer / store / locator pages.
3. **Homepage nav features the merger media release, not a locator.** Both
   homepages prominently link to "Ambuja Cements Limited Board Approves
   Amalgamation of ACC Limited and Orient Cement Limited" — the brands are
   consolidating, not investing in consumer touchpoints. The Adani Cement
   digital team is running BOTH sites as investor-comms pages.

## URL probe sweep — all 19 paths return 404

```
acclimited.com/dealer-locator              -> 404
acclimited.com/contact-us/dealer-locator   -> 404  ← the YAML's claim
acclimited.com/find-a-dealer               -> 404
acclimited.com/find-dealer                 -> 404
acclimited.com/find-a-store                -> 404
acclimited.com/store-locator               -> 404
acclimited.com/locator                     -> 404
acclimited.com/dealers                     -> 404
acclimited.com/stores                      -> 404
acclimited.com/find-us                     -> 404
acclimited.com/dealer-near-me              -> 404
acclimited.com/where-to-buy                -> 404
acclimited.com/channel-partners            -> 404
acclimited.com/our-network                 -> 404
acclimited.com/distributors                -> 404
acclimited.com/for-homebuilders            -> 404
acclimited.com/trade                       -> 404
acclimited.com/products                    -> 404
acclimited.com/our-brands                  -> 404
```

The exact same 19-path sweep against `ambujacement.com` returns the same
404 pattern. Alternative ACC domains:

- `acccement.com` — parked / for-sale (fingerprint redirector)
- `acc.in` — parked by daaz.com (Cloudflare lander)

## Sibling-brand backend comparison vs Ambuja

This was the critical question the recon brief was sized around. Here is
the answer.

| Probe | ACC (acclimited.com) | Ambuja (ambujacement.com) | Shared backend? |
|---|---|---|---|
| Locator paths returning 200 | 0 / 19 | 0 / 19 | n/a — nothing to share |
| `sitemap.xml` | empty `<urlset/>` | 47 URLs, zero locator pages | n/a |
| Next.js static chunk hash (CSS) | `4bb1c53d4d41ca49.css` | `4bb1c53d4d41ca49.css` | ✅ **same shared chunk** |
| Sitecore media routing | `/-/media/Project/ACC-Limited/...` | `/-/media/Project/AmbujaLimited/...` | ✅ same convention |
| Homepage merger banner | yes | yes | ✅ same content team |
| `customer_id` / vendor signal | N/A (no API call) | N/A (no API call) | n/a |
| Akamai / Imperva / Cloudflare | none | none | n/a |

**Diagnosis.** ACC and Ambuja are clearly run by the same Adani Cement
digital team — shared CSS chunks, shared media-routing convention, shared
Next.js scaffolding, shared Sitecore CMS backbone, shared merger-comms
banner. If either ever ships a dealer locator, it will almost certainly
ship on both sites simultaneously and share the same backend.

**But until then, there is no shared backend — because there is no backend.**
The hopeful "one extractor covers both" insight is unrealised through no
fault of the recon. It's a strategic choice by Adani.

## Comparison vs the UltraTech recon (the success case)

UltraTech (Aditya Birla) has the same structural opportunity as Adani
Cement — multi-brand cement major with one digital team. Their choice was
the opposite: publish ONE Excel-converted-to-JSON dump of ~3,956 channel
partners via `/bin/exceltojsonservlet`, exposing the entire public locator
in a single XHR. See `ultratech-cement.md`.

| Dimension | UltraTech | ACC + Ambuja |
|---|---|---|
| Public locator | yes — single XHR returns all rows | NO (both) |
| Framework | AEM | Next.js + Sitecore |
| Channel-partner count published | ~3,956 (≈4% of ~100k corporate disclosure) | 0 (none published) |
| Strategic posture | "consumer-facing find-a-dealer" | "investor-comms only" |
| Recon tier | 2 (single XHR) | n/a — nothing to scrape |

If/when Adani Cement restores a locator, the recipe almost certainly
mirrors UltraTech: a single backend endpoint serving the entire public
slice, easily probed once. **Quarterly re-probe is the right cadence** —
the merger is in-flight (Ambuja-ACC-Orient amalgamation, FY26), and a
unified post-merger website might restore consumer surfaces.

## What an extractor WOULD look like (for the day Adani restores the locator)

Mirror UltraTech: single XHR → schema walk → project to `RetailStore`
with `locationType: "dealer"` (or `"stockist"` if the source labels the
wholesale tier). The schema-walker should be re-pointed at whatever
hydration the restored locator uses.

## Files in this repo

- **Extractor stub:** `scrapers/sales-distribution/acc-limited.ts` — exports
  `RECON_OUTCOME`, `PROBED_PATHS`, `SIBLING_BRAND`, and a fail-loud
  `extractACCDealers()`. No live data path because there's no source.
- **Tests:** `__tests__/acc-limited-stores.test.ts` — 7 tests asserting the
  parked-state shape. A future restored locator will fail one of them and
  force a fresh recon.
- **Live data:** none (no JSONL). Phase D pipeline is not exercised — we
  have no records to score, snapshot, or alert on.
- **YAML:** `targets/industries/sales-distribution/acc-limited.yaml` — status
  flipped to `parked` with a structural reason.

## Pagination

N/A (no source).

## Gotchas

- The YAML's `official_locator_url: /contact-us/dealer-locator` is a **404**.
  Do not trust YAML locator URLs without re-probing. (Same gotcha hit
  UltraTech, Okinawa, Hero Vida, Revolt, and Ola Electric.)
- `acc.in` and `acccement.com` are NOT ACC's domains — they are parked /
  for-sale domains. The only official property is `acclimited.com`.
- ACC's `sitemap.xml` is empty (literal `<urlset/>`). This is unusual but
  consistent with a corporate site that has no consumer journey.
- The home page links to a media release titled "Ambuja Cements Limited
  Board Approves Amalgamation of ACC Limited and Orient Cement Limited" —
  if the merger completes, expect ACC's domain to redirect or shut down
  and Ambuja's site to absorb whatever consumer features survive. Re-probe
  post-merger.

## Re-probe trigger conditions

Set status back to `discovered` and re-run recon when ANY of these flip:

1. `https://www.acclimited.com/contact-us/dealer-locator` returns 200.
2. `https://www.ambujacement.com/sitemap.xml` lists a `/dealer*` or
   `/find*` URL.
3. The ACC-Ambuja-Orient amalgamation closes (FY26 expected) and a
   unified `adanicement.com` (or similar) launches with a locator.

The acc-limited.ts stub's `RECON_OUTCOME` constant is the canonical place
to flip the parked state.
