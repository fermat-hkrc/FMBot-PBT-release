# ArkTS PBT (PandaCheck)

[中文版 / Chinese version](arkts.zh.md)

[Back to installation](../subcommands.md#build-run) ·
[Subcommands](../subcommands.md)

fmbot-pbt supports ArkTS as **HarmonyOS HAP ohosTest**. A property is Hypium +
PandaCheck in `src/ohosTest/ets/test/<Name>Pbt.test.ets`
(`import { check, property, integer, double, ieeeDouble, string, boolean, bigint, dictionary, numericDictionary, primitiveUnion, date, dateUnion, array, recursive, functionModel, FunctionModel, unionCase, unionOf } from 'PandaCheck'`),
assembled with `hvigorw`, and run with `hdc shell aa test`. The verdict is
`OHOS_REPORT_RESULT`. Install the library with `make-pandacheck-har.sh` into
`libs/PandaCheck.har`. Depend on `"PandaCheck": "file:../libs/PandaCheck.har"` from the module `oh-package.json5`. Do not copy `Index.ets` into `libs/`. Do not `ohpm install fast-check`.

This is not Kea. This is not OpenHarmony GN C++. Host `es2panda`/`ark`
(`PBT_PANDA=1`, `text()`/`bool()`, `PANDACHECK_RESULT`) is a **separate** runner
for compiler `.ets` that is not a HAP. Do not mix the two.

`--isolate` (with `--scope file.ets --func Name`) builds a **mini HAP** that
copies the real `--scope` file and its relative imports into `entry/src/common/`,
shared by main and ohosTest. These are independent regular files, never hard links
or symlinks. Only unused named local imports may be stripped by the generator;
function bodies and the checkout stay unchanged. Every generated file and original
dependency is hash-baselined before the build. Passing/failing close-out rejects
changes, missing files or redirected paths on either side. Use it to test a
function in a large app (Photos) without compiling the entire app. The import graph must be public kits (`@ohos.hilog`, `@ohos.i18n`, …).
System APIs (`systemParameterEnhance`, `photoAccessHelper`, …) or a graph bigger than a
function **STOP**. `--isolate` does not edit HARD `--scope`. The campaign cwd is `<out>/isolate-hap`.
Import that generated file under `entry/src/common` (the prompt names the specifier,
for example `../../../common/default/model/common/DurationFormatter`). Do not edit
it, replace it with a symlink, or author another copy under main/ets or ohosTest.
The original graph is also read-only. `--build-cmd` is
replaced by `assembleHap … entry@ohosTest` after `ohpm install` in that HAP.
Isolate STOP and a failed `ohpm install` exit 3, same as a failed build gate.

## Feature inventory

Scored from the OpenHarmony tutorial
[Introduction to ArkTS](https://github.com/openharmony/docs/blob/f41b9345badd47c7ab0c263344cd7f4b5a549afb/en/application-dev/quick-start/introduction-to-arkts.md)
at `f41b9345badd47c7ab0c263344cd7f4b5a549afb`. This is that tutorial's named
types, call shapes, and modules. It is not a percent of ArkTS.

`let`, `const`, inference, `void`, operators, `if` / `switch` / loops, and
`this` are syntax. They are not in the count. A property does not draw them.

**28 supported, 0 partial, 6 not supported** on this 34-row checklist.
Supported means a passing or failing property can use that form, and a
close-out gate requires it. Partial means only some forms of that tutorial
heading are drawn or caught. Not supported means we do not draw it, or the
honest close is `retired`.

| Tutorial feature | fmbot-pbt | What that means |
| --- | --- | --- |
| `number` | supported | `integer()` / bounded finite `double()`, or explicit `ieeeDouble()` for ordinary draws plus representative IEEE edges. Fractional thresholds require `double()` or the explicit IEEE mode; `integer()` does not cross them. See finite/special-domain and display rules below. |
| `bigint` | supported | `bigint()`. `integer()` is not `bigint`. |
| `boolean` | supported | `boolean()`. |
| `string` | supported | `string()`. |
| `Object` | not supported | Bare `Object`, aliases of bare `Object` and direct Object/nullish annotations retire. Only the bounded nullable-Object alias recipe below accepts real-constructor representatives; it does not provide universal SDK/object construction. |
| array of `number` / `string` / `boolean` | supported | `array(integer())`, `array(string())`, or `array(boolean())`. A bare element is rejected. A non-empty array of a `number`/`string`/`boolean` bag, including SDK `FingerInfo[]`, is `array(recordOf(fn, show, shrinkName), { minLength: 1 })`. Empty `T[]` when `T` is SDK branded (`ImageItem[]`, `Length[]`) may only be `[]` / `constant([])`. |
| `enum` | supported | `constantFrom` of every member in the same file, or one relative import. A second hop is invisible. |
| union | supported | The pinned tutorial's class/primitive/nullish unions use typed `unionOf` / `unionCase`: real constructor instances retain `instanceof` and methods, with per-arm shrinking and complete-arm close-out checks. Primitive/literal and interface-bag legacy forms remain only in test files without `unionOf`. Resolved Date and scalar/primitive-array compositions are also supported. This is union composition, not automatic construction of opaque SDK/native leaves; unavailable underlying types still retire. |
| type alias | supported | Tutorial `Matrix`, `Handler`, `Predicate<T>` and `NullableObject`; syntax-aware chains, scopes, imports/re-exports and generic substitution/defaults. Nested arrays and bounded productive recursive arrays require complete typed recipes. Aliases preserve underlying SDK/native/Sendable/constructor limits, not new value constructors. |
| optional parameter | supported | Include the `undefined` case with `unionOf` / `unionCase`. Legacy `oneof` with `constant(undefined)` is allowed only in a test file without `unionOf`. |
| default parameter | supported | A `number`, `string`, or `boolean` default must be both omitted and passed as a drawn value. |
| rest parameter | supported | `func(...xs)` where `xs` is `array()` of `number`, `string`, or `boolean`. A bare list is rejected. |
| function type | supported | The tutorial's concrete function values, including `Math.sin`, real named/arrow/imported/static functions, inferred/contextual arrows and lexical/returned closures. Complete signatures and real call bindings are checked. `functionModel` supplies seeded generated behavior and shrinking through typed closures. Untyped `Function` / `ESObject` interoperability is separate, not a concrete signature. |
| overload | supported | Tutorial free functions, public instance/static methods and constructors (also semicolonless headers). Every complete ordered signature and arity needs a scoped, unchanged, typed call. Supported non-primitive domains keep their own recipes; only an unavailable underlying domain forces retirement. |
| class or `struct` with fields | supported | Every instance field needs a drawn constructor argument or a complete public field assignment, including method-bearing classes. Constructor recipes also need complete display/shrinking. Public data classes may use contextual literals/typed `recordOf`; `record` only has numeric XY/WH/LTRB shapes. Constructor defaults are not open-field draws. |
| `static` field or method | supported | A public static method with a `number`, `string`, or `boolean` parameter must be called as `Class.method(drawn)`. A public static field of those types, including an unannotated `= 0`, `= 'albumType'`, or `= true`, is `Class.field = drawn` and a read of `Class.field` in that property callback. A readonly field is read. A private or protected field, a non-exact type, and a field of `Date` stay outside. |
| getter / setter | supported | Tutorial `Person4.age`, getter-only and setter-only forms, plus Date and supported underlying domains. Actual public instance/static ownership, fresh receivers, typed constructor/setter draws, same-receiver observed reads, bounded getter inference and per-operation catches are checked. Private state, substitutes and unavailable underlying domains do not count. |
| inheritance, `super`, abstract class | not supported | Not constructed. |
| `private` / `protected` field | not supported | The suite must not edit `--scope` to expose it. |
| object literal | supported | Tutorial contextual initialization, assignment, arguments, typed returns, array elements and nested fields use actual source-resolved interface/public data-class types. Direct/shorthand literals retain typed draws; composed `recordOf(generate, show, shrink)` recipes support available nested fields, Date/bigint/matrices, literals/enums, fixed tuple fields, optional values and real constructor leaves. Complete field generation/display and one-component shrinking are checked. No constructor substitution, private/ArkUI/Sendable/native synthesis or source rewriting. |
| `Record<K, V>` | supported | String keys use `dictionary(string(), values)`; numeric keys use `numericDictionary(integer(), values)`. Complete typed values include tutorial `PersonInfo` bags, primitives, Date/bigint, arrays and nested maps. Aliases and supported wrappers retain unchanged typed SUT arguments, complete component display/shrinking, key caps and collision safety. See the exact boundaries and real HAR/HAP/device plus `build-run` delivery evidence below. |
| interface | supported | Actual local/imported methods, accessor properties and interface extension use genuine accessible source-owned implementing classes with complete state recipes. Pure inherited field bags retain literal recipes. See the exact boundaries and real HAR/HAP plus unchanged-callable delivery evidence below. |
| generic function | supported | Matching explicit types or complete declared defaults are required. Primitive instantiations and declared bounds retain their underlying recipes; value-based inference is rejected. |
| generic constraint | supported | Actual imported/substituted bounds, including tutorial `Hashable.hash()`, use fresh source-owned implementations with complete state, display and shrinking. Every direct scoped call retains matching explicit/defaulted types, unchanged draws and observed dispatch. Unavailable construction still retires; see boundaries and real HAR/HAP/device plus delivery evidence below. |
| generic class | supported | Explicit `new CustomStack<string>()` and genuine constrained receivers such as `new MyHashMap<HashedKey, string>()`, freshly constructed inside the property with typed constructor/method draws and observed effects. Complete declared defaults may supply omitted types; value-inferred receivers and unavailable bounds are rejected. |
| generic default | supported | Actual omitted function/class/interface arguments require complete source-resolved defaults, preserving lexical/import identity, dependent substitutions, explicit overrides and bounds. Every value-bearing domain keeps its generator/real constructor, display/shrink and per-call observation gates. See unchanged-source HAR/HAP/device and build-run evidence below; this is not arbitrary inference or automatic Date/class construction. |
| nullish `T` or `null` / `undefined` | supported | Complete `unionOf` / `unionCase` cases; legacy `arkNullish` only in a test file without `unionOf`. The call must not use `!` or `??` to drop the arm. |
| named import of `--func` | supported | A passing or failing property must import the file and call that symbol. |
| import alias or namespace | supported | `id as ident` and `ids.id(...)` count. A sibling call does not. |
| default export | supported | A default-exported function must be imported as the default and called. A class or value default is not this binding. |
| dynamic `import()` | not supported | The result is `ESObject`. |
| `throw` / `try` | supported | A throw in `--func`, a same-file callee, a direct import, or up to four hops must be caught. A fifth hop is invisible. |
| `@interface` annotation | not supported | Not a value. |
| ArkUI | not supported | Declarative UI is not this runner. |

`@Sendable` is in the kit overview, not in this tutorial. A Sendable parameter
is `retired`. Do not add it to this count.

### Explicit IEEE-754 numeric domains

`double()` keeps its bounded finite generation and shrinking. The opt-in
`ieeeDouble()` combines ordinary decimal/threshold draws, representative edges,
wide finite values and subnormals. It includes NaN and both infinities by default;
`includeNaN` and `includeInfinity` independently default **true**. For finite-only
sampling use `ieeeDouble({ includeNaN: false, includeInfinity: false })`, still
including signed zeros, subnormals and large magnitudes. Do not silently broaden
an existing finite-only property to special values.

Campaign witnesses require omitted options or an inline object of literal boolean
flags, with no unknown/duplicate/computed keys, spreads, casts or opaque options.
Use unmodified named PandaCheck imports and stable arbitrary bindings. Numeric
Record **keys** require both flags false; values may use the full mode. All
existing complete-domain, actual-call, constructor ownership, distinct unchanged
typed draw, observation and unsupported-sibling checks remain in force. Closed
`constantFrom<number>` edge sets are supplemental, not an open numeric recipe.

Use each component's `show`, including nested object/array/map/optional recipes:
JSON loses NaN/infinities and signed zero, while raw string coercion loses `-0`.
Built-in wrappers, numeric bags and tuples delegate to their components. Preserve
same-component shrinking into fresh real instances, one field at a time.
PandaCheck retains `recordOf(generate, show, shrink?)`; **campaign** recipes still
require complete effective shrinking. Predicate exceptions count as failures even
if formatting would be empty or throw; framework generation/shrink/show errors
still propagate. This does not remove campaign-specific exception-oracle checks.

For an actual identity-preserving scoped `echo`, retain its result and compare
inline: `actual === value ? (actual !== 0 || 1 / actual === 1 / value) :
(actual !== actual && value !== value)`. This is ArkTS-compatible SameValue;
`Object.is` is restricted, and `===` alone is insufficient. Derive oracles from
the SUT contract: rounding, overflow and NaN invalidate many ordinary algebraic laws.

Sampling is **not uniform or exhaustive**, does not cover all NaN payloads, and
finite trials do not prove every region was visited. Shrinking strictly lowers
finite magnitude ranks, with bounded duplicate-free candidates and `-0 → 0`, but
the runner remains greedy/budget-limited: no arbitrary-predicate global minimum
or convergence within every finite budget is promised.

The bundled numeric card is `references/numbers.md`. `hap/UPSTREAM.json` records
canonical PandaCheck revision `f46b330bbae6a81e409f0bdf442c6ffffaff6e3d` and SHA-256
hashes of the unchanged runtime and IEEE controls; offline tests need no sibling
checkout. Node controls, real HAR/HAP/Hypium execution, a new `fmbot-pbt build-run`
campaign, report/independent campaign review and PR/CI remain separate gates.
The pin explicitly declares `adapterDifferences: []`: there are no runtime
patches; stricter campaign policy stays outside PandaCheck.
`npm run check:pandacheck` verifies the actual source bytes in both `check` gates. Builds
also verify staged `dist` assets, and binary smoke verifies the freshly extracted
runtime and pin rather than trusting the manifest alone. For online provenance,
`node scripts/check-pandacheck.mts --upstream` hashes the allow-listed files at
the exact GitHub revision using authenticated `gh`. Offline CI needs no sibling
checkout or cross-repo token; missing online read access is an error, not success.

The older #710 evidence below remains historical; it did not use `ieeeDouble`.
The inventory stays **28/0/6**: this extends number sampling, not a new language row.

On 2026-11-11 a newly rebuilt fmbot-pbt binary drove `build-run` on unchanged
`hap_defaults_types.ets#read`. It authored `DefaultBoxReadIeeePbt.test.ets` with
**eight passing properties, 0 bugs**, seeds **420101–420108**, **4096** trials each
(32,768 total): omitted defaults with genuine `SomeType` recipes and explicit
`number` receivers, each under all four finite/special modes. All scoped reads
retain identity/SameValue observations, including repeated reads and explicit
matching default elaboration. The supervisor did not hand-repair the suite.

Actual API-26 HAR/application/test HAP rebuilds, signing/install and whole-suite
Hypium passed **28 tests, 0 failures/errors**: eight generated properties, eight
unchanged IEEE controls, four legacy-double controls, seven default controls and
one smoke. Initial baseline was 20/20; first generated run was 28/28 at 1000
trials/property; final strengthening was 28/28 at 4096. IEEE controls retain seeds
1, 42 and 0xC0FFEE, progressing/terminating shrinking, ordinary/wide/subnormal
samples, option separation and faithful NaN/infinity/`-0` replay. Their intentional
failures are framework controls, not SUT bugs.

Source SHA-256 stayed
`a4e315f1a2df6ae0d5a75f122023b46e60cf041960873de9d2587322d246e0ca`; all three supplied
suites stayed byte-identical, and installed runtime bytes match the upstream pin.
Fresh independent review covered all eight properties with **zero findings**;
report and required-review revalidation passed. This is a preprovisioned
`--no-isolate` callable fixture, not a new isolation E2E, production app/native
lifecycle or measured line/branch coverage proof.

### Tutorial-complete generic defaults

This is a **defaulted type argument**, not a default-valued function parameter.
Omission is accepted only when declared defaults determine every missing type.
Explicit arguments win: `first<T = number, U = T[]>` selects `number, number[]`
when omitted, `string, string[]` for `first<string>`, and keeps both arguments
for `first<string, boolean[]>`. No inference from values fills required types.
Resolve imports, aliases and namespaces in the declaration's lexical context;
class defaults precede method defaults without leaking through shadowed names.
Self/forward references cannot borrow globals. Every actual bound still applies.

For unchanged `echo<T = number>(value: T): T` returning its input:

```ets
check(property(integer(), (value: number): boolean => echo(value) === value));
check(property(string(), (value: string): boolean => echo<string>(value) === value));
```

A default is not a generator: `T = Date` needs `date()`. Existing `SomeType`
with a number constructor and exact public `value` reader needs a complete recipe:

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

Import the actual source-owned classes. This assumes the real
`DefaultBox<T = SomeType>` constructor/read contract; `new DefaultBox<string>(text)`
uses its explicit override. `Reader<T = SomeType>` selects that default in an
actual interface formal/annotation, never by constructing an interface or fake
implementation. Conflicting annotations cannot override actual construction.
No-value defaults such as `defaultKind<T = number>()` still need an observed call.
Every value-bearing position retains complete seeded generation, component display,
effective shrinking, distinct unchanged typed draws and surviving call observations.
Void calls retain the affected-state/per-call history checks below. Receivers are
fresh inside each property. Casts, fixed substitutes, missing recipes, mutations,
escapes and unsupported siblings fail closed. Class inheritance, unavailable
native/SDK/Sendable construction, separately scoped generic constructors and
unsupported behavioral compositions remain separate limits.

Use the bundled `references/defaults.md` card and existing verbatim IR binding
(`generators: { value: items }`). Preserve omitted/explicit syntax in tests;
checker-only instantiation views never replace the compiled SUT. Approved IR must
expand successfully, followed by real build, device, report and review gates.

On 2026-11-10, rebuilt fmbot-pbt (pi 1.5.2) drove API-26 PandaCheck HAR and both
application/ohosTest HAP builds, signing/install and final whole-suite `aa test`:
**14 passed, 0 failures/errors**. Seven unchanged supplied
`PandaCheckDefaults.test.ets` cases cover omitted/explicit functions, dependent
defaults/overrides, real class/interface dispatch, receiver independence,
fresh seeded generation/display and effective shrinking. The intentional control
replays seed **710**, 200 runs, minimized **`2`**; it is not a SUT bug.

The same `build-run` on unchanged `hap_defaults_types.ets#read` accepted
**six generated passing properties, 0 bugs**, 1000 trials each at seeds
**71001–71006**: omitted/matching class arguments, string/number/boolean overrides
and independent read interleaving. Together with seven supplied cases and one
smoke, these are the final 14 tests. Independent review raised **three answered
numeric-sampling findings**; final `double()` recipes are finite/size-limited,
not exhaustive IEEE-754 generation. Special numeric cases passed in an intermediate
run but their closed-edge recipe was rejected for final attestation; that is a
tooling limitation, not an API exclusion or a confirmed bug. Generated observation
syntax was corrected by the campaign, not hand-repaired by the supervisor.

Scoped SHA-256 remained
`a4e315f1a2df6ae0d5a75f122023b46e60cf041960873de9d2587322d246e0ca`;
the supplied suite stayed byte-identical. Report and required independent-review
validation passed. This preprovisioned `--no-isolate` callable fixture is not a new
CLI-isolation E2E, production-app/native-lifecycle proof or measured code coverage.
Only the generic-default row is newly promoted: **28/0/6**, still 34 rows.

### Tutorial-complete generic constraints

`Key extends Hashable` means the actual source contract, not `string`. Resolve
local/imported bounds, aliases, dependent type arguments and interface extension
without losing nominal module identity. Class bounds may use the actual accessible
class, not a derived-class substitute. Complete every function/class type argument
explicitly or through the declared-default protocol above; arbitrary inference is not promoted.

Use the shared interface/object recipe witnesses. For an existing source-owned
`HashedKey implements Hashable`, with a real two-number constructor, exact public
`code`/`salt` readers and `hash()`, the bounded integer example is:

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

Import the real declarations from the scoped source. The examples assume that
source's actual hash/latest-write contract, not universal hash-table semantics.
Every generated/shrunk implementation is fresh, uses every varying component,
displays through its component arbitrary and shrinks one component while retaining
siblings. Real public one-field setters follow the interface protocol below;
private state is never exposed. Direct construction from complete scalar callback
draws is also available. Receiver constructor arguments need their own typed draws;
a receiver's normal initial state is not a generated key.

Every direct scoped call needs distinct unchanged correctly typed arguments,
including later calls in the same property. The original callable must exercise its
constrained formal, not a replacement or unused closure. Observe a surviving value
result or compatible public post-state; void instance methods require their own
receiver's affected state, via a public field or exact getter/zero-argument reader
connected to reachable scoped writes (including proven one-field setters).
Constant/unrelated getters and untouched fields are not effect evidence; indirect
helper/alias mutations need a separate protocol. Pre-call snapshots and unrelated
key reads do not count. Observe or retain each call's state before another call
overwrites it; repeated calls need an in-loop observation. A retained post-call
scalar snapshot may be checked later. Void free functions apply the same proof to
input state affected by reachable writes of the original constrained formal,
including proven one-field setters; unchanged components are not effect evidence.
Later input positions count as read-only only through standalone proved public
field/exact-reader accesses; aliases, escapes and unproved calls conservatively invalidate earlier state.
Shared bounded reachability and finalizer/outcome rules apply. No casts, fake/shadowed classes,
constructor/prototype replacement, whole-instance escapes or fixed substitutes.
Metadata-only monomorphic views are never compiled or written over the SUT; raw
import and source-integrity checks remain in force.

Read the bundled `references/constraints.md` card and use the existing verbatim IR
binding. Unsupported sibling domains remain visible. Native/SDK/Sendable/ArkUI,
inaccessible/abstract/inherited construction, opaque recursion and generic/overloaded
interface members still retire. Composed behavioral inputs and separately scoped
constrained constructors need their own protocols. Custom scalar `recordOf<number>`
recipes and keys extracted from generated operation-array loops are not currently
accepted constrained-input witnesses; device execution alone cannot attest them.
Use recognized complete component recipes and direct callback draws for delivery,
retaining wider supplemental checks honestly. `double()` is finite and size-limited;
changing a bound does not guarantee extreme values were sampled. General numeric
oracles use the operator-based SameValue expression below, not `Object.is`.

On 2026-11-10, rebuilt fmbot-pbt (pi 1.5.1) drove real API-26 PandaCheck HAR,
application and ohosTest HAP builds, signed/installed both HAPs, and ran the whole
registered suite: **20 passed, 0 failures/errors**. The six unchanged supplied
`PandaCheckConstraints.test.ets` cases cover genuine dispatch, freshness/replay,
component shrinking, explicit generic function/class calls and independent values.
Its intentional control shrank/replayed **`2:0`**, seed **709**, 200 runs; it is not
a SUT bug. Host transpilation is only a separate cross-check.

The same `build-run` delivery on unchanged `hap_constraints_types.ets#set` accepted
**six generated passing properties, 0 bugs**, 1000 trials each at seeds **709–714**:
string/number/boolean delivery, two-write history, receiver isolation and prefix
invariance. Six earlier broad-domain properties passed on-device but were retired
from machine delivery because their custom scalar/composed bindings were unproved;
they remained registered and executed, not erased or relabeled as SUT failures.
A supplemental deterministic case checked **3375** exceptional-number/history
combinations. Together with six supplied cases and one smoke case, these make the
20-case final run. Independent review found no remaining active-property defects;
six administrative-retirement findings were answered. Tooling rejection does not
exclude a production input or establish a SUT bug.

Scoped source SHA-256 remained
`4744d65b4dcdd85381fac401dc44ef2de135a709a3d68449e2e203afc99e641e`;
the supplied suite was byte-identical. Report attestation and required review
validation passed. The supervisor did not hand-repair generated tests. This was a
preprovisioned `--no-isolate` callable fixture, not a new CLI-isolation E2E,
production-app/native-lifecycle proof or measured code coverage. That milestone
promoted only constraints; generic-default delivery is recorded separately above.

### Tutorial-complete interfaces

Resolve the actual interface, including imported aliases, concrete generic arguments,
methods, property/accessor signatures and every extended interface. Pure inherited
field bags retain the object-literal protocol. Behavioral contracts need an imported
accessible class declaring `implements` for that interface or a derived interface;
its declaration must belong to the scoped module or reachable source imports.
Every inherited public member must match its complete resolved signature. Never
instantiate an interface, invent a test-local implementation, cast a method literal,
replace prototypes or expose private state. **Interface extension is not class inheritance.**

For an existing `RectangleSize implements AreaSize` with a public width/height
constructor and exact public getters, use a concretely typed recipe:

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

The interface is not a factory: the exact tutorial's private zero-initialized
`RectangleSize`, without public initialization/read APIs, still needs an available
state recipe; do not add those APIs to HARD `--scope`. Every constructor argument
and state field must be generated, displayed and component-shrunk. Real instances,
methods and sibling state survive shrinking; defaults do not cover open fields.
Neither do constant/zero-width components in an edge-only property: draw the full
varying state, call the SUT, retain the general oracle and strengthen conditionally.
ArkTS rejects `Object.is` (`arkts-limited-stdlib`). Numeric SameValue uses stored
`actual`/`expected` and `actual === expected ? (actual !== 0 || 1 / actual === 1 / expected) : (actual !== actual && expected !== expected)`;
this preserves NaN/signed-zero semantics without a restricted API.
For accessor-backed `Style`, generation and shrink candidates may create a fresh
`StyledRectangle`, assign a drawn `value.color` through its actual public setter,
and read through its actual getter. A recognized setter assigns its typed formal
unchanged to one backing field; an exact public reader returns that field.
Readonly fields require constructor initialization. Arbitrary setter effects are
not inferred. `ExtendedStyle extends Style { width: number; }` retains both
color and width, whether fulfilled by public fields or accessor implementations.

Every direct call in every checked property passes distinct unchanged typed draws.
The callback may retain the concrete class type to access genuine public readers,
or the actual interface type. The scoped callable (declaration, arrow/function
expression or class method) must directly exercise a contract member on its original
formal, not a shadowed/replaced value or unused closure.
Catch-only use and continuation after unproved exception flow are not witnesses;
direct uses inside try/finally remain available. Source-use evidence and predicate
observations share bounded reachability, including loop completion, with separate
conservative shadowing/replacement checks. Nested closures/classes provide no
positive-use evidence, but their captured-formal writes are still scanned, including
destructured and iteration assignments; genuinely shadowed locals stay distinct.
Observe its result or public
same-receiver post-state after a void call. The SUT call, the read and its result use must
all be reachable on a compatible execution path; constant-dead branches, short
circuits, mutually exclusive paths, terminated paths and unproved catch/exception
flow do not count. Returns/throws overridden by a finalizer and caught throws do
not observe the property outcome, even though their expressions execute. A
finalizer's own surviving observation remains valid. Normal nested try/catch,
switch fallthrough and harmless labels preserve pending observations; their
completion summaries do not admit catch/switch/labeled-only execution evidence.
Labeled transfers and `with` completion remain unproved.
Void/undefined calls require actual public post-state evidence,
not a comparison with their return value. Unresolved return types cannot provide
value-result evidence; concrete generic return annotations retain shared resolution. Concrete-only public fields/getters/exact
readers are allowed after the shared input witness proves the implementation and
recipe; a different instance or pre-call snapshot is not post-state evidence. Do not
escape whole draws through unknown helpers/aliases, containers or closures (also
for direct construction inside the property); scalar snapshots remain valid.
Compute independent oracles from public readers without replacing SUT arguments. A legitimate SUT setter may
mutate its input; test-side substitution is different.

Recognition is bounded, not whole-program purity/type-flow inference. Direct
interface arguments with explicit synchronous signatures are covered; behavioral
interface arrays/records/fields/unions, optional/default/rest parameters,
overloaded/generic member signatures and helper-only dispatch need separate
protocols. Inaccessible/abstract/Sendable/ArkUI classes, class inheritance,
unavailable native/SDK signatures/lifecycles and opaque state still retire.
Pure field bags retain existing compositions. Unavailable sibling inputs remain
visible to their own policies. Use verbatim arbitrary bindings in successfully
expanded IR, preserve raw imports/source SHA, and require real build, device and
accepted report/review evidence. Host fixtures alone do not complete delivery.

On 2026-11-10, the rebuilt fmbot-pbt binary (pi 1.5.1) built the real API-26 HAR
and entry/ohosTest HAPs, signed/installed both, and executed the supplied
`PandaCheckInterfaces.test.ets` byte-identically. Its five conformance cases cover
real implementation identity, freshness/replay, constructor and public-setter
state, inherited contracts, component shrinking and the intentional failing
control (seed **708**, minimized **`2:0`**). The control is not a product bug.

The same `build-run` delivery authored/expanded IR for unchanged
`hap_interfaces_types.ets#measure` and closed with **6 accepted passing properties,
0 bugs**, plus a clean independent review. Source SHA-256 remained
`087ff74120442e62ee1206787973c601c71f6d4d9f60d9c42473ae441e9f2fa8`.
The initial full device run passed **12 tests, 0 failures/errors** (six generated,
five supplied, one existing smoke). After agent-owned recipe/assertion corrections,
the final generated-suite rerun passed **6/6**, while the unchanged conformance and
smoke cases retained their initial passes; this is not a claim of a final full-suite
rerun. Final seed **708** drove five 1000-trial properties and one 10000-trial
property plus 81 explicit exceptional-number pairs. Random draws remain bounded
and size-limited; no exhaustive numeric or measured line-coverage claim is made.
The supervisor did not repair the generated suite. An earlier compiler STOP and
an OHPM registry timeout were retained, not counted as successes; the successful
build used official `ohpm install --metadata_cache` with the cached pinned dependency.
This is an unchanged callable fixture, not production-app/native-lifecycle proof.
That delivery promoted only the interface row.

### Tutorial-complete typed Record domains

`Record<string, V>` uses `dictionary(string(), values)`; `Record<number, V>` uses
`numericDictionary(integer(), values)`, or `double()` for fractional numeric keys.
Wide finite keys may use `ieeeDouble({ includeNaN: false, includeInfinity: false })`;
unrestricted or partly disabled IEEE keys are rejected. Values may use the full
IEEE mode, with component-owned display/shrinking. HAP `double()` takes zero
arguments, not fast-check `min`/`max` options.
Numeric names are canonical finite numbers stored as string property keys; `-0`
becomes `0`. Values use their own complete typed recipe. For the tutorial's actual
imported `PersonInfo { age: number; salary: number; }`, construct `peopleValues`
with the object `recordOf(generate, show, shrink)` protocol below, then:

```ets
const peopleArb = dictionary(string(), peopleValues, { maxLength: 4 });
const numberedPeopleArb = numericDictionary(integer(), peopleValues);
// property(peopleArb, (people: Record<string, PersonInfo>) => actualSut(people))
```

Forward actual RNG/size to both fields, display both with their components and
shrink one field at a time into the returned array while retaining the other.
Primitives, bigint, Date, literals/enums, arrays/matrices, nested records and
available real constructor values compose the same way. The exact primitive
triple may use `primitiveUnion()`; other available unions use complete shrinking
`unionOf`/`unionCase` recipes, not non-shrinking `oneof`.

The default key cap is **4**; an inline positive integer `maxLength` can lower or
raise it. Runtime zero/negative caps yield empty-only maps and cannot attest
coverage; nonfinite/fractional caps throw. Empty maps are valid generated cases,
not a sufficient fixed substitute. Repeated generated keys overwrite; shrinking
tries removing keys, one value, then one key, never overwriting a sibling key.
`__proto__` is excluded to avoid the prototype setter. Maps are fresh, originals
unchanged; retained values keep their component's reference semantics, not deep
cloning. Seeds replay draws and display; greedy shrinking is not global minimality.

Local/imported/chained/generic aliases retain actual nominal/module ownership.
Optional/undefined Records use `optional(mapArb)`, nullable Records use
`nullish(mapArb)`, and both use `optional(nullish(mapArb))`. These wrappers retain
inner component display for non-nullish counterexamples. Complete typed cases
are also accepted. Map-case ownership excludes the declared nullish arms.
This Record-specific component protocol does not change sibling scalar/Date
union selection rules. Root arrays use `array(mapArb, { minLength: 1 })`;
Record fields/nested maps keep their whole component recipe. Defaulted/rest
Records, literal/union/bigint keys, recursive maps and unavailable
native/SDK/Sendable/ArkUI/inaccessible values still retire. Aliases do not create
missing construction support or excuse an unsupported sibling.

Every direct SUT call in every checked property passes distinct unchanged typed
callback draws. A valid baseline call does not license a second call with a
transformed or fixed map: it blocks suite attestation. Compute independent oracle
values from the drawn records; strengthen generators, not the SUT arguments. Casts, mutations, fixed maps/keys/open values, wrong values,
missing recipes, ineffective shrink output and unused draws are rejected by the
shared generation/report witness checks. Use verbatim arbitrary bindings in
property IR; successful expansion, raw imports/SHA and real `aa test` remain
required. Host conformance execution alone is not device or delivery evidence.

On 2026-11-10, rebuilt fmbot-pbt (pi 1.5.1) built the real API-26 PandaCheck HAR
and both entry/ohosTest HAPs, signed and installed them on the emulator, and ran
the unchanged `PandaCheckRecords.test.ets` conformance fixture. It checks replay,
freshness, bounds, typed PersonInfo/other values, integer/fractional keys,
collision-safe key/value shrinking and the intentional seeded failure control
(seed **707**, minimized `{'': 2:0}`). That control is not a product bug.

Separately, the CLI `build-run` campaign authored and successfully expanded IR
for unchanged `hap_records_types.ets#totalCompensation`, retained its SHA-256
`33f6d388dce93882cd49e964c965ac825a0f53ac634bb7d74d061577fddd62d1`, and filed
**6 passing properties, 0 bugs**, with a clean independent close-out review.
The final binary-driven full-suite rerun reported **12 tests, 0 failures/errors,
12 passes**: six generated properties (1000 draws each), five supplied
conformance cases and one pre-existing smoke test. The added wrapper control
verifies exact minimized display and replay through optional, nullish and both
composed wrappers, after rebuilding the HAR with the review fix. Only the agent authored the
delivery tests; the supervisor did not repair that suite. Earlier compiler and
attestation failures were retained and fixed in the shipped runtime/fixtures/steering,
not passed off as success. This is an unchanged callable fixture, not production
application or arbitrary floating-point/native-domain proof; no measured line
coverage is claimed. The Record milestone promoted only that row.

### Tutorial-complete object literals

The pinned tutorial's `class C { n: number; s: string; }` can be supplied by a
contextually typed literal, not only by `new C()`. Typed initializers, a single
assignment, direct arguments, typed local helper returns, array elements and
nested fields retain actual unchanged property draws. Direct/shorthand fields
are supported; do not redeclare production types or cast a fabricated object.
A fresh actual class instance, including a method-bearing class, can use direct
public field assignments from typed draws before the call/return. Constructor
arguments count only for the fields they demonstrably initialize unchanged;
remaining fields need assignments. A matching constructor signature alone does
not cover fixed defaults, omitted open fields, display or shrinking. Readonly state needs its real constructor; private state may also use a verified
public one-field setter, with available public exact-field reads, never forbidden writes. Escaped aliases and later mutations are not witnesses.

For approved property IR, name the object recipe through the existing verbatim
generator channel (`generators: { input: inputs }`), not an invented `gen: object`.
Require successful `expand_property_ir` scaffolding/inlining before filling its
real imports, production callback type and complete recipe; failed IR is not
completion.

For larger inputs, use `recordOf(generate, show, shrink)` with all three
arguments. Bind each component arbitrary with `const`, forward the actual RNG
and size to `.generate(rng, size)`, and return a fresh complete typed literal.
The display must observe every field. Bigint, callable and other non-JSON-safe
fields require their same component arbitrary's `show`, including nested
compositions. `JSON.stringify` throws for bigint and omits callable fields;
it is display evidence only for JSON-safe fields. Shrink each field with that same component's `shrink(value.field)`,
retaining all siblings, and push candidates into the array actually returned.
Both `for-of` and canonical indexed loops are recognized. Singleton fields may
retain their sole literal value. Fixed open fields, missing/duplicate keys,
wrong components, discarded shrink output, casts and mutated draws are rejected.

Components include primitives, bigint, Date, literals/enums, arrays/matrices,
nested bags, arrays of bags, supported dictionaries, concrete callable values
and fixed tuple fields. Compose separate named recipes; there is no primitive-only
or one-level nesting cap. Optional fields wrap the **whole** component with
`optional`; required nullish cases retain every arm. Root bag arrays require
`minLength: 1` or nonempty contextual drawn literals. Generic field arguments and
defaults resolve from their actual local/imported/namespaced declarations.

A public data-only class may use a literal. A class requiring constructor
behavior, methods or private state still needs its genuine accessible imported
constructor and keeps nominal/runtime identity in both draws and shrink candidates.
The bounded constructor check follows direct field assignments and matching
primitive overload branches, not arbitrary constructor effects. Exact public
field readers are recognized; opaque construction/observation retires. No synthetic native/SDK/Sendable,
ArkUI, inaccessible/class-inherited or opaque recursive object values. Optional/rest
tuple slots and unavailable underlying compositions retain separate limits.
Generic **function** restrictions and root Date-array/default/static-field rules
are unchanged. This is the tutorial's object-literal feature, not universal
construction, arbitrary helper purity or whole-program control-flow inference.

Only successfully proved argument positions receive host metadata projections.
Those views are never compiled or written over source/test files; raw imports,
original SHA, actual builds and device execution remain authoritative.

Independent upstream and bundled API-26 HAR plus signed app/test HAP runs each
passed **9 tests, 0 failures/errors**, including tutorial contexts, composed
fields, real class identity, freshness, seeds `1`, `42`, `0xC0FFEE`, and exact
nested/tuple shrinking (`Date(1000)` and `0n:Date(0):undefined:[]:[10,]` with
component-owned bigint display), plus constructor-and-assign shrinking to `10:`.
Method-bearing instances exercise complete constructor arguments, complete public
assignments and mixed construction/assignment, preserving methods and real class
identity through shrinking. These are
language/library conformance models, not production application evidence.

Separately, the recorded rebuilt-binary pi-pbt `build-run` campaign tested unchanged Browser
`DownloadTask.createDownloadTask`, with live `DownloadManager` and
`DownloadRepository` callers: CLI 0, four accepted passing properties, zero bugs,
and actual device **5 tests, 0 failures/errors, 5 passes**. The official IR tool
scaffolded one file with zero skipped entries; the campaign filled it and ran
closeout without supervisor suite edits. Source/caller hashes and original/isolate
source inode stayed unchanged. It built, signed and installed a new test HAP,
reusing the already-installed identical base Ability app. This anchors typed
object construction, optional/literal fields and factory defaults in an app;
it does not claim production Date/bigint/matrix coverage. Final compiled-JS
read-only replay accepted the ledger and rejected seventeen in-memory mutations
plus a wrong source SHA; replay is not another device run.

Object-literal support remains unchanged. With interface, constraint and default completion above,
the current checklist is **28 supported / 0 partial / 6 not supported**, still 34 rows.

### Tutorial-complete type aliases

Aliases name domains, not constructor values. The pinned tutorial's forms are:

```typescript
type Matrix = number[][];
type Handler = (s: string, no: number) => string;
type Predicate<T> = (x: T) => boolean;
type NullableObject = Object | null;
```

`Matrix` uses `array(array(integer()))`, not one row or a fixed matrix. Keep
its complete callback type and pass distinct unchanged draws at the actual SUT
positions. `Handler` / `Predicate<number>` require the actual named/arrow/imported/intrinsic function value's complete arity, resolved parameter types and return type at every SUT
call. Contextual function-variable annotations can provide types, not missing
parameters or conflicting explicit types. Zero-argument Handlers and string
callbacks for `Predicate<number>` are rejected; explicit SUT type arguments take
precedence over defaults. Defaulted/optional/composed callback domains still need
their own underlying protocol.

Only a source parameter naming an alias of exactly `Object | null` or
`Object | null | undefined` uses complete `unionOf` cases with fresh accessible
real-constructor representatives. Bare `Object`, its aliases, direct Object/nullish
annotations and Object arrays/other unions retire. Plain objects must not stand
in for a nominal SUT class.

Resolution follows lexical scopes/namespaces, generic substitutions/dependent
defaults, relative imports/re-exports and local OHPM `file:` source entries.
It is bounded to 40 source files within the source workspace. Missing/private
exports, ambiguity and pure alias cycles fail closed. Constructor/module identity
is retained: an imported class cannot become a same-named local class or builtin.

Productive recursive arrays, e.g. `type Nested = number | Nested[]`, use HAP
`recursive(base, expand, { maxDepth: 3 })`. `base` covers terminating cases;
`expand(child)` retains those cases and uses the actual child arbitrary inside
recursive array positions. Bound branching with `array(child, { maxLength: 3 })`.
Runtime depth is an integer 0–8, default 3; coverage requires omitted options or
an inline depth 1–8. Pure `type A = B; type B = A` is not productive recursion.
Generation is seeded, with component-owned show/shrink; shrinking is greedy,
not a universal minimality claim.

Normalization is checker-only. The actual production source, imports, SHA and
compiled SUT remain authoritative. Object fields use the protocol above; aliases
cannot hide unsupported root Date arrays/default parameters, inaccessible
constructors, inheritance, Sendable or unavailable SDK
leaves. Class defaults follow the complete declared-default protocol above.
The alias milestone changed only that row, not every underlying partial row.

Independent upstream and bundled HAR/HAP conformance on API 26 each passed six
tests, covering the four tutorial forms, imports/defaults and recursive replay,
freshness and nested threshold shrinking. Those are library models, not production
SUTs. A separate unchanged Settings `TimeUtil.obtainFormatTime` campaign through
fmbot-pbt exited 0 with five accepted failing properties; the device reported nine
tests, eight failures, zero errors and one pass. Its candidate findings are not
confirmed bugs. That existing direct timestamp alias is a production delivery
anchor, not evidence that every new imported/generic/recursive form ran in an app.

### Tutorial-complete function types

The tutorial calls `doAction(Math.sin)` with `type trigFunc = (x: number) => number`.
This is a real intrinsic function value, not an invented local stub. Concrete
function annotations and aliases now accept actual named/arrow functions, real
relative named/default/namespace imports, public static function values and
unshadowed/unmodified ECMAScript numeric intrinsics such as `Math.sin`. Simple
expression/block return inference, checked contextual types and lexical captures
are supported. A genuine factory result can itself be passed as a callback.

Every direct SUT call in each property must retain the actual callback's full
formal count, resolved parameter types, return type and nominal/module identity.
Context cannot invent missing formals or override conflicting explicit types.
Optional/rest formals inside a concrete function type retain their complete
annotations. Missing/wrong callbacks, type-only imports, replaced bindings,
quoted calls and uncalled closures do not count. Public instance methods use
real accessible constructors imported from the scoped module; a same-named fake
receiver or private-field/constructor edits cannot stand in for that instance.

Generate callback behavior with HAP `functionModel(outputs)` and a genuinely
typed closure. For a real scoped `apply(fn, value)` with numeric signature:

```ets
import { check, property, integer, functionModel, FunctionModel } from 'PandaCheck';
import { apply } from '../../../main/ets/ActualModule';

const result = check(property(functionModel(integer()), integer(),
  (model: FunctionModel<number>, value: number): boolean => {
    const fn = (x: number): number => model.call(x.toString());
    return apply(fn, value) === model.call(value.toString());
  }), { seed: 42, numRuns: 100 });
```

This model is generated **test input**, never a copy/model of the SUT. Keys must
be stable and include every actual formal, not a call counter, random value or
fixed key for a function with inputs. The result arbitrary must match the actual
`FunctionModel<R>` type and the callback result. A `void` callback may use its
model's result for captured observable effects, checked through the real public
lifecycle. A drawn but unused/mistyped model cannot close out. Its call result must
feed the callback return (directly, via unchanged local aliases or supported value
expressions), or a property-local captured value/statically named field observed
by the predicate. `model.call(key); return 1;`, unused/overwritten results, `void`
or comma-discarded calls and unobserved captures do not count. Keep captured
storage free of subsequent direct overwrites; opaque helper-mediated effects and
arbitrary control flow are not inferred.

The fresh finite table and salt use only the supplied RNG. Runtime `maxOutputs`
is an integer 1..16, default 4; coverage of callbacks with inputs requires omitted
options or an inline cap 2..16. Cap 1 is a valid constant-function runtime recipe.
Shrink tries constant projections, removes a bucket, or changes one output using
that component's `shrink`; `show` is `fn{seed:...,values:[...]}` with component
formatting. Originals are not edited. Results retain their reference semantics,
not arbitrary deep clones. Shrinking is greedy, not a global-minimum guarantee.

A scoped factory returning a closure must have its actual result invoked inside
each property. Merely calling the factory is not closure coverage. Returned
function arguments use distinct unchanged typed draws; exercise repeated calls
when the production contract retains state. Simple inferred closures and real
function-variable context are supported; opaque inference needs an actual typed
wrapper, not a fabricated declaration.

Independent upstream/bundled real API-26 HAR + app/test HAP runs each passed six
conformance tests: intrinsic/imported/static values, inferred arrows, fresh
closure state, generated behavior, seeded replay and exact threshold shrinking.
These are library/language models, not production SUTs. Separately, rebuilt
fmbot-pbt ran unchanged Photos `CustomTransitionManager.registerNavParam`, with its
live `SlideShowPage` caller: CLI 0, four accepted passing properties, zero bugs;
the device reported five tests, zero failures/errors, five passes. Generated
function models drove captured effects through stored real callbacks and public
lifecycle methods. No SUT, SDK or product manifests were rewritten. This is a
production delivery anchor, not app proof of every intrinsic/import/return form.

Function support does not promote unrelated domains. With interface, constraint and default completion above,
the current checklist is **28 supported / 0 partial / 6 not supported**.
Untyped `Function`, `ESObject`, dynamic imports, unknown native lifecycles,
Sendable and inaccessible/inherited constructors retain their separate limits.
Optional/defaulted/composed callback **parameters**, arbitrary generic/async
cross-products and opaque recursive function generation still need their own
underlying protocols. Composed callback domains retire identically when written
directly or through aliases; one function arm is not a direct union of function
types. The gate is bounded syntax/binding analysis, not arbitrary
whole-program type/control-flow inference.

### Tutorial-complete getters and setters

The pinned tutorial's `Person4.age`, including the negative-age throw, and its
explicit getter-only/setter-only forms are supported. Select the actual property
with `--func age`, or `--func Person4.age` / `Ns.Person4.age` when owners are
ambiguous. This is property syntax (`person.age`, `person.age = value`), not a
fabricated `age(...)` method.

Each executed PandaCheck property constructs a fresh accessible imported receiver
inside its callback, or uses the actual static owner. Constructor arguments and
setter values are unchanged typed draws. Named/default/namespace import aliases
retain the real class identity. A paired setter write must be followed by an
observed getter read on the same receiver. Getter-only values must be observed;
setter-only effects need public observations on that receiver or actual generated
callback effects. Catch each throwing access in the property. Invoke a concrete
function returned by a getter with matching typed draws.

Date accessors use `date()`; explicit types and bounded inference from real
backing fields/initializers or simple returns are supported. Arrays/matrices,
literals, enums, interface bags, dictionaries, aliases, real class values, complete
unions and callbacks retain their own generation/shrink protocols. Generic receiver
arguments must be explicit or declared defaults, not guessed. For an existing
`Stamp.when` contract that preserves the assigned timestamp:

```ets
check(property(date(), (value: Date): boolean => {
  const stamp = new Stamp();
  stamp.when = value;
  return stamp.when.getTime() === value.getTime();
}));
```

An unrelated/older read, uncalled closure/object method, unchecked property,
discarded result, fixed/mutated/cast input, fake/shared receiver, private-state
edit or constructor/prototype replacement cannot supply the witness. Unresolved
inference, opaque factory/drawn receivers, inaccessible/inherited/Sendable owners
and unavailable underlying types still retire. Object fields now use the separate
object-literal protocol above; root Date-array/defaulted inputs, plain static
Date fields and unsupported callback compositions retain their own limits. Original bindings are proved before host-only
call projections; those projections never compile or rewrite the SUT. Raw
import/transcription/SHA guards remain authoritative. Recognition is bounded,
not whole-program control-flow or purity proof.

Independent upstream/bundled API-26 HAR plus signed app/test HAP runs each passed
**9 Hypium tests, 0 failures/errors**: tutorial exceptions, Date/inferred and
one-sided accessors, static/generic receivers, composed values, real classes,
generated callbacks, freshness/replay and shrinking to `10` and `Date(1000)`.
These are language/library conformance models; neither runtime API nor the host
implementation changed.

Separately, a fresh rebuilt-CLI campaign tested unchanged Photos
`GridScrollMsg.isScrolling`, with live `AbstractDayGridVc` writes and
`ImageGridItemComponent` reads: CLI 0, four accepted passing properties, zero
bugs; actual device **5 tests, 0 failures/errors, 5 passes** (four PBT + Ability).
Production/source-caller hashes and the original/isolate source inode were
unchanged. The official IR/closeout flow generated the suite; the supervisor did
not edit it. No SUT, SDK or product-manifest rewrites. This application anchor
covers boolean store/read, not Date usage or asynchronous timer semantics.
Compiled-JS read-only replay accepted the ledger, rejected eighteen in-memory
accessor mutations and rejected a wrong source SHA. Replay is not another device
run. The getter/setter milestone reached **23/5/6**; the completed features above
bring the current inventory to **28/0/6**, still 34 rows.

### Tutorial-complete overloads

The pinned tutorial's free `foo1(number/string)`, instance `C.foo`, and
`C(number/string)` constructor forms are supported, including constructor headers
without semicolons. Static methods, import aliases, namespaces and real default
function exports retain the actual overload owner. Select `Class.method` when a
bare method name has multiple owners; unrelated signatures cannot be borrowed.

Each declared signature needs a complete ordered argument/arity witness inside
an executed `check` / `checkWith` property. Use separate typed properties where
signatures differ: `(number, string)` does not cover `(string, number)`, and a
broad union callback is not automatically callable against separate overload
headers. Implementation signatures are not additional public overloads. Quoted
calls, uncalled helpers, unchecked property creation, casts, mutations, shadows,
fixed arguments and invalid extra calls do not establish coverage.

There is no overload-specific non-primitive ban. Arrays, matrices, Date, exact
literals, enums, supported interface bags/dictionaries/unions, class aliases and
concrete callbacks use their existing underlying protocols. Classes need fresh
real public constructor `recordOf(fn, show, shrink)` recipes, not plain objects;
generated callbacks must consume their `functionModel` results. Returned closures
must actually be invoked. Explicit or complete declared-default instantiations, optional/rest/default
coverage and residual Date/native/Sendable/accessibility limits remain in force.
This is the tutorial's overload feature, not automatic generation for arbitrary
higher-arity argument packing or unsupported type cross-products; HAP `property`
still takes one to three arbitrary components.

Original AST/container identity and source offsets choose the signatures. Views
for legacy guards are checker-only strings: the production file is never rewritten
or replaced, and raw import/transcription/SHA checks still apply. Duplicate
resolved parameter lists, ambiguous owners, ambient declarations, inaccessible
implementations and more than 64 headers fail closed. This is bounded
syntax/binding recognition, not general control-flow or whole-program inference.

Independent upstream and bundled API-26 HAR plus signed app/test HAP builds each
passed **8 Hypium tests, 0 failures/errors** on the device. They cover the tutorial
forms, ordered arguments/arities, real class/Date/array/enum/bag/callback values,
freshness, seeded replay and class shrinking to `Item:10`. These are explicitly
language/library conformance models, not a production application campaign.

Separately, rebuilt fmbot-pbt `build-run` in normal HAP mode exercised unchanged
PandaCheck `constantFrom<T>(values)` / `(first, ...rest)`: CLI 0, four accepted
passing properties, zero bugs; the device passed five tests (four generated PBT
cases plus the Ability harness), with no failures/errors. The compiled scoped
file was a hard link to the original library source, verified by inode and SHA;
no SUT, SDK or product manifest was rewritten. Generation used the official
property IR and close-out flow, not a supervisor-edited suite. The run could read
earlier campaign artifacts; its fresh build, installation and actual device run
are the evidence. This is real-library delivery, not application-level proof;
no suitable application overload target was established. Read-only compiled-JS
revalidation rejected twelve signature/draw/import mutations and a wrong SHA.

### Tutorial-complete union composition

The tutorial's union examples are `Cat | Dog | Frog | number | string | null |
undefined` and the class-only form narrowed with `instanceof`, with shared and
variant-specific methods. These are no longer treated as undrawable class arms.
Use `unionOf<T>(unionCase<T, Arm>(label, arbitrary, owns), ...)` with an explicit
case for every arm. A class case uses a fresh `recordOf` constructor recipe for
a real public class imported from the scoped module, not a plain object. Its
methods and identity survive generation and shrinking.

For example, a `string | Date` recipe is:

```typescript
type Value = string | Date;
const values = unionOf<Value>(
  unionCase<Value, string>('string', string(), (v: Value): boolean => typeof v === 'string'),
  unionCase<Value, Date>('Date', date(), (v: Value): boolean => v instanceof Date));
```

The callback keeps the whole union and passes the distinct drawn value unchanged
at the matching SUT argument. Every union argument, including optional/nullish
ones, needs its full recipe. Shared immutable recipes are allowed. Missing arms,
wrong type predicates, substituted factories, scalar-only draws, overwritten
inputs, uncalled closures and plain-object class substitutes cannot close out.
Resolvable union aliases and their imports are normalized. Legacy primitive/literal and interface-bag
unions remain available only in a test file without `unionOf`; class-instance
unions always require the typed constructor protocol.

**Protocol selection:** for a scoped signature with union arguments, any
`unionOf` call in the test file selects composition for every union argument in
every property, including optional/nullish arguments. Do not mix `dateUnion()`,
`oneof`, `nullish` or `arkNullish` union-argument recipes into that file. This rule
is about SUT arguments, not ordinary optional fields inside a bag. An unrelated
compositor does not impose a union protocol on a non-union scoped signature.
A valid composition does not bypass unavailable underlying domains. Verified
object fields use the object-literal protocol; root Date arrays and defaulted
Date parameters elsewhere in the signature still retire.

The runtime uses the supplied RNG, checks disjoint case ownership, shrinks in the
selected arm and formats `label:componentShow`. Predicates must be pure and
mutually exclusive. Overlapping erased shapes require a disjoint representation;
no claim is made that every opaque SDK object or arbitrary ArkTS type can be
manufactured. The close-out recognizer is bounded lexical/structural analysis,
not general control-flow/dataflow proof; helper-mediated or arbitrary deep
mutations are outside it. Inheritance, inaccessible constructors, Sendable/native lifecycles,
opaque recursive/native domains and unsupported leaf types keep their own
limitations. This promotion covers the **pinned tutorial's union forms**, not all
cross-products of the other partial/unsupported checklist rows.

The portable `PandaCheckUnion.test.ets` library conformance fixture exercises all
seven arms, real prototypes/methods, freshness, seeded replay and class threshold
shrinking on a device. That fixture is not a production SUT campaign; production
linkage is separately checked through `fmbot-pbt build-run` on unchanged app source.

### Legacy bounded Date parameters and timestamp unions

A standalone `Date` parameter uses `date()`. In a test file without `unionOf`,
exactly resolved `string | number | Date`, in any arm order or through alias chains/imports,
uses `dateUnion()`. If a sibling optional/timezone argument uses `unionOf`, the
timestamp argument must also use complete `unionOf` / `unionCase` cases. Both are HAP
APIs, not additions to the host Panda runner. Draw directly in `property`, or bind
a `const` arbitrary. Named PandaCheck import aliases are resolved by their local
binding; unrelated local variables named `date` do not shadow a factory elsewhere.
The typed callback parameter must reach the
matching SUT argument unchanged in a direct call. Multiple Date-bearing arguments
need distinct callback draws. Casts, conversion helpers, overwritten arguments,
unused draws, custom union generators, and non-shrinking `oneof` wrappers do not count.

`date()` returns fresh valid Dates with inclusive integer-millisecond bounds:
1900-01-01 UTC through 2100-01-01 UTC by default (`-2208988800000` through
`4102444800000`). `date({ min, max })` accepts ordered finite integer bounds within
TimeClip (`±8640000000000000` milliseconds). Invalid Date is excluded. It samples
bounds and a zero-clamped origin, shrinks toward that origin, and prints `Date(ms)`.
`dateUnion()` draws unrestricted `string()`, `integer()`, or the default `date()`;
shrinking dispatches by the sampled type and never changes its arm. Two- and
three-argument properties delegate to each component arbitrary's `show`, so a
Date counterexample remains `Date(ms)` rather than timezone-local Date text.

For close-out coverage, configured `date()` bounds must be inline numeric literals
and satisfy `min < max`; opaque options and singleton domains do not count. The
runtime API still permits singleton intervals for other uses. Destructured or
parameter-bound factory shadows, bracket setters and uncalled nested SUT closures
cannot establish coverage.

These legacy helpers do not add other Date shapes. Compositional cases above
handle `string | Date` and resolved nullish Date unions. Date-valued object fields
and their array components use the object-literal protocol. Root Date-array/defaulted
inputs, plain static Date fields and opaque construction retain their separate limits. Date getters/setters use the accessor protocol
above; a real SUT backing field is not arbitrary bag construction.
Imported/chained Date aliases and generic Date defaults are visible metadata:
the corresponding Date coverage or unsupported-underlying-domain gate applies.
These remaining Date limitations do not promote additional rows; the current
checklist after default completion is **28/0/6**.

### String-literal fields in interface bags

A flat interface in `--scope` may have required or optional closed string-literal
union fields, inline (including wrapped lines) or through resolvable local/imported alias chains. For
`type Status = 'pending' | 'running'` and `status?: Status`, bind
`const statusArb = optional(constantFrom<Status>('pending', 'running'));`.
For a required field, omit `optional`. Include every literal and no extra values.

Draw `statusArb.generate(rng, size)` directly in the interface literal returned by
`recordOf((rng: Random, size: number): Input => ({ ... }), showInput, shrinkInput)`.
Use that same arbitrary's `shrink(value.status)` and replace one field per candidate,
keeping every other field. Push each candidate into the array returned by the named
shrink function; discarded samples or pushes to another array do not count.
Bind the bag arbitrary with `const`, pass it to `property`, and pass its typed
callback parameter unchanged to the SUT. Each SUT parameter of that bag type needs
a distinct drawn callback parameter; the same arbitrary may be reused to draw them.
`oneof(..., constant(undefined))` is not this field draw: it does not shrink.
Casts, conversion helpers, unrestricted `string()`, missing literals, fixed-only
fields, unused generators, and overwritten inputs are rejected.

The object-literal protocol above also covers literal-compatible data classes,
nested/array literal bags and escaped literal syntax. Opaque recursive shapes
and unavailable union arms still retire. The current checklist is **28 supported /
0 partial / 6 not supported**.

## When a tree is a HAP

A directory is a **HAP root** when it contains `oh-package.json5` and
`build-profile.json5` (here or in a parent). The project-local `hvigorw`
wrapper is optional: the toolchain is whichever `hvigorw` resolves first
(project wrapper, `PATH`, or a DevEco install). `hvigorw` only on `PATH` does
not make the current directory a HAP root.

An OpenHarmony component `--scope` that ends in `.ets` is ArkTS even when there
is no HAP. Do not retire for a missing hvigor root. Import `--func` from that
file and rebuild only with the user `--build-cmd` (host `es2panda`/`ark`, or
`ohos_arkts_unittest`). Do not copy the SUT into another HAP `ohosTest/ets/sut/`.
`--lang` is the write-up language (`zh` / `en`), not C++ vs ArkTS.

## Before you run (prerequisites)

Follow this list **before** `fmbot-pbt build-run`. If a step is skipped, the gate
must fail naming that step (`BUILD FAILED — PBT not started`). fmbot-pbt does not
invent a HAP, rewrite `compileSdkVersion` / `runtimeOS`, or create `ohosTest`.

1. A DevEco SDK **this user can write**. `hvigor` may refuse `/opt/devecostudio/sdk`
   when it is not writable. Point `local.properties` `sdk.dir` (and `DEVECO_SDK_HOME`)
   at a user-owned SDK that actually contains the API you pin (for example `26.0.0`).
2. A HAP or HAR root: `oh-package.json5` + `build-profile.json5`. `--repo` and
   `--workdir` are the **same git** checkout.
3. The project's `compileSdkVersion` / `compatibleSdkVersion` / `runtimeOS` /
   `hvigor` `modelVersion` **match** that SDK. Align by hand. An extra SDK, or
   raising the project's numbers, is operator work.
4. The module you will test has `src/ohosTest/module.json5`. The sample
   `module=entry@ohosTest` assumes that folder exists.
5. `ohpm install` in `--workdir` exits 0. `@ohos/hypium` must resolve. Drop or
   repin registry-404 packages (for example a dead `@ohos/hamock` version).
6. `--build-cmd` **already exits 0** in `--workdir`:
   - HAP (`hapTasks` / `type: hap`): `hvigorw assembleHap --mode module -p module=<name>@ohosTest -p product=default --no-daemon`
   - HAR (`harTasks` / `type: har`): `hvigorw genOnDeviceTestHap --mode module -p module=<name>@ohosTest -p product=default --no-daemon`
   `assembleHap` is not a task on a HAR-only project (hvigor 00306054).
7. Then pass `--scope path/to/file.ets --func ABC` with `--func` **after** `--scope`.
   `--lang` is `zh` / `en` only (the write-up language, not C++ vs ArkTS).

Device `aa test` still needs `hdc` and a signable HAP after the suite is written.
A GN `*_abc` target is not this runner.

`build-run` on a HAP root sets `PBT_HAP=1` for that process. Do not export
`PBT_HAP` in a shell profile: a leftover `1` forces HAP mode on later C++
campaigns. To force HAP mode on a free-form `fmbot-pbt` / `-p` run whose tree is
ArkTS but missing the manifests, set `PBT_HAP=1` **on that command only**.

## How to run it

`--repo` is the module under test (scan tree, `--scope`, default `pbt-out/`).
`--workdir` is where `--build-cmd` runs. They must be the **same git
checkout**. `--workdir` is often the HAP project; `--repo` may be a parent
component that contains both the `.ets` / `.ts` SUT and that HAP.

When both paths are the HAP itself:

```bash
fmbot-pbt build-run \
  --build-cmd "hvigorw assembleHap --mode module -p module=entry@ohosTest -p product=default --no-daemon" \
  --repo /path/to/hap \
  --workdir /path/to/hap \
  --scope entry/src/main/ets/common/Utils.ets \
  --func deepCopyInter \
  --lang en
```

HAR module (library, `harTasks`) — not `assembleHap`:

```bash
fmbot-pbt build-run \
  --build-cmd "hvigorw genOnDeviceTestHap --mode module -p module=<har>@ohosTest -p product=default --no-daemon" \
  --repo /path/to/component \
  --workdir /path/to/component/.../the-har-project \
  --scope path/from/repo/to/file.ets \
  --func myFunc \
  --lang en
```

When the SUT file lives outside the HAP directory but in the same checkout:

```bash
fmbot-pbt build-run \
  --build-cmd "hvigorw assembleHap --no-daemon" \
  --repo /path/to/component \
  --workdir /path/to/component/.../the-hap \
  --out /path/to/component/pbt-out-myfunc \
  --scope path/from/repo/to/file.ts \
  --func myFunc \
  --lang en
```

`--func` requires `--scope` and must come after it. `--out` is optional
(defaults to `<repo>/pbt-out`). A custom `--out` directory may hold
`REPORT.md`.

`--build-cmd` is operator input. If it fails, the campaign does not start
(`BUILD FAILED — PBT not started`). A no-op such as `true` must not attest
`passing` or `failing`. Do not pass `--prompt` on HAP campaigns unless you
have extra domain facts the SOP cannot know.

Toolchain, SDK alignment, PandaCheck HAR, signing, and `aa test` are the
ArkTS testing guide. fmbot-pbt does **not** silently rewrite
`compileSdkVersion`, `runtimeOS`, or the hvigor pin. Align those by hand, or
let the campaign fail loud.

## What a `passing` or `failing` property must do

The suite is `src/ohosTest/ets/test/*.test.ets`, registered in `List.test.ets`.
`report.json` `sourceFile` / `functionName` stay pinned to HARD `--scope` /
`--func`.

| Allowed | Not a test of `--scope` |
|---|---|
| A value import resolving to the scoped checkout file, or the registered integrity-checked common copy under `--isolate` | `declare class` of an image global |
| Retired, if this HAP cannot import `--scope` as it stands | `import type` only, or a comment that mentions the path |
| For `| null`, `| undefined`, or an optional parameter: include every case with `unionOf` / `unionCase`. Legacy `oneof` + `constant(null)` / `constant(undefined)` only in a test file without `unionOf`. Pass unchanged, with no `as`, `!`, or `??` | Skipping the arm, or narrowing it with `as` / `ESObject` |
| For an `enum` declared in `--scope`: `constantFrom` of every member, callback typed as that enum, no `as` | `integer()` / `string()` cast to the enum, or a member list that omits one |
| For `string \| number` and other primitive unions: complete typed `unionOf` / `unionCase`; legacy `oneof` of every arm only in a test file without `unionOf`, no `as` | Drawing only one arm, or casting to one arm |
| If the scoped function body `throw`s or names `BusinessError`, or calls a same-file helper that does: call it inside `arkThrew` or `try`/`catch` and return a boolean | Happy path only, or an exception that escapes `check()` |
| Generics: matching explicit arguments or complete declared defaults, with their actual underlying recipes and bounds. Fresh genuine receivers, unchanged typed inputs and observed calls; no `as` | Value-based inference without complete defaults, conflicting overrides/annotations, fake receivers, unavailable types or `ESObject` / `any` / `@Sendable` |
| Source-resolved literal-compatible objects: contextual literals or complete `recordOf(generate, show, shrink)` recipes above. Show every field, shrink one component and keep siblings. Optional fields wrap their whole component. Root bag arrays use `minLength: 1`. Classes requiring behavior use genuine constructors. | Missing/fixed/cast/mutated fields, empty/discarded shrink output, fake constructor identity, or unavailable native/SDK/Sendable/ArkUI/private/inherited leaves |
| Actual interface contracts: genuine source-owned implementing classes, complete state recipes, matching inherited methods/accessors and observed SUT use | Interface construction, test-local stand-ins, wrong/missing members, private writes, casts, prototype replacement or discarded draws |
| Concrete function parameter: actual named/arrow/imported/intrinsic callable with complete signature, or a typed closure interpreting a drawn `FunctionModel`; invoke returned closures on the real SUT result | Missing/wrong/unused callbacks, fake receivers, type-only imports, overwritten bindings, casts or an unused model |
| For overloads in `--scope`: call every complete ordered signature and arity with distinct unchanged typed draws, no `as` | Drawing only the first overload, or collecting unordered primitive arms |
| For HAR primitives and an interface field bag: legacy `oneof` of every arm only in a test file without `unionOf`; class-instance arms require real constructor cases, no `as` | Drawing only one arm |
| For a Sendable parameter (`@Sendable` class, `SendableMap` / `ISendable`, `collections.Array`): **retire** | `integer()`, or `new Box(n)` on this thread |
| For a branded SDK type (`Length`, `Resource`, a `@kit.*` / `@ohos.*` import not declared in `--scope`): **retire** | `integer()`, or `as Length` |
| For a one-dimensional branded `T[]` / `Array<T>` (`ImageItem[]`): `[]` or `constant([])` at that argument, no `as` | Constructing `ImageItem`, a non-empty branded array, or retiring `--func` only because the element is branded |
| For `Name[]` of a `number`/`string`/`boolean` bag, including SDK `FingerInfo[]`: `array(recordOf(fn, show, shrinkName), { minLength: 1 })`, or `record()` for `{ x, y }` / `{ w, h }` / LTRB. Callback typed `Name[]`. Pass that value to `--func`, no `as` | `[]`, a generator that is never passed, `array()` without `minLength: 1`, or `{ id: 0 } as FingerInfo` |
| For standalone `Date`: `date()`. For the timestamp union: complete `unionOf` / `unionCase`, or `dateUnion()` only in a test file without `unionOf`. Pass unchanged at the matching SUT argument; object fields use their verified component protocol, while root Date arrays/defaults still retire | A missing arm, a shadowed factory, singleton/opaque Date bounds, bracket mutation, or an uncalled closure |
| For class/Date/nullish composition: complete typed `unionOf` / `unionCase` recipes, genuine public constructors and unchanged draws. Retire only if an underlying required leaf cannot actually be constructed | A missing arm, a plain-object class substitute, a fake/unused recipe, or an opaque SDK stand-in |
| For an overload with an unavailable SDK/native/Sendable/inaccessible underlying domain: **retire** | Inventing a stand-in, or `as Length` on one arm |
| If `--func` calls `getStringByNameSync`: `constantFrom` of every name in that module's `string.json`, and assert `n === n` | `string()`, or a prefixed name that cannot exist |
| | A copy or hardlink under `ohosTest/` (`ets/sut/`, `scope_utils/`, a fake `--scope` tree) |
| | `@kit.ArkUI` / `@ohos.arkui*` attested as koala / `arkts_frontend` |
| | Editing `--scope` so the import compiles (`export class`, widening `private`) |
| | A Node / host `es2panda` mirror, or `hvigorw test` on Linux Previewer |
| | `testFile` outside `ohosTest/ets/test/*.test.ets` |

A directory `--scope` is a path prefix. Each property's import must be **that
property's** `sourceFile` under the directory, not any sibling.

If the HAP cannot compile the scoped file, **retire**. Do not make it importable
by changing the production file.

## What is not a HAP

| Tree | Why HAP PBT does not execute it |
|---|---|
| koala / `arkoala-arkts` / `#generated` | `@koalaui/*`. Tests there are Panda `make`, not `aa test`. |
| `state_mgmt` `utils.ts` and other unexported framework TS | The unittest HAP does not compile that file unless ohosTest can import it as it stands. Existing UTs `declare class` the image global; that is not `--scope`. |
| `test/component_test` and many `examples/` HAPs | Often pinned to an SDK this machine does not have. Gate fails. Do not retarget in fmbot-pbt. |
| OpenHarmony GN `build.sh` / `ohos_unittest` | C++ product path. A `.ets` `--scope` is not that path. |

## Evidence

`hdc shell aa test` and the full `OHOS_REPORT_RESULT` line. `strings` on
`modules.abc` shows the SUT was linked, not that a property ran. Native line
coverage is not available for ohosTest.

Do not `ohpm install fast-check`. The ArkTS framework is PandaCheck.

## Current progress

This is the HAP ohosTest path with PandaCheck instead of fast-check. Open issue: [#366](https://github.com/fermat-hkrc/FMBot-PBT/issues/366).

### Supported

Union-argument recipes below using legacy `oneof` / `arkNullish` apply only to a
test file without `unionOf`. Otherwise use the complete compositional protocol
above for every union argument; unrelated non-union signatures are unaffected.

- Detect a HAP root and set `PBT_HAP=1`. Run Hypium + PandaCheck in `src/ohosTest`.
- Gate on the operator's `hvigorw` command. Sign, install, `hdc shell aa test`.
- Public getters/setters, including Date and bounded inference, use the actual
  property on fresh real receivers with typed draws and observed results; see
  the accessor protocol above. This is not private-field or SDK-object synthesis.
- `--repo` (scan / `--scope`) separate from `--workdir` (build), same git
  checkout. Custom `--out` may hold `REPORT.md`.
- A HAP that **owns** the scoped `.ets` and can import it: execute that symbol
  and attest `passing` or `failing`.
- Honesty gates already shipped: no `ohosTest/ets/sut/` copy; no `@kit.ArkUI`
  stand-in for koala; `sourceFile` / `functionName` pinned to HARD `--scope` /
  `--func`; `declare class` is not an import; do not rewrite `--scope` so the
  import compiles; a directory `--scope` matches by path prefix and by that
  property's `sourceFile`; a nullish signature must keep that arm
  ([#523](https://github.com/fermat-hkrc/FMBot-PBT/issues/523)); an enum
  declared in `--scope` must be drawn from every member
  ([#525](https://github.com/fermat-hkrc/FMBot-PBT/issues/525)); a union of
  `string` / `number` / `boolean` must draw every arm
  ([#527](https://github.com/fermat-hkrc/FMBot-PBT/issues/527)); a function
  that throws must be caught inside the property
  ([#529](https://github.com/fermat-hkrc/FMBot-PBT/issues/529)); a generic
  originally required primitive instantiation; constraints and complete declared defaults now use the protocols above
  ([#531](https://github.com/fermat-hkrc/FMBot-PBT/issues/531)); a `struct`
  declared in `--scope` must be constructed (`record` of every field or
  `new StructName`)
  ([#535](https://github.com/fermat-hkrc/FMBot-PBT/issues/535)); a function-typed
  parameter must be a genuine matching callable; see the typed-function protocol above
  ([#537](https://github.com/fermat-hkrc/FMBot-PBT/issues/537)); all overloads,
  including methods and constructors, need complete ordered typed calls
  ([#539](https://github.com/fermat-hkrc/FMBot-PBT/issues/539)); a mixed union of those
  arms must draw every arm
  ([#541](https://github.com/fermat-hkrc/FMBot-PBT/issues/541)); a Sendable
  parameter must retire — `new Box(n)` on this thread is not Sendable
  ([#543](https://github.com/fermat-hkrc/FMBot-PBT/issues/543)); a branded SDK type
  (`Length`, `Resource`) must retire — `as Length` is not that type
  ([#545](https://github.com/fermat-hkrc/FMBot-PBT/issues/545)); a branded `T[]`
  (`ImageItem[]`) is `[]` or `constant([])`, not a constructed element
  ([#630](https://github.com/fermat-hkrc/FMBot-PBT/issues/630)); a union with an
  undrawable arm must retire — drawing only `string()` is not a union with a required class arm
  ([#547](https://github.com/fermat-hkrc/FMBot-PBT/issues/547)); an overload with an
  unavailable underlying domain must retire — supported non-primitives use their own recipes
  ([#549](https://github.com/fermat-hkrc/FMBot-PBT/issues/549)); a throw in a same-file
  callee must be caught inside the property
  ([#551](https://github.com/fermat-hkrc/FMBot-PBT/issues/551)); a successful
  HAP build may close with every property `retired`
  ([#533](https://github.com/fermat-hkrc/FMBot-PBT/issues/533)).
- If this HAP cannot import `--scope` as it stands, **retire**. `report.json`
  then validates.
- **SUT types PandaCheck can drive:** exported functions of `number` / `string` /
  `boolean` / arrays of those, plain objects you can write as `.ets` literals,
  `T | null` / `T | undefined` / optional parameters via `arkNullish`, and an
  `enum` declared in `--scope`, or imported by one relative hop, via `constantFrom` of every member, and
  `string \| number` (and the other unions of those three) via `oneof` of every
  arm, and a union of string literals, numeric literals, or both via `constantFrom` or `oneof` of every literal, and a literal mixed with an open primitive via `oneof` of `constant` and that primitive's generator, and a `[number, number]` parameter via `property2` for one pair or a distinct `numberPair` at each pair argument of the same `property()` call, and a generic instantiated as `name<string>` / `name<number>` /
  `name<boolean>`, a generic class method via `new CustomStack<string>()` then
  `stack.push(drawn)` on that instance, plus complete declared-default and bound-based
  instantiations under the protocols above, and contextual object literals or complete
  component recipes as described above. Nested objects, Date/bigint/matrix and
  fixed tuple fields keep their own construction/show/shrink evidence. Root
  bag arrays require `minLength: 1`; SDK `FingerInfo[]` keeps its established
  exception and branded `ImageItem[]` stays `[]`. Concrete function parameters use
  genuine matching callable values or generated
  `FunctionModel` closures, plus invocation of returned functions, and overloads via a complete ordered typed call per declared signature, and a
  mixed union of those arms via `oneof` of every arm. A Sendable parameter is
  not drawable: retire that `--func`. A branded SDK type (`Length`, `Resource`)
  is not drawable: retire that `--func`. A one-dimensional branded `T[]`
  (`ImageItem[]`) is drawable only as `[]` or `constant([])`
  ([#630](https://github.com/fermat-hkrc/FMBot-PBT/issues/630)). A union with an undrawable arm
  is not drawable: retire that `--func`. An overload with an unavailable
  underlying domain must retire; non-primitive alone is not a reason. A `passing`
  or `failing` property must pass that value with no `as`. Example:
  `export function add(a: number, b: number): number`.

### Not supported

Two lists. The first is the HAP runner. The second is **ArkTS in the SUT** that
`fast-check.har` cannot feed. Closing either means new fmbot-pbt work.

#### Runner / tree

| Gap | What happens now | What to improve |
|---|---|---|
| koala / `@koalaui` / `#generated` | Honest close is all `retired`; `report.json` validates. Cannot compile into ohosTest. | A koala/Panda runner, or a full OH-image path, not HAP ohosTest. |
| Unexported framework `.ts` (e.g. `state_mgmt` `utils.ts`) | ohosTest cannot import the class. Existing UTs `declare class` the image global. Honest close is all `retired`; `report.json` validates. | Either retire (current honest result) or a way to compile that file **without** editing it. |
| SDK / hvigor / `runtimeOS` mismatch | Gate fails, or the toolchain is deferred. fmbot-pbt does not rewrite manifests. | Fail loud with the expected SDK fields. Operator preflight, not a silent retarget. |
| `test/component_test` and many `examples/` HAPs | Often API 12 / HarmonyOS 5.x; this SDK 26 tree does not assemble them. | Same: fail loud. Do not rewrite `compatibleSdkVersion`. |
| OpenHarmony GN `build.sh` | Separate C++ product path. An `.ets` `--scope` is not that path. | Keep the split. Do not spawn GN for ArkTS `--scope`. |
| Native line coverage | `coverage_gaps` is not linked for ohosTest. Evidence is `aa test` plus abc strings. | ArkTS coverage if the toolchain can produce it; do not fake C++ lcov. |
| Sibling export / private-field walk | `passing` / `failing` is rejected unless the suite imports and calls `--func`. Calling a sibling is not that symbol. | Retire if ohosTest cannot bind the symbol. |
| Host `es2panda` / PandaCheck | Removed. Do not bring it back as a stand-in for HAP. | Only if a real Panda test tree is the SUT, as its own runner. |
| Kea GUI | `fmbot-pbt kea` is a different mode (black-box UI). | Keep it separate from source-level ArkTS. |

#### ArkTS SUT vs `fast-check.har`

The HAR is converted JS (`ohpm convert` of npm fast-check). It generates JS
`number` / `string` / `boolean` / `Array` / plain objects. The property file is
ArkTS (named imports, `(x: number)`, no `any`, no destructuring, `check()` not
`fc.assert`). A SUT feature is unsupported when ohosTest cannot call that
function with those values.

| SUT feature | What happens now | What to improve |
|---|---|---|
| `@Sendable` / concurrent types | `passing` / `failing` is rejected. `retired` is the honest close. `new Box(n)` on this thread is not Sendable. | A runner that can construct a Sendable value. Do not stretch HAR. |
| Unexported `class` / `private` fields | Cannot `import` or `new` without editing `--scope`. | Compile/import without rewriting the SUT, or retire. |
| Union of unsupported types | `passing` / `failing` is rejected. `retired` is the honest close. Drawing only `string()` is not a required class/native arm. | Typed `unionCase` factories for every constructible arm. A genuinely unavailable SDK/native leaf still needs its own construction/lifecycle support. |
| branded/opaque SDK types | Bare `Length` / `Resource`: `passing` / `failing` is rejected. `retired` is the honest close. `as Length` is not that type. One-dimensional branded `T[]` may be `[]` / `constant([])` ([#630](https://github.com/fermat-hkrc/FMBot-PBT/issues/630)). | Typed constructors for non-empty branded values. Do not stretch HAR. |
| Overloads with an unavailable underlying domain | Required native/SDK/Sendable/inaccessible values still force `retired`; supported classes/arrays/Date/callbacks do not. | Add the underlying construction/lifecycle protocol, not a cast or missing signature. |
| `throw` outside the scoped function body | A same-file callee, a direct import, and up to four hops must be caught or the `--func` retired. A fifth hop is still invisible. | Catch at the call the suite actually makes, or retire. |
| `getStringByNameSync` | `passing` / `failing` is rejected unless every `string.json` name is drawn. `string()` only hits the throw path. | `constantFrom` of those names. Assert finite. `NaN` is a failing result, not a successful number. |
| Generic constrained to an unsupported type | `passing` / `failing` is rejected. `retired` is the honest close. `name<string>` is not `T extends SomeStruct`. | Retire that `--func`. |
| Untyped callable/interoperability domains (`Function`, `ESObject`) | `passing` / `failing` is rejected. `retired` is the honest close. A named stub is not `Function` or `ESObject`. | Retire that `--func`. |
| `@Component` / `@Observed` / UI | Needs the UI runtime. | Not HAP fast-check. Kea or a UI runner. |
| koala / `@koalaui` | Different compiler. HAR never loads. | Panda/koala runner, not ohosTest. |
| Native / NAPI / `ESObject`-only APIs | `ESObject` compiles and leaves ArkTS typing. | Typed bindings, or retire. |

### Improve next

1. SDK preflight: name the missing `compileSdkVersion` / `runtimeOS` / hvigor
   plugin and stop, without editing the SUT tree.
2. Symbol-level import: `passing` / `failing` must bind the `--func`, not only
   a value import of the file that happens to define it.
3. Koala / Panda: a documented non-HAP runner, or keep retiring koala `--scope`.
4. **ArkTS SUT types:** the legacy union-argument recipes in this item apply only
   to test files without `unionOf`; otherwise use complete typed cases for every
   union argument. Do not stretch TS fast-check inside `.ets`. Nullish is
   `arkNullish` in the suite, not a fast-check fork. An enum declared in
   `--scope` is `constantFrom` of its members. A union of `string` / `number` /
   `boolean` is `oneof` of every arm. A scoped function that throws is caught
   with `arkThrew` or `try`/`catch`. A generic is `name<string>`,
   `name<number>`, or `name<boolean>`; complete declared defaults and actual bound-based
   implementations follow the protocols above, not value-based inference. A generic class method is `new CustomStack<string>()` then `stack.push(drawn)` on that instance. Objects use the complete contextual literal/component protocol above, not a
   primitive-only or one-level cap. Require every-field display and component-owned
   shrinking. Root bag arrays require `minLength: 1`; branded arrays and the
   existing SDK exception keep their own rules. Unavailable leaves still retire.
   Concrete function values use
   the complete typed-function protocol above, including generated callback behavior and returned closures. Overloads, including method/constructor and semicolonless forms, need a complete ordered typed call per
   signature. A mixed union of those arms is `oneof` of every arm. A Sendable
   parameter is retired ([#543](https://github.com/fermat-hkrc/FMBot-PBT/issues/543)).
   A branded SDK type (`Length`, `Resource`) is retired ([#545](https://github.com/fermat-hkrc/FMBot-PBT/issues/545)).
   A branded `T[]` (`ImageItem[]`) is `[]` or `constant([])` ([#630](https://github.com/fermat-hkrc/FMBot-PBT/issues/630)).
   A union with an undrawable arm is retired ([#547](https://github.com/fermat-hkrc/FMBot-PBT/issues/547)).
   An overload with an unavailable underlying domain is retired ([#549](https://github.com/fermat-hkrc/FMBot-PBT/issues/549)); supported non-primitives are no longer blanket-retired.
   A throw in a same-file callee is caught ([#551](https://github.com/fermat-hkrc/FMBot-PBT/issues/551)).
   A generic constrained to an unsupported type is retired. `name<string>` is not `T extends SomeStruct`.
   A callback typed `Function` or `ESObject` is retired. A named stub is only legal for `(...) =>`.
   A passing or failing property must import and call `--func`. Importing the file and calling a sibling is not that symbol.
   An enum imported by one relative hop is `constantFrom` of every member in that file. A second hop is still invisible.
   A union of string literals is `constantFrom` of every literal. `string()` is not that union.
   A union of numeric literals is `constantFrom` of every literal. `integer()` is not that union.
   A union of string and numeric literals is `oneof` of `constant` or `constantFrom` for every literal. `string()` or `integer()` is not that union.
   One `[number, number]` parameter is `property2(integer(), integer(), (x: number, y: number) => func([x, y]))`. Each further pair is a distinct `numberPair(integer(), integer())` in the same `property()` call. `tuple()` returns a `Pair`, not that type. Any other tuple is retired.
   A literal mixed with an open primitive (`'xs' | number`, `0 | string`) is `oneof` of `constant` or `constantFrom` and that primitive's generator. `string()` alone is not that union.
   A throw four hops away is caught or retired. A fifth hop is still invisible.
   Missing `hvigorw` / SDK already defers the build gate. A mismatch is named and does not rewrite the tree.
   Long term: an ArkTS PBT library with the same `check`/`property` shape, no
   `any`, no `<const T>`. Scan the scoped signature and only allow arbitraries
   that match it.

## Related

- [Installation: HAP / ArkTS apps](../subcommands.md#build-run)
  ([中文](../subcommands.zh.md#build-run))
- [Chinese ArkTS page](arkts.zh.md)
- [Subcommands](../subcommands.md) — `build-run --build-cmd "hvigorw assembleHap --mode module -p module=entry@ohosTest -p product=default --no-daemon"`
- [Kea GUI testing](../kea.md) — black-box UI on a phone, not source-level ArkTS
- The bundled ArkTS testing guide: DevEco, SDK alignment, PandaCheck HAR, sign, and `aa test`
