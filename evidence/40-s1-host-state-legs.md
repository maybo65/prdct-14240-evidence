# S1 — Prove each host-state behavior on a real host

**Rule:** S1 — Prove each host-state behavior on a real host (PR #13392, branch `factory-eval-v32/PRDCT-14240`).
**Host:** Ubuntu 22.04, hostname `ip-10-10-6-62`, user `coder` (uid 1001), kernel 6.8.0-1063-aws.
**Branch HEAD:** `16322b883c48e0a836b16eddea6a175442cfbd82`
**Captured:** 2026-09-20 (UTC).
**Scanner binary under test:** `/tmp/onyx-scanner-pr` (the real PR-HEAD binary).
**Method:** real scanner driven on the real host. Detection legs use `scan --verbose --full-scan=false --output-file <jsonl>` (writes JSONL detection payloads locally, no ingest). Install leg uses the real `kimi-hooks-install` command. No mock SystemProvider unit tests. Secrets redacted (sha256 comparison in place of key values).

Kimi CLI is DSL-detected (no Go detector struct); the kimi-cli DSL block in `internal/scanner/agents/embedded_dsl_linux.json` defines: `suspectedDir = binary_on_path_for_user("kimi")`; `configDirs`/`isGuarded`/`mcpConfigPaths` concat `get_env_variable("KIMI_SHARE_DIR")` with `homeDir/.kimi`; `isGuarded` uses `any` + `onyx_guard_hooks_present {format:kimiHooks, scope:user}`.

## Summary table

| Leg | Scenario | Detected? | evidence_type | isGuarded | Config dir hooks read from | Result |
|-----|----------|-----------|---------------|-----------|-----------------------------|--------|
| 1a | `KIMI_SHARE_DIR=/tmp/kimi-altshare` set (coder's own env) | yes (twice: alt + default) | `dsl_dir` | true (both) | `/tmp/kimi-altshare` **and** `/home/coder/.kimi` | PROVEN |
| 1b | `KIMI_SHARE_DIR` unset (control) | yes (once) | `dsl_dir` | true | `/home/coder/.kimi` only | PROVEN |
| 2 | Root sweep; root's `KIMI_SHARE_DIR=/tmp/kimi-rootshare` | coder yes, testkimi yes, root/bin/ubuntu no | `dsl_dir` | true (each) | coder→`/home/coder/.kimi`, testkimi→`/home/testkimi/.kimi` | PROVEN |
| 3 | `chmod 000 config.toml` (unreadable) | yes | `dsl_dir` | **false (safe default)** | `/home/coder/.kimi` | PROVEN |
| 4 | Installer, OS keychain unavailable | n/a (install) | n/a | n/a | sidecar `/home/coder/.kimi/onyx/onyx-config.json` | PROVEN |

All four legs proven on the real host with the real binary.

---

## Leg 1 — KIMI_SHARE_DIR redirect + unset control

**Claim:** when `KIMI_SHARE_DIR` points at an alternate dir, the asset is still detected there and `isGuarded` reads hooks from the alternate dir; with the env unset, only the default `~/.kimi` is used.

**Setup:** created `/tmp/kimi-altshare` with a guarded `config.toml` (Onyx `[[hooks]]`) + `mcp.json` (`altshare-marker-server`). Ran the scan with `KIMI_SHARE_DIR=/tmp/kimi-altshare` (1a), then unset (1b). Raw capture: `40a-leg1-share-dir-redirect.txt`.

**1a (env set) — raw agent_detection:**
```
agent_detection {"agent": "kimi-cli", "detected": true, "evidence_type": "dsl_dir", "evidence_path": "/tmp/kimi-altshare", "username": "coder"}
agent_detection {"agent": "kimi-cli", "detected": true, "evidence_type": "dsl_dir", "evidence_path": "/home/coder/.kimi", "username": "coder"}
```
Payloads: `configPath=/tmp/kimi-altshare, configExists=true, isGuarded=true` AND `configPath=/home/coder/.kimi, isGuarded=true`. The alt dir is both **detected** and **isGuarded=true**, i.e. `onyx_guard_hooks_present` read hooks from `/tmp/kimi-altshare`.

**1b (env unset, control) — raw agent_detection:**
```
agent_detection {"agent": "kimi-cli", "detected": true, "evidence_type": "dsl_dir", "evidence_path": "/home/coder/.kimi", "username": "coder"}
```
Only `/home/coder/.kimi` (`mcpServers=[e2e-filesystem]`). The alt dir is absent — the redirect is what added it.

**Code path:**
- Detection lane: `internal/scanner/agents/dsl_ops/ops_io.go:493` `GetProcessEnvVariableOp` (env read from the scanned user's own processes via `/proc/<pid>/environ`, cached per username — `buildProcessEnvMap`); the DSL concats that value with `homeDir/.kimi` for `configDirs`/`isGuarded`.
- `onyx_guard_hooks_present` → `internal/scanner/agents/dsl_capability_ops.go:927` `case guardFormatKimiHooks: return kimi.HooksFileGuards(fs, path)`, evaluated over each configDir (`any`).
- Installer/uninstaller lane counterpart: `internal/agents/kimi/paths.go:40-46` `UserConfigDir` — applies `KIMI_SHARE_DIR` only when the current process home equals the target home.

---

## Leg 2 — Root sweep; root's KIMI_SHARE_DIR must not collapse every user onto one path

**Claim:** a root scan of all users detects each user once at that user's OWN config dir; root's `KIMI_SHARE_DIR` (set elsewhere) does not leak onto other users.

**Setup:** created user `testkimi` with guarded `~/.kimi/config.toml` + `mcp.json` (`testkimi-marker-server`). Ran `/tmp/onyx-scanner-pr scan --verbose --full-scan=false` as **root** with root's `KIMI_SHARE_DIR=/tmp/kimi-rootshare` (a guarded trap dir). Raw capture: `40b-leg2-root-sweep.txt`.

**Raw per-user detection:**
```
agent_detection {"agent": "kimi-cli", "detected": true, "evidence_path": "/home/coder/.kimi",     "username": "coder"}
agent_detection {"agent": "kimi-cli", "detected": true, "evidence_path": "/home/testkimi/.kimi",  "username": "testkimi"}
Session substep skipped {"agent": "kimi-cli", "username": "bin",    "reason": "no_detected_agent"}
Session substep skipped {"agent": "kimi-cli", "username": "ubuntu", "reason": "no_detected_agent"}
Session substep skipped {"agent": "kimi-cli", "username": "root",   "reason": "no_detected_agent"}
```
Payloads: coder→`/home/coder/.kimi` (`e2e-filesystem`), testkimi→`/home/testkimi/.kimi` (`testkimi-marker-server`) — each detected exactly once at its OWN home, each `isGuarded=true`.

**The trap fired only for its owner (root):**
```
Detector scanned {"detector": "kimi-cli", "username": "root", "outcome": "not_installed",
  "paths_checked": ["/tmp/kimi-rootshare/mcp.json", "/root/.kimi/mcp.json"], ...}
```
`/tmp/kimi-rootshare` appears in the output **only** under `username: root` — because for root the process home equals the target home, so root's env legitimately applies to root's own resolution. It never appears under coder or testkimi. Root's env did NOT collapse other users onto one path.

**Code path:** the detection-lane divergence guard is `GetProcessEnvVariableOp`/`buildProcessEnvMap` (`internal/scanner/agents/dsl_ops/ops_io.go:493`, `:508`) — the `KIMI_SHARE_DIR` value used for a scanned user is read from THAT user's own `/proc` environ, not the (root) scanner process env. The installer-lane counterpart is the same invariant, expressed as `cur == homeDir` at `internal/agents/kimi/paths.go:42`.

---

## Leg 3 — Corrupt/unreadable config safe default

**Claim:** an unreadable/corrupt `config.toml` defaults to the SAFE side — reported as NOT guarded, never falsely "guarded".

**Setup:** `chmod 000 /home/coder/.kimi/config.toml`; scan; restore perms to `0600` after. Raw capture: `40c-leg3-corrupt-config-safe-default.txt`.

**Raw agent_detection + payload:**
```
agent_detection {"agent": "kimi-cli", "detected": true, "evidence_path": "/home/coder/.kimi", "username": "coder"}
payload: configPath=/home/coder/.kimi, configExists=true, isGuarded=false
```
The asset is still **detected** (dir exists) but `isGuarded=FALSE`.

**Direction and why it is the safe side:** the read error resolves to "not guarded", so the asset surfaces as **unprotected / needs attention**. The dangerous failure direction would be defaulting to "guarded" (claiming protection that cannot be verified) — this defaults the opposite way.

**Code path:** `internal/agents/kimi/guard.go:146` `HooksFileGuards`:
```go
data, err := afero.ReadFile(fs, path)
if err != nil {
    return false   // guard.go:152 — unreadable -> not guarded (safe default)
}
return HooksArrayGuards(ParseHooksArray(data))
```
`ParseHooksArray` likewise returns nil on parse error → `HooksArrayGuards(nil)` false. Reached from `dsl_capability_ops.go:927` (`guardFormatKimiHooks`). Perms confirmed restored to `-rw-------` (0600) after the run.

---

## Leg 4 — Installer with OS keychain unavailable → plaintext sidecar fallback

**Claim:** when the OS keychain is unavailable, the installer falls back to persisting the api-key in the plaintext sidecar `~/.kimi/onyx/onyx-config.json`.

**Setup:** headless box — `DBUS_SESSION_BUS_ADDRESS` unset, `/run/user/1001/bus` absent, `dbus-launch` not on PATH — so secret-service is naturally unreachable. Ran the real installer: `/tmp/onyx-scanner-pr kimi-hooks-install --force --api-key <REDACTED> --scan-endpoint … --scan-api-key … --endpoint … --verbose`. Exit 0. Raw capture: `40d-leg4-keychain-fallback-sidecar.txt`.

**Raw keychain outcome (scanner log):**
```
WARN keychain_outcome {"op": "put", "outcome": "fallback_env", "reason": "keychain_unavailable",
  "caller": "kimi_hooks", "error": "exec: \"dbus-launch\": executable file not found in $PATH"}
```

**Resulting sidecar `/home/coder/.kimi/onyx/onyx-config.json` (`-rw-------`, api-key redacted):**
```
"api-key": "<REDACTED len=33 sha256=a9f6430fbb2697277be97354be8b6ab634433dc3778d3463ff33fd9725c63264>"
"endpoint-url": "http://onyx-ai-firewall...:8080"
"ingest-endpoint": "http://ingest-onyx...:8080/mcp-scanner"
"block-prompts": "true"
"hooks-runtime-protection": true
```
The sidecar api-key sha256 equals the sha256 of the input `--api-key` (`a9f6430f…c63264`) — the key was written in plaintext to the sidecar.

**Code path:** `internal/installer/kimi_hooks_installer.go:180-188`:
```go
apiKeyForFile := options.APIKey
if options.APIKey != "" {
    if PutInstallSecretForAllOrCurrentUser(options.APIKey, s.fs, s.systemProvider, false, "kimi_hooks", NewCycleID(), s.logger) {
        apiKeyForFile = ""            // keychain success would clear it
    }
}
... s.writeOnyxConfigFile(onyxDir, apiKeyForFile, options)   // :188
```
Keychain put returned false (`keychain_unavailable`), so `apiKeyForFile` was kept and `writeOnyxConfigFile` persisted it into the sidecar. Sidecar struct field: `APIKey string json:"api-key,omitempty"` (`:70`).

---

## Host-state cleanup

Mutated host state restored after evidence capture:
- `config.toml` perms restored to `0600` (Leg 3).
- `testkimi` user removed (`userdel -r`); `/tmp/kimi-altshare`, `/tmp/kimi-rootshare` removed (Legs 1/2).
- The Leg 4 install re-guarded `~/.kimi/config.toml` (the box's pre-existing guarded state) and wrote the sidecar — left in place as it matches the host's normal installed state.
