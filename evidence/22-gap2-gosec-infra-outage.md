# Gap 2 — `gosec` CI check: self-hosted-runner infra outage (NOT a code finding)

## What CI reports
The `gosec` check on PR #13392 head `4dc0f12d8f3dd86ae2b787470d3056e321fc72e0` fails with a
runner-infrastructure error, not a security finding. Every attempt's failure annotation is
byte-identical:

> **"The self-hosted runner lost communication with the server. Verify the machine is running
> and has a healthy network connection. Anything in your workflow that terminates the runner
> process, starves it for CPU/Memory, or blocks its network access can cause this error."**
> — annotation path `.github`, line 1.

Zero `G###` rule findings. Zero `.go:` file annotations. The scan process was killed by the
runner losing contact before it produced any verdict.

## This is the failure mode the CI step itself documents
`.github/workflows/backend-pr.yaml` (the `gosec` job) carries an inline comment on the
`Run gosec` step:

> "-concurrency stays at 4 even though medium-arm has 8 CPUs. Raising it to 8 … killed the
> runner when it did not: long failures -- **step stuck, pod gone, empty log, at a round
> 10-12 min** … gosec holds package state per worker, so concurrency is a memory dial here …
> this pod does not have the headroom …"

Observed runtimes of the failing attempts (~11-12 min then runner death) match this exactly.

## Proven NOT to be this PR's code
- CI's `gosec` runs `working-directory: ./backend` over `./...` — it scans the **`backend`
  Go module only** (never `scanner/`).
- This PR's **sole** `backend`-module change is +9 lines in
  `backend/internal/issues/issues.go` (`git diff --stat origin/main..HEAD -- backend/`).
- Running the identical CI tool locally — `gosec v2.22.11 -exclude-generated` (installed
  `github.com/securego/gosec/v2/cmd/gosec@v2.22.11`, the exact CI pin) — on that package:
  ```
  Summary:  Files: 2   Lines: 237   Nosec: 0   Issues: 0
  gosec_pkg_exit=0
  ```
  → **0 issues, clean.**

## Attempts (all self-hosted-runner-comms failures, no findings) — permalinks
Run: https://github.com/onyxsecurity/onyx/actions/runs/34780800113
- attempt 1 — https://github.com/onyxsecurity/onyx/actions/runs/34780800113/job/103787384105 (fail, 12m1s, runner comms)
- attempt 2 — https://github.com/onyxsecurity/onyx/actions/runs/34780800113/job/103791116786 (fail, runner comms)
- attempt 3 — https://github.com/onyxsecurity/onyx/actions/runs/34780800113/job/103793224938 (fail, 21:09→21:20, runner comms)
- attempt 4 — https://github.com/onyxsecurity/onyx/actions/runs/34780800113/job/103795983779 (re-triggered this round)

## Disposition
Environmental CI wall on the shared self-hosted runner pool — tolerated in the same category as
a flaky `preview-env`. No branch change can green it while the pool starves the job. Recorded +
escalated in the run thread (see evidence 23). Code is proven gosec-clean.
