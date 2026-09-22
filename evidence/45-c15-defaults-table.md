# C15 — Changed-Defaults Inventory

**Verifier rule:** C15 — "A changed default is an owner decision."
**PR:** onyxsecurity/onyx#13392 · **Ticket:** PRDCT-14240 (Kimi Code/CLI)
**Branch:** `factory-eval-v32/PRDCT-14240` · **HEAD:** `74bded8213` · **Base:** `origin/main` @ `3e71ed17230c` · **Stamped:** 2026-09-20

Scope of this artifact: the **producible inventory** the board's how-to-close asked for — one row per default this diff changes or adds, with before → after, blast radius (who gets the new value), and whether the change is itself an owner decision. Each value below is verified against the real diff at HEAD; the exact code that sets it is quoted underneath the table. This artifact does **not** approve any default (see closing note).

---

## Defaults table

| # | Default | Before | After | Who gets the new value (blast radius) | Owner-decision? |
|---|---------|--------|-------|----------------------------------------|-----------------|
| 1 | `unsanctioned-agent-deployed` catalog entry lifecycle | Retired (`retired=True`; tombstoned) | Active (matcher restored, `retired` removed) | **Fleet-wide.** Every tenant. The issue re-opens on the next detection run for **every non-sanctioned desktop/AI agent in every tenant** — not just Kimi (see #3). | **Y — this is the crux; deferred to owner.** |
| 2 | Tenant migration: `issue_definitions.is_enabled` for this identifier | `false` (retirement tombstone) | `true` | **Fleet-wide, every tenant schema.** One-time `UPDATE` forces the flag true in each tenant that still carries the tombstone, so tenants provisioned before retirement behave like fresh ones and detect the revived issue. | **Y** (executes the revive fleet-wide) |
| 3 | Matcher `_match_unsanctioned_agent_deployed` firing scope | (none — issue retired, no matcher) | Fires for **any** high-risk AI asset whose status ≠ `SANCTIONED` (incl. NULL→DISCOVERED and UNSANCTIONED) | **Any DESKTOP_AGENT, or any HOMEGROWN asset with sub_type AI_AGENT, in any tenant** — Kimi is only one instance of this set. | **Y** (defines the blast radius of #1) |
| 4 | Compliance tags on `unsanctioned-agent-deployed` | `tags=[]` (dropped at retirement) | OWASP `ASI03`; NIST AI RMF `GOVERN-1.2`; NIST AI RMF `GOVERN-2.1`; EU AI Act `Article-9` | Compliance/coverage pages fleet-wide: this issue now maps into those four framework cells for every tenant. | Y (couples to #1) |
| 5 | Inventory icon resolution (list read — `projection.py`) | `AppDefinition.image_url` | `catalog_image_url(AppDefinition)` — curated brand-art fallback when stored `image_url` is NULL; a stored value still wins | All inventory **list** reads, all tenants: endpoint/desktop agents with NULL `image_url` render a brand icon instead of an initials plate. Value-identical parity twin where a stored URL exists. | N (display fallback; value-identical when set) |
| 6 | Inventory icon resolution (graph / model-page read — `service.py`, `agent_image_url`) | `AppDefinition.image_url` | `catalog_image_url(AppDefinition)` fallback | All graph/model-page reads, all tenants: connected agents with NULL image render brand art. | N (display fallback) |
| 7 | Inventory icon resolution (detail read — `service.py`) | `app_def.image_url` (or None) | `resolve_catalog_image_url(app_definition_id, app_def.image_url)` fallback | Asset **detail** header, all tenants: endpoint agents seeded with no image render brand art instead of initials. | N (display fallback) |
| 8 | `AGENT_ISSUE_RECALC_BATCH_SIZE` (`access_control.py`) | (new constant) | `1000` | Internal bound only: caps ids per Temporal workflow payload when re-computing agent issues after a sanction-state change. No customer-visible value change. | N (internal tuning) |
| 9 | `kimi-hooks-install --block-prompts` default | (new flag) | `"true"` (hard block mode) | New CLI installs of Kimi hooks that don't pass the flag: default to hard-block instead of replacement. | N (per-install CLI default, user-overridable) |
| 10 | `kimi-hooks-install --endpoint` default | (new flag) | `https://ai-guard.onyx.security` (when flag + `ONYX_AI_GUARD_ENDPOINT` env both empty) | New Kimi-hooks installs with no explicit endpoint. | N (per-install CLI default, user-overridable) |
| 11 | `kimi-hooks-install` derived hooks-runtime-protection | (new) | `true` — derived (not left to flag default) for the hooks topology; explicit `--hooks-runtime-protection[=false]` wins | New Kimi-hooks installs that don't set the flag: content protection on. | N (per-install CLI default, user-overridable) |
| 12 | Go risk registry entry for `unsanctioned-agent-deployed` (`backend/internal/issues/issues.go`) | **Absent** (no Go-side risk row while the issue was retired) | Present: `Identifier "unsanctioned-agent-deployed"`, `Factor = AccessControls`, `Score = 3.0` | Go issue-risk consumers, fleet-wide: the revived finding now carries its Go-side risk factor/score, mirroring the Python catalog entry so both lanes score it identically. | **Y** (couples to #1 — part of the same revive; the Go score is what the finding surfaces at once #1 fires) |

Rows 1–4 and 12 are one coupled owner decision (revive + fleet-wide enablement + firing scope + compliance mapping + the mirrored Go risk score). Rows 5–7 are one display-fallback change across three reads. Rows 8–11 are internal/CLI defaults.

---

## Verified quotes (code that sets each default, at HEAD `74bded8213`)

### Rows 1 & 4 — `backend_python/src/common/issue_detection/catalog_entries.py`
Retirement removed (`retired=True` deleted, tombstone comment removed) and tags restored:
```python
ISSUE_UNSANCTIONED_AGENT_DEPLOYED = "unsanctioned-agent-deployed"  # Medium posture issue: a discovered/unsanctioned AI agent platform
...
        tags=[
            TagSpec(type=IssueFrameworkTagType.OWASP, key="ASI03"),
            TagSpec(type=IssueFrameworkTagType.NIST_AI_RMF, key="GOVERN-1.2"),
            TagSpec(type=IssueFrameworkTagType.NIST_AI_RMF, key="GOVERN-2.1"),
            TagSpec(type=IssueFrameworkTagType.EU_AI_ACT, key="Article-9"),
        ],
        ...
        applies_to=AppliesTo(
            types=frozenset({AssetType.HOMEGROWN, AssetType.DESKTOP_AGENT}),
            extra_predicate=lambda asset, _ctx: asset.type == AssetType.DESKTOP_AGENT or asset.sub_type == AssetSubType.AI_AGENT,
        ),
        # (retired=True removed)
```
Exact tag identifiers: `ASI03`, `GOVERN-1.2`, `GOVERN-2.1`, `Article-9`.

### Row 2 — tenant migration `a1b7c3e5d9f2` → `sql/reenable_revived_unsanctioned_agent_issue.sql`
`upgrade()` runs `run_sql_file("reenable_revived_unsanctioned_agent_issue.sql")`, whose body is:
```sql
UPDATE issue_definitions
SET is_enabled = true
WHERE identifier = 'unsanctioned-agent-deployed'
  AND is_enabled = false;
```
The SQL header itself documents the blast radius: "at most one row per tenant schema … so every tenant behaves like a fresh one."

### Row 3 — matcher `backend_python/src/python_temporal_worker/issue_detection/catalog.py`
```python
async def _match_unsanctioned_agent_deployed(ctx: MatcherContext) -> bool:
    if not _is_high_risk_ai_asset(ctx.asset):
        return False
    canonical_status = ctx.asset.status or AssetStatus.DISCOVERED
    return canonical_status != AssetStatus.SANCTIONED
```
with predicates:
```python
def _is_ai_agent(asset: Asset) -> bool:
    return asset.type == AssetType.HOMEGROWN and asset.sub_type == AssetSubType.AI_AGENT

def _is_high_risk_ai_asset(asset: Asset) -> bool:
    return _is_ai_agent(asset) or asset.type == AssetType.DESKTOP_AGENT
```
**Confirms the crux: the matcher has no Kimi-specific gate.** It fires for *any* `DESKTOP_AGENT` or any `HOMEGROWN` asset with sub_type `AI_AGENT` whose stored status is not `SANCTIONED` — a NULL status coalesces to `DISCOVERED` and matches, so the issue opens at discovery for the whole non-sanctioned agent fleet across every tenant.

### Rows 5–7 — icon resolution
`backend_python/src/common/repository/asset_inventory/projection.py` (list read):
```python
from common.models.agent_brand_icons import catalog_image_url
...
"image_url": catalog_image_url(AppDefinition).label("image_url"),  # was AppDefinition.image_url
```
`backend_python/src/common/repository/inventory_applications/service.py` (graph + detail reads):
```python
from common.models.agent_brand_icons import catalog_image_url, resolve_catalog_image_url
...
catalog_image_url(AppDefinition).label("image_url"),        # list, was AppDefinition.image_url
catalog_image_url(AppDefinition).label("agent_image_url"),  # graph/model-page, was AppDefinition.image_url
image_url=resolve_catalog_image_url(app_definition_id, app_def.image_url) if app_def else None,  # detail, was app_def.image_url
```
A stored `image_url` still wins; the fallback only supplies curated brand art when the stored column is NULL.

### Row 8 — `backend_python/src/crud_service/api/routes/access_control.py`
```python
AGENT_ISSUE_RECALC_BATCH_SIZE = 1000
```
Used as `batch_size=AGENT_ISSUE_RECALC_BATCH_SIZE` to page asset ids into bounded Temporal recalc workflows after a sanction-state change.

### Rows 9–11 — `scanner/cmd/kimi_hooks_install.go`
Flag registrations:
```go
kimiHooksInstallCmd.Flags().StringVar(&kimiHooksInstallEndpoint, "endpoint", "", "Custom endpoint URL (default: https://ai-guard.onyx.security). Env - ONYX_AI_GUARD_ENDPOINT")
kimiHooksInstallCmd.Flags().StringVar(&kimiHooksInstallBlockPrompts, "block-prompts", "true", "Enable hard block mode instead of replacement (true/false, default: true)")
```
Endpoint default resolution (flag → env → literal):
```go
kimiHooksInstallEndpoint = common.GetStringFromFlagOrEnv(kimiHooksInstallEndpoint, "ONYX_AI_GUARD_ENDPOINT")
if kimiHooksInstallEndpoint == "" {
    kimiHooksInstallEndpoint = "https://ai-guard.onyx.security"
}
```
Derived runtime protection (hooks topology → `true`), overridable:
```go
if !cmd.Flags().Changed(runtimeProtectionFlagName) {
    runtimeProtection = hooksRuntimeProtectionDefaultFor(integrationTypeHooks)
}
```
with (in `scanner/cmd/opencode_install.go`):
```go
func hooksRuntimeProtectionDefaultFor(integrationType string) bool {
    switch integrationType {
    case integrationTypeBaseURL:
        return false
    default:
        return true   // integrationTypeHooks falls here → runtime protection ON
    }
}
```

### Row 12 — `backend/internal/issues/issues.go` (verified at current HEAD `0f0ffca264`)
The revive adds a Go-side risk registry entry so the finding scores identically on the Go lane. Before this diff the identifier had **no** row in `issues.go` (it was retired); after, at `issues.go:130-137`:
```go
{
    // A discovered/unsanctioned AI agent platform. Factor + score mirror the
    // Python catalog entry (unsanctioned-agent-deployed, AccessControls, 3.0).
    Identifier: strPtr("unsanctioned-agent-deployed"),
    risk: schema.Risk{
        Factor: schema.RiskFactorAccessControls.String(),
        Score:  common.Float32(3),
    },
},
```
- **Before → after:** absent (retired) → present with `Factor = AccessControls`, `Score = 3.0`.
- **Who gets it:** every consumer of the Go issue-risk registry, fleet-wide — the same set that gets rows 1–4. The Go score mirrors the Python catalog's risk metadata so the revived issue is scored the same on both lanes.
- **Owner-decision?** Yes, as part of the row-1 revive: this row does not *independently* enable anything (row 2's migration does that), but it is the Go-side risk value the finding surfaces once the revive fires, so it is inventoried here with the coupled decision.

`git diff origin/main...HEAD -- backend/internal/issues/issues.go` shows this block added (`+`), with no matching entry on `origin/main`.

---

## Closing note

The DECISION on the fleet-wide revive (approve vs gate/scope vs split) is deferred to the owner (May) — **bucket a**; this table is the producible inventory the board asked for.
