# Pattern: Sibling-brand probing

Multi-brand parent corporations often host all their brands at the same locator infrastructure with a `?type=<brand>` or `?brand=<slug>` parameter. **If you only scrape one brand, you miss the rest of the portfolio.** The scorecard's "sibling-brands" check exists to catch this.

## Why this matters

The first end-to-end retail extraction was Westside (Tata) — 300 stores, completeness score 100. We almost shipped. The validator flagged: "no sibling-brand probe performed". One follow-up curl returned: **Zudio has 578 stores at the same API**. That's **double the dataset** we'd have shipped without the probe.

This isn't a Tata oddity. The pattern is widespread:

| Parent | Brands at same locator? | Probe verified? |
|---|---|---|
| **Trent (Tata)** | Westside, Zudio, Westside Light, Misbu, Utsa | ✅ Westside + Zudio extracted; others not yet probed |
| **Inditex (Zara)** | Zara, Bershka, Pull&Bear, Stradivarius, COS, Massimo Dutti | Likely yes via Inditex group APIs |
| **LVMH** | Louis Vuitton, Sephora, Tiffany, Dior, ... | Each brand has its own domain but shared CDN patterns |
| **Reliance Retail** | Trends, AJIO, Reliance Digital, Smart Bazaar, ... | All on `relianceretail.com` infrastructure |
| **Aditya Birla Fashion** | Pantaloons, Allen Solly, Van Heusen, Louis Philippe, Peter England | Single locator app |
| **Fast Retailing** | Uniqlo, GU, Theory, Helmut Lang | Uniqlo on `map.uniqlo.com`; GU likely on `map.gu-global.com` |

## When to suspect

Run a sibling probe **before declaring recon done** if:

1. The brand is **owned by a publicly listed parent** (Trent, Inditex, LVMH, Reliance, ABFRL, Tata Group, Fast Retailing)
2. The API host is **brand-agnostic** (`customapp.trent-tata.com`, `api.intent.io`, generic SaaS like Singleinterface / Woosmap / Yext)
3. The URL has a brand-discriminator query param (`?type=`, `?brand=`, `?banner=`, `?ch=`)
4. The locator JS has a `getBrand()` / `BRAND_NAME` / `chain` variable

## How to probe

The probe is cheap — one curl per candidate. Take Westside as the example:

```bash
# 1. Find sibling brand names. Public sources:
#    - Parent company's "Our Brands" page
#    - Wikipedia parent infobox
#    - Annual report's brand list

# 2. For each candidate, try the API with the brand parameter swapped
for brand in westside zudio "westside-light" misbu utsa; do
  curl -s -A "Mozilla/5.0" \
    -H "Origin: https://www.trent-tata.com" \
    --compressed "https://customapp.trent-tata.com/custom/stores?type=$brand" \
    | python3 -c "
import json, sys
try: d = json.load(sys.stdin); print(f'$brand: {d.get(\"store_count\", \"?\")} stores')
except: print(f'$brand: parse error / no data')"
done
```

**Time investment: ~2 minutes per parent corporation. ROI: doubles or triples the dataset when it hits.**

## Documenting the probe

In the site addendum, always record:

```markdown
## What other [Parent] brands to probe next

Same API host (`customapp.example.com`) likely serves:
- **BrandA** — description
- **BrandB** — description
- **BrandC** — description

A 30-second probe per name. The pattern is documented at `../patterns/sibling-brand-probing.md`.
```

And in the scorecard context (`scrapers/validate-retail.ts`):

```typescript
siblingBrandsProbed: [
  { name: "Zudio", expectedCount: 578, extracted: true },   // confirmed extracted
  { name: "WestsideLight", expectedCount: 0, extracted: false }, // probed, 0 stores
]
```

This way the scorecard either:
- Confirms all probed siblings are accounted for (sibling-brands = 100 → high completeness)
- **Flags missed siblings explicitly** (sibling-brands = 10 → forces a recommendation)

## Failure modes

- **Brand not at the same API.** Many brands have separate infrastructure. The probe returns 404 / empty / different JSON shape. Document the negative result — still useful.
- **Brand exists but at a different host.** The brand-discriminator might be in the subdomain, not query param. Look at the brand's own locator HTML before assuming.
- **Brand exists but with a different schema.** Even if the same host, the data shape may differ. The Westside extractor parameterises on brand name but assumes the same field mapping; verify before declaring "same extractor works".

## Sibling probing for the brands in our index

Done:
- **Westside ←→ Zudio** ✅ (both extracted)

Parked candidates (probe next):
- **Mango ←→ Mango Kids, Mango HE, Mango Sport** (probe `shop.mango.com/in/en/stores?brand=...`)
- **Uniqlo ←→ GU** (try `map.gu-global.com/in/en/`)
- **Lifestyle ←→ Max Fashion, Home Centre, SPAR** (all Landmark Group)
- **H&M ←→ &Other Stories, Monki, COS** (also under H&M Group)

A focused 30-minute session probing these could 2-3x the retail dataset.
