# C9 — UI view-state enumeration (endpoint_kimi-cli)

**Rule C9** — "Show every view state your UI change can render."
**Repo:** /home/coder/onyx · **Branch:** factory-eval-v32/PRDCT-14240 · **HEAD:** 74bded8213 · **Stamped:** 2026-09-20

This is the mechanical state list derived from the diff. Each renderable value names the backend field it derives from and quotes the exact null/branch predicate applied. The tri-state icon behavior was introduced by commit `10a7dca209cd` ("Apply curated brand-art fallback in the inventory read path") and touches all three changed inventory read paths.

`endpoint_kimi-cli` is IN the runtime-protection supported set (`frontend/app/src/lib/runtime-protection-support.generated.json`, `appDefinitionIds` list, count 27), and IS NOT in `SWG_GUARDED_APP_DEFINITION_IDS` (only `endpoint_microsoft_copilot`, `endpoint_m365_copilot`). It has a curated icon entry (`"kimi-cli": "/applications/kimi.svg"`).

---

## 1. Runtime-protection group-overview banner

Consumer: `frontend/app/src/routes/ai-assets/desktop-agents/group/-components/group-overview.tsx` (the group **overview** page acts on the predicate; the instance page deliberately does not). Predicate source: `frontend/app/src/lib/runtime-protection-support.ts`.

| State | Backend / source field | Predicate applied | Rendered result |
|---|---|---|---|
| Runtime protection **supported** (banner shows normal install / connect / configure / applied-guard affordances) | `appDefinitionId` = `"endpoint_kimi-cli"` vs `RUNTIME_PROTECTION_SUPPORTED_APP_DEFINITION_IDS` (generated from capability matrix); per-tenant `swgProxyKeyDisplay` LD flag | `notSupported = isRuntimeProtectionUnsupported(appDefinitionId) \|\| isRuntimeProtectionUnavailableForTenant(appDefinitionId, swgProxyKeyDisplay)` → **false** for kimi-cli. `isRuntimeProtectionUnsupported` returns `!!appDefinitionId && !RUNTIME_PROTECTION_SUPPORTED_APP_DEFINITION_IDS.has(appDefinitionId) && !isSwgGuardedApp(appDefinitionId)` = false because the id IS in the set. | `<AiGuardConnectBanner notSupported={false} …>` — Connect / Configure / applied-guard states |
| Runtime protection **not supported** (contrast state; NOT reachable for kimi-cli — shown here for completeness of the boolean) | same fields | `isRuntimeProtectionUnsupported(id)` → **true** for an id absent from the supported set and not SWG-guarded | `<AiGuardConnectBanner notSupported={true}>` — "Runtime protection is not currently supported for this agent type" + Contact Onyx |
| `isSwgGuardedApp` leg | `appDefinitionId` vs `SWG_GUARDED_APP_DEFINITION_IDS` | `!!appDefinitionId && SWG_GUARDED_APP_DEFINITION_IDS.has(appDefinitionId)` → **false** for kimi-cli (id not in the SWG set) | SWG/proxy-guarded branch not taken; banner uses scanner-guarded default |

For `endpoint_kimi-cli` the only reachable banner branch is **supported / notSupported=false**; the not-supported and SWG branches are shown to document both sides of each boolean.

---

## 2. `image_url` tri-state (introduced by 10a7dca209cd)

Rendering mechanism: `InitialsAvatar` (`frontend/app/src/components/ui/initials-avatar.tsx`) renders `<AvatarImage src={src}>` when `src` is present, else `<AvatarFallback>` = `getInitials(name)` (the initials plate). So the resolved `image_url` value drives which of the two mounts.

Backend resolution: `backend_python/src/common/models/agent_brand_icons.py`.
- SQL form `catalog_image_url(AppDefinition)`: `func.coalesce(func.nullif(app_definition.image_url, ""), case(BUNDLED_ART_BY_APP_ID, value=app_definition.app_id))` — `NULLIF(image_url,'')` collapses empty string to NULL, then the curated-art `CASE` keyed on `app_id`.
- Python twin `resolve_catalog_image_url(app_id, stored_image_url)`: `if stored_image_url: return stored_image_url` → `if not app_id: return None` → `return BUNDLED_ART_BY_APP_ID.get(app_id)`.
- `BUNDLED_ART_BY_APP_ID["endpoint_kimi-cli"] = "/applications/kimi.svg"` (from `CURATED_AGENT_ICONS["kimi-cli"]`).

Applied on the **three read paths changed in 10a7dca209cd**:
- Inventory list group row + connected-agents graph node select → `catalog_image_url(AppDefinition).label("image_url")` / `.label("agent_image_url")` in `backend_python/src/common/repository/inventory_applications/service.py`.
- Desktop-agent detail header → `image_url=resolve_catalog_image_url(app_definition_id, app_def.image_url) if app_def else None` in the same file (consumed by `frontend/app/src/routes/ai-assets/desktop-agents/$appId.tsx`, `appLogo={app.image_url ?? undefined}`).
- `asset_inventory` projection twin → `catalog_image_url(AppDefinition).label("image_url")` in `backend_python/src/common/repository/asset_inventory/projection.py` (value-identical, dual-write parity).

| State | Backend field | Predicate applied | Rendered result (all 3 paths) |
|---|---|---|---|
| Stored value **present** | `app_definitions.image_url` (non-empty) | `COALESCE(NULLIF(image_url,''), …)` returns stored value; Python: `if stored_image_url: return stored_image_url` — a stored value always wins | `<AvatarImage src={stored url}>` — the stored/enrichment/human-chosen image |
| Stored **NULL** + **curated hit** (kimi-cli) | `image_url` IS NULL (or `''`) **and** `app_id` = `"endpoint_kimi-cli"` present in `BUNDLED_ART_BY_APP_ID` | `NULLIF` → NULL, `CASE app_id` → `/applications/kimi.svg`; Python: falls through to `BUNDLED_ART_BY_APP_ID.get("endpoint_kimi-cli")` | `<AvatarImage src="/applications/kimi.svg">` — bundled brand icon |
| Stored **NULL** + **no curated hit** | `image_url` IS NULL and `app_id` not in `BUNDLED_ART_BY_APP_ID` | `CASE` misses → NULL; Python: `BUNDLED_ART_BY_APP_ID.get(app_id)` → None | `<AvatarFallback>` — initials plate |

`/applications/kimi.svg` confirmed present at `frontend/app/public/applications/kimi.svg` (664 bytes).

---

## 3. Sanctioned / unsanctioned status badge

Consumer: `frontend/app/src/components/badges/application-inventory-status-badge.tsx`. Backend field: `AssetStatus` (`status`). Maps: `AssetStatusColorMap` / `AssetStatusLabelMap` in `frontend/app/src/types/applications.ts`.

| State | Backend field | Predicate applied | Rendered result |
|---|---|---|---|
| **Sanctioned** | `status` = `"AssetStatusSanctioned"` | `AssetStatusLabelMap[status]` = "Sanctioned"; `AssetStatusColorMap[status]` = `bg-utility-success-500` | Green-dot "Sanctioned" `StatusDotBadge` |
| **Unsanctioned** | `status` = `"AssetStatusUnsanctioned"` | `AssetStatusLabelMap[status]` = "Unsanctioned"; color `bg-utility-warning-500` | Amber-dot "Unsanctioned" `StatusDotBadge` |
| **Discovered** (default) | `status` = `"AssetStatusDiscovered"` | label "Discovered"; color `bg-utility-gray-500` | Gray-dot "Discovered" badge |
| Editable vs read-only | `isEditable` prop | `if (!isEditable \|\| !onEdit)` → tooltip "Status set via vendor", non-interactive; else dropdown offers `sanctioned`/`unsanctioned` | Non-editable tooltip vs dropdown toggle |

---

## 4. Empty / zero-rows inventory state

| State | Backend field | Predicate | Rendered result |
|---|---|---|---|
| No matching rows | `_search` response `data` array | `response.json()["data"]` empty (no DESKTOP_AGENT rows) | Inventory table empty state (no group rows rendered) |

---

## Closing note

Captures of these states in the running UI are blocked on the shared coder-agent@onyx.security JumpCloud SSO lockout (bucket b); the icon mapping is proven in code+test (`test_inventory_applications`: `endpoint_kimi-cli` -> `/applications/kimi.svg`).

Test evidence, at HEAD `74bded8213`, in `backend_python/tests/crud_service/api/test_inventory_applications.py`:
- Class `TestEndpointAgentBrandIconFallback` (line 6642), parametrized case tuple `("endpoint_kimi-cli", None, "/applications/kimi.svg")` (line 6663) — asserts NULL stored image + curated slug resolves to the bundled path; sibling cases prove non-curated+NULL → None and stored → stored (art never overrides).
- `test_inventory_list_applies_brand_icon_fallback` (line 6710) — drives `POST /api/crud/v1/inventory-applications/_search`, asserts `image_url` equals the expected resolved value per app_id (real DB-backed read path).
- `test_desktop_agent_group_detail_applies_brand_icon_fallback` (line 6735) — drives `GET /api/crud/v1/inventory-applications/desktop-agent-group`, asserts detail-header `image_url` equals expected.
