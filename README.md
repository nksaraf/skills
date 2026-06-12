# nksaraf/skills

Agent skills I use and share — installable with the open [skills CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add nksaraf/skills            # pick from the list
npx skills add nksaraf/skills --skill site-reconnaissance
```

Works with Claude Code, Cursor, Codex, OpenCode, and [70+ other agents](https://github.com/vercel-labs/skills#supported-agents).

## Skills

### [site-reconnaissance](./skills/site-reconnaissance/SKILL.md)

Go from "here's a URL" to a working, tested, production-grade scraper — systematically. A six-phase reconnaissance methodology with quality gates:

- **Recon–extract loop** — recon and extraction as a tight loop, not a pipeline; the extractor is how you find out what recon missed
- **Tiered transport** — raw HTTP → JSON API → stealth browser, always preferring the cheapest tier that works
- **Hydration payload walking, XHR capture, visual ground truth, schema walks** — see everything a page knows before committing to selectors
- **Validation scorecards** — numeric completeness / quality / authenticity / freshness scoring, because "ran without errors" is not "good data"
- **A case library that learns** — 60+ site addenda (real estate portals, retail store locators, e-commerce APIs) with framework signatures, payload paths, protection profiles, and the gotchas that actually bit

Pairs with the [`@vinxi/scraper`](https://www.npmjs.com/package/@vinxi/scraper) runtime (Bun + Playwright): `defineScraper()`, stealth browsing, checkpoints, task queues, JSONL streaming, and the recon toolkit the skill drives.

```bash
bun add @vinxi/scraper
```

## License

MIT
