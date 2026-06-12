# Pattern: Anti-bot protection profiles

What we've seen, how to recognise it, what works.

## Profiles observed

### Akamai Bot Manager (AkamaiGHost)

**Sites:** 99acres
**Diagnostic signals:**
- `server: AkamaiGHost` response header on the block page
- `403 Forbidden` with a short HTML body "Access Denied"
- Reference ID: `Reference&#32;&#35;18&#46;5ca5c117…` (URL-encoded `#`)
- The `cf-ray` header is ABSENT (that's Cloudflare, not Akamai)

**Behavior:** First request with realistic Chrome headers slips through. Subsequent requests from the same IP within 1-5 minutes get 403 across ALL paths (including `robots.txt` and the homepage). The session is poisoned, not just the URL.

**What works:**
- **Stealth Playwright Chromium with full client-hints + Indian locale.** The runtime's `ctx.browser.newPage()` includes this.
- One browser context for the whole crawl — cookies + session persist.
- Humanized pacing (`ctx.human.pause(2500, 5500)` between pages).
- Homepage warmup (`ctx.human.warmup(page, "https://www.<site>.com/")`).

**What doesn't:**
- Raw `curl` after the first request (any number of headers).
- Stealth Chromium WITHOUT the Client Hints headers (`Sec-Ch-Ua`, `Sec-Ch-Ua-Mobile`, `Sec-Ch-Ua-Platform`).

### Imperva (Incapsula)

**Sites:** Housing.com (deep URLs)
**Diagnostic signals:**
- HTTP `406 Not Acceptable` (sometimes 403/503)
- `<h1>Request Blocked</h1>` on the response
- Body contains `Block Reference ID`, `Real Client IP`, and a "causes" list including "Use of bots or scrapers"
- Reference ID has the shape `0.<hex>.<timestamp>.<hex>`

**Behavior:** Direct deep-URL hits trigger immediately even with stealth. Homepage navigation works fine; the moment you navigate to a deep URL (especially a search result), Imperva intercepts.

**What works (in theory — not yet tested by us):**
- Long session warmup (visit homepage, dwell, click around — minutes of "real-looking" activity before deep navigation)
- Residential proxy (Imperva fingerprints datacenter IPs aggressively)

**What doesn't:**
- Our current stealth setup (cookies + Client Hints + mouse + locale) — gets blocked anyway

### Cloudflare bot management

**Sites:** none of our current targets
**Diagnostic signals (per docs):**
- `cf-ray` response header
- `set-cookie: __cf_bm=...` or `cf_clearance=...`
- 403/503 with Cloudflare-branded challenge page
- "Checking your browser" interstitial HTML

**What works (per general knowledge):**
- Stealth Chromium often passes the bot management page once cookies are set
- For aggressive Cloudflare setups (Turnstile challenge), only manual interaction or specialised proxies work

## Recognition during recon

The current `auditPage` returns DOM data but does NOT explicitly classify protection pages. The agent has to recognise the block page from the headings:

| Audit signal | Likely provider |
|---|---|
| `headings: ["Access Denied"]` + body 421 bytes | Akamai |
| `headings: ["Request Blocked", "Block Reference ID", ...]` | Imperva |
| `headings: ["Just a moment..."]` + challenge UI | Cloudflare |
| `headings: []` + 323-byte HTML body | Likely a JS-challenge interstitial (any provider) |

**Parked enhancement:** `auditPage` should detect and return a `protectionSignal?: { provider, referenceId?, clientIp? }` so callers don't have to do this classification by string matching.

## Mitigation ladder

When recon shows protection, escalate in this order. Stop at the first level that gives access.

| Level | What | When |
|---|---|---|
| 1. Raw HTTP with full headers | UA, Accept-Language, Sec-Fetch-*, Sec-Ch-Ua-* | Always try first |
| 2. Stealth Chromium | The runtime's default `ctx.browser.newPage()` already does this | If Level 1 returns 403 on second request, or page is JS-rendered |
| 3. Stealth + warmup + humanization | `ctx.human.warmup()` + jittered pauses + mouse activity | If Level 2 still blocked after first request |
| 4. Residential proxy | Routed via proxy pool | If site fingerprints datacenter IPs (Imperva, aggressive Cloudflare) |
| 5. TLS fingerprint spoofing | Use only when downgrading from browser to HTTP for speed | Rarely needed; Chrome's own TLS is legitimate |

## High-protection sites (heuristic — assume Level 3+ from the start)

Per the SKILL.md domain hints and what we've observed:

- LinkedIn (auth-walled + heavy fingerprinting)
- Airlines / ticketing (MakeMyTrip, IRCTC)
- Housing.com (Imperva)
- NoBroker (SPA + API rate-limiting)
- AmazonAircraftCarrier / Amazon listing pages (geo-locked, IP-fingerprinted)

When you target one of these, start at Level 3, not Level 1.

## Sites with mild / no protection (per our recon)

- MagicBricks
- IndiaMART
- OLX (occasional HTTP/2 protocol errors but no Imperva-style blocks)
- CommonFloor

For these, Level 2 (stealth Chromium without warmup) is usually enough.
