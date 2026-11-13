# fmbot-pbt

[中文](#中文) · [Changelog / 更新日志](CHANGELOG.md) · [Usage guide](docs/usage.md) · [使用指南](docs/usage.zh.md)

fmbot-pbt is an agent for **property-based testing**: it extracts the properties code must satisfy from its documented
contracts, checks them against large numbers of generated inputs, and reports the bugs it finds with minimal
reproducers. This repository distributes the released binaries.

## Download

| Platform | Download | Checksum |
|---|---|---|
| Linux x64 | [`fmbot-pbt-linux-x64.tar.xz`](https://github.com/fermat-hkrc/FMBot-PBT-release/releases/latest/download/fmbot-pbt-linux-x64.tar.xz) | [`.sha256`](https://github.com/fermat-hkrc/FMBot-PBT-release/releases/latest/download/fmbot-pbt-linux-x64.tar.xz.sha256) |
| Linux arm64 (aarch64) | [`fmbot-pbt-linux-arm64.tar.xz`](https://github.com/fermat-hkrc/FMBot-PBT-release/releases/latest/download/fmbot-pbt-linux-arm64.tar.xz) | [`.sha256`](https://github.com/fermat-hkrc/FMBot-PBT-release/releases/latest/download/fmbot-pbt-linux-arm64.tar.xz.sha256) |
| macOS Apple Silicon | [`fmbot-pbt-macos-arm64.tar.xz`](https://github.com/fermat-hkrc/FMBot-PBT-release/releases/latest/download/fmbot-pbt-macos-arm64.tar.xz) | [`.sha256`](https://github.com/fermat-hkrc/FMBot-PBT-release/releases/latest/download/fmbot-pbt-macos-arm64.tar.xz.sha256) |
| Windows x64 (experimental) | [`fmbot-pbt-windows-x64.tar.xz`](https://github.com/fermat-hkrc/FMBot-PBT-release/releases/latest/download/fmbot-pbt-windows-x64.tar.xz) | [`.sha256`](https://github.com/fermat-hkrc/FMBot-PBT-release/releases/latest/download/fmbot-pbt-windows-x64.tar.xz.sha256) |

```bash
tar -xJf fmbot-pbt-linux-x64.tar.xz && cd fmbot-pbt-linux-x64 && ./install.sh
```

Then configure a model: [installation guide](docs/installation.md).

## Two modes

- **While you develop:** invoke `/skill:pbt-build-run` with documents, scope, build command and effort; see the [usage guide](docs/usage.md).
- **Merge-request gate:** `fmbot-pbt hook-run <MR head commit> --base <base commit>` in the pipeline; the exit code
  passes or fails it. Configurations: [CI examples](docs/ci-examples.md).

Inputs (documents, optional; code; build command, optional) and outputs: [usage guide](docs/usage.md#inputs-and-outputs).
Step by step: [usage guide](docs/usage.md). Every command and flag: [reference](docs/subcommands.md).

Found a problem? Open an [issue](https://github.com/fermat-hkrc/FMBot-PBT-release/issues).

---

## 中文

fmbot-pbt 是做**性质测试**的智能体：从文档约定中提取代码必须满足的性质，用大量自动生成的输入检查这些性质，
报告找到的缺陷和最小复现。本仓库发布可执行文件。

下载上表中对应平台的压缩包，然后：

```bash
tar -xJf fmbot-pbt-linux-x64.tar.xz && cd fmbot-pbt-linux-x64 && ./install.sh
```

接着配置模型，见[安装指南](docs/installation.zh.md)。

两种用法：

- **边开发边用：**显式调用 `/skill:pbt-build-run`，给出文档、代码范围、构建命令和 effort，见[使用指南](docs/usage.zh.md)。
- **MR 门禁：**在流水线里运行 `fmbot-pbt hook-run <MR 头提交> --base <基准提交>`，退出码决定流水线是否通过。
  配置见 [CI 配置示例](docs/ci-examples.zh.md)。

输入（文档可选、代码、编译命令可选）与输出见[使用指南](docs/usage.zh.md#输入与输出)，分步说明见[使用指南](docs/usage.zh.md)，全部命令与参数见[参考手册](docs/subcommands.zh.md)。

有问题请提 [issue](https://github.com/fermat-hkrc/FMBot-PBT-release/issues)。
