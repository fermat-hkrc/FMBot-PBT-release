# pi-pbt 子命令

[English](subcommands.md)

所有 campaign 都通过 **`pi-pbt` 二进制**启动。光跑
`pi-pbt -p "…"` 是一次自由 prompt 的无头运行，不是 CI 裁决。需要用进程退出码
决定作业成败时用 `hook-run`；`watch` 只适合作为尽力而为的常驻监视器。

支持 `--repo` 的命令默认取当前目录。`build-run` 的 `--workdir` 也默认当前目录；
`hook-run` 则默认使用下文说明的干净同级工作目录。

| 目的 | 命令 |
|---|---|
| 交互式 campaign | `pi-pbt` |
| 无头跑一次（脚本 / nohup，不做门禁） | `pi-pbt -p "…"` |
| campaign 前先跑已知构建 | `pi-pbt build-run --build-cmd "…"` |
| 门禁**一次 git 提交**（hook / CI） | `pi-pbt hook-run <sha>` |
| 监视新提交（尽力而为） | `pi-pbt watch` |
| 模拟开发者（不做 PBT） | `pi-pbt replay --pr <n>` |
| 列函数，不调 LLM | `pi-pbt scan` |
| 对扫描出的候选全部做 PBT | `pi-pbt test-all` |
| 根据 `pbt-out/` 打覆盖率 | `pi-pbt coverage` |
| Findings dashboard | `pi-pbt dashboard` |
| 给 Claude Code / Codex 的 MCP | `pi-pbt mcp` |
| 版本 | `pi-pbt --version` |

provider/model 覆盖不只支持 `build-run`：自由 `pi-pbt`、`build-run`、
`hook-run`、`watch`、`test-all` 都接受 `--provider` 与 `--model`；MCP 的
`pbt_start` / `pbt_watch_start` 有对应的 `provider`、`model` 字段。
`replay` 只接受 `--model`；Kea 从 `kea.config.yml` 读取 `provider` 和
`model`。不传覆盖值时使用 `~/.pi-pbt/agent/` 的配置。

`kea` 是真机 HarmonyOS GUI 测试的**实验功能**；前置条件、配置和命令见
[Kea 使用指南](kea.zh.md)。

下文术语：

- **campaign** 是一轮完整的 PBT：Scan → Plan → Test → Review。
- **产物目录**保存人读结果 `REPORT.md`、经校验的机读结果 `report.json`、
  `bug_reports/` 下的逐 bug 文件，以及 `PLAN.md`、`PROPERTIES.md`、
  `COVERAGE.md` 等 campaign 草稿。
- 自由运行、`scan`、`test-all`、`coverage` 默认使用 `<cwd>/pbt-out`；
  `build-run` 默认使用 `<repo>/pbt-out`；`hook-run` 刻意默认写到仓库外的
  `<parent>/pbt-out-hook`。接受 `--out` 的命令以它为准。campaign 可能把文件直接写在
  该目录，也可能再套一层 `pbt-out/`；pi-pbt 在收尾报告和覆盖率时会解析这两种布局。

---

## 全部运行方式

一个二进制，多个入口。按你想发生什么来选，细节见下文各节。

| | 命令 | 做什么 |
|---|---|---|
| **跑战役** | `pi-pbt` | 交互式，在对话里说明目标。 |
| | `pi-pbt -p "<prompt>"` | 无人值守跑一次。给脚本和自动产出用——**不是 CI 门禁**：进程退出码是 pi 的，不承载裁决，发现 bug 也不会让流水线变红。门禁用 `hook-run`。 |
| | `pi-pbt build-run --build-cmd "<命令>"` | 你的构建命令是 preflight 闸：先跑它，非零退出就在任何 agent 工作之前停掉战役。成功后要读产物；这条命令不套用 `hook-run` 的裁决退出码。之后战役**就地运行**（不建 worktree）——大型组件无法在独立副本里构建。 |
| | `pi-pbt hook-run <sha>` | 单个 commit 的改动集。**退出码就是裁决**：`0` 干净、`1` 发现 bug、`2` 没有报告或报告不合 schema、`3` 构建失败。sha 只用来取 diff 和写进 prompt，**检出由调用方负责**。 |
| | `pi-pbt watch` | 轮询分支，每个新 commit 跑一轮，从旧到新——**每次轮询最多 10 个**，更旧的会被丢弃（有日志）。 |
| | `pi-pbt test-all` | 读 `pbt-out/FUNCTION_INDEX.md`，对其中每个候选函数跑。 |
| **让别的 agent 驱动** | `pi-pbt mcp` | stdio 上的 MCP server：六个工具，产物以 `pbt://` 资源暴露。 |
| | `pi-pbt skill-install` / `skill-uninstall` | 把随包的 `pi-pbt-dev` skill 装进宿主 agent 的 skill 目录。 |
| **纯本地，不调模型** | `pi-pbt scan` | ripgrep 扫函数 → `FUNCTION_INDEX.md`。无 LLM、不起 agent。 |
| | `pi-pbt coverage` | 从已有的 `pbt-out/` 产物打印覆盖率报告。不起 agent。 |
| | `pi-pbt dashboard` | 本机所有运行的 web 看板。 |
| **GUI PBT（实验）** | `pi-pbt kea --config kea.config.yml` | 经 Kea2 做 HarmonyOS 真机 GUI PBT——性质写在 app 的 UI 状态上，不是源码级性质。目前是 demo 切片。 |
| **不是 PBT** | `pi-pbt replay --pr <n>` | 模拟开发者实现一条子需求并提交。硬隔离：不产出任何 PBT 产物。 |

两组容易混：

- **`build-run` 与 `hook-run`**。`build-run` 针对你已经知道怎么构建的目标：agent
  启动前，这条命令必须成功。但它的退出码**不是** campaign 裁决；preflight 成功后只是
  普通 pi 进程的退出码，发现 bug 不会映射成 `1`，缺报告也不会映射成 `2`。`hook-run`
  才是由产物裁决退出码的 CI 检查。两者**不能在一次调用里组合**：`hook-run` 没有
  `--build-cmd`。仓库需要特定构建命令时，在 `hook-run` 前一个 CI 步骤里先构建好，
  campaign 可以复用热的构建树。
- **`-p` 与 `mcp`**。`-p` 在当前进程里跑完战役、阻塞等待。`mcp` 把战役交给另一个进程、
  立刻返回 run id，调用方可以继续干别的。

## Skill:三种不同的东西共用一个词

| | 是什么 | 怎么得到 | 怎么调用 |
|---|---|---|---|
| **1. 随包内置 skill** | 15 份驱动战役本身的 SOP——`pbt-workflow`（四个相位）、`pbt-oracles`、`pbt-patterns`、`openharmony-build-run` 等 | **在二进制里**，并以 `dist/skills/` 放在它旁边。你不需要安装。 | agent 自动加载。其中几个也可以在 `pi-pbt` 会话里按名调用：`/skill:pbt-workflow`、`/skill:pbt-build-run`、`/skill:pbt-watch`、`/skill:kea-harmony-app`。 |
| **2. `pi-pbt-dev` skill** | 教**你日常用的 coding agent**（Claude Code、Codex、opencode、codeagent、chrys）通过调用 `pi-pbt` 二进制来跑 PBT。 | `pi-pbt skill-install`——从发行包里拷进宿主 agent 的 skill 目录。 | 宿主 agent 自己会加载；用自然语言说就行（"给这次改动跑一轮 PBT"）。Codex 需要显式点名 `$pi-pbt-dev`。 |
| **3. MCP 工具** | 根本不是 skill——六个工具加 `pbt://` 资源。想让战役跑在**独立进程**里时，用它替代第 2 种。 | `claude mcp add --transport stdio pi-pbt -- pi-pbt mcp` | 宿主 agent 调 `pbt_start` / `pbt_status` / `pbt_report`。 |

**第 2 种和第 3 种选哪个？** skill 让你的 agent 自己跑完战役并等待；MCP 把战役交给另一个
进程、立刻返回 run id，你可以继续干别的。两者不冲突，愿意的话都装。

**内置 skill 不需要你安装。** 它们与二进制同版本；在宿主机上手工拷一份对不上版本的
`SKILL.md`，正是战役行为与代码强制规则脱节的经典原因。

细节见 [skill-pi-pbt-dev.zh.md](skill-pi-pbt-dev.zh.md)（宿主 agent skill）与
[installation.zh.md](installation.zh.md)（MCP 注册）。

## 什么阶段用哪种模式

战役并不快——在 ArkUI ace_engine 的单个模块上按 `quick` 档实测要 20–30 分钟。
这一条事实决定了下面大半张表。

### 开发阶段:和另一个 coding agent 一起写代码时

你在 Claude Code / Codex / opencode 里写代码,想让这次改动被测到。一轮战役要
20–30 分钟,所以问题是:跑的时候你还能不能继续敲。**这取决于一个参数**,与你用
哪个工具无关:

| 方式 | 战役跑在哪 | 你还能改代码吗 |
|---|---|---|
| `pbt_start` **不带** `build_cmd` | 从不可变 commit 拉的 detached `git worktree` | **能** |
| `pbt_start` **带** `build_cmd` | **就在你的 checkout 里,就地跑** | **不能**——运行事件明写 `do not edit the tree until this run finishes` |
| `pi-pbt-dev` skill(会话内) | 你 agent 自己的会话,阻塞 | 不能——你在等它 |

就地不是缺陷:大型组件按真实 checkout 解析路径、生成头文件和 ccache 状态,无法在
独立副本里构建。但这意味着"委派出去继续写代码"只在 worktree 模式下成立。带构建命令
时,委派出去就去干别的——或者在第二个 checkout 里干活。

#### 在 Claude Code 里的具体流程

注册一次:

```bash
claude mcp add --transport stdio pi-pbt -- pi-pbt mcp
```

之后在会话里用自然语言说,agent 会去调工具:

```
用 pi-pbt 起一个 run:repo /path/to/repo,scope src/parser,include_uncommitted true
```

`pbt_start` 立刻返回 `run_id`。**开发期务必带 `include_uncommitted: true`**——
默认测的是 `HEAD`,不带它就是在测你没改的代码。它把工作树冻结成一个快照 commit;
你的索引、`HEAD`、refs 和文件都不动。

然后继续写代码。稍后:

```
查那个 pbt run       →  pbt_status  (相位、状态、队列位置)
读结果               →  pbt_report  (裁决、REPORT.md 片段、bug 的 URI)
```

#### 结果怎么用

`pbt_report` 返回裁决、`REPORT.md` 片段、精简的 `report.json` 摘要、bug 报告 URI，
以及 patch 是否存在。完整产物通过这些 MCP **资源**读取：

| 资源 | 用途 |
|---|---|
| `pbt://runs/<id>/run.json` | run 元数据，包括 `revision` / `baseRevision` |
| `pbt://runs/<id>/REPORT.md` | 完整的人读报告 |
| `pbt://runs/<id>/report.json` | 完整的机读结果 |
| `pbt://runs/<id>/bug_reports/<文件>.md` | 每个 bug 一份 |
| `pbt://runs/<id>/changes.patch` | **生成的仓库改动**（通常是测试与构建接线） |

`changes.patch` 是最容易被忽略的一环。worktree 模式下，campaign 把仓库改动写在一个
**运行结束就删掉**的 worktree 里，所以生成的测试与构建登记从来不在你的 checkout 里——
这个 patch 是它们进入你的树的途径。它既是 MCP 资源，也是运行目录下的一个实际文件：

```bash
# ~/.pi-pbt/runs/<run_id>/changes.patch (根目录可用 PI_PBT_RUNS_DIR 覆盖)
cd /path/to/repo
git apply --stat ~/.pi-pbt/runs/<run_id>/changes.patch   # 先看它动了什么
git apply        ~/.pi-pbt/runs/<run_id>/changes.patch   # 然后审阅、提交
```

应打到 MCP run 记录里的版本上（`pbt_status` / `pbt_report` 返回的 `revision`，也写在
`pbt://runs/<id>/run.json`）：patch 是针对这个精确快照 commit 的二进制 diff。
`include_uncommitted` run 返回的 `base_revision` 只表示快照建立在哪个 HEAD 上，**不是**
patch base。campaign 的 `report.json` 应记录同一个被测版本，但应用 patch 时以 run 记录为准。
也可以直接用自然语言让 agent 做——“把那次运行的 changes.patch 打到我的 checkout 上”——
它会读这些资源并写盘。

就地运行(设了 `build_cmd`)不产生 patch——测试已经在你的 checkout 里了。

#### 在其他 agent 里用 build-run

三条路,都不需要 skill:

- **MCP**:`pbt_start` 传 `build_cmd`(构建要在模块之上的目录跑时再加 `build_workdir`,
  比如 scope 是单个组件、构建要在 OS 源码根执行)。
- **`pi-pbt-dev` skill**:把构建命令告诉 agent,skill 会让它走 `pi-pbt build-run`。
- **直接调**:它就是个 CLI——任何能执行 shell 命令的 agent 都能跑
  `pi-pbt build-run --build-cmd "<命令>"`。

让循环更紧的两件事:给战役划范围(`scope` / `--scope`,或函数级 prompt),别直接指向
整个仓库;以及保留上一轮的 `INVARIANTS.md`——下一轮会读它,不再重新发现你已确认过的东西。

### MR / CI 流水线

| 情形 | 用 | 为什么 |
|---|---|---|
| 合并请求上的门禁 | **`hook-run <sha>`** | 就是为此而造,**退出码就是裁决**——`0` 干净、`1` 发现 bug、`2` 没有报告或报告不合 schema、`3` 构建失败。不需要再解释别的。先把这个 commit 检出:sha 取的是 diff 和 prompt,不是工作树(见下)。 |
| 仓库需要特定构建命令 | **在上一步先构建** | `hook-run` 没有 `--build-cmd`；campaign 自己编译 SUT，并可复用热的构建树。先跑 CI 自己的构建，再跑 `hook-run`。不要拿 `build-run` 代替裁决门禁：它只闸 preflight 构建。 |
| 给机器人或看板的机读结果 | **所选产物目录里的 `report.json`** | 经 schema 校验，每个 bug 都链到发现它的性质，并带可粘贴的复现命令。普通运行通常在 `pbt-out/report.json`；`--out <dir>` 会改目录；MCP managed run 以 `pbt://runs/<id>/report.json` 暴露。见 [report-schema.md](report-schema.md)。 |
| MR 上该附什么 | 同一目录里的 `REPORT.md` + `report.json` | 不要附整个产物目录——那是 campaign 草稿。见 [reproducing.md](reproducing.md)。 |

**不要**放进流水线的:`watch`(设计上就是长驻)、`dashboard`(是个 UI)、交互模式,
以及 `replay`(开发者模拟,不是 PBT)。

### 战役之后,不分阶段

它写出的测试就是项目自有 harness 里的普通测试。把它们和构建文件的登记一起提交,
之后它们就随 CI 一起跑,与 pi-pbt 无关。[reproducing.md](reproducing.md) 讲了该提交什么、
不该提交什么,以及怎么复现一个已报告的失败。

## `pi-pbt` / `pi-pbt -p`

交互 TUI,或无头跑一次。除非用 `build-run`,否则 agent **自己选**编译方式。

```bash
cd /path/to/repo
pi-pbt -p "Run property-based testing; write results to pbt-out/."
```

钉死 SOP:

```bash
pi-pbt -p "/skill:pbt-workflow 对当前仓库做性质测试,产物写到 pbt-out/。"
```

---

## `build-run` — 用你的构建命令做 preflight

```text
pi-pbt build-run --build-cmd "<command>"
  [--workdir <dir>] [--repo <path>] [--out <dir>]
  [--scope <path>] [--func <name>]
  [--lang zh] [--scan-root <dir>]
  [--effort quick|standard|thorough]
  [--provider <p>] [--model <m>] [--tui]
```

1. 在 `--workdir` 执行 `--build-cmd`。日志写到 `<out>/build.log`；
   `--out` 默认为 `<repo>/pbt-out`。
2. 非 0 → **退出码 3**，PBT 不启动。
3. 0 → 在 `--repo` 开 campaign。重建**只许复用这条命令**（需要时把其中的
   现存目标换成新生成的测试目标）。不许切换构建系统，不许推导替代构建。

这里的 `3` 只表示 preflight 失败。campaign 一旦启动，`build-run` 就是普通 pi 退出行为：
它不会检查 `REPORT.md` / `report.json`，也不会把发现 bug、缺报告或后来写进
`report.json` 的构建失败翻译成 `hook-run` 的 `1` / `2` / `3` 裁决。结果要读产物；
CI 需要裁决时用 `hook-run`。

`--scope` 是**路径**。`--func` 是一个**符号**;**必须先有 `--scope`,且写在
`--scope` 后面**。不加 `--func` 就会覆盖 `--scope` 里所有值得测的函数。

`--workdir` = 构建执行目录。`--repo` = 被测模块(`pbt-out/`、`--scope`)。
同一目录(常见 CMake):`cd` 进去,两个 flag 都省略。

**OpenHarmony 默认先用 `host_product`。** 下面这条真实 ArkUI `ace_engine`
preflight 会在 campaign 写测试之前构建现存的 `base_unittest` group:

```bash
export PBT_OH_WORKSPACE=/path/to/openharmony
pi-pbt build-run --provider xai --model grok-4.6 --lang zh \
  --workdir "$PBT_OH_WORKSPACE" \
  --repo "$PBT_OH_WORKSPACE/foundation/arkui/ace_engine" \
  --build-cmd "./build.sh --export-para PYCACHE_ENABLE:true --product-name host_product --build-target base_unittest --ccache --no-prebuilt-sdk" \
  --scope frameworks/base/geometry
```

host 产物在 `out/host/host_product`,不是 `out/host_product`。初次 preflight
必须使用已经存在的目标;只有 campaign 生成并登记测试之后,才可把命令目标换成新的
`pbt` target。`host_product` 并不覆盖所有组件:只有组件没有 host 目标、且 workspace
已经为对应产物配置 runner 时,才使用 device product。

**普通 CMake 项目。** 这是 `build-run` 泛用构建能力的独立示例,不是 OpenHarmony
等价路径。如果现有 `CMakeLists.txt` 已定义 `calc_test`,就用它做门禁:

```bash
cd /path/to/cmake-project
pi-pbt build-run --provider xai --model grok-4.6 --lang zh \
  --build-cmd "cmake -S . -B build && cmake --build build --target calc_test -j$(nproc)" \
  --scope src/calc.cpp
```

若项目的默认全量构建还包含其他损坏目标,显式指定这个已知可用的现存 target 即可
把 preflight 限住。测试生成后,campaign 可以把 `calc_test` 换成已登记的 PBT target,
但仍保持同一套 CMake 命令结构。

细节见 [installation.zh.md](installation.zh.md#先构建再测试build-run自带构建命令)。

---

## `hook-run` — 一次提交(CI / git hook)

```text
pi-pbt hook-run <sha>
  [--repo <path>] [--out <dir>] [--workdir <dir>] [--spec <file>]
  [--lang zh] [--scan-root <dir>]
  [--effort quick|standard|thorough]
  [--provider <p>] [--model <m>]
  [--scope <path>] [--run-id <id>] [--tui]
```

删除并重建 `--workdir` 与 `--out`，然后扫描并对这次提交的改动集做 PBT。默认目录是
`--repo` 的同级目录：`<parent>/pbt-hook-work` 与 `<parent>/pbt-out-hook`（设置
`--scan-root <root>` 时改为它的**子目录** `<root>/pbt-hook-work` 和
`<root>/pbt-out-hook`），不是该 root 的同级目录或 `<repo>/pbt-out`。默认 effort **quick**。

退出码——这就是 CI 作业需要的全部接口:

| | 含义 |
|---|---|
| `0` | 干净:报告满足 schema,且没有 bug。 |
| `1` | 发现 bug(`bug_reports/` 非空,或 `totals.bugs > 0`)。 |
| `2` | 没有 `REPORT.md`、没有 `report.json`,或 `report.json` 不合 schema——战役死了,或什么都没证明。 |
| `3` | 记录在案的构建失败:什么都没被测。 |

**sha 不会检出任何东西。** 它只用来取改动集(`git diff-tree <sha>`)和写进 prompt
(`git show <sha>`);真正被编译、被测的是 `--repo` 工作树里当下的代码。所以把树切到
那个 commit 是调用方的责任——CI 的 checkout 步骤本来就做了这件事:

```bash
# post-commit 钩子:不需要额外操作——新提交本来就是工作树。
pi-pbt hook-run <sha> --repo /path/to/repo --lang zh

# CI:检出到一次性工作区,并保证父提交在(fetch-depth: 2 或 0——
# depth-1 的浅克隆会让 `git diff-tree <sha>` 返回空改动集)。
git -C /path/to/worktree checkout --detach <sha>
pi-pbt hook-run <sha> --repo /path/to/worktree
```

对着另一个版本的工作树跑不会报错、也不会告警——它会老老实实地测那里的代码,而
拿来对照的却是另一个 commit 的 diff。

OpenHarmony 模块请设 `PBT_OH_WORKSPACE`,或放在带 `.repo/` + `out/` 的树里
(可自动检测)。

---

## `watch` — 每个新提交

```text
pi-pbt watch
  [--repo <path>] [--interval <sec>] [--fetch] [--branch <name>]
  [--out <dir>] [--workdir <dir>] [--spec <file>]
  [--lang zh] [--scan-root <dir>]
  [--cov-mode incremental|full] [--effort …]
  [--provider <p>] [--model <m>] [--tui]
```

每个新提交拉起一次 `hook-run`,从旧到新。默认 effort **quick**。

基线是启动时的 head:已经存在的提交不会被测。某次轮询发现超过 **10** 个新提交时——
一次推送风暴、一次 rebase,或者上一轮 20–30 分钟的战役期间攒下的队列——只测**最新的
10 个**,更旧的被丢弃并记进日志。所以 `watch` 是给人类节奏的分支用的监视器,并不保证
每个提交都被测过。需要这个保证时,就在 hook 或流水线里按提交调 `hook-run`——那条路径
不丢任何东西。

```bash
pi-pbt watch --repo /path/to/repo --lang zh --interval 30
pi-pbt watch --repo /path/to/repo --fetch --branch master --interval 60
```

---

## `replay` — 假装开发者(不是 PBT)

```text
pi-pbt replay (--pr <n> | --commit <sha>)
  [--repo <path>] [--project owner/repo]
  [--lang zh] [--scan-root <dir>] [--model <name>]
```

只做实现。设 `PBT_REPLAY=1`,**不会**开 PBT、不写 PLAN/REPORT。PBT 只应出现在
结果提交的 `hook-run`。

```bash
pi-pbt replay --pr 8721 --lang zh --scan-root /path/to/workspace
```

---

## `scan` / `test-all` / `coverage`

`scan` / `coverage` 不调 LLM。

```text
pi-pbt scan [--dir <path>] [--out <dir>] [--lang <id>] [--module <name>]
pi-pbt test-all [--dir <path>] [--out <dir>] [--provider <p>] [--model <m>]
pi-pbt coverage [--list | --json] [--module <name>]
  [--diff <sha>] [--project-dir <path>]
```

`scan` 写 `pbt-out/FUNCTION_INDEX.md`。语言探测只往下看两层目录;大型
OpenHarmony 模块在仓库根跑 `pi-pbt scan` 可能打印 `no functions found`。把
`--dir` 指到源码子树:

```bash
cd /path/to/telephony_core_service
pi-pbt scan --dir utils/codec --out ./pbt-out
```

`test-all` 对该索引里每个候选开 campaign（没有索引就先 scan）。`coverage` 始终读取
`<cwd>/pbt-out`；`--project-dir` 只改变 `--diff` 计算使用的仓库，不改变产物目录。
`--list` 与 `--json` 互斥。覆盖率语义与产物格式详见
[coverage-tracking.md](coverage-tracking.md)。

---

## `dashboard` / `mcp` / `skill-install`

```text
pi-pbt dashboard [install] [--port 8000]
pi-pbt mcp
pi-pbt skill-install [--host claude|opencode|codex|codeagent|chrys|project] [--dir <path>] [--force]
pi-pbt skill-uninstall [--host claude|opencode|codex|codeagent|chrys|project] [--dir <path>]
```

MCP:[installation.zh.md](installation.zh.md)。
Dashboard：[安装指南](installation.zh.md#5-网页面板实时看它在干什么)。

---

## 环境变量

| 变量 | 含义 |
|---|---|
| `PBT_LANG` | 输出语言(`zh` / `en`);也可用 `--lang` |
| `PBT_OH_WORKSPACE` | 完整 OH 树(`.repo/` + `out/`),供官方 `host_product` 或已配置的 device-product 构建使用。无关的普通 CMake 项目不设置它。 |
| `PBT_EFFORT` | `quick` / `standard` / `thorough` |
| `PBT_SCAN_ROOT` | 扫描目录;也可用 `--scan-root` |
| `PBT_HOOK_TUI=1` | `hook-run` / `watch` 用 TUI |

Hook-run：[CI / git hook 集成](installation.zh.md#ci--git-hook-集成)。覆盖率：
[coverage-tracking.md](coverage-tracking.md)。
