---
name: atribu-attribution
description: Query Atribu attribution data via the Atribu MCP server (mcp.atribu.app) — campaigns, creatives, workspace-level Top Performers (cohort-normalized creative scoring across all client profiles), customer journeys, creative fatigue, anomaly detection, and Meta CAPI write-back. Defaults to cash ROAS and masks PII. Use whenever the user asks about ad/campaign/creative performance, ROAS, CAC, customer journeys, "what's working", "best ads across clients", budget reallocation, cross-channel attribution, or sending conversions back to Meta.
when_to_use: User mentions "ROAS", "ad performance", "which ads to kill", "best performers", "what's working across clients", "top creative", "creative patterns", "creative fatigue", "customer journey", "compare periods", "anomalies", "send to Meta CAPI", "attribution", or asks any question about marketing campaign performance backed by Atribu.
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

Every per-profile data tool accepts `workspace_id` and `profile_id`
parameters. **Pass them explicitly** once you know them (from
`list_workspaces` / `list_profiles`). The server only infers defaults when
the user has exactly one workspace and exactly one profile — pretty rare.

`top_workspace_performers` is **workspace-scoped** — it spans all client
profiles in the workspace, so it takes `workspace_id` only (no
`profile_id`). It self-scopes guests to the profiles they can see.

Concurrent IDE sessions using the same token are safe: the token carries NO
persisted active scope. Each tool call is self-scoping.

## Top Performers (workspace-level creative scoring)

`top_workspace_performers` returns the best-performing ads across **all**
client profiles in a workspace, scored against *comparable* creative (a
"cohort" = channel × format × objective × audience warmth × geo × placement).
The score is cohort-normalized, empirical-Bayes-smoothed (small-sample ads
shrink toward the cohort mean — they land mid-pack, the honest "we don't
know yet"), and maturity-staged. Use it for "what's working across the
agency", spotting creative patterns to replicate, and ranking ads in a way
that's fair to small-budget accounts.

Each ad carries **three distinct measures — never merge them**:

- `composite_score` (0–100) — a *transparent rule-based blend* of the ad's
  within-cohort percentiles across funnel layers (delivery, attention,
  retention, click intent, post-click/messaging, attributed revenue). This
  is the headline rank.
- `top_performer_likelihood` (0–1) — a *probability*: a **likelihood, not a
  guarantee**. When `score_source='model'` it's the calibrated output of a
  LambdaMART creative ranker (`model_version` says which model — also surfaced
  at the top level of the response); otherwise (`score_source='rules'`) it's a
  monotone function of `composite_score`. Narrate it as "likelihood", never as
  "ROAS" or "lift", and say "the model estimates…" when `score_source='model'`.
- `attributed_revenue` / `roas` (and `attributed_outcomes_total` /
  `attributed_pipeline_outcomes` / `attributed_pipeline_value`) — *real
  Atribu-attributed outcomes*. `truth_grade='attributed'` means Atribu attributed
  ≥1 outcome of **any** kind (cash *or* pipeline — leads/appointments/closed-won).
  `meta_reported_conversions` is a *separate, lower-confidence* number — Meta's
  *own* last-click count for the ad, **not** Atribu's chain; never call it
  "attributed". Experimental lift (`truth_grade='lift'`) is the gold standard,
  set by the experiments layer.

`primary_outcome_kind` ∈ {`cash`,`pipeline`,`messaging`,`meta`,`none`} tells you
which outcome to *headline* per ad: `cash` → revenue/ROAS; `pipeline` → "N
attributed leads · $V pipeline value"; `messaging` → "N conversations started";
`meta` → "Meta reports N conversions" (say it's *Meta's* attribution, not
Atribu's); `none` → no attributed outcome yet — lead with `composite_score`.

`maturity_stage` tells you how much to trust the score: `cold` (just
launched — too little data), `early` (warming up), `mature` (proven),
`calibrated` (lift-tested). `reason_codes` explain *why* an ad ranks — which
funnel layer is strong or weak, each with a `sample_confidence` band; lead
with these when narrating ("strong hook — top of its cohort; sparse
post-click data — only N conversations so far").

Narration: lead with `composite_score` + maturity + the top reason code, then
the headline outcome per `primary_outcome_kind` ("and Atribu attributed 6 leads
worth $4.2k pipeline" / "Meta reports 18 conversions — Meta's own attribution").
Don't headline "$0 / 0x ROAS" for a lead-gen or Messages ad — use
`primary_outcome_kind`. Don't say "this ad has a 73% chance of being
a top performer" unless `top_performer_likelihood` literally says 0.73 —
and even then, frame it as a model estimate, not a guarantee. The
`creative.*` block (hook type, format, CTA, angle, offers) is the DNA to
copy when the user asks "make me more like this".

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

- **"Which ads should I kill?"** → `top_workspace_fatigue_risk` (Phase 4 — model-calibrated 30d pause probability × spend) first, then `top_creatives` for cross-reference with ROAS. The legacy `creative_fatigue_check` (heuristic CTR-delta) is the fallback when the Phase 4 model is off (`ML_FORECAST_AND_FATIGUE_ENABLED=false`) or for ads under 7 days active.
- **"What's the week ahead?" / "What should I worry about?"** → `forecast_workspace_outlook` (Phase 4) — one call returns the portfolio rollup: projected impressions + outcomes for next 7d with 80% PI bands, counts of emerging top performers + at-risk top performers, fatigue tier histogram, projected budget at risk. The agent's first lookup for portfolio-level questions.
- **"What's happening this week?"** → `compare_periods` (7d vs prior 7d), then
  `top_campaigns`, then `find_anomalies` if something looks off.
- **"Who's converting?"** → `top_campaigns` → pick a customer →
  `explain_customer_journey`.
- **"Why is ROAS dropping?"** → `compare_periods`, then `compare_attribution_models`
  to rule out model bias, then `find_anomalies` for day-level spikes.
- **"Are we fatigued on this campaign?"** → `explain_campaign` + `top_workspace_fatigue_risk` (or the legacy `creative_fatigue_check` when the Phase 4 Cox model isn't available).
- **"Are my top performers safe?" / "Will my best ads keep performing?"** → `forecast_workspace_outlook` for the `n_at_risk_top_performers` count, then `top_workspace_fatigue_risk` to identify which top-tier ads are at risk.
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
- **"What's working across all my clients?" / "Best ads in the agency?" /
  "Which creative should I replicate?" / "Find creative patterns"** →
  `top_workspace_performers`. Workspace-scoped (one call, all profiles);
  ranked by cohort-normalized `composite_score`. Lead with the score +
  `maturity_stage` + the top `reason_codes`; cite `attributed_revenue`/
  `roas` as a *separate* truth signal, and `top_performer_likelihood` as a
  *probability* — never blend the three. For "which is per-profile best?",
  filter `profile_ids`. Distinct from `top_creatives` (single profile,
  ranked by raw ROAS, no cohort normalization).
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

**This tool queues events for Meta — it does not send them itself.**
`confirm` enqueues rows in the export ledger (`conversion_exports`) plus a
pipeline-worker job; the actual Graph API call happens asynchronously, under
the booking cluster's canonical event_id, which is what lets Meta's 48h
dedup collapse it against the Pixel and any other ingestion path (browser
tracker, CRM webhook, bulk sync). Treat every confirm as irreversible even
though nothing has reached Meta when the call returns — once queued, the run
can't be recalled.

Mandatory flow:
1. Call with `mode='preview'` first. Show the user `counts.scoped` /
   `counts.would_enqueue` (plus `already_delivered` / `already_queued` if
   nonzero) and the destinations it would ship to. There is no sample event
   payload — preview returns counts and destination state, not events.
2. Ask the user to explicitly confirm in chat ("yes, send to Meta").
3. Generate an idempotency key (UUID v4). Keep it in the conversation —
   if the user asks to retry, use the same key.
4. Call with `mode='confirm'` + the idempotency_key. The response is
   `result='queued'`, never `'sent'` — nothing has reached Meta yet.
5. Report the returned `audit_id` AND `batch_id` so the user can reference
   both. Point them at the `observe` block in the response — `GET
   /api/v1/exports/{batch_id}` for this run, `GET /api/v1/exports/ledger`
   for delivery outcomes — as how to confirm the events actually landed at
   Meta.
6. On `idempotency_conflict`, surface the prior result (it already carries
   the original `batch_id`). **Do not retry with a new key** — that would
   duplicate submissions.

`mode='dry_run'` is the same plan as `preview`, additionally recorded in the
write-back audit log as intent — it does not queue anything and no longer
round-trips Meta's test-event endpoint (a tool that can't send synchronously
can't test-send). Use it to leave an audit trail without committing to a
run; it doesn't validate anything against Meta. `pixel_id`/`test_event_code`
are accepted for compatibility but don't route — routing comes from the
profile's configured export destination.

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

## Recommendations & write-back (Phase 5)

Atribu generates ranked, actionable recommendations per profile across four
kinds — backed by the same `creative_feature_store` scoring as Top
Performers, with a per-workspace LLM arbiter (default $2/month cap) ranking
the candidates:

- `scale_winner` — increase budget on consistently-performing ads
- `pause_underperformer` — pause ads with poor signal under maturity gate
- `budget_reallocate_winners` — shift budget from laggard → leader within an adset
- `creative_refresh_pre_fatigue` — external handoff to Ads Lab (no Meta write)

### Tools

- `list_workspace_recommendations(workspace_id?, profile_ids?, statuses?, kinds?, score_window?, limit?)` — read.
  - Defaults: `score_window='28d'`, `statuses=['open']`. Pass `['applied']` to see what was applied + the calibration follow-up.
- `diagnose_recommendation(recommendation_id)` — full audit trail (ranker features, arbiter rationale, apply history, calibration batch row when present).
- `apply_recommendation(recommendation_id, mode='preview'|'dry_run'|'confirm', idempotency_key?)` — write.
  - Requires `mcp:write` scope + workspace setting `mcp_writeback_enabled=true` + admin/owner role.
  - `mode='preview'` returns the planned Meta call(s); no audit row, no side effects.
  - `mode='dry_run'` writes a `mcp_writeback_audits` row with `result='dry_run'` (records intent without enqueuing).
  - `mode='confirm'` enqueues to `pgmq.q_recommendation_applies`; worker picks up within seconds.

### Apply-flow guarantees

- **Audit-before-action**: every `confirm` records an `mcp_writeback_audits` row BEFORE the Meta call.
- **Idempotency**: replays of the same `idempotency_key` for the same workspace return the existing `application_id` — no double Meta write, no duplicate row. Use a stable key per logical attempt.
- **Circuit breaker**: 3 apply failures within 30 min for a workspace blocks further applies (auto-reset after the window). If `apply_recommendation` returns `{ refused: 'circuit_breaker' }`, wait the window out.
- **T+5min verification**: post-apply, a `recommendation-verification-resync` cron re-reads Meta state and marks the application `verified_at` (or sets `meta_state_inconsistent=true` if Meta didn't reflect the change as expected).
- **Realized-vs-expected** (R-11 calibration): once an applied rec is ≥7 days old, the weekly `model-calibration-pass` cron writes `recommendation_calibration_batches` (`predicted_impact_dollars` vs `realized_impact_dollars`, `calibration_error`). `diagnose_recommendation` surfaces this. **Realized impact is observational (attributed in 7d post-apply window), NOT causal incremental lift** — that requires an active Phase 3 experiment.

### Common failure modes

- `circuit_breaker` → see above. Wait the 30min window.
- `meta_state_inconsistent=true` on verification → Meta UI may have been edited manually between apply + verification. Surface to user; don't auto-rollback.
- `403 insufficient_scope` on REST apply → API key needs `campaigns:apply` scope (separate from `campaigns:read` / `campaigns:write`). Mint via Developer > API Keys.
- `pgrst 42501 permission denied` on direct PostgREST → the apply RPC is `service_role`-only; use the MCP `apply_recommendation` tool or the REST `POST /api/v1/recommendations/{id}/apply` route.

## Onboarding

Everything above assumes a workspace that is already configured. This section
is the other half: taking an agent from an empty workspace to a live CAPI
destination, over MCP, end to end.

**Call `whoami` first, every session.** It costs 0 units and returns a
`readiness` summary — how many golden-path steps are done, which is next,
and every step that is not done, with why and what unblocks it. An empty
dashboard is almost always a missing connection, not a missing result, and
this is where that shows.

### The order, and why it is the order

Each step needs the one before it. Skipping is what produces a workspace
that looks finished and reports zeroes.

1. **`create_workspace`** — only when `whoami` shows none. You become the owner.
2. **Write-back starts ON for a workspace this call creates.** A workspace
   minted by `create_workspace` has `mcp_writeback_enabled=true` from
   creation — no human has to flip anything before the write tools below
   will run. Only a workspace created some other way — in the console, or
   by a Shopify/partner-app install — can have it off. Check `whoami`:
   if `active_workspace.mcp_writeback_enabled` is `false`, say so in one
   sentence and hand over the `action_url` from the `writeback_disabled`
   error — do not narrate the whole settings page.
3. **`create_profile`** — one advertiser/brand/client. Everything after this
   is scoped to it. (Or `create_demo_profile`, when the user wants to see the
   product working before connecting anything real.)
4. **`issue_tracking_key`** → **`get_tracker_installer`** — mint the key,
   then hand over the installer **verbatim**. Ask which surface first
   (`snippet` / `gtm` / `shopify_pixel`); guessing produces an install they
   cannot use.
5. **`start_connect({ provider })`** — give the user the `url` exactly as
   returned, then poll `get_handoff({ id })`. Do not open it, do not shorten
   it, do not summarise it.
6. **`list_outcome_events`** → **`suggest_conversion_definitions`** →
   **`create_conversion_definition`** — read what the profile actually
   emits, propose from that, create what the user confirms. Never invent an
   event name.
7. **`set_attribution_windows`** — only if the customer's sales cycle needs
   it. It rewrites already-reported numbers, so say so before confirming.
7b. **`sign_dpa`** — Conversion Sync exports nothing until the DPA is
   accepted. You cannot sign it; hand over the URL. `already_signed: true`
   is a success — say so and move on rather than offering a link.
8. **`configure_meta_capi`** → **`send_test_event`** — the dataset and its
   rules, then one synthetic event the customer watches land in Events
   Manager.

### Rules the skill enforces

- **Preview, show, confirm.** Every write tool takes `mode`. Run `preview`,
  put its `will_call` and (where present) its dry-run counts in front of the
  user in their own words, and only then `confirm`. Never confirm in the
  same turn you previewed unless the user already said yes to that exact
  thing.
- **A same-name `create_workspace` replay from the same caller is safe
  within a short window** — it answers `created: false` and returns the
  ORIGINAL workspace, never a second one. A **different** name is always a
  different workspace. If you did not read the outcome of a call (timeout,
  dropped connection), retry with the identical name rather than guessing a
  new one — do not silently rename to "try again", and do not assume success
  or failure without telling the user which happened.
- **Read the error CODE, not the prose.** `insufficient_scope` → the token
  needs re-consent. `insufficient_role` → a human with an owner/admin role
  has to act; re-consenting cannot fix it. `writeback_disabled` → one
  setting, one URL (see step 2). `connector_required` → `start_connect`.
  Each carries an `action_url`; give that link, do not describe the path.
- **Only `cash` counts for ROAS.** When creating a conversion definition for
  what the customer calls "sales" or "revenue", that is
  `revenue_type: "cash"`. `pipeline` fills the funnel and contributes
  nothing to any ratio — a dashboard of zeroes with no error anywhere.
- **A hand-off is the user's to complete.** Poll `get_handoff` every few
  seconds; do not mint a second one while the first is pending; an
  `expired` status means mint a fresh one, a 404 means the id is wrong.
- **A partial `configure_meta_capi` is not a success.** It returns
  `complete: false` and `failed_rules`. Say which conversions will not
  reach Meta rather than reporting the destination as done.

## Reference material

- [`references/tool-ordering.md`](references/tool-ordering.md) — detailed
  heuristics for which tool to reach for given common user questions.
- [`references/error-playbook.md`](references/error-playbook.md) — full
  error code handling guide.
- [`evals/evals.json`](evals/evals.json) — test cases for skill behavior.
