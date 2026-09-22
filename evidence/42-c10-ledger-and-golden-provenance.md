# C10 — "A changed test must still be a test": golden provenance + run-column ledger

- **PR:** #13392 (ticket PRDCT-14240, Kimi Code/CLI)
- **Branch:** `factory-eval-v32/PRDCT-14240`
- **Local checkout HEAD:** `74bded82133c1ac5da9b5f3ff0d5b47f95e8cbfc`
- **Date:** 2026-09-20

## 0. Commit reconciliation (read first)

The local checkout HEAD is `74bded8213` — a **local, unpushed** merge commit
("Merge remote-tracking branch 'origin/main' into factory-eval-v32/PRDCT-14240").
The **pushed PR head** (what GitHub CI actually ran) is
`6baa4672895569dc8ab0c5fb8d73b614361836c7` — verified via
`git ls-remote origin factory-eval-v32/PRDCT-14240` and `gh pr checks 13392`.

`6baa467` is an ancestor of `74bded8213` (`git merge-base --is-ancestor` = YES).
The delta `6baa467..74bded8213` is **entirely origin/main content**, not this PR's
C10 work: the new `mcp-server-lambda/**` feature tree, plus upstream additions to
two of the C10 files that come from *other* tickets merged from main —
`test_matchers.py` (new `*_coverage_indeterminate` runtime-guard cases) and
`test_parser_tool_call_direction_contract.py` (a new `TRUEFOUNDRY` contract row).
**None of this PR's own C10 test changes differ between `6baa467` and `74bded8213`.**
Therefore the green CI at `6baa467` is authoritative for the C10 test files as this
PR authored them, and the ledger below cites runs at `6baa467`.

---

## 1. Golden provenance — anchored OUTSIDE the scanner

### What the committed golden asserts

`git show HEAD:backend_python/scripts/e2e/scanner_golden/payloads.json` — the
`kimi-cli` entry (identical for `linux` and `macos`) commits this expectation:

```json
"agentCustomData": {
  "_type": "DefaultChannelCustomData",
  "channels": [
    {
      "channelId": "tui",
      "details": { "builtin": true, "path": "${HOME}/.kimi" },
      "displayName": "Terminal UI",
      "enabled": true,
      "key": "tui"
    }
  ]
},
"configExists": true,
"configPath": "${HOME}/.kimi",
"installEvidencePath": "${HOME}/.kimi",
"installEvidenceType": "dsl_dir"
```

### Authority artifact #1 — the real installed Kimi CLI package (upstream source)

Kimi CLI is installed on this box as a `uv` tool at
`/home/coder/.local/share/uv/tools/kimi-cli/lib/python3.12/site-packages/kimi_cli`.
Its build stamp (`kimi_cli/_build_info.py`) is:

```
BUILD_SHA = "github.com/MoonshotAI/kimi-cli@86f136422a0a"
```

The `${HOME}/.kimi` path is defined by the package itself, **independent of the
Onyx scanner**, in `kimi_cli/share.py`:

```python
def get_share_dir() -> Path:
    """Get the share directory path."""
    if share_dir := os.getenv("KIMI_SHARE_DIR"):
        share_dir = Path(share_dir)
    else:
        share_dir = Path.home() / ".kimi"       # <-- ${HOME}/.kimi
    share_dir.mkdir(parents=True, exist_ok=True)
    return share_dir
```

`kimi_cli/config.py` (`get_config_file → get_share_dir()/"config.toml"`) and
`kimi_cli/metadata.py` (`get_share_dir()/"kimi.json"`, `.../"sessions"/…`) both
resolve their state under that same `${HOME}/.kimi` root. This is the upstream
product deciding the path — not the scanner regenerating its own output.

### Authority artifact #2 — the real `~/.kimi` on disk

`ls -la ~/.kimi` and the config files confirm the layout the golden encodes:

```
$ ls -la ~/.kimi
-rw-------  config.toml
-rw-r--r--  mcp.json
-rw-------  kimi.json
drwxr-xr-x  sessions/
drwxr-xr-x  onyx/            # scanner sidecar (copied binary + onyx-config.json)
...

$ cat ~/.kimi/config.toml   (excerpt — Kimi's OWN hook + TUI config)
theme = 'dark'
show_thinking_stream = true
default_model = 'kimi-code/kimi-for-coding'
[[hooks]]
command = '"/home/coder/.kimi/onyx/onyx-scanner" hooks --source=kimi_cli --type=prompt'
event = 'UserPromptSubmit'
```

The real config lives at `~/.kimi/config.toml` (i.e. `${HOME}/.kimi`), exactly
the golden's `configPath` / `installEvidencePath`. `installEvidenceType: "dsl_dir"`
matches: the evidence is the presence of the `${HOME}/.kimi` directory.

### The `tui` / "Terminal UI" builtin channel

Kimi CLI ships **only** as a terminal coding agent (no IDE-extension surface). Its
own repo module documents this verbatim —
`backend_python/src/python_temporal_worker/integrations/desktop_agent_creator/builtin_agent_tools/kimi_cli.py`:

> "Kimi CLI (`kimi_cli`) — Moonshot AI's open-source **terminal coding agent**
> (`github.com/MoonshotAI/kimi-cli`)."

Because a bare CLI agent is reachable through exactly one surface, the scanner's
shared CLI DSL emits a single channel with `channel = "tui"` and
`details.builtin = true, details.path = ${HOME}/.kimi`
(`scanner/tools/dsl_agents/_cli.py`: `channel: str = "tui"`, "report exactly ONE
asset with one channel (`tui` for a terminal agent)"). The human-readable
`displayName` string `"Terminal UI"` is the shared surface label
(`scanner/internal/scanner/agents/channel_surface_identity.go:379`
`const tuiChannelDisplayName = "Terminal UI"`), applied to every terminal surface.

**Anchor verdict:** The load-bearing facts — `${HOME}/.kimi` path and a
`builtin: true` terminal channel keyed `tui` — are established by the upstream
Kimi CLI package (`kimi_cli/share.py`, build `MoonshotAI/kimi-cli@86f136422a0a`)
and by the real `~/.kimi` artifact on disk, **not** by the scanner regenerating
its own output. The scanner's own path helper even cites the package as its
source: `scanner/internal/agents/kimi/paths.go:23` —
`// (kimi_cli/share.py: get_share_dir honors $KIMI_SHARE_DIR, else ~/.kimi)`.
The `"Terminal UI"` string is the scanner's shared display label for that
artifact-established surface. The committed expectation matches the real Kimi CLI
artifact, not a scanner echo.

---

## 2. Run-column ledger — every changed test file ran & passed on a real green run

All Python jobs below belong to run **35246225073** (workflow *backend-python*,
event `pull_request`, conclusion `success`, headSha `6baa467`). All Go jobs
belong to run **35246225576** (workflow *Scanner PR Validation*, event
`pull_request`, conclusion `success`, headSha `6baa467`). Every run/job was
confirmed green via `gh run view`; pass counts are grep counts of
`PASSED <path>` (Python) / `--- PASS: <Test>` and `ok <pkg>` (Go) in the job log.

| # | Changed test file | Workflow / check | Trigger | Run / job id | Commit | Pass evidence (in job log) |
|---|-------------------|------------------|---------|--------------|--------|----------------------------|
| 1 | `backend_python/tests/sanitization_service/api/parsers/test_kimi_parser.py` | backend-python-test-sanitization-shard (1) | pull_request | run `35246225073` / job `105287129479` | `6baa467` | 29 × `PASSED …/test_kimi_parser.py` |
| 2 | `backend_python/tests/sanitization_service/api/parsers/test_parser_tool_call_direction_contract.py` | backend-python-test-sanitization-shard (3) | pull_request | run `35246225073` / job `105287129406` | `6baa467` | 37 × `PASSED …/test_parser_tool_call_direction_contract.py` |
| 3 | `scanner/internal/scanner/agents/kimi_hooks_output_test.go` | scanner-tests (ubuntu-latest) | pull_request | run `35246225576` / job `105287155251` | `6baa467` | 5 × `PASS: TestKimiHandleHookOutput*`; `ok onyx-scanner/internal/scanner/agents 177.898s` |
| 4 | `scanner/internal/scanner/agents/dsl_capability_ops_guard_scope_test.go` | scanner-tests (ubuntu-latest) | pull_request | run `35246225576` / job `105287155251` | `6baa467` | 3 × `PASS: TestOnyxGuardHooksPresentOp*`; `ok onyx-scanner/internal/scanner/agents` |
| 5 | `scanner/internal/scanner/guard_aggregate_test.go` | scanner-tests (ubuntu-latest) | pull_request | run `35246225576` / job `105287155251` | `6baa467` | 38 × `PASS: TestBuildGuardAggregate*` (funcs+subtests); `ok onyx-scanner/internal/scanner 302.762s` |
| 6 | `backend_python/tests/python_temporal_worker/integrations/desktop_agent_creator/test_builtin_tools.py` | backend-python-test-temporal-shard (1) | pull_request | run `35246225073` / job `105287129405` | `6baa467` | 324 × `PASSED …/test_builtin_tools.py` |
| 7 | `backend_python/tests/python_temporal_worker/issue_detection/test_matchers.py` | backend-python-test-temporal-shard (1) | pull_request | run `35246225073` / job `105287129405` | `6baa467` | 416 × `PASSED …/issue_detection/test_matchers.py` (see §3) |
| 8 | `backend_python/tests/common/repository/issues/test_issue_repository.py` | backend-python-test-common-shard (2) | pull_request | run `35246225073` / job `105287129752` | `6baa467` | 106 × `PASSED …/test_issue_repository.py` |
| 9 | `backend_python/tests/crud_service/api/test_access_control.py` | backend-python-test-crud-shard (2) | pull_request | run `35246225073` / job `105287129523` | `6baa467` | 40 × `PASSED …/test_access_control.py` |
| 10 | `backend_python/tests/common/repository/test_asset_repository_integration.py` | backend-python-test-common-shard (4) | pull_request | run `35246225073` / job `105287129669` | `6baa467` | 73 × `PASSED …/test_asset_repository_integration.py` |
| 11 | `backend_python/tests/crud_service/api/test_inventory_applications.py` | backend-python-test-crud-shard (2) | pull_request | run `35246225073` / job `105287129523` | `6baa467` | 160 × `PASSED …/test_inventory_applications.py` |
| 12 | `backend_python/tests/common/capability_matrix/test_catalog_links.py` | backend-python-test-common-shard (4) | pull_request | run `35246225073` / job `105287129669` | `6baa467` | 1 × `PASSED …/test_catalog_links.py` |
| 13 | `backend_python/tests/asset_inventory/_projection_sql_golden.json` (golden) | backend-python-test-asset-inventory | pull_request | run `35246225073` / job `105287129723` | `6baa467` | `PASSED tests/asset_inventory/test_projection_sql_snapshot.py::test_projection_sql_is_byte_identical_to_golden` (630 passed total) — the golden is exercised byte-for-byte, not merely present |

Every row is a real test that CI collected and executed to a PASS (the golden is
consumed by a byte-identity assertion that ran green). No file was reduced to a
no-op or skipped.

---

## 3. Temporal shard is GREEN at head — matcher test collected & ran

**Prior failure (from the how-to-close):** at commit `7cfd87b032`,
`backend-python-test-temporal-shard (4)` died in xdist collection
("Different tests were collected between gw0 and gw1" in `power_automate`).

**Now:** at the pushed PR head `6baa467`, `gh pr checks 13392` reports **all four**
temporal shards `pass`:

- shard (1) — run `35246225073` / job `105287129405` — pass, 9m26s
- shard (2) — job `105287129412` — pass
- shard (3) — job `105287129636` — pass
- shard (4) — run `35246225073` / job `105287129380` — pass, 6m56s

**Which shard collected + ran the matcher test:** `test_matchers.py` was collected
and executed on **temporal shard (1)** (job `105287129405`):

- `416` × `PASSED tests/python_temporal_worker/issue_detection/test_matchers.py::…`
  (e.g. `[gw0] [ 0%] PASSED …TestMatchClaudeDesktopUnmanagedLicense…`,
  `[gw2] [ 91%] PASSED …TestMcpCatalogMetadata…`).
- Shard summary: `================ 3748 passed, 60 warnings in 434.03s =================`.
- **No** `Different tests were collected` / xdist collection mismatch anywhere in
  the shard-1 log (grep for `Different tests were collected|xdist error|INTERNALERROR`
  returns nothing; the only "collection failure" string is a benign conditional
  echo in the coverage-rename step, which did produce a `.coverage` file).

The xdist collection bug is resolved (main added the explicit `ids=[...]` guard in
`power_automate`), and the shard that owns `issue_detection/test_matchers.py`
(shard 1) collected the full module and ran all 416 of its cases to PASS at head.
