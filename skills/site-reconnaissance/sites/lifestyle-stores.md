# Lifestyle (Landmark Group) — store locator

**Domain:** lifestylestores.com (locator subdomain: `stores.lifestylestores.com`)
**Vertical:** India retail (Landmark Group — Dubai parent; department-store competitor to Westside)
**Last verified:** 2026-05-20
**Tier:** 1 (HTML markers, very limited) **OR** 1 per-store detail page (full data, but ~150+ fetches)
**Framework:** **Singleinterface** (vendor `EThemeForLifestyleUpdated` — same SaaS used by Levi's Indonesia and many other brands)
**Protection:** main `www.lifestylestores.com` is behind Cloudflare; the locator subdomain `stores.lifestylestores.com` is unprotected

**Status: ⚠ recon only — extractor not built.** See "Why parked" below.

## URL patterns

| Page type | URL | Notes |
|---|---|---|
| Main shop (Cloudflare) | `https://www.lifestylestores.com/in/en/storelocator` | links out to the locator subdomain |
| Locator landing | `https://stores.lifestylestores.com/` | server-rendered list, but caps at 20 visible stores + 20 map markers |
| Per-store Home | `https://stores.lifestylestores.com/lifestyle-stores-<slug>-<id>/Home` | full store data (hours, phone, address, services) |
| Per-store Map | `https://stores.lifestylestores.com/lifestyle-stores-<slug>-<id>/Map` | lat/lng on this view |
| Per-store Products | `…/Products` | category/products carried |
| State/city filter (UI) | dropdowns post back to locator landing | filter is form-driven, returns paginated HTML |

## Hydration payload

The locator landing **does not** ship a single JSON blob. Instead:

- `<input id="mapMarkerJsonEncoded" type="hidden" value='[[name, lat, lng, icon], ...]'>` — 20 markers max on the homepage. Just name + coords. **No phone, no hours, no full address.**
- Per-store full data lives on `…/Home` — must be DOM-scraped per store.
- Filter dropdowns (`#customState`, `#customCity`, `#customLocality`) populate via state→city cascading, which posts back to the same locator landing URL and re-renders.

## Field availability

| Field | Landing page (markers) | Per-store /Home page |
|---|---|---|
| name | yes (in marker array) | yes |
| lat/lng | yes | yes (on /Map view) |
| address | no | yes (HTML) |
| phone | no | yes |
| hours | no | yes (per-day) |
| services / features | no | yes |
| store id | partially (in detail URL slug) | yes (URL `<slug>-<id>`) |

## Why parked (not full extractor)

A real Lifestyle scrape needs **all three of these**, which together push the effort past "fast win":

1. **State→city→locality cascading enumeration.** The landing-page filter is the only way to enumerate all stores (other than guessing slugs). The dropdowns are populated client-side from JS data + form posts. Need to either reverse-engineer the AJAX endpoint or browser-drive the dropdowns.
2. **Per-store detail crawl.** 150+ DOM-scraped detail pages — each one slow, each with a different layout subsection for hours/services. Singleinterface templates have minor variations.
3. **Pagination after filter.** A filtered list (e.g. "all Delhi stores") may return more than the 20 visible on landing; pagination state lives in cookies + form fields.

This is **possible** with Tier 3 (browser) plus a per-store Tier 1 cheerio extractor — but it's 4–6 hours of work, not the same Tier-2 quick-win as Uniqlo. Recommend doing it when there's appetite for a multi-day Singleinterface generaliser, since the same code would also unlock Levi's Indonesia, Reliance Trends, Croma, and other Singleinterface tenants.

## Concrete blockers

- **No "all stores" JSON endpoint.** Both `stores.lifestylestores.com/api/...` and `…/stores.json` 404.
- **AJAX endpoints for state/city dropdowns** are not visible in static HTML — need browser XHR capture to find them.
- **Per-store detail pages** require DOM parsing (no JSON blob there either, based on quick scan).

## What was confirmed (recon results)

- Locator subdomain is unprotected (no Cloudflare on `stores.`)
- 20 markers visible on landing with name + lat/lng (data preview attached below)
- Detail-page URL pattern is stable: `lifestyle-stores-<area-slug>-<city-slug>-<id>` where `<id>` is a 5-digit number (e.g. `68815`, `78620`, `92828`)
- Indian-state filter dropdown supports the same 36 states + UTs as Uniqlo's API
- "Singleinterface" branding visible in CSS/JS paths (`EThemeForLifestyleUpdated`) — this is a multi-tenant SaaS locator vendor

## Sample marker data (20 stores from landing)

```
Lifestyle Stores, Tagore Garden, New Delhi  28.642951, 77.1062771
Lifestyle Stores, Vasant Kunj 2, New Delhi  28.541371, 77.155145
Lifestyle Stores, Dwarka, Sector 21, New Delhi  28.552411, 77.058511
Lifestyle Stores, Sector 12, Faridabad  28.385478, 77.314301
Lifestyle Stores, MG Road, Gurugram  28.479451, 77.081087
... (15 more, all NCR + Punjab + UP region)
```

The fact that all 20 markers are in north India suggests the landing page is **region-paginated**, not a global cap — confirming that the full list lives behind filter selections.

## Files in this repo

- (none — recon only)
- Captured HTML for reference: see `/tmp/lifestyle-stores.html` from the recon session (not committed)

## Recommended next steps

1. Open `stores.lifestylestores.com` in Playwright with XHR capture, change the state filter to Maharashtra, capture the request — that reveals the AJAX endpoint.
2. Once the endpoint is known: probe whether it accepts a "all states" or empty filter to dump everything.
3. If not: enumerate via 36 state codes (same approach as Uniqlo).
4. Build a per-store /Home DOM extractor (cheerio + selectors for hours, phone, services).
5. The Singleinterface template generaliser is the long-tail prize — many Indian retailers use it.
