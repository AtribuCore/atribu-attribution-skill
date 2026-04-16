# Atribu Attribution Skill

The opinionated Claude Code skill for [Atribu](https://www.atribu.app) — query identity-stitched marketing attribution data via the Atribu MCP server (`mcp.atribu.app`).

Once installed, the skill teaches your AI client (Claude Code, Cursor, etc.) how to:

- Always default to **cash ROAS** (real revenue from Stripe / MercadoPago), never CRM `value_amount`.
- Mask **PII** by default; unmask only when scope + workspace setting + per-call flag all allow it.
- Pick the right tool for the question (e.g. `creative_fatigue_check` for "which ads to kill", `compare_periods` for trends, `find_anomalies` for daily spikes).
- Follow the **preview → confirm** flow for write-back to Meta CAPI, with idempotency keys.

## Install

```
/plugin marketplace add AtribuCore/atribu-attribution-skill
/plugin install atribu-attribution@atribu-attribution
```

Then restart your Claude Code session.

## Prerequisites

You need an MCP token from the Atribu app. Sign in to [atribu.app](https://www.atribu.app), open **Developer → MCP Tokens**, create a token, and configure your client. Full instructions: <https://www.atribu.app/docs/mcp/quickstart>.

## What's in the box

- [`SKILL.md`](plugins/atribu-attribution/skills/atribu-attribution/SKILL.md) — the skill itself (auto-loaded by Claude Code).
- [`references/`](plugins/atribu-attribution/skills/atribu-attribution/references/) — deeper guidance: tool ordering, error playbook.
- [`evals/`](plugins/atribu-attribution/skills/atribu-attribution/evals/) — test cases for skill behavior.

## Updating

When the skill is updated upstream:

```
/plugin update atribu-attribution
```

## Source repo & feedback

This is the canonical home for the skill. The Atribu app and MCP server live in a separate (private) repo. If you find the skill giving bad guidance or out-of-date information, [open an issue](https://github.com/AtribuCore/atribu-attribution-skill/issues) or send a PR.

## License

MIT — see [LICENSE](LICENSE).
