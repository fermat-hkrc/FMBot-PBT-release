# 安装指南

[English](installation.md) · 装好之后怎么用，见[使用指南](usage.zh.md)。

## 1. 系统要求
先准备好项目本身的编译、依赖解析和运行环境，确认现有构建与测试命令可用，再运行 fmbot-pbt。发布的二进制已内置运行时，无须单独安装 Node.js；npm/源码渠道要求 **Node.js 22.x（至少 22.19）或 24.11+**，与内置语法解析器一致。
涉及数据库、服务、设备或模拟器的项目，也要先准备这些运行依赖。下表列出基本工具：

| 被测项目 | 需要 |
|---|---|
| 任何项目 | `git` |
| C / C++ | C++17 编译器、项目自己的构建工具；CMake 项目需 `cmake` ≥ 3.16；随包提供 RapidCheck 源码，无须另行下载 |
| Python | `python3` ≥ 3.9 与 `pip` |
| Go | `go` |
| Java / Kotlin | JDK 与项目自带的 Maven / Gradle |
| HarmonyOS 应用 | DevEco 工具、匹配 SDK、`ohpm`、Python 3 及设备/模拟器。HAR 构建优先选 SDK 内置 Node，独立 SDK 可用 PATH 中的 Node.js 22+；见 [ArkTS 指南](pbt/arkts.zh.md) |
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

`install.sh` 同时把旧命令名 `pi-pbt` 链接到 `fmbot-pbt`。macOS 首次运行前执行 `xattr -d com.apple.quarantine fmbot-pbt`。

Windows：用 7-Zip 或 Windows 11 文件资源管理器解压，把解出的目录加入 `PATH`，运行 `fmbot-pbt.exe`（旧命令名 `pi-pbt.cmd` 会转发给它）。
Windows 上不支持 C/C++ 与 OpenHarmony 项目。

## 3. 配置模型

以提供标准 `/v1/chat/completions` 接口的 GLM-5.3 服务为例，将下面配置写入 `~/.pi-pbt/agent/models.json`（已有文件时合并）：

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

扫描和设计性质阶段决定约定和性质，使用 `xhigh` 思考等级，模型不支持时依次用 `max`、`high`；测试任务其余阶段使用上面的等级。只有模型的 `thinkingLevelMap` 映射了 `xhigh` 或 `max`（如 `"thinkingLevelMap": {"xhigh": "xhigh"}`，值以接口实际接受的为准），pi 才认为模型支持这两个等级。用 `PBT_DESIGN_THINKING` 修改或关闭（[参考](subcommands.zh.md)）。其他字段见[模型配置参考](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/models-configuration.md)；
登录与内置服务商密钥配置见[服务商参考](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/providers.md)。

HTTPS 默认使用系统与内置证书，并对华为及官方依赖源启用[企业证书兼容](network.zh.md)，无需额外配置。

## 4. 验证
运行 `command -v fmbot-pbt`、`fmbot-pbt --version` 和 `fmbot-pbt --list-models`，确认程序可用且模型已加载。

第一次测试：在 `fmbot-pbt` 会话里输入 `/skill:pbt-hook-run <sha>`，或直接用自然语言描述；无人值守时运行 `fmbot-pbt hook-run <sha>`。加 `--target scope --scope <路径>` 可测试整个模块（`.` 即全仓），而不只是改动。目标需要项目构建时，两种方式都加 `--build-cmd "<命令>"`（以及 `--build-workdir <目录>`），且检出必须停在 `<sha>`。退出码：`0` 无缺陷，`1` 发现缺陷，`2` 未完成，`3` 构建失败。全部参数见[参考](subcommands.zh.md)。
大型应用中的单个 ArkTS 函数可用 `build-run --isolate --scope <file.ets> --func <name>`，仍需传 `--build-cmd`。引擎在 `<out>/isolate-hap` 生成 HAP，把产品依赖图复制到 `entry/src/common`，将构建命令换成该 HAP 的 ohosTest 构建，并给出套件的精确导入路径。副本和原始源码都经过完整性检查，不得修改；会话内 `/skill:pbt-build-run` 行为相同。隔离时，字面量 `app.string.*` 引用的字符串条目从源码所属模块复制，保留限定词目录和字符串别名。发现过程保留复制源码中 ArkUI 构建器方法体内的资源调用，但不生成 ArkUI 测试。资源缺失、动态引用、类型不支持或有歧义时停止准备，不补造占位值。资源完整性检查覆盖原所属模块及隔离 HAP 的 `AppScope`、main、ohosTest 资源命名空间，不只保护已有副本。不得增删资源文件或限定词目录，也不得注入覆盖定义；这些操作会被阻止，即使源码哈希未变，报告也不能通过校验。较大的依赖图把基线存入有哈希校验的 `isolate-hap/.pbt-isolate-baseline.tsv`，避免进程环境过大；不要修改或删除该文件。首次构建前，共享引擎会构建并安装 PandaCheck、解析所有模块依赖，日志在 `isolate-setup/build.log`。这一步在所有平台都使用 Bash（Windows 需 Git Bash）；实际构建命令仍使用原平台 shell。引导测试会导入真实源码依赖图和框架，避免只编译空测试架就通过构建门禁；引导检查不算 SUT 性质结论。HAR 重建会刷新消费模块依赖，并拒绝安装字节仍旧的运行库。详见 [ArkTS 指南](pbt/arkts.zh.md)。

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
| `fmbot-pbt --list-models` 没有输出 | 模型尚未配置，见第 3 节 |
| 测试任务中断 | 运行 `fmbot-pbt resume` 继续；中断原因记录在 `pbt-out/recovery.json` 和 `pbt-out/interruption.json` |
| 其他问题 | 附上 `fmbot-pbt --version`、操作系统和 `~/.pi-pbt/agent/pi-pbt-debug.log` 提交 issue |
