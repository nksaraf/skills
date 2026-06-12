# Reference: Indian real estate domain hints

Quick-start guidance for common sites in the Indian commercial-rental space. Use as starting points; always validate in Phase 3 of recon.

For sites where we have a full case study, see `../sites/<site>.md` — those are richer.

## 99acres.com → see [sites/99acres.md](../sites/99acres.md)

Fully reconned. Production extractor + fixtures + tests exist.

## MagicBricks.com → see [sites/magicbricks.md](../sites/magicbricks.md)

Reconned. Extractor not built yet but `window.SERVER_PRELOADED_STATE_` schema mapped.

## Housing.com → see [sites/housing.md](../sites/housing.md)

Blocked by Imperva on deep URLs. Needs XHR capture layer to make progress.

## CommonFloor.com → see [sites/commonfloor.md](../sites/commonfloor.md)

Reconned. Incremental window.X.foo build pattern. JSON-LD Product is canonical on detail.

## IndiaMART (dir.indiamart.com) → see [sites/indiamart.md](../sites/indiamart.md)

Reconned. Next.js + JSON-LD. Best for warehouses (`/impcat/warehouse-on-rent.html`).

## OLX.in → see [sites/olx.md](../sites/olx.md)

Reconned. Cross-vertical classifieds. JSON-LD Product / ItemList is canonical.

## NoBroker.in (not yet reconned)

- **Architecture:** SPA (React)
- **Raw HTML:** Contains skeleton only — no listing data
- **Strategy (expected):** Browser required. Intercept XHR calls made during initial page load.
- **API:** Internal API calls triggered on load; capture with `page.on("response", ...)`
- **Tier:** 3 (browser + API interception)
- **Same XHR-capture limitation as Housing.com.** Whichever site we attempt first will drive the XHR-capture primitive.

## PropertyWala (not yet reconned)

- **Architecture (suspected):** Older Indian portal, likely SSR
- **Worth checking:** the incremental `window.X.foo` pattern (CommonFloor-style)

## Common patterns across this vertical

- **City URL slugs:** lowercase, dash-separated (`delhi`, `new-delhi`, `mumbai`, `gurgaon`, `noida`, `bangalore`, `pune`, `hyderabad`, `chennai`, `kolkata`, `ahmedabad`)
- **Categories:** `warehouse`, `showroom`, `office-space`, `commercial-shops`, `commercial-land`, `agricultural-farm-land`, `industrial-building`
- **Per-page count:** typically 20-30 listings (27 on 99acres, 20 on MagicBricks/IndiaMART)
- **Pagination:** almost always `?page=N`, 1-indexed; rarely cursor-based
- **Phone numbers:** masked in initial payload, revealed via login + CAPTCHA — not pursuable without auth
- **Posted-by:** Owner / Builder / Dealer / Agent — categorical; filter by this for owner-direct deals
- **Status flags to watch:** Verified, Featured, Premium, NOT AVAILABLE / RENTED OUT, Ready to move, Under construction

## Pricing conventions (see [`../patterns/currency-rendering.md`](../patterns/currency-rendering.md))

- All sites use `Lac` and `Cr` for prices above ₹99,999
- Per-sqft prices (`₹46/sqft/month`) are derived (Price ÷ Area), not always primary
- Inclusive vs exclusive of charges: separate boolean
