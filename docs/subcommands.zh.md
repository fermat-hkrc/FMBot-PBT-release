# 参考手册

[English](subcommands.md) · 各模式的分步说明见[使用指南](usage.zh.md)

## 命令一览

| 命令 | 用途 |
|---|---|
| `fmbot-pbt` | 交互式会话，用自然语言提出测试要求 |
| `fmbot-pbt -p "<要求>"` | 无人值守运行一次；退出码不代表判定结果 |
| `fmbot-pbt mcp` | MCP 服务，供 Claude Code、Codex 调用（模式一） |
| `fmbot-pbt hook-run <sha>` | 测试一次提交或一个 MR，退出码即判定结果（模式二） |
| `fmbot-pbt build-run --build-cmd "<命令>"` | 先运行你给的构建命令，成功后再测试 |
| `fmbot-pbt arkts-sign` / `fmbot-pbt arkts-test` | 本地 HAP 签名 / 关联本次运行的设备判定，不调用模型 |
| `fmbot-pbt watch` | 监视分支，每个新提交测试一次 |
| `fmbot-pbt resume` | 继续一次中断的测试任务 |
| `fmbot-pbt finalize` | 不调用模型，校验并渲染已有的测试结果 |
| `fmbot-pbt scan` / `fmbot-pbt coverage` | 列出函数 / 汇总测试进度，不调用模型 |
| `fmbot-pbt dashboard` / `fmbot-pbt setup` | 本机网页面板 / 交互式模型配置 |
| `fmbot-pbt skill-install` | 给 coding agent 安装 `pi-pbt-dev` skill |
| `fmbot-pbt kea` | HarmonyOS 真机界面测试（实验性），见 [Kea 指南](kea.zh.md) |

通用参数：`--help` / `-h`（或 `fmbot-pbt help <命令>`）查看用法；`--lang zh` 输出中文；`--effort quick|standard|thorough` 选择[测试深度](#测试深度)；`--provider`、`--model`、`--thinking` 选择模型；`--target <目录或文件>` 指定被测代码；`--out <目录>` 指定输出目录。不按单个函数筛选时请省略 `--func`。

## `hook-run`

```text
fmbot-pbt hook-run <sha> [--target <dir|file> [--func <name>]] [--whole] [--base <rev>] [--spec <file>]...
  [--build-cmd "<command>" [--build-dir <dir>]] [--out <dir>] [--scratch-dir <dir>]
  [--effort …] [--lang zh] [--cov-mode incremental|full] [--prompt <text> | --prompt-file <path>]
  [--provider <p>] [--model <m>] [--thinking <level>] [--mode text|json|json-full] [--no-coverage]
```

| 输入 | 参数 | 说明 |
|---|---|---|
| 文档（可选） | `--spec <file>` | 需求文档，其中明确写出的行为作为判定依据；可给多次；不给时从代码推断意图 |
| 代码 | `<sha>` | 被测提交；是合并提交时，与它的第一个父提交比较 |
| | `--base <rev>` | MR 的基准提交，改为测试从它到 `<sha>` 的全部改动 |
| | `--target <dir\|file>`、`--func <name>` | 被测目录或文件（默认 `.`）；所在 git 仓库须已检出 `<sha>`（浅克隆用 `fetch-depth: 0`）。`--func` 只测其中一个函数 |
| | `--whole` | 测试 `<sha>` 中 `--target` 下的全部代码，不限于改动；不能与 `--base` 同用 |
| 编译命令（可选） | `--build-cmd "<command>"` | 测试前先用这条命令构建检出代码；不给时请在 CI 中先构建，fmbot-pbt 沿用构建结果 |
| | `--build-dir <dir>` | 构建命令的运行目录，默认 `--target` 所在仓库；必须包含该仓库或位于其中 |
| 其他 | `--scratch-dir <dir>` | 每次运行会删除重建的临时目录，默认是仓库同级的 `pbt-hook-work` |

当前检出必须已处于 `<sha>`（不符则以 `2` 退出，不会切换分支）。省略 `--cov-mode` 时使用 `PBT_COVERAGE_MODE`（默认 `incremental`）。构建失败以 `3` 退出，输出见 `<out>/build.log`。`--out`、`--scratch-dir` 不得包含检出代码、构建目录、当前目录或 `--spec` 文档。

| 输出 | 说明 |
|---|---|
| `<out>/REPORT.md`、`REPORT.html` | 测试报告；`--out` 默认是仓库同级的 `pbt-out-hook` |
| `<out>/report.json` | 供程序读取的结果 |
| `<out>/bug_reports/` | 每个缺陷一份报告 |
| 退出码 `0` / `1` / `2` / `3` | 未发现缺陷 / 发现缺陷 / 报告缺失或无法计算改动范围 / 构建失败 |

## `build-run`

```text
fmbot-pbt build-run --build-cmd "<command>" [--build-dir <dir>] [--target <dir|file> [--func <name>]]
  [--diff <file> | --commit <sha>] [--spec <file>]... [--out <dir>] [--isolate [--isolate-test-sdk <版本>]]
  [--effort …] [--lang zh] [--prompt <text> | --prompt-file <path>]
  [--no-auto-resume | --auto-resume-attempts <n> --auto-resume-wait <seconds>]
  [--provider <p>] [--model <m>] [--thinking <level>] [--mode text|json|json-full] [--no-coverage]
```

先运行你的构建命令，成功后再测试；`--engine acp` 改用 [Claude Code 或 codeagent](acp.zh.md) 运行。

| 输入 | 参数 | 说明 |
|---|---|---|
| 文档（可选） | `--spec <file>` | 需求文档，其中明确写出的行为作为判定依据；可给多次；不给时从代码推断意图 |
| 代码 | `--target <dir\|file>`、`--func <name>` | 被测目录或文件（默认 `.`），新测试写入其所在仓库；`--func` 只测其中一个函数 |
| | `--diff <file>`、`--commit <sha>` | 仅测试补丁文件或指定提交的改动；不给则测 `--target` |
| 编译命令 | `--build-cmd "<命令>"` | 源码测试任务必填，指向一个已经存在、能构建成功的目标 |
| | `--build-dir <dir>` | 构建命令的运行目录，默认 `.`；必须与 `--target` 所在仓库属于同一棵源码树 |

| 输出 | 说明 |
|---|---|
| `<out>/build.log` | 构建日志；`--out` 默认 `<仓库>/pbt-out` |
| `<out>/REPORT.md`、`report.json`、`bug_reports/` | 测试结果，同 `hook-run` |
| 项目测试目录中的新测试 | 登记到项目已有的测试目标；测试生成之后，同一条命令可以换用这个目标构建 |
| 退出码 | `3` 表示构建失败；构建成功后的结果以 `report.json` 为准 |

OpenHarmony 组件优先使用原生 `host_product`，产物位于 `out/host/host_product`。组件没有 host 目标时，才使用已为该 device product 配置好 runner 的设备构建。整个 OpenHarmony 源码树没有一条通用的 `--build-cmd`：`--target` 指向当前组件（或其中的目录、文件），`--build-dir` 指向源码树根目录，构建目标选该组件已有的单元测试。示例：

```bash
export PBT_OH_WORKSPACE=/path/to/openharmony
fmbot-pbt build-run \
  --build-dir "$PBT_OH_WORKSPACE" \
  --target "$PBT_OH_WORKSPACE/foundation/arkui/ace_engine/frameworks/base/geometry" \
  --build-cmd "./build.sh --export-para PYCACHE_ENABLE:true --product-name host_product --build-target base_unittest --ccache --no-prebuilt-sdk" \
  --lang zh
```

普通 CMake 项目：

```bash
fmbot-pbt build-run --build-cmd "cmake -S . -B build && cmake --build build --target calc_test" --target src/calc.cpp
```

HarmonyOS 应用：`--isolate`（配合 `--target <file.ets|file.ts> --func <name>`）在 `<out>/isolate-hap` 生成迷你 HAP 测试单个函数；依赖 OHPM/HAR 包或系统 API 时请省略 `--isolate`，在原工程中测试。ArkTS 隔离限制、签名、预签名 `--hap`/`--test-hap` 与 `--recover-missing-cases` 等参数见 [ArkTS 指南](pbt/arkts.zh.md)与 [ArkTS 设备指南](arkts-device.zh.md)。

保存 JSON 事件日志（stdout 只有事件，诊断信息在 stderr；结果仍以 `report.json` 为准）：
```bash
set -o pipefail
LOG_DIR=/tmp/pbt-logs && mkdir -p "$LOG_DIR"
fmbot-pbt build-run --mode json \
  --build-cmd "./build.sh --product-name host_product --build-target base_unittest" \
  2>"$LOG_DIR/stderr.log" | tee "$LOG_DIR/events.jsonl"
```

## `mcp`

```bash
claude mcp add --transport stdio fmbot-pbt -- fmbot-pbt mcp
codex mcp add fmbot-pbt -- fmbot-pbt mcp
```

| 工具 | 用途 |
|---|---|
| `pbt_start` | 发起测试，立即返回 `run_id` |
| `pbt_status` | 查询进度 |
| `pbt_report` | 读取判定结果、报告摘要和缺陷报告 |
| `pbt_cancel` | 取消测试 |
| `pbt_watch_start` / `pbt_watch_stop` | 监视分支，每个新提交测试一次 |

`pbt_start` 的参数：

| 参数 | 说明 |
|---|---|
| `target` | 被测代码：目录或文件（必填）；所在的 git 仓库自动找出 |
| `spec` | 需求文档（可选），一个路径或路径列表；不给时从代码推断意图 |
| `include_uncommitted` | 连同未提交的改动一起测试 |
| `revision` | 要测试的提交，默认 `HEAD` |
| `base` | 基准提交，测试从它到 `revision` 的全部改动；不能与 `whole` 同用 |
| `whole` | 为 `true` 时测试 `target` 下的全部代码，默认只测改动；整仓测试建议按模块分次运行 |
| `effort`、`lang`、`provider`、`model` | 同命令行参数 |
| `prompt` | 追加给测试任务的说明 |
| `build_cmd`、`build_dir` | 构建命令及其运行目录，同 `hook-run --build-cmd`；测试在停于 `revision` 的原检出中进行 |

不带 `build_cmd` 时，测试在独立的 git worktree 中进行。结果保存在 `~/.pi-pbt/runs/<run_id>/`（生成的测试在 `changes.patch`），用 `pbt_report` 返回的 URI 读取。

## 参数与路径检查

`build-run`、`hook-run`、`watch` 与 MCP `pbt_start` / `pbt_watch_start` 在构建前校验全部输入；遇到未知或已移除参数（`--repo`、`--scope`、`--workdir`、`--build-workdir`）、不可读写的目录、缺失的 `--diff` 或不可执行的本地构建脚本时立即停止（`build-run` 与 `hook-run` 退出 `2`，`watch` 非零退出，MCP 返回错误），并指出参数、路径与原因。

## `watch`

```text
fmbot-pbt watch [--target <dir|file>] [--base <rev>] [--spec <file>]... [--interval <sec>] [--fetch] [--branch <name>] [--out <dir>] [--lang zh] [--effort …]
```

按从旧到新的顺序为每个新提交运行 `hook-run`（单次轮询最多测最新 10 个提交；`--prompt` 与 `--prompt-file` 可按顺序重复混用）。使用 `--build-cmd` 时只测试当前检出的 `HEAD`，且拒绝 `--fetch`；要在独立工作树中测试未检出的提交，请用不带 `build_cmd` 的 MCP `pbt_start` 或 `pbt_watch_start`。

## 会话中的 skill

在交互式 `fmbot-pbt` 会话中，`/skill:pbt-build-run`、`/skill:pbt-hook-run`、`/skill:pbt-watch` 分别接受 `build-run`、`hook-run`、`watch` 的参数，执行同一套准备、构建门禁和判定（`0`–`3`）；用 `stop_watch` 停止监听。

与命令的差异：会话保持自己的工作目录；不支持 `--mode json|json-full`；`--provider`、`--model`、`--thinking` 会切换会话的模型并在之后继续生效（`--provider` 需同时给 `--model`）。

## 其他命令

| 命令 | 说明 |
|---|---|
| `fmbot-pbt resume [--out <dir>] [--attempts <n>] [--wait <sec>]` | 继续中断的测试任务 |
| `fmbot-pbt finalize [--out <dir>] [--lang zh]` | 校验并渲染已有结果；`0` 有效，`2` 无效 |
| `fmbot-pbt scan [--dir <路径>] [--out <目录>] [--lang <语言>]` | 列出函数并写入 `FUNCTION_INDEX.md`（单模块通过 `--dir` 指定源码目录）；路径缺失或 `rg` 扫描不完整时退出 `1`，见[覆盖率跟踪](coverage-tracking.md) |
| `fmbot-pbt coverage [--list \| --json] [--diff <sha>]` | 汇总 `pbt-out/` 中的测试进度；指定提交时逐文件统计匹配的定义，无法匹配时计入该文件的索引函数 |
| `fmbot-pbt dashboard [--port 8000]` / `fmbot-pbt setup` | 本机网页面板 / 交互式配置 `~/.pi-pbt/agent/` 下的密钥与默认模型 |
| `fmbot-pbt skill-install [--host <agent>] [--dir <path>] [--force]` | 安装 `pi-pbt-dev` skill，见 [skill 指南](skill-pi-pbt-dev.zh.md) |

## 恢复与离线收尾

`--tui` 会话暂停后，输入 `/continue` 在原会话继续，输入 `/quit` 保存状态以便在终端恢复。`fmbot-pbt resume [--out <dir>] [--attempts <n>] [--wait <sec>] [--force]` 在独占工作区与输出目录后继续中断的测试任务（默认启动 1 次；自动重试仅适用于确认来自服务商流的中止，内容未变连续中止 3 次后停止，加 `--force` 允许再试一次）。

已记录的证据齐全时，可在活跃任务结束后传入原任务的 `--out` 目录运行 `fmbot-pbt finalize --out <原输出目录>`（或直接复制中断提示给出的命令），离线校验产物并渲染 `REPORT.md`，不调用模型或重跑测试（`0` 交付有效，`2` 未完成；阻塞原因见 `interruption.json`、`recovery.json` 或 `REPORT.incomplete.md`）。ArkTS 设备回执与锁恢复规则见 [ArkTS 设备指南](arkts-device.zh.md)。

## 测试深度

| | `quick` | `standard` | `thorough` |
|---|---|---|---|
| 时间预算 | 约 10 分钟 | 约 30 分钟 | 不限 |
| 每个测试对象的性质数 | 3–5 | 5–8 | 按需 |
| 每条性质的输入数 | 约 100 | ≥ 1000 | ≥ 10000 |

**测试对象**是一个函数、方法或一组连续执行的相关操作；性质数按测试对象分别计算，不是全仓或构建目标共用。`hook-run` 与 `watch` 默认 `quick`，其他入口默认 `standard`。超过 400 个实现文件时，单次扫描整个仓库无法认证完整性（退出 `2`）；大型仓库请按模块目录用 `--target <模块目录>` 和独立的 `--out` 分次测试。显式传入的 `--spec` 文档优先，文档未涉及处才从代码推断意图。

## 任务阶段

模型路由、思考等级、上下文压缩与托管任务状态根据任务产物推断当前阶段：尚无产物时为 `Scan`，存在未完成的 `PLAN.md` 清单或待审性质设计时为 `Plan`，执行测试时为 `Test`，测试完成或写出 `REPORT.md` 后进入 `Review`。

## 环境变量

| 变量 | 说明 |
|---|---|
| `PBT_TLS_EXCEPTIONS=off` | 关闭[企业证书例外](network.zh.md)与包管理器的 TLS 兼容 |
| `NODE_USE_SYSTEM_CA=0` | 关闭默认添加系统 CA；程序保留显式指定的运行时证书选项 |
| `NODE_EXTRA_CA_CERTS=/path/ca.pem` | 添加 PEM 格式的 CA 证书；启动 fmbot-pbt 前设置，子进程继承此配置 |
| `PBT_LANG` | `zh` 或 `en`，输出语言 |
| `PBT_ARKTS_DEVICE_OPTIONS` / `HDC`、`JAVA` | 共用的 JSON ArkTS 设备配置（优先用 CLI 参数） / 指定 `hdc` 与 `java` 路径 |
| `PBT_EFFORT` / `PBT_SKIP_SETUP=1` | 默认测试深度 / 跳过交互式终端首次未配置时的自动配置提示 |
| `PBT_TEST_JOBS` | 测试与构建的并行度：数字、`max` 或 `auto`（默认，空闲核心数） |
| `PBT_CODE_COVERAGE=0` | 关闭原生代码覆盖率 |
| `PBT_CLOSEOUT_REVIEW=off` | 跳过独立复核（由内置的 pi-subagents 执行）；报告会注明未复核 |
| `PBT_SCAN_DELEGATION=off` | 不使用并行扫描子代理 |
| `PBT_VERDICTS=off` / `PBT_CARRY=off` | 不记录 / 不复用历史判定结果 |
| `PBT_OH_WORKSPACE` | OpenHarmony 源码树位置；在源码树内运行时自动识别 |
| `PBT_PHASE_MODELS` / `PBT_DESIGN_THINKING` | 按阶段指定模型，如 `{"plan":"anthropic/claude-opus-5-5"}` / 扫描和设计性质阶段的思考等级（默认 `xhigh`，`keep` 沿用会话等级） |
| `PBT_MCP_MAX_CONCURRENT` | MCP 同时运行的测试数，默认 1 |
| `PI_PBT_RUNS_DIR` | MCP 结果目录，默认 `~/.pi-pbt/runs` |
