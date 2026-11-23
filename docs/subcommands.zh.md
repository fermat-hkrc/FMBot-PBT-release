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
| `fmbot-pbt arkts-sign` / `fmbot-pbt arkts-test` | 本地 HAP 签名 / 关联本次运行的设备判定，不调用模型 |
| `fmbot-pbt watch` | 监视分支，每个新提交测试一次 |
| `fmbot-pbt resume` | 继续一次中断的测试任务 |
| `fmbot-pbt finalize` | 不调用模型，校验并渲染已有的测试结果 |
| `fmbot-pbt scan` / `fmbot-pbt coverage` | 列出函数 / 汇总测试进度，不调用模型 |
| `fmbot-pbt dashboard` | 本机网页面板 |
| `fmbot-pbt skill-install` | 给 coding agent 安装 `pi-pbt-dev` skill |
| `fmbot-pbt kea` | HarmonyOS 真机界面测试（实验性），见 [Kea 指南](kea.zh.md) |

通用参数：`--lang zh` 输出中文；`--effort quick|standard|thorough` 测试深度；`--provider`、`--model`、`--thinking` 选择模型；`--target <目录或文件>` 指定被测代码；`--out <目录>` 指定输出目录。

## `hook-run`

```text
fmbot-pbt hook-run <sha> [--target <dir|file> [--func <name>]] [--whole] [--base <rev>] [--spec <file>]...
  [--build-cmd "<command>" [--build-dir <dir>]] [--out <dir>] [--scratch-dir <dir>]
  [--effort …] [--lang zh] [--prompt <text> | --prompt-file <path>]
  [--provider <p>] [--model <m>] [--thinking <level>] [--mode text|json|json-full] [--no-coverage]
```

| 输入 | 参数 | 说明 |
|---|---|---|
| 文档（可选） | `--spec <file>` | 需求文档，其中明确写出的行为作为判定依据；可给多次；不给时从代码推断意图 |
| 代码 | `<sha>` | 被测提交；是合并提交时，与它的第一个父提交比较 |
| | `--base <rev>` | MR 的基准提交，改为测试从它到 `<sha>` 的全部改动 |
| | `--target <dir\|file>`、`--func <name>` | 被测代码：目录或文件，默认当前目录，所在 git 仓库自动找出且必须已检出 `<sha>`（浅克隆请用 `fetch-depth: 0`）；指定子目录或文件时只测这一部分，`--func` 只测该文件中的一个函数。`--whole` 测试 `<sha>` 中 `--target` 下的全部代码，不限于改动，检出必须停在 `<sha>`，不能与 `--base` 同用 |
| 编译命令（可选） | `--build-cmd "<command>"` | 测试前先用这条命令构建检出代码；测试过程中只用这条命令重建。不给时请在 CI 中先构建，fmbot-pbt 沿用构建结果 |
| | `--build-dir <dir>` | 构建命令的运行目录，默认 `--target` 所在仓库；必须包含该仓库或位于其中 |
| 其他 | `--scratch-dir <dir>` | 会被删除重建的临时目录，默认是仓库同级的 `pbt-hook-work`；不是构建目录 |

使用 `--build-cmd` 时，检出代码必须已处于 `<sha>`（不会替你 checkout，否则以 `2` 退出）；构建失败以 `3` 退出，不开始测试，输出见 `<out>/build.log`。`--out`、`--scratch-dir` 不得包含检出代码、构建目录、当前目录或 `--spec` 文档（含符号链接）。

| 输出 | 说明 |
|---|---|
| `<out>/REPORT.md`、`REPORT.html` | 测试报告；`--out` 默认是仓库同级的 `pbt-out-hook` |
| `<out>/report.json` | 供程序读取的结果 |
| `<out>/bug_reports/` | 每个缺陷一份报告 |
| 退出码 `0` / `1` / `2` / `3` | 未发现缺陷 / 发现缺陷 / 报告缺失或无法计算改动范围 / 构建失败 |

## `build-run`

```text
fmbot-pbt build-run --build-cmd "<command>" [--build-dir <dir>] [--target <dir|file> [--func <name>]]
  [--diff <file> | --commit <sha>] [--spec <file>]... [--out <dir>] [--isolate] [--effort …] [--lang zh]
  [--prompt <text> | --prompt-file <path>] [--no-auto-resume | --recover-missing-cases [--recover-max-cases <n>] [--case-recovery-recipe <file.json>]] [--auto-resume-attempts <n>] [--auto-resume-wait <seconds>]
  [--provider <p>] [--model <m>] [--thinking <level>] [--mode text|json|json-full] [--no-coverage]
```

先运行你的构建命令，成功后再测试。`--engine acp` 改用 [Claude Code](acp.zh.md) 运行。受监督的无界面 Pi 源码测试遇到额度、限流或服务商中断（包括确认来自服务商响应流的中止）时，可恢复同一会话。`--auto-resume-attempts <n>` 限制每次命令的恢复次数（1–10 次，默认 2 次），不计入或重复初始构建。`--auto-resume-wait <seconds>` 设置首次等待（1–300 整秒，默认 15 秒），之后依次翻倍，单次最多 300 秒。重试参数的值须紧跟参数名；缺少值时，命令在构建前退出。Ctrl-C 可取消等待。在未启用监督且正在运行的无界面 Pi 任务中，Ctrl-C 让 Pi 完成 JSON 输出并执行关闭钩子；再次按 Ctrl-C 或五秒内未完成清理时，强制退出 `130` 并保留中断记录。两参数均不能与 `--no-auto-resume` 同用；`--tui`、会话内 skill、ACP、`--isolate`、预签名 HAP 或 bare 模式也不接受两参数。[恢复门禁](#恢复与离线收尾)仍生效；交付缺项不自动重试。用完次数则退出 `2` 并保留中断记录。满足离线收尾条件时，退出 `2` 并给出 `finalize` 命令，不再启动会话。标准输入非终端时，符合条件的无界面 Pi 源码测试默认启用监督。标准输入为终端时，指定 `--auto-resume-attempts`、`--auto-resume-wait` 或 `--recover-missing-cases` 即可启用监督，无需重定向。worker 及其构建进程从标准输入读到 EOF，输出方式不变。如果不指定恢复参数又想使用默认重试次数，可在 POSIX shell 或 Git Bash 中加 `< /dev/null`，在 `cmd.exe` 中加 `< NUL`。在 POSIX 上，若构建需要读取终端输入，请省略恢复参数并保持标准输入连接到终端。已启用监督时，Ctrl-Z 暂停监督进程及其 worker 进程组（包括初始构建），用 `fg` 或 `bg` 继续。已在设备上运行的任务或独立进程组中的任务不会因此暂停。Windows 原生版即使未启用监督（包括从 Git Bash 启动），也不会把标准输入传给构建命令。请使用无需交互的构建命令。

| 输入 | 参数 | 说明 |
|---|---|---|
| 文档（可选） | `--spec <file>` | 需求文档，其中明确写出的行为作为判定依据；可给多次；不给时从代码推断意图 |
| 代码 | `--target <dir\|file>`、`--func <name>` | 被测代码：目录或文件，默认当前目录；所在 git 仓库自动找出，新测试写在那里（不在 git 中时以该目录为仓库）。指定子目录或文件时只测这一部分，`--func` 只测该文件中的一个函数 |
| | `--diff <file>`、`--commit <sha>` | 本次改动：补丁文件或提交；不给则测 `--target`。补丁用相对路径时，依次在当前目录、`--target` 所在仓库、`--build-dir` 下查找；都找不到时报错并列出查过的路径 |
| 编译命令 | `--build-cmd "<命令>"` | 源码测试任务必填，指向一个已经存在、能构建成功的目标 |
| | `--build-dir <dir>` | 构建命令的运行目录，默认当前目录；必须与 `--target` 所在仓库属于同一棵源码树 |

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

HarmonyOS 应用见 [ArkTS 指南](pbt/arkts.zh.md)。`--isolate` 要求 `--target <file.ets|file.ts> --func <name>`（包括 `src/main/ets/` 下的 `.ts` 工具文件），在 `<out>/isolate-hap` 生成 HAP，把产品依赖图复制到 `entry/src/common`（不用硬链接或符号链接），更新套件导入路径，并在 `ohpm install` 后将 `--build-cmd` 替换为生成 HAP 的 ohosTest 构建。副本与原始依赖均只读且校验哈希，修改或缺失的图不能记 passing/failing；会话内 skill 使用同一准备流程。隔离准备和 HAP 构建均使用 `PATH` 或 DevEco 中已安装的 `hvigorw`，不使用被测项目的 wrapper。生成的启动器原地转发到解析出的可执行文件，并将其工具目录加入本次测试的 `PATH`，供 `hvigorw` 与 `ohpm` 使用。已安装工具链缺失时，沿用先安装工具链再构建的延迟构建流程。OHPM/HAR 裸模块导入（如 `@hmos/xyz`）会在框架准备或编译前停止隔离，以 `3` 退出。诊断仅检查 `--target` 所在仓库内的祖先模块及仓库根的 `oh_modules`，区分未安装的包与已安装但尚不能保留 HAR/模块身份投射的包。这两种情况都应省略 `--isolate`，在原 HAP 中测试；不要用桩替代依赖或展平源码。`PandaCheck` 和 `@ohos/hypium` 仍由生成 HAP 提供。系统 API 的 STOP 行为不变。

保存 JSON 事件日志（stdout 只有事件，诊断信息在 stderr；结果仍以 `report.json` 为准）：
```bash
set -o pipefail
LOG_DIR=/tmp/pbt-logs && mkdir -p "$LOG_DIR"
fmbot-pbt build-run --mode json \
  --build-cmd "./build.sh --product-name host_product --build-target base_unittest" \
  2>"$LOG_DIR/stderr.log" | tee "$LOG_DIR/events.jsonl"
```

无界面 Pi ArkTS 源码测试可用 `--recover-missing-cases` 显式允许补齐具名回执。若唯一阻塞是缺少具名回执，worker 在原有锁约束下依次尝试符合条件的用例，再核验交付，不重新调用模型。`--recover-max-cases <n>` 调整符合条件的尝试次数上限（默认 `3`），与服务商中断重试次数无关。另传 [`--case-recovery-recipe <file.json>`](arkts-device.zh.md#测试编辑后的恢复)，允许在先补齐符合条件的当前路径后，若共享的次数及时限仍有余量，为因测试改动而失效的缺口构建并签名一次；未指定时只进行设备侧补齐。见[预算及停止条件](arkts-device.zh.md#设备结果)。
设备参数、系统签名、`arkts-sign`/`arkts-test`、预签名执行与锁恢复见 [ArkTS 设备指南](arkts-device.zh.md)。指定已有包时原样运行包内套件，不声称执行了新生成的性质；两个入口均以本次 receipt 校验 `0/1/2` 判定。无界面全仓库任务可在一次修正次数耗尽后继续无关测试，但交付仍未完成（退出 `2`），详见[失败记录与收尾](arkts-device.zh.md#失败记录与收尾)。

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
| `build_cmd`、`build_dir` | 构建命令及其运行目录，同 `hook-run --build-cmd`；此时测试在原检出中进行，检出必须停在 `revision`，结束前不要修改代码 |

不带 `build_cmd` 时，测试在独立的 git worktree 中进行。结果保存在 `~/.pi-pbt/runs/<run_id>/`，生成的测试在其中的 `changes.patch`。

## 参数与路径检查

`build-run`、`hook-run`、`watch` 在构建之前检查全部输入，不符合预期就退出（`build-run`/`hook-run` 为 `2`）并指出参数、路径和原因，不交给模型去猜。会拒绝：未知参数；已移除的 `--repo`、`--scope`（改用 `--target`），`--workdir`（`build-run` 改用 `--build-dir`，`hook-run`/`watch` 改用 `--scratch-dir`），`--build-workdir`（改用 `--build-dir`），`hook-run --target change|scope`（改用 `--whole`）；不存在、不可读写的 `--build-dir`，不在其中或不可执行的 `./脚本`；不存在或不可读的 `--target`，不可写的被测仓库，不在 git 中的 `hook-run`/`watch` 目标；无法创建或写入的 `--out`、`--scratch-dir`；找不到的 `--diff`。写权限通过实际创建并删除临时文件确认。MCP 的 `pbt_start`、`pbt_watch_start` 同样拒绝未知字段，旧字段 `repo`、`scope`、`build_workdir` 会提示改用 `target`、`build_dir`。

## `watch`

```text
fmbot-pbt watch [--target <dir|file>] [--base <rev>] [--spec <file>]... [--interval <sec>] [--fetch] [--branch <name>] [--out <dir>] [--lang zh] [--effort …]
```

按从旧到新的顺序，每个新提交运行一次 `hook-run`，并带上这里给出的任意 `hook-run` 参数，开始监听前即校验。每次轮询最多测最新的 10 个新提交，需要每个都测请用 `hook-run`。使用 `--build-cmd` 时只测当前检出的 HEAD，且不能与 `--fetch` 同用。

## 会话中的 skill

在交互式 `fmbot-pbt` 会话中，`/skill:pbt-build-run`、`/skill:pbt-hook-run`、`/skill:pbt-watch` 分别接受 `build-run`、`hook-run`、`watch` 的参数，执行同一套准备、构建门禁和判定（`0`–`3`）；测试在当前会话中可见地运行。用 `stop_watch` 工具停止监听。

与命令的差异：会话保持自己的工作目录；不支持 `--mode json|json-full`；`--provider`、`--model`、`--thinking` 会切换会话的模型并在之后继续生效；`--provider` 需要同时给 `--model`。

## 其他命令

| 命令 | 说明 |
|---|---|
| `fmbot-pbt resume [--out <dir>] [--attempts <n>] [--wait <sec>]` | 继续中断的测试任务 |
| `fmbot-pbt finalize [--out <dir>] [--lang zh]` | 校验并渲染已有结果；`0` 有效，`2` 无效 |
| `fmbot-pbt scan [--dir <路径>] [--out <目录>] [--lang <语言>]` | 列出函数，写入 `FUNCTION_INDEX.md`；`--lang arkts` 仅列出 `.ets` 的[具名导出函数](coverage-tracking.md#arkts-ets)，不标为 PBT 候选项。扫描不完整时输出诊断并退出 `1`。 |
| `fmbot-pbt coverage [--list \| --json]` | 汇总 `pbt-out/` 中的测试进度 |
| `fmbot-pbt dashboard [--port 8000]` | 本机网页面板 |
| `fmbot-pbt skill-install [--host <agent>] [--dir <path>] [--force]` | 安装 `pi-pbt-dev` skill，见 [skill 指南](skill-pi-pbt-dev.zh.md) |

## 恢复与离线收尾

使用 Pi 的 `build-run --tui` 或 `hook-run --tui` 时，已保存会话的任务中断后会在当前界面暂停。输入 `/continue`，沿用同一会话、模型与思考设置继续，不重跑初始构建，也不安排自动恢复。暂停时不能切换或派生会话；执行 `/reload` 后请退出，再用终端里的恢复命令。若需更改设置、处理缺失的 ArkTS 用例回执，或证据已可离线收尾，请按命令的提示操作。`/quit` 保留证据并释放工作区租约，之后可在终端执行 `resume` 或 `finalize`。`resume` 在恢复后的 runner 进程中先取得工作区及输出目录所有权，再启动 agent；重叠的恢复或收尾请求退出 2。内部的一次性启动所有权标记在工具/子代理启动前即被消费，子进程继承任务上下文，但不会与父进程争抢租约。CLI 与会话内任务都会收到当前 runner/父进程 PID 及准确的租约文件路径。针对 fmbot-pbt/hdc 或本任务 PID 的常见宿主清理命令（包括末尾不带参数的 `Get-Process ... | Stop-Process`）会被阻止并给出修正提示，不会终止任务；不得通过杀死这些进程恢复设备锁。`resume` 默认只启动一次已保存的任务。`--attempts` 限制启动次数，`--wait` 设置两次启动之间的等待时间。在 POSIX 上，Ctrl-Z 同时暂停命令和恢复进程；用 shell 的 `fg` 或 `bg` 恢复两者。自动重试只继续原任务，并在 runner 取得租约后复查已保存的中断记录和会话。只有确认来自服务商响应流的中止才可自动重试；仅凭错误文本无法确认来源。`interruption.json` 中的 `abortOrigin` 记录中止来源；操作者信号、会话取消请求、策略停止、来源不明或旧版中止记录均不自动重试。每次任务运行期间，Pi 仍可能对 HTTP `503` 重试请求，之后才会写入 `interruption.json`。服务商明确拒绝模型访问或拒绝凭据时停止重试。获得模型权限或修正凭据后再用 `resume --force`；更改 `--thinking` 无法解决这两类问题。`resume-progress.json` 按保存的任务身份、记录产物（含两种输出布局下的设备索引/回执）及引用的源码/测试内容，跨调用跟踪中止次数。内容未变且连续中止三次后停止后续请求（退出 2）：包含原任务中止一次、恢复后中止两次，并非恢复失败三次。先检查取消操作、代理超时及服务商上下文限制；仅凭中止消息不能断定请求过大。排查后可用 `--force` 明确允许再试一次，不清除证据、不换模型。记录内容改变或开始不同任务后重新比较。

首次中止或交付未完成时，终端和 `interruption.json` 会指出缺项，或建议**在活跃任务结束后**执行 `fmbot-pbt finalize --out <dir>`。请复制完整命令：它指向所属任务的输出目录，并在任务已记录语言时带上 `--lang en` 或 `--lang zh`。检查仅读取已保存的证据；显式调用 `resume` 仍可启动。若中断后证据已满足离线条件，停止自动重试，返回 `2` 并给出离线命令。无界面的 `build-run` 在自动恢复前，还会在恢复进程持有租约时校验已保存会话的内容。若只剩符合条件的缺失回执，提示会列出相应运行。`finalize` 取得输出及工作区所有权，校验新报告、设计与结果、已登记设备回执、保留的失败及已记录的收尾复核（保留原任务的禁用/豁免决定），然后渲染，不调用模型、不再次运行设备测试。检查 `<out>`、`<out>/pbt-out` 及任务所记录工作目录下 `pbt-out/` 中由记录工具绑定到本次任务的新报告，不从启动 `finalize` 的目录借用其他任务的报告。实时、hook/会话及离线交付共用跨所属布局的设计/结果和设备证据校验：嵌套空性质清单或通过报告不能隐藏所属任务未完成的设计，也不能隐藏其他报告或恢复侧文件中的失败观察；不借用无关祖先目录的产物。成功后将匹配的检查点标为完成，并把匹配的中断保存在 `interruption.completed-<hash>.json`；不清除其他任务或无效记录。带任务 ID 的报告必须有匹配的检查点；请传入原任务的 `--out` 目录，而不是工作区下的备用报告路径。没有任务 ID 和检查点的独立报告仍可校验和渲染，但不会关闭任务。退出 `0` 表示交付有效，不表示没有缺陷；`2` 表示未完成。合法的全退役/构建失败报告不要求设备执行；已登记但不确定的调用仍阻止完成，hook 的构建失败码仍为 `3`。新任务复用输出目录时，记录工具将旧的 `report.json` 和 `REPORT.md` 保存在 `report.previous-*/`，新报告不继承旧的测试结论或复核结果。工作目录中的备用报告必须同时包含两个新报告文件并匹配本任务的检查点；没有 Pi 会话记录的 ACP 任务也适用。

ArkTS 结论和 `COVERAGE.md` 的门禁依据本任务中仍有效的设备回执判断是否执行过测试，恢复后也是如此。本任务的具名用例（包括直接编写在 `List.test.ets` 中的用例），只有回执与当前源码和测试 HAP 匹配时，才能证明执行过。仅登记其他套件不等于执行过。在属于本任务的任一输出布局中有合法的未执行报告时，即使 `COVERAGE.md` 在另一个所属布局，也可以添加覆盖率缺口说明。说明支持 LF 和 CRLF 换行，但覆盖率表格行必须保持不变。缺失用例和未分诊的失败仍阻止交付；仅构建或调用 `arkts-test` 不能证明执行。缺失回执表示执行情况未知，不是通过。无效登记（包括目标、包名或模块名为空白）或当前运行 ID 对应的登记相互冲突时，会阻止交付。`finalize` 在写入前检查所有归属本任务的索引，不改动这些运行的回执。同一索引中的相同记录不算冲突。在独占任务所有权后，`finalize` 仅对有效且无歧义的登记补写缺失回执，且要求登记已满 30 分钟、已记录的宿主持有者确认退出（旧索引可能没有持有者）、不存在设备锁且保留日志没有失败证据。补写的回执只记录中止（`code: 2`、`reason: parent-exit`）。索引及日志不变，`findings-unrecorded.json` 的 `abortedRuns` 保留处置记录；近期、存活/未知持有者、格式错误及失败证据仍阻止完成。这不能代替所需的已验证设备执行。其他情况先检查原运行，再明确请求 `arkts-test --supersedes-run <id>`，不得裁剪索引。目标缺失或已解决的链接只产生保留警告；身份不匹配、未来/自身引用及真实失败目标仍为错误。被锁拒绝的调用记为未执行，不是已验证结果；过期锁恢复由共享的[设备 runner](arkts-device.zh.md)负责。旧的通过回执不能证明改过的源码、测试、可执行文件或环境仍然有效。新性质或修改过的性质只选受影响的类/用例，共享输入变更时才扩大范围。`finalize` 交付已有结果，不声称有新的执行或覆盖。

## 测试深度

| | `quick` | `standard` | `thorough` |
|---|---|---|---|
| 时间预算 | 约 10 分钟 | 约 30 分钟 | 不限 |
| 每个测试对象的性质数 | 3–5 | 5–8 | 按需 |
| 每条性质的输入数 | 约 100 | ≥ 1000 | ≥ 10000 |

**测试对象**通常是一个函数或方法，也可以是一组需要连续执行的相关操作，例如“开户→存款→取款→销户”。性质数按每个测试对象分别计算，不是整个仓库或模块共用这几条，也不是编译命令中的构建目标。`hook-run` 与 `watch` 默认 `quick`，其他入口默认 `standard`。所有测试入口都先读用户提供的需求、API 或设计文档，再对照范围内的代码提取性质；文档未涉及的行为才从代码推断意图。

## 任务阶段

模型路由、思考等级、上下文压缩和托管任务状态共用阶段判定。存在待批准的性质设计时，即使已有报告也回到 Plan；否则，`REPORT.md` 表示 Review。报告生成前，完整的 `PLAN.md` 清单按第一个未完成模块的首个未勾选阶段判定，因此 Test 完成后即进入 Review，无须等待报告。代码块示例和引用中的清单不算进度。清单缺失、不完整或不可读时，按产物是否存在判定。只有沿用的性质清单、尚无计划时，不视为 Plan 已完成。没有产物时，模型路由暂不切换，托管任务显示 Scan。

## 环境变量

| 变量 | 说明 |
|---|---|
| `PBT_TLS_EXCEPTIONS=off` | 关闭[企业证书例外](network.zh.md)与包管理器的 TLS 兼容 |
| `NODE_USE_SYSTEM_CA=0` | 关闭默认添加系统 CA；程序保留显式指定的运行时证书选项 |
| `NODE_EXTRA_CA_CERTS=/path/ca.pem` | 添加 PEM 格式的 CA 证书；启动 fmbot-pbt 前设置，子进程继承此配置 |
| `PBT_LANG` | `zh` 或 `en`，输出语言 |
| `PBT_ARKTS_DEVICE_OPTIONS` | `build-run`、对应 skill 和设备工具共用的 JSON 设备/结果/签名配置，优先用 CLI 参数 |
| `HDC`、`JAVA` | 可选的设备传输 / SDK 本地签名可执行文件路径 |
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
