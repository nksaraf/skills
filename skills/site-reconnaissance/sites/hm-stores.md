# H&M — store locator

**Domain:** www2.hm.com
**Vertical:** Global fast fashion (India retail — Zara competitor)
**Last verified:** 2026-05-20
**Tier (intended):** 3 + warmup → downgrade to Tier 2 once XHR captured
**Framework:** Adobe Experience Manager (AEM) — Sling selectors visible in the locator URL pattern
**Protection:** **AkamaiGHost** — multi-layer mitigation

**STATUS: ⚠ PARKED.** Live capture failed on 2026-05-20. Extractor + tests + production scraper exist and pass against a synthetic fixture; they're ready to run against a real payload as soon as access is restored.

---

## What we tried (recon log)

### Phase 0 — Raw curl

| URL | Result |
|---|---|
| `https://www2.hm.com/en_in/customer-service/store-locator.html` | HTTP 403 `server: AkamaiGHost`, body "Access Denied", reference `18.…` |
| `https://www2.hm.com/en_in/store-locator/_jcr_content.getStores.json` | HTTP 403 AkamaiGHost |
| `https://www2.hm.com/en_in/store-locator.getStores.json` | HTTP 403 AkamaiGHost |
| `https://www2.hm.com/en_in/store-locator.getStores.IN.json` | HTTP 403 AkamaiGHost |
| `https://api.hm.com/stores/v1/?country=IN` | HTTP 503 AkamaiGHost |
| `https://www2.hm.com/en_us/...` (US locale) | HTTP 403 AkamaiGHost |
| `https://www2.hm.com/sitemap.xml`, `robots.txt` | HTTP 403 AkamaiGHost (blanket block — IP poisoned) |
| `https://app.hm.com/api/...` (mobile-app host) | HTTP 404 (Cloudflare — different infra, but no public stores API) |

Full realistic Chrome client hints + `Referer: https://www.google.com/` was also tried — same 500/403 soft block from a known-bot IP. Googlebot UA → still 403.

### Phase 1 — Stealth browser, homepage warmup

First run (cold IP):
- Warmup `https://www2.hm.com/en_in/index.html` → **200 OK**, real page rendered (title `H&M | Online Fashion, Homeware & Kids Clothes | H&M IN`, body 3392 bytes).
- 25 cookies set: `akainst, AKA_A2, bm_ss, bm_so, bm_sz, bm_lso, akavpau_www2_en_in, _abck, ak_bmsc, …` — full Akamai Bot Manager session.
- Deep-nav to `…/customer-service/store-locator.html` → **200 OK**, but body is 7478 bytes of an Akamai-branded soft-block: title `H&M`, H1 **"IT'S NOT YOU, IT'S US!"**, message **"For some technical reason the page cannot be viewed. Please try later."**

This is the H&M-branded version of Akamai's "softfail" mitigation — distinct from the plain "Access Denied". Triggers on `/customer-service/store-locator.html` even with a fully-warmed Akamai session.

Subsequent locale probes (en_us, en_gb, de_de) — all 200 OK with the same soft block. Within 2 minutes the homepage itself flipped to "Access Denied" (title=`Access Denied`, body=209 bytes) across all locales. The IP is now blanket-blocked at the Akamai edge.

### Phase 1c — XHR capture during soft-block

Only one XHR fires on the soft-block page: `POST www2.hm.com/oSxsyQjrk1xZbCX8sw/EJa9cVu3YQJ5bzYD/VGB1VA/fDA/CUklpPWIB` (Akamai sensor-data beacon). No store data ever loads — the JS bundle that triggers `getStores.json` never executes because the soft-block HTML doesn't include the bootstrap script.

### Phase 4 — Mitigation ladder

| Level | Tried | Result |
|---|---|---|
| 1. Full Chrome headers | ✅ | 403/500 |
| 2. Stealth Chromium | ✅ | First-page works, deep URL gets soft block |
| 3. Stealth + warmup + humanization | ✅ | Same — even with valid Akamai cookies |
| 4. Residential proxy | ❌ not available in this environment | TBD |
| 5. TLS fingerprint spoofing | n/a | wouldn't help (browser TLS is already legitimate) |

We're stuck at Level 3 on this IP. Level 4 (residential proxy or different egress IP) is the unblocker.

## Diagnostic signals (so future recon recognises this)

- **`server: AkamaiGHost`** on the failure response
- **`x-reference-error: 18.…`** — Akamai's reference-ID header (note: NOT `cf-ray` which is Cloudflare)
- **Soft-block HTML signature:** title `H&amp;M`, H1 `IT'S NOT YOU, IT'S US!`, body ~7.8 KB, no real content
- **Cookie set:** `akamref`, `akavpau_*`, `_abck`, `ak_bmsc`, `bm_*` (Akamai Bot Manager full chain — observed only when warmup succeeded)

The soft block returns HTTP 200 — easy to mistake for success. Always check `headings[0]` matches `IT'S NOT YOU, IT'S US!` or `Access Denied`.

## URL patterns (canonical — confirmed by community + Adobe AEM convention; unverified live by us)

| Page type | URL | Method |
|---|---|---|
| Store locator UI | `https://www2.hm.com/<locale>/customer-service/store-locator.html` | GET (HTML) |
| Stores JSON (Adobe AEM) | `https://www2.hm.com/<locale>/store-locator/_jcr_content.getStores.json` | GET (often `?country=XX`) |
| Stores JSON (fallback) | `https://www2.hm.com/<locale>/store-locator.getStores.<XX>.json` | GET |

`<locale>` is `en_in`, `en_us`, `en_gb`, `de_de`, etc. The Sling-selector `_jcr_content.getStores.json` pattern is Adobe AEM's standard component-data endpoint.

## Schema we built against (pending live confirmation)

Documented in `scrapers/retail/hm.ts`'s `HmStoreRaw` interface. Promoted to canonical:

| Canonical field | Source key(s) |
|---|---|
| `storeId` | `storeId` / `storeNumber` |
| `name` | `displayName` / `name` |
| `status` | `status` (`Open` / `ComingSoon` / `Closed`) |
| `address.line1` | `address1` |
| `address.line2` | `address2` |
| `address.city` | `city` |
| `address.state` | `region` / `state` |
| `address.postalCode` | `postCode` / `postalCode` |
| `address.country` | `countryCode` / `country` (normalised to ISO-2) |
| `lat` / `lng` | `latitude` / `longitude` (string OR number — extractor coerces) |
| `phone` | `phoneNumber` / `telephoneNumber` |
| `email` | `email` |
| `url` | `storeUrl` |
| `hours` | `openHours[]` → `{day,opens,closes,closed}` (also `openingHours[]`, `weekdayHours{}` fallback) |
| `storeType` | `storeType` (`Flagship` / `Standard`) |
| `segments` | `concept[]` (`["H&M", "H&M Home", "H&M Kids"]`) |
| `features` | `services[]` (`Click & Collect`, `In-store returns`) |

Preserved in `_extra`: `openingDate`, `closingDate`, `regionCode`, `googlePlaceId`, any other field the payload carries. **Preservation is non-negotiable** — see `reference/retail-store-locator-schema.md`.

## Files in this repo

- **Extractor library:** `scrapers/retail/hm.ts` — `extractHmStore(raw, sourceUrl)`, `extractHmStores(payload, sourceUrl)`
- **Production scraper:** `scrapers/hm-stores.ts` — Tier 3 + Tier 2 hybrid (browser warmup → XHR capture → extract)
- **Recon scraper:** `scrapers/recon-hm-stores.ts` — multi-locale probe (en_in, en_us, en_gb, de_de) with verbose XHR/hydration dump
- **Synthetic fixture:** `__tests__/fixtures/hm-stores-in-synthetic.json` — 4 plausible India records, marked synthetic
- **Tests:** `__tests__/hm-stores.test.ts` — 11 schema-locked tests, all pass

## Gotchas

- **First request succeeds, deep URL soft-blocks.** Pure session warmup is NOT enough for H&M. The locator page itself has a higher-tier Akamai mitigation than the homepage. Expect to need a residential proxy OR a real browser fingerprint chain.
- **Datacenter IPs get poisoned fast.** A handful of homepage hits is fine; the moment you deep-nav to the locator, all subsequent requests (including `robots.txt`) flip to 403. Recovery window appears > 5 minutes — possibly much longer.
- **Soft-block returns HTTP 200, not 403.** Easy to mistake for success. Always inspect `<h1>` or `headings[0]`. Look for `IT'S NOT YOU, IT'S US!`.
- **Locale doesn't help.** `en_us`, `en_gb`, `de_de` all use the same Akamai policy on the locator URL. Picking a non-India locale won't unblock you and may make Akamai more suspicious.
- **Cookie chain is sophisticated:** `_abck` (Akamai Bot Manager session), `ak_bmsc`, `bm_so/ss/sz/sv/s`, `akavpau_*` (per-locale Akamai vault), `akamref` (referrer), `akainst` (instance), `AKA_A2`. Replaying a single cookie won't work — Akamai also fingerprints sensor data and TLS.
- **`getStores.json` schema is unconfirmed live.** Our extractor follows the documented Adobe AEM Sling-selector convention and matches what community scrapers report, but until we capture a real response we don't know which fields actually populate vs. arrive as `null`. The synthetic fixture lock should fail loudly the moment the live shape diverges.
- **`api.hm.com` is also Akamai-protected** (503 from datacenter IPs), and the only `/api/...` path on `app.hm.com` returns Cloudflare 404 — there's no public mobile API to fall back to.

## Re-attempt plan

1. **Wait ≥ 1 hour for IP cooldown** (or run from a different egress).
2. Run `bunx vinxi-scraper run recon-hm-stores` again. Watch for the homepage to load real content (`bodyBytes > 3000`, no "Access Denied").
3. If the homepage works but the locator soft-blocks, escalate immediately to Level 4 (residential proxy) — don't burn the IP further.
4. If the locator loads real content, the XHR capture should pick up the `getStores.json` call. Save the response to `__tests__/fixtures/hm-stores-in-live.json` and re-run the test suite.
5. The extractor in `scrapers/retail/hm.ts` is already shaped against the documented payload — only the fixture needs swapping (and any field-name corrections the test failures surface).

## Cross-brand notes

H&M is the **second** Akamai-protected retail target after Zara (also parked for warmup reasons). The mitigation behaviour is identical in shape: homepage works once, deep URL flips to soft-block, IP poisons quickly. The Westside-style direct-API Tier 2 trick doesn't apply — H&M's only public store-data path runs through `www2.hm.com` itself.

If H&M, Zara, and Mango all need a residential proxy, that's a project-wide constraint worth treating as infrastructure (a single egress pool the whole scraper engine routes through). Filed against `references/scraper-engine` as future work.
