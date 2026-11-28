# Installation guide

[中文](installation.zh.md) · [Usage guide](usage.md)

## 1. System requirements

Prepare the project's build, dependency resolution and runtime environment, and verify its existing build and test commands work. The release binary embeds its runtime (no separate Node.js); the npm/source channel requires **Node.js 22.x (at least 22.19) or 24.11+**. Also prepare any required databases, services, devices or emulators. Basic tools:

| Project under test | Needs |
|---|---|
| Any | `git` (`scan` also uses `rg`, bundled in release archives) |
| C / C++ | a C++17 compiler and the project's own build tools; CMake projects need `cmake` ≥ 3.16; RapidCheck source is bundled for offline use |
| Python | `python3` ≥ 3.9 and `pip` |
| Go | `go` |
| Java / Kotlin | a JDK and the project's own Maven / Gradle |
| HarmonyOS apps | DevEco tools, matching SDK, `ohpm`, `hdc`, Bash (Git Bash on Windows), `unzip` or ZIP-capable `tar`, and a device or emulator; see the [ArkTS guide](pbt/arkts.md) |
| OpenHarmony components | a configured OpenHarmony source tree |

## 2. Download and install

| Platform | File |
|---|---|
| Linux x64 | `fmbot-pbt-linux-x64.tar.xz` |
| Linux arm64 | `fmbot-pbt-linux-arm64.tar.xz` |
| macOS Apple Silicon | `fmbot-pbt-macos-arm64.tar.xz` |
| Windows x64 (experimental) | `fmbot-pbt-windows-x64.tar.xz` |

Download the matching `.sha256` file. Verify it with `sha256sum -c <checksum-file>` on Linux or `shasum -a 256 -c <checksum-file>` on macOS. If verification succeeds:

```bash
PLATFORM=linux-x64        # or linux-arm64, macos-arm64
tar -xJf "fmbot-pbt-${PLATFORM}.tar.xz" && cd "fmbot-pbt-${PLATFORM}" || exit 1
if [ "$(uname -s)" = Darwin ]; then xattr -dr com.apple.quarantine .; fi
./install.sh
```

`install.sh` also links the previous command name `pi-pbt` to `fmbot-pbt`, and places `native/` next to the installed executable for interactive clipboard support.

Windows: compare `Get-FileHash .\fmbot-pbt-windows-x64.tar.xz -Algorithm SHA256` with the checksum file. If it matches, extract with 7-Zip or the Windows 11 file explorer, add the extracted directory to `PATH`, and run `fmbot-pbt.exe` (`pi-pbt.cmd` forwards the previous command name to it). C/C++ and OpenHarmony projects are not supported on Windows.

## 3. Configure a model

In an interactive terminal with no configured model, `fmbot-pbt` opens a setup wizard automatically, or run `fmbot-pbt setup` anytime (in a TUI session, `/setup` opens the bundled `pi-setup` dialog). If you already have a working Claude Code or codeagent, you can skip model setup and [run with it over ACP](acp.md).

To configure `~/.pi-pbt/agent/models.json` manually (for example GLM-5.3 over `/v1/chat/completions`, merged with any existing file):

```json
{
  "providers": {
    "my-provider": {
      "baseUrl": "https://gateway.example.com/v1",
      "api": "openai-completions",
      "apiKey": "YOUR_GATEWAY_KEY",
      "compat": {"supportsStore": false, "supportsDeveloperRole": false, "supportsReasoningEffort": true,
        "maxTokensField": "max_tokens", "thinkingFormat": "openai"},
      "models": [{
        "id": "glm-5.3", "name": "glm-5.3",
        "reasoning": true, "contextWindow": 1048576, "maxTokens": 16384
      }]
    }
  }
}
```

Replace `baseUrl`, `apiKey` and `id` with your gateway's values. `api: "openai-completions"` expects `/v1/chat/completions` (`baseUrl` ends at `/v1`); use `openai-responses` for `/v1/responses` or `anthropic-messages` for `/v1/messages`.

`reasoning: true` and `compat` match the gateway's reasoning parameters; add `"input": ["text", "image"]` only when both model and route accept images. `contextWindow` sets the compaction limit (see [model documentation](https://huggingface.co/zai-org/GLM-5.3)), and `maxTokens` sets the per-request output budget.

Set defaults in `~/.pi-pbt/agent/settings.json` (merge with any existing file):

```json
{
  "defaultProvider": "my-provider",
  "defaultModel": "glm-5.3",
  "defaultThinkingLevel": "high"
}
```

Scan and Plan run at `xhigh` thinking (falling back to `max`, then `high`) when mapped in `thinkingLevelMap`; see [campaign phase](subcommands.md#campaign-phase) and [`PBT_DESIGN_THINKING`](subcommands.md#environment-variables). Other fields: [model configuration reference](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/models-configuration.md) and [provider reference](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/providers.md).

HTTPS uses system and bundled roots, with automatic [enterprise certificate compatibility](network.md) for Huawei and official dependencies.

## 4. Verify

Run `command -v fmbot-pbt` (PowerShell: `Get-Command fmbot-pbt`), then `fmbot-pbt --version`. `fmbot-pbt --list-models` shows loaded models; it does not verify API credentials.

First test: from the project directory, run `fmbot-pbt build-run --target <module-dir> --effort standard --build-cmd "<build command>"` using a module path and working build command, then read `pbt-out/REPORT.md`.
Interactive testing and CI gates: [usage guide](usage.md). HarmonyOS apps: complete [ArkTS preparation](pbt/arkts.md#before-your-first-campaign) first.

## 5. Upgrade

Download the new archive and install it as in section 2. Your configuration in `~/.pi-pbt/agent/` is kept.

If you installed the `pi-pbt-dev` skill for a coding agent, refresh it the way you installed it:

```bash
fmbot-pbt skill-install --force                        # agents detected automatically
fmbot-pbt skill-install --host project --force         # installed in a project directory
fmbot-pbt skill-install --dir /custom/skills --force   # installed in a custom directory
```

## 6. Troubleshooting

| Symptom | What to do |
|---|---|
| `tar` does not support xz | install `xz` and retry |
| `fmbot-pbt --list-models` prints nothing | run `fmbot-pbt setup` or configure a model as in section 3 |
| A run was interrupted | run `fmbot-pbt resume`, or `fmbot-pbt finalize --out <dir>` when evidence is complete; check `pbt-out/interruption.json` and `pbt-out/recovery.json` ([details](subcommands.md#resume-and-offline-completion)) |
| HarmonyOS SDK, HAP build or device error | follow the [ArkTS preparation checklist](pbt/arkts.md#before-your-first-campaign) and [ArkTS device guide](arkts-device.md) |
| Anything else | open an issue with `fmbot-pbt --version`, the OS and `~/.pi-pbt/agent/pi-pbt-debug.log` |
