# Reference: Common pitfalls

Lessons we've learned the hard way. Read before starting recon on a new site.

## "Found in HTML" is not "extractable with cheerio"

Text may be:
- Inside a `<script>` tag as JSON data (not a DOM element) — parse as JSON, not a selector
- In an HTML comment
- Split across multiple elements
- Behind JavaScript (renders as `$--` in raw HTML, real value only after JS executes)

Always verify with an actual extraction test (Phase 3).

## Minified / generated class names

React, Vue, and Angular apps often use build-time class names like `.css-1a2b3c`. These change between deployments. Prefer:

- `data-*` attributes: `[data-testid="price"]`, `[data-aut-id="itemPrice"]`
- Structural selectors: `article:nth-child(1) .price`
- Stable semantic elements: `h2`, `address`, `time[datetime]`

See [`../patterns/label-value-shapes.md`](../patterns/label-value-shapes.md) for the canonical extractor for each convention.

## Partial SSR (skeleton hydration)

Some sites render a skeleton in SSR (`$--` for price) and hydrate with JS. Raw HTML has the structure but placeholder values. Compare raw HTML values against rendered DOM values before deciding on a tier.

## API authentication drifts

Tokens observed in Phase 1 traffic may expire. If a token is needed, prefer:

1. Session cookies (longer-lived, often auto-set from landing page)
2. Public API keys in request headers (stable)
3. Bearer tokens (short-lived — scraper may need to refresh)

## Pagination off-by-one

Always test page 1 AND page 2 before trusting the pagination pattern. Some sites use `page=0` as first page; others use `offset=0&limit=20`.

## Stale "valid" listings

Many sites keep withdrawn / rented-out listings visible with a status badge. **Filter on the badge.** Without this, price benchmarks include dead inventory.

Example: 99acres `availabilityStatus === "NOT AVAILABLE"`.

## First-request-passes, second-fails

Several sites (Akamai-protected especially) allow the first request from a new IP through to gather analytics, then block subsequent requests. **Test with multiple consecutive requests during Phase 4**, not just one. If only the first request works, you need Tier 3 with session persistence.

## JSON-LD looks structured but is partial

JSON-LD on listing pages often has incomplete data — `ItemList` with just URLs and names, no prices. Always check what's actually populated vs. what the schema *could* hold. Prefer site-specific hydration payloads (`window.__initialData__`, etc.) for richer fields; use JSON-LD when there's nothing else (OLX, CommonFloor).

## "It works on my machine" with Playwright defaults

Bare `chromium.launch({ headless: true })` is detected by every serious anti-bot stack. The runtime's `ctx.browser.newPage()` includes stealth args, Indian locale, Client Hints, `navigator.webdriver` masking — DON'T bypass it without a reason. If you're tempted to write your own `chromium.launch(...)` inside a scraper, you almost certainly want the facade instead.

## Recon tools are noisy on purpose

`auditPage()` outputs include false positives. `findMissingFields` produces noise on sites where label/value extraction is weak. **That's by design.** The recon tools help you SEE the page faster; they don't make production-grade decisions. The deliverable is a per-site extractor + fixture + tests, not a perfect generic.

## Source-side data quality issues are common and silent

The source isn't always clean. Real-world examples from this session:

- **`"Tamilnadu"` vs `"Tamil Nadu"`** — Westside has 1 store labelled with the no-space spelling, fragmenting the state aggregate. Caught by the validation scorecard's per-state distribution audit, not by the extractor.
- **Blank lat/lng with a Google Maps embed in the same record** — Zudio had `store_latitude = ""` on 555 of 555 stores but every record had a `google_emmbeded_code` URL containing the coordinates. Two source fields claiming the same data, both populated by different processes, one of them broken. See [`../patterns/google-maps-embed-recovery.md`](../patterns/google-maps-embed-recovery.md).
- **A single GPS-error record** — Westside had 1 store of 209 with coordinates outside India's bounding box. Caught by the validator's `coords-in-bounds` check.

**Implication:** every scrape needs a post-extraction validation pass. See [`../patterns/validation-scorecard.md`](../patterns/validation-scorecard.md).

## When a fix to the recon tools is the wrong move

If you find yourself adding a branch to `visualAudit.ts` or `schemaWalker.ts` to handle "one more case", ask: is this case general (will help future sites) or specific (one site)?

- **General → patch the tool.** Examples: JSON-LD support, `data-testid` recognition, `window.<IDENT>` matcher, incremental-assignment recovery.
- **Specific → write a per-site extractor.** Examples: 99acres `__badgeParent` POI wrapper, OLX `__APP` JS-isms.

The skill's "case library" model (this directory) is how site-specific lessons compound without polluting the shared tools.
