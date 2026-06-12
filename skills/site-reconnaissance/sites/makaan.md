# MakaaN

**Domain:** makaan.com
**Vertical:** India real estate (formerly residential + commercial rental/sale)
**Last verified:** 2026-05-22
**Tier:** n/a (site is defunct)
**Framework:** n/a
**Protection:** n/a

## Current status: DEFUNCT — redirected to housing-buyer-redirect (502/504)

**MakaaN.com is fully shut down.** Every URL on `www.makaan.com` — without exception, including `/robots.txt` and `/favicon.ico` — returns a blanket HTTP 301 to `https://origin.makaan.com/housing-buyer-redirect`. That backend returns HTTP 502 or 504. No listing data is accessible.

The redirect target path name `housing-buyer-redirect` confirms the site's buyer traffic was migrated to Housing.com (both owned by Elara Technologies / PropTiger group). The origin application server behind that endpoint is offline.

**Do not attempt to build a scraper for MakaaN.** There is no data to scrape.

## Evidence of shutdown (2026-05-22)

| Request | HTTP Response | Notes |
|---|---|---|
| `GET https://www.makaan.com/` | 301 → `origin.makaan.com/housing-buyer-redirect` | Blanket redirect |
| `GET https://www.makaan.com/robots.txt` | 301 → same | Even static files redirect |
| `GET https://www.makaan.com/favicon.ico` | 301 → same | Same pattern |
| `GET https://www.makaan.com/delhi-real-estate/rent` | 301 → same | SRP paths redirect too |
| `GET https://www.makaan.com/api/v1/search?...` | 301 → same | API paths redirect too |
| `GET https://origin.makaan.com/housing-buyer-redirect` | 502 Bad Gateway | Backend offline |

The pattern is consistent across all tested paths: every request hits the same nginx 301 rule, and the destination application server is unreachable (502/504 loop).

## Background

MakaaN.com was an Indian real estate portal founded in 2011. In April 2015, PropTiger.com acquired it. PropTiger and Housing.com are both owned by Elara Technologies Private Ltd., Singapore. The three brands — PropTiger, MakaaN, and Housing.com — were run as separate portals within the same group. At some point (evidence suggests late 2024 or 2025), MakaaN was wound down and buyer traffic was redirected to Housing.com.

The SSL certificate for `*.makaan.com` is still valid (issued Dec 2025, expires Jan 2027) and the DNS points to AWS (13.205.40.118, 3.7.235.238), indicating the infrastructure is maintained but the application layer is off.

## Sibling-site relationship

MakaaN shared the PropTiger/Housing.com backend infrastructure (Elara Technologies group). However, since MakaaN is defunct, there is no API to share with Housing.com. Any historical API endpoints that may have been shared (PropTiger API, MakaaN search API) are now offline.

## Recommendation

Use **Housing.com** for Delhi rental data instead. Housing.com (`scrapers/housing.ts`) covers the same market with a working production extractor. See `sites/housing.md` for the current status and bypass strategy.

## Files in this repo

- No extractor built — site is defunct.
- This addendum documents the shutdown status for future reference.
