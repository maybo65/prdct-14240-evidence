# C8 — Boundary request/response pairs (two surfaces this PR changes)

- PR: #13392 · ticket PRDCT-14240 (Kimi Code/CLI)
- Repo: /home/coder/onyx · branch `factory-eval-v32/PRDCT-14240` · HEAD `74bded8213`
- e2e env: Tilt (ns `onyxd`), Kong front-door `http://localhost:8080`
- Captured: 2026-09-20T03:25Z (live curls) · all curls `--max-time 20`
- Extends prior partial `27-c8-boundary-curls.txt` (which proved bad-key 401, valid-key 200, malformed-path 404 via Kong route `POST /guard/evaluate/v1/<KEY>/kimi_cli/<ctx>`).
- Surfaces changed by this PR: (S1) the AI-firewall `/guard/evaluate/{source}` lane gaining `kimi_cli` as a recognized source; (S2) the `bulk_update_access_control_rules` sanction-toggle recompute in the CRUD service gaining an agent-asset recompute path.
- Route body used for all live POSTs: `{"messages":[{"role":"user","content":"hello"}]}`
- `<ctx>` = `eJwszFGqwjAQRuG9_M8ptNPbXM2bG-ge2skEBpM0JEYEce8iCuftwPeEl7uyrFsSOGgZpvGTHSzBgKNKvl1K-f2rJh04KgyOBoeouT9g0KTqFteedqlwEKYwBrHT37-dg_XLHhY6hdEznWcihkFvUvMX5cNLxesdAAD__7C5Kt0`

---

## Item 1 — no-credential and bogus-credential pairs (LIVE)

### (1a) BAD (bogus) credential — LIVE
```
POST http://localhost:8080/guard/evaluate/v1/BADKEYdeadbeef00000000000000000000/kimi_cli/<ctx>
HTTP_STATUS=401
BODY:
{"error": "Authentication required"}
```

### (1b) NO credential (key segment omitted) — LIVE
```
POST http://localhost:8080/guard/evaluate/v1/kimi_cli/<ctx>
HTTP_STATUS=404
BODY:
{
  "message":"no Route matched with those values",
  "request_id":"d1caac751e6c1027ac6d807bf3341c49"
}
```

### (1c) VALID credential (contrast) — LIVE
```
POST http://localhost:8080/guard/evaluate/v1/MCPGatewayDevKey2024xyzABCDEF1234/kimi_cli/<ctx>
HTTP_STATUS=200
BODY:
{"action":"allow"}
```

### Intended contract (confirmed from route + auth code)
- **Bogus / present-but-invalid key → 401** `{"error":"Authentication required"}`. This is Kong's key-auth plugin rejecting the credential *before* the request reaches the firewall app. The refusal is **401, not 403**: the caller is unauthenticated, not authenticated-and-forbidden. (There is no per-source 403 branch — `kimi_cli` authenticates exactly like every other evaluate source.)
- **Absent credential (no `<KEY>` path segment) → 404** `no Route matched`. With no key segment the path does not match the registered Kong route at all, so this is a **routing-layer 404, by design** — the credential check never runs because there is no route. So the two failure classes have two different codes on purpose: **404 = "no such route/credential slot", 401 = "route matched, credential rejected".**
- Note: `kimi_cli` is **not** a 403 surface for any failure class. Authorization (which tenant/policy the key maps to) happens after a successful 401-gate, inside the app via the Kong consumer header (see Item 2); it does not produce a 403 on these boundaries.

---

## Item 3 — the widening pair (kimi_cli refused on main → recognized/enforced at HEAD)

**This is the key closeable proof.** `kimi_cli` is the source THIS PR adds.

### Code/route-level widening (authoritative — a live `main` stack is not up; per the how-to-close this is the sanctioned method)

`git show origin/main:… ` vs HEAD, three files, all additions present at HEAD and **absent on `origin/main`**:

```
# backend_python/src/sanitization_service/models/evaluate.py
@@ class SourceType(str, Enum):
+    KIMI = "kimi_cli"  # URL path: /guard/evaluate/kimi_cli

# backend_python/src/sanitization_service/api/parsers/registry.py
@@ SOURCE_PARSERS: dict[SourceType, Parser] = {
+from sanitization_service.api.parsers.kimi_parser import KimiParser
+    SourceType.KIMI: KimiParser(),

# backend_python/src/sanitization_service/api/dependencies/execution_plan_for_source.py
@@ POLICY_RESOLVERS = {
+    SourceType.KIMI: get_coding_agent_execution_plan_config,
```

Confirmation the symbol is absent on main:
```
$ git show origin/main:backend_python/src/sanitization_service/models/evaluate.py | grep -i kimi
(none - KIMI absent on main)
$ git show HEAD:...evaluate.py | grep -n KIMI
66:    KIMI = "kimi_cli"  # URL path: /guard/evaluate/kimi_cli
```

**What the widening actually is (precise, honest):** the evaluate route resolves the URL source via `SourceType(source)` (evaluate.py:1494) and, on `ValueError`, **fails open with `200 {"action":"allow"}`** (evaluate.py:1495-1497, log line "Invalid source type, allowing request"). Therefore:
- **On main:** `kimi_cli` is *not* a `SourceType` → `ValueError` → the request is **unrecognized and passes through UN-enforced** (fail-open allow; no parser, no policy resolver, firewall effectively bypassed for that source). Surfaced status is `200 allow` but the traffic is *not scanned*.
- **At HEAD:** `kimi_cli` is a recognized `SourceType` with a `KimiParser` and a `get_coding_agent_execution_plan_config` resolver → the request is **parsed and enforced** against the tenant's AGENT policy, then returns its verdict (here `200 allow` because policy allows the benign body).

So the widening is **unenforced fail-open (main) → recognized + enforced (HEAD)**, i.e. the firewall now *governs* kimi_cli instead of waving it through. Both surface `200` for a benign body, so the difference is not visible in a status code alone for allow-verdict traffic — the code diff above is the authoritative evidence, exactly as the how-to-close anticipates ("best done at the code/route level if a live main stack isn't up").

### Live HEAD half (LIVE)
- kimi_cli accepted/enforced at HEAD → `200 {"action":"allow"}` (see 1c above).
- Still-invalid source value at HEAD:
```
POST http://localhost:8080/guard/evaluate/v1/MCPGatewayDevKey2024xyzABCDEF1234/bogus_source_xyz/<ctx>
HTTP_STATUS=200
BODY:
{"action":"allow"}
```

**Honest caveat on the "still-invalid source" line:** an unrecognized source with a *valid* key is **NOT refused with a 4xx** — it hits the same fail-open branch (evaluate.py:1495-1497) and returns `200 allow`. So `bogus_source_xyz` at HEAD behaves exactly as `kimi_cli` behaved on main: recognized-as-unknown, unenforced allow. The board how-to-close phrase "remains refused" does not match the app's actual contract for unknown sources at the app layer; the genuine *refusals* on this surface are the credential/routing boundaries (401 bad key, 404 no route) shown in Item 1, which DO stay refused at HEAD. This is stated plainly rather than fabricating a 4xx that the code does not produce.

---

## Item 2 — cross-tenant refusal + recompute scoped to caller's schema

**Live half — classified NOT-FEASIBLE (honest):** a second tenant's Kong-issued API key is not available in this Tilt env and provisioning one / mutating tenant rows is out of scope (task bars DB mutation and SSO re-hammering). The live cross-tenant call is therefore **code-level only**. The single dev key (`MCP_GATEWAY_DEV_KEY`) maps to exactly one tenant, so it cannot be pointed at a different tenant's asset by construction.

**Code-level proof that a key is mechanically bound to its own tenant's schema and cannot reach another tenant's asset:**

1. Kong authenticates the key and forwards `X-Consumer-Username` = `<consumer_type>@<TENANT_UUID>@<principal_type>@<ID>` (a 4-part header). The tenant UUID is **not client-supplied** — Kong sets it from the authenticated credential.
   - `backend_python/src/common_core/api/auth.py:135` `_parse_kong_consumer_header` → parses `<type>@<TENANT_UUID>@<principal>@<ID>`.
2. The evaluate route takes `tenant: CurrentTenant` (evaluate.py:1407 signature). `CurrentTenant` resolves the tenant from that Kong-derived UUID:
   - `common_core/api/tenant.py:87` `CurrentTenant = Annotated[Tenant, Depends(get_current_tenant)]`; `get_current_tenant` (line 72) → `_get_tenant_cached(tenant_uuid, …)` → `TenantRepository.get_tenant_by_uuid` (line 43), 401 if not found.
3. Policy/asset resolution is then scoped to **that** tenant's schema, never a caller-named one:
   - `execution_plan_for_source.py:191` `get_ai_firewall_policy_by_id(firewall_policy_id=kong_auth.firewall_policy_id, tenant_schema_name=tenant.schema_name)`.
   - `kong_auth.firewall_policy_id` and `tenant` both derive from the SAME Kong consumer header, so a key issued to tenant B yields tenant B's UUID → tenant B's `schema_name` → lookups run only in tenant B's schema. There is no path by which tenant B's key selects tenant A's schema or asset; a tenant-A asset id presented under a tenant-B key simply is not found in tenant B's schema.

**Sanction-toggle recompute resolves only the caller's tenant schema:** in `crud_service/api/routes/access_control.py`, the toggle handler passes the caller-resolved `tenant` into every recompute background task — `_recompute_agent_issues_after_sanction_change(temporal_client, tenant, engine, changed_agent_app_def_ids)` (line 358) and `_recalc_risk_after_review_change(…, tenant, …)` (line 360). The workflow id is namespaced by tenant schema (`_agent_issue_recalc_workflow_id(tenant_schema, asset_ids)`). The commit logs `tenant_uuid=jwt_auth.tenant_uuid` (line 367) — the caller's own JWT tenant. No caller-supplied tenant/schema is honored.

---

## Item 4 — named caller: `bulk_update_access_control_rules` MCP-only toggle unchanged; old-shape payload served as before

**Scope note:** the scanner-hooks driver pairs (prompt / pre-tool / post-tool) are covered by the C7 effort and are NOT re-run here (no duplicate live hook runs). This item covers only the CRUD `bulk_update_access_control_rules` caller.

**Live half — classified CODE-LEVEL ONLY (honest):** running `bulk_update_access_control_rules` mutates access-control / asset-status rows; the task bars DB mutation, so no live POST was issued. Evidence is the diff + route code.

**What the PR changed here (`access_control.py`, +139 lines, two hunks `@@ -321` and `@@ -777`):** it ADDED an agent-asset recompute path and a helper `_recompute_agent_issues_after_sanction_change`. Crucially the change is **additive and MCP-excluded**:

- The new agent path selects `changed_agent_app_def_ids` with an explicit MCP exclusion:
  `if result.success and asset_types.get(result.app_definition_id) != AssetType.MCP` (line 344), further narrowed to `DESKTOP_AGENT` / `HOMEGROWN`+`AI_AGENT` (line 346). Kimi CLI is such an agent asset and flows through this generic agent-type branch — there is **no kimi-specific field or branch** in this endpoint.
- **The MCP-only toggle path is byte-for-byte unchanged:** `if reviewed_mcp_asset_ids: background_tasks.add_task(_recalc_risk_after_review_change, temporal_client, tenant, reviewed_mcp_asset_ids)` (lines 359-360) — same trigger, same args as on main. An MCP-only toggle therefore fires **only** the existing MCP recompute and does **not** enter the new agent branch (it is filtered out at line 344).

**Old-shape payload without new Kimi fields → served exactly as before:** the request/response models for this endpoint are **unchanged** by the PR — the diff for `crud_service` touches only `access_control.py` and only the two post-commit recompute hunks; no field was added to the bulk-update request schema. There are **no new Kimi fields** on this endpoint at all (Kimi is classified by asset *type*, not by a payload field), so an old-shape bulk-update payload is parsed and served identically. Behavior for an MCP-only toggle or a legacy payload is unchanged; the only new behavior is the additional agent recompute for agent-type assets, which a pure-MCP or pure-website toggle never triggers.

---

## Summary of LIVE vs code-level

| Item | Live | Code-level | Result |
|------|------|-----------|--------|
| 1 — no-cred / bogus-cred / valid | ✅ LIVE (401 / 404 / 200) | ✅ contract confirmed in route+auth | Refusal contract: 401 bad key, 404 no route (not 403) |
| 3 — widening (main vs HEAD) | ✅ LIVE HEAD (kimi_cli 200; bogus_source 200) | ✅ git diff: `SourceType.KIMI` + `KimiParser` + `POLICY_RESOLVERS[KIMI]` present at HEAD, ABSENT on origin/main | main = unrecognized fail-open (unenforced allow); HEAD = recognized + enforced |
| 2 — cross-tenant | ❌ not feasible (single-tenant key; no DB mutation) | ✅ Kong consumer header → `tenant.schema_name`; recompute passes caller `tenant` | Key is bound to its own tenant schema; recompute scoped to caller schema |
| 4 — bulk_update toggle | ❌ not run (would mutate rows) | ✅ diff: MCP path unchanged, new agent path MCP-excluded, no new payload fields | MCP-only toggle + old-shape payload served exactly as before |

**Honest deltas from the board how-to-close wording:** (a) unknown sources at the app layer are NOT 4xx-refused — they fail open to `200 allow`; the real refusals on this surface are the 401/404 credential/route boundaries. (b) The main-vs-HEAD widening for kimi_cli is *unenforced-allow → enforced*, both surfacing `200` for benign bodies, so it is proven at the code/route level (the sanctioned method when no live main stack is up), with the live HEAD 200 as the accepted-half corroboration.
