# Pattern: Recovering coordinates from Google Maps embeds

When a store-locator API returns blank `lat`/`lng` but provides a Google Maps embed URL, the coordinates are **already in the URL** — just encoded.

## The pattern

Google Maps embed URLs use a `pb=` parameter to encode "place bounds". Inside the `pb=` string:

```
!2d{longitude}!3d{latitude}
```

Note the order: **longitude first (2d), then latitude (3d)** — the opposite of the conventional (lat, lng) tuple. This is consistent across all Google Maps embed URLs.

## Example

From a Zudio record:

```
https://www.google.com/maps/embed?pb=!1m14!1m8!1m3!1d237.0!2d83.4123683!3d18.110558!3m2!1i1024!2i768!4f13.1!3m3!1m2!1s0x3a3be5563971fafb%3A0x81332e38352919d8!2sSurya%20Mobile!5e0!3m2!1sen!2sin
```

Parse: `!2d83.4123683!3d18.110558` → **lng = 83.4123683, lat = 18.110558**.

## When this matters

If a brand stores its locator data with coordinates as separate string fields (`store_latitude`, `store_longitude`), they may be blank/null due to manual data entry. But the brand's UI still needs to render a map — so they generate the embed URL via a Google geocode at some point. **The embed URL has the canonical coordinates the map renders.**

Real-world impact, this session:

| Brand | Direct lat/lng coverage | After embed-URL recovery |
|---|---|---|
| Westside | 70% (209/300) | **90% (272/300)** |
| Zudio | **0% (1/555)** | **99% (553/555)** |

## The helper

Lives in `scrapers/retail/westside.ts` (the Trent extractor):

```typescript
/**
 * Parse coordinates from a Google Maps embed URL. The `!2d{lng}!3d{lat}`
 * pattern is part of Google's `pb=` parameter encoding (Place Bounds).
 * Order is lng-then-lat (NOT lat-then-lng).
 */
export function parseGoogleEmbedCoords(embedUrl: string): { lat: number; lng: number } | null {
  const m = embedUrl.match(/!2d(-?\d+\.\d+)!3d(-?\d+\.\d+)/)
  if (!m) return null
  const lng = parseFloat(m[1])
  const lat = parseFloat(m[2])
  if (!Number.isFinite(lat) || !Number.isFinite(lng)) return null
  return { lat, lng }
}
```

Usage in an extractor:

```typescript
let lat = toNumber(raw.store_latitude)
let lng = toNumber(raw.store_longitude)
if ((lat == null || lng == null) && raw.google_emmbeded_code) {
  const fromEmbed = parseGoogleEmbedCoords(raw.google_emmbeded_code)
  if (fromEmbed) {
    lat = lat ?? fromEmbed.lat
    lng = lng ?? fromEmbed.lng
  }
}
```

Always prefer the explicit fields when present — they're usually canonical. Use the embed only as a fallback.

## Related sources of geo data

Beyond the Google embed URL, also check:

- **Apple Maps embed URLs** — different encoding: `?coordinate=<lat>,<lng>&...` (lat first)
- **OpenStreetMap iframes** — `bbox=<minLng>,<minLat>,<maxLng>,<maxLat>&marker=<lat>,<lng>`
- **Static map image URLs** — `&markers=<lat>,<lng>` (lat first)
- **`href` to `https://maps.app.goo.gl/<short>`** — short-link; would need to follow the redirect to get coords (more network, last-resort)
- **Schema.org `Place` / `GeoCoordinates`** — `{"latitude": "...", "longitude": "..."}` (explicit)

The pattern generalises: when a record has a "view on map" link or embed, the coordinates are almost always in the URL — read the URL.

## When NOT to use

- The embed URL may be **stale** vs the brand's actual location (if the brand moved a store but didn't regenerate the embed). The explicit `lat`/`lng` fields, when populated, are more current. Prefer them.
- Different brands embed with **different precision** — Westside has 6 decimal places (~10 cm), some brands have 4 (~10 m). Don't over-trust precision.
- Some embeds point at a **landmark near the store** (e.g. the nearest mall entrance) rather than the store itself. For block-precision analysis this matters; for city-level analysis it doesn't.

## Sites where this pattern applies

- **Westside / Zudio** (Trent) — confirmed, this is how we recovered Zudio's coordinates
- **Other Indian retailers with Google Maps integration** — likely candidates: Reliance Retail brands, Pantaloons, V-Mart, Croma
- **Any locator using Google Maps embed iframe** — universal across the Google Maps Embed API
