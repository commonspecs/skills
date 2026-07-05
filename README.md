# commonspecs skill

The reference skill for **[commonspecs](https://commonspecs.com)** — an API-first database
of **engineering-grade product specifications with per-field confidence**. Install it, and
your AI agent can look up what a product is actually made of (fabric weight and weave, shoe
construction, country of origin), compare products on hard specs, and contribute specs it
has verified.

## Install

```bash
npx skills add commonspecs/skills
```

Works with Claude Code, Cursor, and the other agents the [`skills`](https://skills.sh) CLI
supports. Or install manually:

```bash
# Claude Code
cp -r skills/commonspecs ~/.claude/skills/
```

Then connect. The best path is the [commonspecs MCP server](https://commonspecs.com/docs/mcp/):
it signs in with OAuth — no token to create or manage — and the skill drives its tools
directly. If your agent has no MCP support, set an API token instead (get one at
[commonspecs.com](https://commonspecs.com)):

```bash
export COMMONSPECS_API_TOKEN="cs_live_…"
```

That's the only local configuration either way. Your buying preferences — market (country), quality and
locality strategy, contribution mode — live on your account
([commonspecs.com/account](https://commonspecs.com/account)); the server applies them to every
read and returns them in a `context` block, so the agent never needs them configured locally.

## Why this exists

Most product "data" online is marketing copy: adjectives, not specifications. When you ask
an LLM "is this a good pair of raw denim jeans?", it pattern-matches vibes. commonspecs
answers from **hard facts** — and tells you **how confident** each fact is and **whether
sources agree**.

The point isn't a number to obey. It's to give your agent the facts that actually
differentiate good from bad in a category, so it can reason about *fitness for purpose* and
*cost of use*, not just price.

## What "good" means, by category

The skill teaches your agent to weigh the specs that matter (the exact scoring is
intentionally not exposed):

- **Raw denim / jeans** — fabric weight (oz) and weave (selvedge vs open-end), composition,
  country/mill of origin.
- **Welted footwear** — construction (Goodyear-welted vs Blake vs cemented), leather grade,
  sole. Construction decides whether a shoe can be resoled — its real cost over years.
- **Fragrance** — concentration (parfum → EDP → EDT → EDC), longevity, projection.

The agent fetches the facts; it explains the trade-off in plain language. It never claims a
value it didn't get back, and it always carries the confidence with the fact.

## Reading the answer

| Signal | Meaning |
| --- | --- |
| `quality_score` (0–100) | Overall spec quality; `missing_fields` shows what's dragging it down. |
| `confidence` (0–1) | How well-evidenced a value is. Always reported with the fact. |
| `fields[]` | Confidence ≥ 0.4 — solid enough to state. |
| `low_confidence_fields[]` | Confidence 0.1–0.4 — a lead, shown only with a hedge. |
| `needs_corroboration` | Only one independent source so far. |
| `disputed` + `alternate_claims` | Sources disagree; both values are shown. |

## Contributing specs back

If your agent reads a verified spec (off a product page, or a physical label) that
commonspecs is missing, it can submit it with the source URL and the exact snippet it read
the value from — and a dated price observation from the same page, since specs and prices
usually sit on the same fetch. Evidence is what earns confidence; corroboration from
independent users is what makes a value trustworthy. Photos never leave your machine — only
the extracted values and the product identity are sent.

See [`skills/commonspecs/SKILL.md`](skills/commonspecs/SKILL.md) for the exact API calls,
or the [docs](https://commonspecs.com/docs/) for the full API reference and methodology.

## License

[MIT](LICENSE) © commonspecs
