# S3 — Kimi hook command chain (Linux) + executable-resolution hop map

**Rule S3** — board asks: (1) transcript of the Kimi hook command chain run on Linux; (2) same chain on macOS; (3) hop map — how each executable is resolved.

**Branch:** `factory-eval-v32/PRDCT-14240`. Stamped 2026-09-20. Host `Linux 6.8.0-1063-aws x86_64`.

---

## The hop map — how each executable is resolved (no PATH ambiguity)

```
Kimi CLI  ── reads ~/.kimi/config.toml [[hooks]] ──▶  command = '"/home/coder/.kimi/onyx/onyx-scanner" hooks --source=kimi_cli --type={prompt|tool|post_tool}'
   │                                                    (ABSOLUTE path — no $PATH lookup)
   ▼
onyx-scanner sidecar  ── resolves config RELATIVE to its own dir ──▶  ~/.kimi/onyx/onyx-config.json  (api-key, endpoint, timeout)
   │
   ▼
AI-firewall  POST /guard/evaluate/v1/<key>/kimi_cli/<ctx>   (endpoint from onyx-config.json)
```

| hop | resolver | mechanism (code) | live proof |
|---|---|---|---|
| Kimi → hook binary | **absolute path**, not `$PATH` | installer writes the absolute sidecar path into each `[[hooks]].command` (`~/.kimi/config.toml` — 3 hooks captured below) | `which onyx-scanner` → **NOT on PATH**; hooks still resolve because the command is absolute |
| running exe → sidecar copy | `os.Executable()` → `atomicInstallExecutable` → `filepath.Join(onyxDir, "onyx-scanner"[+".exe" on windows])` | `installScannerBinary` (`scanner/internal/installer/kimi_hooks_installer.go:414-431`) | `~/.kimi/onyx/onyx-scanner` present, 39,705,274 bytes, mode 0755 (real `ls`) |
| sidecar → its config | read `onyx-config.json` from the **binary's own dir** (a binary run from `/tmp` finds no config and no-ops — see ev49) | `writeOnyxConfigFile(onyxDir, …)` (`kimi_hooks_installer.go:435`) | `~/.kimi/onyx/onyx-config.json` present, mode 0600 (real `ls`) |
| api-key storage | OS keychain, else plaintext sidecar fallback | keychain put; `includeRoot=false` (`kimi_hooks_installer.go:178-179`) | live install line: `keychain_outcome … outcome:"fallback_env", reason:"keychain_unavailable" … "dbus-launch": executable file not found` → key written to the 0600 sidecar |

The three hook commands, verbatim from the installed `~/.kimi/config.toml`:
```toml
[[hooks]]
command = '"/home/coder/.kimi/onyx/onyx-scanner" hooks --source=kimi_cli --type=prompt'
event   = 'UserPromptSubmit'
timeout = 30
[[hooks]]
command = '"/home/coder/.kimi/onyx/onyx-scanner" hooks --source=kimi_cli --type=tool'
event   = 'PreToolUse'
timeout = 30
[[hooks]]
command = '"/home/coder/.kimi/onyx/onyx-scanner" hooks --source=kimi_cli --type=post_tool'
event   = 'PostToolUse'
timeout = 30
```

## The Linux hook command chain — run for real
The exact chain each event fires (`onyx-scanner hooks --source=kimi_cli --type=…`) was driven end-to-end against the real firewall in **evidence 49 (C7)**: 7 legs (prompt allow/deny, PreToolUse allow/destructive, PostToolUse audit, + codex controls), each with the real wire request/response, the process exit code (the native Kimi hook contract — 0 allow / 2 deny), and the persisted `usage_sessions`/`ai_firewall_alerts` row attributed to asset 95 (`endpoint_kimi-cli`). That matrix IS the Linux command-chain transcript; it is not duplicated here.

## macOS chain — bucket (b), precise why
The cross-platform macOS test box is **retired** (box `10.2.7.152`, eu-west-3 dev VPC, destroyed with the region — PRDCT-12163; no us-east-1 replacement — `scanner/CLAUDE.md` "Cross-platform test machines"), and this eval box is Linux/amd64. The macOS hop map is identical in code except the `.exe` suffix is NOT appended (`runtime.GOOS == "windows"` guard only) and the keychain path uses the macOS Keychain instead of the dbus fallback; the darwin embedded DSL + golden compare **did execute** on CI (scanner-detector-e2e run 35241515207, job "Detection (macos)", commit 7cfd87b032, green — see ev41/S2). A live macOS hook run is not producible on this box.
