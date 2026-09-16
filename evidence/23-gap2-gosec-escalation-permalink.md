# Gap 2 — gosec infra-outage ESCALATION (run thread + owner DM)

Per the round-3 task ("re-trigger gosec green OR record the infra-outage log + an escalation
permalink in the run thread"), the gosec self-hosted-runner outage was escalated via the
run-events emitter, which posts to the run thread (@-tagging the ticket issuer) AND DMs the
owner (May) in one call. Both legs landed (`escalate_exit=0`, `thread_ok:true`, `dm_ok:true`).

- **Escalation kind:** `ci-infra-outage`
- **Sent (UTC):** 2026-09-13T21:42:01Z
- **Run-thread permalink:**
  https://onyx-security.slack.com/archives/C0BCW4JDZB2/p1789288548261129?thread_ts=1789288548.261129&cid=C0BCW4JDZB2
- **Owner-DM permalink:**
  https://onyx-security.slack.com/archives/D0B6V65HCEN/p1789335721415039
- **Failing gosec job permalink (infra outage, no findings):**
  https://github.com/onyxsecurity/onyx/actions/runs/34780800113/job/103795983779

Message content: the byte-identical "self-hosted runner lost communication" annotation across
all 4 attempts, zero findings, the job's own comment documenting the 10-12min runner-death,
the proof it is not this PR's code (backend-module-only scan; sole change +9 lines in
backend/internal/issues/issues.go; local gosec v2.22.11 on that package = 0 issues/exit 0),
and the ask (infra: give the pool headroom / lower concurrency+memory-limit the job; human:
treat gosec as an environmental red for THIS PR). Full detail: evidence/22-gap2-gosec-infra-outage.md.
