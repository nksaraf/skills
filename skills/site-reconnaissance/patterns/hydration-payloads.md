# Pattern: Hydration payload locations

Modern sites embed their initial data into HTML so the first paint is fast. The data is the goldmine — one JSON.parse beats hundreds of CSS selectors. **Always look for a payload first.**

## Locations we've observed (by frequency)

### Tier 1: Standard / framework-canonical

These are well-known framework conventions. `extractAllHydrationPayloads` catches all of them.

| Location | Framework | Sites observed |
|---|---|---|
| `<script id="__NEXT_DATA__">` | Next.js | IndiaMART |
| `window.__NEXT_DATA__ = {...}` | Next.js | (alternative; same data) |
| `window.__NUXT__` | Nuxt.js | (not yet observed in our set) |
| `window.__INITIAL_STATE__` | Vue / Redux SSR | IndiaMART (parallel to __NEXT_DATA__) |
| `window.__APOLLO_STATE__` | Apollo GraphQL SSR | Medium (per CLAUDE.md) |
| `<script type="application/ld+json">` | schema.org / W3C standard | **OLX (canonical)**, CommonFloor (canonical for detail), 99acres (SRP item URLs), IndiaMART (BreadcrumbList + ImageGallery) |
| `<script type="application/json" id="...">` | Remix / SvelteKit / generic | (not yet observed) |

### Tier 2: Site-named globals

Custom globals named by the site. Caught by the generic `window.<IDENT> = {...}` matcher.

| Location | Sites observed |
|---|---|
| `window.__initialData__` | 99acres |
| `window.SERVER_PRELOADED_STATE_` | MagicBricks |
| `window.__APP` | OLX (but see "JS object literal" pitfall below) |
| `window.__APP_CONFIG__` | Housing (metadata only, no listings) |
| `window.stateFromServer` | CommonFloor (but built incrementally — see Tier 3) |

### Tier 3: Incremental construction (legacy pattern)

Older SSR sites declare an empty global and then attach properties one at a time:

```javascript
window.stateFromServer = {};
window.stateFromServer.pageType = 'SerpPage';
window.stateFromServer.city = 'Delhi';
window.stateFromServer.filters = {"priceMax": 50000};
```

The strict extractor returns the empty `{}`. The `synthesizeIncrementalAssignments` recovery in `schemaWalker.ts` aggregates the dot-assignments into a synthesised object. Source name in the output: `window.<name> (synthesized)`.

Sites observed: **CommonFloor** (definitive example).

### Tier 4: XHR-only (no payload in HTML)

The page HTML carries no listing data — everything is loaded after `DOMContentLoaded` via authenticated `/api/*` calls.

Sites observed: **Housing.com** (commercial subhub, deep URLs), **NoBroker** (expected)

**Current status:** Tier 4 capture is **now built**. See `@vinxi/scraper/recon` (`xhrCapture`) and the full recipe at [`./xhr-and-partial-html.md`](./xhr-and-partial-html.md). The helper registers a Playwright response listener BEFORE navigation, collects matching JSON/HTML responses, and returns them in the same `HydrationPayload` shape `walkSchema` already understands.

For protected sites (Housing.com Imperva), combine XHR capture with a homepage warmup — see the pattern doc.

## Recognition heuristics during recon

1. **Open the page HTML in the inspector and search for known data values** (a price you can see on screen, a name, a city). If found in a `<script>` tag, it's a hydration payload.
2. **Run `extractAllHydrationPayloads(html)`** — returns every payload with its `source` name. Look at all of them; the canonical one varies by site.
3. **If none of the standard names appear and `extractAllHydrationPayloads` returns empty:**
   - Check for `window.<IDENT> = {};` followed by many `window.<IDENT>.foo = ...` lines (Tier 3 / incremental)
   - Check for `application/ld+json` script blocks (almost always carries SOMETHING useful even if not canonical)
   - If still empty, the site is Tier 4 (XHR-only) — you need network capture, not HTML scraping

## Pitfalls

### JS object literal vs. strict JSON

OLX emits `window.__APP = { props: { lang: "en-IN", ... } }` — the keys are unquoted JS identifiers. The strict `JSON.parse` rejects this. The permissive recovery in `extractAllHydrationPayloads` quotes the identifier keys and retries — it handles the simple case.

Real-world OLX `window.__APP` also contains `undefined`, function references, and other JS-only constructs that even the permissive parser fails on. **Don't chase this.** OLX has rich JSON-LD that's strictly conformant.

**Rule:** if the strict-JSON globals fail, **prefer JSON-LD over fighting the JS literal.**

### Multiple payloads on the same page

Real sites embed multiple. IndiaMART detail has `<script id="__NEXT_DATA__">` AND two JSON-LD blocks. OLX detail has `window.configTracking` AND three JSON-LD blocks. **The canonical data may not be in the largest payload.** Check each.

### Empty wrappers

Some sites declare `window.X = {}` to register the global early, then never populate it (or populate via XHR). The `extractAllHydrationPayloads` happily returns the empty `{}`. Always check leaf count before trusting a payload — a 0-leaf payload is no payload.

### Class-hashed CSS-in-JS sites

Emotion / styled-components / Tailwind-arbitrary-classes sites (Housing.com is the example) have class names like `css-1a2b3c`. Even if a payload exists, the DOM is hostile to selector-based extraction. **Always prefer the payload over the DOM on these sites.**

## When to add a new tier

Add a tier here when you find a hydration mechanism that doesn't fit the existing four. Recent candidates that could become tiers if more sites use them:

- **Stencil / Lit web components** with `data-*` props (none observed yet)
- **HTMX / Turbo** sites with morphdom partials (none observed)
- **GraphQL Apollo persisted-query payloads** in script tags (some Medium pages)
