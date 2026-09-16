# Astra consult (d10) — enforcement surface for endpoint_kimi-cli

- **When (server-side):** `Sun, 13 Sep 2026 10:51:28 GMT` (gateway `Date` header on the 200 response; raw envelope in `astra-consult-envelope.json`).
- **Channel:** Onyx Agent Gateway `/v1/messages`, model `gpt-6-astra` (Astra), step tag `ask-smart-model:research`, message id `chatcmpl-ENc5XlW88Ennl7jKDEVaiXEngo3q9`, HTTP 200.
- **Topic (request body):** which enforcement/admin surface carries the Kimi runtime block — Option A (AI-Guard BLOCK policy on /runtime-policies, Codex parity) vs Option B (extend asset sanctioning to desktop agents) — and how to model "sanctioned" as an axis distinct from "blocked" for the medium-severity "unless sanctioned" alert.
- **Cardinality:** this is the ONE enforcement-surface consult of the run (gateway-call listing to be attached in at6).

## Astra's recommendation (adopted)
1. **Runtime BLOCK → Option A (AI-Guard policy lane).** Kimi follows Codex's exact enforcement flow: PreToolUse → /guard/evaluate → matching agent/BLOCK policy → hook deny (`hookSpecificOutput.permissionDecision="deny"`). Preserves Codex parity (same policy type/action/admin surface) and keeps enforcement independent of approval.
2. **Sanctioning → an independent organizational-approval state**, NOT policy-derived. Do NOT infer approval from a BLOCK policy; do NOT create/delete policies when approval changes; do NOT add a kimi-only field or branch. Use the existing shared approval representation IF Codex carries it in the shared shape.
3. **Alert → revive the retired posture-issue `ISSUE_UNSANCTIONED_AGENT_DEPLOYED`** (applies_to DESKTOP_AGENT/AI_AGENT, severity 5.0 = Medium 4.0–5.9), keyed on the sanction state — a BLOCK policy change must NOT affect it. "Unless sanctioned" = only an explicit Sanctioned state suppresses the finding (Discovered is NOT implicitly approved).
4. **Critical gate:** if Codex genuinely lacks an independent approval representation in the shared shape (so "unless sanctioned" is unimplementable without new fields), that is a plan-level contradiction → resolve via a shared model change / ticket amendment (route back to research) rather than Option B or a kimi-only field.

Divergence from Astra will be recorded in the PR's "## Design Decisions".
