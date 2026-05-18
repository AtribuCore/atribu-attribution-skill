# Error Playbook

Detailed handling for every MCP error code. Loaded on demand — see SKILL.md for the summary.

Every error envelope looks like:
```json
{
  "error": {
    "code": "...",
    "message": "...",
    "retryable": true,
    "action_url": "https://...",
    "support_hint": "...",
    "request_id": "01968a3b-..."
  }
}
```

## Authentication

### `auth_required`
Token is missing from the request.

**What to tell the user:**
> Your MCP token is missing or malformed. Generate one in **Developer > MCP Tokens**, then re-run `claude mcp add atribu` with the token.

### `token_expired`
Token was revoked or expired.

**What to tell the user:**
> Your token was revoked. Visit **Developer > MCP Tokens** to create a new one and update your MCP config.

## Scope resolution

### `workspace_required`
User has multiple workspaces and the call didn't pass `workspace_id`. The error envelope includes `workspaces: [{id, name}, ...]`.

**What to do:**
1. Surface the workspace list to the user
2. Ask which one ("Which workspace? Acme Inc, Beta Corp, ...")
3. Pass `workspace_id` in the next call
4. Remember it for the rest of the session

Do **not** call `set_active_workspace` — there is no such tool. Each call is self-scoping.

### `profile_required`
Same pattern, but for profile within an already-resolved workspace. Use `profiles: [...]` hint.

### `insufficient_scope`
Token lacks the required scope.

**What to tell the user:**
> Your token doesn't have the `<scope>` scope. Generate a new token in **Developer > MCP Tokens** with that scope enabled.

## Connector freshness

### `connector_expired`
A required integration's OAuth has expired. The data is stale and would be wrong.

**What to do:**
1. Show the `action_url` verbatim
2. Tell the user to reconnect
3. Do **not** try to work around it by querying older data

### `connector_required`
A required integration was never connected.

**What to do:**
1. Name the missing provider
2. Show the `action_url` (Settings > Integrations)

## Rate limits

### `rate_limited`
Either the per-minute burst cap (120 units) or the per-period subscription cap was hit.

**What to tell the user:**
> You've used X of Y {trial|monthly} units. Wait `retry_after` seconds, then retry.

If `retry_after` > 60 seconds, the user is hitting the period cap, not the per-minute cap. Suggest upgrading via the `action_url`.

## Write-back

### `writeback_disabled`
Workspace admin hasn't enabled MCP write-back.

**What to tell the user:**
> A workspace admin must enable write-back in **Settings > Privacy & MCP** before I can send conversions to Meta.

### `idempotency_conflict`
A `confirm` with this `idempotency_key` was already processed.

**What to do:**
1. Surface the prior result to the user (the error includes the original `audit_id`)
2. Do **not** retry with a new key — that would duplicate the submission
3. Treat the conflict as success

### `circuit_open`
3+ consecutive `confirm` failures in the last 30 minutes for this profile.

**What to do:**
1. Stop all write-back attempts
2. Tell the user to check the audit log in the dashboard
3. Investigate the underlying failures before retrying

## Recommendation apply (Phase 5)

### `circuit_breaker` (from `apply_recommendation`)
Same 3-failures-in-30-minutes rule as `send_meta_conversions`, scoped per
workspace. Auto-resets when the window passes (no manual reset RPC).

**What to do:** Tell the user the breaker is open and the window expires in
N minutes. Surface `recent_failures[]` so they can see what failed. Don't
retry inside the window.

### `meta_state_inconsistent=true` (on verification)
The T+5min verification cron re-read Meta and found the state didn't match
expected. Most common cause: someone edited the ad/adset in Meta Ads
Manager between Atribu's apply and the verification check.

**What to do:** Surface to the user — don't auto-rollback. The application
row stays `verified_at=NULL` until they decide. They can either:
- Accept the manual change (mark the rec dismissed)
- Re-apply through Atribu (uses a fresh idempotency key)

### `403 insufficient_scope` (on REST apply)
The API key being used doesn't have `campaigns:apply`. This is a separate
scope from `campaigns:read` / `campaigns:write` and only granted to keys
where the workspace admin has enabled write-back.

**What to do:** Tell the user to mint a new key in Developer > API Keys with
the `campaigns:apply` scope checked. Existing keys are NOT auto-upgraded.

### `pgrst 42501 permission denied` (direct PostgREST call)
The `send_recommendation_apply_job` RPC is `service_role`-only. PostgREST
clients (authenticated or anon) cannot call it directly.

**What to do:** Use the MCP `apply_recommendation` tool OR the public REST
route `POST /api/v1/recommendations/{id}/apply` — both go through the
server-side trust boundary that holds the service-role key.

## Input

### `invalid_input`
Bad parameters (wrong type, missing required field, out of range).

**What to do:** Re-read the tool's parameter schema and fix the call.

### `data_unavailable`
The requested entity or window has no data.

**What to tell the user:** "No data for that range/customer/campaign. Try widening the date window."

## Server

### `internal_error`
Unexpected server error. `retryable: true`.

**What to do:** Retry once. If it persists, surface the `request_id` to the user with a note to contact support.
