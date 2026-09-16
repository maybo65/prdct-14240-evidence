# Gap 2 — "Inventory verify (staging)" completed: FAILURE, environmental (shared-staging ingestion timing, NOT this PR's code)

**Disposition:** environmental CI wall on the shared staging-deploy/ingestion pipeline — tolerated in the same category as a flaky `preview-env`/`gosec` runner-infra outage. Not a Kimi-code defect. Escalated with a permalink in the run thread (evidence 25).

## What the check is
- Job **`Inventory verify (staging)`** in the **`Scanner Registry MDM E2E`** workflow (workflow_id `327817470`, `.github/workflows/scanner-registry-mdm-e2e.yaml`), `pull_request`-triggered on every factory-eval PR.
- It installs real Windows + macOS **registry-MDM** agents and polls the **shared STAGING inventory API** for each agent's asset to land.
- **Registry-MDM is a DIFFERENT product feature from Kimi CLI.** This PR touches no registry-MDM code, installer, or scanner-staging path.

## The re-run I drove to completion (per the round-3 task: "let staging-verify complete")
- Re-triggered run **`34780799936`** (`gh run rerun 34780799936 --failed`), head sha `4dc0f12d`.
- Started `2026-09-13T21:29:33Z`, completed **`2026-09-13T22:15:56Z`**, conclusion **`failure`** (duration 46 min).
- Job permalink: https://github.com/onyxsecurity/onyx/actions/runs/34780799936/job/103795981854
- Sub-jobs that PASSED in the same run: `Windows registry MDM (SYSTEM + UNC guard)` (success), `macOS registry MDM (root normalization)` (success).
- The ONLY failing step: **`Poll staging inventory for macOS registry-MDM assets`** (step 7). The Windows staging poll passed; only the macOS asset never confirmed.

## The failure is a staging-ingestion timeout, verbatim from the log
```
2026-09-13 22:11:53 [info ] poll  elapsed=2470.1  pending_owners=['oxmac-34780799936@ci.onyx.security'] pending_servers=[]
...
2026-09-13 22:15:45 [error] FAIL: assets not confirmed in staging inventory  elapsed=2702.1 \
    missing_owners=['oxmac-34780799936@ci.onyx.security'] missing_servers=[]
##[error]Process completed with exit code 1.
```
The macOS registry-MDM asset `oxmac-34780799936@ci.onyx.security` (owner keyed on the **run id**, freshly installed by this run) never appeared in the **shared staging inventory** within the ~45-min (2702s) poll window. `pending_exposure=None` throughout — the agent installed fine; the shared staging pipeline just did not ingest/expose the asset in time.

## Proof it is NOT a regression from this PR
1. **This exact branch passed this exact check TWICE earlier today**, both green:
   - run `34771967696` @ `2026-09-13T17:33:52Z`, sha `068706d4` — **success**
   - run `34770553200` @ `2026-09-13T17:05:53Z`, sha `7b77b254` — **success**
2. **Nothing registry-MDM-relevant changed since that last green run.** `git diff --name-only 068706d4 4dc0f12d` = 60 files, ALL from the `origin/main` merge commit that brought the branch up to date (frontend alerts UI, `generated/*openapi*`, macOS `.appiconset` PNGs). `grep -iE 'registry|mdm|installer|scanner-registry'` over that diff = **NONE**. The branch's own Kimi feature code is byte-identical to the 17:33 green run.
3. The poll owner is `oxmac-<run_id>` — a per-run synthetic asset with no dependence on PR code; its non-appearance is a shared-staging ingestion latency, not a code path this PR can change.

## Conclusion
`Inventory verify (staging)` was driven to completion as required; it terminated `failure` on the macOS shared-staging ingestion poll timeout — an environmental staging-pipeline wall unrelated to Kimi CLI, on a check this same branch was green on hours earlier. No branch change can green it while shared-staging macOS ingestion is slow; recorded + escalated (evidence 25) exactly like the gosec runner-infra outage (evidence 22/23).
