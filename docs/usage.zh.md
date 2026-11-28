# 使用指南

[English](usage.md)

fmbot-pbt 有三种用法：

| | 模式一：边开发边用 | 模式二：MR 门禁 | 模式三：全量模式 |
|---|---|---|---|
| 谁来用 | 开发者，在 fmbot-pbt 或 coding agent 中显式启动测试 | CI 流水线 | 想对整个仓库做全面测试的开发者 |
| 测什么 | 当前改动，包括未提交的改动 | MR 的全部改动 | 整个仓库，不只改动部分 |
| 结果 | 报告，以及项目测试目录中的测试代码 | 退出码决定流水线是否通过，报告作为流水线产物 | 报告，以及项目测试目录中的测试代码 |

开始之前先按[安装指南](installation.zh.md)装好 fmbot-pbt 并配置模型。

## 输入与输出

| 输入 | 说明 |
|---|---|
| 文档（可选） | 需求文档、接口文档、注释；性质从中提取。没有文档时从代码推断意图 |
| 代码 | 被测仓库；MR 门禁时还包括本次改动的范围 |
| 编译命令（可选） | 构建被测代码和测试的命令；不给时由 fmbot-pbt 按项目自行判断 |

| 输出 | 说明 |
|---|---|
| 性质清单 | `PROPERTIES.md`：每条性质的形式化描述及其依据 |
| 测试代码 | 写入项目自己的测试目录，可直接提交 |
| 报告 | `REPORT.md`、`report.json`，以及每个缺陷一份带最小复现的报告 |

## 模式一：边开发边用

先准备好可构建、可运行的项目。在项目目录运行 `fmbot-pbt`，明确调用内置 skill，并给出本轮测试的输入：

```text
/skill:pbt-build-run --build-dir /path/to/project --target /path/to/project/src/parser
  --spec docs/parser-contract.md --effort standard --lang zh
  --build-cmd "cmake -S . -B build && cmake --build build --target parser_test"
```

这个 skill 接受 `fmbot-pbt build-run` 的参数，执行同一套构建门禁，随后在当前会话中可见地测试。把路径和构建命令替换成
项目实际值；没有需求文档时省略 `--spec`。`effort` 必须明确选 `quick`、`standard` 或 `thorough`，深度说明见
[参考手册](subcommands.zh.md#测试深度)。

要在会话中测试某个提交或 MR，改用 `/skill:pbt-hook-run` 并给出 `hook-run` 的参数，见[会话中的 skill](subcommands.zh.md#会话中的-skill)。

也可在终端直接运行同一流程。在宿主 agent 中先执行一次 `fmbot-pbt skill-install`，Claude Code 用 `/pi-pbt-dev`、
Codex 用 `$pi-pbt-dev` 显式调用 skill，让它执行这条命令：

```bash
fmbot-pbt build-run --build-dir /path/to/project \
  --target /path/to/project/src/parser --spec docs/parser-contract.md --effort standard --lang zh \
  --build-cmd "cmake -S . -B build && cmake --build build --target parser_test"
```

`--spec` 可重复传入；没有文档时省略它。大型仓库建议按模块目录分别指定 `--target` 和独立的 `--out` 分次测试。
测试结束前不要修改当前检出。完成后阅读 `pbt-out/REPORT.md`，审阅项目测试目录中生成的测试，再与代码一起提交。通过 MCP 委派的参数见[参考手册](subcommands.zh.md#mcp)。

## 模式二：MR 门禁

1. 在 runner 上安装 fmbot-pbt 并配置模型。
2. 检出 MR 代码。需要构建时，可先用项目自己的命令构建，或把这条命令作为 `--build-cmd` 传入；构建失败以 `3` 退出，
   不会开始测试。
3. 运行测试，`--base` 指定 MR 的基准提交，有需求文档时用 `--spec` 传入：

   ```bash
   fmbot-pbt hook-run <MR 头提交> --base <基准提交> --spec docs/需求.md --effort standard --target . --out /tmp/pbt-out --lang zh
   ```

4. 按退出码判定：`0` 通过；`1` 发现缺陷；`2` 报告缺失或不完整；`3` 构建失败。
5. 把 `/tmp/pbt-out` 中的 `REPORT.md`、`report.json` 和 `bug_reports/` 上传为流水线产物。

GitHub Actions 与 GitLab CI 的完整配置见 [CI 配置示例](ci-examples.zh.md)。

## 模式三：全量模式

对整个当前仓库做性质测试（PBT），而不只测改动：fmbot-pbt 会找出最值得测的目标，写性质并运行，产出 bug 报告。
在仓库目录启动 `fmbot-pbt` 用自然语言直接提出（或显式点名 `/skill:pbt-workflow`），无人值守的等价形式：

```bash
fmbot-pbt -p "对当前仓库做性质测试(PBT):找出最值得测的目标，写性质并运行，产出 bug 报告" --lang zh
```

深度可在请求里写明（如 `effort 设为 thorough`）或用 `PBT_EFFORT` 设置，默认 `standard`。大型仓库建议每次请求只点名
一个模块。产物写入 `pbt-out/`，生成的测试进项目自己的测试目录，同模式一。`-p` 运行的退出码不是判定，以
`pbt-out/REPORT.md` 为准。

## 查看结果

| 文件 | 内容 |
|---|---|
| `REPORT.md` / `REPORT.html` | 测试报告：测了什么、发现什么、如何复现 |
| `bug_reports/` | 每个缺陷一份报告，带最小复现输入 |
| `report.json` | 供程序读取的结果，格式见[报告格式](report-schema.md) |

复现失败、把生成的测试加入项目，见[复现与回归](reproducing.md)。全部命令与参数见[参考手册](subcommands.zh.md)。
