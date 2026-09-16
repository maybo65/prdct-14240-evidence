# Gap 2 — staging-verify environmental failure: escalation permalink

Escalated via `run-events-emitter.py escalate` (the accepted run-thread escalation path), kind `ci-infra-outage`, sent `2026-09-13T22:18:28Z`. Both legs landed (`thread_ok: true`, `dm_ok: true`, `issuer_tagged: true`).

- **Run-thread permalink (issuer @-tagged):** https://onyx-security.slack.com/archives/C0BCW4JDZB2/p1789288548261129?thread_ts=1789288548.261129&cid=C0BCW4JDZB2
- **Owner-DM permalink:** https://onyx-security.slack.com/archives/D0B6V65HCEN/p1789337908683529
- **Failing job permalink:** https://github.com/onyxsecurity/onyx/actions/runs/34780799936/job/103795981854
- **Details:** `evidence/24-gap2-staging-verify-environmental.md`

Content: recorded the `Inventory verify (staging)` completion (failure) as a shared-staging macOS registry-MDM ingestion timeout — a different feature from Kimi CLI, on a check this branch passed twice earlier today with no registry-MDM code changed since. Human merger should treat it like a flaky preview-env, not a code fix.
