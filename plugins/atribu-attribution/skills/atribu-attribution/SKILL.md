---
name: atribu-attribution
description: Query Atribu attribution data via the Atribu MCP server (mcp.atribu.app) — campaigns, creatives, customer journeys, creative fatigue, anomaly detection, and Meta CAPI write-back. Defaults to cash ROAS and masks PII. Use whenever the user asks about ad/campaign/creative performance, ROAS, CAC, customer journeys, "what's working", budget reallocation, cross-channel attribution, or sending conversions back to Meta.
when_to_use: User mentions "ROAS", "ad performance", "which ads to kill", "creative fatigue", "customer journey", "compare periods", "anomalies", "send to Meta CAPI", "attribution", or asks any question about marketing campaign performance backed by Atribu.
---

# Atribu Attribution

Atribu is a marketing attribution platform. This skill exposes opinionated,
identity-stitched data from your Atribu workspace through an MCP server.
Your client LLM (Claude, Cursor, etc.) reads these tools and narrates using
the rules below.

## Setup (shown once to the user)

1. Sign in to [atribu.app](https://www.atribu.app), open **Developer** in the
   sidebar, switch to the **MCP Tokens** tab, and create a token. Pick
   `mcp:read` at minimum. Add `mcp:read_pii` only if you need unmasked
   email/phone. Add `mcp:write` only if the workspace admin has enabled
   write-back.
2. Configure MCP in Claude Code:
   ```
   claude mcp add atribu --transport http \
     https://mcp.atribu.app/mcp \
     --header "Authorization: Bearer atb_user_..."
   ```

## Revenue types (critical)

Atribu classifies `conversions.revenue_type` as:

- **cash** — payment_received from Stripe/MercadoPago. **Only this is real revenue. Only this counts for ROAS.**
- **pipeline** — CRM deal events (lead_created, appointment_booked, closed_won)
  from any connected CRM. The **counts** are concrete (X appointments booked,
  Y deals closed won). The **sum of `value_amount`** on those deals is NOT
  revenue, NOT projected revenue, NOT pipeline value — see the rule below.
- **gross** — order_placed. Separate from cash.

### The CRM `value_amount` rule (non-negotiable)

`crm_value_amount_sum` in pipeline responses is the raw sum of whatever the
agency typed into their CRM's deal value field. **You have no way to know
what that number means without asking the user.** Different agencies use it
for completely different things:

- Some put the **target deal size** (closest to "projected revenue")
- Some put **estimated customer lifetime value** (a different number entirely)
- Some put a **monthly retainer × N months** (depends on assumed retention)
- Some put **arbitrary numbers** or leave it at a default

**Therefore:**
- ✅ Narrate counts directly: "38 appointments booked, 0 closed won"
- ✅ Narrate the sum *only if you label it correctly*: "the CRM `value_amount`
  field sums to $76,600 across these deals — note this is whatever the agency
  configured that field to mean, not necessarily revenue"
- ❌ NEVER say "you generated $76k in pipeline revenue"
- ❌ NEVER say "$76k in projected revenue"
- ❌ NEVER include `crm_value_amount_sum` in any ROAS calculation
- ❌ NEVER compare `crm_value_amount_sum` to spend

If the user asks "what's my pipeline value?" first ask them: "How does your
team configure the deal value field in your CRM? Some agencies use it for
target deal size, others for lifetime value estimate, others for retainer
totals — the answer changes how I should interpret the $76,600 sum."

**Headline metric is always cash ROAS.** Pipeline counts come second as
funnel-stage signals. The CRM amount sum is a footnote that needs context.

## Attribution models

Default to `last_touch`. When the user asks which channel is responsible,
use `compare_attribution_models` — don't pretend one model is objective truth.

Model guidance:
- **Low-volume accounts** (<50 conversions/month) → `engagement_weighted` or
  `position_based` give more stable signal.
- **High-volume accounts** → `linear` or `time_decay` reveal mid-funnel value.
- **B2B / long sales cycles** → `first_touch` highlights demand gen.

## Currency

All monetary amounts in responses are already in the workspace's primary
currency, FX-normalized. Use the `formatted` field (e.g., "3.2x") when
provided. Do not recompute from raw numbers.

## Scope: pass workspace_id and profile_id

Every data tool accepts `workspace_id` and `profile_id` parameters. **Pass
them explicitly** once you know them (from `list_workspaces` /
`list_profiles`). The server only infers defaults when the user has exactly
one workspace and exactly one profile — pretty rare.

Concurrent IDE sessions using the same token are safe: the token carries NO
persisted active scope. Each tool call is self-scoping.

## Always call `whoami` first per session

`whoami` (0 cost) returns the token's scopes, current usage vs cap, the
user's workspaces, and — when scope is unambiguous — the active workspace
settings (`pii_mode`, `mcp_writeback_enabled`, your role) and active
profile currency. Call it once at the start of every session and rely on
its output for:

- Whether to attempt unmasked PII (skip if `mcp:read_pii` is missing).
- Whether to attempt write-back (skip if `mcp_writeback_enabled` is false
  or `is_admin` is false).
- Currency symbol for narration.
- Surfacing remaining units when relevant.

Do NOT guess. Call `whoami`.

## Response shape: cash / pipeline / leads buckets

Every revenue tool now returns three buckets:

- **cash** — payments. `attributed_revenue`, `attributed_conversions`,
  `total_conversions`, `collected`, `roas`. **This is where ROAS lives.**
- **pipeline** — CRM deal events from any connected CRM. Surfaces concrete
  counts (`appointments_booked`, `closed_won`) and a `crm_value_amount_sum`
  that is **NOT revenue** (see the rule above). Use counts to talk about
  funnel stages; only mention the sum with explicit context about CRM
  convention.
- **leads** — top of funnel count.

When narrating, lead with cash and ROAS. Mention pipeline as a separate
forecast, not as revenue. Cite leads to show funnel volume.

## Tool ordering — quick reference

- **"Which ads should I kill?"** → `creative_fatigue_check` first, then `top_creatives`.
- **"What's happening this week?"** → `compare_periods` (7d vs prior 7d), then
  `top_campaigns`, then `find_anomalies` if something looks off.
- **"Who's converting?"** → `top_campaigns` → pick a customer →
  `explain_customer_journey`.
- **"Why is ROAS dropping?"** → `compare_periods`, then `compare_attribution_models`
  to rule out model bias, then `find_anomalies` for day-level spikes.
- **"Are we fatigued on this campaign?"** → `explain_campaign` + `creative_fatigue_check`.
- **"Which model is right?"** → `compare_attribution_models`. Read the
  per-campaign `credit_share` per model, NOT the totals (totals are
  always equal across models because every conversion is fully credited).
- **"Where do we lose people?" / "What's our funnel conversion rate?"** →
  `get_funnel`. Walks `lead_created` → `appointment_booked` → `showed` →
  `qualified` → `closed_won` with stage-to-stage drop-off. For deal-only
  views pass `scope='pipeline_only'`.
- **"Which channel/country/page converts best?"** → `get_tracking_breakdown`
  with the relevant `dimensions`. The `revenue` field is cash only — same
  as ROAS. Combine with `get_performance_summary` for context.
- **"Which ad set is winning?"** → `top_ad_sets`. Use this between
  `top_campaigns` (broader) and `top_creatives` (narrower) to find the
  audience/placement layer that's delivering before drilling into ads.
- **"Show me the last N conversions / who paid this month?"** →
  `list_conversions` with `goal='payment_received'` (or another event_type).
  Returns paginated rows with masked PII by default. Distinct from
  `explain_customer_journey` (which is per-customer event timeline).
- **"Who are my top visitors / show me recent visitors?"** →
  `list_visitors`. Paginated by `last_seen_at DESC`. Includes both
  identified customers and anonymous visitors. PII masked by default.
- **"How trustworthy is my attribution data?" / "Why are some campaigns
  showing $0?"** → `get_attribution_quality` FIRST. It's a cheap diagnostic
  that surfaces UTM coverage. If `attributable_coverage_pct` is below 50%,
  recommend a UTM audit before trusting `top_campaigns` or
  `get_performance_summary` numbers.
- **"Is the site engaging?" / "Why is bounce rate high?" / "Where are
  users frustrated?" / "How are Core Web Vitals?"** → `get_engagement_summary`.
  Returns scroll depth, rage/dead clicks, form completion, video completion,
  Web Vitals. Pass `breakdown_by='page'` to find the worst-performing pages.
  Different from `get_tracking_breakdown` (traffic mix) — this is about UX
  quality. NEVER mix engagement counts with conversion counts in narration —
  engagement events are display-only and never become conversions.

`find_anomalies` is for "what's unusual," NOT "what's trending." Use
`compare_periods` for trends.

`get_funnel` counts events, not unique customers — one customer can appear
at multiple stages. Don't compute customer-level rates from these counts.

`get_funnel` only counts events whose `event_key` is also defined as a
conversion for the profile (i.e. the key appears in `conversion_definitions`).
If a stage like `disqualified` shows 0 but the team clearly tracks it
(e.g. labels like "Descualificado", "Lead Abandonado" exist under
Outcome Definitions), the conversion definition is missing. Tell the user
to map the event_key in **Settings > Outcomes > Conversion Definitions**.
Don't claim "you have 0 disqualified leads" without checking — that's a
common configuration gap, not a real zero.

`find_anomalies` is for "what's unusual," NOT "what's trending." Use
`compare_periods` for trends.

Never call `explain_customer_journey` without a specific `customer_profile_id`.

For deeper guidance on tool ordering, see [references/tool-ordering.md](references/tool-ordering.md).

## PII handling (non-negotiable)

The default response masks email, phone, and surname fields. `email` comes
back as `j***@e****.com`; `phone` as `********1234`; `name` as first name only.

To get unmasked PII, ALL THREE must be true:
1. The token carries the `mcp:read_pii` scope.
2. The request passes `include_sensitive=true`.
3. The workspace admin set `pii_mode=full_default` in **Settings > Privacy & MCP**.

Every response includes `meta.pii_level_applied` (`masked` or `full`) so you
can cite the effective level.

**When the user asks for unmasked PII**, check `whoami` output first:

- If `token.scopes` is missing `mcp:read_pii`: tell the user to generate
  a new token in **Developer > MCP Tokens** with that scope.
- If `active_workspace.pii_mode != "full_default"`: tell them a workspace
  admin must change PII mode in **Settings > Privacy & MCP**.
- Only if both pass: call `explain_customer_journey` with
  `include_sensitive: true`.

Don't make the call speculatively — `whoami` tells you whether it'll work.

## Errors

For each error code, surface the `action_url` from the envelope and use the
playbook in [references/error-playbook.md](references/error-playbook.md).

Codes you'll see most: `auth_required`, `token_expired`, `workspace_required`,
`profile_required`, `connector_expired`, `connector_required`, `rate_limited`,
`writeback_disabled`, `insufficient_scope`, `idempotency_conflict`,
`circuit_open`, `invalid_input`, `data_unavailable`, `internal_error`.

## Write-back protocol (send_meta_conversions)

**This tool ships events back to Meta. Treat every confirm as irreversible.**

Mandatory flow:
1. Call with `mode='preview'` first. Show the user the event_count and
   sample payload.
2. Ask the user to explicitly confirm in chat ("yes, send to Meta").
3. Generate an idempotency key (UUID v4). Keep it in the conversation —
   if the user asks to retry, use the same key.
4. Call with `mode='confirm'` + the idempotency_key.
5. Report the returned `audit_id` so the user can reference it.
6. On `idempotency_conflict`, surface the prior result. **Do not retry with
   a new key** — that would duplicate submissions.

If in doubt, prefer `mode='dry_run'` (validates with Meta's test_event_code
but doesn't write) over `mode='confirm'`.

## Narration style

- **Always cite `data_as_of`** ("as of 2 hours ago"). Connectors sync on
  different cadences; freshness matters.
- **Mention `data_freshness_warning` when present.** The server flags when a
  connector hasn't synced in >6 hours.
- **Describe fatigue by severity** (`none`/`mild`/`severe`), not raw CTR
  numbers.
- **When reporting across periods, show both numbers**, not just the delta.
- **Quote the workspace's currency symbol** — the amount fields in responses
  are already FX-normalized.
- Keep summaries tight. Users in Claude Code want answers, not essays.

## Performance & cost discipline

- **Call `list_workspaces` once per session.** The result is stable.
- **Don't call every tool.** Pick the ones that answer the user's question.
- **`compare_attribution_models` is heavier than others** (5 units). Only use
  when the user explicitly asks about model choice.
- Tool calls are billed in weighted units against your MCP quota. Most
  reads = 1-2 units; `compare_attribution_models` = 5 units;
  `send_meta_conversions` = 10 units. The **Developer > MCP Tokens** tab
  shows remaining units.
- For breakdown questions, prefer `get_tracking_breakdown` with multiple
  dimensions in one call (up to 6) over multiple separate calls — same
  cost, one round trip.

## Reference material

- [`references/tool-ordering.md`](references/tool-ordering.md) — detailed
  heuristics for which tool to reach for given common user questions.
- [`references/error-playbook.md`](references/error-playbook.md) — full
  error code handling guide.
- [`evals/evals.json`](evals/evals.json) — test cases for skill behavior.
