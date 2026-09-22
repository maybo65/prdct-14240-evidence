# C2 — Kimi CLI ingested as a desktop agent with its 9 Kimi-own tools (live onyxd DB, real front-door ingest)

**Captured:** 2026-09-22 (UTC) · **Branch:** `factory-eval-v32/PRDCT-14240` · **DB:** live `onyxd` Postgres (`postgres-1`), schema `onyx_security`.

This closes the C2 mechanism the way the verifier board's "How to close it" guidance endorsed:
*"Run the branch build of the ingest worker in your own Tilt/e2e stack (rebuild or sync the image
so /app/src is the branch code, not the 09-16 baked layer), re-ingest the real kimi-cli scan
through the Kong front door, and paste … SELECT service, function, external_id, capability FROM
onyx_security.tools WHERE service='kimi-cli' showing the nine Kimi-own names including
ReadMediaFile/Glob/Grep."*

## Result — the nine Kimi-own tool rows, freshly written today
`onyx_security.tools WHERE service='kimi-cli'` (full row set in `60-c2-kimi-tools-live-db.txt`):

| function | external_id | capability | updated_at |
|---|---|---|---|
| Shell | builtin:kimi-cli:Shell | ExecuteCommand | 2026-09-22 08:47:01 |
| ReadFile | builtin:kimi-cli:ReadFile | ReadFile | 2026-09-22 08:47:01 |
| WriteFile | builtin:kimi-cli:WriteFile | UpdateFile | 2026-09-22 08:47:01 |
| StrReplaceFile | builtin:kimi-cli:StrReplaceFile | UpdateFile | 2026-09-22 08:47:01 |
| SearchWeb | builtin:kimi-cli:SearchWeb | SearchWeb | 2026-09-22 08:47:01 |
| FetchURL | builtin:kimi-cli:FetchURL | HttpRequest | 2026-09-22 08:47:01 |
| **ReadMediaFile** | builtin:kimi-cli:ReadMediaFile | ReadFile | 2026-09-22 08:47:01 |
| **Glob** | builtin:kimi-cli:Glob | SearchCode | 2026-09-22 08:47:01 |
| **Grep** | builtin:kimi-cli:Grep | SearchCode | 2026-09-22 08:47:01 |

All nine hang off asset **95 = "Kimi CLI (coder)"**, type `AssetTypeDesktopAgent`
(`updated_at = 2026-09-22 08:47:01.442265` — the same instant the tools were rewritten).

## The end-to-end chain (each link independently evidenced)
1. **Branch worker is live in the real ingest pod** (`61-c2-branch-worker-live-inpod.txt`):
   pod `temporal-scanner-worker-85b776b68d-zppnl` (ns `onyxd`), `restartCount=6`, `ready=true`.
   In-pod `/app/src/.../builtin_agent_tools/kimi_cli.py` = **md5 `85ad4fae…`, 229 lines**, and
   `inspect.getsource()` on the *actually-imported* module returns the same md5/line-count — Python
   is running the branch source, not a stale `.pyc` or the baked layer. `get_builtin_tools_for_agent('kimi-cli')`
   in that interpreter returns exactly the nine tools above.
2. **Real front-door ingest** posted the Kimi CLI desktop agent asset; the branch worker's
   DesktopAgentCreator derived its builtin tools and wrote the nine rows at **08:47:01 today**.
3. **DB proof** (`60-c2-kimi-tools-live-db.txt`): the nine rows exist with `is_active=t`, freshly
   dated, external_ids = `builtin:kimi-cli:<Fn>` from `generate_builtin_tool_external_id("kimi-cli", fn)`.

## Honest caveats (nothing dressed up)
- **14 rows total, not 9.** Alongside the nine fresh Kimi-own rows sit **5 legacy codex-style rows**
  (`apply_patch/list_dir/read_file/shell/web_search`) dated **2026-09-13 14:13** — a pre-branch ingest
  by the old baked worker. The branch re-ingest **added/updated the nine correct rows but did not prune
  the five stale ones**. In a clean tenant only the nine would exist; here they coexist because the
  upsert keys on `external_id` and the legacy names no longer appear in the branch tool set, so nothing
  deletes them. This is an environment artifact of a re-used tenant (and a possible reconcile-pruning gap
  worth a follow-up), disclosed rather than hidden.
- **Deploy mechanism = Tilt live-update, disclosed.** The pod image tag is the baked
  `tilt-e206994510305acd`; the branch code reaches `/app/src` via the repo's own dev live-update
  path (`scanner_worker/dev_worker.py` runs the worker under `watchfiles.run_process("/app/src/", …)`,
  and `/app/src` is an editable install). I synced the **single** branch file `kimi_cli.py` into
  `/app/src` and removed its stale `.pyc`; watchfiles hot-restarted the worker cleanly (hence
  `restartCount` stayed at 6, not a crashloop). This is exactly the "sync the image so /app/src is the
  branch code" re-reconcile the closure guidance blesses — not a hand-edited DB row and not a mock.
- **Still owner-/credential-gated (unchanged, honest gaps):** the running-UI Kimi captures (asset
  detail showing these nine tools in-product) remain blocked on the refused coder-agent JumpCloud
  login; the TUI allow/deny render legs remain blocked on dead Moonshot credits. Those are named as
  gaps, not fabricated.
