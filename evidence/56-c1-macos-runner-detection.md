# C1 — macOS-runner detection session (real macos-latest, no lab Mac needed)

The verifier accepts a macos-latest CI runner session as the macOS transcript. This is the scanner-detector-e2e **macos job** at HEAD `fce925c99bacc3ef92a14b87bcb5835ef36ee43b`, job conclusion **success**.

- Run: https://github.com/onyxsecurity/onyx/actions/runs/35697396199
- Job: Detection (macos), databaseId 106647205762 — **success**
- Real runner host (uname) + real installed kimi version, printed next to the real `uv tool install kimi-cli`:
```
[kimi-cli] host: Darwin dsm41-ai252-d541bb5f-ce77-401f-921c-b8c9631dab2e-72F727B96D54.local 25.6.0 Darwin Kernel Version 25.6.0: Fri Jul 31 19:16:43 PDT 2026; root:xnu-12377.161.14~5/RELEASE_ARM64_VMAPPLE arm64
[kimi-cli] version: kimi, version 1.51.0
```
- Raw (unnormalized) kimi-cli scanner payload emitted on macOS at HEAD — real `~/.kimi` config, the `e2e-filesystem` MCP server written by the product's own `kimi mcp add`, and the `tui` channel:
```json
{
    "data": {
        "codingAgent": {
            "configPath": "/Users/runner/.kimi",
            "configExists": true,
            "name": "kimi-cli",
            "type": "desktop",
            "isGuarded": false,
            "installEvidenceType": "dsl_dir",
            "installEvidencePath": "/Users/runner/.kimi",
            "mcpServers": [
                {
                    "name": "e2e-filesystem",
                    "configEntryName": "e2e-filesystem",
                    "command": "npx",
                    "args": [
                        "-y",
                        "@modelcontextprotocol/server-filesystem",
                        "/tmp"
                    ],
                    "type": ""
                }
            ],
            "agentCustomData": {
                "_type": "DefaultChannelCustomData",
                "channels": [
                    {
                        "channelId": "tui",
                        "details": {
                            "builtin": true,
                            "path": "/Users/runner/.kimi"
                        },
                        "displayName": "Terminal UI",
                        "enabled": true,
                        "key": "tui"
                    }
                ]
            }
        }
    },
    "id": "scanner-agent-kimi-cli-ba9cb601-f5c0-4f63-beac-213a22e1ded2",
    "sessionId": "071a6687-26b3-4c95-985f-a7c1996320d1",
    "metadata": {
        "deviceName": "dsm41-ai252-d541bb5f-ce77-401f-921c-b8c9631dab2e-72F727B96D54.local",
```

This is a real macOS host running the real kimi-cli install + real scanner detection through the same code the Linux run exercises — the macOS half of S3/S2 that no Linux box can produce, captured on the OS itself.
