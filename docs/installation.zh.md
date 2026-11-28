# 安装指南

[English](installation.md) · [使用指南](usage.zh.md)

## 1. 系统要求

先准备好项目本身的编译、依赖解析和运行环境，确认现有构建与测试命令可用，再运行 fmbot-pbt。发布的二进制已内置运行时，无须单独安装 Node.js；npm/源码渠道要求 **Node.js 22.x（至少 22.19）或 24.11+**。涉及数据库、服务、设备或模拟器的项目，也要先准备这些运行依赖。下表列出基本工具：

| 被测项目 | 需要 |
|---|---|
| 任何项目 | `git`（`scan` 使用 `rg`，发布包已内置） |
| C / C++ | C++17 编译器、项目自己的构建工具；CMake 项目需 `cmake` ≥ 3.16；随包提供 RapidCheck 源码，无须另行下载 |
| Python | `python3` ≥ 3.9 与 `pip` |
| Go | `go` |
| Java / Kotlin | JDK 与项目自带的 Maven / Gradle |
| HarmonyOS 应用 | DevEco 工具、匹配 SDK、`ohpm`、`hdc`、Bash（Windows 用 Git Bash）、`unzip` 或支持 ZIP 的 `tar` 及设备或模拟器；见 [ArkTS 指南](pbt/arkts.zh.md) |
| OpenHarmony 组件 | 已配置好的 OpenHarmony 源码树 |

## 2. 下载与安装

| 平台 | 文件 |
|---|---|
| Linux x64 | `fmbot-pbt-linux-x64.tar.xz` |
| Linux arm64 | `fmbot-pbt-linux-arm64.tar.xz` |
| macOS Apple Silicon | `fmbot-pbt-macos-arm64.tar.xz` |
| Windows x64（实验性） | `fmbot-pbt-windows-x64.tar.xz` |

下载配套的 `.sha256` 文件。在 Linux 上用 `sha256sum -c <校验文件>`，在 macOS 上用 `shasum -a 256 -c <校验文件>`。校验成功后再安装：

```bash
PLATFORM=linux-x64        # 或 linux-arm64、macos-arm64
tar -xJf "fmbot-pbt-${PLATFORM}.tar.xz" && cd "fmbot-pbt-${PLATFORM}" || exit 1
if [ "$(uname -s)" = Darwin ]; then xattr -dr com.apple.quarantine .; fi
./install.sh
```

`install.sh` 同时把旧命令名 `pi-pbt` 链接到 `fmbot-pbt`，并把剪贴板模块 `native/` 放在可执行文件旁。

Windows：在 PowerShell 中运行 `Get-FileHash .\fmbot-pbt-windows-x64.tar.xz -Algorithm SHA256` 并与校验文件比对。校验成功后，用 7-Zip 或 Windows 11 文件资源管理器解压，把解出的目录加入 `PATH`，运行 `fmbot-pbt.exe`（旧命令名 `pi-pbt.cmd` 会转发给它）。Windows 上不支持 C/C++ 与 OpenHarmony 项目。

## 3. 配置模型

在尚未配置模型的交互式终端启动 `fmbot-pbt` 时会自动打开配置向导，也可随时运行 `fmbot-pbt setup`（TUI 会话内输入 `/setup` 可打开内置的 `pi-setup` 配置面板）。已有可用的 Claude Code 或 codeagent 时，可跳过模型配置，直接[用 ACP agent 运行](acp.zh.md)。

如需手动配置 `~/.pi-pbt/agent/models.json`（以标准 `/v1/chat/completions` 接口的 GLM-5.3 为例，已有文件时合并）：

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

把 `baseUrl`、`apiKey` 和 `id` 换成网关实际值。`api: "openai-completions"` 对应 `/v1/chat/completions`（`baseUrl` 填到 `/v1`）；`/v1/responses` 用 `openai-responses`，`/v1/messages` 用 `anthropic-messages`。

`reasoning: true` 与 `compat` 按服务商要求填写；只有模型和渠道都支持图片时才加 `"input": ["text", "image"]`。`contextWindow` 控制自动压缩上限（参考[模型说明](https://huggingface.co/zai-org/GLM-5.3)），`maxTokens` 是单次请求输出预算。

再将默认模型写入 `~/.pi-pbt/agent/settings.json`（已有文件时合并）：

```json
{
  "defaultProvider": "my-provider",
  "defaultModel": "glm-5.3",
  "defaultThinkingLevel": "high"
}
```

模型的 `thinkingLevelMap` 映射了 `xhigh` 或 `max` 时，扫描和设计性质阶段优先使用 `xhigh`（依次回退到 `max`、`high`）；见[任务阶段](subcommands.zh.md#任务阶段)与 [`PBT_DESIGN_THINKING`](subcommands.zh.md#环境变量)。其他字段见[模型配置参考](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/models-configuration.md)与[服务商参考](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/providers.md)。

HTTPS 默认使用系统与内置证书，并对华为及官方依赖源启用[企业证书兼容](network.zh.md)，无需额外配置。

## 4. 验证

运行 `command -v fmbot-pbt`（PowerShell 用 `Get-Command fmbot-pbt`），再运行 `fmbot-pbt --version`。`fmbot-pbt --list-models` 只显示已加载的模型，不能验证 API 凭据。

第一次测试：在项目目录运行 `fmbot-pbt build-run --target <模块目录> --effort standard --build-cmd "<构建命令>"`，填入模块路径和可用的构建命令，结果见 `pbt-out/REPORT.md`。
交互式测试与 CI 门禁见[使用指南](usage.zh.md)。HarmonyOS 应用请先完成 [ArkTS 准备清单](pbt/arkts.zh.md#首次测试任务前的检查清单)。

## 5. 升级

下载新版本压缩包，按第 2 节重新安装。`~/.pi-pbt/agent/` 中的配置会保留。

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
| `fmbot-pbt --list-models` 没有输出 | 运行 `fmbot-pbt setup`，或按第 3 节配置模型 |
| 测试任务中断 | 运行 `fmbot-pbt resume` 继续，证据已齐全时运行 `fmbot-pbt finalize --out <dir>`；查看 `pbt-out/interruption.json` 和 `pbt-out/recovery.json`（[恢复说明](subcommands.zh.md#恢复与离线收尾)） |
| HarmonyOS SDK、HAP 构建或设备报错 | 按 [ArkTS 准备清单](pbt/arkts.zh.md#首次测试任务前的检查清单)与 [ArkTS 设备指南](arkts-device.zh.md)排查 |
| 其他问题 | 附上 `fmbot-pbt --version`、操作系统和 `~/.pi-pbt/agent/pi-pbt-debug.log` 提交 issue |
