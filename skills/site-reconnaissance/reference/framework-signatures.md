# Reference: Framework signatures

Look these up during Phase 0 to short-circuit recon.

## Response header signatures

| Header | Value contains | Framework | What to search next |
|---|---|---|---|
| `X-Powered-By` | `Next.js` | Next.js | `__NEXT_DATA__` script tag, `/_next/data/` routes |
| `X-Powered-By` | `Nuxt` | Nuxt.js | `window.__NUXT__` or `__NUXT_DATA__` script tag |
| `X-Powered-By` | `Express` | Express/Node | JSON APIs at `/api/` paths |
| `X-Shopify-Stage` | any value | Shopify | `ld+json`, `/products.json`, `/collections.json` |
| `Link` | `</wp-content/` | WordPress | `/wp-json/wp/v2/` REST API, `ld+json` |
| `X-Drupal-Cache` | any value | Drupal | `/jsonapi/`, `ld+json` |
| `Server` | `cloudflare` | CDN (not a framework) | Note anti-bot layer |
| `cf-ray` | any value | Cloudflare | Note anti-bot layer |
| `Server` | `AkamaiGHost` | CDN | Note anti-bot layer |
| `Server-Timing` | `react;dur=...` | React SSR (custom) | `window.<IDENT>` globals or `__NEXT_DATA__` |
| `set-cookie` | `__cf_bm` or `cf_clearance` | Cloudflare bot mgmt | Phase 4 warranted |

## HTML signatures

| Pattern in HTML | Detected framework | Search first | Skip |
|---|---|---|---|
| `<script id="__NEXT_DATA__"` | Next.js | Parse JSON from that script tag | `__NUXT__`, `__INITIAL_STATE__`, `/wp-json/` |
| `window.__NUXT__` or `__NUXT_DATA__` | Nuxt.js | Parse embedded state object | `__NEXT_DATA__`, `/wp-json/` |
| `window.__INITIAL_STATE__` | Vue / Redux SSR | Parse embedded state JSON | Skip API sniffing if state has all data |
| `/wp-content/` in `<link>` tags | WordPress | `/wp-json/wp/v2/` API, `ld+json` | SPA state objects |
| `ng-version=` | Angular | XHR/fetch API calls (browser required) | SSR data extraction |
| `data-reactroot` | React CSR | XHR/fetch API calls (browser required) | SSR data objects (none exist) |
| `<script type="application/ld+json"` | Any (structured data) | Parse JSON-LD for product/article data | May be partial — verify completeness |
| `_sveltekit` | SvelteKit | Embedded data in `__data` nodes | React/Vue patterns |
| `data-turbo-` | Rails/Turbo | Standard HTML selectors | SPA framework patterns |
| `css-1a2b3c` class pattern | Emotion / CSS-in-JS | `data-testid`/`data-q`/`data-aut-id` attribute extraction | Class-based selectors (they're hashed) |

## Framework → Strategy mapping

| Framework | Tier | Primary approach |
|---|---|---|
| Next.js SSR | 1 or 2 | Parse `__NEXT_DATA__` or hit `/_next/data/*.json` routes |
| Nuxt.js SSR | 1 or 2 | Parse `__NUXT__` or capture API calls |
| WordPress | 2 | `/wp-json/wp/v2/posts?per_page=100&page=N` |
| Shopify | 2 | `/products.json?page=N&limit=250` |
| React CSR | 3 | Browser + API interception |
| Angular | 3 | Browser + API interception |
| Custom React SSR | 1 or 3 | Look for `window.<IDENT>` hydration payload; protection often forces Tier 3 |
| Static HTML | 1 | cheerio selectors on raw HTML |
| GraphQL SPA | 2 or 3 | Replay POST to `/graphql` with captured query |
