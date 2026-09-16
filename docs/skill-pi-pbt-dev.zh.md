# pi-pbt-dev skill 用户教程

> 让 code agent 对代码做 property-based testing（PBT）：全量测整个项目，或增量测你刚改的代码。
> 本教程带你从零装到第一次跑出 bug 报告。英文版见 [skill-pi-pbt-dev.md](skill-pi-pbt-dev.md)。

## 1. 这个 skill 是什么

`pi-pbt-dev` 是把 pi-pbt 接入你日常 code agent 的入口。装好后，你可以对 agent 说
"测一下刚才的修改"，agent 就会自动对改动范围跑一轮 PBT 战役，并把发现的问题呈现给你。

两种模式，由 agent 根据你的意图选择：

| 模式 | 你说什么 | 测什么 |
|---|---|---|
| **全量** | "给这个项目跑一轮 PBT" | 整个项目/指定模块 |
| **增量** | "测一下刚才改的代码" | 只测最近修改的文件 |

架构是**薄壳**：skill 只含协议文本（`SKILL.md`）和三个辅助脚本，真正的 PBT 引擎
是 `pi-pbt` 二进制（独立安装）。升级 skill 只更新文本，升级引擎只更新二进制。

### skill 内进程与 MCP 委派的区别

本教程讲的是**安装 skill** 这条路径。宿主 agent 启动
`scripts/run-pbt.mjs`，脚本再启动一个 `pi-pbt -p` 子进程，并等待它退出后读取结果。
进度行会把等待过程显示出来，但契约仍是同步的：这不是后台委派，同一个 agent turn
不会在 campaign 运行时继续编辑代码。

MCP 是另一套无需安装 skill 的委派接口。`pbt_start` 会立即返回 `run_id`，独立的
pi-pbt run manager 接管 campaign 进程，宿主 agent 可继续开发，之后再调用
`pbt_status` / `pbt_report`。普通 MCP run 在 detached worktree 中测试不可变 commit；
`include_uncommitted: true` 会先把当前改动冻结成快照；传入 `build_cmd` 则是有意的
就地运行例外。这里先介绍 MCP，是为了让后文出现的 `pbt_report` 和 `pbt://...`
结果资源都有上下文。配置方式和隔离规则见
[让别的 coding agent 委派测试](installation.zh.md#让别的-coding-agent-委派测试mcp)。
想按下文同步执行就装 skill；明确要异步委派时再配 MCP。两者可以共存。

## 2. 前置条件

| 项目 | 要求 | 说明 |
|---|---|---|
| Node.js | ≥ 18 | skill 的辅助脚本是 Node 写的（`pi-pbt` 二进制本身不需要 Node） |
| `pi-pbt` 二进制 | 已安装 | 见[第一步](#3-第一步安装-pi-pbt-引擎) |
| git | 推荐 | git 项目增量测试有"客观范围层"（自动列出改动文件）；非 git 也能用，范围由 agent 从对话记忆给出并请你确认 |
| 被测语言的工具链 | 按需 | Python 要 `python3`+`pip`，C/C++ 要 `cmake`+编译器，Rust 要 `cargo`…（pi-pbt 会真实编译运行它写的测试） |
| LLM API | 已配置 | 战役由大模型驱动，见[第一步](#3-第一步安装-pi-pbt-引擎) |

## 3. 第一步：安装 pi-pbt 引擎

官方发行平台为 **Linux x64 / Linux arm64 / macOS arm64**，从你的分发渠道下载
对应 zip（详情见[安装指南](installation.zh.md#2-下载与安装)）。每个压缩包都会解出一个
同名目录；运行随包安装脚本，让主程序和内置 `fd`/`rg` 一起正确安装：

```bash
PLATFORM=linux-x64   # 或 linux-arm64 / macos-arm64
sha256sum -c "pi-pbt-${PLATFORM}.zip.sha256"   # 可选的完整性校验
unzip "pi-pbt-${PLATFORM}.zip"
cd "pi-pbt-${PLATFORM}"
./install.sh
```

macOS 没有 `sha256sum` 时可用 `shasum -a 256 -c`，并按安装指南清除 Gatekeeper
隔离标记。

**Windows**：官方暂未发布 Windows 二进制。替代路径（源码构建 + 本地打包安装）：

```bash
git clone <pi-pbt 仓库> && cd <pi-pbt 仓库>
npm ci --ignore-scripts
npm run build
npm pack --ignore-scripts
npm i -g ./pi-pbt-<版本>.tgz        # 安装后获得 pi-pbt 命令
```

> **注意**：`npm i -g pi-pbt` 目前**不可用**——包尚未发布到 npm registry，请使用上面的官方路径。

### 配置模型

pi-pbt 的配置目录是 `~/.pi-pbt/agent/`（首次运行自动创建）。最快的两种方式：

```bash
# 方式 1：API key 环境变量，立即可用
export ANTHROPIC_API_KEY=sk-ant-...   # 或 OPENAI_API_KEY / GEMINI_API_KEY / ...
pi-pbt --list-models                   # 能看到模型列表即配置成功
```

```text
# 方式 2：交互式登录
pi-pbt            # 启动后 /login 登录，/model（Ctrl+L）选模型
```

自定义模型/代理使用 `~/.pi-pbt/agent/models.json`。CLIProxyAPI 上的 GLM/DeepSeek
见[模型配置示例](models-glm-deepseek.zh.md)，按实际渠道核对 ID 和限制，不要照抄通用
1M 上下文配置。

验证：

```bash
pi-pbt --version
pi-pbt --list-models
```

> **建议用你最强的模型**。PBT 最关键的一步是从代码推导出"什么必须恒成立"——这是纯推理，
> 弱模型只能写出"调用不崩溃"这类空性质，测试全绿但什么都发现不了。

### skill 如何找到 pi-pbt（默认无需配置）

skill 的脚本按以下顺序解析引擎位置：

1. 环境变量 `PI_PBT_BIN`（若设置且文件存在）
2. `PATH` 中的 `pi-pbt`（npm 全局安装或 release 二进制放入 PATH）

前两种都不满足时才报错并引导安装。所以默认情况**不需要任何配置**；仅当二进制放在
PATH 之外（例如 zip 解压到自定义目录）时，显式指定：

```bash
export PI_PBT_BIN=/path/to/pi-pbt      # Windows: set PI_PBT_BIN=C:\path\to\pi-pbt.exe
```

## 4. 第二步：安装 skill

skill 随 **pi-pbt 发行包自带**（内嵌在二进制里，或 npm 包 `skills/` 目录下），
所以安装只需一条命令——把文件从 pi-pbt 拷到你的 agent 的 skill 目录：

```bash
pi-pbt skill-install                # 自动检测已安装的 agent
pi-pbt skill-install --host claude  # 或 opencode / codex / codeagent / chrys
pi-pbt skill-install --host project # 项目级（共享给协作者）
pi-pbt skill-install --dir /自定义/路径  # 任意 skill 目录
pi-pbt skill-uninstall              # 卸载（参数同上，幂等）
```

支持的宿主位置：

| 宿主 | 位置 |
|---|---|
| Claude Code | `~/.claude/skills/pi-pbt-dev/` |
| opencode | `~/.config/opencode/skills/pi-pbt-dev/` |
| Codex CLI | `~/.agents/skills/pi-pbt-dev/`（用户级）或 `<repo>/.agents/skills/pi-pbt-dev/`（项目级） |
| codeagent | `~/.cac/skills/pi-pbt-dev/`（用户级）或 `<repo>/.cac/skills/pi-pbt-dev/`（项目级） |
| chrys | `~/.chrys/skills/pi-pbt-dev/`（用户级）或 `<repo>/.chrys/skills/pi-pbt-dev/`（项目级） |
| 项目级（`--host project`，共享给协作者） | `<repo>/.claude/skills/`、`<repo>/.agents/skills/`、`<repo>/.cac/skills/`、`<repo>/.chrys/skills/`（四个全装） |

更新 pi-pbt 后重新执行并加 `--force` 可刷新旧安装。`skill-uninstall` 与安装使用
相同的目标选择：传同一个 `--host` 或 `--dir` 即可移除对应副本；不带参数时，它会从
当前能检测到的用户级宿主位置移除，不会记忆历史上曾安装到哪些目录。

**Codex 触发方式**：显式输入 `$pi-pbt-dev` 提及，或 `/skills` 浏览选择；隐式时模型按
description 自动选用（与 Claude Code/opencode 一致）。Codex 已废弃 custom slash commands，
skills 是官方唯一推荐的扩展路径，本 skill 结构（frontmatter + scripts/）原生兼容。
Codex 无原生工具注册 API，但本 skill 本就通过 Bash 调用 node 脚本，不受影响。

冒烟验证（在任意 git 仓库内）：

```bash
node <skill>/scripts/diff-scope.mjs   # 应输出改动文件清单 JSON
```

## 5. 第三步（可选）：项目配置

**自动触发**：在项目根 `AGENTS.md`（或 `CLAUDE.md`）追加，让 agent 在每轮代码修改收尾时自动跑增量 PBT：

```markdown
## PBT 自动触发（pi-pbt-dev）

当一轮代码修改完成（生产代码的 write/edit 收尾、连续编辑静默）时，调用 `pi-pbt-dev` skill
对本次改动做增量 PBT：

- 改动仅文档/注释/纯测试/配置文件 → 跳过
- 改动含生产代码 → 按 skill 模式判定执行（范围须经确认，见 skill 契约）
- 战役发现的 bug 摘要必须呈现在对话中，并询问是否修复
```

**产物目录**：`pbt-out/` 建议加入 `.gitignore`：

```bash
echo "pbt-out/" >> .gitignore
```

## 6. 使用教程：三个场景

### 场景 1 — 全量测试

```text
你: 给这个项目跑一轮 PBT，看有没有隐藏的 bug
agent: （加载 pi-pbt-dev skill，判定为全量模式）
       1. 告知你战役已启动与预计时长（默认 standard 档：约 30 分钟；说"快速验证"
          走 quick / "深入测"走 thorough）
       2. 先选一个新的轮次目录，如 pbt-out/2026-09-16T10-30-00，再运行：
          node <skill>/scripts/run-pbt.mjs --prompt "对 <项目路径> 做性质测试(PBT)战役，目标是找出 bug。产物写到 pbt-out/。" --out-dir pbt-out/2026-09-16T10-30-00 --cwd <项目路径> --lang zh --effort standard --timeout-sec 1800
       3. 等待这个子进程退出；期间按阶段进度行向你简报（启动 → 规划中 → 测试进行中（N 个性质）→ 完成）
       4. node <skill>/scripts/summarize.mjs pbt-out/2026-09-16T10-30-00   → 读取摘要
       5. 向你呈现：发现 N 个 bug / 全部通过
```

> 战役等待期间 agent 会报告阶段进展（`[pbt-progress]` 行），不会长时间无反馈。

### 场景 2 — git 项目增量测试

```text
你: 测一下刚才对 src/validators.py 的修改
agent: （判定为增量模式）
       1. node <skill>/scripts/diff-scope.mjs   → 客观改动清单 + 被改动的函数（git 工作区未提交改动）
       2. 用会话记忆交叉校验：清单里是否有本会话没碰过的文件？有则询问你是否纳入
       3. 确认范围后，diff 能提取出函数名时优先函数级 prompt（上下文更窄、深度不变）：
          node <skill>/scripts/run-pbt.mjs --prompt "对下列最近修改的函数做 PBT 战役，目标是找出 bug：for src/validators.py, Function validate_… 产物写到 pbt-out/。" --out-dir pbt-out/2026-09-16T10-30-00 --cwd <项目路径> --lang zh --effort standard --timeout-sec 1800
          （提取不到函数名时回退文件级 prompt）
       4. 呈现结果
```

### 场景 3 — 非 git 项目增量测试

```text
你: 测一下刚才改的代码
agent: （判定为增量模式；diff-scope 报非 git 仓库 → 走会话记忆路径）
       1. 从本会话记忆列出修改的文件（如 src/validators.py、src/utils.py），
          向你确认："将对以下文件做 PBT：… 对吗？"
       2. 你确认（或调整清单）后：
          node <skill>/scripts/run-pbt.mjs --prompt "对下列文件做 PBT 战役：src/validators.py src/utils.py … 产物写到 pbt-out/。" --out-dir pbt-out/2026-09-16T10-30-00 --cwd <项目路径> --lang zh --timeout-sec 1800
       3. 呈现结果
```

> **范围确认门禁**：增量范围（尤其非 git 的记忆清单）必须经你确认后才会执行战役。
> 这是防漏测/误测的设计，不是多余的步骤。

## 7. 读懂结果

skill 会为每轮战役选择一个新目录（如 `pbt-out/2026-09-16T10-30-00/`），并把它
作为 `--out-dir` 传入。campaign 运行时先写仓库顶层 `pbt-out/`；结束后
`run-pbt.mjs` 再把规范产物移入该轮次目录，随后 `summarize.mjs` 从那里读取。

| 文件 | 内容 |
|---|---|
| `REPORT.md` | 给人阅读的总结：测了哪些模块、发现哪些 bug、如何复现 |
| `PROPERTIES.md` | 性质台账：每个被测性质的形式化陈述与状态（passing / failing / retired） |
| `bug_reports/bug_001_*.md` | 每个 bug 一份：Law、最小输入、期望/实际、根因、回归测试位置 |

源码 campaign 契约还会生成 `report.json`，它是同一组事实的机读视图：schema 版本、
被测 revision 与档位、构建证据、每条性质和 bug、每个 bug 的精确复现命令，以及重新
核算的 totals。脚本、CI、dashboard 或 MCP client 需要稳定字段而不是解析 Markdown
时，应读取它。

`run-pbt.mjs` 会把本轮 `report.json` 与 `REPORT.md` 一起归档到 `--out-dir`；请成对
读取该轮次目录，不要取仓库根部可能变化的文件。启动前，根目录已有的 `REPORT.md`、
`report.json`、`bug_reports/` 会保留在 `pbt-out/.previous-result-*/`，不算作新一轮产物。
已有内容的 `--out-dir` 会被拒绝，必须选新目录；失败运行没写 JSON，就不会拿上一轮
JSON 顶替。`summarize.mjs` 仍读 Markdown，不负责 JSON schema 校验。

受管的 `hook-run` 与 MCP 会共同校验 `REPORT.md` / `report.json`，JSON 缺失或无效
意味着未完成。MCP 全文资源为 `pbt://runs/<id>/report.json`，`pbt_report` 返回摘要；
字段及校验规则见[报告契约](report-schema.md)。


**版本注意：**2026-09-16 替换发布的 v0.1.18 包已含上述归档修复，此前下载的
同版本旧包不含。请重新下载包和校验文件，替换二进制后执行
`pi-pbt skill-install --force`；只看版本字符串无法区分这两次构建。

agent 呈现给你的摘要形如：

```text
**状态:** 发现 1 个 bug | **性质:** 5 passing / 1 failing | **测试:** 6 total

### Bug #1: validate_port(0) returns None instead of raising ValueError（Medium）
- 函数: validate_port in example_lib/validators.py
- 最小反例: validate_port(0)
- 期望: ValueError raised / 实际: Returns None
- 根因: 下界判断只排除了负数，0 漏过
```

发现 bug 后你可以让 agent 直接修复，修完再针对修复文件跑一轮增量验证。

## 8. 故障排查

| 现象 | 处理 |
|---|---|
| `pi-pbt: command not found` | 按[第一步](#3-第一步安装-pi-pbt-引擎)重装引擎，确认在 PATH |
| `pi-pbt --list-models` 为空 / 报 401 | 按[第一步](#3-第一步安装-pi-pbt-引擎)检查 API key 与 models.json |
| 战役超时（`run-pbt` 退出码 3） | 按档位使用对应上限：quick 900 秒、standard 1800 秒、thorough 3600 秒；改测更小范围，或有意调大 `--timeout-sec` |
| 摘要显示"未完成"（无 REPORT.md） | 战役 agent 未产出人读报告；看 `<out-dir>.log` 定位原因后重试。对 `hook-run`/MCP 还要检查 `report.json` 是否存在且符合 schema |
| agent 不自动触发 | 确认 AGENTS.md 片段已加；显式说"测一下刚才的修改"一定可用 |

## 9. 成本与建议

- 每轮战役是一次完整的 LLM 会话（分钟级、有 token 成本）。改动很小（一两次编辑）或仅文档时，
  可以要求跳过——agent 的模式判定会尊重你的选择。
- 增量测试聚焦改动面，比全量便宜；建议日常用增量，定期（如发版前）跑一次全量。
- 每轮都要选新的 `--out-dir`（如 `pbt-out/2026-09-16T10-30-00/`）。辅助脚本不会
  自动生成这个名字；复用非空目录会被拒绝且旧内容保留。请选择新目录，避免新旧结果混在一起。

## 10. 升级

替换 pi-pbt 二进制**不会**刷新复制到宿主 agent 的 skill。安装新包后执行
`pi-pbt skill-install --force`；项目/自定义安装应复用原来的 `--host project` 或
`--dir`，并加 `--force`。先备份手工改过的 skill，再刷新并重载宿主 agent。
最后用 `pi-pbt --version`、`pi-pbt --list-models` 检查。
完整步骤见[升级已有安装](installation.zh.md#升级已有安装)。
