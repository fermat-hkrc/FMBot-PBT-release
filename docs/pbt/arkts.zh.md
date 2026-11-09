# ArkTS PBT（PandaCheck）

[English version](arkts.md)

[返回安装说明](../subcommands.zh.md#build-run) ·
[子命令](../subcommands.zh.md)

fmbot-pbt 支持的 ArkTS 路径是 **HarmonyOS HAP ohosTest**。性质测试是 Hypium +
PandaCheck，落在 `src/ohosTest/ets/test/<Name>Pbt.test.ets`
（`import { check, property, integer, string, boolean, bigint, dictionary, primitiveUnion, date, dateUnion, array, recursive, functionModel, FunctionModel, unionCase, unionOf } from 'PandaCheck'`），
用 `hvigorw` 组装，以 `hdc shell aa test` 跑。判定结果看 `OHOS_REPORT_RESULT`。
用 `make-pandacheck-har.sh` 装到 `libs/PandaCheck.har`。模块 `oh-package.json5` 依赖写 `"PandaCheck": "file:../libs/PandaCheck.har"`。不要把 `Index.ets` 放进 `libs/`。不要 `ohpm install fast-check`。

这不是 Kea。这不是 OpenHarmony GN C++。主机 `es2panda`/`ark`（`PBT_PANDA=1`）
是**另一条** runner，不要和 ohosTest 混用。

`--isolate`（配合 `--scope file.ets --func Name`）生成**迷你 HAP**，把真实 `--scope` 文件及其相对 import 放到 `entry/src/main/ets/`。文件系统允许且放置文本与仓库一致时，这是硬链接（同一 inode）。未使用的 `import { Name } from './Name'` 会在普通文件副本上删掉，这样编译器不需要那个模块；函数体不变，也不改仓库。跨设备回退也是该路径上的普通文件，不是符号链接，也不是贴进 `ohosTest/ets/sut/`。用来测大应用里的函数（如 Photos），不必编整个 APK。import 图必须是公开 kit。系统 API 或整仓 HAR **STOP**。不改 HARD `--scope`。测试任务工作目录是 `<out>/isolate-hap`。从 `entry/src/main/ets` 的该文件导入（提示里写了 specifier）。不要换成符号链接。`entry/src/common` 或 `ohosTest` 里的拷贝不是 `--scope`。`--build-cmd` 换成该 HAP 里 `ohpm install` 之后的 `assembleHap … entry@ohosTest`。隔离 STOP 和 `ohpm install` 失败的退出码是 3。

## 特性清单

依据 OpenHarmony 教程
[Introduction to ArkTS](https://github.com/openharmony/docs/blob/f41b9345badd47c7ab0c263344cd7f4b5a549afb/en/application-dev/quick-start/introduction-to-arkts.md)
的提交 `f41b9345badd47c7ab0c263344cd7f4b5a549afb`。这是该教程点名的类型、调用形态和模块。不是 ArkTS 的百分比。

`let`、`const`、类型推断、`void`、运算符、`if` / `switch` / 循环和 `this` 是语法，不计入。性质不抽取它们。

这份 34 行清单里，**23 项已支持，5 项部分支持，6 项不支持**。已支持表示 `passing` / `failing` 能使用该形态，且收尾门禁会要求它。部分支持表示该教程标题下只有一部分形态能抽取或必须捕获。不支持表示抽不了，或诚实收尾是 `retired`。

| 教程特性 | fmbot-pbt | 含义 |
| --- | --- | --- |
| `number` | 已支持 | `integer()` / `double()`。与小数阈值比较用 `double()`；`integer()` 跨不过那个阈值。 |
| `bigint` | 已支持 | `bigint()`。`integer()` 不是 `bigint`。 |
| `boolean` | 已支持 | `boolean()`。 |
| `string` | 已支持 | `string()`。 |
| `Object` | 不支持 | 裸 `Object`、它的别名以及直接写出的 Object/nullish 类型仍 retired。只有下方有界的 nullable-Object 别名配方接受真实构造器代表，不代表通用 SDK/对象构造。 |
| `number` / `string` / `boolean` 的数组 | 已支持 | `array(integer())`、`array(string())` 或 `array(boolean())`。只抽元素会被拒绝。原语字段袋的非空数组，包括 SDK `FingerInfo[]`，用 `array(recordOf(fn, show, shrinkName), { minLength: 1 })`。品牌 `ImageItem[]` / `Length[]` 仍只能传 `[]`。 |
| `enum` | 已支持 | 同文件，或相对导入一跳，用每个成员的 `constantFrom`。第二跳看不见。 |
| 联合 | 已支持 | 固定教程的 class/原语/nullish 联合用带类型的 `unionOf` / `unionCase`，保留真实构造实例、`instanceof`、方法、分臂收缩与全臂门禁。既有原语/字面量/interface 字段袋联合仅在没有 `unionOf` 的测试文件中保留；也支持可解析的 Date 及标量/原语数组组合。这是联合组合能力，不是凭空构造不透明 SDK/native 叶子；不可用的底层类型仍 retired。 |
| 类型别名 | 已支持 | 教程的 `Matrix`、`Handler`、`Predicate<T>` 和 `NullableObject`，以及按语法解析的链、作用域、导入/重导出、泛型代入/默认值。嵌套数组和有界生产性递归数组要求完整带类型配方。别名保留 SDK/native/Sendable/构造器限制，不创建新的值构造器。 |
| 可选参数 | 已支持 | 用 `unionOf` / `unionCase` 包含 `undefined`；既有 `oneof` 加 `constant(undefined)` 仅适用于没有 `unionOf` 的测试文件。 |
| 默认参数 | 已支持 | `number`、`string` 或 `boolean` 的默认参数必须既省略又传入抽取值。 |
| rest 参数 | 已支持 | `func(...xs)`，`xs` 是 `number`、`string` 或 `boolean` 的 `array()`。裸元素列表会被拒绝。 |
| 函数类型 | 已支持 | 教程中的具体函数值，包括 `Math.sin`、真实具名/箭头/导入/static 函数、返回类型推断/上下文类型及词法/返回闭包。门禁检查完整签名与真实调用绑定；`functionModel` 通过类型化闭包提供固定种子生成与收缩。无具体签名的 `Function` / `ESObject` 互操作是另一限制。 |
| 重载 | 已支持 | 教程的自由函数、public 实例/static 方法和构造器（含无分号声明）。每个完整有序签名及元数都要有 scoped、未修改且带类型的调用。支持的非原语沿用自身配方；只有底层域不可用才 retired。 |
| 带字段的 class 或 `struct` | 已支持 | 在 `--scope` 里声明时，用 `new` 或每个字段的 `record`。 |
| `static` 字段或方法 | 已支持 | 参数为 `number`、`string` 或 `boolean` 的 public static 方法必须写成 `Class.method(drawn)`。这些类型的 public static 字段，包括未标注的 `= 0`、`= 'albumType'`、`= true`，要在同一 property 回调里 `Class.field = drawn` 再读取 `Class.field`。readonly 字段只读取。private、protected、非精确类型和 `Date` 字段不在此门禁内。 |
| getter / setter | 已支持 | 教程的 `Person4.age`、只有 getter/只有 setter 的形态，以及 Date 和已支持底层域。校验真实 public 实例/static 归属、新 receiver、带类型的构造/赋值抽取值、同一实例的可观察读取、有界返回类型推断及逐操作异常捕获。private 状态、替身和不可用底层域不算覆盖。 |
| 继承、`super`、抽象类 | 不支持 | 不构造。 |
| `private` / `protected` 字段 | 不支持 | 套件不得改 `--scope` 来暴露它。 |
| 对象字面量 | 部分支持 | `record()` 是 `{ w, h }`、`{ x, y }` 或 `{ left, top, right, bottom }`。其它 `number`/`string`/`boolean` 字段袋用 `recordOf`。可选字段用 `optional(integer())`、`optional(string())` 或 `optional(boolean())`。扁平 interface 的封闭字符串字面量联合字段（直接写，可换行，或可解析的本地/导入别名链）用列出全部字面量的 `constantFrom<Alias>`，可选字段外包 `optional()`。`[number, number]` 字段用 `numberPair`。可选数对用 `optional(numberPair(x, y))`，不是始终有值的字面量。同文件里另一个 interface 字段袋用 `record().generate` 或 `recordOf().generate`，一层，结果上需要 `as Name`。`number[]`、`string[]`、`boolean[]` 字段用 `array().generate`。可选数组字段用 `optional(array(integer())).generate`。`[]` 不是该字段。`number[][]`、class、ArkUI `struct`、`@Sendable` class、`Date` 和其它元组仍不支持。 |
| `Record<K, V>` | 部分支持 | 教程形态 `Record<string, number>` 用 `dictionary(string(), integer())`。额外可抽取、不改变本行：`Record<string, number \| string \| boolean>` 用 `dictionary(string(), primitiveUnion())`。值参数必须就是 `primitiveUnion()`。`oneof(primitiveUnion())` 不收缩。默认 4 个键。`{ maxLength }` 设定该上限，包括调低。收缩先丢掉键，再收缩值，再收缩键（重名则跳过）。回调参数类型是该 Record。`{}` 和 `record()` 不是该映射。`Record<number, number>`、其它值类型、那些类型的同文件别名、`Record \| null`、可选 Record 和带默认值的 Record 仍然不是。 |
| interface | 部分支持 | `--scope` 里的字段袋可以构造。只有方法的契约不是值。 |
| 泛型函数 | 已支持 | 调用必须是 `name<string>`、`name<number>` 或 `name<boolean>`。推断调用被拒绝。 |
| 泛型约束 | 部分支持 | 约束到这三种能抽取。其他约束 `retired`。 |
| 泛型类 | 已支持 | `new CustomStack<string>()`、`<number>` 或 `<boolean>`，再在该实例上以该类型的抽取值调用方法。推断的 `new CustomStack()` 会被拒绝。这三种以外的约束应 retired。 |
| 泛型默认值 | 部分支持 | 元数据解析别名及默认值依赖，包括 class 默认值。仍要求既有的显式原语泛型实参及底层域门禁；Date/class 默认值不会自动提供生成器。 |
| nullish，`T` 或 `null` / `undefined` | 已支持 | 完整 `unionOf` / `unionCase`；既有 `arkNullish` 仅用于没有 `unionOf` 的测试文件。调用不得用 `!` 或 `??` 丢掉该臂。 |
| `--func` 的具名导入 | 已支持 | `passing` / `failing` 必须导入该文件并调用该符号。 |
| 导入别名或命名空间 | 已支持 | `id as ident` 和 `ids.id(...)` 算。调用兄弟符号不算。 |
| 默认导出 | 已支持 | 默认导出的函数必须按默认导入并调用。class 或值的默认导出不是这个绑定。 |
| 动态 `import()` | 不支持 | 结果是 `ESObject`。 |
| `throw` / `try` | 已支持 | `--func`、同文件被调、直接导入，以及最多四跳的 throw 必须捕获。第五跳看不见。 |
| `@interface` 注解 | 不支持 | 不是值。 |
| ArkUI | 不支持 | 声明式 UI 不是这条 runner。 |

`@Sendable` 在 kit 概览里，不在这份教程里。Sendable 参数应 `retired`。不要把它算进这个计数。

### 覆盖固定教程的类型别名

别名给域命名，不产生构造器值。固定教程的四种声明是：

```typescript
type Matrix = number[][];
type Handler = (s: string, no: number) => string;
type Predicate<T> = (x: T) => boolean;
type NullableObject = Object | null;
```

`Matrix` 用 `array(array(integer()))`，不能只抽一行或固定矩阵；回调保留完整类型，
在实际 SUT 参数位置传入各自独立、未修改的抽取值。`Handler` / `Predicate<number>`
要求每次 SUT 调用处的真实具名/箭头/导入/内建函数值具有匹配的参数个数、解析后的参数类型及
返回类型。函数变量的完整类型标注可提供上下文类型，不能补上缺失参数或覆盖冲突的
显式类型。零参数 Handler 和接收字符串的 `Predicate<number>` 会被拒绝；SUT 明确
指定的类型实参优先于默认值。带默认值、可选或组合回调域仍需自己的底层协议。

只有 SUT 参数本身引用精确解析为 `Object | null` 或 `Object | null | undefined`
的别名，才能用完整 `unionOf` 覆盖 nullish 臂及每次新建的可访问真实构造器代表。
裸 `Object`、它的别名、直接写出的 Object/nullish 类型以及 Object 数组/其它联合
仍 retired，不能用普通对象伪造 SUT 的 nominal 类。

解析覆盖词法作用域/命名空间、泛型代入和依赖默认值、相对导入/重导出，以及本地
OHPM `file:` 包的源码入口；限同一源码工作区内 40 个源码文件。遗漏/私有导出、歧义
和纯别名环失败关闭。构造器/模块身份保留，导入类不能变成本地同名类或内建类型。

`type Nested = number | Nested[]` 这类生产性递归数组使用 HAP
`recursive(base, expand, { maxDepth: 3 })`。base 覆盖终止情况，`expand(child)` 保留
base 臂，并在递归数组位置使用实际 child arbitrary；用 `{ maxLength: 3 }` 限制分支。
运行时深度为整数 0–8，默认 3；覆盖要求省略选项或内联深度 1–8。纯
`type A = B; type B = A` 不属于生产性递归。生成由 seed 决定，显示与反例最小化由组件
负责；贪心收缩不承诺任意谓词下的全局最小树。

规范化仅用于检查元数据，不改真实生产源码、导入、SHA 或编译的 SUT。Date 字段/
数组/默认参数、不可访问构造器、继承、Sendable 和不可用 SDK 叶子不能借别名绕过。
class 默认值也不能替代明确指定类型实参的实例化协议。本次只升级别名行，不升级全部底层部分支持项。

上游和内置版本分别独立构建 HAR/HAP，在 API 26 各通过六条一致性测试，覆盖教程
四形态、导入/默认值、递归重放、新实例和嵌套阈值收缩。这些是库模型，不是生产 SUT。
另由 fmbot-pbt 在未修改的 Settings `TimeUtil.obtainFormatTime` 上完成真实测试任务：
退出 0，五条 failing 性质被接纳；设备九条测试、八失败、零错误、一通过。候选发现
不等于确认 bug。该既有直接时间戳别名证明生产交付链路，不声称所有新导入/泛型/
递归形态都已在应用中执行。

### 覆盖固定教程的函数类型

教程以 `type trigFunc = (x: number) => number` 调用 `doAction(Math.sin)`。
这是实际内建函数值，不是假造的本地 stub。具体函数标注和别名支持真实具名/箭头
函数、相对具名/默认/命名空间导入、public static 函数值，以及未被遮蔽或修改的
数值内建函数，例如 `Math.sin`。支持简单表达式/代码块的返回类型推断、已检查的
上下文类型和词法捕获；真实工厂返回的闭包也能作为回调传入。

每条性质中的每次直接 SUT 调用都保留真实回调的完整参数个数、解析后的参数类型、
返回类型及 nominal/模块身份。上下文不能补上缺失参数或覆盖冲突的显式类型；具体
函数类型内部的可选/rest 形参保留完整标注。遗漏/错误回调、仅类型导入、替换绑定、
字符串里的调用和未调用闭包都不算。public 实例方法在从 scoped 模块导入的真实
可访问构造器实例上调用，不能换成同名假接收者，也不能改 private 字段或构造器。

用 HAP `functionModel(outputs)` 抽取函数行为，再由真实类型化闭包解释。例如真实
scoped `apply(fn, value)` 的签名为数值函数时：

```ets
import { check, property, integer, functionModel, FunctionModel } from 'PandaCheck';
import { apply } from '../../../main/ets/ActualModule';

const result = check(property(functionModel(integer()), integer(),
  (model: FunctionModel<number>, value: number): boolean => {
    const fn = (x: number): number => model.call(x.toString());
    return apply(fn, value) === model.call(value.toString());
  }), { seed: 42, numRuns: 100 });
```

模型是生成的**测试输入**，不是 SUT 的拷贝或替代模型。稳定键必须包含全部实际形参，
不能用调用计数、随机值，或对有输入的函数使用固定键。结果 arbitrary 必须匹配实际
`FunctionModel<R>` 类型及回调结果；`void` 回调可将模型结果用于捕获的可观察副作用，
再通过真实 public 生命周期检查。抽取却没使用的模型或类型不匹配的配方不能收尾。
调用结果必须进入回调返回值（可经未修改的局部别名或受支持的值表达式），或进入由
性质谓词观察的性质局部捕获值/静态命名字段。`model.call(key); return 1;`、未使用/
覆盖的结果、`void` 或逗号表达式丢弃的调用、未观察的捕获值都不算覆盖。捕获存储不能
随后被直接覆盖；不推断不透明 helper 间接效果或任意控制流。

每次抽取的新有限结果表和 salt 只使用传入 RNG。运行时 `maxOutputs` 是 1..16 的
整数，默认 4；有输入的回调覆盖要求省略选项或内联上限 2..16。上限 1 是合法的
常量函数运行时配方。收缩尝试常量投影、删除一个桶，或用分量的 `shrink` 每次改
一个结果；`show` 为含分量显示的 `fn{seed:...,values:[...]}`。不修改原表，结果对象
仍保持其引用语义，不承诺任意深拷贝。贪心收缩不保证全局最小反例。

scoped 工厂返回闭包时，每条性质必须调用其真实结果；仅创建闭包不算覆盖。返回
函数的参数使用各自独立、未修改且带类型的抽取值。生产契约保留状态时检查重复调用。
简单推断和真实函数变量的上下文类型受支持；复杂不透明推断需要实际类型化包装，
不能编造声明。

上游和内置版本分别独立构建真实 API-26 HAR、应用/测试 HAP，各通过六条一致性测试：
内建/导入/static 函数值、推断箭头、新闭包状态、生成行为、固定种子重放和精确阈值
收缩。这些是库/语言模型，不是生产 SUT。另用重建的 fmbot-pbt 对未修改的 Photos
`CustomTransitionManager.registerNavParam` 运行真实测试任务，并核实其 `SlideShowPage`
调用者：退出 0，四条 passing 性质被接纳、零 bug；设备五条测试、零失败/错误、五通过。
生成的函数模型通过真实存储的回调及 public 生命周期驱动捕获效果；未改 SUT、SDK 或
产品清单。这证明生产交付链路，不声称全部内建/导入/返回形态均已在应用中执行。

函数能力不升级其它底层域；加上下方重载及 accessor 能力后，当前清单是
**23 已支持 / 5 部分支持 / 6 不支持**。无具体签名的 `Function`、
`ESObject`、动态导入、未知 native 生命周期、Sendable、不可访问/继承构造器仍有各自
限制。可选/默认/组合回调**参数**、任意泛型/异步组合和不透明递归函数生成仍需自身
底层协议。组合回调域无论直接标注还是经别名引用都同样 retired；一个函数臂不等于
函数类型的直接联合。门禁是有界语法/绑定分析，不是一般全程序类型或控制流推断。

### 覆盖固定教程的 getter / setter

支持教程的 `Person4.age`（含负数年龄抛错）及明确允许的仅 getter/仅 setter。
用 `--func age` 选择真实属性；owner 多义时用 `Person4.age` 或 `Ns.Person4.age`。
操作是 `person.age`、`person.age = value`，不是编造的 `age(...)` 方法。

每条实际执行的 PandaCheck 性质在回调内新建可访问、真实导入的实例，或使用实际
static owner。构造参数和 setter 值都是未修改且带类型的抽取值；具名/default/
namespace 导入别名保留真实 class 身份。成对 setter 写入后，必须在同一 receiver
上读取并观察 getter。仅 getter 要观察读取值；仅 setter 要经同一实例的 public
方法/字段或实际生成回调的捕获效果观察副作用。每个可能抛错的访问都要在性质内
捕获；getter 返回具体函数时，要以匹配的带类型抽取值调用它。

Date accessor 使用 `date()`。支持明确类型及从真实 backing field/初始化值或简单
返回表达式进行的有界推断。数组/矩阵、字面量、enum、interface 字段袋、字典、
别名、真实 class 值、完整联合和回调保留各自生成/收缩协议。泛型 receiver 实参
必须明确给出或来自声明默认值，不做猜测。例如现有 `Stamp.when` 契约确实保留
赋入时间戳时：

```ets
check(property(date(), (value: Date): boolean => {
  const stamp = new Stamp();
  stamp.when = value;
  return stamp.when.getTime() === value.getTime();
}));
```

其它/更早的读取、未调用闭包或对象方法、未执行性质、丢弃结果、固定/覆盖/cast
输入、替身/共享实例、private 状态修改及构造器/prototype 替换均不能提供该证据。
无法解析的推断、不透明工厂/作为回调参数抽取的 receiver、不可访问/继承/Sendable
owner 及不可用底层类型仍 retired。不因此新增 Date 字段袋生成、Date 数组/默认
实参、普通 static Date 字段或不支持的回调组合。先证明原始绑定，再生成仅供主机
checker 使用的调用视图；视图不编译、不改写 SUT，原始 import/转录/SHA 门禁不变。
这是有界识别，不是全程序控制流或纯度证明。

上游与内置版本分别独立构建 API-26 HAR 和签名应用/测试 HAP，设备各通过
**9 条 Hypium 测试、0 失败/错误**：教程异常、Date/推断及单侧 accessor、static/
泛型实例、组合值、真实 class、生成回调、新实例/重放，以及收缩到 `10` 和
`Date(1000)`。这些是语言/库一致性模型；运行时 API 与主机实现均未改动。

另用重建 CLI 对未改动的 Photos `GridScrollMsg.isScrolling` 运行新任务，核实
`AbstractDayGridVc` 的真实写入与 `ImageGridItemComponent` 的读取：退出 0，四条
passing 性质被接纳、零 bug；真实设备 **5 条测试、0 失败/错误、5 通过**（四条
PBT 加 Ability）。生产源码/调用者 SHA 及原始/isolate 源码 inode 不变；正式 IR/
收尾流程生成套件，监督会话未手改；未改 SUT、SDK 或产品清单。应用证据针对
boolean 存取，不代表 Date 应用场景或异步定时器语义。编译后 JS 只读重验接纳该
性质清单，并拒绝十八种内存中的 accessor 变异及错误源码 SHA；重验不是新设备执行。
本次只升级 getter/setter 一行：**23/5/6**，仍是 34 行。

### 覆盖固定教程的重载

支持教程的自由函数 `foo1(number/string)`、实例 `C.foo` 与
`C(number/string)` 构造器，包括没有分号的构造器声明。static 方法、导入别名、
命名空间和真实默认函数导出保留实际 owner；方法名有多个 owner 时选择
`Class.method`，不能借用其它类或命名空间的签名。

每个声明签名需要在实际 `check` / `checkWith` 的 property 中完整匹配参数顺序与
元数。不同签名优先使用独立带类型性质；`(number, string)` 不能覆盖
`(string, number)`，宽联合参数也不自动满足分开的重载头。实现签名不是额外公开
重载。字符串里的调用、未调用 helper、仅创建而未执行的 property、cast、修改、
遮蔽、固定实参和额外无效调用都不算覆盖。

不再因非原语而统一 retired。数组、矩阵、Date、字面量、enum、已支持的 interface
字段袋/字典/联合、class 别名与具体回调沿用各自协议。class 用真实 public 构造器的
新 `recordOf(fn, show, shrink)` 实例，不用形似对象。生成回调必须消费
`functionModel` 的结果；返回闭包必须真实调用。泛型实参仍须明确给出；可选/rest/默认参数
及 Date/native/Sendable/访问限制仍独立生效。这里完整支持的是固定教程的重载形态，
不代表自动构造任意参数打包或不支持的类型组合；HAP `property` 仍接收一到三
个 arbitrary 分量。

按原始 AST 容器及声明偏移选择签名。给旧门禁的视图只是 checker 元数据字符串，
不会改写或替换真实编译源码；原始 import、转录、SHA 门禁仍有效。解析后相同的参数
列表、多义 owner、ambient 声明、不可访问实现及超过 64 个声明都 fail closed。
这是有界语法/绑定识别，不是全程序控制流推断。

上游和内置版本分别独立构建 API-26 HAR 与签名应用/测试 HAP，设备各通过
**8 条 Hypium 测试、0 失败/错误**：教程形态、参数顺序/元数、真实 class/Date/数组/
enum/字段袋/回调、新实例、固定种子重放及收缩到 `Item:10`。这些是语言/库一致性
模型，不是生产应用测试任务。

另用重建后的 fmbot-pbt 在普通 HAP 模式执行 `build-run`，测试未修改的 PandaCheck
`constantFrom<T>(values)` / `(first, ...rest)`：退出 0，四条 passing 性质被接纳，
零 bug；设备五条测试全部通过（四条生成的 PBT 加 Ability），没有失败或错误。
编译的 scoped 文件是原始库源码的硬链接，inode 和 SHA 均已核实；没有改写 SUT、SDK
或产品清单。生成使用正式性质 IR 与收尾流程，不由监督会话手改套件。任务可读取
之前的测试产物，但证据来自本次重新构建、安装与真实设备执行。这证明真实库交付，
不是应用级证明；未找到适合的应用重载目标。编译后 JS 的只读重验拒绝了十二种
签名/抽取值/import 变异以及错误 SHA。

### 覆盖固定教程的联合组合

教程的 `Cat | Dog | Frog | number | string | null | undefined`、纯 class 联合、
`instanceof` 缩窄、公共方法及分支专属方法，使用
`unionOf<T>(unionCase<T, Arm>(label, arbitrary, owns), ...)`。每个臂都要有一个
带类型的 case。class 臂用 `recordOf` 真实公开构造器配方，每次创建从 scoped 模块
导入的真实实例，不能用形似的对象字面量替代。生成和收缩后仍保留方法及原型。

例如 `string | Date`：

```typescript
type Value = string | Date;
const values = unionOf<Value>(
  unionCase<Value, string>('string', string(), (v: Value): boolean => typeof v === 'string'),
  unionCase<Value, Date>('Date', date(), (v: Value): boolean => v instanceof Date));
```

回调保留完整联合，并在 SUT 对应参数位置原样传入各自独立的抽取值。每个联合参数，
包括可选/nullish 参数，都需完整配方；允许复用不可变 arbitrary。遗漏臂、错误谓词、
替换工厂、只抽标量、覆盖输入、未调用闭包和对象伪造 class 均不能收尾。
规范化可解析的联合别名及其导入；既有原语/字面量/interface 字段袋形态仅用于没有 `unionOf`
的测试文件，class 实例联合始终要求真实构造器组合配方。

**协议选择：** scoped 签名含联合参数时，测试文件内任意 `unionOf` 调用都会为每条
性质的每个联合参数选择组合协议，包括可选/nullish 参数。不得混用 `dateUnion()`、
`oneof`、`nullish` 或 `arkNullish` 联合实参配方；普通字段袋内部的可选字段不是该规则。
签名没有联合参数时，不相关的组合器不会强加联合协议。组合成功也不能绕过签名中
Date 字段、数组或带默认值 Date 参数的 retired 要求。

运行时使用传入 RNG、检查唯一归属、仅在当前臂内收缩，显示为 `label:componentShow`。
归属谓词必须纯且互斥；擦除后重叠的形态需互斥表示。门禁只是有界词法/结构识别，
不是一般控制流/数据流证明，不覆盖 helper 间接或任意深层修改。继承、不可访问构造器、Sendable/
native 生命周期、不透明递归/native 域及其它不支持的叶子类型，仍有各自限制。
本次“已支持”指**固定教程的联合形态**，不是其它部分支持/不支持行的所有笛卡尔组合。

`PandaCheckUnion.test.ets` 是独立库一致性设备夹具，覆盖七臂、真实方法/原型、
新实例、seed 重放和 class 阈值收缩；不把它称作生产 SUT 测试任务。
生产关联另由 `fmbot-pbt build-run` 在未改动的应用源码上验证。

### 既有有界 Date 参数和时间戳联合

单独的 `Date` 参数用 `date()`；没有 `unionOf` 的测试文件中，精确的
可解析的 `string | number | Date`（顺序不限，可经别名链/导入）用 `dateUnion()`。
若同签名的可选/timezone 参数使用 `unionOf`，时间戳也必须使用完整组合配方。
仅适用于 HAP，不扩展 host Panda runner。
在 `property` 对应位置直接抽取，或绑定一个 `const` arbitrary。PandaCheck 的具名导入
别名按本地绑定解析；其它作用域里的局部 `date` 值不会遮蔽这里的工厂。
回调参数保留完整类型，并在直接 SUT 调用的对应参数位置原样传入。多个 Date 参数
需要不同的回调抽取值。转换、强转、覆盖参数、未使用的抽取、自定义联合生成器和
不收缩的 `oneof` 包装都不计入覆盖。

`date()` 默认生成 1900-01-01 UTC 至 2100-01-01 UTC 的新有效 Date，闭区间整数毫秒
为 `-2208988800000` 至 `4102444800000`。`date({ min, max })` 的边界必须是有序有限
整数，并位于 TimeClip（`±8640000000000000` 毫秒）内。不生成 Invalid Date。
生成包含边界及截断到区间内的零点，向该零点收缩，反例显示为 `Date(ms)`。
`dateUnion()` 抽取普通 `string()`、`integer()` 或默认 `date()`；仅按当前值类型
收缩，不跨臂，也不会对 Date 调用字符串收缩器。两个、三个参数的性质使用每个
arbitrary 的 `show`，Date 反例仍为 `Date(ms)`，不依赖本地时区。

收尾门禁要求 `date()` 的自定义边界为内联数字字面量且满足 `min < max`；不透明选项
或单值区间不计入 Date 域覆盖。运行时 API 仍允许单值区间作其它用途。解构/参数
遮蔽的工厂、下标 setter 和未调用的嵌套 SUT 闭包不能构成覆盖。

既有 helper 本身不扩展其它 Date 形态；上方组合配方支持 `string | Date` 和可解析的
nullish Date 联合。Date 字段袋、Date 数组/默认实参、普通 static Date 字段和
不透明构造仍保留限制。Date getter/setter 使用上方 accessor 协议；真实 SUT 的
backing field 不等于任意字段袋构造。导入/链式 Date 别名与泛型 Date 默认值是可见
元数据，适用对应 Date 覆盖或不支持底层域门禁。这些残余限制不升级其它行；
accessor 完成后的当前清单是 **23/5/6**。

### interface 字段袋中的字符串字面量字段

`--scope` 中的扁平 interface 可包含必选或可选的封闭字符串字面量联合字段，
可以直接写联合（允许换行），也可以引用可解析的本地/导入别名链。例如
`type Status = 'pending' | 'running'` 和 `status?: Status` 使用
`const statusArb = optional(constantFrom<Status>('pending', 'running'));`。
必选字段不加 `optional`。必须列出每个字面量，不能添加额外值。

在 `recordOf((rng: Random, size: number): Input => ({ ... }), showInput, shrinkInput)`
返回的 interface 字面量中直接使用 `statusArb.generate(rng, size)`。
收缩调用同一个 arbitrary 的 `shrink(value.status)`，每个候选只替换一个字段，
保留其它字段，并 push 到具名收缩函数最终返回的数组。丢弃的样本或 push 到另一个数组不算。
字段袋 arbitrary 绑定为 `const` 并传给 `property`，再将有类型的回调参数原样传给 SUT。
同类型的每个 SUT 参数必须对应不同的抽取回调参数；可以重复使用同一个 arbitrary 来分别抽取。`oneof(..., constant(undefined))` 不适用于该字段：它不收缩。
强制转换、转换 helper、开放的 `string()`、遗漏字面量、固定字段、未传入的 generator
和覆盖输入字段都会被拒绝。

此扩展不包括 class、嵌套或数组形式的字面量字段袋、递归字面量字段形态、
含转义字符的字面量语法或含不可抽臂的联合。导入/链式字面量别名的规范化不解除
这些字段限制。accessor 完成后，清单为 **23 已支持 / 5 部分支持 / 6 不支持**。

## 什么时候算 HAP

目录里（本级或上级）有 `oh-package.json5` 和 `build-profile.json5` 就是
**HAP 根**。项目内的 `hvigorw` 包装不是必需的：工具链取最先解析到的
`hvigorw`（项目包装、`PATH`、或 DevEco 安装）。只在 `PATH` 上有 `hvigorw`
不会把当前目录变成 HAP 根。

OpenHarmony 组件上 `--scope` 以 `.ets` 结尾就是 ArkTS，即使没有 HAP。不要因为缺
hvigor 根就 retired。从该文件导入 `--func`，只复用用户 `--build-cmd` 重建(主机
`es2panda`/`ark`，或 `ohos_arkts_unittest`)。不要把 SUT 抄进另一个 HAP 的
`ohosTest/ets/sut/`。`--lang` 是文案语言(`zh` / `en`)，不是 C++ 还是 ArkTS。

## 跑之前（前置条件）

在 `fmbot-pbt build-run` **之前**做完下面每一步。漏一步，门禁必须点名失败
（`BUILD FAILED — PBT not started`）。fmbot-pbt 不会发明 HAP、改写
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
shell profile：残留的 `1` 会把之后的 C++ 测试任务也逼进 HAP 模式。树是
ArkTS 但缺清单、又要用自由入口 `fmbot-pbt` / `-p` 时，只在**这一条命令**上设
`PBT_HAP=1`。

## 怎么跑

`--repo` 是被测模块（扫描树、`--scope`、默认 `pbt-out/`）。`--workdir` 是
`--build-cmd` 的执行目录。两者必须在**同一份 git 检出**里。`--workdir` 常常是
HAP 工程；`--repo` 可以是同时装着 `.ets` / `.ts` SUT 和该 HAP 的上级组件。

两个路径都是 HAP 本身时：

```bash
fmbot-pbt build-run \
  --build-cmd "hvigorw assembleHap --mode module -p module=entry@ohosTest -p product=default --no-daemon" \
  --repo /path/to/hap \
  --workdir /path/to/hap \
  --scope entry/src/main/ets/common/Utils.ets \
  --func deepCopyInter \
  --lang zh
```

HAR 模块（库，`harTasks`）——不要用 `assembleHap`：

```bash
fmbot-pbt build-run \
  --build-cmd "hvigorw genOnDeviceTestHap --mode module -p module=<har>@ohosTest -p product=default --no-daemon" \
  --repo /path/to/component \
  --workdir /path/to/component/.../the-har-project \
  --scope path/from/repo/to/file.ets \
  --func myFunc \
  --lang zh
```

SUT 文件在 HAP 目录外、但仍在同一检出内时：

```bash
fmbot-pbt build-run \
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

`--build-cmd` 是操作者输入。失败则测试任务不启动（`BUILD FAILED — PBT not started`）。
`true` 这类空操作不能证明 `passing` / `failing`。HAP 测试任务不要加 `--prompt`，
除非有 SOP 不知道的领域事实。

工具链、SDK 对齐、PandaCheck HAR、签名、`aa test` 见 ArkTS 测试指南。
fmbot-pbt **不会**静默改写 `compileSdkVersion`、`runtimeOS` 或 hvigor 钉。自己对齐，
或让测试任务大声失败。

## `passing` / `failing` 性质必须满足什么

套件是 `src/ohosTest/ets/test/*.test.ets`，并登记到 `List.test.ets`。
`report.json` 的 `sourceFile` / `functionName` 钉在硬性 `--scope` / `--func` 上。

| 允许 | 不是在测 `--scope` |
|---|---|
| 值导入能解析到检出里的 scoped 文件 | 对镜像全局做 `declare class` |
| 若本 HAP 无法按现状导入 `--scope`，则 `retired` | 仅 `import type`，或注释里提一下路径 |
| 签名含 `| null`、`| undefined` 或可选参数时：用完整 `unionOf` / `unionCase`；既有 `oneof` + `constant(null)` / `constant(undefined)` 仅用于没有 `unionOf` 的测试文件。原样传入，不用 `as`、`!`、`??` | 丢掉该臂，或用 `as` / `ESObject` 收窄 |
| `--scope` 里声明的 `enum`：`constantFrom` 列出每个成员，回调类型为该 enum，不用 `as` | 用 `integer()` / `string()` 再 `as` 成 enum，或漏掉一个成员 |
| `string \| number` 等原语联合：完整带类型的 `unionOf` / `unionCase`；每臂 `oneof` 仅用于没有 `unionOf` 的测试文件，不用 `as` | 只抽一臂，或 `as` 成其中一臂 |
| `--scope` 里声明的 `struct`：`record({ w, h })`、`record({ x, y })`、LTRB `record`，或其它 `number`/`string`/`boolean` 字段袋用 `recordOf(fn, show, shrinkName)`，可选字段用 `optional()`（或 `new StructName`）。扁平 interface 的封闭字符串字面量联合字段（直接写或可解析的别名链/导入）用列出全部字面量的 `constantFrom`，可选字段外包 `optional()`；收缩候选只改一个字段并 push 到返回数组。嵌套的 interface 字段用 `record().generate` 或 `recordOf().generate`，只在该结果上 `as Name`。`number[]` / `string[]` / `boolean[]` 字段用 `array().generate`。回调类型为该 struct，此外不用 `as` | 对普通对象 `as Rect` / `ESObject`，漏字段，内层袋里的 `optional()` 冒充可选嵌套字段，把 class / ArkUI `struct` / `@Sendable` 字段当成嵌套袋，用 `[]` 冒充 `number[]` 字段，或对 `[number, number]` 以外的元组 / `number[][]` / 不支持的字段类型用 `recordOf`（上述封闭字符串字面量字段已支持），或可选字段没用 `optional()`，或 `recordOf` 只有一个参数 |
| 具体函数参数：签名匹配的真实具名/箭头/导入/内建函数值，或解释已抽取 `FunctionModel` 的类型化闭包；工厂返回函数须调用真实结果 | 缺失/错误/未用回调、假接收者、仅类型导入、覆盖绑定、强转或未使用模型 |
| `--scope` 重载：用独立、未修改且带类型的抽取值调用每个完整有序签名和元数，不用 `as` | 只抽第一个签名，或只收集无序原语臂 |
| HAR 原语与 interface 字段袋联合：每臂 `oneof` 仅用于没有 `unionOf` 的测试文件；class 实例臂要求真实构造器 case，不用 `as` | 只抽一臂 |
| Sendable 参数（`@Sendable` 类、`SendableMap` / `ISendable`、`collections.Array`）：**retired** | `integer()`，或本线程上的 `new Box(n)` |
| 品牌 SDK 类型（`Length`、`Resource`、未在 `--scope` 声明的 `@kit.*` / `@ohos.*` 导入）：**retired** | `integer()`，或 `as Length` |
| 一维品牌 `T[]` / `Array<T>`（`ImageItem[]`）：该实参只传 `[]` 或 `constant([])`，不用 `as` | 构造 `ImageItem`、非空品牌数组，或仅因元素是品牌类型就对该 `--func` retired |
| 原语字段袋的 `Name[]`，包括 SDK `FingerInfo[]`：`array(recordOf(fn, show, shrinkName), { minLength: 1 })`；`{ x, y }` / `{ w, h }` / LTRB 也可用 `record()`。回调类型为 `Name[]`，把该值传给 `--func`，不用 `as` | `[]`、画出了却没传入、没有 `minLength: 1` 的 `array()`，或 `{ id: 0 } as FingerInfo` |
| 独立 `Date` 用 `date()`；时间戳联合用完整组合配方，或仅在没有 `unionOf` 的测试文件中用 `dateUnion()`。原样传入对应实参；Date 字段/数组/默认参数仍 retired | 缺臂、工厂遮蔽、单值/不透明 Date 边界、下标修改或未调用的闭包 |
| class/Date/nullish 联合用完整带类型的 `unionOf` / `unionCase`、真实公开构造器和原样抽取值；确实无法构造底层叶子时 retired | 遗漏臂、对象伪造 class、假配方或不透明 SDK 替身 |
| 含不可用 SDK/native/Sendable/不可访问底层域的重载：**retired** | 编造替身或对一臂 `as Length` |
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
| `test/component_test` 与许多 `examples/` HAP | 常钉在本机没有的 SDK。门禁失败。不要在 fmbot-pbt 里更换测试目标。 |
| OpenHarmony GN `build.sh` / `ohos_unittest` | C++ 产品路径。`.ets` 的 `--scope` 不是那条路。 |

## 证据

`hdc shell aa test` 与完整的 `OHOS_REPORT_RESULT` 行。对 `modules.abc` 做
`strings` 只说明 SUT 被链进，不说明性质跑过。ohosTest 没有原生行覆盖。

不要 `ohpm install fast-check`。ArkTS 框架是 PandaCheck。

## 当前进度

这是主机 `ark` 上的 PandaCheck。跟踪议题：
[#366](https://github.com/fermat-hkrc/FMBot-PBT/issues/366)。

### 已支持

下列使用既有 `oneof` / `arkNullish` 的联合实参配方仅用于没有 `unionOf` 的测试文件；
否则每个联合参数使用上方完整组合协议。不相关的非联合签名不受影响。

- 识别 HAP 根并设 `PBT_HAP=1`。在 `src/ohosTest` 跑 Hypium + PandaCheck。
- 以操作者的 `hvigorw` 命令为门禁。签名、安装、`hdc shell aa test`。
- public getter/setter（含 Date 与有界推断）在新建真实 receiver 上使用实际属性、
  带类型抽取值与可观察结果，见上方 accessor 协议；不代表 private 字段或 SDK 对象合成。
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
  `struct` 必须用 `record({ w, h })`、`record({ x, y })`、LTRB `record`、其它 `number`/`string`/`boolean` 字段袋的 `recordOf(fn, show, shrinkName)`，可选字段用 `optional()`，或 `new StructName` 构造。嵌套的 interface 字段用 `record().generate` 或 `recordOf().generate`，只在该结果上 `as Name`。class、ArkUI `struct`、`@Sendable` 字段不是这个画法。原语字段袋的 `Name[]`，包括 `FingerInfo[]`，用 `array(recordOf(fn, show, shrinkName), { minLength: 1 })` 传给 `--func`。带嵌套字段的袋不是这个数组。`ImageItem[]` 仍是 `[]`
  （[#535](https://github.com/fermat-hkrc/FMBot-PBT/issues/535)）。函数型参数必须传入签名匹配的真实函数值，见上方完整函数协议
  （[#537](https://github.com/fermat-hkrc/FMBot-PBT/issues/537)）。全部重载（含方法及构造器）必须按完整参数顺序带类型调用
  （[#539](https://github.com/fermat-hkrc/FMBot-PBT/issues/539)）。这些臂的混合联合必须抽出每一臂
  （[#541](https://github.com/fermat-hkrc/FMBot-PBT/issues/541)）。Sendable 参数必须 retired——本线程上的 `new Box(n)` 不是 Sendable
  （[#543](https://github.com/fermat-hkrc/FMBot-PBT/issues/543)）。品牌 SDK 类型（`Length`、`Resource`）必须 retired——`as Length` 不是该类型
  （[#545](https://github.com/fermat-hkrc/FMBot-PBT/issues/545)）。品牌 `T[]`（`ImageItem[]`）用 `[]` 或 `constant([])`，不要构造元素
  （[#630](https://github.com/fermat-hkrc/FMBot-PBT/issues/630)）。含不可抽臂的联合必须 retired——只抽 `string()` 不是含必需 class/native 臂的联合
  （[#547](https://github.com/fermat-hkrc/FMBot-PBT/issues/547)）。含不可用底层域的重载必须 retired——已支持的非原语改用自身配方
  （[#549](https://github.com/fermat-hkrc/FMBot-PBT/issues/549)）。同文件被调函数里的 throw 必须在性质里捕获
  （[#551](https://github.com/fermat-hkrc/FMBot-PBT/issues/551)）。
- **PandaCheck 能驱动的 SUT 类型：** 已导出的 `number` / `string` / `boolean` /
  其数组，可写成 `.ets` 字面量的普通对象，以及经套件 `arkNullish` 的
  `T | null` / `T | undefined` / 可选参数。`passing` / `failing` 必须抽出该臂并无 `as`、`!`、`??` 传入。例如
  `export function add(a: number, b: number): number`。相对导入一跳的 enum 用该文件每个成员的 `constantFrom`。第二跳仍然看不见。字符串字面量联合（`'xs' | 'sm'`）、数字字面量联合（`0 | 1 | 2`），以及二者混合（`'xs' | 1`），用每个字面量的 `constantFrom` 或 `oneof(constant(...))`。`string()` / `integer()` 不是该联合。`[number, number]` 一个参数用 `property2` 构造 `[x, y]`。每个更多的数对用不同的 `numberPair`，传给同一个 `property()`。不要传入 `tuple()` 的 `Pair`。字面量与开放原语的联合（`'xs' | number`）用 `oneof(constant('xs'), integer())`。

### 不支持

两张表。第一张是 HAP runner。第二张是 **`fast-check.har` 喂不了的 ArkTS SUT**。
任一列要关掉，都需要新的 fmbot-pbt 工作。

#### Runner / 树

| 缺口 | 现在会怎样 | 要改进什么 |
|---|---|---|
| koala / `@koalaui` / `#generated` | 诚实收尾是全 `retired`；`report.json` 可通过校验。编不进 ohosTest。 | koala/Panda runner，或完整 OH 镜像路径，不是 HAP ohosTest。 |
| 未导出框架 `.ts`（如 `state_mgmt` `utils.ts`） | ohosTest 导不进该类。既有 UT 对镜像全局 `declare class`。诚实收尾是全 `retired`；`report.json` 可通过校验。 | 继续 retired（当前诚实结果），或在**不改文件**的前提下把它编译进来。 |
| SDK / hvigor / `runtimeOS` 不匹配 | 门禁失败，或工具链被推迟。fmbot-pbt 不改清单。 | 大声报出期望的 SDK 字段。操作者预检，不要静默改靶。 |
| `test/component_test` 与许多 `examples/` HAP | 常是 API 12 / HarmonyOS 5.x；本机 SDK 26 树装不了。 | 同上：大声失败。不要改写 `compatibleSdkVersion`。 |
| OpenHarmony GN `build.sh` | 独立的 C++ 产品路径。`.ets` 的 `--scope` 不是那条路。 | 保持拆分。ArkTS `--scope` 不要起 GN。 |
| 原生行覆盖 | ohosTest 链不上 `coverage_gaps`。证据是 `aa test` 加 abc 字符串。 | 若工具链能产出 ArkTS 覆盖再做；不要伪造 C++ lcov。 |
| 兄弟导出 / 私有字段绕行 | 套件可导入 `--scope` 里已导出的 helper，再在活实例上调未导出方法。这不是改写，导入检查会放行。 | 决定 `passing` 是否要求导入**符号**，而不只是定义它的文件。 |
| 主机 `es2panda` / PandaCheck | 已移除。不要当 HAP 替身加回来。 | 仅当真实 Panda 测试树就是 SUT，并自有 runner。 |
| Kea GUI | `fmbot-pbt kea` 是另一模式（黑盒 UI）。 | 与源码级 ArkTS 分开。 |

#### ArkTS SUT 与 `fast-check.har`

HAR 是转换后的 JS（对 npm fast-check 做 `ohpm convert`）。它生成 JS 的
`number` / `string` / `boolean` / `Array` / 普通对象。性质文件是 ArkTS
（命名导入、`(x: number)`、无 `any`、无解构、`check()` 而非 `fc.assert`）。
当 ohosTest 无法用这些值调用该函数时，该 SUT 特性就不支持。

| SUT 特性 | 现在会怎样 | 要改进什么 |
|---|---|---|
| `@Sendable` / 并发类型 | `passing` / `failing` 被拒绝。诚实收尾是 `retired`。本线程上的 `new Box(n)` 不是 Sendable。 | 能构造 Sendable 值的 runner。不要硬撑 HAR。 |
| 未导出 `class` / `private` 字段 | 不改 `--scope` 就无法 `import` 或 `new`。 | 不改写 SUT 也能编/导，或 retired。 |
| 不支持类型的联合 | `passing` / `failing` 被拒绝。诚实收尾是 `retired`。只抽 `string()` 不是含必需 class/native 臂的联合。 | 每个可构造臂用带类型的 `unionCase`；真正不可用的 SDK/native 叶子仍需自己的构造/生命周期支持。 |
| 品牌/不透明 SDK 类型 | 裸 `Length` / `Resource`：`passing` / `failing` 被拒绝。诚实收尾是 `retired`。`as Length` 不是该类型。一维品牌 `T[]` 可以是 `[]` / `constant([])`（[#630](https://github.com/fermat-hkrc/FMBot-PBT/issues/630)）。 | 非空品牌值的类型化构造。不要硬撑 HAR。 |
| 含不可用底层域的重载 | 必需 native/SDK/Sendable/不可访问值仍须 retired；支持的 class/数组/Date/回调不因非原语而 retired。 | 增加底层构造/生命周期协议，不用 cast 或漏签名。 |
| 被调函数里的 `throw` | 同文件被调函数、直接相对导入，以及最多四跳，必须捕获或对该 `--func` retired。第五跳仍然看不见。 | 在套件实际调用处捕获，或 retired。 |
| `getStringByNameSync` | 未抽出 `string.json` 里每个 name 时，`passing` / `failing` 被拒绝。`string()` 只打到 throw 路径。 | 对这些 name 用 `constantFrom`。断言有限。`NaN` 是失败结果，不是成功的 number。 |
| 约束到不支持类型的泛型 | `passing` / `failing` 被拒绝。诚实收尾是 `retired`。`name<string>` 不是 `T extends SomeStruct`。 | 对该 `--func` retired。 |
| 无具体签名的函数/互操作域（`Function`、`ESObject`） | `passing` / `failing` 被拒绝。诚实收尾是 `retired`。具名 stub 不是 `Function` 或 `ESObject`。 | 对该 `--func` retired。 |
| `@Component` / `@Observed` / UI | 需要 UI 运行时。 | 不是 HAP fast-check。Kea 或 UI runner。 |
| koala / `@koalaui` | 另一套编译器。HAR 永远装不上。 | Panda/koala runner，不是 ohosTest。 |
| Native / NAPI / 仅 `ESObject` 的 API | `ESObject` 能编过，但丢掉 ArkTS 类型。 | 类型化绑定，或 retired。 |

### 下一步改进

1. SDK 预检：点名缺失的 `compileSdkVersion` / `runtimeOS` / hvigor 插件并停下，
   不改 SUT 树。
2. 符号级导入：`passing` / `failing` 必须绑定 `--func`，不能只值导入碰巧定义
   它的文件。
3. Koala / Panda：文档化的非 HAP runner，或继续对 koala `--scope` retired。
4. **ArkTS SUT 类型：** 本项既有联合实参配方仅用于没有 `unionOf` 的测试文件；
   否则每个联合参数使用完整带类型的 case。不要在 `.ets` 里硬撑 TS fast-check。nullish 用套件里的
   `arkNullish`，不 fork fast-check。`--scope` 里的 enum 用成员的 `constantFrom`。`--scope` 里的 struct 用 `record({ w, h })`、`record({ x, y })`、LTRB `record`、其它 `number`/`string`/`boolean` 字段袋的 `recordOf(fn, show, shrinkName)`，或 `new StructName`。可选原语字段用 `optional()`。这种字段袋的 `Name[]`，包括 `FingerInfo[]`，用 `array(recordOf(fn, show, shrinkName), { minLength: 1 })` 传给 `--func`。`ImageItem[]` 仍是 `[]`。`[number, number]` 字段用 `numberPair`。可选数对用 `optional(numberPair(x, y))`。嵌套的 interface 字段用 `record().generate` 或 `recordOf().generate`，只在该结果上 `as Name`。class、ArkUI `struct`、`@Sendable` 字段不是这个画法。带嵌套字段的袋不是 `Name[]`。`number[]` / `string[]` / `boolean[]` 字段用 `array().generate`。`number[][]` 不是。扁平 interface 的封闭字符串字面量联合字段（直接写或可解析的别名链/导入）用列出全部字面量的 `constantFrom<Alias>`，可选字段外包 `optional()`；收缩候选每次只改一个字段并 push 到返回数组。其它元组和不支持的字段类型不是 `recordOf`。具体函数参数使用上方完整协议，包括真实导入/内建函数值、生成回调行为和返回闭包调用。重载（含方法、构造器和无分号形式）的每个签名都要按完整参数顺序带类型调用。这些臂的混合联合用每一臂的 `oneof`。Sendable 参数应 retired（[#543](https://github.com/fermat-hkrc/FMBot-PBT/issues/543)）。品牌 SDK 类型（`Length`、`Resource`）应 retired（[#545](https://github.com/fermat-hkrc/FMBot-PBT/issues/545)）。品牌 `T[]`（`ImageItem[]`）用 `[]` 或 `constant([])`（[#630](https://github.com/fermat-hkrc/FMBot-PBT/issues/630)）。含不可抽臂的联合应 retired（[#547](https://github.com/fermat-hkrc/FMBot-PBT/issues/547)）。含不可用底层域的重载仍应 retired（[#549](https://github.com/fermat-hkrc/FMBot-PBT/issues/549)）；支持的非原语不再统一 retired。同文件被调函数里的 throw 应捕获（[#551](https://github.com/fermat-hkrc/FMBot-PBT/issues/551)）。约束到不支持类型的泛型应 retired。`name<string>` 不是 `T extends SomeStruct`。泛型类方法是 `new CustomStack<string>()` 再在该实例上 `stack.push(drawn)`。`Function` 或 `ESObject` 回调应 retired。具名 stub 只适用于 `(...) =>`。`passing` / `failing` 必须导入并调用 `--func`。只导入文件再调用兄弟符号不是该符号。字符串字面量联合、数字字面量联合，以及二者混合，用每个字面量的 `constantFrom` 或 `oneof(constant(...))`。`string()` / `integer()` 不是该联合。`[number, number]` 一个参数用 `property2` 构造 `[x, y]`，更多的数对用不同的 `numberPair`。不要传入 `Pair`。字面量与开放原语的联合用 `oneof(constant(...), integer()/string())`。四跳之外的 throw 应捕获或 retired。第五跳仍然看不见。缺失的 `hvigorw` / SDK 已经推迟构建门。长期：同
   `check`/`property` 形态的 ArkTS PBT 库，无 `any`、无 `<const T>`。扫描 scoped
   签名，只允许匹配的 arbitrary。

## 相关

- [安装：HAP / ArkTS 应用](../subcommands.zh.md#build-run)
  （[English](../subcommands.md#build-run)）
- [英文 ArkTS 页](arkts.md)
- [子命令](../subcommands.zh.md) — `build-run --build-cmd "hvigorw assembleHap --mode module -p module=entry@ohosTest -p product=default --no-daemon"`
- [Kea GUI 测试](../kea.zh.md) — 真机黑盒 UI，不是源码级 ArkTS
- 内置 ArkTS 测试指南：DevEco、SDK 对齐、PandaCheck HAR、签名、`aa test`
