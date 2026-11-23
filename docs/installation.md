# Installation guide

[中文](installation.zh.md) · [Usage guide](usage.md) · ArkTS: [Before your first campaign](pbt/arkts.md#before-your-first-campaign).

## 1. System requirements
Prepare the project's build, dependency resolution and runtime environment; verify its existing build and test commands work. The release binary embeds its runtime (no separate Node.js). The npm/source channel requires **Node.js 22.x (at least 22.19) or 24.11+**, matching the bundled syntax parser. Source builds verify the SHA-256 of the PandaCheck bytecode HAR included in the checkout. Packages and binaries embed that HAR, not the PandaCheck implementation source. There is no separate PandaCheck download for building fmbot-pbt or running campaigns. Also prepare any required databases, services, devices or emulators. Basic tools:

| Project under test | Needs |
|---|---|
| Any | `git` |
| C / C++ | a C++17 compiler and the project's own build tools; CMake projects need `cmake` ≥ 3.16; RapidCheck source is bundled for offline use |
| Python | `python3` ≥ 3.9 and `pip` |
| Go | `go` |
| Java / Kotlin | a JDK and the project's own Maven / Gradle |
| HarmonyOS apps | DevEco tools, matching SDK, `ohpm`, `hdc`, Bash (Git Bash on Windows), `unzip` or ZIP-capable `tar`, and device/emulator. HAR preparation uses SDK Node, or PATH Node.js 22+ for standalone SDKs; Python is not required. `build-run` tests in the original project by default; use `--isolate` for a single function in a mini HAP. Both source builds check the installed `--sdk` against the product's exact `compileSdkVersion` or load the project with Hvigor when unpinned. The build and resume set both `DEVECO_SDK_HOME` and `OHOS_BASE_SDK_HOME` to the verified SDK for the child process; nothing is installed. Versioned SDK directories (for example `26/ets` and `26.0.0/ets`) are checked for a matching manifest even if the default SDK differs. Without a pin or default, select a single installation. Check `<out>/arkts-preflight.json` on a stop. On relaunch, `changedSinceSetup` lists checkout changes separately from the original generated-file hashes. A campaign that has started keeps its verified product and SDK when resumed. Pre-signed runs skip compilation. See the [ArkTS guide](pbt/arkts.md). Listing `.ets` exports with [`scan --lang arkts`](subcommands.md#other-commands) needs no SDK or device. |
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

`install.sh` also links the previous command name `pi-pbt` to `fmbot-pbt`. On macOS, run `xattr -dr com.apple.quarantine .` in the extracted directory before `./install.sh`, so the executable, `tools/` and the clipboard addon under `native/` all load. The installer puts `native/` next to the installed executable: the interactive TUI loads its clipboard support from there.

Windows: extract with 7-Zip or the Windows 11 file explorer, add the extracted directory to `PATH`, and run `fmbot-pbt.exe` (`pi-pbt.cmd` forwards the previous command name to it).
C/C++ and OpenHarmony projects are not supported on Windows. PandaCheck installation finds the installed DevEco tools on PATH or under `DEVECO_HOME`/standard locations (including `Program Files` on Windows). The bundled bytecode requires API 26 or later on the SDK, device and consuming product (`compatibleSdkVersion`, not `targetSdkVersion`). Enable `buildOption.strictMode.useNormalizedOHMUrl: true` on the product or selected build mode. An OpenHarmony isolate with no inherited floor uses the HAR's minimum API; existing product settings are not raised. To copy the HAR without building or installing it, `make-pandacheck-har.sh --project <root> --archive-only` needs Bash and Node but not Hvigor or OHPM. This does not replace the user's SUT build command. For `--sign-profile`, `--result-source`, pre-signed `--hap`/`--test-hap`, the shared `PBT_ARKTS_DEVICE_OPTIONS` carrier and `HDC`/`JAVA` overrides, see [ArkTS signing and device verdicts](arkts-device.md). The native `arkts-sign`/`arkts-test` commands need no external Node. Device launch recovery defaults to `--ability-grace-ms 20000` (`0` disables); `--test-ability` selects the ability. `--idle-timeout-ms` bounds the wait without test progress (300000; `0` disables the idle cutoff). Source campaigns verify with `fmbot-pbt arkts-test --class <PbtSuite> --test-hap <current-signed-ohosTest.hap>` after rebuilding/signing the latest tests; the runner installs and fingerprints that exact HAP. Each reported test file needs reachable registration in `List.test.ets`, completed suite/case status records, and unchanged input fingerprints. Aggregate smoke passes, selection text and receipts predating an edit cannot attest PBT execution. Missing or stale evidence and overlapping checkout/output ownership exit 2. At launch, headless Pi ArkTS source campaigns can opt into [`--recover-missing-cases`](arkts-device.md#device-results), set the eligible attempt cap with `--recover-max-cases N` (default `3`), and authorize a rebuild after test edits with [`--case-recovery-recipe`](arkts-device.md#recovery-after-test-edits) on an output filesystem that supports hard links (such as NTFS on Windows). For manual recovery, copy the diagnostic's full `Suite#case` command line after the run ends. Keep `PBT_OUT_DIR` for the receipt and `PBT_SCOPE_FILE` for the scoped source when it is shown. See that guide for safe lock cleanup. Source builds need Bun as well as Node: after `npm ci --ignore-scripts`, both `npm run build:binary` and standalone `npm run build:binary:windows-x64` prepare all inputs and work in Windows cmd/PowerShell without a `script-shell` override. Git Bash is still needed for Bash campaign commands. If redirecting stdin in Git Bash, use `/dev/null`, not the `cmd.exe` device `NUL`. For new ArkTS test HAPs, the [bundled Hypium reporter](arkts-device.md#named-results-without-hilog) retains named outcomes even when report logs are dropped; generated isolates already include it.

## 3. Configure a model

If you already have a working Claude Code, you can skip model setup and [run with Claude Code](acp.md) using `build-run --engine acp` or `hook-run <sha> --engine acp`. In either command, `--out --engine=acp` names an output directory; it does not select an engine. For GLM-5.3 served through the standard `/v1/chat/completions` API, put this in `~/.pi-pbt/agent/models.json` (merge with any existing file):

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

Scan and Plan, where a campaign decides the contract and designs the properties, run at `xhigh` thinking, falling back to `max`, then `high`; the rest of the campaign uses the level above. pi treats `xhigh` and `max` as supported only when the model's `thinkingLevelMap` maps them (e.g. `"thinkingLevelMap": {"xhigh": "xhigh"}`, with the value the API accepts). `PBT_DESIGN_THINKING` changes or turns this off; optional `PBT_PHASE_MODELS` routes models by the same [campaign phase](subcommands.md#campaign-phase) used for thinking and managed-run status. Other fields: [model configuration reference](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/models-configuration.md). Login and built-in provider keys: [provider reference](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/providers.md).

HTTPS uses system and bundled roots, with automatic [enterprise certificate compatibility](network.md) for Huawei and official dependencies.

## 4. Verify
Run `command -v fmbot-pbt`, `fmbot-pbt --version` and `fmbot-pbt --list-models` to check the executable, version and loaded models. Listing models does not verify API access. If a provider rejects credentials or denies model access, see [recovery steps](subcommands.md#resume-and-offline-completion).

First test: from the project directory, run `fmbot-pbt build-run --target <module-dir> --effort standard --build-cmd "<build command>"` using a module path and its working build command. Read `pbt-out/REPORT.md`. For headless runs, the [recovery flags](subcommands.md#build-run) can enable supervision from a terminal without redirection. Use a build command that doesn't need terminal input when supervised or when running on native Windows.
For interactive testing and scope selection, see the [usage guide](usage.md). For HarmonyOS apps, complete [ArkTS preparation](pbt/arkts.md#before-your-first-campaign) first.

## 5. Upgrade
Download the new archive and install it as in section 2. Your configuration in `~/.pi-pbt/agent/` is kept. Remove `--no-isolate` from older `build-run` commands to test in the original project. Both the CLI and the in-session skill return `2` for this argument before building.

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
| HAP metadata or SDK preparation fails | Metadata rejected by the HAP policy reader stops `build-run` preparation with exit 2 before a campaign is queued; duplicate-key errors name the file, key and both 1-based declaration locations. Resolve the ambiguity, never silently pick a value. For `sdk.dir` / `OHOS_BASE_SDK_HOME`, SDK or signing errors, follow the [ArkTS preparation checklist](pbt/arkts.md#before-your-first-campaign); `DEVECO_SDK_HOME` alone may not satisfy the plugin. For a nested Hvigor project, use a direct `hvigorw` command with the actual `--build-dir` and rebuild in that directory. |
| A run was interrupted | In a paused Pi `--tui` campaign, use [`/continue`](subcommands.md#resume-and-offline-completion) to continue in place, or `/quit` to leave the evidence saved. Read the first abort or incomplete-delivery hint in the terminal or `interruption.json` before starting another model/device round. It checks saved evidence read-only: after the active campaign ends, copy the full offline command, including its output directory and any `--lang` flag. If it lists blockers instead, address those first. See also `pbt-out/recovery.json`. Recording correction is bounded. If corrections remain outstanding and none are resolved for 5 minutes, the campaign stops with exit 2. The timer starts at the first refusal and restarts only when an accepted correction clears an outstanding item and others remain. Headless whole-repository runs without a change filter can [set aside an attributable finding](arkts-device.md#filing-failures-and-close-out) after its third refusal and continue unrelated tests, but delivery remains incomplete (exit 2). `rejected-records.jsonl` preserves the evidence. Raw device failures survive in `report.json.deviceFailures`/`findings-unrecorded.json`, not as invented product bugs. If the parent left no interruption record, `resume --out <out>` uses the pre-tool `campaign-state.json` checkpoint; empty class selections exit 2 with evidence. See the [device guide](arkts-device.md) for canonical per-run layout and timestamps. |
| Evidence complete, agent aborted | After the active campaign ends, use its original output directory with `fmbot-pbt finalize --out <dir>`, even if the report is in the checkout fallback. It validates recorded receipts, results and review, renders the report and completes the matching checkpoint without a model or device rerun. Matching interruptions are archived, not discarded. All owned output layouts contribute design-ledger obligations and retained failure observations (including recovery sidecars); a clean report in another layout cannot bypass them. Never kill fmbot-pbt/hdc or edit `runs.jsonl`; resume owns its checkout and exposes PIDs/lease paths; descendant tools/subagents inherit campaign context, not another lease request. The runner archives a 30-minute-old device lock only for an exited host and confirmed idle bundle. Offline `finalize` can record old abandoned missing receipts as aborted when no device lock or failure evidence exists; required verified execution remains mandatory. Other missing evidence stays exit 2. See [recovery details](subcommands.md#resume-and-offline-completion). |
| Repeated provider aborts | `resume --attempts <n>` can retry a confirmed provider stream abort in the same saved campaign. A local cancel, policy stop or abort of unknown origin does not auto-retry; error text alone does not confirm the source. `resume-progress.json` stops requests after three consecutive aborts with unchanged recorded work (the original abort plus two resumed aborts), across resume invocations. Inspect the cause; `--force` permits one explicit retry, not evidence deletion or a model switch. |
| Anything else | open an issue with `fmbot-pbt --version`, the OS and `~/.pi-pbt/agent/pi-pbt-debug.log` |
