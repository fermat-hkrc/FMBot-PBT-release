# ArkTS PBT (PandaCheck)

[中文版 / Chinese version](arkts.zh.md)

[Back to installation](../subcommands.md#build-run) ·
[Subcommands](../subcommands.md)

fmbot-pbt supports ArkTS as **HarmonyOS HAP ohosTest**. A property is Hypium +
PandaCheck in `src/ohosTest/ets/test/<Name>Pbt.test.ets`
(`import { check, property, integer, string, boolean, bigint, dictionary, primitiveUnion, date, dateUnion, array, recursive, functionModel, FunctionModel, unionCase, unionOf } from 'PandaCheck'`),
assembled with `hvigorw`, and run with `hdc shell aa test`. The verdict is
`OHOS_REPORT_RESULT`. Install the library with `make-pandacheck-har.sh` into
`libs/PandaCheck.har`. Depend on `"PandaCheck": "file:../libs/PandaCheck.har"` from the module `oh-package.json5`. Do not copy `Index.ets` into `libs/`. Do not `ohpm install fast-check`.

This is not Kea. This is not OpenHarmony GN C++. Host `es2panda`/`ark`
(`PBT_PANDA=1`, `text()`/`bool()`, `PANDACHECK_RESULT`) is a **separate** runner
for compiler `.ets` that is not a HAP. Do not mix the two.

`--isolate` (with `--scope file.ets --func Name`) builds a **mini HAP** that
places the real `--scope` file and its relative imports at `entry/src/main/ets/`.
That file is a hard link (same inode as the checkout) when the filesystem allows it
and the placed text matches the checkout. An unused `import { Name } from './Name'` is
stripped on a regular-file copy so the compiler does not need that module; the function
body is unchanged and the checkout is not edited. A cross-device fallback is also a
regular file at that path, not a symlink and not a paste into `ohosTest/ets/sut/`. Use it to test a function in a large app (Photos) without
compiling the APK. The import graph must be public kits (`@ohos.hilog`, `@ohos.i18n`, …).
System APIs (`systemParameterEnhance`, `photoAccessHelper`, …) or a graph bigger than a
function **STOP**. `--isolate` does not edit HARD `--scope`. The campaign cwd is `<out>/isolate-hap`.
Import that file under `entry/src/main/ets` (the prompt names the specifier). Do not replace it
with a symlink. A copy under `entry/src/common` or `ohosTest` is not `--scope`. `--build-cmd` is
replaced by `assembleHap … entry@ohosTest` after `ohpm install` in that HAP.
Isolate STOP and a failed `ohpm install` exit 3, same as a failed build gate.

## Feature inventory

Scored from the OpenHarmony tutorial
[Introduction to ArkTS](https://github.com/openharmony/docs/blob/f41b9345badd47c7ab0c263344cd7f4b5a549afb/en/application-dev/quick-start/introduction-to-arkts.md)
at `f41b9345badd47c7ab0c263344cd7f4b5a549afb`. This is that tutorial's named
types, call shapes, and modules. It is not a percent of ArkTS.

`let`, `const`, inference, `void`, operators, `if` / `switch` / loops, and
`this` are syntax. They are not in the count. A property does not draw them.

**23 supported, 5 partial, 6 not supported** on this 34-row checklist.
Supported means a passing or failing property can use that form, and a
close-out gate requires it. Partial means only some forms of that tutorial
heading are drawn or caught. Not supported means we do not draw it, or the
honest close is `retired`.

| Tutorial feature | fmbot-pbt | What that means |
| --- | --- | --- |
| `number` | supported | `integer()` / `double()`. A comparison with a fractional threshold uses `double()`; `integer()` does not cross it. |
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
| class or `struct` with fields | supported | `new` or `record` of every field, when declared in `--scope`. |
| `static` field or method | supported | A public static method with a `number`, `string`, or `boolean` parameter must be called as `Class.method(drawn)`. A public static field of those types, including an unannotated `= 0`, `= 'albumType'`, or `= true`, is `Class.field = drawn` and a read of `Class.field` in that property callback. A readonly field is read. A private or protected field, a non-exact type, and a field of `Date` stay outside. |
| getter / setter | supported | Tutorial `Person4.age`, getter-only and setter-only forms, plus Date and supported underlying domains. Actual public instance/static ownership, fresh receivers, typed constructor/setter draws, same-receiver observed reads, bounded getter inference and per-operation catches are checked. Private state, substitutes and unavailable underlying domains do not count. |
| inheritance, `super`, abstract class | not supported | Not constructed. |
| `private` / `protected` field | not supported | The suite must not edit `--scope` to expose it. |
| object literal | partial | `record()` is `{ w, h }`, `{ x, y }`, or `{ left, top, right, bottom }`. Any other `number`/`string`/`boolean` bag uses `recordOf(fn, show, shrinkName)`. `shrinkName` changes one field per candidate from that field's `shrink` and keeps the others. An optional field is `optional(integer())`, `optional(string())`, or `optional(boolean())`. Flat interface fields may also be closed string-literal unions, inline (including wrapped lines) or resolvable local/imported alias chains: `constantFrom<Alias>(every literal)`, wrapped in `optional()` for optional fields. A `[number, number]` field is `numberPair`. An optional pair is `optional(numberPair(x, y))`, not an always-present literal. A field that is another in-scope interface `{ x, y }`, `{ w, h }`, LTRB, or primitive bag is `record().generate` or `recordOf().generate` with `as Name` on that result, one level only. A `number[]`, `string[]`, or `boolean[]` field is `array().generate`. An optional array field is `optional(array(integer())).generate`. `[]` is not that field. `number[][]`, a class, ArkUI `struct`, `@Sendable` class, `Date`, or other tuple stays out. |
| `Record<K, V>` | partial | Tutorial form `Record<string, number>` is `dictionary(string(), integer())`. An extra draw, not this row: `Record<string, number \| string \| boolean>` is `dictionary(string(), primitiveUnion())`. The value argument must be `primitiveUnion()` itself. `oneof(primitiveUnion())` does not shrink. Default 4 keys. `{ maxLength }` sets that cap, including downward. Shrink drops keys, then values, then keys (skip a rename that collides). The callback parameter is that Record. `{}` and `record()` are not that map. `Record<number, number>`, any other value type, a same-file alias of those, `Record \| null`, optional Record, and a defaulted Record stay out. |
| interface | partial | A field bag in `--scope` can be constructed. A method-only contract is not a value. |
| generic function | supported | The call must be `name<string>`, `name<number>`, or `name<boolean>`. An inferred call is rejected. |
| generic constraint | partial | A constraint of those three is drawn. Any other constraint is `retired`. |
| generic class | supported | `new CustomStack<string>()`, `<number>`, or `<boolean>`, then call that instance's method with a drawn value of that type. An inferred `new CustomStack()` is rejected. A constraint other than those three is retired. |
| generic default | partial | Alias/default dependencies are resolved in metadata, including class defaults. Existing explicit primitive generic instantiations and underlying-domain gates still apply; a Date/class default does not supply its own generator. |
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
compiled SUT remain authoritative. Aliases cannot hide Date fields/arrays/default
parameters, inaccessible constructors, inheritance, Sendable or unavailable SDK
leaves. Class defaults do not replace the explicit generic-instantiation protocol.
This promotion changes only the alias row, not every underlying partial row.

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

Function support does not promote unrelated domains. With overload and accessor completion below,
the checklist is **23 supported / 5 partial / 6 not supported**.
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
and unavailable underlying types still retire. This does not enable Date-valued
bag-field generators, Date-array/defaulted inputs, plain static Date fields or
unsupported callback compositions. Original bindings are proved before host-only
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
run. Only the getter/setter row is newly promoted: **23/5/6**, still 34 rows.

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
must actually be invoked. Explicit generic instantiations, optional/rest/default
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
A valid composition does not bypass retirement of Date fields, arrays or
defaulted Date parameters elsewhere in the signature.

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
handle `string | Date` and resolved nullish Date unions. Date-valued bag fields,
Date-array/defaulted inputs, plain static Date fields and opaque construction
retain their separate limits. Date getters/setters use the accessor protocol
above; a real SUT backing field is not arbitrary bag construction.
Imported/chained Date aliases and generic Date defaults are visible metadata:
the corresponding Date coverage or unsupported-underlying-domain gate applies.
These remaining Date limitations do not promote additional rows; the current
checklist after accessor completion is **23/5/6**.

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

This does not add classes, nested/array literal bags, recursive literal-field
shapes, escaped literal syntax, or unions with undrawable arms. Imported/chained
literal aliases are normalized without lifting those field restrictions. The
checklist is **23 supported / 5 partial / 6 not supported** after accessor completion.

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
| A value import that resolves to the scoped checkout file | `declare class` of an image global |
| Retired, if this HAP cannot import `--scope` as it stands | `import type` only, or a comment that mentions the path |
| For `| null`, `| undefined`, or an optional parameter: include every case with `unionOf` / `unionCase`. Legacy `oneof` + `constant(null)` / `constant(undefined)` only in a test file without `unionOf`. Pass unchanged, with no `as`, `!`, or `??` | Skipping the arm, or narrowing it with `as` / `ESObject` |
| For an `enum` declared in `--scope`: `constantFrom` of every member, callback typed as that enum, no `as` | `integer()` / `string()` cast to the enum, or a member list that omits one |
| For `string \| number` and other primitive unions: complete typed `unionOf` / `unionCase`; legacy `oneof` of every arm only in a test file without `unionOf`, no `as` | Drawing only one arm, or casting to one arm |
| If the scoped function body `throw`s or names `BusinessError`, or calls a same-file helper that does: call it inside `arkThrew` or `try`/`catch` and return a boolean | Happy path only, or an exception that escapes `check()` |
| A generic: call `name<string>`, `name<number>`, or `name<boolean>` (or a union of those), no `as`. A generic class method is `new CustomStack<string>()` then `stack.push(drawn)` on that instance | An inferred `name(x)` or `new CustomStack()`, `xs.push` on another value, or `name<ESObject>` / `any` / `@Sendable` |
| For a `struct` declared in `--scope`: `record({ w, h })`, `record({ x, y })`, LTRB `record`, or `recordOf(fn, show, shrinkName)` for any other `number`/`string`/`boolean` bag (or `new StructName`). Flat interface string-literal union fields (inline or resolvable alias chains/imports) use every literal in `constantFrom`, with `optional()` for optional fields. `shrinkName` pushes one-field candidates into its returned array. An optional field uses `optional()`. A nested interface field is `record().generate` or `recordOf().generate` with `as Name` on that result. A `number[]`, `string[]`, or `boolean[]` field is `array().generate`. Callback typed as that struct, no other `as` | `as Rect` / `ESObject` on a plain object, a bag that omits a field, a bare draw of an optional field, `optional()` inside the inner bag standing in for an optional nested field, a class / ArkUI `struct` / `@Sendable` field drawn as a nested bag, `[]` for a `number[]` field, or `recordOf` on a tuple other than `[number, number]` / `number[][]` / unsupported field type (closed string-literal fields are supported as described above) |
| Concrete function parameter: actual named/arrow/imported/intrinsic callable with complete signature, or a typed closure interpreting a drawn `FunctionModel`; invoke returned closures on the real SUT result | Missing/wrong/unused callbacks, fake receivers, type-only imports, overwritten bindings, casts or an unused model |
| For overloads in `--scope`: call every complete ordered signature and arity with distinct unchanged typed draws, no `as` | Drawing only the first overload, or collecting unordered primitive arms |
| For HAR primitives and an interface field bag: legacy `oneof` of every arm only in a test file without `unionOf`; class-instance arms require real constructor cases, no `as` | Drawing only one arm |
| For a Sendable parameter (`@Sendable` class, `SendableMap` / `ISendable`, `collections.Array`): **retire** | `integer()`, or `new Box(n)` on this thread |
| For a branded SDK type (`Length`, `Resource`, a `@kit.*` / `@ohos.*` import not declared in `--scope`): **retire** | `integer()`, or `as Length` |
| For a one-dimensional branded `T[]` / `Array<T>` (`ImageItem[]`): `[]` or `constant([])` at that argument, no `as` | Constructing `ImageItem`, a non-empty branded array, or retiring `--func` only because the element is branded |
| For `Name[]` of a `number`/`string`/`boolean` bag, including SDK `FingerInfo[]`: `array(recordOf(fn, show, shrinkName), { minLength: 1 })`, or `record()` for `{ x, y }` / `{ w, h }` / LTRB. Callback typed `Name[]`. Pass that value to `--func`, no `as` | `[]`, a generator that is never passed, `array()` without `minLength: 1`, or `{ id: 0 } as FingerInfo` |
| For standalone `Date`: `date()`. For the timestamp union: complete `unionOf` / `unionCase`, or `dateUnion()` only in a test file without `unionOf`. Pass unchanged at the matching SUT argument; Date fields/arrays/defaults still retire | A missing arm, a shadowed factory, singleton/opaque Date bounds, bracket mutation, or an uncalled closure |
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
  must be instantiated at `string` / `number` / `boolean`
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
  `name<boolean>`, a generic class method via `new CustomStack<string>()` then `stack.push(drawn)` on that instance, and a `struct` declared in `--scope` via `record({ w, h })`, `record({ x, y })`, LTRB `record`, `recordOf(fn, show, shrinkName)` for any other `number`/`string`/`boolean` bag, `optional()` for an optional field, or `new StructName`, a nested interface field via `record().generate` or `recordOf().generate` with `as Name` on that result (a class, ArkUI `struct`, or `@Sendable` field stays out), a `number[]` / `string[]` / `boolean[]` field via `array().generate` (`number[][]` stays out), a `Name[]` of a primitive-field bag including `FingerInfo[]` via `array(recordOf(fn, show, shrinkName), { minLength: 1 })` passed to `--func` (`ImageItem[]` stays `[]`; a bag with a nested field is not that array), and concrete function parameters via genuine matching callable values or generated
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
   `name<number>`, or `name<boolean>`. A generic class method is `new CustomStack<string>()` then `stack.push(drawn)` on that instance. A struct declared in `--scope` is `record({ w, h })`, `record({ x, y })`, LTRB `record`, `recordOf(fn, show, shrinkName)` for any other `number`/`string`/`boolean` bag, or `new StructName`. `shrinkName` changes one field per candidate. An optional field uses `optional()`. A nested interface field is `record().generate` or `recordOf().generate` with `as Name` on that result. A class, ArkUI `struct`, or `@Sendable` field stays out. A `number[]`, `string[]`, or `boolean[]` field is `array().generate`. `number[][]` stays out. A `Name[]` of a primitive-field bag, including `FingerInfo[]`, is `array(recordOf(fn, show, shrinkName), { minLength: 1 })` passed to `--func`. `ImageItem[]` stays `[]`. A bag with a nested field is not that array. A `[number, number]` field is `numberPair`. An optional pair is `optional(numberPair(x, y))`. Flat interface fields that are closed string-literal unions, inline or resolvable alias chains/imports, use `constantFrom<Alias>(every literal)` and `optional()` for optional fields. Shrink candidates must be pushed to the returned array, one field changed at a time. Other tuples and unsupported field types are not `recordOf`. Concrete function values use
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
