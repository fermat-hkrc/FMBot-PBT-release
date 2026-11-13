# 参考手册

[English](subcommands.md) · 两种用法的分步说明见[使用指南](usage.zh.md)

## 命令一览

| 命令 | 用途 |
|---|---|
| `fmbot-pbt` | 交互式会话，用自然语言提出测试要求 |
| `fmbot-pbt -p "<要求>"` | 无人值守运行一次；退出码不代表判定结果 |
| `fmbot-pbt mcp` | MCP 服务，供 Claude Code、Codex 调用（模式一） |
| `fmbot-pbt hook-run <sha>` | 测试一次提交或一个 MR，退出码即判定结果（模式二） |
| `fmbot-pbt build-run --build-cmd "<命令>"` | 先运行你给的构建命令，成功后再测试 |
| `fmbot-pbt watch` | 监视分支，每个新提交测试一次 |
| `fmbot-pbt resume` | 继续一次中断的测试任务 |
| `fmbot-pbt finalize` | 不调用模型，校验并渲染已有的测试结果 |
| `fmbot-pbt scan` / `fmbot-pbt coverage` | 列出函数 / 汇总测试进度，不调用模型 |
| `fmbot-pbt dashboard` | 本机网页面板 |
| `fmbot-pbt skill-install` | 给 coding agent 安装 `pi-pbt-dev` skill |
| `fmbot-pbt kea` | HarmonyOS 真机界面测试（实验性），见 [Kea 指南](kea.zh.md) |

通用参数：`--lang zh` 输出中文；`--effort quick|standard|thorough` 测试深度；`--provider`、`--model`、
`--thinking` 选择模型；`--scope <路径>` 限定测试范围；`--out <目录>` 指定输出目录。

## `hook-run`

```text
fmbot-pbt hook-run <sha> [--base <rev>] [--spec <file>]... [--repo <path>] [--out <dir>] [--workdir <dir>]
  [--build-cmd "<command>" [--build-workdir <dir>]] [--scope <path> [--func <name>]] [--target change|scope]
  [--effort …] [--lang zh] [--prompt <text> | --prompt-file <path>]
  [--provider <p>] [--model <m>] [--thinking <level>] [--mode text|json|json-full] [--no-coverage]
```

| 输入 | 参数 | 说明 |
|---|---|---|
| 文档（可选） | `--spec <file>` | 需求文档，其中明确写出的行为作为判定依据；可给多次；不给时从代码推断意图 |
| 代码 | `<sha>` | 被测提交；是合并提交时，与它的第一个父提交比较 |
| | `--base <rev>` | MR 的基准提交，改为测试从它到 `<sha>` 的全部改动 |
| | `--repo <path>` | 已检出 `<sha>` 的工作树，默认当前目录；浅克隆请用 `fetch-depth: 0` |
| | `--scope <path>`、`--func <name>` | 只测这个文件或目录，以及其中一个函数（`--func` 需要 `--scope`） |
| | `--target scope` | 测试 `<sha>` 中 `--scope` 下的全部代码，而不只是改动（`--scope .` 即全仓）；检出必须停在 `<sha>`；不能与 `--base` 同用 |
| 编译命令（可选） | `--build-cmd "<command>"` | 测试前先用这条命令构建检出代码；测试过程中只用这条命令重建。不给时请在 CI 中先构建，fmbot-pbt 沿用构建结果 |
| | `--build-workdir <dir>` | 构建命令的运行目录，默认 `--repo`；必须包含 `--repo` 或位于其中 |
| | `--workdir <dir>` | 会被删除重建的临时目录，默认是 `--repo` 同级的 `pbt-hook-work`；不是构建目录 |

使用 `--build-cmd` 时，检出代码必须已处于 `<sha>`（不会替你 checkout，否则以 `2` 退出）；构建失败以 `3` 退出，
不开始测试，输出见 `<out>/build.log`。`--out`、`--workdir` 不得包含检出代码、构建目录、当前目录或 `--spec` 文档（含符号链接）。

| 输出 | 说明 |
|---|---|
| `<out>/REPORT.md`、`REPORT.html` | 测试报告；`--out` 默认是 `--repo` 同级的 `pbt-out-hook` |
| `<out>/report.json` | 供程序读取的结果 |
| `<out>/bug_reports/` | 每个缺陷一份报告 |
| 退出码 `0` / `1` / `2` / `3` | 未发现缺陷 / 发现缺陷 / 报告缺失或无法计算改动范围 / 构建失败 |

## `build-run`

```text
fmbot-pbt build-run --build-cmd "<command>" [--spec <file>]... [--workdir <dir>] [--repo <path>] [--out <dir>]
  [--scope <path>] [--func <name>] [--diff <file> | --commit <sha>] [--isolate]
  [--effort …] [--lang zh] [--prompt <text> | --prompt-file <path>]
  [--provider <p>] [--model <m>] [--thinking <level>] [--mode text|json|json-full] [--no-coverage]
```

先运行你的构建命令，成功后再测试；测试任务只沿用这条命令构建，不另选构建方式。

| 输入 | 参数 | 说明 |
|---|---|---|
| 文档（可选） | `--spec <file>` | 需求文档，其中明确写出的行为作为判定依据；可给多次；不给时从代码推断意图 |
| 代码 | `--repo <path>` | 被测仓库，默认当前目录 |
| | `--scope <path>`、`--func <name>` | 只测这个文件或目录，以及其中的某个函数 |
| | `--diff <file>`、`--commit <sha>` | 本次改动：补丁文件或提交；不给则按 `--scope` 测试 |
| 编译命令 | `--build-cmd "<命令>"` | 必填，指向一个已经存在、能构建成功的目标 |
| | `--workdir <dir>` | 构建命令的运行目录，默认当前目录 |

| 输出 | 说明 |
|---|---|
| `<out>/build.log` | 构建日志；`--out` 默认 `<repo>/pbt-out` |
| `<out>/REPORT.md`、`report.json`、`bug_reports/` | 测试结果，同 `hook-run` |
| 项目测试目录中的新测试 | 登记到项目已有的测试目标；测试生成之后，同一条命令可以换用这个目标构建 |
| 退出码 | `3` 表示构建失败；构建成功后的结果以 `report.json` 为准 |

OpenHarmony 组件优先使用原生 `host_product`，产物位于 `out/host/host_product`。组件没有 host 目标时，
才使用已为该 device product 配置好 runner 的设备构建。整个 OpenHarmony 源码树没有一条通用的 `--build-cmd`：
`--repo` 指向当前组件，构建目标选该组件已有的单元测试。示例：

```bash
export PBT_OH_WORKSPACE=/path/to/openharmony
fmbot-pbt build-run \
  --workdir "$PBT_OH_WORKSPACE" \
  --repo "$PBT_OH_WORKSPACE/foundation/arkui/ace_engine" \
  --build-cmd "./build.sh --export-para PYCACHE_ENABLE:true --product-name host_product --build-target base_unittest --ccache --no-prebuilt-sdk" \
  --scope frameworks/base/geometry \
  --lang zh
```

普通 CMake 项目：

```bash
fmbot-pbt build-run --build-cmd "cmake -S . -B build && cmake --build build --target calc_test" --scope src/calc.cpp
```

HarmonyOS 应用见 [ArkTS 指南](pbt/arkts.zh.md)。`--isolate` 要求 `--scope <file.ets> --func <name>`，在 `<out>/isolate-hap` 生成 HAP，把产品依赖图复制到 `entry/src/common`（不用硬链接或符号链接），更新套件导入路径，并在 `ohpm install` 后将 `--build-cmd` 替换为生成 HAP 的 ohosTest 构建。副本与原始依赖均只读且校验哈希，修改或缺失的图不能记 passing/failing；会话内 skill 使用同一准备流程。

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
| `repo` | 被测仓库（必填） |
| `spec` | 需求文档（可选），一个路径或路径列表；不给时从代码推断意图 |
| `include_uncommitted` | 连同未提交的改动一起测试 |
| `revision` | 要测试的提交，默认 `HEAD` |
| `base` | 基准提交，测试从它到 `revision` 的全部改动；不能与 `target: "scope"` 同用 |
| `scope`、`target` | 限定测试范围；`target: "scope"` 测试 `scope` 下的全部代码，而不只是改动（`.` 即全仓；按模块分次运行更易阅读） |
| `effort`、`lang`、`provider`、`model` | 同命令行参数 |
| `prompt` | 追加给测试任务的说明 |
| `build_cmd`、`build_workdir` | 构建命令及其运行目录，同 `hook-run --build-cmd`；此时测试在原检出中进行，检出必须停在 `revision`，结束前不要修改代码 |

不带 `build_cmd` 时，测试在独立的 git worktree 中进行。结果保存在 `~/.pi-pbt/runs/<run_id>/`，生成的测试在
其中的 `changes.patch`。

## `watch`

```text
fmbot-pbt watch [--repo <path>] [--base <rev>] [--spec <file>]... [--interval <sec>] [--fetch] [--branch <name>] [--out <dir>] [--lang zh] [--effort …]
```

按从旧到新的顺序，每个新提交运行一次 `hook-run`，并带上这里给出的任意 `hook-run` 参数，开始监听前即校验。每次轮询
最多测最新的 10 个新提交，需要每个都测请用 `hook-run`。使用 `--build-cmd` 时只测当前检出的 HEAD，且不能与 `--fetch` 同用。

## 会话中的 skill

在交互式 `fmbot-pbt` 会话中，`/skill:pbt-build-run`、`/skill:pbt-hook-run`、`/skill:pbt-watch` 分别接受 `build-run`、
`hook-run`、`watch` 的参数，执行同一套准备、构建门禁和判定（`0`–`3`）；测试在当前会话中可见地运行。用 `stop_watch`
工具停止监听。

与命令的差异：会话保持自己的工作目录；不支持 `--mode json|json-full`；`--provider`、`--model`、`--thinking` 会切换
会话的模型并在之后继续生效；`--provider` 需要同时给 `--model`。

## 其他命令

| 命令 | 说明 |
|---|---|
| `fmbot-pbt resume [--out <dir>] [--attempts <n>] [--wait <sec>]` | 继续中断的测试任务 |
| `fmbot-pbt finalize [--out <dir>] [--lang zh]` | 校验并渲染已有结果；`0` 有效，`2` 无效 |
| `fmbot-pbt scan [--dir <path>] [--out <dir>]` | 列出函数，写入 `FUNCTION_INDEX.md`，见[覆盖率跟踪](coverage-tracking.md) |
| `fmbot-pbt coverage [--list \| --json]` | 汇总 `pbt-out/` 中的测试进度 |
| `fmbot-pbt dashboard [--port 8000]` | 本机网页面板 |
| `fmbot-pbt skill-install [--host <agent>] [--dir <path>] [--force]` | 安装 `pi-pbt-dev` skill，见 [skill 指南](skill-pi-pbt-dev.zh.md) |

## 测试深度

| | `quick` | `standard` | `thorough` |
|---|---|---|---|
| 时间预算 | 约 10 分钟 | 约 30 分钟 | 不限 |
| 每个测试对象的性质数 | 3–5 | 5–8 | 按需 |
| 每条性质的输入数 | 约 100 | ≥ 1000 | ≥ 10000 |

**测试对象**通常是一个函数或方法，也可以是一组需要连续执行的相关操作，例如“开户→存款→取款→销户”。
性质数按每个测试对象分别计算，不是整个仓库或模块共用这几条，也不是编译命令中的构建目标。

`hook-run` 与 `watch` 默认 `quick`，其他入口默认 `standard`。

所有测试入口都先读用户提供的需求、API 或设计文档，再对照范围内的代码提取性质；文档未涉及的行为才从代码推断意图。

## 环境变量

| 变量 | 说明 |
|---|---|
| `PBT_TLS_EXCEPTIONS=off` | 关闭[企业证书例外](network.zh.md)与包管理器的 TLS 兼容 |
| `NODE_USE_SYSTEM_CA=0` | 关闭默认添加系统 CA；程序保留显式指定的运行时证书选项 |
| `NODE_EXTRA_CA_CERTS=/path/ca.pem` | 添加 PEM 格式的 CA 证书；启动 fmbot-pbt 前设置，子进程继承此配置 |
| `PBT_LANG` | `zh` 或 `en`，输出语言 |
| `PBT_EFFORT` | 默认测试深度 |
| `PBT_TEST_JOBS` | 测试与构建的并行度：数字、`max` 或 `auto`（默认，空闲核心数） |
| `PBT_CODE_COVERAGE=0` | 关闭原生代码覆盖率 |
| `PBT_CLOSEOUT_REVIEW=off` | 跳过独立复核（由内置的 pi-subagents 执行）；报告会注明未复核 |
| `PBT_SCAN_DELEGATION=off` | 不使用并行扫描子代理 |
| `PBT_VERDICTS=off` / `PBT_CARRY=off` | 不记录 / 不复用历史判定结果 |
| `PBT_OH_WORKSPACE` | OpenHarmony 源码树位置；在源码树内运行时自动识别 |
| `PBT_PHASE_MODELS` / `PBT_DESIGN_THINKING` | 按阶段指定模型，如 `{"plan":"anthropic/claude-opus-5-5"}` / 扫描和设计性质阶段的思考等级，默认 `xhigh`（模型不支持时依次用 `max`、`high`），不低于会话等级；`keep` 表示沿用会话等级 |
| `PBT_MCP_MAX_CONCURRENT` | MCP 同时运行的测试数，默认 1 |
| `PI_PBT_RUNS_DIR` | MCP 结果目录，默认 `~/.pi-pbt/runs` |
