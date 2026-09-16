# Changelog / 更新日志

Every release is recorded here in English and Simplified Chinese. GitHub Release
notes are generated from the matching version entry; the two copies must not be
maintained independently.

每个版本均在此以英文和简体中文记录。GitHub Release notes 从对应版本条目自动生成，
不再单独维护另一份文案。

Entries before v0.1.7 predate this file and remain available in the Release
history. / v0.1.7 之前的版本早于本文件，仍可在 Release 历史中查看。

## 0.1.18 - 2026-09-16

### English

#### Replacement build — 2026-09-16

The v0.1.18 archives were replaced under the same version at the maintainer's
request. Download the ZIP and its checksum again; an older cached archive and
a new checksum are not a matching pair. The version string is unchanged.

- Fix installer discovery of bundled `fd`/`rg`: retain executable-adjacent tools
  by default and honor an explicit agent directory.
- Archive each dev-skill run's `report.json` with its matching `REPORT.md`;
  preserve previous completed results separately and reject nonempty round
  destinations instead of mixing runs. Refresh an installed skill with
  `pi-pbt skill-install --force` after replacing the binary.
- Publish the complete linked user-guide set; introduce structured results before
  CI/MCP references and validate relative document and heading links.

#### Added

- **Structured campaign reports.** `report.json` records the tested revision,
  absolute repository/test paths, bidirectional property-to-bug links and exact
  reproduction commands. Validation runs when the report is written and at the
  terminal gate; an empty successful campaign cannot pass. An honest failed
  build may report zero executed properties and yields hook-run exit 3.
- **Local C++ dependency inventory.** Native campaigns discover existing project
  RapidCheck/GoogleTest dependencies before querying installed dpkg, pacman, RPM
  or Homebrew packages. `dependencies.json` records paths, versions, architecture
  and query diagnostics. Discovery is read-only and offline; configured system
  repositories/internal mirrors precede permission-gated public downloads.
  Host packages are not assumed compatible with an OpenHarmony GN/ARM target.
- **Bounded empty-response recovery.** A normal empty/thinking-only `stop` or
  output-budget `length` response gets one recovery continuation in the same
  awaited pi prompt. Older successful tool text, and oversized recent multipart
  results, are shortened for model requests without modifying the saved session
  or breaking tool-call/result pairs. Recovery prioritizes honest report
  completion; repeated empty responses stop, and cancellation/errors do not
  trigger a new campaign. Configured-window pressure also prioritizes closeout.
- **Guard diagnostics.** Campaign-only `guards.jsonl` records corrective blocks,
  redacts credentials and follows the selected artifact directory. Prior-run
  logs do not leak into a later clean run.

#### Fixed

- Manifest edits preserve existing entries and their containing scopes, reject
  duplicate-key shadowing, and require shell mutations to use checked edits.
  Additive sibling registrations and quoted commands are no longer misclassified.
- Native-build recognition handles `timeout`, environment/locking wrappers,
  compiler caches and supported shell/interpreter flags, instead of rejecting
  real builds because their command starts with a wrapper. Help/version probes,
  arithmetic expressions and invalid shell-builtin chains are not build evidence.
- Managed runs prefer a complete report pair over a stray nested directory and
  count bug evidence across both supported artifact layouts. Worktree JSON paths
  are relocated through parsing/serialization rather than raw substitution.
- OpenHarmony harness detection reads generated build files rather than relying
  only on PLAN prose. `COVERAGE_STATUS.md` is accepted consistently, and Chinese
  empty Design Caveats markers/table skeletons no longer trigger false warnings.
- Project dependency candidates require the correct framework headers. Installed
  library/config paths survive inventory truncation, query failures are not
  silently treated as missing packages, and non-C++ campaigns do not receive
  irrelevant RapidCheck installation advice.

#### Documentation and validation

- OpenHarmony examples now prefer native `host_product`, with an existing
  `base_unittest` preflight before generated PBT targets exist. Device products
  remain a fallback for components without host support and with a configured
  runner; CMake is not presented as an interchangeable OH default.
- Consolidated CLI/skill/MCP entry-point guidance, development versus CI usage,
  generated-test recovery via `changes.patch`, standalone regression and replay
  commands. Clarified that plain `-p` is not a CI verdict, hook-run does not check
  out its SHA, CI needs the parent commit, and watch batches may drop older commits.
- Real ArkUI `standard` campaign verified a wrapped host build, one actual
  `coverage_gaps` call, five properties and a schema-valid report with one finding.
  Empty-response recovery is tested through the public pi SDK with a scripted
  provider; it has not been revalidated against the user's original GLM gateway.
- Automated-review tooling now distinguishes absent reviews from unresolved
  findings, preserves the exact merge SHA, and verifies thread ownership before
  replying/resolving. CI remains a mandatory merge gate.
- Embedded pi remains `0.85.1`; `pi-pbt --version` reports
  `pi-pbt 0.1.18 (pi 0.85.1)`.

### 中文

#### 同版本替换构建 — 2026-09-16

按维护者要求替换 v0.1.18 的发行包，版本号不变。请重新下载 ZIP 和对应校验文件；
旧缓存包不能搭配新校验值。仅看 `--version` 无法区分本次重发前后的包。

- 修复安装后内置 `fd`/`rg` 的发现目录：默认保留主程序相邻的 tools，并遵循显式配置目录。
- dev-skill 的 `report.json` 与本轮 `REPORT.md` 一起归档，旧完成结果单独保留；
  非空轮次目录直接拒绝，不串轮。替换二进制后执行 `pi-pbt skill-install --force`
  刷新已安装 skill。
- 补齐公开仓所引用的用户文档；在 CI/MCP 之前介绍结构化结果，并校验文件和章节链接。

#### 新增

- **结构化战役报告。** `report.json` 记录被测提交、仓库/测试绝对路径、性质与问题的
  双向关联，以及完整复现命令。写入时和最终门禁都会校验；没有执行任何性质的成功
  报告不能通过。诚实的构建失败可以记录零条已执行性质，hook-run 返回退出码 3。
- **本机 C++ 依赖清单。** 原生战役先发现项目已有 RapidCheck/GoogleTest，再查询
  dpkg、pacman、RPM 或 Homebrew 已安装包。`dependencies.json` 记录路径、版本、
  架构及查询诊断。探测只读、不联网；依赖获取优先配置好的系统仓库/内网镜像，
  公网下载放最后且须获准。不会假定 host 系统包兼容 OH GN/ARM 目标。
- **有界空响应恢复。** 正常空/thinking-only `stop` 或输出预算耗尽的 `length`
  响应，只在同一个受等待的 pi prompt 内恢复一次。模型请求会缩短旧成功工具输出
  及过大的近期多文本块结果，不修改持久化会话、不破坏工具调用与结果的配对。
  恢复优先如实完成报告，再次空响应即停止；取消和错误不会重新拉起战役。接近
  配置窗口上限时也优先收尾，不硬编码模型的真实 token 上限。
- **守卫诊断。** 战役专属 `guards.jsonl` 记录纠正性拦截，对凭据脱敏，并跟随产物
  目录；上一轮的日志不会污染下一轮没有拦截的战役。

#### 修复

- Manifest 编辑保留既有条目及所属容器，拒绝重复键覆盖；shell 改写须改走可校验
  的 edit。合法的兄弟对象注册与带引号的命令不再误伤。
- 原生构建识别支持 `timeout`、环境/锁包装器、编译缓存及受支持的 shell/interpreter
  参数，不再因为命令前有包装层就声称“未构建”。版本/帮助查询、算术表达式及无效
  shell 内建命令链不能充当构建证据。
- 托管运行优先选择完整报告对，不被空嵌套目录误导；两种产物布局的 bug 证据均参与
  裁决。Worktree JSON 路径通过解析后重写迁移，避免特殊字符破坏报告。
- OH harness 判断读取真实构建文件，不再只依赖 PLAN 文案。统一放行
  `COVERAGE_STATUS.md`；中文空 Design Caveats 标记与空表骨架不再误报。
- 项目依赖需具备对应框架头文件；清单截断优先保留库/构建配置路径；查询失败不冒充
  未安装；非 C++ 战役不再收到无关的 RapidCheck 安装建议。

#### 文档与验证

- OH 默认示例改为 `host_product`；初次 preflight 使用已有 `base_unittest`，不要求
  尚未生成的 PBT target。设备产品只用于无 host 支持且已有 runner 的组件；不再把
  CMake 描述为可随意切换的 OH 默认方案。
- 整理 CLI/skill/MCP 入口、开发期与 CI 用法、通过 `changes.patch` 取回生成测试、
  固化入库及独立回归/复现命令。明确普通 `-p` 不是 CI 裁决、hook-run 不会检出 SHA、
  CI 需要父提交，以及 watch 的批次上限。
- ArkUI 的真实 `standard` 战役完成带包装的 host 构建，实际调用一次 `coverage_gaps`，
  产生五条性质和含一个问题的合法报告。空响应恢复通过公开 pi SDK 的受控 provider
  测试验证；尚未在用户原始 GLM 网关上复测，不能声称所有 provider 均已实证恢复。
- 评审工具区分“评审缺席”与“已有未解决问题”，合并仍钉住精确 SHA；答复/关闭线程
  前先校验所属 PR。CI 全绿仍是合并硬门禁。
- 内嵌 pi 保持 `0.85.1`；版本输出为 `pi-pbt 0.1.18 (pi 0.85.1)`。

## 0.1.17 - 2026-09-14

### English

#### Added

- **Bulk-rewrite guard.** A full-file `write` to a pre-existing file of 30 lines
  or more that this campaign did not author is blocked and redirected to `edit`.
  Generalizes v0.1.16's manifest allow-list: the defect is not *which file* but
  *how much of someone else's file one tool call destroys*. Four exemptions keep
  the honest cases working — the campaign's own generated trees, generated-test
  naming (a re-run legitimately rewrites last campaign's property test), files
  the campaign created earlier in the session, and files small enough that
  replacing them hides nothing. Corrective, never terminating.
- **Per-phase model routing** (`PBT_PHASE_MODELS`), off unless configured. The
  phase is read off the artifacts, never from an agent-declared transition.
  There is deliberately no built-in routing table: Scan is where the campaign
  extracts the spec clauses that override the oracle, so downgrading it makes
  every downstream property test the wrong law and report a false pass.
- `scripts/pr-review-comments.sh` — the automated-review merge gate. Exits 0
  when the current head has been reviewed with no unresolved P0/P1, 1 while one
  is open, 2 on a usage or API error, and 3 until the current head is reviewed;
  `--wait` polls through that last state and the clean path prints the gated sha
  for `gh pr merge --match-head-commit`.

#### Fixed

- **Artifact-driven checks now look where the campaign actually wrote.** They
  resolved `<cwd>/pbt-out` unconditionally, but both subcommands redirect that
  directory with `--out` and the MCP path points it at the run directory. The
  missing-REPORT steer was therefore dead on both subcommand paths since it
  landed, and the same hardcoding sat in sixteen other places including the
  whole coverage-ledger machinery. Resolution now probes the configured
  directory, its nested `pbt-out/`, and the `<cwd>/pbt-out` fallback, and takes
  the one holding an artifact from **this** campaign — freshness matters because
  `build-run --out` defaults to `<repo>/pbt-out` and only creates it, so a
  repeat run finds the previous campaign's files sitting there.
- The missing-REPORT steer names the resolved absolute path. `hook-run` chdirs
  to its own clean workdir, so an agent following a relative `pbt-out/REPORT.md`
  wrote where the gate does not look and the corrective round still exited 2.
- `hook-run`'s gate accepts a third artifact root, its own cleaned workdir, so a
  campaign whose `REPORT.md` landed in the `<cwd>/pbt-out` fallback is no longer
  reported as died-before-Review. Opt-in, and only `hook-run` opts in: `kea`
  shares the output variable but runs in the persistent SUT root.
- `build-run --out <custom-dir>` now finalizes coverage in that directory.
  Report generation and the execution-evidence append silently targeted
  `<cwd>/pbt-out`. Native profiles are rendered from the instrumentation
  directory bound at the first campaign command, which is not always the
  artifact root the agent ended up using.
- In-place runs sweep their own test binaries' leftovers from the repository
  root; a clean passing campaign no longer hands back a dirty tree.
- A campaign that narrates its report instead of writing `pbt-out/REPORT.md`
  gets one last-chance steer at `agent_end`.
- Incremental scope no longer treats previously generated `*_pbt_test.*` files
  as production code to test.

#### Changed

- **`pbt-native/` is the last resort.** Harness placement has three rungs now:
  extend the repository's existing test target, then build the test tree the
  project *should* have in its own conventional location, and only then fall
  back to a standalone `pbt-native/`.
- Registering the campaign's test target is stated as a targeted `edit` in the
  workflow SOP and in both languages of the `build-run` prompt.

#### Documentation

- `PBT_OUT_DIR` and `PBT_PHASE_MODELS` documented in both installation guides,
  including that the subcommands set `PBT_OUT_DIR` automatically and that users
  must not export it.
- The worktree isolation contract now names `build_cmd` as its one deliberate
  exception, next to the contract it qualifies.
- `AGENTS.md` gains the automated-review merge gate as a cardinal rule, and the
  requirement to write commit messages to a file (backticks in a quoted `-m`
  run as command substitution).

#### Embedded SDK

- Embedded pi `0.85.1` (unchanged). `pi-pbt --version` reports
  `pi-pbt 0.1.17 (pi 0.85.1)`.

### 中文

#### 新增

- **整文件改写守卫。** 对已存在、30 行及以上、且非本轮战役创建的文件做整文件
  `write` 会被挡下并指向 `edit`。这是对 v0.1.16 那份清单白名单的泛化：缺陷不在
  *哪个文件*，而在*一次工具调用毁掉别人文件的多少行*。四个豁免保住了整写确实
  诚实的场合——战役自己的生成树、生成测试命名（重跑时理应整写上一轮的性质
  测试）、本轮早先自己创建的文件，以及小到整写也藏不住东西的文件。纠正性，
  绝不终止运行。
- **按相位路由模型**（`PBT_PHASE_MODELS`），不配置则不启用。相位从产物读出，
  绝不采信 agent 自报的相位切换。刻意不内置路由表：Scan 是战役提取「会覆盖
  oracle 的 spec 条款」的地方，给它降级会让下游每一条性质都在测错误的法则，
  然后报一个假绿。
- `scripts/pr-review-comments.sh`——自动审查的合并门禁。当前 head 已被审且无
  未解决 P0/P1 时退出 0，有未解决时 1，用法/API 错误 2，当前 head 尚未被审时
  3；`--wait` 会轮询到 3 消失，通过时把被审的 sha 打出来供
  `gh pr merge --match-head-commit` 钉住。

#### 修复

- **产物驱动的检查现在会去战役真正写入的地方找。** 它们一律按 `<cwd>/pbt-out`
  解析，而两条子命令都用 `--out` 改了这个目录，MCP 路径更是指向 run 目录。于是
  「口述报告不算交付」那条 steer 自落地起在两条子命令路径上一直是死的，同样的
  硬编码还散在另外十六处，包括整个覆盖率账本机制。现在会依次探查配置目录、
  其下的 `pbt-out/`、以及 `<cwd>/pbt-out` 回退，取其中持有**本轮**产物的那个
  ——要判新鲜度是因为 `build-run --out` 默认就是 `<repo>/pbt-out` 且只创建不
  清理，重复跑会看到上一轮战役的文件还在那儿。
- 缺失 REPORT 的 steer 改为给出解析后的绝对路径。`hook-run` 会 chdir 到自己的
  干净工作目录，agent 照着相对的 `pbt-out/REPORT.md` 写就写到了门禁不看的地方，
  纠正一轮之后仍然退出 2。
- `hook-run` 的门禁接受第三个产物根，即它自己的干净工作目录，于是 `REPORT.md`
  落在 `<cwd>/pbt-out` 回退里的战役不再被判成「Review 前就死了」。该根是选择性
  启用，且只有 `hook-run` 启用：`kea` 共用同一个输出变量，但它跑在持久的 SUT 根。
- `build-run --out <自定义目录>` 现在会在该目录收尾覆盖率。此前报告生成与执行
  证据追加都悄悄指向 `<cwd>/pbt-out`。原生 profile 从首条战役命令绑定的
  instrumentation 目录渲染，那个目录未必是 agent 最终写产物的地方。
- 就地运行会清掉自己测试二进制留在仓库根的产物，干净通过的战役不再交还一棵
  脏树。
- 只口述报告而没写 `pbt-out/REPORT.md` 的战役，会在 `agent_end` 收到最后一次
  纠正。
- 增量范围不再把上一轮生成的 `*_pbt_test.*` 当成待测生产代码。

#### 变更

- **`pbt-native/` 降为最后手段。** harness 落位现在三档：先扩展仓库已有的测试
  目标，再在项目自己的惯例位置建出它本该有的测试树，最后才退到独立的
  `pbt-native/`。
- 战役登记自己的测试目标「只用 `edit` 改那一行」已写进 workflow SOP 和
  `build-run` 的双语提示。

#### 文档

- `PBT_OUT_DIR` 与 `PBT_PHASE_MODELS` 写入两份安装文档，含「子命令会自动设置
  `PBT_OUT_DIR`、用户不要自己 export」的说明。
- worktree 隔离契约旁边点明 `build_cmd` 是它唯一一处刻意的例外。
- `AGENTS.md` 新增自动审查合并门禁军规，以及「提交信息写文件、不要内联」
  （引号内 `-m` 里的反引号会被当成命令替换执行）。

#### 内嵌 SDK

- 内嵌 pi `0.85.1`（未变）。`pi-pbt --version` 输出
  `pi-pbt 0.1.17 (pi 0.85.1)`。

## 0.1.16 - 2026-09-11

### English

#### Added

- **Delegate a campaign with your own build command.** MCP `pbt_start` gains
  `build_cmd` and `build_workdir`: the command runs first and a non-zero exit
  stops the campaign before any agent work, so a broken build costs seconds
  instead of an exploratory build hunt. Because large components (OpenHarmony,
  AOSP, a monorepo subtree) resolve paths, generated headers, and ccache state
  against the real checkout and cannot build from a detached worktree, a run
  with a build command executes **in place, in the repository itself**, with no
  worktree created. Validated end to end against a real ArkUI ace_engine on an
  OpenHarmony trunk checkout: the official `host_product` build command, scope
  limited to one module, campaign passed, artifacts read back over
  `pbt://runs/<id>/...`.
- `pbt_status` accepts `run_id` and `watch_id` as aliases for `id`, so a caller
  that echoes back the field name `pbt_start` returned no longer errors.

#### Changed

- **`pbt-native/` is demoted to the last resort.** Harness placement now has
  three rungs: extend the repository's existing test target, then build the
  test tree the project *should* have in its own conventional location, and
  only then fall back to a standalone `pbt-native/`. A fresh directory rarely
  reproduces the original harness's include paths, compile definitions, link
  closure, and fixtures, so what survives that gap is the shallow surface while
  the properties that need the real environment silently drop out.
- Registering the campaign's test target is stated as a targeted `edit` in the
  workflow SOP and in both languages of the `build-run` prompt, so the runtime
  guard below is the backstop rather than the teacher.

#### Fixed

- **A campaign no longer rewrites a project-owned manifest wholesale.** Observed
  on a live ace_engine run: registering one `build.test` entry round-tripped
  `bundle.json` through a JSON parse and stringify, reindenting 372 lines into
  419 — a diff no component owner would take, burying the one line that
  mattered. A full-file `write` to a register-only manifest (`bundle.json`,
  `BUILD.gn`, `package.json`, `Cargo.toml`, `pyproject.toml`, `CMakeLists.txt`,
  `meson.build`, `Makefile`) that already exists outside the campaign's own
  generated tree is now blocked and redirected to `edit`. The verification
  re-run produced a 2-insertion, 1-deletion diff, with the guard never firing.
- **In-place runs clean up after their own test binaries.** The same re-run
  left five untracked gtest result files in the component root, one per
  invocation. Files matching the generated test naming that appeared or changed
  after the run started are now swept from the repository root; a same-named
  file the user already had is never touched, and the generated test sources
  never match.
- A campaign that narrates its report in the transcript instead of writing
  `pbt-out/REPORT.md` gets one last-chance steer at `agent_end` when the other
  phase artifacts are on disk but the report is not.
- Incremental scope no longer treats previously generated `*_pbt_test.*` files
  as production code to test, wherever they live.
- OpenHarmony close-out guards: a wiki-sourced REPORT can no longer close on
  header-only leftovers, on a missing CMake configuration, or without the real
  production source compiled, and the `Path().write_text` bash route around
  those checks is closed. Close-out on the component's official harness and on
  `*PbtTest` targets is allowed, scoped to harness placement rung 1.

#### Documentation

- `docs/installation.md` and `docs/installation.zh.md` previously asserted the
  worktree isolation contract unconditionally; `build_cmd` is documented as the
  one deliberate exception, next to the contract it qualifies.
- The bundled `pi-pbt-dev` skill's `INSTALL.md` now points at the MCP path
  (`claude mcp add --transport stdio pi-pbt -- pi-pbt mcp`), the six tools, the
  `pbt://` resources, and `build_cmd` for large components.

#### Embedded SDK

- Embedded pi `0.85.1` (unchanged). `pi-pbt --version` reports
  `pi-pbt 0.1.16 (pi 0.85.1)`.

### 中文

#### 新增

- **用自己的构建命令委派一轮战役。** MCP `pbt_start` 新增 `build_cmd` 与
  `build_workdir`：构建命令先跑，非零退出会在任何 agent 工作之前停掉战役，于是
  构建坏了只花几秒，而不是一轮探索式编译。大型组件（OpenHarmony、AOSP、monorepo
  子树）按真实 checkout 解析路径、生成头文件和 ccache 状态，无法在独立 worktree
  里构建，因此带构建命令的运行**就地在仓库本身执行，不创建 worktree**。已在
  OpenHarmony trunk 上的真实 ArkUI ace_engine 完整验证：官方 `host_product`
  构建命令、scope 限定到单个模块、战役通过，产物经 `pbt://runs/<id>/...` 读回。
- `pbt_status` 接受 `run_id`、`watch_id` 作为 `id` 的别名，调用方把 `pbt_start`
  返回的字段名原样回传不再报错。

#### 变更

- **`pbt-native/` 降级为最后手段。** harness 落位现在分三档：先扩展仓库已有的
  测试目标，再在项目自己的惯例位置建出它*本该有*的测试树，最后才退到独立的
  `pbt-native/`。新目录很难复现原 harness 的 include 路径、编译定义、链接闭包
  和 fixture，活下来的只有浅层表面，而需要真实环境的性质会悄悄掉队。
- 战役登记自己的测试目标时"只用 `edit` 改那一行"已写进 workflow SOP 和
  `build-run` 双语提示，让下面那条运行时守卫只当兜底，而不是唯一的老师。

#### 修复

- **战役不再整文件重写项目自有的清单文件。** 真实 ace_engine 运行中观察到：
  为了登记一条 `build.test`，它把 `bundle.json` 走了一遍 JSON 解析再序列化，
  372 行重排成 419 行——这种 diff 没有组件 owner 会收，真正要改的那一行被淹没。
  现在对"已存在且不在战役自有生成树内"的 register-only 清单（`bundle.json`、
  `BUILD.gn`、`package.json`、`Cargo.toml`、`pyproject.toml`、`CMakeLists.txt`、
  `meson.build`、`Makefile`）做整文件 `write` 会被挡下并指向 `edit`。复测的 diff
  是 2 insertions / 1 deletion，守卫一次都没触发。
- **就地运行会清理自己测试二进制的产物。** 同一次复测在组件根留下了五个未跟踪
  的 gtest 结果文件，每跑一次一个。现在仓库根下"名字符合生成测试命名**且**在
  运行开始之后出现或改动"的文件会被清掉；用户原本就有的同名文件绝不碰，生成的
  测试源码也永不匹配。
- 战役若只在对话里口述报告而没写 `pbt-out/REPORT.md`，在其他阶段产物已落盘而
  报告缺失时，会在 `agent_end` 收到最后一次纠正。
- 增量范围不再把上一轮生成的 `*_pbt_test.*` 当成待测生产代码，无论它落在哪。
- OpenHarmony 收尾守卫：来自 wiki 的 REPORT 不能再靠只剩头文件的残留收尾、
  不能在缺 CMake 配置时收尾、不能在没编译真实生产源码时收尾，绕过这些检查的
  `Path().write_text` bash 路径也已封掉。用组件官方 harness 和 `*PbtTest` 目标
  收尾是允许的，范围限定在 harness 落位第一档。

#### 文档

- `docs/installation.md` 与 `docs/installation.zh.md` 原先无条件宣称 worktree
  隔离契约；`build_cmd` 作为唯一一处刻意的例外，已写在该契约旁边。
- 随包分发的 `pi-pbt-dev` skill 的 `INSTALL.md` 现在指向 MCP 路径
  （`claude mcp add --transport stdio pi-pbt -- pi-pbt mcp`）、六个工具、
  `pbt://` 资源，以及大型组件用的 `build_cmd`。

#### 内嵌 SDK

- 内嵌 pi `0.85.1`（未变）。`pi-pbt --version` 输出
  `pi-pbt 0.1.16 (pi 0.85.1)`。

## 0.1.15 - 2026-09-11

### English

#### Added

- Property seeds from the project's own tests: Phase 1 now mines every test
  covering a candidate function and generalizes each example assertion to its
  input domain — a one-literal round-trip becomes a round-trip over all
  inputs, three boundary cases become a bound over the range, an asserted call
  sequence becomes a state-machine trace. The origin is recorded as
  `Seed: <test file:line>` in the property ledger, and the campaign then
  widens the domain past what the examples covered. Seeds capture author
  intent; they never serve as oracle evidence for the function under test.

#### Changed

- **OpenHarmony property tests now land in a sibling `test/pbt/` tree**, not
  inside the unit-test group. OH organizes test kinds as parallel directories
  (`test/unittest/`, `test/fuzztest/`, `test/moduletest/`, …), each with its
  own BUILD.gn group registered in `bundle.json` `component.build.test`;
  property tests are one more kind, carrying their own framework dependency
  and compile flags. Campaigns write `test/pbt/<subpath>/` mirroring
  `test/unittest/<subpath>/`, define targets with the component's own unittest
  template plus `//third_party/rapidcheck:rapidcheck`, aggregate them as
  `group("pbt")`, and add one line to `build.test`. A corrective runtime guard
  redirects the old placement. Validated on ArkUI ace_engine: 19 properties,
  `max_success=2000`, all passing, built by bare target name `pbt` and run
  natively on the build host; the pre-existing unit-test group was untouched.
- **Exploratory OpenHarmony build recipes are retired — builds are
  command-driven.** The `project-build` skill (dependency-closure derivation,
  standalone-subset builds, host-compile with framework stubs, and
  `oh-closure.py`) has been removed, and `openharmony-build-run` was rewritten
  from a fidelity-ordered cookbook into a deck of official build command cards:
  `host_product` native host tests first (measured: ace_engine's aggregate test
  group yields 200 host binaries, no emulator or device needed), the
  component's own test target under a device product second, qemu-user to run
  arm artifacts. When no card applies, the campaign STOPs and asks for a build
  command instead of deriving one.

#### Fixed

- Close-out and property-IR steers now fire once per issue class per campaign
  instead of re-nagging every turn as ledger names change (110 redundant
  steers observed across 25 campaigns).
- The release workflow and mirror script follow the repository renames
  (development repo `FMBot-PBT`, release mirror `FMBot-PBT-release`).

#### Embedded SDK

- Embedded pi `0.85.1` (unchanged). `pi-pbt --version` reports
  `pi-pbt 0.1.15 (pi 0.85.1)`.

### 中文

#### 新增

- 用项目自带测试作性质种子:Phase 1 现在会读取覆盖候选函数的每个测试文件,
  把 example 断言泛化到其输入域——单点 round-trip 变成全域 round-trip,
  0/1/max 三个边界用例变成整段区间的界性质,带中间断言的调用序列变成状态机
  轨迹。来源以 `Seed: <测试文件:行号>` 记入性质账本,随后战役再把域拓宽到
  用例未覆盖的地方。种子表达作者意图,绝不充当被测函数的 oracle 证据。

#### 变更

- **OpenHarmony 的性质测试改为落在与 `test/unittest` 并排的 `test/pbt/`
  目录**,不再塞进单测 group。OH 本就按测试*种类*并排组织目录
  (`test/unittest/`、`test/fuzztest/`、`test/moduletest/` …),每种一个
  BUILD.gn group 并登记在 `bundle.json` 的 `component.build.test`;性质测试
  是又一个种类,带有自己的框架依赖与编译选项。战役写
  `test/pbt/<子路径>/` 镜像 `test/unittest/<子路径>/`,用组件自带的 unittest
  模板 + `//third_party/rapidcheck:rapidcheck` 定义目标,聚合为
  `group("pbt")`,并在 `build.test` 增加一行。运行时守卫会把旧落位纠正过来。
  已在 ArkUI ace_engine 上验证:19 条性质、每条 `max_success=2000` 全部通过,
  用裸组名 `pbt` 构建并在构建机本地直接运行;既有单测 group 零改动。
- **OpenHarmony 的探索式构建配方退役——构建改为命令驱动。** `project-build`
  skill(依赖闭包推导、自含子集构建、host 编译加框架 stub,以及
  `oh-closure.py`)已整体删除;`openharmony-build-run` 从按保真度排序的配方
  手册重写为官方构建命令卡片:优先 `host_product` 本地测试(实测:ace_engine
  聚合测试组产出 200 个 host 二进制,无需模拟器或设备),其次设备产品下组件
  自带的测试目标,再以 qemu-user 运行 arm 产物。无卡片可用时战役直接 STOP
  并请用户给出构建命令,而不是自行推导。

#### 修复

- 收尾与性质 IR 的引导现在按问题类别每场战役只触发一次,不再因账本名称变化
  而每轮重复唠叨(25 场战役中观察到 110 次冗余引导)。
- release workflow 与镜像脚本同步仓库改名(开发仓 `FMBot-PBT`,发布镜像
  `FMBot-PBT-release`)。

#### 内嵌 SDK

- 内嵌 pi `0.85.1`(未变)。`pi-pbt --version` 显示 `pi-pbt 0.1.15 (pi 0.85.1)`。

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
