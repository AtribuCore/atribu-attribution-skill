# Tool Ordering Heuristics

Detailed guidance on which tool to reach for given common user questions. SKILL.md has the summary.

## "Which ads should I kill?"

1. `creative_fatigue_check` — gets fatigue signals (CTR drop, CPM rise) for the last 14 days vs prior 14 days
2. `top_creatives` ordered by ROAS — see which low-ROAS ads also show fatigue
3. Cross-reference: kill ads that are both fatigued AND low ROAS. Just-fatigued or just-low-ROAS ads need investigation.

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
2. `creative_fatigue_check` to get fatigue scores for ads in that campaign
3. Look at the daily trend in `explain_campaign` to confirm declining ROAS over time

## "What should I budget more on?"

1. `top_campaigns` ordered by ROAS
2. Cross-reference with `creative_fatigue_check` — don't recommend pouring budget into a fatigued campaign
3. Mention the `data_as_of` so the user knows whether the data is fresh

## "Did the WhatsApp ads work?"

1. `whatsapp_attribution_summary` for the relevant window
2. Compare to `top_campaigns` to see what % of total revenue WhatsApp drove

## "What's working across all my clients?" / "Best ads in the agency?" / "What creative should I replicate?"

1. `top_workspace_performers` — one call, all profiles in the workspace, ranked by cohort-normalized `composite_score` (an ad is scored against *comparable* creative, so a $500/mo clinic and a $30k/mo clinic are compared fairly).
2. Read the top rows: lead with `composite_score` + `maturity_stage`, then the top 1-2 `reason_codes` ("strong hook — top of its cohort", "cheap conversations"). Cite `attributed_revenue`/`roas` as a *separate* line when `truth_grade='attributed'` — it's the real-cash signal, not the same thing as the score. `top_performer_likelihood` is a probability; narrate it as one, never as ROAS.
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
