# PBT 覆盖登记与代码覆盖率

pi-pbt 有两类不同的“覆盖率”，不能混为一谈：

1. **PBT 覆盖登记**：`FUNCTION_INDEX.md` 列出启发式扫描到的函数，`COVERAGE.md`
   记录 campaign 声称已为哪些函数写过并运行过性质测试；`pi-pbt coverage` 根据这两份
   Markdown 台账做汇总。这是测试进度登记，不是编译器或运行器测出的真实执行覆盖率。
2. **代码覆盖率（execution evidence，执行证据）**：可用工具链会对 campaign 自己的
   构建和测试做插桩，产出 `pbt-out/code-coverage/` 下的原生 HTML 报表，并把
   `COVERAGE.md` 的声明与实际执行过的符号交叉核对。安装、工具链和开关见
   [安装指南的代码覆盖率章节](installation.zh.md#代码覆盖率报表)。

本页主要解释第一类台账及其命令。命令总览见
[`scan` / `test-all` / `coverage`](subcommands.zh.md#scan--test-all--coverage)。

---

## 子命令

### `pi-pbt scan`

纯本地操作（不调用 LLM）。实际语法是 `--dir` 选项，而不是位置参数：

```bash
pi-pbt scan --dir /path/to/project --out /path/to/project/pbt-out
```

它用 `rg`（ripgrep，文本搜索工具）的语言正则启发式匹配函数定义，并写
`<out>/FUNCTION_INDEX.md`。支持 C/C++、Python、Rust、TypeScript/JavaScript、Go；
一次运行只选择一种探测到的语言。它不是语法树解析器，因此不能保证找全所有函数，也
可能误认宏、声明或复杂签名。所谓 `PBT Candidate` 也只是按函数名排除少量已知
internal/trivial（内部或简单包装）模式后的启发式分类，不表示该函数已经可构建或可测试。

自动语言探测只用 `find -maxdepth 2` 看 `--dir` 下两层；源码更深时应把 `--dir`
直接指向源码子树，或用 `--lang c/c++|python|rust|typescript|go`。这与
[子命令文档的说明](subcommands.zh.md#scan--test-all--coverage)一致。

`test-all` 在索引缺失时会自动扫描。`hook-run` 也会尝试预扫描，但当前实现把索引写到
`<out>/../pbt-out/FUNCTION_INDEX.md`，不一定是 campaign 最终写产物的根目录；扫描失败时
只打印 `auto-scan skipped` 并继续。显式 `scan --out <dir>` 适合独立检查索引，但要让
`test-all` 复用它，`--out` 必须是同一个目录；`hook-run` 会清空自己的 `--out`，不能
把事先生成的同目录索引当作持久输入。

### `pi-pbt test-all`

读取 `<out>/FUNCTION_INDEX.md`，把其中标为 PBT candidate 的函数清单放进一次
campaign 的 prompt，要求 agent 逐个处理。它会真实调用模型；“清单进入 prompt”不等于
工具本身已经证明每个函数都成功构建、执行或覆盖，最终应检查 `report.json`、
`REPORT.md`、测试日志和代码覆盖执行证据。

```bash
pi-pbt test-all --dir . --out ./pbt-out --provider <provider> --model <model>
```

若索引不存在，`test-all` 会先用与 `scan` 相同的启发式逻辑生成它；无法生成时退出。

### `pi-pbt hook-run <sha>`

面向 CI / git hook 的单提交检查：

```bash
pi-pbt hook-run <sha> --repo . --out ./pbt-out
```

`<sha>` 用于读取 diff 并写入 prompt，不会替你 checkout；被编译和测试的是 `--repo`
当前工作树。完整前提和退出码见[安装指南的 CI / git hook 集成](installation.zh.md#ci--git-hook-集成)。

命令启动前会清空选定的 `--out` 和工作目录，再尽力预扫描；因此不要把“增量模式”理解为
该目录里的台账一定跨 `hook-run` 自动保留。需要跨 campaign 保存登记表时，应另外归档
这些产物或使用 managed run 的产物目录。

### `pi-pbt coverage [--diff <sha>]`

`coverage` 不调用 LLM，从**当前工作目录的** `pbt-out/` 读取 `FUNCTION_INDEX.md`、
`COVERAGE.md`（没有时回退到 `COVERAGE_STATUS.md`）以及根目录或一级子目录中的
`SCAN.md`，然后把台账汇总打印到 stdout。它没有 `--out` 选项；若产物在别处，先
`cd` 到含有该 `pbt-out/` 的目录。

```bash
cd /path/to/project
pi-pbt coverage --list
pi-pbt coverage --json
pi-pbt coverage --diff <sha> --project-dir .
```

可用选项还包括 `--module <name>`（按 registry 的 module 字段筛选输出）；`--list` 与
`--json` 互斥。

`--diff` 先解析 `<sha>^..<sha>` patch 中的新增行，尝试识别 C/C++、Python、Rust、Go
函数定义。识别到函数时按函数名筛选索引；识别不到时退化为“改动文件中的所有索引
函数”。当前文件匹配只保留 basename（文件名最后一段）并做后缀匹配，同名文件可能混淆。
所以 diff 段是启发式的登记表视图，不能笼统称为“实际被修改函数”的精确集合，也不是
行覆盖率或分支覆盖率。

---

## campaign 结束时会自动追加什么

`hook-run`、`build-run`、`test-all` 以及普通 print mode（命令行打印后退出）campaign 结束时，入口会在
找到台账数据后：

1. 把 PBT 覆盖登记摘要打印到 stderr；
2. 若 `REPORT.md` 存在且还没有 `## Coverage Report`，追加一节由台账计算出的报告；
3. 对 `hook-run`、`build-run` 和普通 print mode，若原生插桩确实产出了可收集的数据，再追加单独的
   `## Code coverage (execution evidence)` 执行证据节，并生成
   `code-coverage/index.html`。

`test-all` 的专用收尾路径当前只执行前两项，不会自动生成原生 HTML 或追加执行证据节；
不能把它的登记表摘要当作完整原生覆盖率报告。

对 `hook-run`，stderr 可能额外打印按提交计算的 diff 摘要；当前自动追加进
`REPORT.md` 的 `Coverage Report` 本身是全量台账报告，并不会自动带上
`pi-pbt coverage --diff` 命令所输出的完整 Diff Coverage 段。需要该段或 JSON 时仍要
手动运行 `pi-pbt coverage --diff ...`。

若没有索引/台账、没有 `REPORT.md`，或没有插桩数据，相应步骤会跳过；不能据此声称所有
项目和框架都自动得到完整覆盖率报告。

---

## 输出产物

### `pbt-out/FUNCTION_INDEX.md`

函数索引台账。可由 `pi-pbt scan` 的正则启发式生成，也可由 agent 在 campaign 的
Scan 阶段生成或补充。它只描述已登记的扫描结果；即使表头写着 `Total functions`，也不
证明编译器意义上的所有函数都已被完整枚举。

**格式：**
```markdown
> Total files: <N> | Total functions: <M> | PBT candidates: <K> | Excluded: <J>

| Function | Source File | Line | Kind | PBT Candidate | Reason |
|----------|-------------|------|------|---------------|--------|
| compress2 | compress.c | 67 | function | yes | round-trip |
| compress | compress.c | 82 | function | no | wrapper |
```

解析器会去掉常见 Markdown 强调和反引号，但函数名、路径等字段最好保持纯文本，避免
竖线等表格分隔符破坏列解析。`Source File` 是相对扫描根的路径。

### `pbt-out/COVERAGE.md`

测试结果登记表，由 agent 在 Review 阶段写入。每一行会把匹配到的 registry entry
标为 `covered`，表示 campaign 台账声称该函数已有性质测试；这本身不证明函数体真的
执行过。真实执行情况看 `code-coverage/` 及 `REPORT.md` 的 execution-evidence 节。

**格式：**
```markdown
# PBT Coverage Status

| Function | Source file | Test file | Test target | Notes |
|----------|-------------|-----------|-------------|-------|
| compress2 | compress.c | t.cpp | t | pass (4/4 properties) |
| uncompress_auto | uncompr.c | t2.cpp | t2 | fail (3/6 properties failing) — see bug report |
```

- `Source file` 应与 `FUNCTION_INDEX.md` 的路径一致；解析器支持完整相对路径、文件名
  后缀匹配，也会兼容性地剥掉末尾 `:行号`。仍建议不要写行号，以免同名文件产生歧义。

### `pbt-out/COVERAGE_STATUS.md`

全量 PBT 覆盖登记的 Markdown 快照。编排扩展会在有 registry 数据时计算并写入；
agent 也可能写它。`pi-pbt coverage` 在找不到 `COVERAGE.md` 时会把它当作输入回退，
但它不是原生代码覆盖率文件。

### `pi-pbt coverage --list` 完整登记报告

从可找到的 `FUNCTION_INDEX.md`、`COVERAGE.md` / `COVERAGE_STATUS.md` 和
`SCAN.md` 综合生成。下面的百分比都表示“登记为 tested/covered 的函数条目占台账分母
多少”，不是语句、行或分支执行覆盖率。只有当索引和登记表本身完整、路径匹配正确时，
这些数字才适合作为 campaign 进度指标。

**示例输出：**
```
# PBT Coverage Status

> Last updated: 2026-07-23 16:52 | Files: 4/5 scanned (80%)
> Functions: 10/17 total | PBT candidates: 10 | Tested: 8 (80%) | 7 pass, 1 fail

## Summary

| Metric                                     | Value              |
|--------------------------------------------|--------------------|
| Total source files                         | 5                  |
| Files scanned                              | 4 / 5 (80%)       |
| Total functions (all files)                | 17                 |
| PBT candidates (from FUNCTION_INDEX)       | 10                 |
| **Tested (of PBT candidates)**            | **8 / 10 (80%)**  |
|   ↳ Pass / Fail / Other                   | 7 / 1 / 0          |
| **Overall (tested / all functions)**      | **8 / 17 (47%)**  |
| Untested                                   | 2                  |
| Skipped                                    | 0                  |

## Module Breakdown

| Module      | Scanned | Tested | Coverage |
|-------------|---------|--------|----------|
| campaign-1  | 3       | 3      | 100%     |
| campaign-2  | 5       | 3      | 60%      |

## Oracle Type Distribution

| Oracle Type   | Total | Covered | Coverage |
|---------------|-------|---------|----------|
| round-trip    | 4     | 3       | 75%      |
| reference     | 4     | 4       | 100%     |
| state-machine | 2     | 0       | 0%       |

## File Coverage

| Source File | Funcs | Candidates | Tested | Coverage | Status    |
|-------------|-------|------------|--------|----------|-----------|
| adler32.c   | 2     | 2          | 2      | 100%     | covered   |
| compress.c  | 4     | 4          | 3      | 75%      | partial   |
| crc32.c     | 2     | 2          | 2      | 100%     | covered   |
| uncompr.c   | 3     | 2          | 1      | 50%      | partial   |
| deflate.c   | 4     | 2          | 0      | 0%       | untested  |
| inflate.c   | 3     | 2          | 0      | 0%       | untested  |

## Untested Functions (2)

### state-machine (2)
| Function      | Source    | Module       |
|---------------|-----------|--------------|
| inflateInit_  | inflate.c | campaign-2   |
| deflate       | deflate.c | campaign-2   |

## Recommended Focus

> **Priority 1 — Fix failing tests**
> | Function          | Source    |
> |-------------------|-----------|
> | uncompress_auto   | uncompr.c |

> **Priority 2 — Test untested PBT candidates (strongest oracle first)**
> | Priority | Function      | Source    | Oracle         | Campaign   |
> |----------|---------------|-----------|----------------|------------|
> | T1       | inflateInit_  | inflate.c | state-machine  | campaign-2 |
> | T1       | deflate       | deflate.c | state-machine  | campaign-2 |

> **Priority 3 — Scan uncovered files**
> 2 file(s) not yet scanned: deflate.c, inflate.c
> Run `pi-pbt scan --dir <dir> --out ./pbt-out` to add them to FUNCTION_INDEX.md.
```

### `pi-pbt coverage --diff <sha>` Diff 登记段

在完整登记报告末尾追加的 diff 作用域视图。

**示例：**
```
## Diff Coverage

> Scope: commit `2fcbd5b0` — functions matched from changed lines
> Changed source files: compress.c

| Metric                                   | Value            |
|------------------------------------------|------------------|
| Functions changed in diff (heuristic)    | 1                |
| PBT candidates (changed)                 | 1                |
| Tested (of changed candidates)           | 1 / 1 (100%)    |
| Test gaps                                | 0                |
```

- 若新增行能匹配受支持的函数定义正则，函数名集合用于筛选索引；匹配只按函数名，重载或
  不同文件同名函数可能产生歧义。
- 若没有识别出函数定义，则回退为改动文件中索引到的全部函数；此时报告会明确写
  `functions in changed files (function-level scope unavailable)`。
- `Tested` 来自 `COVERAGE.md` / registry 中状态为 `covered` 的登记，不是运行时命中数。
- `Test gaps` 是该启发式台账范围减去 covered 登记数。

JSON 模式也支持 `--json --diff <sha>`：
```json
{
  "totalCovered": 1,
  "totalFunctions": 17,
  "diffSha": "2fcbd5b0...",
  "diffCoverage": { "totalCovered": 1, "totalFunctions": 1, ... },
  "diffChangedFiles": ["compress.c"]
}
```

---

## 推荐使用流程

```bash
# 1. 先确认启发式索引是否符合目标范围
pi-pbt scan --dir /path/to/project/src --out /path/to/project/pbt-out

# 2. 对索引中的候选发起 campaign
pi-pbt test-all --dir /path/to/project --out /path/to/project/pbt-out \
  --provider <provider> --model <model>

# 3. 查看 PBT 覆盖登记（在包含 pbt-out/ 的目录运行）
cd /path/to/project
pi-pbt coverage --list

# 4. 单提交门禁；调用方先保证工作树就是该提交
pi-pbt hook-run <full-sha> --repo /path/to/project --out /path/to/run-output

# 5. 如需完整 diff 台账段，手动查询
pi-pbt coverage --diff <full-sha> --project-dir /path/to/project
```

自动追加只是方便阅读；判断 campaign 是否完成，应看 `report.json` / `REPORT.md`、
构建与测试日志，以及可用时的原生代码覆盖执行证据。不要仅凭登记百分比宣布真实代码
已经覆盖。

---

## 手动查询

| 方式 | 命令 | 内容 |
|------|------|------|
| **全文登记报告** | `pi-pbt coverage --list` | 台账的 Summary / Module / Oracle / File / Untested / Recommended Focus 等章节 |
| **精简登记报告** | `pi-pbt coverage` | 隐去 Untested Functions、File Coverage、Files Not Yet Scanned；其他章节仍可能保留 |
| **JSON 登记数据** | `pi-pbt coverage --json` | `CoverageStatusReport` 字段；有可计算的 diff 时再附 `diffCoverage`、`diffChangedFiles`、`diffSha` |
| **Diff 登记视图** | `pi-pbt coverage --diff <sha> --project-dir .` | 在文本输出末尾追加启发式 Diff Coverage 段；若无索引或无法形成范围，可能没有该段 |
| **自动追加** | campaign 完成后 | 有台账且有 `REPORT.md` 时追加登记报告；有原生 profile 时另追加 execution evidence |
