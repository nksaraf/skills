# Patterns

Cross-cutting observations distilled from multiple sites. When you see the same shape on 2+ sites, write it here. Per-site oddities stay in `sites/<site>.md`.

## Index

| Pattern | Doc | First seen | Also seen on |
|---|---|---|---|
| Hydration payload locations | [hydration-payloads.md](./hydration-payloads.md) | 99acres | MagicBricks, IndiaMART, OLX, Housing, CommonFloor, Mango |
| Label/value DOM shapes | [label-value-shapes.md](./label-value-shapes.md) | 99acres | MagicBricks, IndiaMART, OLX, CommonFloor |
| Anti-bot protection profiles | [protection-profiles.md](./protection-profiles.md) | 99acres (Akamai) | Housing (Imperva), Zara, H&M |
| Indian rupee / currency rendering | [currency-rendering.md](./currency-rendering.md) | 99acres | CommonFloor (icon-font) |
| XHR-driven SPAs / partial HTML | [xhr-and-partial-html.md](./xhr-and-partial-html.md) | Housing.com (Tier 4) | NoBroker (expected) |
| **Sibling-brand probing** | [sibling-brand-probing.md](./sibling-brand-probing.md) | Westside → Zudio at same API | Trent (5+ brands), Inditex, ABFRL, Reliance |
| **Google Maps embed coord recovery** | [google-maps-embed-recovery.md](./google-maps-embed-recovery.md) | Zudio (0% → 99% lat/lng) | Westside (70% → 90%) |
| **Validation scorecard** | [validation-scorecard.md](./validation-scorecard.md) | All retail brands | All future scrapes (mandatory Phase 6) |

## When to write a new pattern doc

A pattern doc earns its place when:
- The same DOM/JSON shape appears on 2+ sites
- The observation generalises (it's NOT a single site's quirk)
- A future agent needs the abstraction to write a faster extractor

A pattern doc does NOT replace the per-site addendum. The site addendum has the concrete paths and selectors *for that site*; the pattern doc has the general shape and how to recognise it.
