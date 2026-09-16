# pi-pbt — releases

[中文](#中文) · [Changelog / 更新日志](CHANGELOG.md) · [Installation guide](docs/installation.md) · [安装指南](docs/installation.zh.md)

pi-pbt is an AI agent that runs **property-based testing** campaigns on a
codebase. Point it at a Python, Rust, Go, Java, or C++ repository and it reads
the source, works out what must always be true of it, writes and runs tests that
check those rules against large numbers of generated inputs, and reports the bugs
it finds with a minimal reproducer for each.

**This repository distributes the released binaries.** The source lives in a
separate repository; here you get the executables and the installation guide.

## Download

A self-contained main executable — no Node.js or Bun runtime. The archive also
includes an installer and search tools; tested projects need their own toolchain.

| Platform | Download | Checksum |
|---|---|---|
| Linux x64 | [`pi-pbt-linux-x64.zip`](https://github.com/fermat-hkrc/FMBot-PBT-release/releases/latest/download/pi-pbt-linux-x64.zip) | [`.sha256`](https://github.com/fermat-hkrc/FMBot-PBT-release/releases/latest/download/pi-pbt-linux-x64.zip.sha256) |
| Linux arm64 (aarch64) | [`pi-pbt-linux-arm64.zip`](https://github.com/fermat-hkrc/FMBot-PBT-release/releases/latest/download/pi-pbt-linux-arm64.zip) | [`.sha256`](https://github.com/fermat-hkrc/FMBot-PBT-release/releases/latest/download/pi-pbt-linux-arm64.zip.sha256) |
| macOS Apple Silicon | [`pi-pbt-macos-arm64.zip`](https://github.com/fermat-hkrc/FMBot-PBT-release/releases/latest/download/pi-pbt-macos-arm64.zip) | [`.sha256`](https://github.com/fermat-hkrc/FMBot-PBT-release/releases/latest/download/pi-pbt-macos-arm64.zip.sha256) |

Not sure which Linux build you need? `uname -m` — `x86_64` takes the x64 file,
`aarch64` the arm64 one. Older versions are on the
[releases page](https://github.com/fermat-hkrc/FMBot-PBT-release/releases).

```bash
sha256sum -c pi-pbt-linux-x64.zip.sha256    # optional integrity check
unzip pi-pbt-linux-x64.zip                  # yields pi-pbt-linux-x64/, already executable
cd pi-pbt-linux-x64                         # holds pi-pbt, install.sh, and tools/
./install.sh
pi-pbt --help
```

The archive also carries the two search tools the agent uses (`fd`, `rg`), so it
works on a machine that has neither; the installation guide shows where to put
them.

Then configure a model and start a run — see the
**[installation guide](docs/installation.md)**, which also covers the toolchains
the language under test needs, CI and git-hook integration, and the live
dashboard.

A completed source campaign writes `REPORT.md` for people and **`report.json`**
for CI/integrations, plus detailed bug reports. Start with the
[results guide](docs/installation.md#read-the-results); use the
[CLI reference](docs/subcommands.md) to choose a mode and the
[reproduction guide](docs/reproducing.md) to keep the generated tests.

Found a problem? Open an [issue](https://github.com/fermat-hkrc/FMBot-PBT-release/issues).

---

## 中文

pi-pbt 是一个做**性质测试**(property-based testing)的 AI agent。把它指向一个
Python / Rust / Go / Java / C++ 仓库,它会读源码,推断出「这段代码无论输入什么都
必须成立的规律」,据此写测试并用大量自动生成的输入去跑,最后把找到的 bug 连同
最小复现一起报出来。

**本仓库只发布构建产物**:源码在另一个仓库,这里提供可执行文件和安装文档。

下载对应平台的压缩包(见上表),然后:

```bash
sha256sum -c pi-pbt-linux-x64.zip.sha256    # 可选:校验完整性
unzip pi-pbt-linux-x64.zip                  # 解出 pi-pbt-linux-x64/,已带可执行权限
cd pi-pbt-linux-x64                         # 里面是 pi-pbt、install.sh 和 tools/
./install.sh
pi-pbt --help
```

压缩包里还带了 agent 用到的两个搜索工具(`fd`、`rg`),所以在没装它们的机器上也
能用;放置位置见安装指南。

主程序是自包含二进制，不需要 Node.js/Bun；发行包另带安装脚本和搜索工具，
被测项目仍需自己的工具链。不确定该下哪个
Linux 版本就看 `uname -m`:`x86_64` 用 x64,`aarch64` 用 arm64。

接下来配置模型、跑第一次测试,见 **[安装指南](docs/installation.zh.md)** —— 里面
还写了被测语言需要的工具链、CI 与 git hook 接入方式,以及实时看板。

完成源码测试后，`REPORT.md` 给人看，**`report.json`** 给 CI/程序读取，另有逐个问题的
详细报告。先看[运行后看什么](docs/installation.zh.md#运行后看什么)，选运行方式看
[子命令参考](docs/subcommands.zh.md)，固化生成用例看[复现与回归](docs/reproducing.md)。

有问题请提 [issue](https://github.com/fermat-hkrc/FMBot-PBT-release/issues)。
