# Kea GUI 测试（实验性）

[English version / 英文版](kea.md) · [返回安装指南](installation.zh.md)

`pi-pbt kea` 是一个**实验性**入口，用来测试已经装在 USB 手机上的 HarmonyOS
应用。它把应用当作黑盒，通过 `hdc` 驱动 Kea2 引擎：Kea2 一边自动探索 GUI，
一边检查“无论用户怎么操作都应该成立”的性质，并报告崩溃、ANR 和性质违例。

## 前置条件

运行前有三样东西必须就位：

| | |
|---|---|
| 手机 | USB 连接，且解锁不需要 PIN；`hdc` 在 `PATH` 上（`hdc list targets` 能看到序列号） |
| Kea2 | 一份带虚拟环境的 Kea2，即 `<kea_home>/.venv/bin/kea2`（或 `venv/bin/kea2`）存在 |
| 反编译信号 | `<decompile_home>/mined_all/<package>/signals.json`，需事先从应用挖出。没有就直接中止，不做无目标的盲目探索 |

## 配置

配置写在你启动它的那个目录里的 `kea.config.yml`（即“SUT 目录”）。最小配置：

```yaml
package: com.example.app
kea_home: /path/to/Kea2
decompile_home: /path/to/harmony-decompile
```

然后：

```bash
cd /path/to/sut-folder     # 放 kea.config.yml 的目录
pi-pbt kea                 # 交互式
pi-pbt kea -p              # headless（CI、nohup）
pi-pbt kea -c              # 续跑：沿用上一次的产物目录
pi-pbt kea --config other.yml --lang zh
```

命令行参数就这四个，其余全在配置文件里：

| 配置项 | 默认值 | 含义 |
|---|---|---|
| `package` | **必填** | 被测应用的 bundle 名 |
| `kea_home` | `~/github/Kea2` | Kea2 所在目录（其虚拟环境提供 `kea2` 命令） |
| `decompile_home` | **必填** | 存放 `mined_all/<package>/signals.json` 的目录 |
| `device` | 当前连着的那台 | 设备序列号，接了多台手机时需要 |
| `out` | `pbt-out` | 产物目录，相对 SUT 目录 |
| `depth` | 由 effort 档位决定 | `fast`（短跑）或 `deep`（跑完整的性质包） |
| `events`、`running_minutes`、`throttle` | 15 / 6 / 500（fast），70 / 12 / 200（deep） | 探索预算：最大步数、时长（分钟）、两次事件间隔（毫秒） |
| `mode_a_packs` | 内置的那几个包 | 仅 `deep`：要跑的性质包所在的 Python 模块 |
| `stamp_runs` | `true` | 每次运行单独一个带时间戳的目录；`false` 则平铺写进 `out` |
| `provider`、`model`、`lang` | —— | 与全局选项含义相同 |

depth 和安装指南里的 [effort 档位](installation.zh.md#挖多深effort-档位)是同一个
“挖多深”的旋钮，不重复设：配置里的 `depth:` 优先级最高，其次
`PBT_KEA_DEPTH`，最后是档位（`quick` → `fast`，`standard`/`thorough` →
`deep`）。

## 产物与失败

一次运行留下的产物：

```text
pbt-out/
  LATEST                                    # 指向最新一次运行的目录
  runs/<package>_modeB_<depth>_<时间戳>/
    layout.json      一次界面 dump
    kea-run/         Kea2 自己的输出（res_*/result_*.json）
    LAST_RUN.json    解析出来的计数：执行数、失败数、每条性质的统计
    REPORT.md        结论
```

配置文件不存在、配置里没写 `package:`、或反编译信号缺失时，它会打印具体原因
并以退出码 `1` 结束，不会去动手机。

## 环境变量

| 变量 | 作用 |
|---|---|
| `PBT_KEA_DEPTH=fast\|deep` | 探索深度，覆盖 effort 档位；配置里的 `depth:` 仍然优先 |
| `PBT_KEA_MODE_A_PACKS="pkg.a pkg.b"` | 配置里没写 `mode_a_packs` 时，`deep` 下要跑的性质包 |

通用的 `PBT_EFFORT=quick|standard|thorough` 只在 `kea.config.yml` 未设置
`depth:` 且未设置 `PBT_KEA_DEPTH` 时提供深度：`quick` 选择短跑 `fast`，
`standard` 和 `thorough` 选择 `deep`。
