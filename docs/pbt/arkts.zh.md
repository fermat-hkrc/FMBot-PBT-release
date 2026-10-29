# ArkTS PBT（PandaCheck）

[English version](arkts.md)

[返回安装说明](../installation.zh.md#harmonyos-hap--arkts-应用) ·
[子命令](../subcommands.zh.md)

pi-pbt 支持的 ArkTS 路径是 **HarmonyOS HAP ohosTest**。性质测试是 Hypium +
PandaCheck，落在 `src/ohosTest/ets/test/<Name>Pbt.test.ets`
（`import { check, property, integer, string, boolean } from 'PandaCheck'`），
用 `hvigorw` 组装，以 `hdc shell aa test` 跑。裁决看 `OHOS_REPORT_RESULT`。
用 `make-pandacheck-har.sh` 装到 `libs/PandaCheck`。不要 `ohpm install fast-check`。

这不是 Kea。这不是 OpenHarmony GN C++。主机 `es2panda`/`ark`（`PBT_PANDA=1`）
是**另一条** runner，不要和 ohosTest 混用。

## 特性清单

依据 OpenHarmony 教程
[Introduction to ArkTS](https://github.com/openharmony/docs/blob/f41b9345badd47c7ab0c263344cd7f4b5a549afb/en/application-dev/quick-start/introduction-to-arkts.md)
的提交 `f41b9345badd47c7ab0c263344cd7f4b5a549afb`。这是该教程点名的类型、调用形态和模块。不是 ArkTS 的百分比。

`let`、`const`、类型推断、`void`、运算符、`if` / `switch` / 循环和 `this` 是语法，不计入。性质不抽取它们。

这份 34 行清单里，**14 项已支持，11 项部分支持，9 项不支持**。已支持表示 `passing` / `failing` 能使用该形态，且收尾门禁会要求它。部分支持表示该教程标题下只有一部分形态能抽取或必须捕获。不支持表示抽不了，或诚实收尾是 `retired`。

| 教程特性 | pi-pbt | 含义 |
| --- | --- | --- |
| `number` | 已支持 | `integer()` / `double()`。 |
| `bigint` | 不支持 | 没有生成器。 |
| `boolean` | 已支持 | `boolean()`。 |
| `string` | 已支持 | `string()`。 |
| `Object` | 不支持 | 不是能构造的字段袋。 |
| `number` / `string` / `boolean` 的数组 | 已支持 | `array(integer())`、`array(string())` 或 `array(boolean())`。只抽元素会被拒绝。SDK 品牌类型的空 `T[]`（`ImageItem[]`）只能传 `[]` / `constant([])`；非空品牌数组仍 `retired`。 |
| `enum` | 已支持 | 同文件，或相对导入一跳，用每个成员的 `constantFrom`。第二跳看不见。 |
| 联合 | 部分支持 | 原语、字符串字面量、数字字面量、混合字面量，以及字面量加开放原语，能抽取。不可抽的臂 `retired`。 |
| 类型别名 | 部分支持 | 可抽取的别名，如 `type Mode = 'xs' \| 'sm'`，会展开并走既有门禁。`Date` 或 class 的别名不展开。 |
| 可选参数 | 已支持 | `oneof` 加 `constant(undefined)`。 |
| 默认参数 | 已支持 | `number`、`string` 或 `boolean` 的默认参数必须既省略又传入抽取值。 |
| rest 参数 | 已支持 | `func(...xs)`，`xs` 是 `number`、`string` 或 `boolean` 的 `array()`。裸元素列表会被拒绝。 |
| 函数类型 | 部分支持 | 真正的 `(...) =>` 用具名函数或箭头。`Function` 和 `ESObject` `retired`。 |
| 重载 | 部分支持 | 每个原语臂或每种元数都要调用。非原语臂 `retired`。 |
| 带字段的 class 或 `struct` | 已支持 | 在 `--scope` 里声明时，用 `new` 或每个字段的 `record`。 |
| `static` 字段或方法 | 部分支持 | 参数为 `number`、`string` 或 `boolean` 的 public static 方法必须写成 `Class.method(drawn)`。static 字段不在此门禁内。 |
| getter / setter | 部分支持 | `number`、`string` 或 `boolean` 的 public getter 或 setter 必须在构造出的实例上读取或赋值。setter 的参数要抽取。`Date` 的 getter 不在此门禁内。 |
| 继承、`super`、抽象类 | 不支持 | 不构造。 |
| `private` / `protected` 字段 | 不支持 | 套件不得改 `--scope` 来暴露它。 |
| 对象字面量 | 部分支持 | 只限已知字段袋。`record()` 本身只有 `{w, h}`。 |
| `Record<K, V>` | 不支持 | 不是通用映射。 |
| interface | 部分支持 | `--scope` 里的字段袋可以构造。只有方法的契约不是值。 |
| 泛型函数 | 已支持 | 调用必须是 `name<string>`、`name<number>` 或 `name<boolean>`。推断调用被拒绝。 |
| 泛型约束 | 部分支持 | 约束到这三种能抽取。其他约束 `retired`。 |
| 泛型类 | 不支持 | 不构造 `CustomStack<string>`。 |
| 泛型默认值 | 部分支持 | 可抽取的默认值，如 `T = 'xs' \| 'sm'`，会展开并走既有门禁。`Date` 或 class 的默认值不展开。 |
| nullish，`T` 或 `null` / `undefined` | 已支持 | `arkNullish`。调用不得用 `!` 或 `??` 丢掉该臂。 |
| `--func` 的具名导入 | 已支持 | `passing` / `failing` 必须导入该文件并调用该符号。 |
| 导入别名或命名空间 | 已支持 | `id as ident` 和 `ids.id(...)` 算。调用兄弟符号不算。 |
| 默认导出 | 已支持 | 默认导出的函数必须按默认导入并调用。class 或值的默认导出不是这个绑定。 |
| 动态 `import()` | 不支持 | 结果是 `ESObject`。 |
| `throw` / `try` | 部分支持 | `--func`、同文件被调、直接导入，以及最多三跳的 throw 必须捕获。第四跳看不见。 |
| `@interface` 注解 | 不支持 | 不是值。 |
| ArkUI | 不支持 | 声明式 UI 不是这条 runner。 |

`@Sendable` 在 kit 概览里，不在这份教程里。Sendable 参数应 `retired`。不要把它算进这个计数。

## 什么时候算 HAP

目录里（本级或上级）有 `oh-package.json5` 和 `build-profile.json5` 就是
**HAP 根**。项目内的 `hvigorw` 包装不是必需的：工具链取最先解析到的
`hvigorw`（项目包装、`PATH`、或 DevEco 安装）。只在 `PATH` 上有 `hvigorw`
不会把当前目录变成 HAP 根。

OpenHarmony 组件上 `--scope` 以 `.ets` 结尾就是 ArkTS,即使没有 HAP。不要因为缺
hvigor 根就 retired。从该文件导入 `--func`,只复用用户 `--build-cmd` 重建(主机
`es2panda`/`ark`,或 `ohos_arkts_unittest`)。不要把 SUT 抄进另一个 HAP 的
`ohosTest/ets/sut/`。`--lang` 是文案语言(`zh` / `en`),不是 C++ 还是 ArkTS。

## 跑之前（前置条件）

在 `pi-pbt build-run` **之前**做完下面每一步。漏一步，门禁必须点名失败
（`BUILD FAILED — PBT not started`）。pi-pbt 不会发明 HAP、改写
`compileSdkVersion` / `runtimeOS`、或创建 `ohosTest`。

1. 当前用户**可写**的 DevEco SDK。`hvigor` 可能拒绝不可写的 `/opt/devecostudio/sdk`。
   把 `local.properties` 的 `sdk.dir`（以及 `DEVECO_SDK_HOME`）指到你钉住的 API
   真正所在的用户 SDK（例如 `26.0.0`）。
2. HAP 或 HAR 根：`oh-package.json5` + `build-profile.json5`。`--repo` 与 `--workdir`
   必须是**同一份 git** 检出。
3. 工程的 `compileSdkVersion` / `compatibleSdkVersion` / `runtimeOS` / hvigor
   `modelVersion` **对齐**该 SDK。自己改。装额外 SDK 或抬高工程版本号是操作者的事。
4. 要测的模块已有 `src/ohosTest/module.json5`。示例里的 `module=entry@ohosTest`
   假定这个目录存在。
5. 在 `--workdir` 里 `ohpm install` 退出 0。必须能解析 `@ohos/hypium`。仓库里 404 的钉
   （例如失效的 `@ohos/hamock` 版本）删掉或改钉。
6. `--build-cmd` 在 `--workdir` 里**已经退出 0**：
   - HAP（`hapTasks` / `type: hap`）：`hvigorw assembleHap --mode module -p module=<name>@ohosTest -p product=default --no-daemon`
   - HAR（`harTasks` / `type: har`）：`hvigorw genOnDeviceTestHap --mode module -p module=<name>@ohosTest -p product=default --no-daemon`
   纯 HAR 工程上没有 `assembleHap` 任务（hvigor 00306054）。
7. 再传 `--scope path/to/file.ets --func ABC`，且 `--func` 在 `--scope` **后面**。
   `--lang` 只有 `zh` / `en`（文案语言，不是 C++ 还是 ArkTS）。

写完套件后，设备 `aa test` 仍需要 `hdc` 和可签名的 HAP。GN `*_abc` 不是这条 runner。

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

HAR 模块（库，`harTasks`）——不要用 `assembleHap`：

```bash
pi-pbt build-run \
  --build-cmd "hvigorw genOnDeviceTestHap --mode module -p module=<har>@ohosTest -p product=default --no-daemon" \
  --repo /path/to/component \
  --workdir /path/to/component/.../the-har-project \
  --scope path/from/repo/to/file.ets \
  --func myFunc \
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

工具链、SDK 对齐、PandaCheck HAR、签名、`aa test` 见 `arkts-build-run` 技能卡。
pi-pbt **不会**静默改写 `compileSdkVersion`、`runtimeOS` 或 hvigor 钉。自己对齐，
或让战役大声失败。

## `passing` / `failing` 性质必须满足什么

套件是 `src/ohosTest/ets/test/*.test.ets`，并登记到 `List.test.ets`。
`report.json` 的 `sourceFile` / `functionName` 钉在硬性 `--scope` / `--func` 上。

| 允许 | 不是在测 `--scope` |
|---|---|
| 值导入能解析到检出里的 scoped 文件 | 对镜像全局做 `declare class` |
| 若本 HAP 无法按现状导入 `--scope`，则 `retired` | 仅 `import type`，或注释里提一下路径 |
| 签名含 `| null`、`| undefined` 或可选参数时：用 `oneof` + `constant(null)` / `constant(undefined)` 抽出该臂，并原样传入，不用 `as`、`!`、`??` | 丢掉该臂，或用 `as` / `ESObject` 收窄 |
| `--scope` 里声明的 `enum`：`constantFrom` 列出每个成员，回调类型为该 enum，不用 `as` | 用 `integer()` / `string()` 再 `as` 成 enum，或漏掉一个成员 |
| `string \| number` 以及其他 `string` / `number` / `boolean` 联合：`oneof` 抽出每一臂，不用 `as` | 只抽一臂，或 `as` 成其中一臂 |
| `--scope` 里声明的 `struct`：`record` 列出每个字段（或 `new StructName`），回调类型为该 struct，不用 `as` | 对普通对象 `as Rect` / `ESObject`，或 `record` 漏字段 |
| 函数型参数：传入具名 `.ets` 函数或箭头，不用 `as` | 丢掉回调，或 `as ESObject` / `as (x) =>` |
| `--scope` 里用 `;` 声明的重载：每个签名都要调用（每一原语臂或每一元数），不用 `as` | 只抽第一个重载 |
| HAR 原语与 `--scope` 里具名字段袋的联合：`oneof` 抽出每一臂，不用 `as` | 只抽一臂 |
| Sendable 参数（`@Sendable` 类、`SendableMap` / `ISendable`、`collections.Array`）：**retired** | `integer()`，或本线程上的 `new Box(n)` |
| 品牌 SDK 类型（`Length`、`Resource`、未在 `--scope` 声明的 `@kit.*` / `@ohos.*` 导入）：**retired** | `integer()`，或 `as Length` |
| 一维品牌 `T[]` / `Array<T>`（`ImageItem[]`）：该实参只传 `[]` 或 `constant([])`，不用 `as` | 构造 `ImageItem`、非空品牌数组，或仅因元素是品牌类型就对该 `--func` retired |
| 含不可抽臂的联合（`string | Date`）：**retired** | 只抽可抽的那一臂 |
| 臂不是 HAR 原语且元数相同的重载：**retired** | 一次 JS 调用，或对一臂 `as Length` |
| `--func` 调用 `getStringByNameSync`：`constantFrom` 该模块 `string.json` 里的每个 name，并断言 `n === n` | `string()`，或加前缀以致不可能存在的名字 |
| `--func` 调用同文件会 `throw` 的 helper：在性质里用 `arkThrew` / `try`/`catch` 接住 | 只走快乐路径，或让异常逃出 `check()` |
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

不要 `ohpm install fast-check`。ArkTS 框架是 PandaCheck。

## 当前进度

这是主机 `ark` 上的 PandaCheck。跟踪议题：
[#366](https://github.com/fermat-hkrc/FMBot-PBT/issues/366)。

### 已支持

- 识别 HAP 根并设 `PBT_HAP=1`。在 `src/ohosTest` 跑 Hypium + PandaCheck。
- 以操作者的 `hvigorw` 命令为门禁。签名、安装、`hdc shell aa test`。
- `--repo`（扫描 / `--scope`）与 `--workdir`（构建）可分离，同一 git 检出。
  自定义 `--out` 可放 `REPORT.md`。
- HAP **拥有** scoped `.ets` 且能导入时：执行该符号并记 `passing` / `failing`。
- 已落地的诚实门禁：禁止 `ohosTest/ets/sut/` 抄写；禁止用 `@kit.ArkUI` 冒充
  koala；`sourceFile` / `functionName` 钉在硬性 `--scope` / `--func`；
  `declare class` 不算导入；禁止改写 `--scope` 好让导入通过；目录
  `--scope` 按路径前缀且按该条性质的 `sourceFile` 匹配；空值签名必须保留该臂
  （[#523](https://github.com/fermat-hkrc/FMBot-PBT/issues/523)）；`--scope`
  里声明的 enum 必须抽出每个成员
  （[#525](https://github.com/fermat-hkrc/FMBot-PBT/issues/525)）。
- 若本 HAP 无法按现状导入 `--scope`，**retired**。真实 `hvigorw` 成功后全
  `retired` 的 `report.json` 可通过校验
  （[#533](https://github.com/fermat-hkrc/FMBot-PBT/issues/533)）。`--scope` 里声明的
  `struct` 必须用每个字段的 `record` 或 `new StructName` 构造
  （[#535](https://github.com/fermat-hkrc/FMBot-PBT/issues/535)）。函数型参数必须传入具名
  `.ets` 函数或箭头
  （[#537](https://github.com/fermat-hkrc/FMBot-PBT/issues/537)）。用 `;` 声明的重载必须全部调用
  （[#539](https://github.com/fermat-hkrc/FMBot-PBT/issues/539)）。这些臂的混合联合必须抽出每一臂
  （[#541](https://github.com/fermat-hkrc/FMBot-PBT/issues/541)）。Sendable 参数必须 retired——本线程上的 `new Box(n)` 不是 Sendable
  （[#543](https://github.com/fermat-hkrc/FMBot-PBT/issues/543)）。品牌 SDK 类型（`Length`、`Resource`）必须 retired——`as Length` 不是该类型
  （[#545](https://github.com/fermat-hkrc/FMBot-PBT/issues/545)）。品牌 `T[]`（`ImageItem[]`）用 `[]` 或 `constant([])`，不要构造元素
  （[#630](https://github.com/fermat-hkrc/FMBot-PBT/issues/630)）。含不可抽臂的联合必须 retired——只抽 `string()` 不是 `string | Date`
  （[#547](https://github.com/fermat-hkrc/FMBot-PBT/issues/547)）。臂不是 HAR 原语的重载必须 retired——一次 JS 调用不是该集合
  （[#549](https://github.com/fermat-hkrc/FMBot-PBT/issues/549)）。同文件被调函数里的 throw 必须在性质里捕获
  （[#551](https://github.com/fermat-hkrc/FMBot-PBT/issues/551)）。
- **PandaCheck 能驱动的 SUT 类型：** 已导出的 `number` / `string` / `boolean` /
  其数组，可写成 `.ets` 字面量的普通对象，以及经套件 `arkNullish` 的
  `T | null` / `T | undefined` / 可选参数。`passing` / `failing` 必须抽出该臂并无 `as`、`!`、`??` 传入。例如
  `export function add(a: number, b: number): number`。相对导入一跳的 enum 用该文件每个成员的 `constantFrom`。第二跳仍然看不见。字符串字面量联合（`'xs' | 'sm'`）、数字字面量联合（`0 | 1 | 2`），以及二者混合（`'xs' | 1`），用每个字面量的 `constantFrom` 或 `oneof(constant(...))`。`string()` / `integer()` 不是该联合。`[number, number]` 用 `property2` 构造 `[x, y]`。不要传入 `tuple()` 的 `Pair`。字面量与开放原语的联合（`'xs' | number`）用 `oneof(constant('xs'), integer())`。

### 不支持

两张表。第一张是 HAP runner。第二张是 **`fast-check.har` 喂不了的 ArkTS SUT**。
任一列要关掉，都需要新的 pi-pbt 工作。

#### Runner / 树

| 缺口 | 现在会怎样 | 要改进什么 |
|---|---|---|
| koala / `@koalaui` / `#generated` | 诚实收尾是全 `retired`；`report.json` 可通过校验。编不进 ohosTest。 | koala/Panda runner，或完整 OH 镜像路径，不是 HAP ohosTest。 |
| 未导出框架 `.ts`（如 `state_mgmt` `utils.ts`） | ohosTest 导不进该类。既有 UT 对镜像全局 `declare class`。诚实收尾是全 `retired`；`report.json` 可通过校验。 | 继续 retired（当前诚实结果），或在**不改文件**的前提下编进它。 |
| SDK / hvigor / `runtimeOS` 不匹配 | 门禁失败，或工具链被推迟。pi-pbt 不改清单。 | 大声报出期望的 SDK 字段。操作者预检，不要静默改靶。 |
| `test/component_test` 与许多 `examples/` HAP | 常是 API 12 / HarmonyOS 5.x；本机 SDK 26 树装不了。 | 同上：大声失败。不要改写 `compatibleSdkVersion`。 |
| OpenHarmony GN `build.sh` | 独立的 C++ 产品路径。`.ets` 的 `--scope` 不是那条路。 | 保持拆分。ArkTS `--scope` 不要起 GN。 |
| 原生行覆盖 | ohosTest 链不上 `coverage_gaps`。证据是 `aa test` 加 abc 字符串。 | 若工具链能产出 ArkTS 覆盖再做；不要伪造 C++ lcov。 |
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
| `@Sendable` / 并发类型 | `passing` / `failing` 被拒绝。诚实收尾是 `retired`。本线程上的 `new Box(n)` 不是 Sendable。 | 能构造 Sendable 值的 runner。不要硬撑 HAR。 |
| 未导出 `class` / `private` 字段 | 不改 `--scope` 就无法 `import` 或 `new`。 | 不改写 SUT 也能编/导，或 retired。 |
| 不支持类型的联合 | `passing` / `failing` 被拒绝。诚实收尾是 `retired`。只抽 `string()` 不是 `string \| Date`。 | 为每一臂写具名 `.ets` 适配器。不要硬撑 HAR。 |
| 品牌/不透明 SDK 类型 | 裸 `Length` / `Resource`：`passing` / `failing` 被拒绝。诚实收尾是 `retired`。`as Length` 不是该类型。一维品牌 `T[]` 可以是 `[]` / `constant([])`（[#630](https://github.com/fermat-hkrc/FMBot-PBT/issues/630)）。 | 非空品牌值的类型化构造。不要硬撑 HAR。 |
| 臂不是 HAR 原语的重载 | `passing` / `failing` 被拒绝。诚实收尾是 `retired`。一次 JS 调用不是该集合。 | 其余每个签名一条性质。不要硬撑 HAR。 |
| 被调函数里的 `throw` | 同文件被调函数、直接相对导入，以及最多三跳，必须捕获或对该 `--func` retired。第四跳仍然看不见。 | 在套件实际调用处捕获，或 retired。 |
| `getStringByNameSync` | 未抽出 `string.json` 里每个 name 时，`passing` / `failing` 被拒绝。`string()` 只打到 throw 路径。 | 对这些 name 用 `constantFrom`。断言有限。`NaN` 是失败结果，不是成功的 number。 |
| 约束到不支持类型的泛型 | `passing` / `failing` 被拒绝。诚实收尾是 `retired`。`name<string>` 不是 `T extends SomeStruct`。 | 对该 `--func` retired。 |
| 类型不是函数的回调（`Function`、`ESObject`） | `passing` / `failing` 被拒绝。诚实收尾是 `retired`。具名 stub 不是 `Function` 或 `ESObject`。 | 对该 `--func` retired。 |
| `@Component` / `@Observed` / UI | 需要 UI 运行时。 | 不是 HAP fast-check。Kea 或 UI runner。 |
| koala / `@koalaui` | 另一套编译器。HAR 永远装不上。 | Panda/koala runner，不是 ohosTest。 |
| Native / NAPI / 仅 `ESObject` 的 API | `ESObject` 能编过，但丢掉 ArkTS 类型。 | 类型化绑定，或 retired。 |

### 下一步改进

1. SDK 预检：点名缺失的 `compileSdkVersion` / `runtimeOS` / hvigor 插件并停下，
   不改 SUT 树。
2. 符号级导入：`passing` / `failing` 必须绑定 `--func`，不能只值导入碰巧定义
   它的文件。
3. Koala / Panda：文档化的非 HAP runner，或继续对 koala `--scope` retired。
4. **ArkTS SUT 类型：** 不要在 `.ets` 里硬撑 TS fast-check。nullish 用套件里的
   `arkNullish`，不 fork fast-check。`--scope` 里的 enum 用成员的 `constantFrom`。`--scope` 里的 struct 用字段的 `record` 或 `new StructName`。函数型参数用具名 `.ets` 函数或箭头。用 `;` 声明的重载每个签名都要调用。这些臂的混合联合用每一臂的 `oneof`。Sendable 参数应 retired（[#543](https://github.com/fermat-hkrc/FMBot-PBT/issues/543)）。品牌 SDK 类型（`Length`、`Resource`）应 retired（[#545](https://github.com/fermat-hkrc/FMBot-PBT/issues/545)）。品牌 `T[]`（`ImageItem[]`）用 `[]` 或 `constant([])`（[#630](https://github.com/fermat-hkrc/FMBot-PBT/issues/630)）。含不可抽臂的联合应 retired（[#547](https://github.com/fermat-hkrc/FMBot-PBT/issues/547)）。臂不是 HAR 原语的重载应 retired（[#549](https://github.com/fermat-hkrc/FMBot-PBT/issues/549)）。同文件被调函数里的 throw 应捕获（[#551](https://github.com/fermat-hkrc/FMBot-PBT/issues/551)）。约束到不支持类型的泛型应 retired。`name<string>` 不是 `T extends SomeStruct`。`Function` 或 `ESObject` 回调应 retired。具名 stub 只适用于 `(...) =>`。`passing` / `failing` 必须导入并调用 `--func`。只导入文件再调用兄弟符号不是该符号。字符串字面量联合、数字字面量联合，以及二者混合，用每个字面量的 `constantFrom` 或 `oneof(constant(...))`。`string()` / `integer()` 不是该联合。`[number, number]` 用 `property2` 构造 `[x, y]`，不要传入 `Pair`。字面量与开放原语的联合用 `oneof(constant(...), integer()/string())`。三跳之外的 throw 应捕获或 retired。第四跳仍然看不见。缺失的 `hvigorw` / SDK 已经推迟构建门。长期：同
   `check`/`property` 形态的 ArkTS PBT 库，无 `any`、无 `<const T>`。扫描 scoped
   签名，只允许匹配的 arbitrary。

## 相关

- [安装：HAP / ArkTS 应用](../installation.zh.md#harmonyos-hap--arkts-应用)
  （[English](../installation.md#harmonyos-hap--arkts-apps)）
- [英文 ArkTS 页](arkts.md)
- [子命令](../subcommands.zh.md) — `build-run --build-cmd "hvigorw assembleHap --mode module -p module=entry@ohosTest -p product=default --no-daemon"`
- [Kea GUI 测试](../kea.zh.md) — 真机黑盒 UI，不是源码级 ArkTS
- 内置 `arkts-build-run` skill：DevEco、SDK 对齐、PandaCheck HAR、签名、`aa test`
