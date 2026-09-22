# C2 + C11 — clean, fully-logged reproduction of the Kimi-CLI desktop-agent ingest through the real Kong front door

**Captured:** 2026-09-22 UTC · **Branch:** `factory-eval-v32/PRDCT-14240` (HEAD `fce925c99b`) · **DB:** live `onyxd` Postgres, schema `onyx_security`.

This supersedes the write-provenance framing in `62-c2-kimi-tools-e2e-summary.md`. That earlier file
proved the nine rows *exist* and that the branch worker *derives* them in-pod, but its clean scan log
(`scan_full.log`) had run in **legacy** DSL mode 31 s after the rows' timestamp and so did **not** attribute
the write. The reproduction below closes that gap end-to-end with matching logs on both sides, and in doing
so also closes **C11** (double-ingest idempotency against the branch worker).

## What was run
- **Scanner binary:** freshly rebuilt from branch HEAD (`scanner/bin/onyx-scanner`, 09:18 UTC) — not a stale
  `/tmp` build. The 09-16 `/tmp/onyx-scanner` was verified *not* to post kimi-cli as a coding agent, which is
  exactly why the earlier write could not be reproduced with it.
- **Front door:** Kong route `POST /mcp-scanner` on `svc/onyx-onyxd-platform-gateway-proxy` (port-forward
  `127.0.0.1:8080`, `Host: ingest-onyx.onyxd.svc.cluster.local` per `/etc/hosts`). Auth is **real and
  enforced**: no key → **401**, seeded dev `ingestion` key → **200** (`tilt/infra.star:1108`). No synthesized
  Kong/consumer headers, no minted JWT, no DB hand-edit.
- **Branch worker:** pod `temporal-scanner-worker-85b776b68d-zppnl` (ns `onyxd`), branch `kimi_cli.py` live in
  `/app/src` (md5 `85ad4fae`, 229 lines) via the repo's own Tilt live-update path.
- **Command (run twice, 09:19:36 and 09:22:48 UTC):**
  `onyx-scanner scan --endpoint http://ingest-onyx.onyxd.svc.cluster.local:8080/mcp-scanner
  --api-key <ingestion> --dsl-enabled --dsl-file scanner/internal/scanner/agents/embedded_dsl_linux.json
  --full-scan=false`

## The chain, each link logged (files 64/65/66)
1. **Scanner detects + posts** (`64-c2-c11-scan-ingest-log.txt`): DSL mode `dsl` (decided_by flag),
   `agent_detection kimi-cli detected=true evidence=dsl_dir`, `Scan run completed payloads_sent: 3`. The
   `--output-file` dump shows the posted record: `type=request method=scan source_type=mcp-scanner`,
   `data.codingAgent.name='kimi-cli'` — **no builtin tools in the payload** (they are derived server-side).
2. **Branch worker processes + derives 9 tools** (`65-c2-c11-worker-processing-log.txt`): the scanner-lane
   `DesktopAgentCreatorScannerChunkWorkflow` logs, on **both** ingests,
   `Processing single agent app_name=kimi-cli normalized_app_def_id=endpoint_kimi-cli` →
   `Creating builtin tools for agent agent_name=kimi-cli asset_id=95 tool_count=9` →
   `Builtin tools creation complete agent_name=kimi-cli asset_id=95 tools_created=9`.
   (Same batch: claude-code=10, codex=5 — 9 is unique to kimi-cli.)
3. **DB holds exactly those 9** (`66-c2-c11-db-proof.txt`): rows ids 21–29,
   external_ids `builtin:kimi-cli:{Shell,ReadFile,WriteFile,StrReplaceFile,SearchWeb,FetchURL,ReadMediaFile,
   Glob,Grep}`, `is_active=t`, all on asset **95 "Kimi CLI (coder)"** (`AssetTypeDesktopAgent`), including the
   three the audit named — **ReadMediaFile / Glob / Grep**.

## C11 — double-ingest idempotency, against the BRANCH worker
The verifier's C11 gap was that the prior idempotency check ran against pre-branch (5-tool) code. Here the
**branch** worker (`tools_created=9`, `endpoint_kimi-cli`) processed **two consecutive** front-door ingests
(09:20:00 and 09:24:00). Result (`66-…-db-proof.txt`): builtin row ids **stable at 21–29**, count stays **9**,
**zero** duplicate `external_id`s, and exactly **one** kimi desktop-agent asset (95). The re-ingest is a clean
content-hash upsert — no new rows, no id churn, no duplicate asset. (This is also why `updated_at` stays at
the first write's `08:47:01` — an identical-content upsert does not bump it.)

## Honest notes (nothing dressed up)
- **14 rows total, still not pruned to 9.** Alongside the nine Kimi-own rows sit **5 legacy codex-style rows**
  (`apply_patch/list_dir/read_file/shell/web_search`, dated 09-13) from a pre-branch ingest by the old baked
  worker. The branch re-ingest correctly writes the nine and does **not** delete the five stale ones (the
  upsert keys on `external_id`; legacy names no longer appear in the branch tool set, so nothing removes them).
  Environment artifact of a re-used tenant / possible reconcile-pruning follow-up — disclosed, not hidden.
- **Deploy mechanism = Tilt live-update, disclosed** (same as file 62): the branch `kimi_cli.py` reaches
  `/app/src` via the repo's dev live-update path, not a rebuilt image tag.
- **C11 DLQ leg (malformed → dead-letter)** is tracked separately; this file covers the idempotency half.
- **Still owner-/credential-gated (unchanged honest gaps):** the running-UI Kimi captures (asset-detail
  showing these nine tools in-product) remain blocked on the refused coder-agent JumpCloud login; the TUI
  allow/deny render legs remain blocked on dead Moonshot credits. Named as gaps, not fabricated.
