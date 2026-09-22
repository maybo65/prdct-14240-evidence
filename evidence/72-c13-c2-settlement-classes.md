# C13 / C2 — settlement classes beyond first-write, driven through BOTH ingest lanes

**Board rules addressed**
- **C13** — "Same payload, both lanes, identical outcome" — the prior two-lane diff (`70-c13-two-lane-diff.md`)
  proved only the **first-write** outcome; the removal rescan and other settlement classes had run in one
  lane only. This file drives the remaining settlement classes through **both** lanes.
- **C2** — "no post-uninstall rescan rows are shown." This file shows exactly what the DB does on a
  post-uninstall (agent-absent) rescan, per lane.

## The two lanes (unchanged from file 70)
- **Lane A — scanner Temporal lane, DEPLOYED worker.** `DesktopAgentCreatorScannerChunkWorkflow` on the
  real deployed scanner worker (task_queue `scanner`), tenant **`kimi_diff_scanner`** (id 2). Driver:
  `/tmp/c13_laneA_scanner.py` (re-delivery) / `/tmp/c13_laneA_nokimi.py` (omit-kimi).
- **Lane B — single-writer, in-process.** `ObjectIngestionProcessor.process(ref)` (scanner branch,
  `service.py:240` else-branch) reading the real localstack S3 object, tenant **`kimi_diff_newlane`**
  (uuid `ddd1baa8-…`). Driver: `/tmp/c13_laneB_newlane.py` / `/tmp/c13_laneB_nokimi.py`.

Both lanes converge on the **same shared write activities** — `process_single_agent`
(`…/desktop_agent_creator/activities.py:2097`) and `create_builtin_tools` (`activities.py:2790`, upsert
at `:2834`) — so a per-lane row difference would be pure lane-mechanism. The builtin-tool set is resolved
by agent name inside `create_builtin_tools` (`KIMI_CLI_BUILTIN_TOOLS`, `kimi_cli.py:75`, 9 tools), not
from the scan bytes.

## The settlement model (what the code actually does — mapped from source)
Desktop-agent ingestion is **upsert-only for the agent asset and its builtin tools**, in BOTH lanes.
There is **no reaper** for a real device agent asset — stated verbatim at
`endpoint_asset_ingestion/ingestion/per_user_asset_writer.py:38` ("Nothing here can touch a real device
asset, and no reaper exists"). `create_builtin_tools` calls `upsert_tools_bulk`
(`common/database/repository/tools.py:206`) **without** the paired `deactivate_tools_not_in_list`
(`tools.py:665`, the helper exists but is deliberately not called on this lane) — so a builtin tool that
drops out of a rescan keeps `is_active=true`. The only removal-shaped reconciles that exist are:
(a) MCP **connections** (`asset_connections`, never the asset — `mcp_writer.py:37`), (b) the per-device
**guard verdict** age-out (`last_guard_status` true→false, `desktop_agent_application.py:48/114`,
invoked `activities.py:3705`), and (c) effective-settings roster winners (`activities.py:1253`) — none
delete the agent or its tools.

So the honest, correct settlement outcomes to drive and observe are:
1. **Idempotent re-delivery** (same payload again) → converges, no duplicate rows.
2. **Rescan omitting the agent** ("uninstall then rescan") → asset + 9 tools **persist unchanged**
   (upsert-only, no reaper) — this *is* the "post-uninstall rescan rows" C2 asks about.
3. **Guard-verdict age-out** = the real "uninstall" reconcile that exists — code-level, **time-gated on a
   30-day `GUARDABILITY_WINDOW`**, so not runtime-forceable without clock manipulation (disclosed below).

The inputs are byte-identical across lanes: the full scan `/tmp/kimi_payload.jsonl` (5 records: kimi
`scan`, claude `scan`, codex `scan`, `scan_summary`, `scan_logs`) and the omit-kimi variant
`/tmp/kimi_payload_nokimi.jsonl` (the kimi `scan` record `scanner-agent-kimi-cli-…` dropped). Dry-run of
the shared extractor confirmed: full → agents `[kimi-cli, claude-code, codex]`; nokimi →
`[claude-code, codex]` (kimi absent, no error).

## Baseline T0 (both tenants, after the file-70 first-write)
```
Lane A  kimi_diff_scanner : asset id=1 Kimi CLI is_deleted=false ; 9 tools active ; daa guard=true last_scanned=10:42:33 ; desktop-agent assets=3
Lane B  kimi_diff_newlane : asset id=3 Kimi CLI is_deleted=false ; 9 tools active ; daa guard=true last_scanned=10:37:34 ; desktop-agent assets=3
```

## Settlement class 1 — idempotent re-delivery (identical full payload #2)
Same scan driven a second time into the same tenant, each lane.

**Lane A (Temporal scanner worker), T1:**
```
kimi asset count=1 (NO dup) ; is_deleted=false ; asset.updated_at UNCHANGED (10:35:00.99)
9 tools ; active=9 ; tools.updated_at UNCHANGED (10:35:01.68)   [read-before-write: no write when unchanged]
daa: guard=true ; last_scanned ADVANCED 10:42→13:06:59   ; desktop-agent assets=3 (no dup)
```
**Lane B (single-writer), T1:**
```
kimi asset count=1 (NO dup) ; is_deleted=false ; asset.updated_at UNCHANGED (10:35:01.62)
9 tools ; active=9 ; tools.updated_at UNCHANGED (10:35:01.86)
daa: guard=true ; last_scanned ADVANCED 10:37→13:09:06   ; desktop-agent assets=3 (no dup)
```
**Identical outcome:** re-delivery converges in both lanes — no duplicate asset / tool / daa rows, the
asset and tool rows are not rewritten (read-before-write short-circuit), only `last_scanned_at` advances.
Lane B log confirms the conservative reconcile: *"reconcile SKIPPED for agent with empty keep-set … empty
scan treated as no-signal, not uninstall-everything"* and *"Guard aggregate skipped: scan incomplete
(partial/cancelled run must not downgrade)"*.

## Settlement class 2 — rescan omitting kimi ("uninstall then rescan")
The omit-kimi scan (kimi record dropped; claude+codex still present) driven into the same tenant, each
lane. The write set is `[claude-code, codex]`; kimi is not processed.

**Lane B (single-writer), T2:**
```
kimi asset STILL PRESENT: id=3 Kimi CLI is_deleted=false ; count=1
9 tools STILL active ; tools.updated_at UNCHANGED (10:35:01.86)
daa STILL present: guard=true ; last_scanned UNCHANGED (13:09:06)   [kimi row untouched — not processed this scan]
desktop-agent assets=3
```
**Lane A (Temporal scanner worker), T2:** (workflow extracted 2 agents `[claude-code, codex]`, kimi omitted — SAME set as Lane B)
```
kimi asset STILL PRESENT: id=1 Kimi CLI is_deleted=false ; count=1
9 tools STILL active ; tools.updated_at UNCHANGED (10:35:01.68)
daa STILL present: guard=true ; last_scanned UNCHANGED (13:06:59)   [kimi row untouched — not processed this scan]
desktop-agent assets=3
```
**Identical outcome:** an agent-absent ("uninstalled") rescan **does not remove or deactivate** the kimi
asset or its builtin tools in EITHER lane — `assets.is_deleted` stays false, all 9 `tools.is_active` stay
true, the `desktop_agent_applications` row remains. This is the documented no-reaper behavior
(`per_user_asset_writer.py:38`), identical across lanes. **This is the answer to C2's "post-uninstall
rescan rows": no removal rows are produced — by design; the rows persist and simply stop being
re-stamped.** The only signal that the agent stopped being seen is that its `last_scanned_at` /
`last_guard_status_at` stop advancing (see class 3).

## Settlement class 3 — guard-verdict age-out (the real "uninstall" reconcile) — code-level, time-gated
The one settlement that *does* flip on "no longer present" is the **guard verdict**, not the asset:
`age_out_stale_guard_statuses_for_device` / `…_for_container`
(`common/repository/desktop_agent_application.py:48-112 / 114-170`) lower
`desktop_agent_applications.last_guard_status` **true→false** and stamp `last_guard_status_at = cutoff`
for rows whose last confirmation predates `GUARDABILITY_WINDOW`; invoked once per host/container scan
summary at `activities.py:3705-3711`, and it is a **shared** activity reached by **both** lanes
(`activities.py:3662-3667`). Because the cutoff is `now − GUARDABILITY_WINDOW` (a 30-day window) and both
tenants' `last_guard_status_at` are minutes old, the sweep is a **no-op here** and cannot be forced
without clock manipulation — which would be a fabricated environment, so it is **not** done. This
settlement is therefore closed at the **code level** (shared, both lanes) with the citations above and an
honest "not runtime-forced (time-gated)" note, rather than dressed up as a live run.
Additionally, in these unit-context runs the `scan_summary` is reported `scan incomplete`, so the guard
**aggregate** (which would authoritatively lower to false on a complete scan) is correctly skipped in
both lanes — a partial/cancelled run must never downgrade guard status (log line quoted in class 1).

## Settlement classes NOT run, with reason (honest ledger)
- **Guard age-out live flip** — time-gated on a 30-day window (above); code-level only, not forced.
- **MCP-connection uninstall reconcile** — this scan carries no MCP-detection payload, so the connection
  reconcile is correctly a no-op ("empty keep-set … not uninstall-everything"); a dedicated MCP-uninstall
  scan is an MCP-lane settlement, out of scope for the kimi builtin-tool/asset rows C13/C2 track.
- **Feature-flag off path** — `desktop-agent-generation-succession` reads default=false in this unit
  context (flag service not initialized); the succession/generation branch is flag-gated platform
  behavior, not a kimi-specific write, so it is not exercised as a kimi settlement here.

## Reproducibility
Drivers (real infra, no mocks): `/tmp/c13_laneA_scanner.py`, `/tmp/c13_laneA_nokimi.py`,
`/tmp/c13_laneB_newlane.py`, `/tmp/c13_laneB_nokimi.py`. Snapshots via `/tmp/snap.sh <schema>` against the
real Postgres (:5433). Temporal frontend :7244 (deployed scanner worker), localstack S3 :4599.
