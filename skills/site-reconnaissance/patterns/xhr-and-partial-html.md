# Pattern: XHR-driven SPAs and partial HTML sites

Sites where the data isn't in the initial HTML — it arrives via subsequent network calls. The schema walker and visual audit see nothing useful because there's nothing in the document to look at. **The data is in the network responses.**

## When this applies

- **Pure XHR SPAs.** Initial HTML is a shell (`<div id="root"></div>` + script bundle). Data fetched via `/api/*` calls fired during or after `DOMContentLoaded`. Examples: Housing.com (deep URLs), NoBroker (expected), any React/Vue/Angular CSR app.
- **Partial-HTML sites.** Initial page is SSR'd, but navigation between pages or filter changes fetch HTML *fragments* via XHR (Turbo, HTMX, morphdom). The response body is `text/html`, not JSON.
- **PHP/jQuery hybrids.** SSR'd initial page, then `$.ajax` calls populate dropdowns, dealer details, or "load more" sections. Common on older Indian portals.

## Recognition

Run Phase 0 (HTTP) and Phase 0a (page-type enumeration). Signals you're in this category:

1. **Initial HTML is suspiciously small** (under 50 KB) and you can't find a target value via grep.
2. **`extractAllHydrationPayloads` returns empty or only metadata** (e.g. Housing.com returns `window.__APP_CONFIG__` build metadata, nothing else).
3. **The DOM is mostly empty until JavaScript runs.** `curl` returns a shell; the rendered page (Phase 1 browser recon) has content.
4. **`auditPage` returns 0 label-value pairs** despite the page clearly displaying data.
5. **Network tab shows JSON XHR responses to `/api/`, `/graphql`, `/_next/data/`, or similar.**

## The XHR capture primitive

Use `@vinxi/scraper/recon` (`xhrCapture`). Registers a Playwright response listener BEFORE you navigate, collects matching responses, returns them in the same `HydrationPayload` shape `walkSchema` already understands.

### Recipe

```typescript
import { startXhrCapture, capturedToPayloads, groupByEndpoint }
  from "@vinxi/scraper/recon"
import { walkSchema, filterLeaves } from "@vinxi/scraper/recon"

const page = await ctx.browser.newPage({ deviceScaleFactor: 1 })

// 1. START THE LISTENER FIRST. If you navigate before this, you miss
//    the initial-load XHRs.
const capture = await startXhrCapture(page, {
  urlMatcher: /\/api\/|\/graphql|\/_next\/data\//,
  maxResponses: 50,
})

// 2. Navigate. networkidle waits for the network to settle so the
//    initial-load XHRs complete.
await page.goto(targetUrl, { waitUntil: "networkidle", timeout: 60_000 })

// 3. Humanize a bit; sometimes lazy XHRs fire later.
await ctx.human.moveMouse(page)
await ctx.human.pause(2000, 4000)

// 4. Stop and inspect.
const responses = capture.stop()
ctx.log.info("captured", { count: responses.length })

// 5. Group by endpoint for an at-a-glance summary.
const groups = groupByEndpoint(responses)
for (const [endpoint, batch] of groups) {
  ctx.log.info("endpoint", { endpoint, hits: batch.length })
}

// 6. Walk each JSON response like any other payload.
for (const p of capturedToPayloads(responses)) {
  const leaves = walkSchema(p.data, { arraySampleSize: 2, maxDepth: 6 })
  ctx.log.info(p.source, { total: leaves.length, pricing: filterLeaves(leaves, ["price","rent"]).length })
}
```

### What you do next

Once you have the JSON responses, the recon flow looks like a regular Tier 2 (API) scrape:

1. **Identify the canonical endpoint.** Of N responses, which one carries the actual data (price, area, location)? Usually 1-3 endpoints do most of the work; the rest are ads, analytics, or recommendations.
2. **Replay it via `curl`.** Get the URL, headers (especially `Authorization`, `X-API-Key`, `Cookie`), query params, and try to hit it directly. If it works, the production scraper becomes Tier 2 (HTTP + JSON) — far faster than running a browser per request.
3. **Document the endpoint in the site addendum.** URL pattern, required headers, pagination shape (cursor / offset / page), rate limits observed.
4. **If replay fails (auth-coupled, fingerprinted),** stay on Tier 3 (browser) but call the API via `page.evaluate(() => fetch(...))` to inherit the page's session.

## Partial HTML capture

Some sites (especially older PHP / Turbo / HTMX) fetch HTML fragments via XHR. The response body is `text/html`. Enable with:

```typescript
const capture = await startXhrCapture(page, {
  urlMatcher: /\/ajax\/|\/partials\//,
  captureHtmlPartials: true,
})
```

The captured response is the HTML string. Parse it with `cheerio.load(response.data)` and extract from that — same selectors you'd use on the full page, but applied to just the fragment.

## Pitfalls specific to this category

### Authenticated endpoints

The XHR call may carry an `Authorization` header or a session cookie. If you replay via `curl` outside the browser session, you'll get 401/403. Two options:

- **Capture the session cookie from the browser** and pass it to `ctx.fetch` for subsequent calls.
- **Stay on Tier 3** and call the API via `page.evaluate(() => fetch(...))` so the browser's session is used.

### Token expiry

JWT-style tokens visible during recon may expire in minutes. Don't bake them into the production scraper. Either refresh from the landing page on each run, or use longer-lived session cookies.

### Streaming / SSE responses

Some XHRs are server-sent-events or chunked transfer-encoded. The current `startXhrCapture` reads the full body — if the connection stays open (SSE feed), the capture will hang waiting. Skip these endpoints via `urlMatcher` exclusion.

### Multiple firings per page

A single page navigation may fire the same endpoint 5+ times (debounced searches, infinite scroll, prefetch). `groupByEndpoint` collapses them; pick the most-recent or largest response as canonical.

### Cookie / Imperva session warmup before XHRs work

On protected sites (Housing.com), the deep URL fails with 406 before the XHR even gets a chance to fire. The mitigation is **homepage warmup** (cookie dwell + mouse activity) BEFORE deep navigation. The XHR capture starts AFTER warmup but BEFORE the deep `goto`.

```typescript
// Warmup
await page.goto("https://www.<site>.com/", { waitUntil: "domcontentloaded" })
await ctx.human.moveMouse(page)
await ctx.human.pause(8000, 15000) // long, deliberate dwell

// Now start capture and navigate to the protected URL
const capture = await startXhrCapture(page, { urlMatcher: /\/api\// })
await page.goto(deepUrl, { waitUntil: "networkidle" })
// ...
```

This pattern is the next thing to test on Housing.com.

## Decision flow

| Initial HTML has data? | Has hydration payload? | Approach |
|---|---|---|
| ✅ Yes | ✅ Yes | Tier 1 (cheerio) or Tier 1 + payload extraction (best of both) |
| ✅ Yes | ❌ No | Tier 1 (cheerio) |
| ❌ No | ✅ Yes (XHR responses) | **Tier 3 + XHR capture → maybe downgrade to Tier 2 (direct API call) for production** |
| ❌ No | ❌ No | Verify by viewing the rendered DOM; if data appears post-load, this is XHR-only; otherwise the site uses canvas/image rendering and is out of scope |

## Sites currently in this category

| Site | Status |
|---|---|
| Housing.com (deep URLs) | Imperva blocks deep URLs even with stealth. Needs warmup-then-XHR-capture pattern. See [`../sites/housing.md`](../sites/housing.md). |
| NoBroker | Not yet reconned. Per CLAUDE.md domain hints: SPA, XHR-only, browser+API interception required. |

When you successfully scrape one of these, **update the site addendum AND this pattern doc with the canonical endpoint, header requirements, and any session-handling gotchas.**
