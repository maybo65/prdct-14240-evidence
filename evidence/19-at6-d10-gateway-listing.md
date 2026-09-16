# AT6 / d10 — CARDINALITY, machine-proven from the gateway audit log

**PASS bar (test_plan at6):** *"CARDINALITY is proven by a LISTING, not the envelope — an
enumeration of ALL ask-smart-model/gateway calls made under this run's key over the run window
(gateway audit/usage log, quoted), showing exactly one enforcement-surface consult and naming
any other calls."*

## Source (authoritative, server-side — front-door SSO, not the box key)
- Surface: **Onyx Agent Gateway management console**, `https://gw.onyx.security` (customer alias
  `https://1lnhnxppl7.gw.onyx.security`), Observe → **Audit trail**.
- Auth: real Frontegg SSO as `coder-agent@onyx.security` (role `onyx.admin`, capabilities `*`) via
  `browser-sso-login` — the box's data-plane `oe-` virtual key cannot read `/mgmt` (401), so this
  is pulled through the console's own authenticated session, not the forbidden mgmt token on the
  data plane.
- API: `GET /mgmt/v1/audit/events` (per-request events; each carries `tags.{ticket,step,virtual_key}`).
- Run key / credential: `cred_01a05d5b24247e908275bd1fce773d89` (the shared `factory-runs` key; the
  **`ticket` tag** `PRDCT-14240` is the run discriminator, since the credential is shared across boxes).
- Window scanned: **2026-09-13T00:00:00Z → 2026-09-13T20:45:40Z** (the whole run window), complete
  coverage — fixed 10-minute forward slices on `kind=request.authorized` (exactly one tag-bearing
  event per authorized gateway request), capped slices subdivided; 135 pages fetched, page budget
  NOT hit, so the enumeration is exhaustive. Raw: `19-at6-d10-gateway-cardinality.json`.

## Enumeration of ALL gateway calls under this run's key, by step tag (distinct request_ids)
Total distinct `request_id`s tagged `ticket=PRDCT-14240` over the window: **1991**, broken down:

| `x-gw-tag-step` | distinct requests | what it is |
|---|---:|---|
| `engineer` | 1222 | Claude Code runtime inference (engineer phase) — NOT a consult |
| `setup` | 360 | Claude Code runtime inference (setup phase) — NOT a consult |
| `research` | 249 | Claude Code runtime inference (research phase) — NOT a consult |
| `orchestrator` | 118 | Claude Code runtime inference (orchestrator) — NOT a consult |
| `verify` | 40 | Claude Code runtime inference (verify phase) — NOT a consult |
| **`ask-smart-model:research`** | **1** | **the ONE enforcement-surface consult (d10)** |
| `ask-smart-model:issue` | 1 | a separate issue-diagnosis consult — NAMED, different topic |

`ask-smart-model` is the only step prefix that denotes a deliberate smart-model **consult** (the
`ask-smart-model` skill); every other step is the agent runtime's own inference traffic, not a consult.

## The two `ask-smart-model` calls (the only consults), enumerated

1. **Enforcement-surface consult — the d10 consult** ✓
   - `request_id`: `01a09a64596a7eb2abd09f0156aaf803`
   - `step`: `ask-smart-model:research`  ·  `ts`: **2026-09-13T10:51:02.890Z** (authorized) → usage.finalized 10:51:28Z
   - Full server-side lifecycle for this request (from `?request_id=...` lookup,
     `19-at6-d10-gateway-requestid-lookup.json`): `request.authorized` → `request.accepted` →
     `guardrail.evaluated` (post_request, allow, guard "runtime CC") → `guardrail.evaluated`
     (post_response, allow) → `usage.finalized` — all tagged `ticket=PRDCT-14240`,
     `step=ask-smart-model:research`, `virtual_key=cred_01a05d5b24247e908275bd1fce773d89`.
   - Topic (from the raw envelope, `astra-consult-envelope.json`): the enforcement-surface question —
     Option A (AI-Guard policy-BLOCK lane, Codex parity) vs Option B (extend sanctioning to
     desktop-agent platforms). Resolved model `gpt-6-astra` (Astra), HTTP 200.

2. **Other call, NAMED** (not an enforcement-surface consult)
   - `request_id`: `01a09b677a1871c3b34e929057a313d1`
   - `step`: `ask-smart-model:issue`  ·  `ts`: **2026-09-13T15:34:05.080Z**
   - `--context issue` = an error/issue-diagnosis consultation (per the `ask-smart-model` skill,
     the `issue` context is "paste a real error and ask for a diagnosis + fix"). Distinct topic,
     made ~4h43m AFTER the enforcement-surface consult and after coding began — it is NOT a second
     enforcement-surface design consult.

## Verdict against d10
- **Cardinality:** exactly **one enforcement-surface consult** (`ask-smart-model:research`,
  `01a09a64…`). The only other `ask-smart-model` call (`ask-smart-model:issue`, `01a09b67…`) is
  enumerated and named above; all remaining 1989 tagged requests are agent runtime inference, not
  consults. → satisfies *"exactly one enforcement-surface consult and naming any other calls."* ✓
- **Topic:** the request body of `01a09a64…` poses the enforcement-surface question. ✓ (envelope)
- **Timing:** 2026-09-13T10:51:02Z precedes the first non-doc source commit
  `ff2cdbc9…` @ 2026-09-13T11:41:35Z (`git log --reverse … main..HEAD -- scanner/ backend/
  backend_python/ frontend/`). ✓
