# C14 — Copy inventory extension (install/uninstall CLI + capability-matrix strings)

**Stamp:** branch HEAD `fce925c99b` · PR #13392 · closes the *inventory-completeness* half of C14 that the re-audit flagged.
**For:** PR-body inclusion beside `44-c14-c16-copy-and-render-matrix.md` (which already inventories the nine tool `external_description`s + 32 parameter descriptions + the three reuse block sentences).

> This file adds every remaining user-facing string the re-audit found that `44-…` did not cover: the install/uninstall Cobra command copy + terminal summary lines, and the capability-matrix profile label + degradation-reason copy. Each is quoted verbatim from HEAD with `file:line`, classed Invented / Reuse, and located by where it renders.
>
> **Unchanged buckets (honest):** the OWNER sign-off on the invented copy is still May's (bucket a). The running-UI brand-icon (`kimi.svg`) screenshots are still blocked on the SSO lockout (bucket b) — this file does not claim them.

---

## C14-a — `scanner/cmd/kimi_hooks_install.go` (installer CLI copy)

| String (verbatim) | file:line | Invented / Reuse | Renders where |
|---|---|---|---|
| `kimi-hooks-install` | :29 (`Use`) | Invented | CLI usage line / `--help` |
| `Install Kimi CLI hooks for AI Guard evaluation` | :30 (`Short`) | Invented | `--help` synopsis, command list |
| `Install Kimi CLI hooks that integrate with the Onyx AI Guard.` (+ the 7-step "This command will:" body, the per-user `~/.kimi/config.toml` note, and the `--dry-run`/`--force` lines) | :31–:52 (`Long`) | Invented | `kimi-hooks-install --help` |
| `🔍 Dry run summary:` | :171 | Invented | terminal (dry-run) |
| `📊 Installation summary:` | :173 | Invented | terminal |
| `hooks_file: %s` / `file_existed: %v` / `hooks_would_be_added: %d` / `hooks_added: %d` / `backup_created: %s` | :176,:177,:181,:187,:192 | Invented (structured summary keys) | terminal |
| `  - UserPromptSubmit: Onyx AI Guard prompt evaluation` | :182,:188 | Invented | terminal |
| `  - PreToolUse (all tools): Onyx AI Guard tool-execution evaluation` | :183,:189 | Invented | terminal |
| `  - PostToolUse (all tools): Onyx AI Guard tool-result audit` | :184,:190 | Invented | terminal |
| `💡 Run without --dry-run to apply these changes.` | :185 | Invented | terminal (dry-run) |
| `✅ Kimi hooks installed successfully!` | :194 | Invented | terminal |
| `   Restart Kimi to apply changes.` | :195 | Invented | terminal |
| `⏭️  Skipped: %s` | :198 | Invented | terminal |
| `✅ Kimi hooks already installed - no changes needed` | :200 | Invented | terminal |
| `❌ Encountered errors during installation:` / `   error: %v` | :204,:207 | Invented | terminal (error) |

## C14-b — `scanner/cmd/kimi_hooks_uninstall.go` (uninstaller CLI copy)

| String (verbatim) | file:line | Invented / Reuse | Renders where |
|---|---|---|---|
| `kimi-hooks-uninstall` | :25 (`Use`) | Invented | CLI usage line / `--help` |
| `Uninstall Kimi CLI hooks for AI Guard` | :26 (`Short`) | Invented | `--help` synopsis |
| `Uninstall Kimi CLI hooks that integrate with the Onyx AI Guard.` (+ the 4-step body + `--dry-run` line) | `Long` | Invented | `kimi-hooks-uninstall --help` |
| `🔍 Dry run summary:` | :107 | Invented | terminal (dry-run) |
| `📊 Uninstallation summary:` | :109 | Invented | terminal |
| `hooks_file: %s` | :112 | Invented | terminal |
| `❌ Uninstallation did NOT fully complete:` / `   error: %v` | :116,:119 | Invented | terminal (error) |
| `Would remove Onyx Kimi hooks (UserPromptSubmit + PreToolUse + PostToolUse) and the onyx directory.` | :127 | Invented | terminal (dry-run) |
| `💡 Run without --dry-run to apply these changes.` | :128 | Invented | terminal (dry-run) |
| `✅ Kimi hooks uninstalled successfully!` | :130 | Invented | terminal |
| `   Onyx hooks removed; non-Onyx hooks and your Kimi config were preserved.` | :131 | Invented | terminal |
| `   Restart Kimi to apply changes.` | :132 | Invented | terminal |
| `✅ No Onyx Kimi hooks found - no changes needed` | :135 | Invented | terminal |

> The **rendered** install+uninstall terminal output for these is captured live in `52-c14-kimi-hooks-terminal.txt` (real `onyx-scanner kimi-hooks-install` / `-uninstall` runs). This table is the source→class map for that transcript.

## C14-c — `backend_python/src/common/capability_matrix/profiles/hooks.py` (Kimi profile copy)

| String (verbatim) | file:line | Invented / Reuse | Renders where |
|---|---|---|---|
| `Kimi CLI (agent hooks)` | :337 (`label=`) | Invented (profile display label) | capability-matrix answers / capability UI for the Kimi hooks profile |
| `ask_not_supported` | :385 (`reason=`) | Invented (degradation reason code) | capability degradation explanation |
| `kimi's PostToolUse fires after the tool already executed: it detects and reports but cannot withhold the result (agents/kimi_hooks.go:182-183)` | :391–:393 (`reason=`) | Invented (degradation reason) | capability degradation explanation (TOOL_RESULT BLOCK→ALERT) |
| `PostToolUse audits the tool result on the INPUT direction and no hook can rewrite a tool result in place (kimi_parser.py post_tool; evaluate.py:256-264)` | :400–:402 (`reason=`) | Invented (degradation reason) | capability degradation explanation (TOOL_RESULT MASK→ALERT) |

The profile-header comment at :329–:333 documents that the Kimi lane deliberately mirrors Codex's hooks lane "by construction; the citations point at Kimi's own files" — i.e. the *structure* is reuse, the *citations/labels* are Kimi-specific.

---

## Classification summary
- **Invented (Onyx-authored):** all of the above — installer/uninstaller CLI help + terminal summary copy, the capability-matrix profile label, and the three degradation-reason strings. None is lifted from Kimi; only the Kimi command names (`kimi-hooks-install` etc. reference Kimi's own `config.toml`/tool identifiers) and file citations point at Kimi.
- **Reuse:** the three end-user **block sentences** remain byte-identical reuse from `codex_hooks.go` via `BuildBlockMessage` — proven in `44-…`, not re-listed here.

**Still owner (bucket a):** sign-off that this invented copy is acceptable product wording → @maybo65.
**Still SSO-blocked (bucket b):** running-UI `kimi.svg` brand-icon screenshots — not claimed here.
