# S2 refresh — OS-gated kimi-cli detector re-run green at branch HEAD

The prior S2 citation (run 35241515207, commit 7cfd87b032) was staled when main was merged into the branch. This re-points S2 to a fresh detector-e2e run **at the current branch HEAD**.

- Run: https://github.com/onyxsecurity/onyx/actions/runs/35697396199
- HEAD: `fce925c99bacc3ef92a14b87bcb5835ef36ee43b` (branch `factory-eval-v32/PRDCT-14240`)
- Detection (linux) job 106647206105 — **success**
- Detection (macos) job 106647205762 — **success**
- Detection (windows) — kimi-cli `# os: linux,macos`; the Windows `.exe` branches are compiled-only and honestly NOT executed (no kimi install on the windows runner), unchanged from the prior S2 disposition.

## Golden byte-compare verdicts at HEAD (the OS gate actually executing)
```
linux:  [kimi-cli] version: kimi, version 1.51.0
Emitted normalized kimi-cli/linux payload -> goldens/golden-kimi-cli-linux.json
PASS: kimi-cli/linux payload matches golden

macos:  [kimi-cli] version: kimi, version 1.51.0  (Darwin 25.6.0 arm64)
Emitted normalized kimi-cli/macos payload -> goldens/golden-kimi-cli-macos.json
PASS: kimi-cli/macos payload matches golden
```

## Real per-OS install evidence printed next to the real `uv tool install kimi-cli`
```
[kimi-cli] host: Linux runnervmlun5p 6.17.0-1022-azure #22-Ubuntu SMP Mon Jul 27 17:24:03 UTC 2026 x86_64 x86_64 x86_64 GNU/Linux
[kimi-cli] version: kimi, version 1.51.0
[kimi-cli] host: Darwin dsm41-ai252-d541bb5f-ce77-401f-921c-b8c9631dab2e-72F727B96D54.local 25.6.0 Darwin Kernel Version 25.6.0: Fri Jul 31 19:16:43 PDT 2026; root:xnu-12377.161.14~5/RELEASE_ARM64_VMAPPLE arm64
[kimi-cli] version: kimi, version 1.51.0
```

Both OSes install the real kimi-cli 1.51.0, write `~/.kimi` via the product's own first-run + `kimi mcp add`, and the scanner's normalized payload byte-matches the committed golden — on the OS itself, at branch HEAD.
