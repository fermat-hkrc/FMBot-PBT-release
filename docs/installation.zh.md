# 安装指南

[English](installation.md) · [使用指南](usage.zh.md) · ArkTS：[首次测试任务前的检查清单](pbt/arkts.zh.md#首次测试任务前的检查清单)。

## 1. 系统要求
先准备好项目本身的编译、依赖解析和运行环境，确认现有构建与测试命令可用，再运行 fmbot-pbt。发布的二进制已内置运行时，无须单独安装 Node.js；npm/源码渠道要求 **Node.js 22.x（至少 22.19）或 24.11+**，与内置语法解析器一致。源码构建会校验仓库内 PandaCheck 字节码 HAR 的 SHA-256。npm 包和二进制内置该 HAR，不包含 PandaCheck 实现源码；构建 fmbot-pbt 和运行测试任务都无需单独下载 PandaCheck。涉及数据库、服务、设备或模拟器的项目，也要先准备这些运行依赖。下表列出基本工具：

| 被测项目 | 需要 |
|---|---|
| 任何项目 | `git` |
| C / C++ | C++17 编译器、项目自己的构建工具；CMake 项目需 `cmake` ≥ 3.16；随包提供 RapidCheck 源码，无须另行下载 |
| Python | `python3` ≥ 3.9 与 `pip` |
| Go | `go` |
| Java / Kotlin | JDK 与项目自带的 Maven / Gradle |
| HarmonyOS 应用 | DevEco 工具、匹配 SDK、`ohpm`、`hdc`、Bash（Windows 用 Git Bash）、`unzip` 或支持 ZIP 的 `tar` 及设备/模拟器。HAR 准备使用 SDK 内置 Node，独立 SDK 可用 PATH 中的 Node.js 22+；不需要 Python。`build-run` 默认在原工程中测试；要在迷你 HAP 中测试单个函数，可加 `--isolate`。两种源码构建都会校验已安装的 `--sdk` 与产品的 `compileSdkVersion` 精确匹配；未声明版本时由 Hvigor 加载项目。构建和恢复时仅在子进程内将 `DEVECO_SDK_HOME` 和 `OHOS_BASE_SDK_HOME` 设为已验证的 SDK，不安装 SDK。即使默认 SDK 不匹配，也会检查 `26/ets`、`26.0.0/ets` 等版本目录中的清单；若未指定版本且没有默认 SDK，请选择单一安装目录。预检停止后见 `<out>/arkts-preflight.json`，修复环境后重新启动。再次启动时，`changedSinceSetup` 单独列出工作区改动，不覆盖预检生成文件的原始哈希。已启动的任务恢复时沿用预检通过的产品和 SDK。预签名包不编译。见 [ArkTS 指南](pbt/arkts.zh.md)。仅用 [`scan --lang arkts`](subcommands.zh.md#其他命令) 列出 `.ets` 的导出函数时，不需要 SDK 或设备。 |
| OpenHarmony 组件 | 已配置好的 OpenHarmony 源码树 |

## 2. 下载与安装
| 平台 | 文件 |
|---|---|
| Linux x64 | `fmbot-pbt-linux-x64.tar.xz` |
| Linux arm64 | `fmbot-pbt-linux-arm64.tar.xz` |
| macOS Apple Silicon | `fmbot-pbt-macos-arm64.tar.xz` |
| Windows x64（实验性） | `fmbot-pbt-windows-x64.tar.xz` |

每个文件都附带 `.sha256` 校验文件。Linux 和 macOS：

```bash
PLATFORM=linux-x64        # 或 linux-arm64、macos-arm64
sha256sum -c "fmbot-pbt-${PLATFORM}.tar.xz.sha256"
tar -xJf "fmbot-pbt-${PLATFORM}.tar.xz"
cd "fmbot-pbt-${PLATFORM}" && ./install.sh
```

`install.sh` 同时把旧命令名 `pi-pbt` 链接到 `fmbot-pbt`。macOS 上先在解压目录执行 `xattr -dr com.apple.quarantine .` 再运行 `./install.sh`，让可执行文件、`tools/` 和 `native/` 下的剪贴板模块都能加载。安装脚本会把 `native/` 放在安装后的可执行文件旁边，交互界面的复制粘贴依赖它。

Windows：用 7-Zip 或 Windows 11 文件资源管理器解压，把解出的目录加入 `PATH`，运行 `fmbot-pbt.exe`（旧命令名 `pi-pbt.cmd` 会转发给它）。
Windows 上不支持 C/C++ 与 OpenHarmony 项目。PandaCheck 安装脚本从 PATH 或 `DEVECO_HOME`/标准位置（包括 Windows 的 `Program Files`）查找 DevEco 工具。内置字节码要求 SDK、设备和使用方产品的 `compatibleSdkVersion` 均为 API 26 或更高版本；`targetSdkVersion` 不能代替兼容版本。产品或所选构建模式需启用 `buildOption.strictMode.useNormalizedOHMUrl: true`。OpenHarmony 隔离工程没有继承兼容版本时使用 HAR 的最低 API，不会调高已有产品的版本。若只需复制 HAR，运行 `make-pandacheck-har.sh --project <root> --archive-only`，只需 Bash 和 Node，无需 Hvigor 或 OHPM。用户的被测项目构建命令不会被替换。`--sign-profile`、`--result-source`、预签名 `--hap`/`--test-hap`、共用的 `PBT_ARKTS_DEVICE_OPTIONS` 配置及 `HDC`/`JAVA` 覆盖方式见 [ArkTS 签名与设备判定](arkts-device.zh.md)。原生 `arkts-sign`/`arkts-test` 命令不需要外部 Node。设备启动恢复默认为 `--ability-grace-ms 20000`（`0` 禁用），`--test-ability` 选择 Ability。`--idle-timeout-ms` 限制无测试进展时的等待时间（300000，`0` 禁用此上限）。源码测试任务在重新构建、签名最新测试后，用 `fmbot-pbt arkts-test --class <PbtSuite> --test-hap <current-signed-ohosTest.hap>` 验证；运行器安装并记录该 HAP 的指纹。报告中的每个测试文件都须在 `List.test.ets` 中有可达注册，且回执包含对应套件、用例的完成状态及未改变的输入指纹。无关冒烟测试的汇总通过、选择参数或修改前的旧回执均不能证明性质已执行。缺少或过期的证据、工作区或输出目录被其他任务占用时退出 2。无界面 Pi ArkTS 源码测试可在启动时加 [`--recover-missing-cases`](arkts-device.zh.md#设备结果) 允许自动补齐，用 `--recover-max-cases N` 调整符合条件的尝试次数上限（默认 `3`），也可另传 [`--case-recovery-recipe`](arkts-device.zh.md#测试编辑后的恢复) 以授权在测试编辑后重建；此时输出目录需位于支持硬链接的文件系统中（如 Windows 的 NTFS）。手动补齐时，先等正在执行的调用结束，再复制诊断给出的完整 `Suite#case` 命令行。保留 `PBT_OUT_DIR` 以将回执写入原任务；若命令中有 `PBT_SCOPE_FILE`，也应保留，以便为指定源码计算指纹。安全清理方式见该指南。源码构建还需要 Bun：执行 `npm ci --ignore-scripts` 后，`npm run build:binary` 和独立的 `npm run build:binary:windows-x64` 都会准备全部输入，可直接在 Windows cmd/PowerShell 中使用，无须覆盖 `script-shell`。测试任务的 Bash 命令仍需要 Git Bash。在 Git Bash 中如需重定向标准输入，请用 `/dev/null`，不要使用 `cmd.exe` 的设备名 `NUL`。构建新的 ArkTS 测试 HAP 时，可使用[随包 Hypium 报告器](arkts-device.zh.md#不依赖-hilog-的具名结果)，在日志丢失时仍保留具名结果；生成的隔离项目已包含报告器。

## 3. 配置模型

已有能用的 Claude Code 时，可以跳过模型配置，通过 `build-run --engine acp` 或 `hook-run <sha> --engine acp` [用 Claude Code 运行](acp.zh.md)。这两个命令中的 `--out --engine=acp` 都是指定输出目录，不会切换引擎。以提供标准 `/v1/chat/completions` 接口的 GLM-5.3 服务为例，将下面配置写入 `~/.pi-pbt/agent/models.json`（已有文件时合并）：

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

把 `baseUrl` 换成你的网关地址，`apiKey` 换成网关访问密钥。`id` 用网关实际提供的 ID；本例为 `glm-5.3`。
`api: "openai-completions"` 表示 `/v1/chat/completions` 接口格式，`baseUrl` 填到 `/v1`，不要再追加 `/chat/completions`。
如果网关提供 `/v1/responses` 格式，用 `openai-responses`；提供 `/v1/messages` 格式，用 `anthropic-messages`，地址按渠道文档填写。

本例 `reasoning: true` 声明推理能力，`compat` 使用 `reasoning_effort` 和 `max_tokens`；换接口时按服务商要求调整。`input` 省略时默认纯文本；只有模型和渠道都支持图片时才加 `"input": ["text", "image"]`。
`contextWindow: 1048576` 表示 1M 上下文，影响自动压缩；服务商限制更小时填实际限制。推理等级参考[模型说明](https://huggingface.co/zai-org/GLM-5.3)。
`maxTokens: 16384` 是单次输出预算，不是上下文容量，也不是模型最大输出；可在渠道输出上限内调大，给推理和回答留足空间。

再将下面配置写入 `~/.pi-pbt/agent/settings.json`（已有文件时合并），以后直接运行 `fmbot-pbt` 即使用该模型和思考等级：

```json
{
  "defaultProvider": "my-provider",
  "defaultModel": "glm-5.3",
  "defaultThinkingLevel": "high"
}
```

扫描和设计性质阶段决定约定和性质，使用 `xhigh` 思考等级，模型不支持时依次用 `max`、`high`；测试任务其余阶段使用上面的等级。只有模型的 `thinkingLevelMap` 映射了 `xhigh` 或 `max`（如 `"thinkingLevelMap": {"xhigh": "xhigh"}`，值以接口实际接受的为准），pi 才认为模型支持这两个等级。用 `PBT_DESIGN_THINKING` 修改或关闭；可选的 `PBT_PHASE_MODELS` 按阶段切换模型，与思考等级和托管任务状态共用[阶段判定](subcommands.zh.md#任务阶段)。其他字段见[模型配置参考](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/models-configuration.md)；登录与内置服务商密钥配置见[服务商参考](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/providers.md)。

HTTPS 默认使用系统与内置证书，并对华为及官方依赖源启用[企业证书兼容](network.zh.md)，无需额外配置。

## 4. 验证
运行 `command -v fmbot-pbt`、`fmbot-pbt --version` 和 `fmbot-pbt --list-models`，检查程序、版本和已加载的模型。列出模型不代表 API 可访问。服务商拒绝凭据或模型访问时，见[恢复说明](subcommands.zh.md#恢复与离线收尾)。

第一次测试：在项目目录运行 `fmbot-pbt build-run --target <模块目录> --effort standard --build-cmd "<构建命令>"`，填入实际模块路径和已验证可用的构建命令。结果见 `pbt-out/REPORT.md`。无人值守运行可用[恢复参数](subcommands.zh.md#build-run)从终端启用监督，无需重定向。启用监督或使用 Windows 原生版时，请使用不需要读取终端输入的构建命令。
交互式测试和范围选择见[使用指南](usage.zh.md)。HarmonyOS 应用请先完成 [ArkTS 准备清单](pbt/arkts.zh.md#首次测试任务前的检查清单)。

## 5. 升级
下载新版本压缩包，按第 2 节重新安装。`~/.pi-pbt/agent/` 中的配置会保留。从旧版 `build-run` 命令中删除 `--no-isolate` 即可在原工程中测试。命令行和会话内 skill 遇到这个参数都会返回 `2`，不会启动构建。

如果装过供 coding agent 使用的 `pi-pbt-dev` skill，升级后按原来的安装方式刷新：

```bash
fmbot-pbt skill-install --force                        # 自动检测到的 agent
fmbot-pbt skill-install --host project --force         # 装在项目目录里的
fmbot-pbt skill-install --dir /custom/skills --force   # 装在自定义目录里的
```

## 6. 常见问题
| 现象 | 处理 |
|---|---|
| `tar` 提示不支持 xz | 安装 `xz` 后重试 |
| `fmbot-pbt --list-models` 没有输出 | 模型尚未配置，见第 3 节 |
| HAP 元数据或 SDK 准备失败 | HAP 策略读取器拒绝元数据时，`build-run` 在准备阶段退出 2，不启动测试任务；重复键错误指出文件、键名及两处声明位置（行、列从 1 开始）。请解决歧义，不得静默选择一个值。`sdk.dir` / `OHOS_BASE_SDK_HOME`、SDK 或签名问题见 [ArkTS 准备清单](pbt/arkts.zh.md#首次测试任务前的检查清单)；仅有 `DEVECO_SDK_HOME` 可能不满足工程插件要求。嵌套的 Hvigor 项目请用 `--build-dir` 指定实际构建目录，直接调用 `hvigorw`，并在同一目录重建。 |
| 测试任务中断 | Pi `--tui` 任务暂停后可输入 [`/continue`](subcommands.zh.md#恢复与离线收尾) 原地继续，也可输入 `/quit` 退出并保留证据。在再次调用模型或运行设备前，先看终端或 `interruption.json` 中首次中止或交付未完成的提示。提示只读检查已保存的证据：在活跃任务结束后复制完整的离线命令，保留输出目录及提示中的 `--lang` 参数；若提示列出缺项，则先处理缺项。另见 `pbt-out/recovery.json`。记录修正次数与时长有限。仍有待修正项且 5 分钟内未解决任何一项时，任务停止并退出 2。计时从首次拒绝开始，仅在一项修正通过且仍有其他待修正项时重新计时。无界面全仓库任务未指定改动范围时，同一问题的第 3 次拒绝可[暂停能明确归属的记录修正](arkts-device.zh.md#失败记录与收尾)并继续无关测试，但仍未完成交付（退出 2）。`rejected-records.jsonl` 保留证据；设备原始失败另存于 `report.json.deviceFailures`/`findings-unrecorded.json`，不会伪造产品缺陷。父进程没有留下中断记录时，`resume --out <out>` 使用工具执行前的 `campaign-state.json` 检查点；空套件退出 2 并保留证据。统一的单次运行目录和时间戳见[设备指南](arkts-device.zh.md)。 |
| 证据齐全但 agent 中止 | 先结束活跃任务，再用原任务的输出目录运行 `fmbot-pbt finalize --out <dir>`，即使报告在工作区的备用目录中也一样。该命令校验已记录的回执、结果与复核，渲染报告并完成匹配的检查点，不调用模型或重跑设备。匹配的中断被归档而非丢弃。所有所属输出布局的性质清单及已保留失败观察（含恢复侧文件）都参与校验，另一个布局中的干净报告不能绕过这些义务。不得杀死 fmbot-pbt/hdc 或编辑 `runs.jsonl`；恢复后的 runner 持有工作区并提供 PID/租约路径；工具/子代理继承任务上下文，不重复申请租约。共享 runner 只在宿主退出且确认原设备包不运行时归档超过 30 分钟的锁。离线 `finalize` 在没有设备锁或失败证据时可把较早且已遗弃的缺失回执记为中止，不能代替所需的已验证执行。其他缺证据情况仍退出 2。见[恢复说明](subcommands.zh.md#恢复与离线收尾)。 |
| 服务商反复中止 | `resume --attempts <n>` 可以在同一个已保存的任务中重试确认来自服务商响应流的中止。主动取消、策略停止或来源不明的中止不会触发自动重试；仅凭错误文本无法确认来源。`resume-progress.json` 跨恢复调用累计内容未变的连续中止，三次后停止请求（原任务中止一次，加上恢复后中止两次）。先排查原因；`--force` 仅明确允许再试一次，不删除证据、不换模型。 |
| 其他问题 | 附上 `fmbot-pbt --version`、操作系统和 `~/.pi-pbt/agent/pi-pbt-debug.log` 提交 issue |
