# ArkTS PBT（PandaCheck）

[English version](arkts.md)

[返回安装说明](../subcommands.zh.md#build-run) ·
[子命令](../subcommands.zh.md)

fmbot-pbt 支持的 ArkTS 路径是 **HarmonyOS HAP ohosTest**。性质测试是 Hypium +
PandaCheck，落在 `src/ohosTest/ets/test/<Name>Pbt.test.ets`
（`import { check, property, integer, double, ieeeDouble, string, boolean, bigint, dictionary, numericDictionary, primitiveUnion, date, dateUnion, array, recursive, functionModel, FunctionModel, unionCase, unionOf } from 'PandaCheck'`），
用 `hvigorw` 组装，以 `hdc shell aa test` 跑。判定结果看 `OHOS_REPORT_RESULT`。
用 `make-pandacheck-har.sh` 装到 `libs/PandaCheck.har`。模块 `oh-package.json5` 依赖写 `"PandaCheck": "file:../libs/PandaCheck.har"`。不要把 `Index.ets` 放进 `libs/`。不要 `ohpm install fast-check`。

这不是 Kea。这不是 OpenHarmony GN C++。主机 `es2panda`/`ark`（`PBT_PANDA=1`）
是**另一条** runner，不要和 ohosTest 混用。

`--isolate`（配合 `--scope file.ets --func Name`）生成**迷你 HAP**，把真实 `--scope` 文件及其相对 import 复制到 main 与 ohosTest 共用的 `entry/src/common/`。副本是独立普通文件，不是硬链接或符号链接。只有生成器可移除未使用的本地具名导入；函数体及仓库保持不变。构建前记录每份副本和原始依赖的哈希基线；任一侧被修改、删除或重定向时，拒绝 passing/failing 收尾。用来测大应用里的函数（如 Photos），无须编整个应用。import 图必须是公开 kit；系统 API 或整仓 HAR **STOP**。测试任务工作目录是 `<out>/isolate-hap`。从提示指定的 `entry/src/common` 副本导入，例如 `../../../common/default/model/common/DurationFormatter`；不得修改、替换该图，也不得另抄到 main/ets 或 ohosTest，原始依赖图同样只读。`--build-cmd` 换成该 HAP 里 `ohpm install` 之后的 `assembleHap … entry@ohosTest`。隔离 STOP 和 `ohpm install` 失败的退出码是 3。

## 特性清单

依据 OpenHarmony 教程
[Introduction to ArkTS](https://github.com/openharmony/docs/blob/f41b9345badd47c7ab0c263344cd7f4b5a549afb/en/application-dev/quick-start/introduction-to-arkts.md)
的提交 `f41b9345badd47c7ab0c263344cd7f4b5a549afb`。这是该教程点名的类型、调用形态和模块。不是 ArkTS 的百分比。

`let`、`const`、类型推断、`void`、运算符、`if` / `switch` / 循环和 `this` 是语法，不计入。性质不抽取它们。

这份 34 行清单里，**28 项已支持，0 项部分支持，6 项不支持**。已支持表示 `passing` / `failing` 能使用该形态，且收尾门禁会要求它。部分支持表示该教程标题下只有一部分形态能抽取或必须捕获。不支持表示抽不了，或诚实收尾是 `retired`。

| 教程特性 | fmbot-pbt | 含义 |
| --- | --- | --- |
| `number` | 已支持 | `integer()` / 有界有限值 `double()`，或显式 `ieeeDouble()`，同时采样普通值和代表性 IEEE 边界。小数阈值用 `double()` 或显式 IEEE 模式；`integer()` 跨不过阈值。有限/特殊值域和显示规则见下方。 |
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
| 带字段的 class 或 `struct` | 已支持 | 每个实例字段须来自构造参数抽取值或完整的 public 字段赋值，含带方法的类；构造配方也须完整显示/收缩。public 数据类可用上下文字面量/类型化 `recordOf`；`record` 只支持数字 XY/WH/LTRB。构造默认值不算开放字段抽取。 |
| `static` 字段或方法 | 已支持 | 参数为 `number`、`string` 或 `boolean` 的 public static 方法必须写成 `Class.method(drawn)`。这些类型的 public static 字段，包括未标注的 `= 0`、`= 'albumType'`、`= true`，要在同一 property 回调里 `Class.field = drawn` 再读取 `Class.field`。readonly 字段只读取。private、protected、非精确类型和 `Date` 字段不在此门禁内。 |
| getter / setter | 已支持 | 教程的 `Person4.age`、只有 getter/只有 setter 的形态，以及 Date 和已支持底层域。校验真实 public 实例/static 归属、新 receiver、带类型的构造/赋值抽取值、同一实例的可观察读取、有界返回类型推断及逐操作异常捕获。private 状态、替身和不可用底层域不算覆盖。 |
| 继承、`super`、抽象类 | 不支持 | 不构造。 |
| `private` / `protected` 字段 | 不支持 | 套件不得改 `--scope` 来暴露它。 |
| 对象字面量 | 已支持 | 教程的上下文初始化、赋值、实参、带类型返回、数组元素与嵌套字段，使用源码解析出的真实 interface/public 数据类。直接/简写字面量保留带类型抽取值；组合 `recordOf(generate, show, shrink)` 支持可用的嵌套字段、Date/bigint/矩阵、字面量/enum、固定元组字段、可选值及真实构造器叶子。检查全部字段的生成/显示及逐组件收缩；不替代构造器、不合成 private/ArkUI/Sendable/native 值、不改写源码。 |
| `Record<K, V>` | 已支持 | 字符串键用 `dictionary(string(), values)`，数字键用 `numericDictionary(integer(), values)`。完整类型化值包括教程 `PersonInfo` 字段袋、原语、Date/bigint、数组及嵌套映射。别名和受支持包装保留原样类型化 SUT 实参、完整组件显示/收缩、键上限与冲突安全。精确边界及真实 HAR/HAP/设备和 `build-run` 交付证据见下方。 |
| interface | 已支持 | 实际本地/导入方法、访问器属性及接口扩展使用真实可访问的源码实现类，保持完整状态配方；纯继承字段袋保留字面量配方。精确边界及真实 HAR/HAP、未改动可调用源码的交付证据见下方。 |
| 泛型函数 | 已支持 | 类型实参须明确匹配，或由声明默认值完整确定。原语实例化和声明约束保留底层配方；拒绝从值推断缺失类型。 |
| 泛型约束 | 已支持 | 解析实际导入/代入后的约束，包括教程的 `Hashable.hash()`；使用新建的源码实现类，保持完整状态、显示及收缩。每个直接 scoped 调用均保留匹配的显式/默认类型、原样抽取值及可观察分派。不可用构造仍 retired；边界与真实 HAR/HAP/设备及交付证据见下方。 |
| 泛型类 | 已支持 | 明确的 `new CustomStack<string>()` 和真实约束实例（如 `new MyHashMap<HashedKey, string>()`），在每条性质内新建 receiver，保留带类型构造/方法抽取值及可观察效果。可由完整声明默认值补齐省略的类型；拒绝从值推断 receiver 和不可用约束。 |
| 泛型默认值 | 已支持 | 实际省略的函数/类/接口类型实参须由源码默认值完整确定，保留词法/导入身份、依赖代入、显式覆盖及约束。每个有值域仍需生成器/真实构造、显示/收缩和逐调用观察门禁。下方记录未改源码的 HAR/HAP/设备及 build-run 证据；不扩展任意推断或自动 Date/class 构造。 |
| nullish，`T` 或 `null` / `undefined` | 已支持 | 完整 `unionOf` / `unionCase`；既有 `arkNullish` 仅用于没有 `unionOf` 的测试文件。调用不得用 `!` 或 `??` 丢掉该臂。 |
| `--func` 的具名导入 | 已支持 | `passing` / `failing` 必须导入该文件并调用该符号。 |
| 导入别名或命名空间 | 已支持 | `id as ident` 和 `ids.id(...)` 算。调用兄弟符号不算。 |
| 默认导出 | 已支持 | 默认导出的函数必须按默认导入并调用。class 或值的默认导出不是这个绑定。 |
| 动态 `import()` | 不支持 | 结果是 `ESObject`。 |
| `throw` / `try` | 已支持 | `--func`、同文件被调、直接导入，以及最多四跳的 throw 必须捕获。第五跳看不见。 |
| `@interface` 注解 | 不支持 | 不是值。 |
| ArkUI | 不支持 | 声明式 UI 不是这条 runner。 |

`@Sendable` 在 kit 概览里，不在这份教程里。Sendable 参数应 `retired`。不要把它算进这个计数。

### 显式 IEEE-754 数字域

`double()` 保留原有有界有限值生成及收缩。显式 `ieeeDouble()` 同时提供普通小数/
阈值、代表性边界、宽范围有限值和非规格化数；默认包括 NaN 和正负无穷。
`includeNaN`、`includeInfinity` 各自默认 **true**。仅有限值使用
`ieeeDouble({ includeNaN: false, includeInfinity: false })`，仍包括正负零、
非规格化数和大幅值。不得暗中把既有有限域性质扩展到特殊值。

活动门禁要求省略选项，或使用内联布尔字面量对象；拒绝未知/重复/计算键、spread、
强转和不透明选项。使用未被覆盖的 PandaCheck 具名导入和稳定 arbitrary 绑定。
数字 Record **键**要求两个标志都为 false；值可用完整模式。完整域、实际调用、
构造器归属、独立原样类型化抽取值、观察及不支持兄弟域的门禁不变。
封闭 `constantFrom<number>` 边界集仅是补充证据，不替代开放数字配方。

显示须委托同一组件的 `show`，包括嵌套对象/数组/映射/optional 配方：JSON 会丢失
NaN、无穷和负零，直接字符串转换会丢失 `-0`。内置包装、数字字段袋和元组委托
组件显示。收缩仍须从相同组件取得候选，每次仅改一个字段，重建真实新实例。
PandaCheck 保留 `recordOf(generate, show, shrink?)`；**活动**配方仍要求完整有效
收缩。谓词抛出的任何值均算失败，即使格式化为空或会抛错；生成/收缩/显示的框架
错误仍向外传播。活动自身的异常预言检查并未取消。

真实 scoped `echo` 若保留输入身份，保存结果后内联比较：
`actual === value ? (actual !== 0 || 1 / actual === 1 / value) :
(actual !== actual && value !== value)`。这是 ArkTS 可用的 SameValue；
`Object.is` 受限，仅 `===` 不够。Oracle 必须来自 SUT 约定，舍入、溢出和 NaN
会破坏许多通常的代数恒等式。

采样**不均匀也不穷尽**，不覆盖所有 NaN payload；有限试验不能证明每个区域均已
抽到。收缩严格降低有限值幅值的位序，候选有界且无 SameValue 重复，`-0 → 0`；
runner 仍是贪心且有预算，不保证任意谓词的全局最小值或任意有限预算内收敛。

内置数字卡为 `references/numbers.md`。`hap/UPSTREAM.json` 记录 PandaCheck
规范版本 `f46b330bbae6a81e409f0bdf442c6ffffaff6e3d` 及未修改运行库/IEEE 控制文件
的 SHA-256，离线校验不依赖兄弟仓库。Node 控制、真实 HAR/HAP/Hypium、新的
`fmbot-pbt build-run` 活动、报告/独立活动 review、PR/CI 是不同门禁。
pin 显式声明 `adapterDifferences: []`：无运行库补丁，更严格的活动策略留在代理中。
`npm run check:pandacheck` 在两种 check 门禁中核验实际源码字节；构建另查 `dist`
打包树，binary smoke 核验新 binary 实际解包的运行库和 pin，而不只信清单。
在线来源验证用 `node scripts/check-pandacheck.mts --upstream`，通过已认证 `gh`
获取精确提交的允许列表文件并计算哈希。离线 CI 无需兄弟检出或跨仓 token；在线
读取权限缺失会报错，不算核验成功。

下方 #710 仍是历史证据，并未使用 `ieeeDouble`。清单仍为 **28/0/6**：
本次扩展数字采样，不增加语言特性行。

2026-11-11，新重建的 fmbot-pbt binary 对未修改的
`hap_defaults_types.ets#read` 执行 `build-run`，自行生成
`DefaultBoxReadIeeePbt.test.ets`：**8 条 passing 性质、0 bug**，种子
**420101–420108**，每条 **4096** 次（共 32768 次）。省略默认值的真实
`SomeType` 配方和显式 `number` receiver 各覆盖四种有限/特殊值模式，保留
身份/SameValue、重复读取和显式匹配默认值观察；监督会话未手改生成套件。

实际 API-26 HAR、应用/测试 HAP 重建、签名安装和完整 Hypium：**28 通过、
0 失败/错误**。其中 8 条生成性质、8 条未改 IEEE 控制、4 条 legacy double
控制、7 条默认值控制和 1 条 smoke。基线 20/20；首次生成运行每条 1000 次、
28/28；最终强化每条 4096 次、28/28。IEEE 控制保留种子 1、42、0xC0FFEE，
检查收缩进展/终止、普通/宽范围/非规格化数采样、选项分离及 NaN/无穷/`-0`
忠实显示重放。故意失败是框架控制，不是 SUT bug。

源码 SHA-256 保持
`a4e315f1a2df6ae0d5a75f122023b46e60cf041960873de9d2587322d246e0ca`；三份给定套件
逐字节不变，已安装运行库与上游固定哈希一致。新的独立 review 检查全部 8 条性质，
**0 finding**；报告及 required-review 重验通过。这是预先准备的 `--no-isolate`
可调用夹具，不是新的隔离 E2E、生产应用/native 生命周期或实测行/分支覆盖证明。

### 覆盖固定教程的泛型默认值

这里是**类型实参默认值**，不是函数值参数的默认值。只有声明默认值能完整确定所有
缺失类型时才接受省略。显式实参优先：`first<T = number, U = T[]>` 省略时选择
`number, number[]`，`first<string>` 选择 `string, string[]`，
`first<string, boolean[]>` 保留两个显式覆盖。不从值推断必需类型。
按声明词法上下文解析导入、别名和命名空间；先完成 class 默认值，再完成 method
默认值，保留同名遮蔽。自引用/前向引用不能借用全局同名类型；实际约束始终生效。

未改动的 `echo<T = number>(value: T): T` 若原样返回输入，可写：

```ets
check(property(integer(), (value: number): boolean => echo(value) === value));
check(property(string(), (value: string): boolean => echo<string>(value) === value));
```

默认值不是生成器：`T = Date` 需要 `date()`。现有 `SomeType` 若有真实 number
构造器及精确公开 `value` reader，使用完整配方：

```ets
const numbers = integer();
const items = recordOf((rng: Random, size: number): SomeType =>
  new SomeType(numbers.generate(rng, size)),
  (value: SomeType): string => numbers.show(value.value),
  (value: SomeType): SomeType[] => numbers.shrink(value.value)
    .map((n: number): SomeType => new SomeType(n)));
check(property(items, (value: SomeType): boolean => {
  const box = new DefaultBox(value);
  return box.read() === value;
}));
```

须导入实际源码类；示例依赖真实 `DefaultBox<T = SomeType>` 的构造/read 约定。
`new DefaultBox<string>(text)` 使用显式覆盖。`Reader<T = SomeType>` 在实际接口
形参/标注中选择该默认值，不构造接口或伪造实现。冲突标注不能改变真实构造类型。
无有值形参的 `defaultKind<T = number>()` 也须观察实际调用。
所有有值位置保留完整固定种子生成、组件显示、有效收缩、独立原样类型化抽取值及
最终有效调用观察；void 调用保留下方受影响状态/逐调用历史门禁。每条性质内新建
receiver。强转、固定替身、缺失配方、修改、逃逸及不支持兄弟参数均拒绝。
类继承、不可用 native/SDK/Sendable 构造、单独指定的泛型构造器和不支持的行为
组合仍有独立限制。

使用内置 `references/defaults.md` 与既有原样 IR 绑定（`generators: { value: items }`）。
测试保留省略/显式语法；仅校验用的实例化视图不替代编译 SUT。IR 成功展开后，仍须
通过真实构建、设备执行、报告校验和独立 review。

2026-11-10，重建的 fmbot-pbt（pi 1.5.2）驱动 API-26 PandaCheck HAR、应用与
测试 HAP 构建，签名安装后完整 `aa test`：**14 通过、0 失败/错误**。
七条未修改的 `PandaCheckDefaults.test.ets` 覆盖函数省略/显式类型、依赖默认值/覆盖、
真实 class/interface 分派、receiver 独立性、新对象固定种子生成/显示及有效收缩。
故意失败控制以种子 **710**、200 次运行最小化并重放 **`2`**，不是 SUT bug。

同一 `build-run` 对未修改的 `hap_defaults_types.ets#read` 接纳 **6 条生成的 passing
性质、0 bug**，种子 **71001–71006**、每条 1000 次：省略/匹配 class 实参、
string/number/boolean 覆盖及独立读取交错。加上七条给定用例和一条 smoke，构成
最终 14 条测试。独立 review 提出**三条已答复的数字采样 finding**；最终 `double()`
配方有限且受 size 限制，不穷尽 IEEE-754。特殊数字在中间运行通过，但其封闭边界
配方未获最终校验接纳；这是工具限制，不是 API 输入排除或已确认 bug。观察语法由
测试代理自行修正，监督会话未手改生成测试。

源码 SHA-256 保持
`a4e315f1a2df6ae0d5a75f122023b46e60cf041960873de9d2587322d246e0ca`，给定套件逐字节
不变，报告及 required 独立 review 校验通过。这是预先准备的 `--no-isolate` 可调用
夹具，不是新的 CLI 隔离 E2E、生产应用/native 生命周期或实测代码覆盖率证明。
本次仅升级泛型默认值行：**28/0/6**，仍是 34 行。

### 覆盖固定教程的泛型约束

`Key extends Hashable` 表示实际源码约定，不是 `string`。解析本地/导入约束、别名、
依赖类型实参和接口扩展，保留名义类型的模块身份。class 约束可使用该可访问类本身，
不以派生类替代。函数与类的每个类型实参须明确给出或按上方声明默认值协议补齐；不升级任意推断。

约束沿用共享的接口/对象配方校验。已有源码类型 `HashedKey implements Hashable`
若有真实双 number 构造器、精确公开 `code`/`salt` 读取器和 `hash()`，有界整数示例是：

```ets
const codes = integer();
const salts = integer();
function shrinkKey(key: HashedKey): HashedKey[] {
  const out: HashedKey[] = [];
  for (const code of codes.shrink(key.code)) {
    out.push(new HashedKey(code, key.salt));
  }
  for (const salt of salts.shrink(key.salt)) {
    out.push(new HashedKey(key.code, salt));
  }
  return out;
}
const keys = recordOf((rng: Random, size: number): HashedKey =>
  new HashedKey(codes.generate(rng, size), salts.generate(rng, size)),
  (key: HashedKey): string => codes.show(key.code) + ':' + salts.show(key.salt), shrinkKey);
check(property(keys, (key: HashedKey): boolean =>
  hashKey<HashedKey>(key) === key.code * 31 + key.salt),
  { seed: 709, numRuns: 1000 });
check(property(keys, string(), (key: HashedKey, value: string): boolean => {
  const map = new MyHashMap<HashedKey, string>();
  map.set(key, value);
  return map.lastHash === key.code * 31 + key.salt && map.lastValue === value;
}), { seed: 709, numRuns: 1000 });
```

从 scoped 源码导入真实声明；示例依赖该源码实际的 hash/最后写入约定，不代表通用
哈希表语义。每个生成/收缩实例都是新对象，使用全部变化组件，以各组件 arbitrary
显示，每次只收缩一个组件并保留兄弟状态。真实公开单字段 setter 沿用下方接口协议；
不暴露 private 状态。也可从完整标量回调抽取值直接构造。receiver 构造参数同样需要
带类型抽取值；receiver 的正常初态不等于生成的 key。

每个直接 scoped 调用，包括同一性质内的后续调用，都须传入彼此独立、类型匹配且
未修改的抽取值。原始函数须操作其约束形参，而不是替换值或未调用闭包。观察最终
有效的返回值或兼容执行路径上的公开后态；void 实例方法必须通过公开字段或精确
getter/零参数读取器，观察自身 receiver 被 scoped 方法可达写入影响的状态（包括
已证明的单字段 setter）。常量/无关 getter、未修改字段不能证明效果；经 helper/别名
间接修改需要独立协议。不能以调用前快照或无关 key 读取替代。每次调用的后态须在
后续调用覆盖前观察或保留；重复调用须在循环内观察。已保留的调用后标量快照可以稍后
检查。void 独立函数同样必须观察原始约束形参的可达写入所影响的输入状态，包括
已证明的单字段 setter；未改变的分量不能证明效果。后续调用的输入位置只有通过
独立表达式中的已证明公开字段/精确读取器访问，才可视为只读；别名、逃逸及未证明的调用会保守地
使先前状态证据失效。沿用共享有界可达性及 finalizer/结果规则。
禁止强转、伪造/遮蔽实现类、构造器/原型替换、整实例逃逸和固定替身。仅供校验的单态
元数据视图不会编译或覆盖 SUT；原始 import 和源码完整性门禁仍生效。

阅读内置 `references/constraints.md`，使用既有原样 IR 绑定。兄弟参数仍受自己的域
门禁约束。native/SDK/Sendable/ArkUI、不可访问/抽象/继承构造、不透明递归和泛型/
重载接口成员仍 retired。组合行为输入及单独指定的约束构造器需要自身协议。自定义
标量 `recordOf<number>` 和从生成操作数组循环提取的 key，目前不是可接纳的约束
输入证据；设备执行本身不能替代校验。交付采用已识别的完整组件配方及直接回调抽取值，
诚实保留更宽的补充检查。`double()` 有限且受 size 限制，扩大声明边界不保证抽到了极值；
一般数字 Oracle 使用下方基于运算符的 SameValue，而非 `Object.is`。

2026-11-10，重建的 fmbot-pbt（pi 1.5.1）驱动真实 API-26 PandaCheck HAR、应用及
ohosTest HAP 构建，签名安装两个 HAP，并执行全部注册套件：**20 通过、0 失败/错误**。
六条未修改的 `PandaCheckConstraints.test.ets` 用例覆盖真实分派、新实例/重放、组件
收缩、明确的泛型函数/类调用及独立值。故意失败控制以种子 **709**、200 次运行收缩并
重放 **`2:0`**，不是 SUT bug；主机转译只是独立交叉检查。

同一 `build-run` 对未修改的 `hap_constraints_types.ets#set` 接纳 **6 条生成的 passing
性质、0 bug**，每条 1000 次，种子 **709–714**：string/number/boolean 交付、两步写入、
receiver 隔离和前缀不变性。此前六条更宽域性质设备执行通过，但其自定义标量/组合绑定
未获校验证明，故从机器交付中 retired；它们继续注册和执行，未删除或改称 SUT 失败。
一个补充确定性用例覆盖 **3375** 组异常数字/历史组合；加上六条给定用例和一条 smoke，
构成最终 20 条测试。独立 review 对当前有效性质无剩余缺陷，六条行政性 retired finding
均有答复。工具拒绝不表示生产输入无效，也不能单独证明 SUT bug。

源码 SHA-256 保持
`4744d65b4dcdd85381fac401dc44ef2de135a709a3d68449e2e203afc99e641e`，给定套件逐字节
不变。报告校验与 required review 校验通过，监督会话未手改生成测试。这是预先准备的
`--no-isolate` 可调用夹具，不是新的 CLI 隔离 E2E、生产应用/native 生命周期或实测
代码覆盖率证明。该阶段只升级泛型约束；泛型默认值交付另见上方。

### 覆盖固定教程的接口

解析实际接口的导入别名、具体泛型实参、方法、属性/访问器签名及全部扩展接口。
纯继承字段袋保留对象字面量协议；行为约定须导入真正可访问、明确 `implements`
该接口或派生接口的类，其声明属于 scoped 模块或可达源码导入图。每个继承的公开
成员都须匹配完整解析签名。不得实例化接口、编造测试本地实现、强转方法字面量、
替换原型或暴露 private 状态。**接口扩展不等于类继承。**

实际 `RectangleSize implements AreaSize` 若有公开宽高构造器和精确公开 getter，
可用具体类类型的配方：

```ets
const widths = integer();
const heights = integer();
function shrinkRectangle(value: RectangleSize): RectangleSize[] {
  const out: RectangleSize[] = [];
  for (const width of widths.shrink(value.width)) {
    out.push(new RectangleSize(width, value.height));
  }
  for (const height of heights.shrink(value.height)) {
    out.push(new RectangleSize(value.width, height));
  }
  return out;
}
const rectangles = recordOf((rng: Random, size: number): RectangleSize =>
  new RectangleSize(widths.generate(rng, size), heights.generate(rng, size)),
  (value: RectangleSize): string => widths.show(value.width) + ':' + heights.show(value.height),
  shrinkRectangle);
```

接口不是工厂。教程原样的 `RectangleSize` 只有初始化为零的私有尺寸、没有公开
初始化/读取 API 时，仍缺少状态配方；不得向硬性 `--scope` 添加这些 API。
每个构造实参及状态字段都要生成、显示并逐组件收缩；收缩保留真实实例、方法及兄弟
状态，默认值不等于开放字段抽取。边界性质里的常量/零宽度组件同样不能替代完整变化
状态；先调用 SUT、保留通用 Oracle，再按抽取值条件加强。ArkTS 以
`arkts-limited-stdlib` 拒绝 `Object.is`；数值 SameValue 先保存实际/预期结果，再用
`actual === expected ? (actual !== 0 || 1 / actual === 1 / expected) : (actual !== actual && expected !== expected)`，
保留 NaN/正负零语义，不调用受限 API。访问器支撑的 `Style` 可在生成及收缩候选中
新建 `StyledRectangle`，经实际公开 setter 给 `value.color` 赋抽取值，再经实际
getter 读取。识别的 setter 将带类型形参原样赋给一个 backing field，精确公开
reader 返回该字段；readonly 字段仍须经构造器初始化。不推断任意 setter 副作用。
`ExtendedStyle extends Style { width: number; }` 必须保留 color 和 width，
无论由公开字段还是访问器实现。

每条实际执行性质的每个直接调用都原样传入各自独立且带类型的抽取值。回调可保留
具体类类型以读取真实公开状态，也可使用实际接口类型。scoped 可调用对象（函数声明、
箭头/函数表达式或类方法）须直接在原始形参上调用/读写约定成员，不能用被遮蔽/替换值或未调用闭包冒充。仅在 catch 中，
或依赖未经证明的异常流继续执行后的使用不构成证明；try/finally 内的直接使用仍可识别。
源码使用证明及性质观察共享有界可达性分析，包括循环完成情况；遮蔽及可能的形参替换
仍单独保守检查。嵌套闭包/类不提供正向使用证据，但其中对捕获形参的写入仍要扫描，
包括解构及循环迭代赋值；真正遮蔽的局部变量/参数保持独立。
性质须观察结果，或在 void 调用后观察同一实例的公开状态。SUT 调用、读取及其结果使用都须
在兼容的执行路径上可达；常量死分支、短路、互斥路径、已终止路径和未经证明的
catch/异常流不算。被 finally 覆盖的 return/throw，以及被 catch 吸收的 throw，
即使其表达式执行了，也不能证明性质结果被观察；finally 自身未被覆盖的观察仍有效。
正常结束的嵌套 try/catch、switch 贯穿及无害标签保留待返回的观察，但完成状态摘要
不会把仅在 catch/switch/标签内执行的代码变成正向执行证据；带标签跳转及 with
的完成状态仍未经证明。
void/undefined 调用须有真实公开后状态证据，比较其返回值不算。
未解析的返回类型也不能提供值结果证据；泛型 SUT 的具体返回标注仍共享类型解析。共享输入证明先验证真实
实现及配方后，允许仅具体类具有的公开字段/getter/精确 reader；另一实例或调用前
快照不构成后状态证据。不能让整个抽取值经未知
helper/别名、容器或闭包逃逸，性质内直接构造的实例也一样；标量快照仍可使用。
用公开 reader 计算独立 Oracle，不替换 SUT 实参。真实 SUT setter 合法修改输入
不同于测试侧替换。

这是有界识别，不是全程序纯度/类型流推断。支持明确同步签名的直接接口实参；行为
接口的数组/Record/字段/联合、可选/默认/rest 参数、重载/泛型成员签名及仅经 helper
间接调用仍需独立协议。不可访问/抽象/Sendable/ArkUI 类、类继承、不可用 native/SDK
签名和生命周期及不透明状态仍 retired；纯字段袋保留已有组合。兄弟参数仍受自己的
策略约束。IR 使用原样 arbitrary 绑定并成功展开，保留原始 import/SHA，要求真实
构建、设备执行及被接纳的报告/review；主机夹具本身不是交付完成证据。

2026-11-10，重建的 fmbot-pbt 二进制（pi 1.5.1）构建真实 API-26 HAR 与
entry/ohosTest HAP，签名并安装两者，逐字节不变地执行给定的
`PandaCheckInterfaces.test.ets`。五条一致性用例覆盖真实实现身份、新对象/重放、
构造器及公开 setter 状态、继承约定、组件收缩和故意失败控制（种子 **708**，最小化
为 **`2:0`**）；该控制不是产品 bug。

同一 `build-run` 交付为未修改的 `hap_interfaces_types.ets#measure` 编写并展开 IR，
收尾接纳 **6 条 passing 性质、0 bug**，独立 review 无 finding。源码 SHA-256 保持
`087ff74120442e62ee1206787973c601c71f6d4d9f60d9c42473ae441e9f2fa8`。
初次完整设备运行 **12 条测试、0 失败/错误**（六条生成、五条给定、一条原有 smoke）。
代理自行修正配方/断言后，最终生成套件重跑 **6/6 通过**，未改动的一致性与 smoke
用例保留初次通过记录；不声称最终又完整重跑了全部套件。最终种子 **708** 驱动五条
各 1000 次抽取性质和一条 10000 次抽取性质，另有 81 对明确的异常数值。随机抽取
仍受范围及 size 限制，不声称穷尽数字域或实测行覆盖率。监督会话未手改生成套件。
此前编译 STOP 与 OHPM 仓库超时均保留，不算成功；成功构建使用官方
`ohpm install --metadata_cache` 复用已缓存的固定版本依赖。这是未改动的可调用夹具，
不是生产应用/native 生命周期证明。该次交付只升级了 interface 行。

### 覆盖固定教程的类型化 Record 域

`Record<string, V>` 用 `dictionary(string(), values)`；`Record<number, V>` 用
`numericDictionary(integer(), values)`，需要小数键时用 `double()`。
宽范围有限键可用 `ieeeDouble({ includeNaN: false, includeInfinity: false })`；
未禁用全部特殊值的 IEEE 键会被拒绝。值可用完整 IEEE 模式，保持组件显示/收缩。
HAP `double()` 无参数，不能传 fast-check 的 `min`/`max` 选项。数字键必须有限，
运行时存成规范字符串属性名，`-0` 变为 `0`。值沿用自身完整类型化配方。教程的真实
导入类型 `PersonInfo { age: number; salary: number; }` 先按下方对象协议构造完整
`recordOf(generate, show, shrink)` 的 `peopleValues`，再组合：

```ets
const peopleArb = dictionary(string(), peopleValues, { maxLength: 4 });
const numberedPeopleArb = numericDictionary(integer(), peopleValues);
// property(peopleArb, (people: Record<string, PersonInfo>) => actualSut(people))
```

两个字段都转发实际 RNG/size、用各自组件显示；每个收缩候选只改一字段并保留另一字段，
推入最终返回的数组。原语、bigint、Date、字面量/enum、数组/矩阵、嵌套 Record 及可用
真实构造器值按同一协议组合。精确三原语联合可用 `primitiveUnion()`；其它可用联合
须完整可收缩的 `unionOf`/`unionCase`，不能用不收缩的 `oneof`。

默认最多 **4 个键**，内联正整数 `maxLength` 可调低或调高。运行时零/负上限只生成空
映射，不能证明覆盖；非有限/小数上限抛错。空映射是合法抽取情况，不是足够的固定替身。
重复生成的键覆盖旧值；收缩依次尝试删键、改一个值、改一个键，重名不会覆盖兄弟键。
排除 `__proto__`，避免 prototype setter。映射是新对象，原映射不变；保留值遵循组件
引用语义，不保证深拷贝。种子可重放抽取及显示；贪心收缩不保证全局最小。

本地/导入/链式/泛型别名保留实际 nominal/模块身份。可选/undefined Record 用
`optional(mapArb)`，nullable 用 `nullish(mapArb)`，两者同时存在用
`optional(nullish(mapArb))`；非空反例的显示委托给内部组件。也可用完整带类型 case；map 分支谓词须排除声明的
nullish 臂。这个 Record 组件协议不改变兄弟标量/Date 联合的选择规则。根数组用
`array(mapArb, { minLength: 1 })`；Record 字段和嵌套映射使用完整组件配方。
带默认值/rest Record、字面量/联合/bigint 键、递归映射及不可用 native/SDK/Sendable/
ArkUI/不可访问值仍 retired；别名不制造缺失构造支持，也不能掩盖不支持的兄弟参数。

每条执行性质的每个直接 SUT 调用都原样传入各自独立且带类型的回调抽取值。合法基准调用
不能为第二个使用转换/固定映射的调用背书，后者会阻止整套结果确认。根据抽取映射计算独立
Oracle 值，增强生成器而不是替换 SUT 实参。cast、修改、
固定映射/键/开放值、错误值、缺失配方、无效收缩输出及未使用抽取均由共享生成/报告
检查拒绝。IR 使用原样 arbitrary 绑定，仍要求成功展开、原始 import/SHA 和真实
`aa test`。主机一致性执行本身不是设备或交付证据。

2026-11-10，重建版 fmbot-pbt（pi 1.5.1）构建真实 API-26 PandaCheck HAR 及
entry/ohosTest HAP，签名并安装到模拟器，执行未修改的 `PandaCheckRecords.test.ets`
一致性夹具。覆盖重放、新对象、上限、PersonInfo/其它值、整数/小数键、键/值收缩与
重名安全，以及故意失败的种子控制（种子 **707**，最小化为 `{'': 2:0}`）；控制不是产品 bug。

另由 CLI `build-run` 为未修改的 `hap_records_types.ets#totalCompensation` 编写并
成功展开 IR，保留源码 SHA-256
`33f6d388dce93882cd49e964c965ac825a0f53ac634bb7d74d061577fddd62d1`，收尾接纳
**6 条 passing 性质、0 bug**，独立收尾 review 无 finding。最终由二进制驱动完整重跑：
**12 条测试、0 失败/错误、12 通过**，含六条生成性质（每条 1000 次抽取）、五条给定
一致性用例和一条原有 smoke 测试。新增包装控制在 review 修复后重建 HAR，验证
optional、nullish 及二者组合的精确最小化显示和重放。交付套件由代理编写，监督会话未手改修复。
此前编译/结果确认失败被保留，根因修复进发布运行时/夹具/引导，没有冒充成功。这是未改动的
可调用夹具，不是生产应用、任意浮点或 native 域证明，也不声称实测行覆盖率。
Record 阶段仅升级该行。

### 覆盖固定教程的对象字面量

教程的 `class C { n: number; s: string; }` 可用上下文类型化字面量，不只用
`new C()`。带类型的初始化、单次赋值、直接实参、类型化局部 helper 返回、数组元素
及嵌套字段保留真实且未修改的性质抽取值；支持直接字段和简写。不要重声明生产类型，
不要对伪造对象 cast。真实新实例（含带方法的类）可在调用/返回前将类型化抽取值
直接赋给 public 字段。构造参数只覆盖经证明原样初始化的字段，其余字段需要赋值；
匹配构造签名不能替代固定默认值、遗漏开放字段、显示或收缩证据。readonly 状态
须经真实构造器初始化；private 状态也可经验证过的公开单字段 setter 初始化，并用
public 精确字段读取观察，不能非法赋值；逃逸别名和
后续修改均不算证据。

已批准的性质 IR 使用既有的原样生成器绑定通道（`generators: { input: inputs }`），
不要发明 `gen: object`。必须先成功调用 `expand_property_ir` 生成/内联骨架，再填入
真实导入、生产回调类型和完整配方；失败的 IR 调用不算完成。

较大输入使用三个参数齐全的 `recordOf(generate, show, shrink)`。组件 arbitrary
绑定为 `const`，`.generate(rng, size)` 转发真实 RNG/size，返回新的完整类型化字面量。
显示必须观察每个字段；bigint、函数及其它非 JSON 安全字段（包括嵌套组合）必须用
同一组件 arbitrary 的 `show`。`JSON.stringify` 对 bigint 抛异常，并省略函数字段，
只能作为 JSON 安全字段的显示证据。每次用同一组件
的 `shrink(value.field)` 只替换一个字段，保留其它字段，并 push 到最终返回的数组。
支持 `for-of` 与规范下标循环；单值字段可保留唯一字面量。固定开放字段、缺失/重复键、
错误组件、丢弃收缩结果、cast 或修改抽取值均拒绝。

组件包括原语、bigint、Date、字面量/enum、数组/矩阵、嵌套字段袋、字段袋数组、
已支持字典、具体真实函数值及固定元组字段。嵌套形态各用独立具名配方，不再只有原语
或一层嵌套。可选字段在整个组件外包 `optional`；nullish 联合保留全部臂。根字段袋
数组需要 `minLength: 1` 或非空上下文抽取字面量。泛型字段实参及默认值从实际本地/
导入/命名空间声明解析。

public 纯数据类可用字面量；需要构造行为、方法或 private 状态的类仍用真实可访问
导入构造器，在生成和收缩候选中保持 nominal/运行时身份。构造检查只跟踪直接字段
赋值及匹配的原语重载分支，不推断任意构造副作用；识别 public 精确字段读取，复杂
不透明构造/观察仍 retired。不合成 native/SDK/Sendable、ArkUI、不可访问/
类继承或不透明递归对象；可选/rest 元组槽及不可用底层组合仍有独立限制。泛型**函数**、
根 Date 数组/默认参数/static 字段规则不变。这是固定教程的对象字面量能力，不是通用
构造、任意 helper 纯度或全程序控制流证明。

只对已证明参数位置生成主机元数据视图，不编译、不覆盖源码或套件；原始导入、SHA、
实际构建和设备执行仍是权威证据。上游与内置版本分别独立构建 API-26 HAR 与签名
应用/测试 HAP，各通过 **9 条测试、0 失败/错误**：教程上下文、组合字段、真实类身份、
新对象、种子 `1`、`42`、`0xC0FFEE`，以及收缩到 `Date(1000)` 和
`0n:Date(0):undefined:[]:[10,]`（使用组件的 bigint 显示），以及构造后赋值收缩到
`10:`。带方法的实例分别验证完整构造实参、完整 public 赋值及两者混合，收缩后仍
保留方法与真实类身份。这些是语言/库一致性模型，不是生产应用证据。
另外，已记录的重建版 pi-pbt `build-run` 曾测试未修改的 Browser
`DownloadTask.createDownloadTask`，核实 `DownloadManager` 和 `DownloadRepository`
真实调用者：退出 0，四条 passing 性质被接纳、零 bug；真实设备 **5 条测试、0 失败/
错误、5 通过**。正式 IR 工具生成一个文件、零跳过；测试任务填充骨架并完成收尾，
监督会话未手改套件。源码/调用者 SHA 及原始/isolate 源码 inode 不变；新建、签名并
安装测试 HAP，复用已安装且相同的基础 Ability 应用。这证明应用里的类型化对象、
可选/字面量字段及工厂默认值，不声称生产 Date/bigint/矩阵覆盖。最终编译后 JS
只读重验接纳性质清单，拒绝十七种内存变异及错误源码 SHA；重验不是新设备执行。

对象字面量能力保持不变；加上上方接口、约束及默认值完成，当前为 **28 已支持 / 0 部分支持 / 6 不支持**，仍是 34 行。

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

规范化仅用于检查元数据，不改真实生产源码、导入、SHA 或编译的 SUT。对象字段使用
上方完整协议；根 Date 数组/默认参数、不可访问构造器、继承、Sendable 和不可用 SDK
叶子不能借别名绕过。
class 默认值遵循上方完整声明默认值协议。别名阶段只升级该行，不升级全部底层部分支持项。

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
函数的参数使用各自独立、未修改且带类型的抽取值。生产约定保留状态时检查重复调用。
简单推断和真实函数变量的上下文类型受支持；复杂不透明推断需要实际类型化包装，
不能编造声明。

上游和内置版本分别独立构建真实 API-26 HAR、应用/测试 HAP，各通过六条一致性测试：
内建/导入/static 函数值、推断箭头、新闭包状态、生成行为、固定种子重放和精确阈值
收缩。这些是库/语言模型，不是生产 SUT。另用重建的 fmbot-pbt 对未修改的 Photos
`CustomTransitionManager.registerNavParam` 运行真实测试任务，并核实其 `SlideShowPage`
调用者：退出 0，四条 passing 性质被接纳、零 bug；设备五条测试、零失败/错误、五通过。
生成的函数模型通过真实存储的回调及 public 生命周期驱动捕获效果；未改 SUT、SDK 或
产品清单。这证明生产交付链路，不声称全部内建/导入/返回形态均已在应用中执行。

函数能力不升级其它底层域；加上接口、约束及默认值能力后，当前清单是
**28 已支持 / 0 部分支持 / 6 不支持**。无具体签名的 `Function`、
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
必须明确给出或来自声明默认值，不做猜测。例如现有 `Stamp.when` 约定确实保留
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
owner 及不可用底层类型仍 retired。Date 对象字段现用上方独立对象字面量协议；
根 Date 数组/默认实参、普通 static Date 字段及不支持的回调组合仍有各自限制。先证明原始绑定，再生成仅供主机
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
getter/setter 阶段达到 **23/5/6**；上方特性完成使当前清单达到 **28/0/6**，仍是 34 行。

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
`functionModel` 的结果；返回闭包必须真实调用。泛型实参须明确给出或由声明默认值完整确定；可选/rest/默认参数
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
签名没有联合参数时，不相关的组合器不会强加联合协议。组合成功也不能绕过不可用
底层域；对象字段用独立验证协议，根 Date 数组与带默认值 Date 参数仍 retired。

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
nullish Date 联合。Date 对象字段及其数组组件使用独立对象字面量协议；根 Date
数组/默认实参、普通 static Date 字段和不透明构造仍保留限制。Date getter/setter 使用上方 accessor 协议；真实 SUT 的
backing field 不等于任意字段袋构造。导入/链式 Date 别名与泛型 Date 默认值是可见
元数据，适用对应 Date 覆盖或不支持底层域门禁。这些残余限制不升级其它行；
默认值完成后的当前清单是 **28/0/6**。

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

上方对象字面量协议也支持兼容字面量的数据类、嵌套/数组字段袋及转义字面量。
不透明递归形态和不可用联合臂仍 retired。当前清单为 **28 已支持 / 0 部分支持 /
6 不支持**。

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
| 值导入能解析到检出里的 scoped 文件，或 `--isolate` 注册并通过完整性检查的 common 副本 | 对镜像全局做 `declare class` |
| 若本 HAP 无法按现状导入 `--scope`，则 `retired` | 仅 `import type`，或注释里提一下路径 |
| 签名含 `| null`、`| undefined` 或可选参数时：用完整 `unionOf` / `unionCase`；既有 `oneof` + `constant(null)` / `constant(undefined)` 仅用于没有 `unionOf` 的测试文件。原样传入，不用 `as`、`!`、`??` | 丢掉该臂，或用 `as` / `ESObject` 收窄 |
| `--scope` 里声明的 `enum`：`constantFrom` 列出每个成员，回调类型为该 enum，不用 `as` | 用 `integer()` / `string()` 再 `as` 成 enum，或漏掉一个成员 |
| `string \| number` 等原语联合：完整带类型的 `unionOf` / `unionCase`；每臂 `oneof` 仅用于没有 `unionOf` 的测试文件，不用 `as` | 只抽一臂，或 `as` 成其中一臂 |
| 源码解析出的兼容字面量对象：上下文字面量或上方完整 `recordOf(generate, show, shrink)` 配方。显示全部字段、逐组件收缩并保留其它字段；可选字段包裹整个组件。根字段袋数组用 `minLength: 1`；需要行为的类用真实构造器。 | 缺失/固定/cast/修改字段、空或丢弃收缩结果、伪造构造器身份，或不可用 native/SDK/Sendable/ArkUI/private/继承叶子 |
| 泛型：匹配的显式实参或完整声明默认值，保留实际底层配方/约束、新建真实 receiver、原样类型化输入及调用观察 | 无完整默认值的值推断、冲突覆盖/标注、假 receiver、不可用类型或 `ESObject` / `any` / `@Sendable` |
| 实际接口约定：真实源码实现类、完整状态配方、匹配的继承方法/访问器及可观察 SUT 使用 | 接口构造、测试本地替身、错误/缺失成员、private 写入、强转、原型替换或丢弃抽取值 |
| 具体函数参数：签名匹配的真实具名/箭头/导入/内建函数值，或解释已抽取 `FunctionModel` 的类型化闭包；工厂返回函数须调用真实结果 | 缺失/错误/未用回调、假接收者、仅类型导入、覆盖绑定、强转或未使用模型 |
| `--scope` 重载：用独立、未修改且带类型的抽取值调用每个完整有序签名和元数，不用 `as` | 只抽第一个签名，或只收集无序原语臂 |
| HAR 原语与 interface 字段袋联合：每臂 `oneof` 仅用于没有 `unionOf` 的测试文件；class 实例臂要求真实构造器 case，不用 `as` | 只抽一臂 |
| Sendable 参数（`@Sendable` 类、`SendableMap` / `ISendable`、`collections.Array`）：**retired** | `integer()`，或本线程上的 `new Box(n)` |
| 品牌 SDK 类型（`Length`、`Resource`、未在 `--scope` 声明的 `@kit.*` / `@ohos.*` 导入）：**retired** | `integer()`，或 `as Length` |
| 一维品牌 `T[]` / `Array<T>`（`ImageItem[]`）：该实参只传 `[]` 或 `constant([])`，不用 `as` | 构造 `ImageItem`、非空品牌数组，或仅因元素是品牌类型就对该 `--func` retired |
| 原语字段袋的 `Name[]`，包括 SDK `FingerInfo[]`：`array(recordOf(fn, show, shrinkName), { minLength: 1 })`；`{ x, y }` / `{ w, h }` / LTRB 也可用 `record()`。回调类型为 `Name[]`，把该值传给 `--func`，不用 `as` | `[]`、画出了却没传入、没有 `minLength: 1` 的 `array()`，或 `{ id: 0 } as FingerInfo` |
| 独立 `Date` 用 `date()`；时间戳联合用完整组合配方，或仅在没有 `unionOf` 的测试文件中用 `dateUnion()`。原样传入对应实参；对象字段用独立组件协议，根 Date 数组/默认参数仍 retired | 缺臂、工厂遮蔽、单值/不透明 Date 边界、下标修改或未调用的闭包 |
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
- 泛型实参可明确给出或由声明默认值完整确定；使用上方实际约束、底层配方及逐调用门禁，不从值推断缺失类型。
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
  对象使用上方完整上下文字面量/组件协议，支持可用的嵌套、Date/bigint/矩阵和固定
  元组字段，要求全部字段显示及组件收缩。根字段袋数组用 `minLength: 1`；SDK
  `FingerInfo[]` 保留既有例外，品牌 `ImageItem[]` 仍是 `[]`
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
   `arkNullish`，不 fork fast-check。`--scope` 里的 enum 用成员的 `constantFrom`。对象使用上方完整上下文字面量/组件协议，不再只有原语或一层嵌套；要求每个字段
   的显示及组件收缩。根字段袋数组用 `minLength: 1`，品牌数组与既有 SDK 例外保留自身
   规则；不可用叶子仍 retired。具体函数参数使用上方完整协议，包括真实导入/内建函数值、生成回调行为和返回闭包调用。重载（含方法、构造器和无分号形式）的每个签名都要按完整参数顺序带类型调用。这些臂的混合联合用每一臂的 `oneof`。Sendable 参数应 retired（[#543](https://github.com/fermat-hkrc/FMBot-PBT/issues/543)）。品牌 SDK 类型（`Length`、`Resource`）应 retired（[#545](https://github.com/fermat-hkrc/FMBot-PBT/issues/545)）。品牌 `T[]`（`ImageItem[]`）用 `[]` 或 `constant([])`（[#630](https://github.com/fermat-hkrc/FMBot-PBT/issues/630)）。含不可抽臂的联合应 retired（[#547](https://github.com/fermat-hkrc/FMBot-PBT/issues/547)）。含不可用底层域的重载仍应 retired（[#549](https://github.com/fermat-hkrc/FMBot-PBT/issues/549)）；支持的非原语不再统一 retired。同文件被调函数里的 throw 应捕获（[#551](https://github.com/fermat-hkrc/FMBot-PBT/issues/551)）。约束到不支持类型的泛型应 retired。`name<string>` 不是 `T extends SomeStruct`。泛型类方法是 `new CustomStack<string>()` 再在该实例上 `stack.push(drawn)`。`Function` 或 `ESObject` 回调应 retired。具名 stub 只适用于 `(...) =>`。`passing` / `failing` 必须导入并调用 `--func`。只导入文件再调用兄弟符号不是该符号。字符串字面量联合、数字字面量联合，以及二者混合，用每个字面量的 `constantFrom` 或 `oneof(constant(...))`。`string()` / `integer()` 不是该联合。`[number, number]` 一个参数用 `property2` 构造 `[x, y]`，更多的数对用不同的 `numberPair`。不要传入 `Pair`。字面量与开放原语的联合用 `oneof(constant(...), integer()/string())`。四跳之外的 throw 应捕获或 retired。第五跳仍然看不见。缺失的 `hvigorw` / SDK 已经推迟构建门。长期：同
   `check`/`property` 形态的 ArkTS PBT 库，无 `any`、无 `<const T>`。扫描 scoped
   签名，只允许匹配的 arbitrary。

## 相关

- [安装：HAP / ArkTS 应用](../subcommands.zh.md#build-run)
  （[English](../subcommands.md#build-run)）
- [英文 ArkTS 页](arkts.md)
- [子命令](../subcommands.zh.md) — `build-run --build-cmd "hvigorw assembleHap --mode module -p module=entry@ohosTest -p product=default --no-daemon"`
- [Kea GUI 测试](../kea.zh.md) — 真机黑盒 UI，不是源码级 ArkTS
- 内置 ArkTS 测试指南：DevEco、SDK 对齐、PandaCheck HAR、签名、`aa test`
