# 覆盖率跟踪功能

pi-pbt 覆盖率跟踪功能注册、追踪并报告项目函数的 PBT 测试进度，同时支持**全量**和**增量（diff）**两种视角。

覆盖率数据**全自动产生**——用户只需运行测试命令，不需要手动执行 scan 或查询报告。

---

## 子命令

### `pi-pbt scan <dir>`

纯本地操作（零 LLM 调用），用 `rg` 提取全部源文件中的函数定义，写入 `pbt-out/FUNCTION_INDEX.md`。

```
pi-pbt scan --dir /path/to/project --out ./pbt-out

→ pbt-out/FUNCTION_INDEX.md
```

支持语言：C/C++、Python、Rust、TypeScript/JavaScript、Go。

> **通常不需要手动执行。** `hook-run` 和 `test-all` 在启动前会自动执行 scan（见下文）。 此命令只在你想单独查看函数清单时有用。

### `pi-pbt test-all`

读 `pbt-out/FUNCTION_INDEX.md`，收集所有 PBT 候选函数，构造一个 prompt 一次性提交给 pi agent 逐个测试。 产物写入 `pbt-out/`。

```
pi-pbt test-all --dir . --out ./pbt-out --provider deepseek
```

**自动 scan：** 若 `pbt-out/FUNCTION_INDEX.md` 不存在，启动前自动执行 rg 扫描。

### `pi-pbt hook-run <sha>`

标准增量 PBT 测试命令。 在 CI/hook 场景下使用。

```
pi-pbt hook-run <sha> --repo . --out ./pbt-out
```

**自动 scan：** 启动前自动执行 rg 扫描生成 `pbt-out/FUNCTION_INDEX.md`。

### `pi-pbt coverage --diff <sha>`

在全量覆盖率报告末尾追加一个 **Diff 覆盖率** 段。 分母是 git diff 中实际被修改的函数（解析 patch 中的新增行匹配函数定义），不是该文件中的所有函数。

```
pi-pbt coverage --diff <sha> --project-dir .
```

---

## 自动报告

每次 `hook-run` 或 `test-all` 完成后，自动执行：

1. **打印覆盖率摘要到 stderr**
2. **追加完整覆盖率报告到 `pbt-out/REPORT.md`**（追加在末尾的 `## Coverage Report` 章节）
3. **（仅 hook-run）自动输出 diff 覆盖率**，基于本次提交的 sha

无需手动执行 `pi-pbt coverage`。

---

## 输出产物

### `pbt-out/FUNCTION_INDEX.md`

全量函数索引。由 `pi-pbt scan` 生成，也可由 agent 在 campaign 的 Scan 阶段生成（必须枚举所有源文件，不是可选步骤）。

**格式：**
```markdown
> Total files: <N> | Total functions: <M> | PBT candidates: <K> | Excluded: <J>

| Function | Source File | Line | Kind | PBT Candidate | Reason |
|----------|-------------|------|------|---------------|--------|
| compress2 | compress.c | 67 | function | yes | round-trip |
| compress | compress.c | 82 | function | no | wrapper |
```

- 表格单元格必须为纯文本，不得使用 `**bold**`、`` `code` `` 或 `*italic*` 等 Markdown 格式

### `pbt-out/COVERAGE.md`

测试结果记录，由 agent 在 Review 阶段写入。

**格式：**
```markdown
# PBT Coverage Status

| Function | Source file | Test file | Test target | Notes |
|----------|-------------|-----------|-------------|-------|
| compress2 | compress.c | t.cpp | t | pass (4/4 properties) |
| uncompress_auto | uncompr.c | t2.cpp | t2 | fail (3/6 properties failing) — see bug report |
```

- `Source file` 列必须为纯文件名，**不得追加行号后缀**（如 `compress.c:101`），否则文件级匹配会失败

### `pbt-out/COVERAGE_STATUS.md`

由 agent 在 campaign 结束时生成，是全量覆盖率报告的一个快照。 可选产物。

### `pi-pbt coverage --list` 全量覆盖率报告

从 FUNCTION_INDEX.md + COVERAGE.md + SCAN.md 综合生成的完整报告。

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
> Run `pi-pbt scan <dir>` to add them to FUNCTION_INDEX.md.
```

### `pi-pbt coverage --diff <sha>` Diff 覆盖率段

在全量报告末尾追加的 diff 作用域覆盖率。

**示例：**
```
## Diff Coverage

> Scope: commit `2fcbd5b0` — functions actually changed
> Changed source files: compress.c

| Metric                                   | Value            |
|------------------------------------------|------------------|
| Functions changed in diff                | 1                |
| PBT candidates (changed)                 | 1                |
| Tested (of changed candidates)           | 1 / 1 (100%)    |
| Test gaps                                | 0                |
```

- `PBT candidates (changed)` = 从 git diff patch 中解析出的新增函数定义（`^+type name(params) {`）。 不受同文件中其他函数影响
- `Tested (of changed candidates)` = COVERAGE.md 中匹配这些函数的记录
- `Test gaps` = 修改了但未测试的函数数

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

## 使用流程（全自动）

```
# 1. 全量 PBT（自动 scan → 测试 → 报告）
pi-pbt test-all --dir /path/to/project --out ./pbt-out --provider deepseek
#    ↑ 自动 rg 扫描 → FUNCTION_INDEX.md → 测所有候选 → 打印覆盖率 → 追加到 REPORT.md

# 2. 增量开发循环
git commit -m "feat: add new function"
pi-pbt hook-run HEAD --repo . --out ./pbt-out
#    ↑ 自动 rg 扫描 → FUNCTION_INDEX.md → 测 diff → 全量覆盖率 + diff 覆盖率
```

每条命令做完，覆盖率**自动打印 + 自动写 REPORT.md**。 不需要手动查。

---

## 手动查询

| 方式 | 命令 | 内容 |
|------|------|------|
| **全文报告** | `pi-pbt coverage --list` | 所有章节（Summary / Module / Oracle / File / Untested / Recommended Focus） |
| **摘要报告** | `pi-pbt coverage` | 仅 Summary + Module + Oracle 章节 |
| **JSON** | `pi-pbt coverage --json` | 结构化数据，含所有字段供 CI 消费 |
| **Diff 覆盖率** | `pi-pbt coverage --diff <sha> --project-dir .` | 在全文报告末尾追加 Diff 段 |
| **自动报告** | `hook-run` / `test-all` 完成后自动 | 打印到 stderr + 追加到 REPORT.md |
