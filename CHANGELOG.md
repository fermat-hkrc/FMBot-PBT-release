# Changelog / 更新日志

Every release is recorded here in English and Simplified Chinese. GitHub Release
notes are generated from the matching version entry; the two copies must not be
maintained independently.

每个版本均在此以英文和简体中文记录。GitHub Release notes 从对应版本条目自动生成，
不再单独维护另一份文案。

Entries before v0.1.7 predate this file and remain available in the Release
history. / v0.1.7 之前的版本早于本文件，仍可在 Release 历史中查看。

## 0.1.14 - 2026-09-09

### English

#### Fixed

- Empty or thinking-only assistant turns no longer queue settle steers
  (contract-surface sweep, depth, build). A completed empty `stop` (seen on
  GLM-5.1; any model can do the same) used to continue headless `-p` into more
  empty stops — a silent loop (#380, #381).

#### Embedded SDK

- Embedded pi `0.85.1`. `pi-pbt --version` reports
  `pi-pbt 0.1.14 (pi 0.85.1)`.

### 中文

#### 修复

- 空回复或仅 thinking 的 assistant 回合不再排队收尾 steer(合同面扫描、加深、
  构建)。一次正常结束的空 `stop`(GLM-5.1 上出现过;任何模型都可能)会让无头
  `-p` 继续空转 —— 静默死循环(#380, #381)。

#### 内嵌 SDK

- 内嵌 pi `0.85.1`。`pi-pbt --version` 显示 `pi-pbt 0.1.14 (pi 0.85.1)`。

## 0.1.13 - 2026-09-08

### English

#### Changed

- **The project's repositories were renamed.** The development repository is
  now `fermat-hkrc/FMBot-PBT` (was `pbt-agent`), and the public release
  mirror — where you download pi-pbt — is now
  **`fermat-hkrc/FMBot-PBT-release`** (was `pbt-agent-release`):
  https://github.com/fermat-hkrc/FMBot-PBT-release/releases. GitHub keeps
  permanent redirects on the old names, so existing links, clones, and
  download scripts continue to work; please update bookmarks and automation
  to the new URLs at your convenience. The product, binary, and config
  directory names (`pi-pbt`, `~/.pi-pbt/`) are unchanged.

#### Fixed

- Language-feature discipline in the `build-run` build contract: campaigns
  must never enable convention-disabled features (`-frtti`, …) on test
  targets to appease a PBT framework. RapidCheck's no-rtti mode
  (`RC_DONT_USE_RTTI`, defined publicly in its third_party BUILD.gn) is the
  documented fix for `typeid` errors under `-fno-rtti` trees, validated on
  OpenHarmony (rebuild + full property suite green). Exceptions remain a
  framework-public-config-only, testonly-scoped hard requirement.

#### Embedded SDK

- Embedded pi `0.85.1`. `pi-pbt --version` reports
  `pi-pbt 0.1.13 (pi 0.85.1)`.

### 中文

#### 变更

- **项目仓库已更名。** 开发仓现为 `fermat-hkrc/FMBot-PBT`(原 `pbt-agent`);
  下载 pi-pbt 的公开发布镜像现为 **`fermat-hkrc/FMBot-PBT-release`**
  (原 `pbt-agent-release`):
  https://github.com/fermat-hkrc/FMBot-PBT-release/releases。GitHub 对旧名
  保留永久重定向,现有链接、克隆与下载脚本均继续可用;请在方便时把书签与
  自动化脚本更新到新地址。产品、二进制与配置目录名(`pi-pbt`、`~/.pi-pbt/`)
  不变。

#### 修复

- `build-run` 构建契约的语言特性纪律:campaign 绝不为迁就 PBT 框架而在测试
  目标上开启项目约定禁用的特性(`-frtti` 等)。`-fno-rtti` 树上的 `typeid`
  报错,正解是 RapidCheck 的 no-rtti 模式(在其 third_party BUILD.gn 的
  public config 定义 `RC_DONT_USE_RTTI`),已在 OpenHarmony 上验证(重建 +
  全部性质通过)。异常仍仅由框架 public config 携带、限 testonly 作用域的
  硬需求。

#### 内嵌 SDK

- 内嵌 pi `0.85.1`。`pi-pbt --version` 显示 `pi-pbt 0.1.13 (pi 0.85.1)`。

## 0.1.12 - 2026-08-31

### English

#### Added

- New user-invocable `/skill:pbt-build-run`: the in-session, watch-it-live
  equivalent of the `build-run` subcommand. Say
  `/skill:pbt-build-run 构建命令: <cmd>` in an interactive session and the
  agent runs your build command as the gate right in front of you — on
  failure it STOPS and presents the log (never explores how to compile), on
  success it runs the full `pbt-workflow` campaign in the same session under
  the build contract. The orchestration extension recognizes the invocation
  and activates the campaign plus the prebuilt contract in-session
  (session-scoped: a later unrelated prompt clears it). The subcommand
  remains the headless/CI form; both entries are documented in the install
  guides.

#### Embedded SDK

- Embedded pi `0.84.4` (unchanged). `pi-pbt --version` reports
  `pi-pbt 0.1.12 (pi 0.84.4)`.

### 中文

#### 新增

- 新增用户可调用的 `/skill:pbt-build-run`:`build-run` 子命令的会话内实时
  观测形态。在交互会话里说 `/skill:pbt-build-run 构建命令: <cmd>`,agent
  当着你的面先跑构建命令做门禁——失败即 STOP 并呈现日志(绝不探索编译方式),
  成功则在同一会话里按 `pbt-workflow` SOP 跑完整 campaign,构建契约全程生效。
  编排扩展识别该调用并就地激活 campaign 与 prebuilt 契约(会话级作用域:
  后续无关 prompt 自动清除)。子命令仍是无头/CI 形态;两种入口均已写入
  双语安装文档。

#### 内嵌 SDK

- 内嵌 pi `0.84.4`(未变)。`pi-pbt --version` 显示
  `pi-pbt 0.1.12 (pi 0.84.4)`。

## 0.1.11 - 2026-08-31

### English

#### Fixed

- Skill-side coverage for the `build-run` build contract, closing the gaps
  left after v0.1.10: the two build-exploration cookbooks
  (`openharmony-build-run`, `project-build`) now open with a "does NOT apply
  under the build contract" refusal guard — defense in depth for an agent
  that reads them despite the workflow saying to skip them — and the
  `pi-pbt-dev` host-agent delegation skill documents when and how to launch
  `pi-pbt build-run` (user-supplied build command) instead of its default
  campaign path. All three surfaces are pinned by regression tests.

#### Embedded SDK

- Embedded pi `0.84.4` (unchanged). `pi-pbt --version` reports
  `pi-pbt 0.1.11 (pi 0.84.4)`.

### 中文

#### 修复

- 补齐 `build-run` 构建契约的 skill 侧覆盖(收掉 v0.1.10 后的缺口):两本
  编译探索手册(`openharmony-build-run`、`project-build`)开头新增"构建契约
  下本手册不适用"的拒绝守卫——即使 agent 无视 workflow 的指引误读手册也会被
  顶回,属纵深防御;`pi-pbt-dev` 宿主 agent 委托 skill 新增"用户自带构建命令
  (build-run 模式)"小节,说明何时改用 `pi-pbt build-run` 而非默认战役路径。
  三处均有回归测试钉住。

#### 内嵌 SDK

- 内嵌 pi `0.84.4`(未变)。`pi-pbt --version` 显示
  `pi-pbt 0.1.11 (pi 0.84.4)`。

## 0.1.10 - 2026-08-31

### English

#### Added

- New `build-run` subcommand: bring your own build command. The command is
  USER-PREPARED INPUT — `pi-pbt build-run --workdir <dir> --build-cmd "<cmd>"`
  executes it first (full log in `<out>/build.log`); if it fails, PBT never
  starts and the process exits `3`. On success the campaign runs under an
  immutable build contract: rebuilds reuse exactly that command (only the
  build target may switch to the new test target), a failing rebuild is a
  STOP recorded in `REPORT.md`, and the agent never explores alternative
  ways to compile (no `oh-closure.py`, no standalone build derivation — the
  in-campaign steers are rewritten accordingly in this mode).
- Under the build contract, new tests are wired the repository's OFFICIAL
  unit-test way — the component's own GN unittest template added to its
  existing test group — with third-party PBT frameworks referenced through
  the tree's `third_party/` conventions (the same way gtest is integrated),
  never fetched from the network.
- Validated end-to-end on OpenHarmony trunk (master) against ArkUI
  ace_engine: `./build.sh --product-name host_product --build-target
  base_unittest` gated the campaign, the agent added an
  `ace_unittest("geometry_pbt_test")` (`host_components`) target depending on
  `//third_party/rapidcheck:rapidcheck`, and the resulting binary runs
  locally on the build host (x86_64, no emulator or device) — 14 RapidCheck
  properties, `max_success=2000`, all passing.

#### Fixed

- The release workflow provisions its own pinned `gh` CLI when the runner
  host lacks one (a runner migration broke the v0.1.9 build at the notes
  step with every later step skipped).

#### Embedded SDK

- Embedded pi `0.84.4`. `pi-pbt --version` reports
  `pi-pbt 0.1.10 (pi 0.84.4)`.

### 中文

#### 新增

- 新增 `build-run` 子命令:自带构建命令。构建命令是**用户准备好的输入**——
  `pi-pbt build-run --workdir <dir> --build-cmd "<cmd>"` 先执行它(完整日志
  在 `<out>/build.log`):失败则 PBT 根本不启动,进程以退出码 `3` 结束;成功
  则 campaign 在不可变更的"构建契约"下运行——重建只允许复用这条命令(仅可把
  构建目标换成新增的测试目标),重建失败是 STOP 条件、原样记入 `REPORT.md`,
  agent 绝不探索其他编译方式(不跑 `oh-closure.py`、不推导独立编译;此模式下
  campaign 内的引导 steer 相应改写)。
- 构建契约下,新增测试按仓库**官方单元测试方式**接入——用组件自带的 GN
  unittest 模板挂到既有 test group,第三方 PBT 框架走仓库 `third_party/`
  惯例引用(与 gtest 的入库方式一致),绝不从网络拉取。
- 已在 OpenHarmony trunk(master)上对 ArkUI ace_engine 端到端验证:
  `./build.sh --product-name host_product --build-target base_unittest` 作
  门禁,agent 新增 `ace_unittest("geometry_pbt_test")`(`host_components`)
  目标并依赖 `//third_party/rapidcheck:rapidcheck`,产物二进制在构建机本地
  直接运行(x86_64,无需模拟器/设备)——14 条 RapidCheck 性质,
  `max_success=2000`,全部通过。

#### 修复

- release workflow 在 runner 缺少 `gh` CLI 时自带一份钉定版本(此前一次
  runner 迁移让 v0.1.9 的构建死在 notes 步骤、后续步骤全部跳过)。

#### 内嵌 SDK

- 内嵌 pi `0.84.4`。`pi-pbt --version` 显示 `pi-pbt 0.1.10 (pi 0.84.4)`。

## 0.1.9 - 2026-08-28

### English

#### Added

- Native per-language code coverage: campaign builds are instrumented with
  each toolchain's own machinery (gcovr / lcov+genhtml for C/C++, coverage.py,
  cargo-llvm-cov, go cover), every tool renders its own HTML report under
  `pbt-out/coverage/`, and the final REPORT.md cross-checks the ledger's
  coverage claims against what actually executed. On by default, silently
  degrading when tools are absent; `PBT_CODE_COVERAGE=0` disables.
  Cross-compiled OpenHarmony/qemu targets are not instrumented in this
  version (documented limitation).
- Coverage feedback loop: the campaign-scoped `coverage_gaps` tool reports
  documented-behavior functions and line ranges the properties never
  executed, and the Phase 4 contract-surface sweep is driven by it. Sweep
  length is gated by effort tier (`PBT_EFFORT`): quick 0, standard 1,
  thorough unlimited — commit gates stay short.
- `PBT_BARE=1` runs the same binary as plain pi — no orchestration extension,
  bundled skills, or guards — the control arm for scaffold A/B measurement.
- pi-pbt-dev skill: function-level scope and effort tiers for incremental
  PBT; `skill-install` supports the codeagent and chrys hosts.

#### Fixed

- The settle self-checks never engaged in real headless campaigns — three
  defects deep: test outcomes were parsed ANSI-blind (pi's pty colors
  pytest/ctest summaries), a self-check steer's own prompt re-entered
  activation and reset the campaign it was extending, and steers issued at
  settle were torn down by print mode. Self-checks now fire at `agent_end`,
  steered rounds keep their campaign, and the entry holds teardown until the
  extra rounds finish.
- Campaign rules from the leaderboard failure taxonomy: found bugs are not
  the finish line (tier-gated continuation), stateful modules mandate
  `state_machine` properties, and boundary claims are sampled at bound±1.
- An explicitly given deliverable path outranks repo-convention placement,
  and deliverables are written early and incrementally — a timeout no longer
  yields an empty test file.
- Headless `-p` output redirected to a file is no longer truncated at exit
  (the process exits only after stdout drains).
- Congestion-friendly retry defaults for unattended runs: when the user has
  not configured `retry`, settings.json is seeded with ~8.5 min of
  exponential backoff and a 2-minute server-hint cap, so a short provider
  rate-limit window no longer kills a campaign 30–90 s in. Multi-hour
  cooldowns still fail fast.
- Oracle precision: crash claims must reproduce under product flags (host
  `-O0` CMake configures are blocked), `from_chars` leftover tautologies are
  blocked, non-issue findings are filtered from the report, and the scope
  guard confines whole-tree scans.

#### Changed

- The benchmark suite was renamed to pbt-arena (submodule `bench/pbt-arena`),
  gaining PBT-Bench leaderboard adapters, baseline arms, and results.

#### Embedded SDK

- Embedded pi `0.84.3`. `pi-pbt --version` reports
  `pi-pbt 0.1.9 (pi 0.84.3)`.

### 中文

#### 新增

- 原生分语言代码覆盖率:campaign 构建按各工具链自己的机制插桩(C/C++ 用
  gcovr / lcov+genhtml,Python 用 coverage.py,Rust 用 cargo-llvm-cov,Go 用
  go cover),每个工具输出自己的原生 HTML 报表到 `pbt-out/coverage/`,最终
  REPORT.md 会把台账声称的覆盖与实际执行到的代码交叉核对。默认开启,工具缺失
  时静默降级;`PBT_CODE_COVERAGE=0` 关闭。交叉编译的 OpenHarmony/qemu 目标
  本版本不插桩(已在文档注明)。
- 覆盖率反馈闭环:campaign 内新增 `coverage_gaps` 工具,报告性质从未执行到的
  有文档行为的函数与行段,Phase 4 契约面清扫由它驱动。清扫轮数按档位门控
  (`PBT_EFFORT`):quick 0 轮、standard 1 轮、thorough 不限——提交门禁不拉长。
- `PBT_BARE=1` 让同一个二进制以纯 pi 运行——不加载编排扩展、内置 skills 与
  守卫,作为脚手架 A/B 对照臂。
- pi-pbt-dev skill:函数级 scope 与档位,支持增量 PBT;`skill-install` 支持
  codeagent 与 chrys 宿主。

#### 修复

- settle 自检在真实无头 campaign 中从未生效——三层缺陷:测试结果解析不剥
  ANSI(pi 的 bash 走 pty,pytest/ctest 输出带色),自检 steer 的 prompt 重新
  进入激活逻辑并重置了它要延长的 campaign,settle 时发出的 steer 会被 print
  mode 的收尾撕掉。现在自检在 `agent_end` 触发,steer 轮保持原 campaign,
  入口会等额外轮次跑完才退出。
- 来自 leaderboard 失败归因的 campaign 规则:找到 bug 不是终点(按档位
  继续)、有状态模块必须写 `state_machine` 性质、边界声明按 bound±1 采样。
- 显式给定的交付路径高于仓库惯例位置,且交付文件必须尽早、增量写——超时不再
  产生空测试文件。
- 无头 `-p` 输出重定向到文件时不再在退出时被截断(进程等 stdout 排空后才退出)。
- 无人值守运行的拥塞友好重试默认值:用户未配置 `retry` 时,向 settings.json
  种入约 8.5 分钟的指数退避与 2 分钟的服务器指示等待上限,短暂的限流窗口
  不再让 campaign 在 30–90 秒内暴毙;数小时级冷却仍快速失败。
- Oracle 精度:崩溃类结论必须在产品编译选项下复现(拦截宿主 `-O0` CMake
  配置)、拦截 `from_chars` 剩余字符类恒真断言、报告过滤非问题结论、scope
  守卫约束全树扫描。

#### 变更

- benchmark 套件更名为 pbt-arena(submodule `bench/pbt-arena`),新增
  PBT-Bench leaderboard 适配器、baseline 臂与结果。

#### 内嵌 SDK

- 内嵌 pi `0.84.3`。`pi-pbt --version` 显示 `pi-pbt 0.1.9 (pi 0.84.3)`。

## 0.1.8 - 2026-08-18

### English

#### Added

- Added `pi-pbt mcp`, a stdio MCP server built on the official
  `@modelcontextprotocol/sdk`, so coding agents such as Claude Code and Codex
  can delegate campaigns while they keep developing. Six tools — `pbt_start`,
  `pbt_status`, `pbt_report`, `pbt_cancel`, `pbt_watch_start`,
  `pbt_watch_stop` — with campaign artifacts readable as
  `pbt://runs/<id>/...` MCP resources.
- Every delegated run snapshots one immutable commit into a detached git
  worktree with persistent state under `~/.pi-pbt/runs/` (`PI_PBT_RUNS_DIR`),
  a bounded queue (`PBT_MCP_MAX_CONCURRENT`), cancellation, and a captured
  `changes.patch`; the caller's checkout is never touched. Watches supersede:
  a newer commit cancels a pre-Review run, while a run already in Review
  finishes first and pending commits coalesce to the newest.
- Added `pbt_start`'s `include_uncommitted` option: the current uncommitted
  working-tree state is frozen into an immutable snapshot commit through a
  throwaway git index — the caller's index, HEAD, refs, stash, and files stay
  untouched — so code can be reviewed before any commit exists.
- Added this bilingual `CHANGELOG.md` as the single source of release notes:
  GitHub Release notes are generated from the tagged entry, and CI enforces
  that the version and both language sections stay in lockstep.

#### Changed

- Compacted the bundled PBT skills: `pbt-patterns` and `pbt-oracles` are now
  short executable protocols whose framework/generator/oracle/C++/research
  material loads from on-demand reference files, and `pbt-in-practice` became
  one of those references instead of a separately advertised skill.
- Historical `pbt-native/` and `pbt-out/` directories no longer select the
  harness: campaigns must probe the project-owned test framework first, record
  the probe in `PLAN.md`, and may write new fallback harness files only after
  that probe fails.
- Build-failure recovery is now scoped by project type: only campaigns
  detected as OpenHarmony are pointed at `openharmony-build-run` /
  `oh-closure.py`; ordinary repositories are steered to repair their own
  build, once per campaign.

#### Fixed

- The hook-run gate and the MCP run manager now judge campaigns by their
  artifacts and accept both `<out>/` and the SOP-literal `<out>/pbt-out/`
  layouts; a successful campaign could previously be reported as dead, and a
  child that died before Review could masquerade as `bugs_found`.
- Registered the bundled OAuth flows in the Bun single binary, fixing
  `/login` to xAI and the other OAuth providers
  (`Cannot find module './xai.js'`).
- Release downloads of the pinned `fd`/`rg` tools now retry transient network
  failures, and the release mirror no longer depends on a `gh` flag newer
  than the runners ship (`--clobber`), which had left a public release empty.

#### Embedded SDK

- Embedded pi `0.84.2`. `pi-pbt --version` reports
  `pi-pbt 0.1.8 (pi 0.84.2)`.

### 中文

#### 新增

- 新增 `pi-pbt mcp`:基于官方 `@modelcontextprotocol/sdk` 的 stdio MCP
  server,让 Claude Code、Codex 等 coding agent 把测试委派给 pi-pbt、同时继续
  开发。共六个工具 —— `pbt_start`、`pbt_status`、`pbt_report`、`pbt_cancel`、
  `pbt_watch_start`、`pbt_watch_stop`;campaign 产物可通过
  `pbt://runs/<id>/...` MCP resources 读取。
- 每次委派运行都把一个不可变 commit 快照进独立的 detached git worktree,状态
  持久化在 `~/.pi-pbt/runs/`(`PI_PBT_RUNS_DIR`),带串行队列
  (`PBT_MCP_MAX_CONCURRENT`)、取消能力和 `changes.patch` 捕获;调用方的
  工作树完全不被触碰。watch 支持 supersede:新 commit 会取消尚未进入 Review
  的运行;已在 Review 的运行先跑完,积压的 commit 合并为最新一个。
- `pbt_start` 新增 `include_uncommitted`:用一次性 git index 把当前未提交的
  工作树状态冻结成不可变快照 commit —— 调用方的 index、HEAD、refs、stash 和
  文件全部不动 —— 代码不用先 commit 也能提前审核。
- 新增本双语 `CHANGELOG.md` 作为 release notes 的唯一来源:GitHub Release
  notes 由对应版本条目自动生成,CI 强制版本号与中英两节保持同步。

#### 变更

- 精简内置 PBT skill:`pbt-patterns` 与 `pbt-oracles` 改为简短的可执行协议,
  framework/generator/oracle/C++/研究资料拆分为按需加载的 reference 文件;
  `pbt-in-practice` 不再单独注册为 skill,降为其中一份 reference。
- 历史遗留的 `pbt-native/`、`pbt-out/` 目录不再决定 harness 位置:campaign
  必须先实测项目自有测试框架并把探测结果记入 `PLAN.md`,探测失败后才允许
  新建 fallback harness 文件。
- 构建失败的引导按项目类型区分:只有确认为 OpenHarmony 的 campaign 才会被
  指向 `openharmony-build-run` / `oh-closure.py`;普通仓库被引导修复自己的
  构建,且每个 campaign 只提示一次。

#### 修复

- hook-run gate 与 MCP run manager 改为按产物判定结果,并同时兼容 `<out>/`
  与 SOP 字面的 `<out>/pbt-out/` 两种布局;此前成功的 campaign 可能被误判为
  "无 REPORT.md",Review 前死亡的子进程可能伪装成 `bugs_found`。
- 在 Bun 单文件二进制中注册内置 OAuth 流程,修复 `/login` 到 xAI 等 OAuth
  服务商时的 `Cannot find module './xai.js'`。
- release 打包下载固定版本的 `fd`/`rg` 时会对瞬时网络故障重试;镜像脚本不再
  依赖 runner 上不存在的 `gh --clobber` 参数(该问题曾导致公开 release 空无
  一物)。

#### 内嵌 SDK

- 内嵌 pi `0.84.2`。`pi-pbt --version` 输出
  `pi-pbt 0.1.8 (pi 0.84.2)`。

## 0.1.7 - 2026-08-12

### English

#### Security

- Disabled Bun's automatic loading of a scanned repository's `bunfig.toml` in
  every compiled target. A repository-controlled `preload` could previously run
  before pi-pbt's entry point and tool guards. The binary smoke test now verifies
  this against a hostile fixture.

#### Added

- Added `quick`, `standard`, and `thorough` campaign effort tiers, covering time
  budgets, property counts, generated-input counts, deepening rounds, and oracle
  requirements. Select them with `--effort` or `PBT_EFFORT`.
- Added `pi-pbt kea` for config-driven HarmonyOS application testing with Kea2
  and a connected device.
- Added pinned `fd` and `rg` binaries to every release archive, so pi-pbt can run
  on systems where neither search tool is installed.

#### Changed

- Campaigns must use the repository's existing test harness and a real
  language-specific PBT framework instead of creating a parallel harness or a
  hand-written random loop.
- Release artifacts are three DEFLATE-compressed zip archives: Linux x64, Linux
  arm64, and macOS arm64. Each extracts to `pi-pbt-<platform>/` with the binary,
  bundled search tools, notices, and executable permissions intact.

#### Fixed

- Asset extraction now honors the run's `$HOME` under Bun instead of polluting
  the operator's real home directory.
- Binary smoke testing now refuses to validate a stale `dist/pi-pbt`.
- `watch` now forwards coverage mode to campaigns, and dashboard startup errors
  identify the failed port or installation step.
- Release packaging copies staged tools across filesystems and keeps package
  output separate from cross-compiled binaries, avoiding the `EXDEV` and path
  collision failures encountered while cutting this release.

#### Embedded SDK

- Embedded pi `0.84.1`. `pi-pbt --version` reports
  `pi-pbt 0.1.7 (pi 0.84.1)`.

### 中文

#### 安全修复

- 所有编译目标均已禁用 Bun 自动加载被扫描仓库中的 `bunfig.toml`。此前仓库可通过
  `preload` 在 pi-pbt 入口和工具守卫启动前执行代码。binary smoke test 现已使用恶意
  fixture 对该场景做回归验证。

#### 新增

- 新增 `quick`、`standard`、`thorough` 三档 campaign effort，分别约束时间预算、
  性质数量、生成输入数量、深化轮次和 oracle 要求。可通过 `--effort` 或
  `PBT_EFFORT` 选择。
- 新增 `pi-pbt kea`，通过配置文件、Kea2 和已连接设备对 HarmonyOS 应用执行测试。
- 每个平台的 release 压缩包均内置经过校验并固定版本的 `fd` 和 `rg`，系统未安装这
  两个搜索工具时也可运行 pi-pbt。

#### 变更

- campaign 必须使用仓库已有的测试框架和对应语言的真实 PBT framework，不再另建
  平行 harness，也不得手写随机循环冒充性质测试。
- release 产物改为三个使用 DEFLATE 压缩的 zip：Linux x64、Linux arm64 和 macOS
  arm64。每个压缩包解出 `pi-pbt-<platform>/`，其中包含主程序、搜索工具、notice，
  并保留可执行权限。

#### 修复

- Bun 下的资源解压现会遵循本次运行的 `$HOME`，不再污染操作者真实的 home 目录。
- binary smoke test 不再误验旧的 `dist/pi-pbt`。
- `watch` 会把 coverage mode 传给 campaign；dashboard 启动失败时会指出具体端口或
  安装步骤。
- release 打包改为跨文件系统复制暂存工具，并将打包输出与交叉编译二进制分离，避免
  本次发版过程中遇到的 `EXDEV` 和路径冲突失败。

#### 内嵌 SDK

- 内嵌 pi `0.84.1`。`pi-pbt --version` 输出
  `pi-pbt 0.1.7 (pi 0.84.1)`。
