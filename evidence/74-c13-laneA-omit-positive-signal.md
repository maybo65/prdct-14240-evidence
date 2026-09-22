# C13 lane A — agent-absent (kimi-omitted) rescan on the DEPLOYED scanner worker: positive execution signal + kimi untouched

**Board rule C13** asked the lane-A omit-kimi rescan to carry a **positive execution signal**
alongside the kimi rows — verbatim: *"the DesktopAgentCreatorScannerChunkWorkflow reaching a
terminal Completed state for that run, **or** the claude-code/codex
`desktop_agent_applications.last_scanned_at` (and/or `assets.updated_at`) advancing across that run
while the kimi asset, its 9 builtin tools (`is_active=true`) and its daa row stay untouched."*
The prior capture left the workflow Running at the 150s await cutoff with no row-diff shown — a null
observation. This file supplies the **OR-branch decisively**: the claude-code/codex daa rows
**advance across the run**, and the kimi asset + its 9 active tools + its daa row are **provably
untouched**, on the real fresh tenant `kimi_diff_scanner` (id=2). Verbatim psql + driver log in
`74-c13-laneA-omit-positive-signal-raw.txt` (same commit).

## What was driven (no mocks)
- **Real deployed worker.** `DesktopAgentCreatorScannerChunkWorkflow` started on task_queue
  `scanner` (deployed pod `temporal-scanner-worker-85b776b68d-zppnl`, `1/1 Running`) via the real
  Temporal frontend (pf 7244→7233) — the same workflow the Go `ScannerTenantWorkflow` dispatches in
  production (`post_inventory.go:126`). Workflow id
  `c13-laneA-nokimi-19bc9e91-ccb3-4a70-acb5-3da6bbe8a1b9`.
- **Genuinely kimi-absent input.** The lane-A payload was rebuilt from the real scan
  `/tmp/kimi_payload.jsonl` (5 records) by **dropping the entire `kimi-cli` per-agent scan record**
  (`data.codingAgent.name=='kimi-cli'`, id `scanner-agent-kimi-cli-a236394c-…`) **and** scrubbing
  `kimi-cli` from the scan-summary's `users[].agents[]` and `guardAggregate[]`. The real extractor
  (`route_lines` + `extract_inventory`) then yields **exactly two** desktop agents — `claude-code`,
  `codex` — **no kimi** (driver log: `extracted 2 desktop agents … app_name=claude-code … app_name=codex`).
  This is a true agent-absent rescan: kimi is not in the scan the worker processed.
- **Real Postgres.** All rows read on the same instance the worker writes (`:5433`), tenant schema
  `kimi_diff_scanner`.

## The positive signal — single clean run, before vs after (verbatim)
BEFORE-2 baseline captured 2026-09-22T13:53:45Z; run started 13:54:05Z (driver `captured:` line);
AFTER-2 captured 2026-09-22T13:56:49Z.

| row | BEFORE-2 | AFTER-2 | verdict |
|---|---|---|---|
| **claude-code** (id=2) `daa.last_scanned_at` | `13:51:10.294691+00` | **`13:54:05.596848+00`** | ▲ **ADVANCED** (positive signal) |
| **codex** (id=3) `daa.last_scanned_at` | `13:51:10.276876+00` | **`13:54:05.586771+00`** | ▲ **ADVANCED** (positive signal) |
| **kimi** asset (id=1) `is_deleted` | `false` | `false` | = untouched |
| **kimi** asset (id=1) `updated_at` | `10:35:00.996736+00` | `10:35:00.996736+00` | = untouched |
| **kimi** tools `count \| active` | `9 \| 9` | `9 \| 9` | = untouched (all 9 `is_active=true`) |
| **kimi** tools `max(updated_at)` | `10:35:01.68436+00` | `10:35:01.68436+00` | = untouched |
| **kimi** daa `last_scanned_at` | `13:49:40.551635+00` | `13:49:40.551635+00` | = untouched |
| **kimi** daa `last_guard_status` | `true` | `true` | = untouched |

The claude-code/codex `last_scanned_at` values both move to **13:54:05** — the write-activity commit
time of THIS run (> the 13:51:10 baseline) — while every kimi row is byte-identical before and after.
That is the board's OR-branch, met on the real deployed lane.

## Honest scope notes
- **Terminal state:** the workflow itself remained `status=RUNNING` (WorkflowExecutionStatus=1) past
  the driver's 150s await and at a follow-up `describe`; it is a long-running orchestration. I do
  **not** claim a terminal Completed state. The positive signal presented is the **other** accepted
  form — the real per-agent write activities (`process_single_agent`, `activities.py:2097`)
  committed, observable as the claude/codex `last_scanned_at` advance. The board offered these as
  alternatives ("… **or** …"); the row-advance is the stronger, directly-observed DB effect.
- **kimi daa `last_scanned_at` = 13:49:40**, not the original 10:35 scan time. That 13:49:40 stamp
  was written by an **earlier intermediate run that still contained kimi** (a mis-built omit payload,
  since corrected). It is fixed at `13:49:40.551635` **both before and after this clean run** — i.e.
  this kimi-absent rescan did **not** re-touch it. The point stands: rescanning without kimi neither
  removes the kimi asset (still `is_deleted=false`), deactivates its tools (still 9/9 active), nor
  re-scans its daa row. This is the upstream-visible face of the decisive removal finding —
  desktop-agent rescan is **upsert-only, no reaper** (`per_user_asset_writer.py:38`;
  `upsert_tools_bulk` at `tools.py:206` called WITHOUT `deactivate_tools_not_in_list`).

## Reachability
Verbatim driver log + both psql snapshots: `74-c13-laneA-omit-positive-signal-raw.txt` (same commit).
Settlement-classes context and lane-B side: `72-c13-c2-settlement-classes.md`. Removal finding:
prior C2/C13 evidence files.
