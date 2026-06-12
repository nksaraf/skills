# Pattern: Currency rendering

How the same price gets rendered differently across sites.

## Forms observed

| Form | Example | Sites | Detected by default audit? |
|---|---|---|---|
| Unicode rupee with number | `₹85,000`, `₹3 Lac`, `₹2.5 Cr` | 99acres, MagicBricks, OLX | ✅ Yes |
| Unicode with spaced suffix | `₹ 30,000`, `₹ 36,000` | 99acres similar listings, OLX | ✅ Yes |
| Naked number near rupee-icon font | `25,000` next to `<i class='cf-rupee'>` | CommonFloor | ❌ No — icon-font is not Unicode |
| Words-only ("Rs") | `Rs30,000`, `Rs. 30,000` | 99acres payload (`Price_Text`) | ⚠ partial — `$|₹|€|£|¥` regex misses `Rs` |
| Currency as JSON-LD `Offer.price` | `{"@type":"Offer","price":"36000","priceCurrency":"INR"}` | OLX (canonical), CommonFloor | n/a (structured, not text-extracted) |

## Indian-specific conventions

- **`Lac` = 1,00,000** (1 lakh = 100,000)
- **`Cr` = 1,00,00,000** (1 crore = 10,000,000)
- **`K` = 1,000**
- Numbers use Indian digit grouping: `1,00,000` not `100,000` (parsers must strip commas before `parseFloat`)
- "Negotiable" / "On request" / "Price on call" are valid values for price fields — don't assume numeric
- Per-unit prices: `₹46/sqft/month` is common — parse the unit too
- Inclusive vs exclusive of charges (`Charges not included`) is a separate boolean — see 99acres `electricityWaterCharges`

## Parser

The 99acres extractor (`scrapers/99acres-test.ts`) has a battle-tested `parsePrice()`:

```typescript
parsePrice("₹85,000 /month")  // { value: 85000, cadence: "month" }
parsePrice("₹3 Lac /month")    // { value: 300000, cadence: "month" }
parsePrice("₹2.5 Cr /month")   // { value: 25000000, cadence: "month" }
parsePrice("₹50 K")            // { value: 50000, cadence: null }
parsePrice("Rs30,000")         // { value: 30000, cadence: null }  (handled in detail extractor)
```

For a new site, lift this and adapt for the site's specific format. Don't try to make one parser work for all sites — the variations are small but real.

## Recognition during recon

If `auditPage().currencyMentions` returns 0 but you can see prices on screen, the site is using one of the non-Unicode forms:

1. Inspect the DOM around a visible price for icon-font classes (`*='rupee'`, `*='currency'`, `*='inr'`)
2. Check the payload for structured `Offer.price` / `Price` / `priceText` fields — usually present, often a cleaner extraction target than the DOM
3. If both fail, the price might be image-rendered (extremely rare on Indian sites; common on some classifieds for anti-scraping)

## Parked enhancement

`visualAudit.ts` could detect naked numbers adjacent to currency-name classes (`*='rupee' i`, `*='currency' i`) and infer the currency from the class name. CommonFloor would benefit. Not built yet because (a) JSON-LD is a cleaner channel on every site that uses icon-fonts, and (b) it's a single-site win for now.
