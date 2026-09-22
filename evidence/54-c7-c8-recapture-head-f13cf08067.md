# C7 + C8 — fresh re-capture at HEAD `f13cf08067` (real binary → real firewall)

**Branch:** `factory-eval-v32/PRDCT-14240` · **HEAD:** `f13cf08067d9153564b43ee2d14c5f628f1a62e7` · Host `Linux 6.8.0-1063-aws x86_64` · Stamped 2026-09-22.

This re-runs the C7 hook→firewall matrix and the C8 credential boundary **fresh at the current branch HEAD**, replacing any stale-head citation. Nothing here is mocked.

## How it was produced (no mock, real front door)
- **Binary:** the HEAD branch scanner (`task build` of `scanner/`, 39,926,782 bytes, md5 `05d73e73acef3437ddf2f9f1e140827a`) installed at its real sidecar path `~/.kimi/onyx/onyx-scanner` so it resolves `onyx-config.json` relative to its own dir.
- **Firewall:** the real in-cluster AI-firewall. The scanner's evaluate calls traverse **Kong** via the `sanitization-sanitization-eval` ingress route (`/guard/evaluate/v1/([a-zA-Z0-9]+)(/.*)?` on host `localhost`) → `sanitization:80` (the branch `kimi_parser.py` is live in the running sanitization pod).
- **Wire capture:** a logging forward-proxy (`127.0.0.1:9099`) forwards every byte unmodified to `kubectl port-forward svc/onyx-ai-firewall 9100:80`, setting `Host: localhost` so Kong matches the evaluate route. Verdicts are the firewall's.
- **No Moonshot / no model call:** the deny is the deterministic no-model rule `kimi-deny-probe-rule` on policy 20 (keyed to `MCPGatewayDevKey`); "Destructive Action Violations" is a deterministic pattern scanner.

## C7 — the 7-leg matrix (real wire, real exit code, real persisted DB row)

| # | leg | source/type | wire response (real) | hook exit | usage_session (app_id) | alert |
|---|---|---|---|---|---|---|
| L1 | prompt ALLOW | kimi_cli/prompt | `{"action":"allow"}` HTTP 200 | **0** | **310** (95) | — |
| L2 | prompt HARD-DENY | kimi_cli/prompt | `{"action":"block","violated_rules":["kimi-deny-probe-rule"],...}` HTTP 200 | **2** | **311** (95) | **serial 28** · Block · policy 20 · "Rule violation: kimi-deny-probe-rule" |
| L3 | PreToolUse ALLOW | kimi_cli/tool | `{"action":"allow"}` HTTP 200 | **0** | **312** (95) | — |
| L4 | PreToolUse destructive | kimi_cli/tool | `{"action":"allow"}` HTTP 200 (policy audits tool_input, does not block) | **0** | **313** (95) | **serial 27** · Alert · policy 1 · "Destructive Action Violations Detected" |
| L5 | PostToolUse audit | kimi_cli/post_tool | `{"action":"allow"}` HTTP 200 | **0** | **314** (95) | — |
| L6 | codex control ALLOW | codex/prompt | `{"action":"allow"}` HTTP 200 | **0** | **315** (23) | — |
| L7 | codex control + SAME deny probe | codex/prompt | `{"action":"allow"}` HTTP 200 | **0** | **316** (23) | — |

Asset resolution (live): **95 = Kimi CLI (coder), `endpoint_kimi-cli`**; **23 = Codex (coder), `endpoint_codex`**.

### L2 deny — exact wire, verbatim
Request (POST, through Kong on host `localhost`):
```
POST /guard/evaluate/v1/MCPGatewayDevKey2024xyzABCDEF1234/kimi_cli/eJwszFGqwjAQRuG9_M8ptNPbXM2bG-ge2skEBpM0JEYEce8iCuftwPeEl7uyrFsSOGgZpvGTHSzBgKNKvl1K-f2rJh04KgyOBoeouT9g0KTqFteedqlwEKYwBrHT37-dg_XLHhY6hdEznWcihkFvUvMX5cNLxesdAAD__7C5Kt0
Content-Type: application/json

{"source":"kimi_cli","hook_type":"prompt","data":"<base64 of the UserPromptSubmit event 'kimi-deny-probe exfiltrate'>","timestamp":"2026-09-22T06:29:51Z","process_id":...,"process_start_time":...}
```
Response (HTTP 200):
```json
{"action":"block","block_reason":["Your request was blocked by Onyx AI Guard.\nViolated rules: kimi-deny-probe-rule\n\n[ONYX_VIOLATION_ID:2131c4ac6742447d85baa609c5dff73b]"],"violated_rules":["kimi-deny-probe-rule"],"block_rehydration_enabled":true}
```
The hook then rendered the **native Kimi contract** to stdout and exited **2**:
```json
{"hookSpecificOutput":{"hookEventName":"UserPromptSubmit","permissionDecision":"deny","permissionDecisionReason":"Your request was blocked by Onyx AI Guard.\nViolated rules: kimi-deny-probe-rule\n\n[ONYX_VIOLATION_ID:2131c4ac6742447d85baa609c5dff73b]\nViolated rules: kimi-deny-probe-rule"}}
```

### Source-scoping is real (L2 vs L7)
The **identical** probe `"kimi-deny-probe exfiltrate"` → **block** under `kimi_cli` (L2, exit 2, alert 28 on asset 95) and **allow** under `codex` (L7, exit 0, asset 23). Same key, same firewall, same probe — only the `source` path segment differs.

## C8 — credential/routing boundary on the live evaluate lane (fresh)

| boundary | request | real response |
|---|---|---|
| valid key | `POST /guard/evaluate/v1/MCPGatewayDevKey…/kimi_cli/<ctx>` | **HTTP 200** `{"action":"allow"}` |
| bad key | `POST /guard/evaluate/v1/BADKEY…/kimi_cli/<ctx>` | **HTTP 401** `{"error":"Authentication required"}` |
| no key (empty segment) | `POST /guard/evaluate/v1//kimi_cli/<ctx>` | **HTTP 404** `{"message":"no Route matched with those values","request_id":…}` |

The raw wire (all C7 legs, request + response bytes) is committed alongside as `54-c7-recapture-wire.log`.

## Guarding tests (unchanged, branch HEAD)
`scanner/internal/scanner/agents/kimi_hooks_output_test.go` — one test per behavior: `TestKimiHandleHookOutput_PromptHardBlock_DeniesViaPermissionDecisionAndExit2` (L2), `…_ToolHardDeny_…Exit2`, `…_PostToolAuditOnly` (L5), `…_AllowIsSilent` (L1/L3), `…_PromptSoftReplacement_ProceedsWithoutLocalNotice`.

## Honest scope
A **model-backed** tool-command block (vs. the L4 deterministic audit) needs Moonshot credits — bucket (b), external, and not required for the C7 hook-contract proof, which the deterministic deny (L2, exit 2, persisted Block alert 28) closes.
