# Site Index

Every site we've reconned, in one table. Look here FIRST before reconning a new target. If the site is listed, the addendum has everything we already know.

> **Provenance note:** pointers to extractors, fixtures, and test files in these addenda refer to the workspace where the site was originally reconned — they won't resolve in your project. The portable intelligence is everything else: URL patterns, framework signatures, hydration payload paths, API endpoints + required headers, protection profiles, and gotchas. Re-create the extractor in your own `scrapers/` from those.

## Index

### Real estate

| Site | Domain | Vertical | Framework | Tier | Protection | Status | Last verified | Addendum |
|---|---|---|---|---|---|---|---|---|
| 99acres | 99acres.com | India real estate | Custom React SSR (`window.__initialData__`) | 3 | Akamai Bot Manager | ✅ extractor + fixture + 11 tests | 2026-05-19 | [99acres.md](./99acres.md) |
| MagicBricks | magicbricks.com | India real estate | Custom React SSR (`window.SERVER_PRELOADED_STATE_`) | 3 | mild | ⚠ recon only | 2026-05-19 | [magicbricks.md](./magicbricks.md) |
| IndiaMART | dir.indiamart.com | India B2B catalog | Next.js (`__NEXT_DATA__`) | 3 | mild | ⚠ recon only | 2026-05-19 | [indiamart.md](./indiamart.md) |
| OLX | olx.in | India classifieds | Custom + JSON-LD heavy | 3 | mild | ⚠ recon only | 2026-05-19 | [olx.md](./olx.md) |
| **Housing.com** | housing.com | India real estate (commercial rental — office/shop/showroom/warehouse) | Custom SPA (Next.js; SSR's first page into HTML) | 3 + warmup chain | **Imperva** (bypass: homepage → Hyderabad SRP → Delhi overview → target) | ✅ extractor + fixture + 24 tests + 2,310+ record Delhi sweep | 2026-05-21 | [housing.md](./housing.md) |
| CommonFloor | commonfloor.com | India real estate | Old SSR (incremental `window.X.foo` build) | 3 | none observed | ⚠ recon only | 2026-05-19 | [commonfloor.md](./commonfloor.md) |
| **NoBroker** | nobroker.in | India real estate (owner-direct, no-brokerage) | Custom React/Redux SSR (`nb.appState.rentListPage.listPageProperties`) | 3 | mild (nginx; homepage warmup required) | ✅ extractor + fixture + 18 tests | 2026-05-21 | [nobroker.md](./nobroker.md) |
| **MakaaN** | makaan.com | India real estate (formerly residential + commercial rental/sale) | n/a (defunct) | n/a | n/a | ❌ **DEFUNCT** — every URL 301→`origin.makaan.com/housing-buyer-redirect` (502/504); buyer traffic migrated to Housing.com; no data accessible | 2026-05-22 | [makaan.md](./makaan.md) |
| **Square Yards** | squareyards.com | India real estate (residential + commercial rentals; 10 metros) | Custom Node.js SSR (no Next.js / no `window.*` hydration; data in `data-*` attrs) | 3 (Akamai present; stealth browser) | Akamai Bot Manager (mild — SRPs serve 200 to plain curl; homepage warmup sufficient) | ✅ extractor + fixture + 21 tests + Delhi sweep running | 2026-05-22 | [squareyards.md](./squareyards.md) |
| **PropertyWala** | propertywala.com | India real estate (NCR-focused — residential + commercial rental) | Custom ASP.NET/jQuery **pure SSR** (`<article id="P{id}">` cards; no XHR/hydration) | **1** (lowest complexity — plain HTML parse) | None observed (no Akamai/Cloudflare/Imperva) | ✅ extractor + fixture + 25 tests + Delhi sweep | 2026-05-22 | [propertywala.md](./propertywala.md) |

### Retail / brand store locators (fast fashion — Zara competitors)

| Brand | Domain | Tier | API host | Protection | Stores | Status | Last verified | Addendum |
|---|---|---|---|---|---|---|---|---|
| **Westside** (Tata) | westside.com | 2 (direct API) | customapp.trent-tata.com | none | 300 | ✅ extractor + fixture + 11 tests + scorecard 95/100 | 2026-05-20 | [westside-stores.md](./westside-stores.md) |
| **Zudio** (Tata) | zudio.com | 2 (same Trent-tata API as Westside) | customapp.trent-tata.com `?type=zudio` | none | 555 | ✅ extractor (shared with Westside) + scorecard 89/100; **discovered via sibling-brand probe** | 2026-05-20 | [zudio-stores.md](./zudio-stores.md) |
| Zara | zara.com | 3 + warmup needed | (TBD) | Akamai (Access Denied on default) | - | ⚠ blocked on first try; needs warmup | 2026-05-20 | (parked) |
| **H&M** | www2.hm.com | 3 + warmup → Tier 2 (once XHR captured) | www2.hm.com Adobe AEM `_jcr_content.getStores.json` | **AkamaiGHost** — homepage ok; locator deep URL soft-blocks ("IT'S NOT YOU, IT'S US!") | - | ⚠ extractor + scraper + 11 tests against synthetic fixture; live capture parked (residential proxy needed) | 2026-05-20 | [hm-stores.md](./hm-stores.md) |
| **Mango** | shop.mango.com | 1 (Next.js RSC stream — no API) | n/a (RSC payload in page HTML) | mild Akamai (gentle pacing) | 9 | ✅ extractor + fixture + 11 tests + production output | 2026-05-20 | [mango-stores.md](./mango-stores.md) |
| **Uniqlo** (India) | map.uniqlo.com | 2 (direct JSON API) | map.uniqlo.com `/in/api/storelocator/v1/en/` | mild Akamai (origin/referer suffices; no warmup) | 20 | ✅ extractor + fixture + 11 tests + production output | 2026-05-20 | [uniqlo-stores.md](./uniqlo-stores.md) |
| Lifestyle (Landmark) | lifestylestores.com | 1 (HTML map markers — minimal) **OR** 1 per-store detail (extensive) | stores.lifestylestores.com (Singleinterface) | Cloudflare on www; stores subdomain unprotected | ~150+ (estimate) | ⚠ recon only — landing has only 20 markers (lat/lng/name); full per-store data lives on 150+ detail pages | 2026-05-20 | [lifestyle-stores.md](./lifestyle-stores.md) |
| **Bata India** | bata.in / stores.bata.com | 1 (sitemap + JSON-LD per page) | stores.bata.com (Singleinterface / Promanage `domainid=59`) | none on `stores.bata.com`; `www.bata.com` is **Akamai-blackholed** but unused | 1,620 | ✅ extractor + fixture + 8 tests + production output | 2026-05-21 | [bata.md](./bata.md) |

### E-commerce — product listings (PLP / activity / category pages)

| Brand | Domain | Tier | API host | Protection | Coverage | Status | Last verified | Addendum |
|---|---|---|---|---|---|---|---|---|
| **Savana** (India + Iraq; **Urbanic rebrand**) | savana.com (sister: urbanic.com for UK/US/CA/MX/BR/ZA) | 2 (direct JSON API across 3 endpoints) | api-shop-in.savana.com (`/n/api/buyer/guide/user/goods-flow/pageList` for listings, `module/pageDetail?sceneType=INDIVIDUATION_H5_HOME` for homepage modules → activity discovery, `intention/item/v4/detail` for per-product rich data); sister IQ host `api-shop-iq.savana.com` | mild Cloudflare on `www.savana.com` (no challenge); API host accepts plain Chrome UA + required tenant headers (`client_type:h5`, `country-language:en-IN`, `h5-version:6.34.0`, `x-platform:web`, `x-source:h5`, `app_version:6.34.0`); robots.txt `Disallow:/` for unknown UA but bypassed by browser-UA + tenant headers | **42,172 product pages in sitemap; 1,097 live `/activity/<id>` feeds running (104 daily flash sales, 7 BOGO, 24 clearance, 73 mood-aesthetic, 71 %-off, 27 under-₹N, 5 members-only); only 25 (2.3%) surface on homepage; dresses-12769 walked → 2,707 unique goods (~5 min); per-product detail call exposes style/occasion/fabric/size-chart/review-count + multi-region size standards (IN/EU/UK/US/BR/MEX); tag taxonomy = `fastDelivery` (img: 3-5 business days per their own shipping policy) + `marketing` (text: BOGO labels); no best-seller tag; coupon system anonymous-gated (welcome 20% off ≥₹500); pricing sweet-spot ₹500-2k (median ₹1,390, range ₹489-7,590)** | ✅ 3 extractor modules + 4 fixtures + 31 tests + 2 production scrapers + business-intel companion doc + 1,097-activity catalog dump | 2026-06-11 | [savana.md](./savana.md) + [savana-business-intel.md](./savana-business-intel.md) |

### Retail / QSR (Quick Service Restaurants)

| Brand | Domain | Tier | Discovery | Protection | Stores | Status | Last verified | Addendum |
|---|---|---|---|---|---|---|---|---|
| **Domino's Pizza India** (Jubilant FoodWorks) | dominos.co.in | 1 (sitemap → per-store HTML) | `sitemap_store.xml` lists 1,722 URLs | none (CloudFront caches) | 1,669 | ✅ extractor + fixtures + 13 tests + scorecard 84/100 | 2026-05-21 | [dominos-india.md](./dominos-india.md) |

### Retail / Recommerce (used-phone buy-back + repair)

| Brand | Domain | Tier | Discovery | Protection | Stores | Status | Last verified | Addendum |
|---|---|---|---|---|---|---|---|---|
| **Cashify** | cashify.in | 1 (Next.js RSC stream, three-layer fetch) | `/offline-stores` listing → 91 city pages → 206 detail pages | none observed | 206 stores (0 pickup-points — locator surfaces owned outlets only) | ✅ extractor + fixture + 14 tests + scorecard 84/100 (high) | 2026-05-21 | [cashify.md](./cashify.md) |
| **Yaantra / ReGlobe** (Reliance Retail Ventures) | reglobe.in (+ legacy yaantra.com) | n/a (no locator) | reglobe.in `/stores` + 14 other paths probed (incl. sitemap.xml / robots.txt) — every URL 301s to `https://www.cashify.in/?__utmrg=____ref_*&__utmh=reglobe.in`; yaantra.com homepage renders but all 57 sitemap.xml `/repair/*` URLs (lastmod 2023-01-24) return a `canonical=/pagenotfound` 404 stub | n/a (nothing to protect) | 0 (vs pre-acquisition ~150 stores + repair centres) | ❌ **PARKED — no public locator** — post-2022 Reliance acquisition, reglobe.in has been wired up as an affiliate funnel to **Cashify** (an INDEPENDENT competitor — not a backend share); yaantra.com is a homepage-only husk. Stub extractor + 10 tests assert parked-state shape | 2026-05-22 | [yaantra-reglobe.md](./yaantra-reglobe.md) |

### Retail / Mobile + consumer-electronics specialist (South India)

| Brand | Domain | Tier | Discovery | Protection | Stores | Status | Last verified | Addendum |
|---|---|---|---|---|---|---|---|---|
| **Sangeetha Mobiles** | sangeethamobiles.com | 2 (single POST to Django backend) | Next.js SPA; `/store-locate` (yaml's `store-locator` is 404); endpoint recovered from `_app-*.js` Redux thunk | none — hardcoded `client_id`+`secret_key` in SPA bundle; Django backend has `DEBUG=True` (URLconf leak, not exploited) | 924 stores (vs ~750 press-cited; +23% likely reflects FY24-25 expansion) | ✅ extractor + fixture + 14 tests + scorecard 91/100 (high) | 2026-05-21 | [sangeetha-mobiles.md](./sangeetha-mobiles.md) |

### Sales / distribution — channel partners + B2B locators

| Brand | Domain | Tier | Discovery | Protection | Count | Status | Last verified | Addendum |
|---|---|---|---|---|---|---|---|---|
| **Varun Beverages (VBL)** | varunbeverages.com | 1 (HTML table on `/plants/`) | WordPress `/wp-json/wp/v2/pages` enumeration (page not in main nav) | none | 39 plants | ✅ extractor + fixture + 8 tests; 39/39 matches FY24 disclosure | 2026-05-21 | [vbl-varun-beverages.md](./vbl-varun-beverages.md) |
| **UltraTech Cement** (Aditya Birla) | ultratechcement.com | 2 (single XHR POST to AEM `/bin/exceltojsonservlet`) | www.ultratechcement.com `/bin/exceltojsonservlet?filePath=...storelocator.xlsx` | none (CloudFront, plain Chrome UA passes) | 3,956 dealers across 25 states (public locator ≈ 4% of corporate ~100k channel-partner claim — wholesale tier not published) | ✅ extractor + fixture + 13 tests + scorecard 81/100 (high) | 2026-05-21 | [ultratech-cement.md](./ultratech-cement.md) |
| **Shree Cement** (Shree Cement Ltd) | shreecement.com (locator on `dealers.shreecement.com`) | 1 (state×page HTML sweep — every field as hidden inputs per card) | Synup white-label `EThemeForMasterGrid` (customerId 435076) | none (default Chrome UA passes; no Cloudflare/Akamai) | 242 dealers across 12 states — sub-brands Shree Cement (213) + Rock Strong (29); Bangur on separate subdomain (out of scope) | ✅ extractor + 2 fixtures + 17 tests + scorecard 89/100 (high) | 2026-05-21 | [shree-cement.md](./shree-cement.md) |
| **Ambuja Cement** (Adani Cement) | ambujacement.com (corporate, no locator) → **ambujahelp.in** | 2 (state-sweep on `/dealersapi/dealers/{state}`) | www.ambujahelp.in Sitecore + Next.js + custom REST | none (default Chrome UA passes) | 10,353 dealers across 25 states (YAML's `/dealer-locator` is 404; real locator is on the customer microsite); **same backend as sibling ACC** (`acchelp.in`) — completely different vendor from UltraTech | ✅ extractor + fixture + 17 tests + scorecard 80/100 (high) | 2026-05-21 | [ambuja-cement.md](./ambuja-cement.md) |
| **Okinawa Autotech** (E2W) | okinawascooters.com | 1 (single HTML GET on `/find-us`) | apex domain (www. 404s; `/dealer-locator` 404s; sitemap.xml lists only 7 flagship "Galaxy Stores") | none (ASP.NET WebForms on IIS, default Chrome UA passes) | 315 dealers (vs press-cited ~350; real FAME-II clawback contraction documented in addendum) | ✅ extractor + fixture + 10 tests + scorecard 65/100 (medium — capped by absent coords/phone/hours) | 2026-05-21 | [okinawa.md](./okinawa.md) |
| **Ampere Electric** (Greaves Cotton, E2W) | amperevehicles.com | 1 (sitemap + per-page JSON-LD) | ampere-showroom.greaveselectricmobility.com (Singleinterface `customerId=537585`) | Cloudflare WAF on `www.amperevehicles.com` (locator unaffected); microsite unprotected | 389 dealers (vs 400+ live source-claim, vs 600+ FY24 Greaves disclosure) | ✅ extractor + fixtures + 10 tests + scorecard 86/100 (high) | 2026-05-21 | [ampere-electric.md](./ampere-electric.md) |
| **Hero Vida** (Hero MotoCorp E2W) | vidaworld.com (yaml's `hero-vida.com` is a parked GoDaddy domain) | 2 (AEM JCR selector JSON) | www.vidaworld.com `/content/vida/in/en/sf-master/jcr:content.dealerlistwithoutsku.<STATE>.<CITY>.json` | none | 8 outlets (across 5 cities) | ✅ extractor + 4 fixtures + 9 tests + scorecard 71/100 (medium); **authoritative ~100 — substantial gap, see addendum** | 2026-05-21 | [hero-vida.md](./hero-vida.md) |
| **Revolt Motors** (RattanIndia, E2W motorcycles) | revoltmotors.com | 1 (Next.js RSC `__next_f` reassembly on `/contact-us`) | RSC embeds state[] + city[] + dealer[] arrays in one HTML response | Cloudflare (no challenge) | 259 dealers (yaml's `/dealer-locator` is a 404; canonical is `/contact-us`; press-cited "70+" is stale) | ✅ extractor + fixture + 9 tests + scorecard 81/100 (high) | 2026-05-21 | [revolt-motors.md](./revolt-motors.md) |
| **Ather Energy** (E2W) | atherenergy.com | 3 (stealth Playwright — Cloudflare WAF blocks plain HTTP including robots-allowed bot UAs) | `/locate-ather-dealer` page renders one inline GeoJSON FeatureCollection with every dealer in country | Cloudflare WAF (403 on every UA; only stealth PW passes) | 1,115 touchpoints (629 SALES + 478 SERVICE + 8 SALES_AND_SERVICE; vs 250 FY24 DRHP claim — 4× growth since IPO) | ✅ extractor + fixtures + 10 tests + scorecard 87/100 (high) | 2026-05-21 | [ather-energy.md](./ather-energy.md) |
| **JK Cement** (JK Cement Ltd) | jkcement.com | 2 (single XHR POST on WordPress `admin-ajax.php`) | www.jkcement.com `/wp-admin/admin-ajax.php` body=`action=get_all_stores` (yaml's `/dealer-locator/` is 404; real locator at `/store-locator` → SEO-canonical `/grey-cement/find-a-dealer/`) | none (Imperva-Incapsula `__uzm*` issued on apex but unenforced on admin-ajax) | 8,905 dealers across 18 canonical states (Uttaranchal collapsed into Uttarakhand from source's 19 buckets); north-India dominant (UP / Rajasthan / Karnataka / MP / Gujarat / Bihar / Haryana ≈ 85%); WB/Odisha/Jharkhand/Telangana/AP/TN/NE absent | ✅ extractor + fixture + 16 tests + scorecard 81/100 (high); **fifth distinct cement-locator stack reconned — WordPress vs UltraTech AEM / Ambuja Sitecore / Shree Synup / Dalmia CodeIgniter; closest cousin UltraTech (single-call), but JK ships lat/lng (72%) + tier (Bronze-Titanium)** | 2026-05-22 | [jk-cement.md](./jk-cement.md) |
| **India Cements** (being acquired by UltraTech, deal pending FY26) | indiacements.co.in | 1 (single GET of `/js/jquery.mapit.js`; parsed via single-quote-aware bracket scan over the inline `locations: [...]` literal) | YAML's `/dealer-locator` is 404; legacy `/dealers-locations.html` exists but is **parked** ("process of updating", `dealers_action.php` 404); canonical surface is `/plants-locations.html` whose Google Maps canvas reads `js/jquery.mapit.js` | none (Microsoft IIS, default Chrome UA passes) | 9 cement plants (4 TN + 2 AP + 2 TG + 1 RJ; 14.45 mtpa total disclosed); mobile header embeds UltraTech logo (acquisition signal) but stack has NOT migrated yet | ✅ extractor + fixture + 15 tests + scorecard 86/100 (high); dealer surface parked at source pending UltraTech deal closure | 2026-05-22 | [india-cements.md](./india-cements.md) |
| **Ola Electric** (E2W) | olaelectric.com | 2 (pincode-keyed XHR on Next.js host) | ola-ev-ui.olaelectric.com `/api/getExperienceCenterDetails?pincode=<6digit>` returns ≤5 outlets within ~30 km; sweep + adaptive zoom | mild (Akamai-fronted Magento storefront passes plain UA; API soft-403s on burst — backoff recovers) | 350 (of ~800 published in IPO prospectus — API's 5-nearest cap systematically hides dense urban clusters) | ✅ extractor + 2 fixtures + 10 tests + scorecard 71/100 (medium — completeness 48 dragged by 5-nearest cap); `/stores` is 404, canonical is `/experience-centres` | 2026-05-21 | [ola-electric.md](./ola-electric.md) |
| **Birla Corporation** (M.P. Birla Group cement; mid-tier ~12 MTPA, listed) | birlacorporation.com (corporate; `/dealer-locator` is 404) → **mpbirlacement.com** (consumer microsite) | 1 (WordPress `admin-ajax.php` with strict-pincode-equality semantics — coarse stride-10 sweep across India 100000-855999, then ±50 zoom around dealer-bearing seeds) | www.mpbirlacement.com `/wp-admin/admin-ajax.php` body=`action=search_dealers&pincode=<6digit>` (HTML response: `<div class="dealer-grid">` of `<div class="dealer-card">`); WP REST `/wp-json/wp/v2/dealers?per_page=1` ships `x-wp-total: 6415` as authoritative-count oracle | none (plain Chrome UA passes) | TBD live (≤ 6,415 universe ceiling per WP CPT total); locator surfaces dealers in 8 nominal-footprint states (WB, Bihar, UP, MP, Rajasthan, Haryana, Gujarat, Maharashtra) **plus** out-of-footprint Telangana (and likely others) — empirically widened from the "Become a Dealer" form subset | ✅ extractor + 4 fixtures + 19 tests; **sub-brand**: locator does NOT surface per-dealer sub-brand (Perfect Plus / Unique Plus / Ultimate / Samrat Advanced / Concrecem / Multicem all roll up to composite "MP Birla Cement" outlet — structurally distinct from Shree Cement which splits) | 2026-05-22 | [birla-corp.md](./birla-corp.md) |
| **Ramco Cements** (Madras-HQ, south India's largest cement player by volume) | ramcocements.in | **3 (stealth Playwright — domain-locked reCAPTCHA v3 forces real-browser captcha mint)** | www.ramcocements.in `/cms/graphql` (Strapi v4 GraphQL); five queries `loadStates`/`loadDistricts`/`loadLocations`/`getStoresFromZIP`/`getStoresFromLatLon`, all captcha-gated; **yaml's `/dealer-locator` is a Next.js 404 — live is `/locator`** | mild **Imperva** front-end + **reCAPTCHA v3** invisible widget (site key `6Le3Qq...A6Qt3dsmeG`, domain-locked) | south + east only (TN/AP/TS/KA/KL/WB/OR/BR/AS); `Cust_No` is per-product-line not per-location (TRIMURTHI distributors appears 3× at same address under 3 codes); **ONLY Indian cement major with a real anti-bot captcha gate** — five vendors in five cement majors (UltraTech AEM / Ambuja Sitecore-REST / Shree Synup / Dalmia CodeIgniter+Radware / Ramco Strapi+reCAPTCHA / JK WordPress / Birla WordPress) | ✅ extractor + 2 fixtures + 20 tests | 2026-05-22 | [ramco-cements.md](./ramco-cements.md) |
| **Nuvoco Vistas** (Nirma Group, #5 cement player by capacity, east-India dominant post-Lafarge 2016) | nuvoco.com | 1 (CRA SPA bundle mining — `/static/js/main.<hash>.js`) | yaml's `/dealer-locator` is a SPA fallback (renders homepage shell); NO consumer dealer locator exists anywhere on nuvoco.com; React bundle ships an internal `greatPlaces:[…]` array (default-prop of the "Our Presence" Google-Maps widget); bracket-balanced scan + JS-literal → JSON munge | none (plain Chrome UA passes; CRA static assets on CloudFront) | 72 facilities = 11 cement plants + 44 RMX plants + 16 offices + 1 innovation centre (CDIC Mumbai); NO public dealer data despite FY24 disclosure of ~25,000 channel partners | ✅ extractor + 2 fixtures + 16 tests + scorecard 85/100 (high); confirms cement-major pattern: only top-3 + JK + Birla + Ramco publish public dealer locators; Nuvoco / ACC / Dalmia ship NO consumer-facing channel-partner surface | 2026-05-22 | [nuvoco-vistas.md](./nuvoco-vistas.md) |
| **ACC Limited** (Adani Cement) | acclimited.com | n/a (no locator) | 19 candidate locator paths probed (yaml's `/contact-us/dealer-locator`, `/dealer-locator`, `/find-a-dealer`, `/where-to-buy`, `/store-locator`, …) all 404; `sitemap.xml` is empty `<urlset/>`; sister brand `acccement.com` is a parked GoDaddy fingerprint redirector; `acc.in` is a parked daaz.com lander. **Sibling-brand probe vs Ambuja: NEGATIVE — no shared backend because Ambuja has the same absent-locator posture (47-URL sitemap, zero locator pages, same Next.js+Sitecore stack and shared CSS chunk `4bb1c53d4d41ca49.css`)**. UltraTech remains the only Indian cement major with a usable public locator | none (nothing to protect) | 0 (no public locator on ACC or sibling Ambuja) | ❌ **PARKED — no public locator** — Adani Cement (post-Sept-2022 acquisition + in-flight Ambuja-ACC-Orient amalgamation) has stripped consumer surfaces from BOTH sites in favour of investor-comms-only websites; stub extractor + 7 tests assert parked-state shape | 2026-05-21 | [acc-limited.md](./acc-limited.md) |

### FMCG — packaged water / beverages

| Brand | Domain | Tier | Discovery | Protection | Count | Status | Last verified | Addendum |
|---|---|---|---|---|---|---|---|---|
| **Bisleri** | bisleri.com | n/a (no locator) | Salesforce Commerce Cloud (Demandware) D2C storefront; `/book-plant-visit` is a 5-city booking form (not a plant list); sitemap.xml is 404; not WordPress (wp-json N/A); no `/our-plants`, `/distributors`, `/store-locator`, `/cafe-bisleri` — all 404 | mild Cloudflare (irrelevant) | 0 (no public locator of any kind) | ⚠ **PARKED** — privately held, no investor-relations site; D2C domain has no corporate-locator section. Re-check on site redesign / IPO / FSSAI licensee registry as secondary source | 2026-05-21 | [bisleri.md](./bisleri.md) |
| **Parle Agro** | parleagro.com (locator on `/contact-us`) | 1 (HTML state-tabbed list parsed via regex) | `/contact-us` Beverage manufacturing facilities section — 15 plants (10 PAPL + 5 Franchise) across 12 Indian states + Nepal; NOT WordPress (wp-json N/A); `/our-business`, `/plants`, `/factories`, sitemap.xml all 404 — page name belies content | none (default Chrome UA passes) | 15 plants | ✅ extractor + fixture + 11 tests + scorecard 73/100 (medium — data-quality capped by no lat/lng/phone) | 2026-05-21 | [parle-agro.md](./parle-agro.md) |
| **Coca-Cola India** (brand owner, NOT bottler) | coca-cola.com/in | n/a (no locator) | Adobe AEM "OneXP" (`/content/onexp/in/en/...`) on CloudFront `CEP`; full `/in/en/sitemap` enumerates 24 corp/marketing/legal pages (about-us, 14 brand pages, sustainability, legal, media-center); `/our-plants`, `/plants`, `/find-us`, `/store-locator`, `/where-to-buy`, `/manufacturing` all 404; `/sitemap.xml` 404; Helix `query-index.json` 404; contact-us is an AEM Forms consumer-feedback widget. **NOT WordPress — VBL wp-json playbook does not apply.** Sister brand HCCB is the bottler (separate slug, out of scope). | none observed (CloudFront `CEP`, default Chrome UA passes) | 0 (no public locator) | ⚠ **PARKED** — global beverage brand-owner pattern: location data sits at the bottler tier (HCCB private; VBL listed and publishes plants). Re-check on site redesign / India D2C launch | 2026-05-21 | [coca-cola-india.md](./coca-cola-india.md) |
| **Dabur** (Real / Hajmola / Vatika) | dabur.com (locator on `/contact-information`) | 1 (HTML `views-row` parse over six quicktabs) | sitemap_contact.xml led to canonical page; wp-json 404s (Drupal, not WP); guess-paths `/plants`, `/our-presence`, `/manufacturing`, `/locations` all 404 with 168 KB pretty error page | none (CloudFront, plain Chrome UA passes) | 58 (29 factory units + 14 branch offices + 10 overseas + 2 corp + 2 subsidiaries + 1 private-label) | ✅ extractor + fixture + 14 tests + scorecard 79/100 (medium — source ships no coords/hours/native storeId) | 2026-05-21 | [dabur-beverages.md](./dabur-beverages.md) |
| **Hindustan Coca-Cola Beverages (HCCB)** (Coca-Cola bottler — being divested to Jubilant Bhartia 2024) | hccb.in (locator on `/contact-us`) | 1 (HTML mobile-infocard + accordion + map-marker layers parsed) | guess-paths `/our-plants`, `/plants`, `/factories`, `/manufacturing`, `/about`, `/our-locations` all soft-404 (HTTP 200 with `<title>404 \| HCCB</title>`); NOT WordPress (`/wp-json/wp/v2/pages` returns the 404 page); sitemap.xml exhaustive but no plant-URL — canonical is the `/contact-us` "Where Our Beverages Come to Life" section | none | 16 plants across 9 states; 12 desktop-visible + 4 infocard-only (Aryana, Kanchenkanya, Sanand, Raninagar — divestment-in-flight pattern); matches FY24 ~16 disclosure exactly; **direct structural sibling to VBL** (39 plants — PepsiCo's bottler) | ✅ extractor + fixture + 9 tests + scorecard 71/100 (medium — capped by source shipping no street/pincode/coords/phone/hours) | 2026-05-22 | [hccb.md](./hccb.md) |

### Banking — branch + ATM locators

| Brand | Domain | Tier | API host | Protection | Coverage | Status | Last verified | Addendum |
|---|---|---|---|---|---|---|---|---|
| **HDFC Bank** | hdfcbank.com (locator at `www.hdfc.bank.in/branch-locator`) | 2 (AEM JSON selector-suffix) | www.hdfc.bank.in `/content/hdfcbankpws/api/branchlocator.{State}.{City}.json` | mild (CloudFront, plain Chrome UA passes) | Goa: 80 branches + 87 ATMs (national ~8,700 + ~18,000) | ✅ extractor + fixture + 14 tests + scorecard 95/100; branches & ATMs returned in ONE payload distinguished by `type` | 2026-05-21 | [hdfc-bank.md](./hdfc-bank.md) |

### Sales / distribution — electronics retail + dealer networks

| Brand | Domain | Tier | API host | Protection | Coverage | Status | Last verified | Addendum |
|---|---|---|---|---|---|---|---|---|
| **Samsung India** | samsung.com (locator at `/in/storelocator/`) | 2 (direct JSON API) | searchapi.samsung.com `/v6/front/b2c/storelocator/list` | none on searchapi; Akamai geo-gate on `www.samsung.com` (blocks service-centre API from outside IN) | 1,173 unique locations = 875 Experience Store + 246 Brand Store (SmartPlaza) + 52 Others (dealer); service centres parked (geo-gated) | ✅ extractor + fixture + 15 tests + scorecard 76/100 (medium); THREE brandTypes in ONE payload distinguished by `brandTypeCode` (E/B/O) | 2026-05-21 | [samsung-india.md](./samsung-india.md) |
| **Apple India Authorised** | apple.com (locator at `locate.apple.com/in/en/`) | 2 (direct JSON API on same origin as SPA) | locate.apple.com `/api/v1/grlui/in/en/sales` (GET; POST 405 at LB) | none (plain Chrome UA + Referer; same-origin) | 3,998 unique locations = 5 Apple-owned retail (BKC, Borivali, Saket, Noida, Koregaon Park) + 3,806 authorised dealers (APR + Apple Shop + plain authorised resellers) + 187 Apple Authorised Service Providers | ✅ extractor + 2 fixtures + 16 tests + scorecard 75/100 (medium); THREE locationTypes in ONE payload distinguished by `storeTypes` badge + `salesProducts`/`serviceProducts` presence | 2026-05-21 | [apple-india-authorised.md](./apple-india-authorised.md) |
| **Sony India** | sony.co.in (locator on a SEPARATE TLD: `locator.sony/en_IN/`) | 2 (direct JSON API; ONE call per channel) | locator.sony `/api/v1/dealers?orgLevel2=consumer&orgLevel3={smk\|customer_support}&min/maxLat&min/maxLng&limit&locale=en_IN` | **AkamaiGHost** (requires full Chrome `sec-ch-ua*` + `Sec-Fetch-*` + gzip headers; no IP geo-gate); **all `sony.co.in/**` paths 403/404** | 468 = 217 retail (112 Sony Centers + 87 Alpha Flagship + 16 Sony Exclusive + 2 Camera Lounge) + 251 authorised service centres | ✅ extractor + 2 fixtures + 20 tests + scorecard 78/100 (medium); **CRITICAL: omit `orgLevel4` — its presence empties the response**; India-wide bbox returns full channel in 1 call | 2026-05-21 | [sony-india.md](./sony-india.md) |
| **OPPO India** (BBK Electronics) | oppo.com (locator on `support.oppo.com/in/service-center/`) | 2 (direct JSON API, two-step) | `ind-sow-cms-app.oppo.com/oppo-api/siteInfo/v1/querySiteInfoByArea` (POST, province-scoped) | none on GCSM gateway; plain Chrome UA + Referer to support page passes | 596 authorised service centres across all 28 Indian state/UT provinces; locator surfaces SERVICE_SITE only — OPPO India has NO brand-store retail estate (online-only direct sales) | ✅ extractor + 2 fixtures + 15 tests + scorecard 76/100 (medium); two-step flow (divisions tree → province sweep), siteCode dedup | 2026-05-21 | [oppo-india.md](./oppo-india.md) |
| **Realme India** (BBK Electronics) | realme.com (locator on `www.realme.com/in/support/store-address`) | 2 (direct JSON API, single GET) | `api.realme.com/in/offlineStore/page/list?page=1&limit=10000` (GET; full corpus in one call) | none on Realme gateway; plain Chrome UA + Referer to support page passes | 131 retail stores across 23 states/UTs (1 Coco + 1 Flagship + 7 Experience + 59 Smart + 63 Realme/Mini); locator surfaces STORE ONLY — service-centre endpoint exists but returns 0 records; **DOES NOT share OPPO's GCSM** (probed `*-sow-cms-app.realme.com` — unreachable; Realme runs its own first-party `api.realme.com`, same pattern as sibling Vivo) | ✅ extractor + 3 fixtures + 16 tests + scorecard 76/100 (medium); single GET, id dedup, numeric storeType (12-16) → human labels | 2026-05-22 | [realme-india.md](./realme-india.md) |
| **Xiaomi India** (Mi) | mi.com (locators at `/in/service/mihome/`, `/in/service/authorized_stores/`, `/in/service/service-center/`) | 2 (three direct JSON/JSONP APIs across three sub-domains) | `store.mi.com/in/mihome/getdata` + `store.mi.com/in/authorizedstore/getdata` + `in-go.buy.mi.com/in/serviceplus/api/fos/public/v1/mi-store/near-by-asc` | none on data sub-domains; `www.mi.com` is Akamai-fronted but data hosts are open; service-centre endpoint uses public Basic-auth credential hard-coded in the React bundle | 2,033 unique = 58 Mi Home (store) + 1,026 Mi Authorized (dealer) + 949 Service Centres | ✅ extractor + 3 fixtures + 20 tests + scorecard 83/100 (high); **THREE channels surfaced by THREE separate endpoints across THREE Xiaomi sub-domains** | 2026-05-21 | [xiaomi-india.md](./xiaomi-india.md) |
| **Vivo India** (BBK Electronics) | vivo.com (locators at `vivo.com/in/where-to-buy` + `vivo.com/in/support/service-center`) | 2 (direct JSON via jQuery POST) | `vivo.com/in/retailers/queryByCondition` (all Exclusive Stores in one call, regionId=1) + `vivo.com/in/support/queryServiceCenterByCondition` (city-filtered, REQUIRES regionCode/stateCode/cityCode triple) | none — plain Chrome UA + Referer + `X-Requested-With: XMLHttpRequest` passes | 1,301 locations = 589 vivo Exclusive Stores + 712 authorised service centres across 32 states / 546 cities; **inline `regionList` catalogue** ships in the service-center HTML | ✅ extractor + 3 fixtures + 17 tests + scorecard 73/100 (medium); **DIFFERENT stack from sibling OPPO** despite shared BBK parent — Vivo serves everything in-house at vivo.com/in/*; param-name mismatch between the two endpoints (retailers use Id, support uses Code) | 2026-05-21 | [vivo-india.md](./vivo-india.md) |
| **OnePlus India** (BBK Electronics) | oneplus.in (locators on `service.oneplus.com/in/service-center` + `www.oneplus.in/experience-and-retail`) | 2 (BBK API, shared with OPPO) + 1 (inline HTML literal) | `ind-sow-cms.oneplus.com/oppo-api/siteInfo/v1/...` province sweep — **same `oppo-api` URL path as sibling OPPO, just on the oneplus.com host** + `www.oneplus.in/experience-and-retail` inline `var _store_data` JS object literal (six retail buckets) | none — plain Chrome UA passes both endpoints; no warmup, no cookies | 1,043 locations = 905 service centres (620 SERVICE_SITE + 275 OUTSOURCE_POINT + 10 SALE_AND_SERVICE_SITE) + 96 OnePlus-owned retail (Exclusive 38 + Authorized 33 + Kiosks 18 + Mini-store 7) + 42 multi-brand dealer (Croma 21 + Reliance Digital 21) | ✅ extractor + 4 fixtures + 27 tests + scorecard 76/100 (medium); **CONFIRMS BBK API is shared between OPPO and OnePlus** (same envelope, same siteCode/siteType/operationTime schema, only the gateway host differs). Vivo and Realme (also BBK) are outliers with in-house stacks. yaml's `/service/store-locator` is 404 — canonical is `service.oneplus.com/in/service-center`. | 2026-05-22 | [oneplus-india.md](./oneplus-india.md) |
| **LG India** (LG Electronics) | lg.com (locator at `/in/support/contact-us/locate-repair-center/`) | 2 (TWO direct JSON APIs via AEM `ncms` proxy) | `www.lg.com/ncms/api/v1/support/proxy/retrieveFindServiceCenterList?locale=IN` (service centres) + `/ncms/api/v1/proxy/brandshop-wtbDistributorData?locale=IN` (brand shops) | none — public `X-Lge-Application-Key` + `X-Lge-System-ID: LGCOM_AEM` headers embedded in HTML; plain Chrome UA + Origin/Referer passes | 824 outlets = 637 service-centre (ASC + DSC) + 185 store (Best Shop / LG Shoppe / LG Best Shop / LG Exclusive / BS) + 2 dealer; ~3.3% of LG-cited 25k touchpoints (~24k DSP channel partners unlisted in public locator) | ✅ extractor + 2 fixtures + 19 tests + scorecard 79/100 (medium); MIXED locationType across TWO APIs — ONE nationwide call returns all service centres, per-state sweep for brand shops; `pscDiv` (ASC/DSC) + `distributorTypeCode` (Best Shop / LG Shoppe / DEALER / ...) preserved in `_extra` | 2026-05-21 | [lg-india.md](./lg-india.md) |
| **TCL India** (TCL Tech, HK-listed) | tcl.com (locator at `/in/en/support/maps`) | 2 (ABP / ASP.NET CRM — country/province/city GUID sweep then `Account/FindNearestASPService`) | `cseleinind.tcl.com:8081` (TCL India customer-service ELE backend, on MS Azure) | none (plain Chrome UA + Origin/Referer passes; backend is 10-15s/call slow so 8-way `Promise.all` is required) | service-centre only (TCL India has no public retail / dealer locator — yaml's `/service/find-service-center` is 404; canonical is `/support/maps`); India-wide sweep across 38 real provinces × ≤50 cities (Goa converged at 8 ASPs in 30 cities) | ✅ extractor + 5 fixtures + 17 tests + Phase D sweep (50-cities-per-province cap, 8-way parallel) — see addendum for live count | 2026-05-22 | [tcl-india.md](./tcl-india.md) |
| **Indian Oil (IOCL)** | iocl.com / sdms.indianoil.in | 2 (pipe/CSV via POST, session-scoped) | `sdms.indianoil.in/PumpLocator/MapLocations` (warmup via `routeROs.jsp`) | Sucuri on iocl.com (irrelevant — locator on sdms is unprotected) | Goa: 8 records via fixture (national ~37k retail + ~12.5k LPG + ~6.5k KSK = ~56k) | ⚠ **PARKED — backend in scheduled maintenance** (state-wise JSPs return inline `exception.jsp` redirect). Extractor + parser + 14 tests + Phase D pipeline ready; fixture-backed scorecard 88/100 high | 2026-05-21 | [iocl-indian-oil.md](./iocl-indian-oil.md) |

### Indian company filings (MCA data — registration, directors, financials)

| Site | Domain | Tier | Discovery | Protection | Coverage | Status | Last verified | Addendum |
|---|---|---|---|---|---|---|---|---|
| **Tofler** | tofler.in | 1 (SSR HTML + cheerio) | `/companylist/{state}/pg-{N}` → 30 per page; detail at `/{slug}/company/{CIN}` | none (plain Chrome UA passes) | Delhi: ~270K-285K companies across ~9,000+ pages; **all Indian states available** | ✅ list extractor + detail extractor + 2 fixtures + 34 tests + live smoke test (90 list + 3 detail) | 2026-05-22 | [tofler.md](./tofler.md) |
| **Zauba Corp** | zaubacorp.com | n/a (403 on all list endpoints) | `/companies-list/city-{CITY}`, `/company-by-address/...` | WAF blocks all automated access (403 on Chrome UA) | n/a | ❌ **BLOCKED** — every list/detail URL returns 403 even with Chrome UA + Accept-Language; needs Playwright stealth or residential proxy | 2026-05-22 | (parked) |

### Industry associations — member directories

| Site | Domain | Tier | Discovery | Protection | Coverage | Status | Last verified | Addendum |
|---|---|---|---|---|---|---|---|---|
| **RAI** (Retailers Association of India) | rai.net.in | 1 (SSR HTML table + cheerio) | `/our-members.php?page={N}` → 20 per page, 107 pages | Imperva (passive — cookies set, no challenge) | 2,127 members across 232 cities; 6 membership categories (Core 87%, Associate 9%, Real Estate, Affiliate, Founder, Academic) | ✅ extractor + 2 fixtures + 19 tests + live sweep (2,127 records) | 2026-05-22 | [rai-members.md](./rai-members.md) |

**Status legend:**
- ✅ — production extractor + fixture + tests exist
- ⚠ recon only — methodology validated, extractor not built
- ❌ blocked — site protection beyond current toolkit

## Sites to recon next (parked candidates)

- **JustDial commercial** — B2B-style cards similar to IndiaMART.
- **OLX commercial / car / job / electronics** — same site as OLX, different verticals — test if our OLX addendum transfers.
- **Zauba Corp** — requires Playwright stealth or residential proxy to bypass WAF. Same MCA data as Tofler but with different presentation.

## Site addendum template

When you finish recon on a new site, create `sites/<site>.md` using this template:

```markdown
# <Site Name>

**Domain:** <example.com>
**Vertical:** <India real estate / B2B / classifieds / etc.>
**Last verified:** YYYY-MM-DD
**Tier:** 1 / 2 / 3
**Framework:** <Custom React SSR / Next.js / Nuxt / etc.>
**Protection:** <none / Akamai / Imperva / Cloudflare / DataDome>

## URL patterns

| Page type | Pattern | Example |
|---|---|---|
| SRP | ... | ... |
| Detail | ... | ... |
| Dealer/profile | ... | ... |

## Hydration payload

- **Location:** `window.X` / `<script id="...">` / `application/ld+json`
- **Source name (as reported by `extractAllHydrationPayloads`):** ...
- **Schema (key paths):**
  - `payload.foo.bar` → price
  - `payload.foo.baz` → area

If multiple payloads exist (e.g. `__NEXT_DATA__` + JSON-LD), list each and what's canonical for what.

## Extraction strategy

Why Tier N. Stealth requirements (cookies, warmup, mouse). Pacing. Whether one or multiple page types are needed.

## Field inventory

| Field | Page type | Source (JSON path / DOM selector) | Notes |
|---|---|---|---|
| price | detail | `prop_data.Price` | INR/month |
| ... | ... | ... | ... |

## Files in this repo

- **Extractor:** `scrapers/<site>-test.ts` (smoke), `scrapers/<site>.ts` (production)
- **Fixture:** `__tests__/fixtures/<site>-detail.html`
- **Tests:** `__tests__/<site>.test.ts`, `__tests__/<site>-detail.test.ts`
- **Recon demo:** `scrapers/recon-<site>.ts` (if applicable)

## Pagination

URL param? Cursor? Infinite scroll? Terminator?

## Gotchas

The specific things that bit you. Be concrete.

- e.g. "First request succeeds without cookies; subsequent requests get 403. Need session warmup."
- e.g. "POI list elements have `__badgeParent` class — false-positives in status badge audit."
```
