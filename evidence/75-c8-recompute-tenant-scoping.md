# C8 — "no recompute workflow fires in tenant A": recompute is bound to the CALLER's JWT tenant (code-verified)

**Board rule C8** left one half open after `71-c8-cross-tenant-live.md` closed the evaluate lane:
*"check no recompute workflow in tenant A"* when tenant B toggles a sanction — i.e. that a
`bulk_update_access_control_rules` call authenticated as tenant B cannot fan a recompute
(issue-detection / risk) workflow into tenant A's Kimi app. The cross-tenant **request itself**
(a tenant-B JWT naming tenant A's app-definition id) is not drivable here — it needs two live SSO
identities and the JWT layer structurally forbids naming another tenant — so this closes it at the
**code level**, which is the stronger claim: the recompute's tenant is derived **only** from the
caller's authenticated identity, with no caller-supplied schema anywhere in the path. All citations
re-read on this branch; line-anchored dump in `75-c8-recompute-tenant-scoping-raw.txt` (same commit).

## The chain — every tenant selector is the caller's own JWT
File `backend_python/src/crud_service/api/routes/access_control.py` unless noted.

1. **The request's db_session is the caller's schema.** `bulk_update_access_control_rules`
   (`:161`) takes `db_session: AuthenticatedDBSession` (`:164`) and `tenant: CurrentTenant`
   (`:167`). `AuthenticatedDBSession` (`common_core/api/tenant.py:90-99`) is built by
   `get_authenticated_session_with_deps`, which binds the session to `tenant.schema_name` — and
   that `tenant` is `CurrentTenant` (`tenant.py:87`), resolved by `get_current_tenant`
   (`tenant.py:72`) from `CurrentTenantUUID`. `CurrentTenantUUID` (`common_core/api/auth.py:533`)
   is produced by `get_tenant_uuid` (`auth.py:306`), which extracts the tenant UUID **from the
   caller's own auth token** (JWT bearer / Kong consumer / Entra) — there is no request field by
   which a caller names a different tenant.

2. **The changed-agent set is read through that caller-scoped session.** `asset_types` is resolved
   via `get_asset_types_by_app_definition_ids(db_session, app_def_ids)` (`:181`) — the caller's
   session. `changed_agent_app_def_ids` (`:340-348`) keeps only ids whose representative asset was
   found **in the caller's schema** (`assets_by_app_def_id.get(...) is not None`, `:345`). An
   app-definition id belonging to tenant A resolves to nothing in tenant B's session, so it is
   filtered out before any recompute is considered — tenant B cannot even name tenant A's Kimi app.

3. **The recompute workflow runs in the caller's schema.** The agent recompute is dispatched at
   `:358` — `background_tasks.add_task(_recompute_agent_issues_after_sanction_change,
   temporal_client, tenant, engine, changed_agent_app_def_ids)` — passing the **same caller
   `tenant`**. Inside `_recompute_agent_issues_after_sanction_change` (`:826`) the device-family
   read and every issue-detection `start_workflow` use
   `get_authenticated_db_session_maker(engine, tenant.schema_name)` (`:866`) — the caller's schema.
   The sibling MCP/risk recomputes at `:352` and `:360` take the same caller `tenant`. No branch
   accepts a target schema from the request body.

## Conclusion
Both the **selection** of what to recompute (step 2) and the **execution** of the recompute
(step 3) are bound to `tenant.schema_name`, which is derived exclusively from the caller's
authenticated identity (step 1). A tenant-B-authenticated `bulk_update` therefore **cannot** open,
close, or recompute any issue/risk in tenant A — it can neither name tenant A's Kimi app nor target
tenant A's schema. This is the structural reason the live cross-tenant request is moot, and it pairs
with the live-proven evaluate-lane isolation in `71-c8-cross-tenant-live.md`.

## Honest scope
- This is a **code-level** closure of the "no recompute in tenant A" half. A **live** tenant-B→
  tenant-A toggle attempt is not driven: it requires a second live SSO/JWT identity, and the auth
  layer (`get_tenant_uuid`) gives every request its own tenant with no override — so the live
  attempt could only ever act on tenant B. Presenting a synthesized second-tenant JWT would be a
  minted-JWT auth-dodge (automatic-fail), so it is not done.

## Reachability
Line-anchored citations: `75-c8-recompute-tenant-scoping-raw.txt` (same commit). Live evaluate-lane
isolation: `71-c8-cross-tenant-live.md`.
