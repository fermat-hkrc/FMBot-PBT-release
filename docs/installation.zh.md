# 安装 PBT Agent

[English version / 英文版](installation.md)

pi-pbt 是一个 AI agent:你把它指向一个代码仓库,它自己读代码、想出这段代码
**应该始终成立的规律**,写出测试去大量随机地验证这些规律,把发现的 bug 连同
最小复现一起写成报告。这种测法叫**性质测试**(property-based testing,PBT)。

下载对应平台的发行包即可安装。主程序是自包含二进制，不需要 Node.js/Bun；
压缩包还包含安装脚本和搜索工具。被测项目仍须准备自己的工具链（见 §1）。

## 1. 系统要求

pi-pbt 自己没有依赖。但它会**真实编译并运行它写出来的测试代码**,所以被测
语言的工具链必须先装好:

| 被测目标 | 需要预装 |
|---|---|
| 任何仓库 | `git`(要读提交历史和改动) |
| C / C++ | C++17 编译器（`clang`/`g++`）和项目本身的构建工具（`make`、`ninja`、GN 等）；CMake 项目需 `cmake` ≥ 3.16。GoogleTest/RapidCheck 的准备方式见下文。 |
| Python | `python3` ≥ 3.9 + `pip`(会在探测到的环境里安装 `hypothesis`) |
| Rust | `cargo`(会加 `proptest` 开发依赖) |
| Go | `go` 工具链 |
| Java | JDK + Maven/Gradle(jqwik) |
| OpenHarmony 组件 | 一份已配置好的 OpenHarmony 源码树及官方构建依赖。优先用原生 `host_product` 测试；只有组件没有 host 目标时才需要已经配置好的 device-product runner。 |
| 真机上的 HarmonyOS 应用 | `hdc`（HarmonyOS 设备连接工具）以及一份带虚拟环境的 Kea2 —— 见实验性的 [`pi-pbt kea` 指南](kea.zh.md) |

Debian/Ubuntu(C++ 目标)示例:

```bash
sudo apt-get install -y git cmake ninja-build clang
```

Arch Linux:

```bash
sudo pacman -S --needed git cmake ninja clang
```

### 准备 PBT 依赖

C++ campaign 会把本机可用的 RapidCheck 和 GoogleTest 信息写入
`pbt-out/dependencies.json`。campaign 会按以下顺序逐级尝试，取第一个成功的：
**项目自带依赖 → 已安装的系统包 → 已配置的系统仓库或企业内网仓库 → 把框架源码
vendor 进测试脚手架**。这个顺序可以避免不必要的联网和重复副本。

缺依赖不会让 campaign 停下。agent 会自行获取，不需要你授权：系统包管理器一律以
已是 root 时直接执行，需要提权时才加 `sudo -n`，绝不交互，因此密码提示不可能卡住
无人值守的运行；语言包管理器装在用户态；源码 vendor 完全不需要 root，在受限机器上
往往正是唯一能走通的一级。vendor 进来的框架会固定到记录在案的 revision 并在编译前
校验，campaign 不会去构建上游默认分支当下碰巧是什么。它不会往你的系统里添加新的
软件源。若每一级都确实失败（内网隔离、
没有 sudo、拿不到源码），campaign 仍会跑完，改用自写驱动。工作流要求该驱动在测试
文件顶部标注 `PBT-FALLBACK: <原因>` 并在 `REPORT.md` 写明同一原因；运行时会在能识别
出手写随机数的地方强制这一点（`std::mt19937`、`random.randint`、`Math.random()`
以及各语言的等价物——这正是"拿自写生成器顶替框架"的实际形态）。

清单会区分包路径、架构与目标是否真正兼容。给宿主机 ABI 安装的库不能链接到 ARM
目标；OpenHarmony 的 `host_product` 构建也可能仍须使用与自身工具链兼容的源码，
例如 `third_party/rapidcheck`。因此 `dependencies.json` 只作为依赖准备的依据，
不表示该依赖已经验证为可成功链接。

## 2. 下载与安装

从你获取 pi-pbt 的渠道(Releases 页面、内部镜像或直接分发)取得对应平台的
压缩包,每个都附带 `.sha256` 校验文件。已发布的文件为:

| 平台 | 文件 | 下载体积 |
|---|---|---|
| Linux x64 | `pi-pbt-linux-x64.zip` | 约 42 MiB |
| Linux arm64(aarch64) | `pi-pbt-linux-arm64.zip` | 约 42 MiB |
| macOS Apple Silicon | `pi-pbt-macos-arm64.zip` | 约 31 MiB |

不确定该拿哪个 Linux 版本?跑 `uname -m` —— 显示 `x86_64` 用 x64 那个,显示
`aarch64` 用 arm64 那个。

每个压缩包解出来都是一个同名目录，其中包含 `pi-pbt`、安装脚本 `install.sh`、同级
`tools/fd` 和 `tools/rg`。运行随包安装脚本即可安装主程序，并把工具放到嵌入式 pi
可发现的目录：默认与安装后的主程序相邻，位于 `/usr/local/bin/tools`；显式设置
`PI_CODING_AGENT_DIR` 时使用 `$PI_CODING_AGENT_DIR/bin`。

```bash
PLATFORM=linux-x64   # 或 linux-arm64 / macos-arm64
sha256sum -c "pi-pbt-${PLATFORM}.zip.sha256"   # 可选的完整性校验
unzip "pi-pbt-${PLATFORM}.zip"
cd "pi-pbt-${PLATFORM}"
./install.sh
```

**v0.1.18 替换构建（2026-09-16）：**当前发行包已包含上述安装修复。此前下载过
v0.1.18 的用户，请重新下载 ZIP 和 `.sha256` 并运行 `./install.sh`。版本号未变，
请用当前发行包校验值区分新旧包。已安装宿主 agent skill 的，还须执行
`pi-pbt skill-install --force` 刷新。

普通 shell 中直接执行 `fd` 或 `rg` 仍可能提示找不到，这是正常的，无需额外修改
`PATH`。在 Windows 上解压会丢掉 Unix 权限位；若文件中转过 Windows 机器，先给
`pi-pbt`、`install.sh`、`tools/fd` 和 `tools/rg` 执行 `chmod +x`。

arm64 和 macOS 版都是在 x64 Linux 上交叉编译出来的,只做了格式校验(发布流程
会断言产物确实是 aarch64 ELF / Mach-O arm64),没有在目标机器上实跑，不能据此保证目标平台的运行行为。若遇到问题，请向发行方反馈
OS/架构、`pi-pbt --version` 和启动日志。

macOS 还需清除 Gatekeeper 隔离标记:

```bash
xattr -d com.apple.quarantine pi-pbt
```

验证:

```bash
pi-pbt --help          # 打印用法
pi-pbt --list-models   # 配好模型服务商后列出可用模型
```

首次运行时 pi-pbt 会创建自己的配置目录 `~/.pi-pbt/agent/`(存放登录凭据、
模型配置和历史记录),与 `pi` 自己的 `~/.pi/agent/` 有意分开、互不影响。

### 安装 `pi-pbt-dev` skill（可选）

发行包自带一个 skill,让日常编码 agent(Claude Code / opencode / Codex)能通过
pi-pbt 对代码跑**全量/增量 PBT 战役**。skill 文件就随发行包内置(内嵌在二进制里),
安装只需要一条命令:

```bash
pi-pbt skill-install                 # 自动检测已安装的 agent
pi-pbt skill-install --host claude   # 或 opencode / codex / codeagent / chrys / project
pi-pbt skill-install --dir /自定义/路径  # 任意 skill 目录
pi-pbt skill-uninstall               # 卸载（参数同上）
```

它把 `pi-pbt-dev` skill(协议文本 + 辅助脚本)拷入宿主 agent 的 skill 目录——
一个薄壳,运行时从 PATH 解析 `pi-pbt` 二进制,无需其他配置。更新 pi-pbt 后重跑
`skill-install --force` 即可刷新；卸载默认只处理检测到的用户级宿主，项目/自定义安装
请复用原来的 `--host` 或 `--dir` 选择。
之后直接对你的 agent 说"测一下我刚才的改动"即可,
完整教程见 [宿主 agent skill 用法](skill-pi-pbt-dev.zh.md)。

### 升级已有安装

1. 停止正在运行的战役。重新下载新版 ZIP **和对应校验文件**，解压到新目录、校验，
   然后运行该包的 `./install.sh` 替换主程序。
2. 如果给宿主 agent 安装过 `pi-pbt-dev`，替换二进制后**必须刷新复制出去的 skill**。
   内置战役技能随二进制更新，但宿主 agent 目录里的副本不会自动更新。

   ```bash
   pi-pbt skill-install --force                       # 自动检测的用户级宿主
   pi-pbt skill-install --host claude --force         # 原来指定的单个宿主
   pi-pbt skill-install --host project --force        # 在各相关项目目录执行
   pi-pbt skill-install --dir /custom/skills --force   # 使用原来的自定义路径
   ```

   按原安装方式选一条，不是全部执行。`--force` 会替换 skill 文件，手工修改过的内容
   请先备份；刷新后让宿主 agent 重新加载技能或重启会话。
3. 检查 PATH 上实际使用的版本和模型配置：

   ```bash
   command -v pi-pbt
   pi-pbt --version
   pi-pbt --list-models
   ```

保留 `~/.pi-pbt/agent/` 中的凭据、模型配置和历史，不需要为升级删除它。同版本替换
包（如重发的 v0.1.18）请按当前下载校验值区分，不能只看版本字符串。

## 3. 配置模型

### 先说最重要的:用你手里最强的模型

这一条比其他任何配置都重要。整个流程里最难的一步是**提炼性质** —— 从代码里
想明白"这段代码无论输入什么都应该满足什么"。这一步没有标准答案,全靠推理:

- **强模型**能提炼出真正有约束力的性质(比如"编码再解码必须还原成原值"
  "无论怎么并发调用,余额都不会变成负数"),这类性质才可能揪出真 bug。
- **弱模型**只会写出"调用了不崩溃就算过"这种空性质。测试全绿、报告漂亮,
  但一个 bug 也找不到 —— 而且你很难从结果看出它其实什么都没测。

拿什么当"正确答案"来判断结果对错、以及失败之后定位是代码的错还是测试的错,
同样吃推理。所以**别为了省钱在这里用小模型**:省下的推理会直接变成漏掉的 bug。

### 怎么配

pi-pbt 用 [pi](https://github.com/earendil-works/pi) 作为底层引擎,模型和
服务商的配置方式与 pi 完全一致 —— 唯一区别:**配置目录是 `~/.pi-pbt/agent/`
而不是 `~/.pi/agent/`**。pi 文档里凡是写 `~/.pi/agent/<文件>` 的地方,请读作
`~/.pi-pbt/agent/<文件>`。

最快的两条路:

**用环境变量里的 API key** —— 立即可用:

```bash
export ANTHROPIC_API_KEY=sk-ant-...   # 或 OPENAI_API_KEY、GEMINI_API_KEY 等
pi-pbt --list-models
```

**交互式登录** —— 启动 `pi-pbt` 后,用 `/login` 登录(支持订阅账号),
`/model`(Ctrl+L)选模型。

其余细节直接看 pi 的官方文档:

- [服务商与登录](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/providers.md) —— 内置服务商、API key、订阅账号登录
- [模型与 `models.json`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/models.md) —— 添加自定义模型或服务商(兼容 OpenAI/Anthropic/Google 接口的都行;文件放 `~/.pi-pbt/agent/models.json`)
- [自定义服务商](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/custom-provider.md) —— 自定义接口或 OAuth

通过 CLIProxyAPI 使用 GLM、DeepSeek，请看[模型配置示例](models-glm-deepseek.zh.md)。
示例列出来源提交、渠道上下文窗口、最大输出与日常请求预算；不要给所有模型/代理统一
填 1M 上下文。

## 4. 开始测

**全部运行方式、以及各自适合哪个阶段**(和另一个 coding agent 一起开发,还是 MR
流水线)汇总在 [subcommands.zh.md](subcommands.zh.md#全部运行方式) 的一张表里;
被同一个词"skill"指代的三种东西也在那份文档里区分清楚。下面讲的是第一次运行。

### 交互式

`cd` 进被测仓库,直接启动:

```bash
cd /path/to/your/repo
pi-pbt
```

然后直接提要求:

```text
对当前仓库做性质测试(PBT):找出最值得测的目标,写性质并运行,产出 bug 报告。
```

它会依次完成扫描、计划、测试、复核四步；这样的一次完整运行称为 **campaign（战役）**。

### 运行后看什么

完成一次**源码性质测试战役**后，默认在 `pbt-out/` 查找下列产物。指定了 `--out` 就去
所选目录；MCP 托管运行的产物位置由 `pbt_status` / `pbt_report` 返回。

| 产物 | 谁看、有什么用 |
|---|---|
| `REPORT.md` | 用户先看这份：被测范围、结果、问题、限制和复现命令。 |
| `report.json` | **v0.1.18 新增的结构化战役结果**，供 CI、MCP 和其他程序读取；包含 `schemaVersion`、被测提交、构建结果、性质、问题和统计。每个问题都关联发现它的性质与复现命令。 |
| `bug_reports/*.md` | 每个确认的问题一份详细报告，包含反例与重放步骤。 |
| `PROPERTIES.md` / `PLAN.md` | 性质清单与战役进度，便于检查测了什么、哪些还没完成。 |
| `COVERAGE.md` / `COVERAGE_STATUS.md` | 函数/测试进度登记，不是原生行覆盖率百分比；见[覆盖率跟踪](coverage-tracking.md)。 |

`report.json` 是**运行产物，不是用户开跑前要填的配置**。两份报告由 agent 编写，pi-pbt
校验 JSON 契约；并没有把整个 Markdown 报告从 JSON 自动渲染出来。缺失或截断的报告
表示未完成，不能当成“零问题”。字段解释见[报告 schema](report-schema.md)；重放失败、
取回生成用例及固化入库见[复现与回归指南](reproducing.md)。

生成的测试位于项目自己的测试树，不在 `pbt-out/`。MCP worktree 模式要先取回
`changes.patch` 再应用、审阅测试；就地模式的测试已经留在 checkout 中。Kea GUI
运行使用另一套[产物格式](kea.zh.md#产物与失败)。

诊断文件按需出现：`dependencies.json` 是 C++ 依赖清单，`guards.jsonl` 记录工具拦截，
`recovery.json` 记录恢复尝试；缺少这些可选诊断本身不表示战役失败。

全部子命令(`build-run`、`hook-run`、`watch`、`replay`、`scan` …)见
[subcommands.zh.md](subcommands.zh.md)([English](subcommands.md))。

### 命令行方式（无人值守脚本）

```bash
pi-pbt -p "对当前仓库做性质测试(PBT),产物写到 pbt-out/。"
```

`-p` 跑完退出、不等待终端输入，适合无人值守生成；但其退出码**不是战役裁决**。
需要在发现 bug 或报告不完整时让 CI 失败，请用 [hook-run](#ci--git-hook-集成)。

你不需要按项目类型挑选什么模式:入口只有一个,后面的事它自己判断。目标无法
用简单的 `cmake`/`cargo`/`pytest` 构建时(比如大型操作系统或 monorepo 里的
一个组件),它会自己把这个组件先立起来。OpenHarmony 的 C++ 组件走官方构建路径,
并优先检查原生 `host_product` 测试。只有组件没有 host 目标、且 workspace 已经为
对应产物配置 runner 时才退到 device product;不是所有组件都支持 host 构建。
判断依据是仓库内容本身(例如 `@ohos/` 的 `bundle.json`)。

若想在脚本里把入口写死,首条消息以 `/skill:pbt-workflow` 开头:

```bash
pi-pbt -p "/skill:pbt-workflow 对当前仓库做性质测试(PBT),目标是找出 bug。产物写到 pbt-out/。"
```

### 挖多深:effort 档位

一次 campaign 跑在三档之一。档位决定了 wall-clock 预算、写多少条性质、每条
性质拿多少组输入去打、首批性质全过算不算收工,以及难编译的目标能不能换成
容易的。

| | `quick` | `standard` | `thorough` |
|---|---|---|---|
| Wall-clock | ≈10 分钟 | ≈30 分钟 | 不设上限 |
| 每个目标的性质条数 | 3–5 | 5–8 | 按目标行为需要,不设下限 |
| 每条性质的输入组数 | 框架默认(约 100) | ≥1000 | ≥10000 |
| 首批性质全过 | 可以收工 | 必须加强并重跑 ≥1 轮 | ≥2 轮 |
| 蜕变 / 差分性质 | 可选 | 至少一条 | 目标支持就都必须做 |
| 难编译时换个容易的目标 | 允许 | 允许,但要在报告里写明 | 禁止 |
| 契约面清扫(coverage 缺口驱动) | 无 | 1 轮 | 扫到覆盖完或预算耗尽 |

默认值按入口区分：`hook-run` 检查单次提交，默认 `quick`；`watch` 同样默认
`quick`，但每次轮询有批次上限，见下文。其他战役入口默认 `standard`。

单次运行用 `--effort`,整个环境用 `PBT_EFFORT`:

```bash
pi-pbt hook-run <sha> --repo /path/to/repo --effort thorough
pi-pbt watch --repo /path/to/repo --effort standard
PBT_EFFORT=thorough pi-pbt -p "/skill:pbt-workflow ..."
```

什么时候用 `thorough`:找到 bug 比早点跑完更重要的场合 —— 发版候选、
安全相关模块、或者构建本身就很贵的组件。这一档没有时间上限、没有工具调用
上限,并且**禁止**agent 在真实目标不好编译时偷偷换一个容易的目标。

这跟"用多强的模型"是两回事:档位决定它搜索得多充分,模型决定它写出的性质
有多锋利。弱模型跑 `thorough` 依然只会写出弱性质 —— 见前面选模型那节。

### 跑多宽:并行度

pi-pbt 会把它驱动的每个测试运行器和构建,都按机器**当前空闲的核心数**来定
并行度(总核心数减去已有负载)。它在执行命令前设置各框架自己的标准环境变量,
所以无论 agent 怎么调用都生效 —— 包括经由 Makefile 或包装脚本间接调用:

| 自动设置的变量 | 作用 |
|---|---|
| `CTEST_PARALLEL_LEVEL` | `ctest` 同时跑多少个测试(它默认是串行的) |
| `CMAKE_BUILD_PARALLEL_LEVEL`、`MAKEFLAGS` | `cmake --build` / `make` 的并行度 |
| `RUST_TEST_THREADS` | `cargo test` 的线程数 |
| `GOFLAGS`(`-p=N`) | `go test` 的包级并行度 |
| `PYTEST_ADDOPTS`(`-n N`) | `pytest` 的 worker 数 —— 仅在装了 `pytest-xdist` 插件时设置 |
| `PBT_TEST_JOBS` | 解析出的数值本身,供没有自己环境变量的运行器使用(`ninja -j"$PBT_TEST_JOBS"`、gradle、maven) |

这是有意双向的:`ctest` 和 `pytest` 会**变快**(它们默认一次只跑一个);而
`cargo test`、`go test` 和构建会被**限流**(它们默认吃满所有核心,否则会和机器
上其他事情抢资源)。你自己已经设过的值不会被覆盖。

用 `PBT_TEST_JOBS` 覆盖:填数字就固定,`max` 用满所有核心,`auto`(默认)是空闲
核心数。在容器里会遵守 cgroup 的 CPU 配额,而不是宿主机的核心数。

`PBT_TEST_JOBS=1` 同时是排查工具:并行执行会暴露出属于**测试本身**的不稳定
(两个用例抢同一个端口、xdist 不安全的 fixture)。campaign 被要求在把任何失败
判定为 bug 之前先串行重跑一次,并在报告里写明结论来自哪一次运行。

### 代码覆盖率报表

campaign 还会测量**代码到底执行了哪些**,用的是每种语言自带的覆盖率工具,并把
该工具**原生的 HTML 报表**放在 `pbt-out/code-coverage/` 下,再用一个
`index.html` 把它们串起来。

| 语言 | 使用的工具 | 需要 |
|---|---|---|
| C/C++ | 在 `--coverage` 构建上跑 `gcovr`(或 `lcov` + `genhtml`) | `gcovr` 或 `lcov` |
| Rust | `cargo llvm-cov` | `cargo-llvm-cov` |
| Python | 通过 `pytest-cov` 调 `coverage.py` | `pytest-cov` |
| Go | `go tool cover` | Go 工具链 |
| JavaScript/TypeScript | 直接采纳你的运行器已经产出的报表(c8、vitest、jest) | 该运行器的覆盖率插件 |
| Java | 采纳构建产出的 JaCoCo 报表 | 构建里配好 JaCoCo |

插桩是在 campaign 构建任何东西**之前**,把各工具链的编译选项加进环境来打开的
(覆盖率是构建期决定的,事后补不上)。你自己设过的值一律保留不动。工具缺失、
构建没插桩,报表里就如实写明这一行没测到:**覆盖率永远不会让 campaign 失败。**

覆盖率需要目标确实插桩，且收集器能读取生成的 profile。设备/模拟器运行时，应安排
把数据取回 host；文件缺失不能解释为零覆盖。`host_product` 更便于本地执行，但测试
成功本身不证明已经生成可用的覆盖率数据。

### 报表给 `REPORT.md` 补上了什么

`COVERAGE.md` 记的是 campaign **自己声称**为哪些函数写了性质测试;覆盖率记的是
**实际跑了什么**。报表把两者交叉起来:

- *executed* —— 声明有执行证据支撑
- *claimed but never executed* —— 该性质根本没碰它挂在名下的那个函数,这是真正
  值得处理的发现
- *no evidence* —— 这个符号在所有报表里都没出现,通常是目标没被插桩,而不是声明
  作假

整个功能可以用 `PBT_CODE_COVERAGE=0` 关掉。

### 先构建再测试:`build-run`(自带构建命令)

对于你已经知道怎么构建的项目——OpenHarmony 组件、大型 monorepo、任何有官方构建
入口的仓库——把构建命令交给 pi-pbt,由它做 campaign 的门禁。
需要边跑边看日志，直接看本节的 [build-run 实时 JSON 日志](#build-run-实时-json-日志)。

OpenHarmony 默认优先走原生 `host_product`。下面这条 ArkUI `ace_engine`
preflight 已经在真实源码树上验证过;它先构建**现存**的 `base_unittest` group,
因此能在 campaign 写任何东西之前证明真实 SUT 可编译:

```bash
export PBT_OH_WORKSPACE=/path/to/openharmony
pi-pbt build-run \
  --workdir "$PBT_OH_WORKSPACE" \
  --repo "$PBT_OH_WORKSPACE/foundation/arkui/ace_engine" \
  --build-cmd "./build.sh --export-para PYCACHE_ENABLE:true --product-name host_product --build-target base_unittest --ccache --no-prebuilt-sdk" \
  --scope frameworks/base/geometry \
  --lang zh
```

`host_product` 在 Linux x86_64 上原生运行测试,产物目录是
`out/host/host_product`(不是 `out/host_product`),受支持的组件不需要模拟器或
设备。但它并不覆盖所有 OpenHarmony 组件。只有组件没有 host 目标时才退到 device
product,而且 workspace 必须已经给该 product 的产物配好 runner;不要把 device 构建
描述成 host 路径的通用替代品。

**加快 OH 重复构建。** `--ccache` 已经是 hb 的默认值，写出来只是显式声明。实测有效的
是 `--fast-rebuild`：它跳过 prepare/preloader/loader/gn，直接从 ninja 开始，同一次无改动
的 `base_unittest` 增量构建，不加是 **27 秒**，加上是 **13 秒**。

但不要把它放进上面的 preflight。OH 官方 help 限定它只能用于"gn 相关脚本没有改动"的构建，
而战役恰好会改这些文件：新增 `test/pbt/<子路径>/BUILD.gn`，并在 `bundle.json` 的
`build.test` 里注册目标。因此它只适合 gn 输入未变的重复构建；战役生成测试目标后的第一次
重建必须去掉它：

```bash
./build.sh --export-para PYCACHE_ENABLE:true --product-name host_product \
  --build-target base_unittest --ccache --no-prebuilt-sdk --fast-rebuild
```

有两个参数不要动：`--jobs` 在 hb 里已标记 deprecated；`--load-test-config`（默认开启）
正是读取 `bundle.json` 中 `test` 字段的开关，关掉它会让生成的 PBT 目标不被加载。
OH 文档列出的 `--gn-args enable_notice_collection=false` 面向出镜像的全量构建，在这个
单元测试 preflight 上实测是 26 秒对 27 秒，且改 gn args 会触发一次 gn 重新生成，不划算。

如果你的 HarmonyOS 源码树用的是 `build_system.sh` 而不是 `build.sh`，先对该脚本执行
`--help` 确认参数是否存在：这里没有验证过它的参数集，不被支持的参数只会让 preflight
直接失败。

初次 `--build-cmd` 是 preflight,必须指向 campaign 开始前就存在的目标,例如
`base_unittest`。不要拿尚未生成的 `pbt`、`<name>_pbt_test` 等目标做初次门禁。
campaign 生成测试并按组件自己的 GN test family 完成登记后,重建才可以沿用同一条
`build.sh` 命令、把目标换成新的 `pbt` target。

`build-run` 仍然完全支持 CMake,但它是独立的普通项目示例,不是 OpenHarmony 的
等价构建路径。例如项目现有 `CMakeLists.txt` 已定义 `calc_test` 时,就用它做门禁:

```bash
pi-pbt build-run \
  --workdir /path/to/cmake-project \
  --repo /path/to/cmake-project \
  --build-cmd "cmake -S . -B build && cmake --build build --target calc_test -j$(nproc)" \
  --scope src/calc.cpp \
  --lang zh
```

构建命令是**你准备好的输入**,不是让 agent 去摸索的东西。`build-run` 先在
`--workdir` 里执行它(完整日志在 `<out>/build.log`):失败则 PBT 根本不启动,
进程以退出码 `3` 结束——修好构建或命令后重跑;成功则 campaign 在 `--repo` 下
以"构建契约"运行:重建只允许复用这条命令(需要时只能把现存目标换成新生成的测试
目标),重建失败是 STOP 条件、原样记入 `REPORT.md`。agent 不会切换构建系统或推导
替代编译方式。`--scope` 把 campaign 限制在该路径;只有全仓才省略。

注意：`build-run` 的退出码 3 只表示启动前构建失败。preflight 成功后，它沿用普通
pi 进程退出行为，不会把发现 bug / 缺失报告映射成 hook-run 的 1/2；请读报告，或通过
MCP 的 `pbt_report` 读取托管裁决。

`--scope` 与 `--func` 是写进 prompt 的范围限制(不是沙箱):

```bash
pi-pbt build-run --build-cmd "…" \
  --scope path/to/file.cpp \
  --func Foo::Bar
```

`--scope <path>` 是文件或目录(只有全仓才省略)。`--func <name>`(可选)
指该路径里的**一个符号**。必须先有 `--scope`,且命令行里 `--func` 必须写在
`--scope` **之后**;不加 `--func` 则覆盖 `--scope` 里所有值得测的函数。

`--out`、`--lang`、`--effort`(默认 `standard`)、`--provider`、`--model`、
`--mode text|json|json-full`、`--tui` 与 `hook-run` 一致。`--mode` 默认是 `text`；选择
`json` 只改变 stdout 格式，不改变 campaign 或退出行为。

#### build-run 实时 JSON 日志

**需要 pi-pbt v0.1.19 或更新版本**，先用 `pi-pbt --version` 确认。
给 `build-run` 加 **`--mode json`**，无需另加 `-p`：构建通过后，模型文本更新和
工具调用/结果以 JSONL（一行一个 JSON 事件）实时输出，不必等待最后总结，也不必扫描
session 文件。默认 `--mode text` 不是完整实时工具日志。

下面沿用本节的 OH `host_product` 构建。替换 workspace 路径、先配好模型，在 Bash
或 zsh 中执行；普通项目保留自己的 `--build-cmd`，不要照搬 OH 命令：

```bash
set -o pipefail
export PBT_OH_WORKSPACE=/path/to/openharmony
LOG_DIR=$(mktemp -d)
printf 'Logs: %s\n' "$LOG_DIR"

pi-pbt build-run \
  --workdir "$PBT_OH_WORKSPACE" \
  --repo "$PBT_OH_WORKSPACE/foundation/arkui/ace_engine" \
  --build-cmd "./build.sh --export-para PYCACHE_ENABLE:true --product-name host_product --build-target base_unittest --ccache --no-prebuilt-sdk" \
  --scope frameworks/base/geometry \
  --lang zh \
  --mode json \
  2>"$LOG_DIR/stderr.log" | tee "$LOG_DIR/events.jsonl"
```

- 终端会显示 JSON 事件，同一份事件保存在 `$LOG_DIR/events.jsonl`。
- 构建 preflight 输出和运行诊断保存在 `$LOG_DIR/stderr.log`。需要实时看构建进度，
  可在另一终端对上面打印的日志目录执行 `tail -f <日志目录>/stderr.log`。
- preflight 的完整构建日志仍在 `<out>/build.log`；这里未指定 `--out`，所以是
  `$PBT_OH_WORKSPACE/foundation/arkui/ace_engine/pbt-out/build.log`。
- 不要把 `2>&1` 加到流水线上，否则普通日志会混入 JSON。事件可能包含源码、工具参数
  和结果，公开前须检查敏感内容。`events.jsonl` **不是**最终的 `report.json`。
- `set -o pipefail` 防止 `tee` 掩盖命令失败；`--mode json` 不改变 `build-run` 的退出码
  语义。显式 `--tui` 与 JSON 互斥，继承的 `PBT_HOOK_TUI=1` 会被 JSON 模式覆盖。

#### 交互式 skill 用法

想在交互会话里**实时观测**同一条流程,直接说:

```
/skill:pbt-build-run 构建命令: <你的构建命令>
构建目录: <构建执行目录>
```

门禁与整个 campaign 就在你眼前跑,构建契约完全一致;子命令则是无头/CI 形态。

### CI / git hook 集成

针对**单个提交**测一遍,用这个子命令:

```bash
pi-pbt hook-run <sha> --repo /path/to/repo --lang zh
```

它用退出码表示结论,可以直接当 CI 的一道检查:

| | 含义 |
|---|---|
| `0` | 干净:报告满足 schema,且没有 bug。 |
| `1` | 发现 bug(`bug_reports/` 非空,或 `totals.bugs > 0`)。 |
| `2` | 没有 `REPORT.md`、没有 `report.json`,或 `report.json` 不合 schema——挂了、超时,或什么都没证明。 |
| `3` | 记录在案的构建失败:什么都没被测。 |

`--mode json` 是**精简事件日志**：agent 每做一件事输出一行 —— 它说了什么
（`message_end`，含工具调用、停止原因与 token 用量）、启动了哪个工具及其参数
（`tool_execution_start`）、返回了什么（`tool_execution_end`），以及围绕这些的重试、
压缩与 settle 转换。pi 流式输出的 token 级增量（`text_delta`、`thinking_delta`、
`toolcall_delta`）、工具的中间输出（`tool_execution_update`、`bash_execution_update`）
以及重复的全量快照（`turn_end`、`agent_end` 会把已经发过的消息再发一遍）都会被丢弃；
工具参数和结果里的长字符串会被截断，避免一次写文件淹没整个日志。思考块只保留
`thinkingChars` 长度计数。

`--mode json-full` 是 pi 未经过滤的原始会话事件流：每个增量、每份完整消息快照都在，
每个流式 token 一行。只有在需要消费完整 SDK 协议（而不是读日志）时才用它。

自动化若需要 SDK 事件流，加 `--mode json`（默认是 `--mode text`）。JSON 模式把
stdout 专用于 SDK 事件；preflight 输出、扫描诊断及其他运行信息写 stderr；
`build-run` 仍把完整 preflight 写入 `<out>/build.log`。事件流可能包含 prompt、模型
输出、工具输入/结果及敏感仓库内容，保存和发布时须按敏感数据处理。它是实时事件流，
不等于 campaign 的 `report.json`，也不等于 `~/.pi-pbt/agent/sessions/` 下的会话文件。

显式 `--tui` 与 `--mode json` 互斥；但显式 JSON 模式会覆盖继承的
`PBT_HOOK_TUI=1`。输出模式不改变上表退出码，`build-run` 也仍只在 preflight 失败时
返回 `3`。分别保存两条通道且不污染 stdout JSON：

```bash
set -o pipefail
pi-pbt hook-run <sha> --repo /path/to/repo --mode json \
  2>stderr.log | tee events.jsonl
```

不要使用 `2>&1`，否则诊断会混入 JSON 流。`build-run` 语法与完整说明见
[子命令参考](subcommands.zh.md#机器可读的-sdk-事件流)。

**sha 不会检出任何东西。** 它只用来取改动集(`git diff-tree <sha>`)和写进
prompt;真正被编译、被测的是 `--repo` 工作树里当下的代码。把树切到那个 commit
是调用方的责任。对着别的版本跑既不报错也不告警:它会测那里的代码,而拿来对照的
是另一个 commit 的 diff。

由此有两条前提:

- **从 `post-commit` 钩子调用,不需要任何额外操作**——你刚提交的那个 commit 本来
  就是工作树。**不要**在钩子里加 checkout,那会让开发者每次提交后停在 detached HEAD。
- **在 CI 里,把 commit 检出到一次性工作区,并确保它的父提交也在。** 默认的浅克隆
  (`fetch-depth: 1`)没有父提交,`git diff-tree <sha>` 会返回空改动集,`git show
  <sha>` 会把浅边界当成根提交——战役看到的要么什么都没有,要么整个仓库都是新增:

  ```yaml
  - uses: actions/checkout@v4
    with:
      fetch-depth: 2          # 这个 commit 和它的父;要全历史用 0
  ```

  ```bash
  git -C /path/to/worktree checkout --detach <sha>   # 只在一次性检出里这么做
  pi-pbt hook-run <sha> --repo /path/to/worktree
  ```

### 盯着仓库：测试新提交

两种方式,取决于你要不要**亲眼看到**它测。

**交互式 —— 在你眼前跑(推荐,只要你开着 pi-pbt)。** 在交互式 `pi-pbt`
里说:

```
/skill:pbt-watch
```

它在后台盯着当前仓库(或你指定的仓库),一旦有新提交落地,就**在你眼前**
把整套测试跑一遍(找目标、定计划、写测试、跑、复核,全程可见),跑完继续
盯下一个提交。盯着的过程不占用会话:命令立刻返回,期间你照常提别的要求;
一轮测完它自己继续,不需要你敲任何命令恢复。全程不会甩到后台看不见的地方;
不用时让它停下来即可。OpenHarmony 模块预先准备好的源码环境会自动识别并复用。
(这个会话也会同步显示在下面的 dashboard 里。)

**无人值守 —— 没人盯着的机器。** CI 镜像、没开 pi-pbt 的服务器、或装不了
git hook 的场景,改成常驻后台运行:它定时检查新提交，测试选中的批次，
结果打在日志里:

```bash
# 监控本地新 commit,每 30s 轮询(放 tmux/systemd 驻留)
pi-pbt watch --repo /path/to/repo --lang zh --provider xai-oauth --model grok-4.5

# 改为监控远端分支的新推送
pi-pbt watch --repo /path/to/repo --fetch --branch master --interval 60 --lang zh --provider xai-oauth --model grok-4.5
```

起点是启动时的最新提交，已有提交不补测。CLI watch 每轮最多取**最新 10 个**提交，
在这一批内从旧到新测试；超出的旧提交会丢弃并记日志。它是监视器，不保证每个提交都测，
也不是 CI 门禁。每个子运行的结论记录在日志里。`hook-run` 的全部参数
(`--out`/`--workdir`/`--spec`/`--lang`/`--scan-root`/`--effort`/`--provider`/`--model`/`--tui`)原样透传。`--tui`(或 `PBT_HOOK_TUI=1`)会在同一个终端里用完整的交互式界面跑(适合演示,别用在 CI:每轮结束后它会停下来等你 `/quit`)。

### 让别的 coding agent 委派测试(MCP)

如果你日常用的是 Claude Code 或 Codex,可以让它通过
[Model Context Protocol](https://modelcontextprotocol.io) 把性质测试委派给
pi-pbt。只有**不传 `build_cmd` 的 worktree 模式**才能边测边改同一个 checkout。`pi-pbt mcp` 用同一个单文件二进制以 stdio
方式提供 MCP 服务:

```bash
# Claude Code
claude mcp add --transport stdio pi-pbt -- pi-pbt mcp

# Codex
codex mcp add pi-pbt -- pi-pbt mcp
```

服务端提供六个工具:

| 工具 | 作用 |
|---|---|
| `pbt_start` | 对**一个不可变的 commit**(默认当前 `HEAD`,解析成完整 sha)发起 campaign;立即返回 `run_id`。`build_cmd` / `build_workdir` 传入已备好的构建命令,并让 campaign 就地运行(见下) |
| `pbt_status` | 查询某个 run 或 watch 的状态/阶段/排队位置/产物 URI |
| `pbt_report` | 结论(`passed` / `bugs_found` / `failed`)、报告摘要、来自 `report.json` 的结构化结果、bug 报告清单、是否有 patch |
| `pbt_cancel` | 取消排队或进行中的 campaign(幂等) |
| `pbt_watch_start` | 监听分支,每来一个新 commit 发起一次 campaign |
| `pbt_watch_stop` | 停止监听(默认连带取消进行中的 run) |

不传 `build_cmd` 时，运行在 detached `git worktree` 里测试不可变提交，可继续编辑
原 checkout。传入 `build_cmd` 则**就地测试**，运行结束前不要编辑这个 checkout；
构建命令模式见下文。

代码还没 commit?给 `pbt_start` 传 `include_uncommitted: true`,它会先把当前
未提交状态(已跟踪文件的修改 + 未跟踪且未被 ignore 的新文件)冻结成一个不可变
的快照 commit,再像普通 revision 一样去测。快照用一次性的 git index 生成 ——
你的 index、`HEAD`、refs、stash 和文件全都不动,期间可以继续写代码。工作树
干净时就直接测 `HEAD`。该参数与 `revision` 互斥;run 会返回 `snapshot: true`
和快照所基于的 `base_revision`。

**一个刻意的例外:`build_cmd`。** 大型组件(OpenHarmony、AOSP、monorepo 子树)
无法在独立 worktree 里构建 —— 它们的构建系统按真实 checkout 解析路径、生成头文件
和 ccache 状态。因此给 `pbt_start` 传 `build_cmd` 时,campaign **就地在仓库本身
运行,不创建 worktree**。结束前不要编辑这个 checkout。该命令先跑,非零退出会在任何 agent 工作之前停掉 campaign,
于是构建坏了只花几秒,而不是一轮探索式编译。构建需要在被测模块之上的目录执行时
(例如 scope 是单个组件、构建要在 OpenHarmony 源码根跑)用 `build_workdir` 指定。
你知道自己的构建命令就用它;不传则保持 worktree 隔离。

run 结束后 worktree 被清掉;留下的东西在 `~/.pi-pbt/runs/<run-id>/`
(用 `PI_PBT_RUNS_DIR` 改位置):`run.json`、`events.jsonl`、两份日志、campaign
产物,以及一份 `changes.patch`(campaign 在自己 worktree 里写下的全部内容)。
这些也都能通过 MCP resources 按 `pbt://runs/<run-id>/...` 读取(`REPORT.md`、`report.json`、
`PROPERTIES.md`、`bug_reports/<slug>.md`、`changes.patch` 等)。

重量级 campaign 串行执行:默认同时只跑一个子进程(`PBT_MCP_MAX_CONCURRENT`
可调高),多余的 `pbt_start` 排队。watch 默认 `supersede: true` —— 新 commit
会取消一个还没进 Review 的进行中 run;已经在 Review 的 run 先跑完,中间积压的
commit 合并成最新那个,所以 watch 永远收敛到最新代码、不会堆积 campaign。

进度有两条推送路径。标准 MCP logging 通知在任何客户端都能用,轮询
`pbt_status` 永远可靠。Claude Code 会话还能收到实时 channel 事件(阶段变化和
最终结论,只携带 run ID 和状态 —— 绝不携带仓库内容)。channel 是 Claude Code
的 research preview 功能,这部分需要启动时显式开启:

```bash
claude --dangerously-load-development-channels server:pi-pbt
```

不加这个 flag,除了推送以外的一切照常工作。

### 真机上的 HarmonyOS 应用（`pi-pbt kea`，实验性）

`pi-pbt kea` 通过 GUI 对已经装在 USB 手机上的 HarmonyOS 应用做黑盒测试。
该入口仍属实验性；前置条件、配置、命令、产物和 Kea 专属环境变量见
[双语 Kea 指南](kea.zh.md)。

### 环境变量

| 变量 | 作用 |
|---|---|
| `PBT_LANG=zh` / `en` | 全程用中文或英文思考并写产物;子命令也可用 `--lang`。`en` 在战役提示本身是中文时仍会注入英文指令 |
| `PBT_SCAN_ROOT=/path` | 要扫描的仓库路径。用于工作目录是一份干净副本的场景(git hook / CI) |
| `PBT_OH_WORKSPACE=/path` | 预先准备好的完整 OpenHarmony 源码环境(源码 + 编译工具链 + 已编译好的依赖),直接复用而不是从头推导怎么单独构建。仓库位于这样的环境内部时(某个上级目录同时有 `.repo/` 和 `out/`)会**自动识别**,只有要覆盖时才需要显式设置;启动日志会打印实际用的是哪个 |
| `PBT_HOOK_TUI=1` | 等同 `hook-run --tui` / `watch --tui`:用交互式界面跑(需要终端;结束后等你 `/quit`) |
| `PBT_EFFORT=quick\|standard\|thorough` | 一次 campaign 挖多深(见[effort 档位](#挖多深effort-档位));`hook-run` / `watch` 上的 `--effort` 优先级更高。默认:`hook-run`/`watch` 为 `quick`,其他入口为 `standard`。实验性的 Kea 模式在未单独配置深度时也使用该档位，见 [Kea 指南](kea.zh.md)。 |
| `PBT_TEST_JOBS=8` | 测试和构建跑多宽(见[并行度](#跑多宽并行度));填数字、`max` 或 `auto`(默认:机器空闲核心数)。`PBT_TEST_JOBS=1` 强制串行 —— 把失败判定为 bug 之前的串行复核必须用它 |
| `PBT_CODE_COVERAGE=0` | 关掉原生代码覆盖率的插桩与报表(见[代码覆盖率报表](#代码覆盖率报表));默认开启,工具链缺失时静默降级 |
| `PBT_BARE=1` | 以纯 pi 运行:不加载编排扩展、内置技能与 campaign 守卫。用于 A/B 度量 harness 自身贡献(benchmark 场景),非日常使用 |
| `PI_PBT_RUNS_DIR=/path` | [`pi-pbt mcp`](#让别的-coding-agent-委派测试mcp) 保存 run 状态与产物的目录(默认 `~/.pi-pbt/runs`;必须在被测仓库之外) |
| `PBT_OUT_DIR=/path` | campaign 产物(`PLAN.md`、`PROPERTIES.md`、`COVERAGE.md`、`REPORT.md`、`report.json`、`bug_reports/`)所在目录。`build-run` 与 `hook-run` 会用各自的 `--out` **自动设置**,正常情况下你不需要自己设;它的存在是为了让产物驱动的检查读到与 campaign 写入相同的目录。普通 `pi-pbt -p` 不设它,用 `<cwd>/pbt-out`。**不要写进 shell profile**:残留的值会跟着之后每一次 campaign,而继承来的、指向别处的值正是它要防的那种「两份账本」 |
| `PBT_MCP_MAX_CONCURRENT=2` | MCP 委派的 campaign 同时最多跑几个(默认 1,多余的排队) |
| `PBT_PHASE_MODELS='{"plan":"anthropic/claude-opus-5"}'` | 按 campaign 相位(`scan`、`plan`、`test`、`review`)路由不同模型,写作 `<provider>/<modelId>`。不设则全程一个模型,这是默认。相位从 `pbt-out/` 下的产物读出,不由 agent 自报;模型不存在或 provider 未配置时静默保持当前模型。刻意不内置路由表:`scan` 与 `review` 是决定契约是什么、以及失败是否成立的地方,给它们降级是拿假绿换 token |

## 5. 网页面板:实时看它在干什么

pi-pbt 可以起一个网页面板,实时展示这台机器上正在跑的每一次运行(你手动开的、
git hook 触发的、盯仓库触发的都算)—— 对话过程、token 用量、产物,以及历史
记录。不需要额外装 npm 包,pi-pbt 自己负责安装和运行:

```bash
pi-pbt dashboard install       # 一次性:把 dashboard 包锁版安装到
                               # ~/.pi-pbt/dashboard,并把 bridge 扩展接入
                               # ~/.pi-pbt/agent/extensions/
pi-pbt dashboard --port 8000   # 驻留运行(放 tmux/systemd)
```

例如:

```bash
tmux new-session -d -s dash 'pi-pbt dashboard --port 8000 >> ~/pbt-dash.log 2>&1'
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8000/   # 期望 200(约 20s 启动)
```

然后浏览器打开 **http://localhost:8000**。远程主机先做端口转发:
`ssh -N -f -L 8000:localhost:8000 <host>`。

工作原理:install 那步做了一次性接线,之后**这台机器上每一次 pi-pbt 运行都会
自动出现在面板里** —— git hook、CI、盯仓库拉起的运行一开始就能看到(名字形如
`pbt_hook_<sha>`),不用逐次配置。历史记录读自 `~/.pi-pbt/agent/sessions/`
(不是 `~/.pi/agent/sessions/`)—— pi-pbt 的配置和记录与普通 `pi` 安装互不干扰。

注意事项:
- 侧边栏按**收藏的目录**分组 —— 把关心的仓库目录收藏上(界面里的 folder
  菜单),否则它的记录只能从完整列表里翻。
- 用 `curl localhost:8000` 验证是否活着(有些机器上 `ss`/`netstat` 看不到端口)。
- 更新 pi-pbt 后,再次执行 `pi-pbt dashboard install`,以刷新这个独立安装且精确
  锁版的 dashboard 包。
- 每个用户只能运行一个 dashboard:它会占用网页端口以及 bridge gateway 的
  **9999** 端口。`--port` 只改变网页端口。重启前先停止旧实例;`pi-pbt dashboard`
  会在调用上游服务之前报告任一被占用的端口。

## 6. 常见问题

- **它不按内置流程走,自由发挥** —— 内置流程文件没能就绪(启动日志里
  `skills=0`)。删掉缓存目录 `~/.pi-pbt/cache/` 再跑一次让它重建;若仍为 0,
  多半是家目录不可写或磁盘满。
- **中途报 `command not found: cmake`** —— 按 §1 装好目标语言的工具链后重跑。
  它不会绕过编译假装测过。
- **在 CI 里卡住不动** —— 不该发生:它不会等输入,跑完会强制退出。若确实卡住,
  请带日志提 issue。
- **macOS 拦住不让运行** —— 执行 §2 里的 `xattr -d com.apple.quarantine`。

- **agent 说要调用工具，随后却停止了** —— 如果明确承诺立即调用某个可用工具，却没有
  真实 tool call，每轮战役最多触发**两次纠正重试**，要求提交实际参数和调用，不能再
  只说一遍。给用户的建议、引用示例、已有真实调用、取消或 provider 错误不触发此规则。
  若返回的是空响应，则由另一条恢复机制尝试**一次**。先检查 `REPORT.md` 和 `report.json` 是否完整；“准备继续”的口头承诺
  不算运行完成。查看所选产物目录（默认 `pbt-out/`）里的 `recovery.json`，以及
  `~/.pi-pbt/agent/sessions/` 下的会话日志。并非每种停止都会触发恢复，取消或 provider
  错误也不由此机制重试。若仍停止，请检查模型配置的上下文上限及 provider 的错误/额度，
  再缩小测试范围或换用合适的模型重新运行。反馈问题时附上日志，不要只反复发送“继续”。
