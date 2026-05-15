# Tool Ordering Heuristics

Detailed guidance on which tool to reach for given common user questions. SKILL.md has the summary.

## "Which ads should I kill?"

1. `top_workspace_fatigue_risk` (Phase 4) — model-calibrated 30-day pause probability × spend (the budget-at-risk metric). Ads in fatigue tier `high`/`critical` are predicted to pause; sort by `risk_rank_score` descending.
2. `top_creatives` ordered by ROAS — see which at-risk ads also have weak attribution
3. Cross-reference: kill ads that are both at-risk AND low ROAS. Just-at-risk or just-low-ROAS ads need investigation.

`creative_fatigue_check` is the legacy heuristic 2-window CTR/CPM delta — useful as a flag when the Phase-4 Cox model is gated off (`ML_FORECAST_AND_FATIGUE_ENABLED=false`) or for a workspace whose ads are too young (`days_active < 7`) for the calibrated model. Prefer `top_workspace_fatigue_risk` once the model is live.

## "What's the week ahead?" / "What should I worry about?" / "Where am I losing budget?"

1. `forecast_workspace_outlook` (Phase 4) — one call. Returns the portfolio rollup: projected impressions + outcomes for the next 7 days (with 80% prediction-interval bands from split-conformal calibration), counts of `n_emerging_top_performers` (climbing ads) + `n_at_risk_top_performers` (top-tier ads in high/critical fatigue), per-tier fatigue histograms, and `projected_budget_at_risk` (total spend across at-risk ads). The agent's first lookup for portfolio-level questions.
2. If `projected_budget_at_risk > 0` and `n_at_risk_top_performers > 0`: drill in with `top_workspace_fatigue_risk` to see WHICH ads carry that risk.
3. If `n_emerging_top_performers > 0`: cross-reference with `top_workspace_performers` to see the climbers — these are the candidates to scale.

Caveats:
- Empty-workspace return reads "No ads scored yet" or "all under cold-start gate" — don't say "looks healthy" in those cases.
- The PI bands are calibrated 80% coverage, not Gaussian σ. Describe them as "the model is 80% confident the true value lies between X and Y", not "1-sigma".
- The Phase 4 model is OFF by default; if `oldest_forecast_at` is null the cron's prediction pass hasn't run for this workspace yet.

## "What's happening this week?"

1. `compare_periods` — last 7 days vs prior 7 days
2. `top_campaigns` — see which campaigns drove the change
3. `find_anomalies` — only if a metric jumped or dropped sharply, identify which day(s)

Don't lead with `find_anomalies` for a "what's happening" question — it's noise unless the user is asking about something unusual.

## "Who's converting?"

1. `top_campaigns` — get the highest-revenue campaigns
2. Pick a specific high-revenue customer from the data
3. `explain_customer_journey` for that customer — show the full path

Never call `explain_customer_journey` without a `customer_profile_id` you got from elsewhere. The tool only works for one customer at a time.

## "Why is ROAS dropping?"

1. `compare_periods` — confirm the drop is real, not noise
2. `compare_attribution_models` — rule out model bias (sometimes ROAS "drop" is just a shift in attribution distribution)
3. `find_anomalies` — find day-level spikes/drops in spend or revenue that explain the trend
4. `top_campaigns` — see which campaigns are dragging the weighted average down

## "Are we fatigued on this campaign?"

1. `explain_campaign` for the named campaign
2. `top_workspace_fatigue_risk` (Phase 4) to get model-calibrated 30-day pause hazards for ads in that campaign (or `creative_fatigue_check` for the legacy heuristic when the Phase-4 model isn't available)
3. Look at the daily trend in `explain_campaign` to confirm declining ROAS over time

## "Are my top performers safe?" / "Will my best ads keep performing?"

1. `forecast_workspace_outlook` (Phase 4) — the `n_at_risk_top_performers` count is the headline answer
2. If non-zero, `top_workspace_fatigue_risk` to identify which top-tier ads are at risk
3. Compare to `top_workspace_performers` to confirm which are the current top performers

## "What should I budget more on?"

1. `top_campaigns` ordered by ROAS
2. Cross-reference with `top_workspace_fatigue_risk` (Phase 4) — don't recommend pouring budget into an ad predicted to pause soon. The legacy `creative_fatigue_check` is the fallback when the calibrated model isn't available.
3. Mention the `data_as_of` so the user knows whether the data is fresh

## "Did the WhatsApp ads work?"

1. `whatsapp_attribution_summary` for the relevant window
2. Compare to `top_campaigns` to see what % of total revenue WhatsApp drove

## "What's working across all my clients?" / "Best ads in the agency?" / "What creative should I replicate?"

1. `top_workspace_performers` — one call, all profiles in the workspace, ranked by cohort-normalized `composite_score` (an ad is scored against *comparable* creative, so a $500/mo clinic and a $30k/mo clinic are compared fairly).
2. Read the top rows: lead with `composite_score` + `maturity_stage`, then the top 1-2 `reason_codes` ("strong hook — top of its cohort", "cheap conversations"). Then the *headline outcome* per `primary_outcome_kind` as a *separate* line: `cash` → "$X attributed cash · Yx ROAS"; `pipeline` → "N attributed leads · $V pipeline value"; `messaging` → "N conversations started"; `meta` → "Meta reports N conversions" (Meta's *own* attribution, not Atribu's — say so); `none` → skip it. `truth_grade='attributed'` ⇔ ≥1 Atribu-attributed outcome of any kind. `top_performer_likelihood` is a probability; narrate it as one, never as ROAS. Don't headline "$0 / 0x ROAS" for a lead-gen or Messages ad — `primary_outcome_kind` is exactly there to prevent that.
3. To replicate: the winner's `creative.*` block (hook_type, format, cta_type, primary_angle, offers, pain_points_addressed) is the DNA. Tell the user "the pattern is X — UGC video, question hook, send-DM CTA, [pain point] angle".
4. For "which is my single best ad per client?", call again with `profile_ids: [...]` or just group the rows by `profile_id` client-side.

Don't reach for `top_creatives` here — that's single-profile, ranked by raw ROAS, with no cohort normalization (so it'll over-rank the big-budget accounts). `top_workspace_performers` is the agency-wide, apples-to-apples view.

Caveats:
- Trust the score by `maturity_stage`: `cold` ads have too little data — say so. `mature`/`calibrated` are the ones to act on.
- `reason_codes` carry a `sample_confidence` band. A `low`-confidence "strong" signal on a 2-day-old ad is a guess, not a verdict.
- If `top_workspace_performers` returns nothing, the workspace's creative scores haven't been built yet (the daily score-rebuild runs ~12:00 UTC; a fresh Meta connection takes a sync cycle first). Fall back to per-profile `top_creatives`.

## "Send last week's purchases to Meta"

See the dedicated write-back protocol section in SKILL.md. Always: preview → user-confirm → confirm with idempotency_key.

## Anti-patterns

- **Don't call every tool**. Pick the 1-3 that answer the user's question. Each call costs units.
- **Don't call `compare_attribution_models` for trend questions**. It's the heaviest tool (5 units). Only use when the user asks "which channel really matters?" or "is our model right?"
- **Don't call `list_workspaces` more than once per session**. The result is stable.
- **Don't pre-call tools speculatively**. Wait for the user to ask, then call the right tool.
