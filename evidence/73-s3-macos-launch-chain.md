# S3 — kimi→onyx-scanner launch chain on macOS: hop map, OS-agnosticism, and the honest live-deny blocker

**Board rule S3** — the launch chain "where kimi spawns onyx-scanner" was shown on Linux
(`53-s3-hop-map-and-linux-chain.md`); the macOS leg was flagged as not shown (the macOS CI job is
detection-scan only). This file closes what is branch-closable — the **hop is OS-agnostic by
construction, proven at the code level with citations** — and records, honestly and specifically,
why a **live macOS deny-through-the-chain** run is an environment blocker rather than a faked pass.
All source citations below were re-read on this branch's HEAD; raw line-anchored dump in
`73-s3-macos-launch-chain-raw.txt` (same commit).

## The hop map (identical bytes on macOS and Linux)
```
kimi (parent process)
  └─ fires a hook on UserPromptSubmit / PreToolUse / PostToolUse   (kimi's own config.toml hooks array)
       └─ execs:  "<onyx-scanner>" hooks --source=kimi_cli --type={prompt|tool|post_tool}   (child process)
            └─ generic dispatcher → NewKimiHooksHandler()          (cmd/hooks.go:49)
                 └─ GenericEvaluateClient.Evaluate → POST {endpoint}/guard/evaluate/v1/{apiKey}/kimi_cli/{ctx}
                      └─ response.Action=="block"  → emitKimiHardDeny → permissionDecision:"deny" + exit 2
                         response.Action=="allow"  → nil (allow)
```

## Why the hop is OS-agnostic — code-level proof
The entire chain is written by **one POSIX installer + one generic dispatcher**, with **no per-OS
branch** in any of the spawn/command-line/verdict logic:

1. **In scope on macOS.** `scanner/test/detector-install/agents.json:39` — `kimi-cli` `"os": ["linux","macos"]`.
2. **Identical hook command lines.** `scanner/internal/agents/kimi/constants.go:12-14,27-29` — the three
   event names (`UserPromptSubmit`/`PreToolUse`/`PostToolUse`) and the three
   `hooks --source=kimi_cli --type=…` command lines are plain constants, no OS switch.
3. **Identical install mechanics.** `kimi_hooks_installer.go:386` (`hookCommand` quotes the command for
   every OS — Kimi is POSIX-only) and `:418` (`os.Executable()` copies the **same** running scanner
   binary into the `onyx/` sidecar beside `~/.kimi/config.toml`). No macOS/Linux fork.
4. **Identical runtime dispatch + deny render.** `cmd/hooks.go:49` registers the one
   `NewKimiHooksHandler()` for `SourceKimi`; the deny is rendered by
   `scanner/internal/scanner/agents/kimi_hooks.go:132-145` (`emitKimiHardDeny` → `permissionDecision:"deny"`
   on stdout **and** `ExitError{Code: 2}`). Neither has an OS branch.

**The SOLE per-OS divergence in the whole chain is credential storage**, and it is proven at the code
level: `secretstore/keyring_linux.go:20` `SystemKeychainSupported = false` vs
`keyring_darwin.go:32` `= true`; on install the key is routed to the OS keychain first and omitted
from the sidecar file on success (`kimi_hooks_installer.go:180-185`), else it stays in the sidecar
`onyx-config.json` at **`0o600`** in-home (`:457-462`); `system_keychain.go:65-119` does the
machine-scoped keychain write on macOS/Windows vs per-user libsecret on Linux. The hook reads the key
back with the **same precedence on both OSes** — `--api-key` → `ONYX_API_KEY` env → keychain → file
(`cmd/hooks.go:577,590,598`). So macOS differs only in *where the api-key rests*, never in *whether or
how kimi spawns onyx-scanner or how a verdict is rendered*.

## The deny verdict requires a LIVE guard endpoint — this is why a stub is not an option
`scanner/internal/client/evalute.go:184` — a block **only** ever comes from the server response body
(`response.Action=="block"`); **every** failure mode (nil request, unreachable endpoint, non-200,
unparseable body) **fails OPEN to allow** (`:144,158,167,180,252-255`). The one network-free block
(`hooks.go` `sanction.forceBlockReason`) is MCP server-identity ambiguity under `--mcp-guard-hooks`,
**not** a content/prompt deny. Therefore a genuine macOS *deny-through-the-chain* requires a
**reachable guard endpoint returning `{"action":"block"}` plus its api-key** — and a stub/mock host
returning "block" is an **automatic-fail shortcut** under the testing standard, so it is **not used**
here.

## What is proven live today, and on which OS
- **The deny verdict + exit-2 hard-block is proven LIVE on Linux**, end-to-end through the **real
  in-cluster AI-firewall** on this box — `54-c7-c8-recapture-head-*.md` (C7): real HEAD scanner binary
  → Kong `sanitization-eval` route → firewall running the branch parser → `{"action":"block"}`, hook
  **exit 2**, persisted alert. The deny rule used is a **deterministic no-model rule**, so this needs
  no Moonshot.
- **The deny render is unit-proven** (synthetic `Action:"block"` → `ExitError{Code:2}` + reason) in
  `kimi_hooks_output_test.go`, executed green in CI (C10).
- **The macOS-specific delta (credential storage)** is proven at the **code level** above.

## Honest blocker — a LIVE macOS deny-through-chain run (owner/env, NOT a closed live row)
The only macOS host available to this box is the **GitHub-hosted `macos-latest` runner**; there is no
interactive macOS machine on this workstation. Driving a real deny there is blocked by two things I
cannot satisfy honestly from this branch:
1. **No reachable real guard endpoint from GitHub CI.** The firewall I drove for C7 is this box's
   in-cluster Kong (localhost/port-forward) — unreachable from a GitHub runner. The real
   `ai-guard.onyx.security` would need a **tenant api-key with an enabled deny rule exposed as an org
   CI secret**, which is an org-admin action, not a branch change.
2. **A stub guard host is an automatic-fail shortcut** (above) — so I will not point the runner at a
   fake block server and present it as a real run.

The existing `macos-latest` job (`scanner-detector-e2e.yaml`) is **detection-scan only**, its
`E2E_USER_EMAIL/PASSWORD` `workflow_call` secrets are **declared but never referenced by any step**,
and its only network target is the **ingest** mirror (not `guard/evaluate`) — so there is no existing
auth path to a staging firewall to reuse. Adding a live macOS guarded-kimi job therefore depends on an
**owner-provisioned guard tenant + CI secret**; it is escalated to @maybo65, listed plainly, and is
**not** marked resolved or faked. The launch-chain hop itself is closed OS-agnostically at the code
level (above) and proven live on Linux (C7).

## Reachability
Line-anchored source dump: `73-s3-macos-launch-chain-raw.txt` (same commit). Linux live chain:
`53-s3-hop-map-and-linux-chain.md`, `54-c7-c8-recapture-head-f13cf08067.md`.
