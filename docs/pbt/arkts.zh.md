# ArkTS PBT（HarmonyOS HAP）

[English version](arkts.md)

[返回安装说明](../installation.zh.md#harmonyos-hap--arkts-应用) ·
[子命令](../subcommands.zh.md)

pi-pbt 支持的 ArkTS 路径是 **HarmonyOS HAP ohosTest**，不是给任意 `.ets`
开一台主机编译器。性质测试落在
`src/ohosTest/ets/test/<Name>Pbt.test.ets`，框架是 Hypium + fast-check；用
`hvigorw` 组装、签名、安装，再以 `hdc shell aa test` 跑。裁决看
`OHOS_REPORT_RESULT` 那一行。`hvigorw` 成功只证明编过，不证明测过。

这不是 Kea（`pi-pbt kea` 是黑盒 GUI）。这不是 OpenHarmony GN C++。这不是
koala / PandaCheck。

## 什么时候算 HAP

目录里（本级或上级）有 `oh-package.json5` 和 `build-profile.json5` 就是
**HAP 根**。项目内的 `hvigorw` 包装不是必需的：工具链取最先解析到的
`hvigorw`（项目包装、`PATH`、或 DevEco 安装）。只在 `PATH` 上有 `hvigorw`
不会把当前目录变成 HAP 根。

`build-run` 碰到 HAP 根会为该进程设 `PBT_HAP=1`。不要把 `PBT_HAP` 写进
shell profile：残留的 `1` 会把之后的 C++ campaign 也逼进 HAP 模式。树是
ArkTS 但缺清单、又要用自由入口 `pi-pbt` / `-p` 时，只在**这一条命令**上设
`PBT_HAP=1`。

## 怎么跑

`--repo` 是被测模块（扫描树、`--scope`、默认 `pbt-out/`）。`--workdir` 是
`--build-cmd` 的执行目录。两者必须在**同一份 git 检出**里。`--workdir` 常常是
HAP 工程；`--repo` 可以是同时装着 `.ets` / `.ts` SUT 和该 HAP 的上级组件。

两个路径都是 HAP 本身时：

```bash
pi-pbt build-run \
  --build-cmd "hvigorw assembleHap --mode module -p module=entry@ohosTest -p product=default --no-daemon" \
  --repo /path/to/hap \
  --workdir /path/to/hap \
  --scope entry/src/main/ets/common/Utils.ets \
  --func deepCopyInter \
  --lang zh
```

SUT 文件在 HAP 目录外、但仍在同一检出内时：

```bash
pi-pbt build-run \
  --build-cmd "hvigorw assembleHap --no-daemon" \
  --repo /path/to/component \
  --workdir /path/to/component/.../the-hap \
  --out /path/to/component/pbt-out-myfunc \
  --scope path/from/repo/to/file.ts \
  --func myFunc \
  --lang zh
```

`--func` 必须带 `--scope`，且写在 `--scope` **后面**。`--out` 可选（默认
`<repo>/pbt-out`）。自定义 `--out` 目录可以放 `REPORT.md`。

`--build-cmd` 是操作者输入。失败则战役不启动（`BUILD FAILED — PBT not started`）。
`true` 这类空操作不能证明 `passing` / `failing`。HAP 战役不要加 `--prompt`，
除非有 SOP 不知道的领域事实。

工具链、SDK 对齐、fast-check HAR、签名、`aa test` 见 `arkts-build-run` 技能卡。
pi-pbt **不会**静默改写 `compileSdkVersion`、`runtimeOS` 或 hvigor 钉。自己对齐，
或让战役大声失败。

## `passing` / `failing` 性质必须满足什么

套件是 `src/ohosTest/ets/test/*.test.ets`，并登记到 `List.test.ets`。
`report.json` 的 `sourceFile` / `functionName` 钉在硬性 `--scope` / `--func` 上。

| 允许 | 不是在测 `--scope` |
|---|---|
| 值导入能解析到检出里的 scoped 文件 | 对镜像全局做 `declare class` |
| 若本 HAP 无法按现状导入 `--scope`，则 `retired` | 仅 `import type`，或注释里提一下路径 |
| | 拷贝/硬链到 `ohosTest/` 下（`ets/sut/`、`scope_utils/`、假的 `--scope` 树） |
| | 把 `@kit.ArkUI` / `@ohos.arkui*` 当成 koala / `arkts_frontend` 的 sourceFile |
| | 改 `--scope` 好让导入通过（`export class`、放宽 `private`） |
| | Node / 主机 `es2panda` 镜像，或 Linux Previewer 上的 `hvigorw test` |
| | `testFile` 不在 `ohosTest/ets/test/*.test.ets` |

目录形式的 `--scope` 是路径前缀。每条性质的导入必须是**该条**在目录下的
`sourceFile`，不能是任意兄弟文件。

若 HAP 编不过 scoped 文件，**retired**。不要靠改生产文件让它可导入。

## 什么不是 HAP

| 树 | 为什么 HAP PBT 跑不起来 |
|---|---|
| koala / `arkoala-arkts` / `#generated` | `@koalaui/*`。那边的测试是 Panda `make`，不是 `aa test`。 |
| `state_mgmt` 的 `utils.ts` 等未导出框架 TS | unittest HAP 按现状编不进该文件。既有 UT 对镜像全局 `declare class`；那不是 `--scope`。 |
| `test/component_test` 与许多 `examples/` HAP | 常钉在本机没有的 SDK。门禁失败。不要在 pi-pbt 里改靶。 |
| OpenHarmony GN `build.sh` / `ohos_unittest` | C++ 产品路径。`.ets` 的 `--scope` 不是那条路。 |

## 证据

`hdc shell aa test` 与完整的 `OHOS_REPORT_RESULT` 行。对 `modules.abc` 做
`strings` 只说明 SUT 被链进，不说明性质跑过。ohosTest 没有原生行覆盖。

fast-check 不来自 ohpm 注册表。用
`arkts-build-run/scripts/make-fast-check-har.sh`，再在 `oh-package.json5` 里
引用本地 HAR。

## 当前进度

这是当前 `main` 上的 HAP ohosTest 路径，不是通用 ArkTS 引擎。跟踪议题：
[#366](https://github.com/fermat-hkrc/FMBot-PBT/issues/366)。

### 已支持

- 识别 HAP 根（`oh-package.json5` + `build-profile.json5`）并设 `PBT_HAP=1`。
  在 `src/ohosTest` 跑 Hypium + fast-check。
- 以操作者的 `hvigorw` 命令为门禁。签名、安装、`hdc shell aa test`。
- `--repo`（扫描 / `--scope`）与 `--workdir`（构建）可分离，同一 git 检出。
  自定义 `--out` 可放 `REPORT.md`。
- HAP **拥有** scoped `.ets` 且能导入时：执行该符号并记 `passing` / `failing`。
- 已落地的诚实门禁：禁止 `ohosTest/ets/sut/` 抄写；禁止用 `@kit.ArkUI` 冒充
  koala；`sourceFile` / `functionName` 钉在硬性 `--scope` / `--func`；
  `declare class` 不算导入；禁止改写 `--scope` 好让导入通过；目录
  `--scope` 按路径前缀且按该条性质的 `sourceFile` 匹配。
- 若本 HAP 无法按现状导入 `--scope`，**retired**。
- fast-check 用本地 HAR（`make-fast-check-har.sh`），不是 ohpm 注册表。
- **HAR 能驱动的 SUT 类型：** 已导出的 `number` / `string` / `boolean` /
  其数组，以及可写成 `.ets` 字面量的普通对象。例如
  `export function add(a: number, b: number): number`。

### 不支持

两张表。第一张是 HAP runner。第二张是 **`fast-check.har` 喂不了的 ArkTS SUT**。
任一列要关掉，都需要新的 pi-pbt 工作。

#### Runner / 树

| 缺口 | 现在会怎样 | 要改进什么 |
|---|---|---|
| koala / `@koalaui` / `#generated` | 诚实收尾是全 `retired`。编不进 ohosTest。 | koala/Panda runner，或完整 OH 镜像路径，不是 HAP ohosTest。 |
| 未导出框架 `.ts`（如 `state_mgmt` `utils.ts`） | ohosTest 导不进该类。既有 UT 对镜像全局 `declare class`。 | 继续 retired（当前诚实结果），或在**不改文件**的前提下编进它。 |
| SDK / hvigor / `runtimeOS` 不匹配 | 门禁失败，或工具链被推迟。pi-pbt 不改清单。 | 大声报出期望的 SDK 字段。操作者预检，不要静默改靶。 |
| `test/component_test` 与许多 `examples/` HAP | 常是 API 12 / HarmonyOS 5.x；本机 SDK 26 树装不了。 | 同上：大声失败。不要改写 `compatibleSdkVersion`。 |
| OpenHarmony GN `build.sh` | 独立的 C++ 产品路径。`.ets` 的 `--scope` 不是那条路。 | 保持拆分。ArkTS `--scope` 不要起 GN。 |
| 原生行覆盖 | ohosTest 链不上 `coverage_gaps`。证据是 `aa test` 加 abc 字符串。 | 若工具链能产出 ArkTS 覆盖再做；不要伪造 C++ lcov。 |
| 全 `retired` 的 `report.json` | **空操作** `--build-cmd`（`true` / `exit 0`）已允许全 `retired`。**真实成功**的 HAP 构建（`hvigorw …`）若每一行都是 `retired`，仍会以「没有性质被执行」被拒。 | 在 HAP 已构建且无法导入 `--scope` 时允许全 `retired`——不只是构建命令为空操作的情形。 |
| 兄弟导出 / 私有字段绕行 | 套件可导入 `--scope` 里已导出的 helper，再在活实例上调未导出方法。这不是改写，导入检查会放行。 | 决定 `passing` 是否要求导入**符号**，而不只是定义它的文件。 |
| 主机 `es2panda` / PandaCheck | 已移除。不要当 HAP 替身加回来。 | 仅当真实 Panda 测试树就是 SUT，并自有 runner。 |
| Kea GUI | `pi-pbt kea` 是另一模式（黑盒 UI）。 | 与源码级 ArkTS 分开。 |

#### ArkTS SUT 与 `fast-check.har`

HAR 是转换后的 JS（对 npm fast-check 做 `ohpm convert`）。它生成 JS 的
`number` / `string` / `boolean` / `Array` / 普通对象。性质文件是 ArkTS
（命名导入、`(x: number)`、无 `any`、无解构、`check()` 而非 `fc.assert`）。
当 ohosTest 无法用这些值调用该函数时，该 SUT 特性就不支持。

| SUT 特性 | 现在会怎样 | 要改进什么 |
|---|---|---|
| `@Sendable` / 并发类型 | `integer()` 是 JS `number`，不是 `Box`。`boxId(n)` 类型过不了。在谓词里 `new Box(n)` 测的是 `boxId`，不是 Sendable（无跨线程对象、无 `SendableMap`）。 | ArkTS 形态的 arbitrary，或能构造 Sendable 值的 runner。在此之前，参数只有 Sendable 的 `--func` 应 retired。 |
| 未导出 `class` / `private` 字段 | 不改 `--scope` 就无法 `import` 或 `new`。 | 不改写 SUT 也能编/导，或 retired。 |
| 联合 / 可选 / `T \| null` | 无类型化 arbitrary。手写 `oneof` 仍可能过不了 `.ets` 回调类型。 | 适配器（`arkNullish()`、`arkEnum`），`.ets` 能直接写、不靠 `ESObject`。 |
| `enum`、`struct`、品牌/不透明 SDK 类型 | HAR 不会 `new` 它们。 | 套件里的类型化构造，或 retired。 |
| 重载 | 一次 JS 调用；HAR 没有 ArkTS 重载解析。 | 每个签名一条性质，来自 scoped 符号。 |
| `throw` / `BusinessError` | `check()` 要 `boolean`。try/catch 靠手写。 | Hypium 能打印的否定/错误 oracle。 |
| 泛型（`foo<T>`） | HAR 没有 `T`。 | 在套件里按扫描到的签名实例化，或 retired。 |
| 回调 / 函数型参数 | 没有 ArkTS 合法的 `func()` arbitrary。 | 在 `.ets` 里显式 stub，或 retired。 |
| `@Component` / `@Observed` / UI | 需要 UI 运行时。 | 不是 HAP fast-check。Kea 或 UI runner。 |
| koala / `@koalaui` | 另一套编译器。HAR 永远装不上。 | Panda/koala runner，不是 ohosTest。 |
| Native / NAPI / 仅 `ESObject` 的 API | `ESObject` 能编过，但丢掉 ArkTS 类型。 | 类型化绑定，或 retired。 |

### 下一步改进

1. **真实** HAP 构建的全 `retired` 收尾：`hvigorw` 已成功、且因无法导入
   `--scope` 而每一行都是 `retired` 时，`report.json` 必须通过校验。
   （空操作 `--build-cmd` 已允许全 `retired`；那是另一回事。）
2. SDK 预检：点名缺失的 `compileSdkVersion` / `runtimeOS` / hvigor 插件并停下，
   不改 SUT 树。
3. 符号级导入：`passing` / `failing` 必须绑定 `--func`，不能只值导入碰巧定义
   它的文件。
4. Koala / Panda：文档化的非 HAP runner，或继续对 koala `--scope` retired。
5. **ArkTS SUT 类型：** 不要在 `.ets` 里硬撑 TS fast-check。先加 nullish/enum/object
   的 `.ets` 适配器，再做 Sendable（或对这些 `--func` retired）。长期：同
   `check`/`property` 形态的 ArkTS PBT 库，无 `any`、无 `<const T>`。扫描 scoped
   签名，只允许匹配的 arbitrary。

## 相关

- [安装：HAP / ArkTS 应用](../installation.zh.md#harmonyos-hap--arkts-应用)
  （[English](../installation.md#harmonyos-hap--arkts-apps)）
- [英文 ArkTS 页](arkts.md)
- [子命令](../subcommands.zh.md) — `build-run --build-cmd "./hvigorw assembleHap"`
- [Kea GUI 测试](../kea.zh.md) — 真机黑盒 UI，不是源码级 ArkTS
- 内置 `arkts-build-run` skill：DevEco、SDK 对齐、fast-check HAR、签名、`aa test`
