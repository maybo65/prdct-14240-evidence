# C13 — old-lane vs new-lane row diff (REAL two-lane ingest of the SAME kimi scan into two fresh tenants)

**Captured:** 2026-09-22T10:49Z · branch HEAD `fce925c99bacc3ef92a14b87bcb5835ef36ee43b`
**Raw dumps (this commit):** [`70a-c13-laneA-scanner-rows.txt`](70a-c13-laneA-scanner-rows.txt) · [`70b-c13-laneB-newlane-rows.txt`](70b-c13-laneB-newlane-rows.txt) · lane-B full run log [`70c-c13-laneB-run.log`](70c-c13-laneB-run.log)

This upgrades C13 from the earlier **outcome-label inventory** (code citation) to a **real, executed two-lane row diff**: the *same* real kimi scan artifact ingested **twice**, once down **each** ingest lane, **each into its own fresh, empty, freshly-migrated tenant schema**, then the resulting tenant-DB rows placed side by side and diffed.

## What was driven (no mocks, both lanes real)

Both lanes were fed the **byte-identical `DesktopAgentData`** extracted from the one real kimi scan artifact `/tmp/kimi_payload.jsonl` (the 1.2 MB `method=scan` capture: kimi-cli + claude-code + codex, `type=desktop`, `tools=null`, kimi's one `e2e-filesystem` MCP server, `systemSerialNumber=ec2f0fe614763f6d5bf528f0dc29322c`). Same `route_lines` → `extract_inventory` on both sides, so the *only* variable is the lane mechanism — which is exactly what C13 diffs.

| | **Lane A — old / scanner lane** | **Lane B — new / single-writer lane** |
|---|---|---|
| Mechanism | `DesktopAgentCreatorScannerChunkWorkflow` on the **DEPLOYED** Temporal scanner worker (`task_queue=scanner`, pod `temporal-scanner-worker-85b776b68d-zppnl`) — the exact workflow the Go `ScannerTenantWorkflow` dispatches in prod | `endpoint_asset_ingestion` in-process single writer: `ObjectIngestionProcessor.process(ref)` built exactly as `service.py:240` scanner else-branch builds it |
| Tenant resolution | Go/workflow input fields (`tenant_id=2`, `tenant_schema="kimi_diff_scanner"`) | S3 key path `tenant=<uuid>/…` → `get_tenant_by_uuid_cached` → `kimi_diff_newlane` |
| Real infra | real Temporal frontend (pf 7244→7233), real deployed worker, real Postgres | real localstack S3 (staged kimi object), real Postgres |
| Fresh tenant | `kimi_diff_scanner` (id=2, uuid `ee4c04e0-…`), migrated to head `87d86fc4d57f`, 218 tables, **empty at start** | `kimi_diff_newlane` (id=3, uuid `ddd1baa8-…`), migrated to head `87d86fc4d57f`, 218 tables, **empty at start** |

**Shared-writer basis (code, confirmed by Explore):** both lanes converge on the literal same `DesktopAgentCreatorActivities.process_single_agent` (`activities.py:2097`) → `create_builtin_tools` (`activities.py:2790`) → `_resolve_builtin_tool_definitions` (`activities.py:1162`) → `get_builtin_tools_for_agent("kimi-cli")` → `KIMI_CLI_BUILTIN_TOOLS` (`kimi_cli.py:75`, exactly 9). Scanner lane reaches them at `workflow.py:558/595`; single-writer lane at `per_user_asset_writer.py:337` + `writers.py:322`. The lanes differ **only** in tenant-resolution + outer orchestration. This diff is the *executed* confirmation that the shared-writer basis produces identical rows.

## Side-by-side row diff

| Row / field | Lane A (`kimi_diff_scanner`) | Lane B (`kimi_diff_newlane`) | Diff |
|---|---|---|---|
| **asset** `name` | `Kimi CLI (coder)` | `Kimi CLI (coder)` | **identical** |
| asset `type` | `AssetTypeDesktopAgent` | `AssetTypeDesktopAgent` | **identical** |
| asset `sub_type` | `AssetSubTypeAIAgent` | `AssetSubTypeAIAgent` | **identical** |
| asset `unique_identifier` | `endpoint_kimi-cli::ec2f0fe614763f6d5bf528f0dc29322c` | `endpoint_kimi-cli::ec2f0fe614763f6d5bf528f0dc29322c` | **identical** |
| asset `id` (surrogate) | `1` | `3` | seq-assigned (expected) |
| **builtin tools** (external_id) | 9: `builtin:kimi-cli:{FetchURL,Glob,Grep,ReadFile,ReadMediaFile,SearchWeb,Shell,StrReplaceFile,WriteFile}` | same 9, same names | **identical** |
| tools `is_active` | all `t` | all `t` | **identical** |
| kimi tools / codex-named on kimi asset | **9 / 0** | **9 / 0** | **identical** |
| **desktop_agent_applications** `last_guard_status` | `t` | `t` | **identical** |
| dap `id` / `asset_id` (surrogate) | `1` / `1` | `3` / `3` | seq-assigned (expected) |
| **channel** `channel_id` / `display_name` / `is_active` | `tui` / `Terminal UI` / `t` | `tui` / `Terminal UI` / `t` | **identical** |
| channel `id` (surrogate) | `1` | `2` | seq-assigned (expected) |
| **public.app_definitions** (shared) | `540 · endpoint_kimi-cli · Kimi CLI · OriginChina · AssetSubTypeAIAgent` | same shared row `540` | **identical (single shared row)** |

## Divergence ledger

- **Semantic divergence: ZERO.** Every business-meaningful field the new (single-writer) lane writes is byte-identical to what the old (scanner) lane writes: the asset identity, all 9 kimi-own builtin tools, the `0` codex-named leak, the guard-status flag, the TUI channel, and the shared public app-definition.
- **New lane LOSES nothing.** Every row the old lane produces (asset, 9 tools, dap, channel, app-def) is present and identical on the new lane. C13's invariant — *the new lane may ADD but must never LOSE* — holds: it neither loses nor adds a row versus the old lane on this artifact.
- **The only differences are sequence-assigned surrogate keys** (`asset.id` 1 vs 3, `dap.id` 1 vs 3, `channel.id` 1 vs 2). Those are per-tenant identity-column positions, not lane behavior — each fresh schema's sequences simply start from a different point because the tenants were migrated/seeded independently. They carry no semantic weight and are never referenced across tenants.
- **C2 corollary (fresh clean-room):** on *both* fresh tenants the kimi asset carries **exactly 9 kimi-own tools and 0 codex-named rows**. This independently reconfirms — in a clean room with no pre-branch residue — that the 5 codex-named rows that coexist on the shared `onyx_security` tenant are stale pre-branch (2026-09-13) upsert residue, **not** a branch defect. Neither lane writes a codex-named tool onto the kimi asset.

## Honest caveats

- **Lane A parent workflow was `status=Running` at capture.** The write activities (`process_single_agent` → `create_builtin_tools`) **committed** all rows above (they are present and queried), while the parent `DesktopAgentCreatorScannerChunkWorkflow` remained in `Running` (downstream orchestration on the deployed worker) when the bounded 150 s driver wait elapsed. The *committed write rows* are what C13 diffs; the parent's terminal state is orthogonal to them. The payload was ~529 KB — just over Temporal's 512 KB **soft-warn** threshold (a `PayloadSizeWarning`, not an error); the workflow accepted it and ran.
- **Lane B has no pod on this frozen kind-slate cluster** (only `endpoint-swg-ingestion` is deployed). It therefore ran here via `uv run --no-sync` against the **same branch code** + the **same real localstack S3** + the **same real Postgres** the deployed lane would use; the full SQS→handler→process delivery path on this lane was separately exercised in file 69 (DLQ run). `temporal_client=None` on lane B only skips the optional fire-and-forget custom-data EMBEDDING workflow (`writers.py:488`) — the core asset + 9 tools + channel + dap rows are all written on this path (and match lane A exactly, above).
- These are **fresh, disposable diff tenants** (`kimi_diff_scanner` / `kimi_diff_newlane`) created solely for this control; they are not product tenants.
