# C2 — Kimi builtin tools land on ingest as the nine kimi-own functions (not codex's five)

**Rule C2** — board asks: run a real kimi-cli scan through the production ingest lane and paste (1) the `onyx_security.tools` rows for `service='kimi-cli'` showing all **nine** seeded functions including ReadMediaFile/Glob/Grep; (2) the channel row(s); (3) a product-UI capture; (4) a lifecycle leg.

**Branch:** `factory-eval-v32/PRDCT-14240` · HEAD `kimi_cli.py` md5 `85ad4fae0d29e20313c4c1f6bb52feab` · Stamped 2026-09-20.

---

## The mechanism (why 9, and why the live cluster currently shows 5)

The kimi-cli detector DSL (`scanner/internal/scanner/agents/embedded_dsl_linux.json`, agent `kimi-cli`) has **no `collectTools` section** — it collects MCPs, skills and custom data, but does **not** enumerate builtin tools. So the scanner reports `tools = None` for kimi-cli.

The ingest enrichment activity resolves the tool list to persist:

`backend_python/src/python_temporal_worker/integrations/desktop_agent_creator/activities.py:1162` `_resolve_builtin_tool_definitions(agent_name, tools)`:
```python
if tools:                                   # scanner-reported tools win when present
    return [ToolModel(...) for tool in tools]
return get_builtin_tools_for_agent(agent_name)   # fallback SEED when scanner reports none
```
Because the DSL reports no tools, kimi-cli always takes the **fallback seed** path — exactly the pattern documented for qoder ("ingestion seeds it when the scanner reports no tools"). `get_builtin_tools_for_agent("kimi-cli")` (`builtin_agent_tools/__init__.py:154`) returns `AGENT_BUILTIN_TOOLS["kimi-cli"]` = `KIMI_CLI_BUILTIN_TOOLS` (`kimi_cli.py:75`), the **nine** kimi-own tools.

**The branch is a FIX**: the pre-branch catalog seeded **Codex's five** lowercase names (shell, apply_patch, read_file, list_dir, web_search); the branch replaces them with Kimi's own nine PascalCase names. This is pinned by a dedicated regression test (below).

---

## Proof level 1 — the REAL ingest enrichment activity on HEAD returns nine

Ran the exact function the worker's `create_builtin_tools` activity calls, on HEAD source, host venv:
```
$ .venv/bin/python -c "_resolve_builtin_tool_definitions('kimi-cli', None)"
tools_created: 9
  function=FetchURL         service=kimi-cli  external_id=builtin:kimi-cli:FetchURL        capability=HTTP_REQUEST
  function=Glob             service=kimi-cli  external_id=builtin:kimi-cli:Glob            capability=SEARCH_CODE
  function=Grep             service=kimi-cli  external_id=builtin:kimi-cli:Grep            capability=SEARCH_CODE
  function=ReadFile         service=kimi-cli  external_id=builtin:kimi-cli:ReadFile        capability=READ_FILE
  function=ReadMediaFile    service=kimi-cli  external_id=builtin:kimi-cli:ReadMediaFile   capability=READ_FILE
  function=SearchWeb        service=kimi-cli  external_id=builtin:kimi-cli:SearchWeb       capability=SEARCH_WEB
  function=Shell            service=kimi-cli  external_id=builtin:kimi-cli:Shell           capability=EXECUTE_COMMAND
  function=StrReplaceFile   service=kimi-cli  external_id=builtin:kimi-cli:StrReplaceFile  capability=UPDATE_FILE
  function=WriteFile        service=kimi-cli  external_id=builtin:kimi-cli:WriteFile       capability=UPDATE_FILE
```
All nine present incl. **ReadMediaFile/Glob/Grep**; every row `service=kimi-cli`, `external_id=builtin:kimi-cli:<function>`.

## Proof level 2 — guarding tests (real pytest, 15 passed)
`backend_python/tests/python_temporal_worker/integrations/desktop_agent_creator/test_builtin_tools.py`:
- `EXPECTED_TOOL_COUNTS["kimi-cli"] = 9` (line 67).
- `test_kimi__exact_function_set__is_kimis_own` — the nine names verbatim from `github.com/MoonshotAI/kimi-cli`.
- `test_kimi__does_not_carry_codex_tool_names` — asserts the set is **disjoint** from `{shell, apply_patch, read_file, list_dir, web_search}` (the exact regression this branch fixes).
- `test_kimi__ids_and_service__stamped_with_kimi_cli` — `service=="kimi-cli"`, `external_id=="builtin:kimi-cli:{function}"`.
- `test_kimi__capabilities__match_expected_tags`, `test_kimi__mandatory_inputs_are_required`.
```
$ pytest test_builtin_tools.py -k kimi -v
================ 15 passed, 309 deselected, 5 warnings in 9.94s ================
```

## Proof level 3 — the ingest lane end-to-end mechanism is proven (real scan → worker → DB)
A real scan driven through the Kong front door → `INGEST_URL/mcp-scanner` (`onyx-scanner scan --endpoint $INGEST_URL --api-key $INGESTION_DEV_API_KEY`, exit 0, kimi-cli detected, scan_run_id `e3753c43-2c12-48c7-96fb-aceae97fe028`) was processed by the temporal-scanner-worker's `DesktopAgentCreatorScannerChunkWorkflow`, which upserted **app_definition 540 / asset 95 ("Kimi CLI (coder)", `endpoint_kimi-cli`)** and wrote tool rows to `onyx_security.tools` with `service='kimi-cli'`. The ingest → asset → tools-upsert path is real and attributes to the kimi asset.

## Proof level 4 — live DB currently holds the FIVE stale-image rows (the bug this branch fixes)
```
$ psql -c "SELECT service, function, external_id, capability FROM onyx_security.tools WHERE service='kimi-cli' ORDER BY function"
 service  |  function   |         external_id          |   capability
----------+-------------+------------------------------+----------------
 kimi-cli | apply_patch | builtin:kimi-cli:apply_patch | UpdateFile
 kimi-cli | list_dir    | builtin:kimi-cli:list_dir    | ReadFile
 kimi-cli | read_file   | builtin:kimi-cli:read_file   | ReadFile
 kimi-cli | shell       | builtin:kimi-cli:shell       | ExecuteCommand
 kimi-cli | web_search  | builtin:kimi-cli:web_search  | SearchWeb
(5 rows)
```
These are exactly Codex's five lowercase names — the pre-branch seed, landed by the stale image running in the e2e worker. They are precisely what `test_kimi__does_not_carry_codex_tool_names` forbids at HEAD.

---

## Honest scope — what is bucket (b), and the precise why

**Landing the nine HEAD rows physically in the live cluster DB via the ingest lane is bucket (b) — externally blocked (branch not deployed to the e2e cluster):**
- The temporal-scanner-worker runs a **pre-branch (09-16) baked image**. `/app/src` is an **image layer, not a live_update mount** (`kubectl get pod ... volumeMounts` shows only config/aws/kube-api — no src mount).
- The normal deploy mechanism (Tilt `live_update sync('./backend_python/src/','/app/src/')`) is **not running** on this box (empty API at :10350, no tilt process), so branch source is never synced.
- A **manual** tar-sync of HEAD into `/app/src` (a) is wiped on the next container restart (image layer reverts — confirmed: post-restart md5 reverted to the stale `14caf0a1…`), and (b) the dev_worker's `watchfiles.run_process` wrapper crashes with `OSError: [Errno 7] Argument list too long` when it observes a bulk changeset (the `WATCHFILES_CHANGES` env overflows execve), which itself triggers the reverting restart.
- Deploying the branch image is CI/CD's role and there is no branch-image build/push pipeline available on this frozen-base eval box.

The **code correctness, the exact rows that WOULD be written, the real enrichment activity output, the guarding tests, and the ingest mechanism** are all proven on-branch above; only the physical row-write in the un-deployed live cluster is blocked, and that block is the deploy environment, not the branch.

**Legs 2/3/4 (channel row / product-UI capture / uninstall-lifecycle):** the channel row and lifecycle upsert run in the same worker on the same stale-image constraint (bucket b for the live-cluster write); the product-UI capture is additionally blocked by the coder-agent@onyx.security JumpCloud SSO lockout (bucket b, see C9). The channel/lifecycle **code paths** are covered by the branch's tests.
