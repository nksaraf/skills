# Reference: The recon-extract loop

**The single most important framing of this skill.** Read this when you find yourself iterating on a scraper longer than you expected — it's almost always because someone treated recon as a linear "look once → build" pipeline. It's not.

## The core insight

Pure recon — "looking at one page" — is **incomplete by construction.** No matter how thorough you are, you will only see:

- Whatever fields are present on the *specific listing* you opened
- Whatever DOM shape happens to render *that one card*
- Whatever payload paths your *first guess* searched for

The extractor is where the gaps surface. The act of writing `extractDetail(html) → DetailRecord` is what forces you to confront:

- Field X is present in the payload but renders as null on this listing — is it always null, or just here?
- Field Y appears via selector `.priceWrap` for 26 out of 27 cards — what happened to card #2?
- The `nearByPlacesOfInterest` array is 47 items on this listing — is it always 47, or variable?
- The "NOT AVAILABLE" badge is right there on the screenshot — how did I not see it during recon?

**This is normal. This is the point.** The extractor is BOTH the deliverable AND the validation tool for recon. They're not separable phases.

## The right mental model

Recon and extraction form a **tight loop**, not a pipeline. Each turn through the loop sharpens the other:

```
1. Sample one URL, run recon tools → first guess at fields
2. Write a DRAFT extractor (5 fields, 10 lines)
3. Run draft on 5-10 URLs → see variation, see gaps
4. Visual audit one URL fully → see what extractor missed
5. Re-recon any newly-suspect field (e.g. inspect the row where extractor failed)
6. Expand extractor (now 20 fields, 50 lines)
7. Lock fixture + schema-locked tests
8. Move on
```

Steps 1-6 are the LOOP. The temptation is to skip 3-5 — to write a "complete" extractor on the first pass. That's where mistakes pile up. Resist it.

## Why this is true

Three forces conspire to make recon-without-iteration incomplete:

### 1. Sample bias — you looked at ONE listing

99acres has 1,277 warehouses in Delhi alone. The first listing you open is one data point. The next 9 might have:
- A field present in the payload that was null/empty on listing #1
- A different `locationName` shape (`"FIRST FLOOR"` instead of `"Kirti Nagar, Delhi, West Delhi"` — this actually happened)
- A different status (Verified vs. unverified, Featured vs. not, RENTED OUT vs. live)
- A different poster type (Owner vs. Dealer)

You can't predict variation from one example. **You have to scrape, diff, and learn.**

### 2. Confirmation bias — you searched for what you expected

When you `grep` the payload for "price", "area", "location", you find them. That makes you believe you've covered the field surface. But you haven't checked for:

- `posted_on`, `register_date`, `updated_at` — temporal fields you didn't ask about
- `washroom_count`, `floors`, `parking`, `furnishing` — feature fields specific to the property type
- `verified`, `premium`, `featured`, `availability_status` — quality/freshness signals
- `lat`, `lng`, `nearby_pois` — geo signals

These only become visible when you (a) walk the WHOLE schema with no prior filter, or (b) compare the screenshot to your scraped record side-by-side.

### 3. Reality is messier than your model

Real sites have:
- Half-populated rows (some fields filled, others empty — represented as `""` vs `null` vs missing key)
- Promotional cards that look like listings but aren't (sponsored content)
- "Similar properties" sections mixed into the SRP
- Locale-dependent rendering (Hindi vs English, sq ft vs sq.m. toggle)
- A/B test variants — different DOM for the same field on different sessions
- Stale entries that should be filtered (`NOT AVAILABLE` is the canonical example)

A linear recon will catch ~70% of these. The remaining 30% only surface during the loop.

## The framing for the agent

When you start work on a new site:

1. **Don't promise a "complete extractor by phase 5d".** Promise a draft.
2. **Set the iteration budget upfront.** "I'll run the loop 3 times before declaring done."
3. **Treat each loop turn as a discovery, not a failure.** Finding a missed field is the point.

When you find yourself thinking *"the extractor is wrong"*, reframe to *"the extractor revealed a gap recon couldn't see — good, that's the loop working."*

## The structural rules

These keep the loop disciplined rather than chaotic.

### Sample 5-10 URLs per loop turn, not 1

The first draft extractor runs on a single test URL — that's fine for "does anything come out?" but useless for finding variation. Loop turns 2+ must run on 5-10 URLs sampled across at least 2 cities OR 2 categories. The diff between outputs is where variation surfaces.

### Take screenshots EVERY loop turn

Phase 5e isn't a one-time step. Every time you change the extractor, take new screenshots and re-audit. Visual diff is the cheapest way to find missed fields. The `recon-99acres-detail.ts` scraper is a template: it screenshots + audits + walks the schema in one run.

### Lock the fixture only on the last turn

Save the captured HTML as `__tests__/fixtures/<site>-detail-<id>.html` and write schema-locked tests AFTER the extractor stabilises (i.e. after 2-3 loop turns produce the same record shape across all sample URLs). Locking too early bakes in mistakes.

### Capture the "I didn't see this" moments

When a loop turn surfaces something recon missed, write it down. Either in the per-site addendum (`sites/<site>.md` → Gotchas section) OR — if the lesson generalises — in a pattern doc (`patterns/`). This is how the skill compounds.

## How many turns are enough?

For most sites: 3 turns of the loop. For sites with complex DOM variation or rich payloads: 4-5. Stop when:

- A diff of the extractor output across 10 sample URLs shows zero schema differences (every record has the same shape)
- Visual audit on a fresh URL surfaces no `findMissingFields` items
- Schema-locked tests pass against the captured fixture
- The site addendum is written

These four together are Gate D from the SKILL.md. They exist precisely because they can't be checked after a single pass.

## Worked example: 99acres

This is what actually happened building the 99acres scraper. The loop took 4 turns:

**Turn 1** — Recon found JSON-LD `ItemList` + CSS selectors on `.tupleNew__outerTupleWrap`. Draft extractor pulled 5 fields (title, price, location, area, URL). Ran on page 1 → 27 listings extracted.

**Turn 2** — User asked "is this also getting detail page info?" Turn 1 had ignored detail pages entirely. Opened one detail URL, walked `window.__initialData__.pd.pageData.propertyDetails.prop_data`, expanded extractor to ~30 fields including lat/lng, posted-by, description.

**Turn 3** — Tests passed, but user asked for visual verification. Screenshots showed:
- **`NOT AVAILABLE` red badge** (DOM-only, not in payload) — missed
- **`Featured` + `Verified` badges** — missed
- **`10 private washrooms`** — present in payload but no extractor field
- **`Water Storage`** amenity in `societyAmenities` (we'd only checked `homeAmenities`)
- **`Register_Date` "Apr 10, 2026"** — absolute date, only relative captured

Added 8 new fields. Tests went from 4 → 11.

**Turn 4** — User asked "did we miss POIs?" Yes — the `nearByPlacesOfInterest` array (47 items per listing) was a 30-second extractor add. Added `nearbyPlaces[]` + derived `nearestMetroKm` / `nearestMetroName`.

**Result:** 65+ fields, 11 schema-locked tests, two fixtures, an addendum. **Each turn surfaced gaps that recon alone would have missed.** The user's prompts were the loop trigger — but they shouldn't have been needed. The methodology should have driven the iteration.

## What's different about pure-XHR sites (Housing.com)

For XHR-only sites, the loop has a different first turn — you can't read the HTML to find data, you have to read the network. See `../patterns/xhr-and-partial-html.md`.

After that first turn, the loop is the same: sample URLs → diff outputs → visual audit → expand extractor → lock.

## What to do when the loop won't converge

If turn 4, 5, 6 keep surfacing gaps:

- **The variation is real and broader than you sampled.** Sample more URLs (across more cities, categories, listing ages).
- **The site is A/B testing.** Two layouts exist; you only saw one. Compare runs across days, or with different cookies.
- **You're missing a page type.** A "dealer profile" or "locality page" carries fields the listing pages don't. Re-check Phase 0a (page-type enumeration).
- **Stop and write an addendum anyway.** Document the known limitations, ship what works, come back later.
