# Installation guide

[中文](installation.zh.md) · How to use it once installed: [usage guide](usage.md).

## 1. System requirements

Prepare the project's build, dependency resolution and runtime environment; verify its existing build and test commands work. The release binary embeds its runtime (no separate Node.js). The npm/source channel requires **Node.js 22.x (at least 22.19) or 24.11+**, matching the bundled syntax parser.
Also prepare any required databases, services, devices or emulators. Basic tools:

| Project under test | Needs |
|---|---|
| Any | `git` |
| C / C++ | a C++17 compiler and the project's own build tools; CMake projects need `cmake` ≥ 3.16; RapidCheck source is bundled for offline use |
| Python | `python3` ≥ 3.9 and `pip` |
| Go | `go` |
| Java / Kotlin | a JDK and the project's own Maven / Gradle |
| HarmonyOS apps | DevEco tools, a matching SDK and a connected device or emulator; see the [ArkTS guide](pbt/arkts.md) |
| OpenHarmony components | a configured OpenHarmony source tree |

## 2. Download and install
| Platform | File |
|---|---|
| Linux x64 | `fmbot-pbt-linux-x64.tar.xz` |
| Linux arm64 | `fmbot-pbt-linux-arm64.tar.xz` |
| macOS Apple Silicon | `fmbot-pbt-macos-arm64.tar.xz` |
| Windows x64 (experimental) | `fmbot-pbt-windows-x64.tar.xz` |

Each file comes with a `.sha256` checksum. Linux and macOS:

```bash
PLATFORM=linux-x64        # or linux-arm64, macos-arm64
sha256sum -c "fmbot-pbt-${PLATFORM}.tar.xz.sha256"
tar -xJf "fmbot-pbt-${PLATFORM}.tar.xz"
cd "fmbot-pbt-${PLATFORM}" && ./install.sh
```

`install.sh` also links the previous command name `pi-pbt` to `fmbot-pbt`. On macOS, run `xattr -d com.apple.quarantine fmbot-pbt` before the first start.

Windows: extract with 7-Zip or the Windows 11 file explorer, add the extracted directory to `PATH`, and run `fmbot-pbt.exe` (`pi-pbt.cmd` forwards the previous command name to it).
C/C++ and OpenHarmony projects are not supported on Windows.

## 3. Configure a model

For GLM-5.3 served through the standard `/v1/chat/completions` API, put this in `~/.pi-pbt/agent/models.json` (merge with any existing file):

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

Replace `baseUrl` and `apiKey` with your gateway URL and access key. Use its actual model ID; this route uses `glm-5.3`.
`api: "openai-completions"` means the `/v1/chat/completions` format; end `baseUrl` at `/v1`, without `/chat/completions`.
For `/v1/responses`, use `openai-responses`; for `/v1/messages`, use `anthropic-messages`, with the URL specified by the gateway.

`reasoning: true` declares reasoning support; `compat` uses `reasoning_effort` and `max_tokens`. Match these to the actual API. Omitting `input` means text only; add `"input": ["text", "image"]` only if both model and route support images.
`contextWindow: 1048576` means 1M context and controls compaction; lower it if the service has a smaller limit. Reasoning levels: [model documentation](https://huggingface.co/zai-org/GLM-5.3).
`maxTokens: 16384` is the output budget per request, not context capacity or the model's output limit; increase within the route's limit to allow for reasoning and answers.

Put this in `~/.pi-pbt/agent/settings.json` (merge with any existing file); future `fmbot-pbt` starts use this model and thinking level:

```json
{
  "defaultProvider": "my-provider",
  "defaultModel": "glm-5.3",
  "defaultThinkingLevel": "high"
}
```

Other fields: [model configuration reference](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/models-configuration.md).
Login and built-in provider keys: [provider reference](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/providers.md).

HTTPS uses system and bundled roots, with automatic [enterprise certificate compatibility](network.md) for Huawei and official dependencies.

## 4. Verify

Run `command -v fmbot-pbt`, `fmbot-pbt --version` and `fmbot-pbt --list-models` to check the executable and loaded models.

First test: in a `fmbot-pbt` session type `/skill:pbt-hook-run <sha>` or ask in plain language; unattended, run `fmbot-pbt hook-run <sha>`. Add `--target scope --scope <path>` to test a whole module (`.` for the repository) instead of the change. When the target needs the project build, add `--build-cmd "<command>"` (and `--build-workdir <dir>`) to either; the checkout must be at `<sha>`. Exit codes: `0` no bug, `1` bugs, `2` incomplete, `3` build failed. All flags: [reference](subcommands.md).
## 5. Upgrade
Download the new archive and install it as in section 2. The configuration in `~/.pi-pbt/agent/` is kept.

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
| `fmbot-pbt --list-models` prints nothing | no model is configured yet; see section 3 |
| A run was interrupted | `fmbot-pbt resume` continues it; the cause is in `pbt-out/recovery.json` and `pbt-out/interruption.json` |
| Anything else | open an issue with `fmbot-pbt --version`, the OS and `~/.pi-pbt/agent/pi-pbt-debug.log` |
