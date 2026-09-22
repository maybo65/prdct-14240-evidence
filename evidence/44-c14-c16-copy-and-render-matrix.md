# C14 + C16 — Copy Inventory & Render Matrix (PR #13392, PRDCT-14240)

**Stamp:** HEAD `74bded8213` (`74bded82133c1ac5da9b5f3ff0d5b47f95e8cbfc`) / 2026-09-20
**For:** PR-body inclusion.

> Producible doc halves of C14/C16; the OWNER DECISION (approve invented strings / fund brand-icon captures) remains with May (bucket a), and the running-UI icon/deny captures are blocked on the SSO lockout (bucket b).

Every string below is quoted verbatim from HEAD. Sources:
- `backend_python/src/python_temporal_worker/integrations/desktop_agent_creator/builtin_agent_tools/kimi_cli.py` (the catalog; ships NINE tools)
- `scanner/internal/scanner/agents/kimi_hooks.go` (the runtime hook lane)
- `scanner/internal/scanner/agents/codex_hooks.go` (the sibling lane the block messages reuse)
- `scanner/internal/messages/block_message.go` (`BuildBlockMessage`, the composer)
- `backend_python/src/common/capability_matrix/links.py` (the hooks-only lane decision)

---

## C14 — Invented-vs-Reuse copy inventory (refreshed to HEAD)

All nine tool `external_description` strings and every `description` in each tool's
`parameters` are Onyx-authored catalog copy (INVENTED — they are not lifted from Kimi;
only the tool `name`/`function` identifiers are Kimi's verbatim, per the module docstring).
They are stored on `ToolModel.external_description` / JSON-Schema `parameters[].description`
and surface in the Onyx tool inventory / catalog UI, NOT rendered by the agent to an end user.
The three scanner block messages are REUSE — byte-identical to `codex_hooks.go` (see proof below).

### Tool descriptions (9)

| # | String | file:symbol | Invented / Reuse | Rendered where |
|---|--------|-------------|------------------|----------------|
| 1 | `Execute a shell command.` | kimi_cli.py: `Shell` external_description | Invented | Onyx tool catalog (external_description) |
| 2 | `Read the contents of a file.` | kimi_cli.py: `ReadFile` external_description | Invented | Onyx tool catalog |
| 3 | `Write content to a file (overwrite or append).` | kimi_cli.py: `WriteFile` external_description | Invented | Onyx tool catalog |
| 4 | `Edit a file by replacing string(s) in place.` | kimi_cli.py: `StrReplaceFile` external_description | Invented (StrReplaceFile row corrected to shipped wording) | Onyx tool catalog |
| 5 | `Search the web for information.` | kimi_cli.py: `SearchWeb` external_description | Invented | Onyx tool catalog |
| 6 | `Fetch the content of a URL.` | kimi_cli.py: `FetchURL` external_description | Invented | Onyx tool catalog |
| 7 | `Read an image or video file.` | kimi_cli.py: `ReadMediaFile` external_description | Invented (NEW at HEAD) | Onyx tool catalog |
| 8 | `Match files and directories by glob pattern.` | kimi_cli.py: `Glob` external_description | Invented (NEW at HEAD) | Onyx tool catalog |
| 9 | `Search file contents with a regular expression (ripgrep).` | kimi_cli.py: `Grep` external_description | Invented (NEW at HEAD) | Onyx tool catalog |

### Parameter descriptions (32)

| # | String | file:symbol | Invented / Reuse | Rendered where |
|---|--------|-------------|------------------|----------------|
| 1 | `The command to execute` | Shell.command | Invented | catalog / JSON-Schema |
| 2 | `Timeout in seconds` | Shell.timeout | Invented | catalog / JSON-Schema |
| 3 | `Whether to run the command as a background task` | Shell.run_in_background | Invented | catalog / JSON-Schema |
| 4 | `A short description for the background task (required when run_in_background=true)` | Shell.description | Invented | catalog / JSON-Schema |
| 5 | `Path to the file to read` | ReadFile.path | Invented | catalog / JSON-Schema |
| 6 | `Line to start reading from` | ReadFile.line_offset | Invented | catalog / JSON-Schema |
| 7 | `Number of lines to read` | ReadFile.n_lines | Invented | catalog / JSON-Schema |
| 8 | `Path to the file to write` | WriteFile.path | Invented | catalog / JSON-Schema |
| 9 | `The content to write to the file` | WriteFile.content | Invented | catalog / JSON-Schema |
| 10 | `Whether to overwrite the file or append to it` | WriteFile.mode | Invented | catalog / JSON-Schema |
| 11 | `Path to the file to edit` | StrReplaceFile.path | Invented (corrected to shipped wording) | catalog / JSON-Schema |
| 12 | `A single edit, or an array of edits applied in order.` | StrReplaceFile.edit | Invented (corrected to shipped wording) | catalog / JSON-Schema |
| 13 | `The original string to replace (multi-line supported)` | `_STR_REPLACE_EDIT`.old (shared edit-object) | Invented (corrected to shipped wording) | catalog / JSON-Schema |
| 14 | `The replacement string (multi-line supported)` | `_STR_REPLACE_EDIT`.new (shared edit-object) | Invented (corrected to shipped wording) | catalog / JSON-Schema |
| 15 | `Replace all occurrences of old; defaults to false` | `_STR_REPLACE_EDIT`.replace_all (shared edit-object) | Invented (corrected to shipped wording) | catalog / JSON-Schema |
| 16 | `The query text to search for` | SearchWeb.query | Invented | catalog / JSON-Schema |
| 17 | `Maximum number of results to return` | SearchWeb.limit | Invented | catalog / JSON-Schema |
| 18 | `Whether to include the page content of each result` | SearchWeb.include_content | Invented | catalog / JSON-Schema |
| 19 | `The URL to fetch content from` | FetchURL.url | Invented | catalog / JSON-Schema |
| 20 | `Path to the media file to read` | ReadMediaFile.path | Invented (NEW at HEAD) | catalog / JSON-Schema |
| 21 | `Glob pattern (e.g., *.py, src/**/*.ts)` | Glob.pattern | Invented (NEW at HEAD) | catalog / JSON-Schema |
| 22 | `Search directory, defaults to the working directory` | Glob.directory | Invented (NEW at HEAD) | catalog / JSON-Schema |
| 23 | `Include directories in the results; defaults to true` | Glob.include_dirs | Invented (NEW at HEAD) | catalog / JSON-Schema |
| 24 | `Regular expression pattern` | Grep.pattern | Invented (NEW at HEAD) | catalog / JSON-Schema |
| 25 | `Search path, defaults to the current directory` | Grep.path | Invented (NEW at HEAD) | catalog / JSON-Schema |
| 26 | `File filter glob (e.g., *.js)` | Grep.glob | Invented (NEW at HEAD) | catalog / JSON-Schema |
| 27 | `File type filter (e.g., py, js, go)` | Grep.type | Invented (NEW at HEAD) | catalog / JSON-Schema |
| 28 | `Output mode; defaults to files_with_matches` | Grep.output_mode | Invented (NEW at HEAD) | catalog / JSON-Schema |
| 29 | `Limit output lines; defaults to 250` | Grep.head_limit | Invented (NEW at HEAD) | catalog / JSON-Schema |

*(Rows 20–29 are the ReadMediaFile / Glob / Grep parameter descriptions new at HEAD; rows 11–15 are the StrReplaceFile family whose wording the inventory is corrected to.)*

### Scanner block messages (3) — REUSE

| # | String | file:symbol | Invented / Reuse | Rendered where |
|---|--------|-------------|------------------|----------------|
| 30 | `The prompt has been blocked by the Onyx AI Guard` | kimi_hooks.go: `kimiPromptBlockMessage` | **Reuse** — byte-identical to codex_hooks.go `codexPromptBlockMessage` | Kimi TUI stderr reason + permissionDecision doc (UserPromptSubmit hard deny) |
| 31 | `The tool execution has been blocked by the Onyx AI Guard` | kimi_hooks.go: `kimiToolBlockMessage` | **Reuse** — byte-identical to codex_hooks.go `codexToolBlockMessage` | Kimi TUI stderr reason + permissionDecision doc (PreToolUse deny) |
| 32 | `The tool output has been blocked by the Onyx AI Guard` | kimi_hooks.go: `kimiPostToolBlockMessage` | **Reuse** — byte-identical to codex_hooks.go `codexPostToolBlockMessage` | Kimi additionalContext doc (PostToolUse audit) |

**Byte-identity proof.** The constant NAMES differ by prefix (`kimi*` vs `codex*`), but the string VALUES are identical to the byte:

```
kimi_hooks.go:33-35
  kimiPromptBlockMessage   = "The prompt has been blocked by the Onyx AI Guard"
  kimiToolBlockMessage     = "The tool execution has been blocked by the Onyx AI Guard"
  kimiPostToolBlockMessage = "The tool output has been blocked by the Onyx AI Guard"

codex_hooks.go:33-35
  codexPromptBlockMessage   = "The prompt has been blocked by the Onyx AI Guard"
  codexToolBlockMessage     = "The tool execution has been blocked by the Onyx AI Guard"
  codexPostToolBlockMessage = "The tool output has been blocked by the Onyx AI Guard"
```

`diff` of the three extracted string values across the two files reports **IDENTICAL** (0 differences).

**Copy-string totals:** 9 tool descriptions + 32 parameter descriptions = **41 catalog strings (all Invented)**, + **3 block messages (all Reuse)** = **44 strings inventoried**.

---

## C16 — Agent-by-surface render matrix

The base sentence is a scanner constant; the final rendered text is composed by
`messages.BuildBlockMessage(baseMessage, response.BlockReason, response.ViolatedRules)`
(`scanner/internal/messages/block_message.go:18`), which substitutes the policy's
`BlockReason` for the base when present and appends `\nViolated rules: <rules>` when any
are set. Kimi event names are `kimi.UserPromptSubmitEvent="UserPromptSubmit"`,
`kimi.PreToolUseEvent="PreToolUse"`, `kimi.PostToolUseEvent="PostToolUse"`
(`scanner/internal/agents/kimi/constants.go:12-14`).

| Lane | Renders this text? | Byte attribution (composed where → emitted how) | Decided-row reason (lanes Kimi can't reach) |
|------|--------------------|-----------------------------------------------|---------------------------------------------|
| **Kimi UserPromptSubmit hard deny** | **Yes** | Composed in `handleKimiPromptSubmit` (blockPrompts=true) as `BuildBlockMessage(kimiPromptBlockMessage, …)` → `emitKimiHardDeny(kimi.UserPromptSubmitEvent, reason)` writes BOTH: `hookSpecificOutput.permissionDecision:"deny"` + `permissionDecisionReason` JSON on **stdout**, AND the reason on **stderr** with process **exit 2**. Kimi TUI surfaces the stderr reason to the user. | — |
| **Kimi PreToolUse tool deny** | **Yes** | Composed in `handleKimiToolExecution` as `BuildBlockMessage(kimiToolBlockMessage, …)` → same `emitKimiHardDeny(kimi.PreToolUseEvent, reason)` pair (permissionDecision:"deny" doc on stdout + stderr reason + exit 2). One catch-all PreToolUse entry covers built-in shell + every MCP tool. | — |
| **Kimi PostToolUse additionalContext notice** | **Yes** | Composed in `handleKimiPostToolExecution` as `BuildBlockMessage(kimiPostToolBlockMessage, …)` → emitted as `hookSpecificOutput.additionalContext` JSON on **stdout only** (no permissionDecision, no stderr, exit 0). Audit/observe: the tool already ran; the string is fed to the MODEL as context, not shown as a deny. | — |
| **Kimi soft / monitor prompt mode** | **No (renders nothing)** | `handleKimiPromptSubmit` with blockPrompts=false `return nil` — emits NO stdout and NO stderr. Kimi's `UserPromptSubmit` runner acts only on a `deny` permissionDecision and ignores `additionalContext`/plain stdout on that event, so a soft prompt notice would be a discarded no-op; the guard verdict is persisted server-side by the `/guard/evaluate` call instead. | — (in-lane decision: deliberately silent) |
| **Gateway (MCP-guard-hooks) lane** | **No** | Not composed or emitted here — Kimi ships no gateway handler. | `links.py:155-157`: `# Kimi CLI reaches Onyx through the agent-hooks lane only (discovery + runtime block); it has no base-url, MCP-guard-hooks, or MCP-proxy lane.` `"endpoint_kimi-cli": (ProfileKey.KIMI_CLI_HOOKS,)` — no `MCP_GUARD_HOOKS_*` profile. |
| **Proxy (MCP-proxy) lane** | **No** | Not composed or emitted here — Kimi ships no proxy handler. | Same `links.py` comment: "…no base-url, MCP-guard-hooks, or MCP-proxy lane." `endpoint_kimi-cli` binds only `ProfileKey.KIMI_CLI_HOOKS`. |
| **Browser extension** | **No** | Not composed or emitted here — the extension is a browser surface with no Kimi CLI reach. | Hooks-only per the same `links.py` decision (single `KIMI_CLI_HOOKS` profile; no browser/base-url lane). Kimi CLI is a terminal agent reached solely via the agent-hooks lane. |

**Render-matrix lane count:** **7 lanes** (3 that render text, 1 in-lane silent, 3 decided/unreachable).
