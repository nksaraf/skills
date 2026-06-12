# Reference: Tier selection

Use the simplest tier that covers all target data.

## The three tiers

| Tier | Transport | When to use | Selector/path style |
|---|---|---|---|
| **Tier 1** | `ctx.fetch` + cheerio | All data in raw HTML (SSR sites, WordPress, Shopify, static pages) AND no protection | CSS selectors on HTML |
| **Tier 2** | `ctx.fetch` (JSON API) | Clean API discovered in traffic; data cleaner than HTML | JSON path on API response |
| **Tier 3** | `ctx.browser` (Playwright) | SPA, anti-bot blocks HTTP, data only available after JS executes, OR site has Akamai/Imperva/Cloudflare protection | Playwright locators or intercepted API responses |

**Prefer Tier 1 > 2 > 3 in that order. Never use a browser if HTTP can do the job — unless protection forces it.**

## Hybrid pattern (common in our codebase)

```typescript
// Tier 1 for pagination discovery, Tier 2 for data
const indexPage = await ctx.fetch(indexUrl)
const $ = cheerio.load(await indexPage.text())
const ids = $(".listing").map((_, el) => $(el).attr("data-id")).get()

for (const id of ids) {
  const detail = await ctx.fetch(`https://example.com/api/listings/${id}`)
  ctx.pushData(await detail.json() as Record<string, unknown>)
  await ctx.sleep(300)
}
```

## When the protection forces a tier upgrade

A site can have all its data in SSR'd raw HTML AND still require Tier 3 because of anti-bot protection. 99acres is the canonical example: every field is in `window.__initialData__` (Tier 1 territory in principle), but Akamai blocks the second request — so the production scraper must use Playwright Chromium for session persistence.

**Rule:** if Phase 0 sees protection signals, jump to Tier 3 regardless of where the data is.

## When the data forces a tier upgrade

Even without protection, a site can require Tier 3 because the data isn't in the HTML:

- React CSR / pure SPA: only a `<div id="root"></div>` in the SSR; data loaded after `DOMContentLoaded` via XHR
- Heavy client-side hydration: SSR has skeleton placeholders (`$--`) that JS fills in
- Auth-gated content: requires login flow that's easier to script with a real browser

## When NOT to upgrade tiers

- **Tier 2 looks tempting on Next.js sites** — `/_next/data/*.json` routes exist — but they're route-coupled and break on Next.js upgrades. Prefer parsing `__NEXT_DATA__` from the SSR'd HTML (Tier 1).
- **Don't use Playwright just because you've never seen the site before.** Run Phase 0 curl first; it's seconds of effort and saves hours of crawl time.
