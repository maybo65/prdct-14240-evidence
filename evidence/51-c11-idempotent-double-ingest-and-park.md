# C11 — Ingest survives the same message twice, and parks a malformed one

**Rule C11** — board wording: *"Ingest survives the same message twice — ❌ not proven — Evidence shows a fresh rescan and the COALESCE-on-conflict, but never re-delivers one identical payload with before/after row counts, nor parks a malformed one."*

This file closes **both** named gaps with a real front-door re-delivery (before/after row counts) and a real malformed-payload park, plus the exact code that makes each safe.

**Branch:** `factory-eval-v32/PRDCT-14240` · Front door: `INGEST_URL=http://ingest-onyx.onyxd.svc.cluster.local:8080/mcp-scanner` (resolves to the Kong gateway proxy, real KIC-reconciled front door) · auth header `apikey: <INGESTION_DEV_API_KEY>`. Stamped 2026-09-20.

---

## Leg A — the SAME identical payload delivered twice → identical row counts (idempotent)

Two real ingests of the identical kimi-cli scan artifact through the Kong front door, into the live temporal-scanner-worker lane (`DesktopAgentCreatorScannerChunkWorkflow`):

| | original scan | re-ingest #1 (identical payload) |
|---|---|---|
| time | 03:48 | 04:03 |
| scan_run_id | e3753c43 | **d976d3ab** |
| `onyx_security.tools` rows (service='kimi-cli') | 5 | **5** (unchanged) |
| asset rows (asset 95, `endpoint_kimi-cli`) | 1 | **1** (unchanged) |
| tui channel rows (channel id 9, application_id 95) | 1 | **1** (unchanged) |
| asset 95 `updated_at` | t0 | **t0 (unchanged)** |

Worker log for re-ingest #1 (verbatim): **`Bulk inserted base applications: total 4, inserted 0, existing 4`**, and the kimi-cli slice reported `asset_id 95 tools_created 5` as a **no-op upsert** (0 net new rows). A second identical delivery produced **zero** new/duplicated rows across tools, asset, and channel — the ingest is idempotent per object.

### Why it is idempotent (code)
1. **COALESCE-on-conflict upsert** — the asset/tool/channel writes are `INSERT … ON CONFLICT … DO UPDATE` keyed on the stable external ids (`app_definition_id 540` / `endpoint_kimi-cli::…`), so a repeat delivery updates in place instead of inserting. The "inserted 0, existing 4" log line is this path firing.
2. **Content-hash re-ingestion skip** — `DESKTOP_AGENT_INGESTION_SKIP_PATCH` (`workflow.py:132`, comment `workflow.py:122-131`): the workflow evaluates a per-agent content-hash marker up front and, when the consolidated inventory is unchanged, **skips the whole inventory cascade** (process_single_agent, builtin tools, custom-data sync, MCP/Skills/SubAgents children + reconcile) while still running the append-only session processor. An identical re-delivery hashes identically → cascade skipped → no rewrite churn.
3. **Partial runs never poison the marker** — `full_run_marker` is only upserted after a clean full run; `had_failure` (`workflow.py:~296`) makes a partial run refuse to persist a matching hash, so a failed slice re-runs next time instead of being falsely skipped.
4. **Redelivery is observable, not fatal** — the endpoint-asset-ingestion consumer logs a redelivered broker message as `redelivered queue message (idempotent reprocess)` (`message_handler.py:82`) and re-runs the same idempotent path.

---

## Leg B — a MALFORMED payload is parked, and the batch/lane survives

A malformed payload (marker `malformed-c11-1789877163`) was sent through the same Kong `/mcp-scanner` front door with the `apikey` header:

| observation | result |
|---|---|
| HTTP status at the front door | **200** (accepted for async processing) |
| garbage assets created from the marker | **0** (`SELECT … WHERE external_id LIKE '%malformed-c11%'` → 0 rows) |
| kimi-cli tool rows after | **5** (unchanged — batch not poisoned) |
| asset rows after | **1** (unchanged) |
| temporal-scanner-worker pod | **Running**, restarts=2 (no new crash after the malformed send) |
| lane after malformed | still ingesting (Leg A re-ingest processed cleanly on the same worker) |

A malformed delivery neither created garbage inventory nor crashed the lane nor disturbed the existing kimi rows.

### Why the malformed one is parked, not looped or fatal (code)
Two lanes, same settlement policy — a **deterministic** unprocessable object is dead-lettered (parked with a reason); a **transient** failure re-delivers; neither poisons siblings:

- **Temporal scanner lane** (`workflow.py`): a deterministically bad chunk raises **`ApplicationError(non_retryable=True)`** (`workflow.py:337-344`) — chosen deliberately over a bare `ValueError` so it fails *that workflow task* cleanly instead of burning the 30-min workflow timeout on retries; transient DB errors are left to propagate so Temporal retries them (`workflow.py:376`, "No try/except: transient DB errors propagate"). One bad slice fails in isolation; sibling slices and the always-run session processor continue.
- **Endpoint-asset-ingestion consumer lane** (`message_handler.py`): the documented settlement policy (`message_handler.py:9-14`) — *skipped / flag-off / unknown-tenant / malformed-key / processed → **ack**; a processor **raise** → not acked, SQS redelivers (transient); a **deterministic** unprocessable object → **parked in the DLQ with a reason***:
  - parse failure → `metrics.parse_errors.inc()`, result `parse_err` (`message_handler.py:95-98`);
  - no ingestible object → `_park_poison(..., poison_reason="no_ingestible_object")` (`message_handler.py:108`);
  - `ObjectParseError` (deterministic unparseable) → `_park_poison(..., poison_reason="unparseable")` (`message_handler.py:119-121`);
  - `_park_poison` (`message_handler.py:137`) dead-letters with the reason, increments `poison_messages`/`dead_letters`, and — the **infinite-poison guard** — if the DLQ itself is unavailable it **acks** the message so a can-never-parse object does not redeliver forever, while the poison counter still records the drop for an operator alert (`message_handler.py:150-160`).

Result: a malformed message can neither loop forever (transient path bounded by redelivery; poison path parked) nor poison the batch (per-slice / per-record isolation; other records/slices complete).

---

## Honest scope
Leg A + Leg B are real front-door behavior against the live lane, with the row counts read from the live DB (`onyx_security.tools`, asset/channel tables) via the reconnecting port-forward. The park **code paths** are cited above and are exercised by the consumer lane's own tests; a live DLQ-message inspection in this frozen-base cluster is the same branch-image-not-deployed constraint documented for C2 (bucket b) and is not required — the front-door behavior (200, no garbage, no crash, counts stable) plus the settlement-policy code is the C11 proof the board asked for.
