# Installation guide

[中文](installation.zh.md)

How to use it once installed: [usage guide](usage.md).

## 1. System requirements

Prepare the project's build, dependency resolution and runtime environment; verify its existing build and test commands work.
Also prepare any required databases, services, devices or emulators. Basic tools:

| Project under test | Needs |
|---|---|
| Any | `git` |
| C / C++ | a C++17 compiler and the project's own build tools; CMake projects need `cmake` ≥ 3.16 |
| Python | `python3` ≥ 3.9 and `pip` |
| Go | `go` |
| Java / Kotlin | a JDK and the project's own Maven / Gradle |
| HarmonyOS apps | DevEco tools, a matching SDK and a connected device or emulator; see the [ArkTS guide](pbt/arkts.md) |
| OpenHarmony components | a configured OpenHarmony source tree |

## 2. Download and install
| Platform | File |
|---|---|
| Linux x64 | `pi-pbt-linux-x64.tar.xz` |
| Linux arm64 | `pi-pbt-linux-arm64.tar.xz` |
| macOS Apple Silicon | `pi-pbt-macos-arm64.tar.xz` |
| Windows x64 (experimental) | `pi-pbt-windows-x64.tar.xz` |

Each file comes with a `.sha256` checksum. Linux and macOS:

```bash
PLATFORM=linux-x64        # or linux-arm64, macos-arm64
sha256sum -c "pi-pbt-${PLATFORM}.tar.xz.sha256"
tar -xJf "pi-pbt-${PLATFORM}.tar.xz"
cd "pi-pbt-${PLATFORM}" && ./install.sh
```

On macOS, run `xattr -d com.apple.quarantine pi-pbt` before the first start.

Windows: extract with 7-Zip or the Windows 11 file explorer, add the extracted directory to `PATH`, and run
`pi-pbt.exe`. C/C++ and OpenHarmony projects are not supported on Windows.

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

`reasoning: true` declares reasoning support; `compat` uses `reasoning_effort` and `max_tokens`. Match these to the actual API.
Omitting `input` means text only; add `"input": ["text", "image"]` only if both model and route support images.
`contextWindow: 1048576` means 1M context and controls compaction; lower it if the service has a smaller limit. Reasoning levels: [model documentation](https://huggingface.co/zai-org/GLM-5.3).
`maxTokens: 16384` is the output budget per request, not context capacity or the model's output limit; increase within the route's limit to allow for reasoning and answers.

Put this in `~/.pi-pbt/agent/settings.json` (merge with any existing file); future `pi-pbt` starts use this model and thinking level:

```json
{
  "defaultProvider": "my-provider",
  "defaultModel": "glm-5.3",
  "defaultThinkingLevel": "high"
}
```

Other fields: [model configuration reference](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/models-configuration.md).
Login and built-in provider keys: [provider reference](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/providers.md).

## 4. Verify

Run `command -v pi-pbt`, `pi-pbt --version` and `pi-pbt --list-models` to check the executable and loaded models.

## 5. Upgrade
Download the new archive and install it as in section 2. The configuration in `~/.pi-pbt/agent/` is kept.

If you installed the `pi-pbt-dev` skill for a coding agent, refresh it the way you installed it:

```bash
pi-pbt skill-install --force                        # agents detected automatically
pi-pbt skill-install --host project --force         # installed in a project directory
pi-pbt skill-install --dir /custom/skills --force   # installed in a custom directory
```

## 6. Troubleshooting

| Symptom | What to do |
|---|---|
| `tar` does not support xz | install `xz` and retry |
| `pi-pbt --list-models` prints nothing | no model is configured yet; see section 3 |
| A run was interrupted | `pi-pbt resume` continues it; the cause is in `pbt-out/recovery.json` and `pbt-out/interruption.json` |
| Anything else | open an issue with `pi-pbt --version`, the OS and `~/.pi-pbt/agent/pi-pbt-debug.log` |
