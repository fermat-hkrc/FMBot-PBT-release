# ArkTS PBT (PandaCheck)

[中文版 / Chinese version](arkts.zh.md)

[Back to installation](../installation.md#harmonyos-hap--arkts-apps) ·
[Subcommands](../subcommands.md)

pi-pbt supports ArkTS as **HarmonyOS HAP ohosTest**. A property is Hypium +
PandaCheck in `src/ohosTest/ets/test/<Name>Pbt.test.ets`
(`import { check, property, integer, string, boolean } from 'PandaCheck'`),
assembled with `hvigorw`, and run with `hdc shell aa test`. The verdict is
`OHOS_REPORT_RESULT`. Install the library with `make-pandacheck-har.sh` into
`libs/PandaCheck`. Do not `ohpm install fast-check`.

This is not Kea. This is not OpenHarmony GN C++. Host `es2panda`/`ark`
(`PBT_PANDA=1`, `text()`/`bool()`, `PANDACHECK_RESULT`) is a **separate** runner
for compiler `.ets` that is not a HAP. Do not mix the two.

## Feature inventory

Scored from the OpenHarmony tutorial
[Introduction to ArkTS](https://github.com/openharmony/docs/blob/f41b9345badd47c7ab0c263344cd7f4b5a549afb/en/application-dev/quick-start/introduction-to-arkts.md)
at `f41b9345badd47c7ab0c263344cd7f4b5a549afb`. This is that tutorial's named
types, call shapes, and modules. It is not a percent of ArkTS.

`let`, `const`, inference, `void`, operators, `if` / `switch` / loops, and
`this` are syntax. They are not in the count. A property does not draw them.

**14 supported, 9 partial, 11 not supported** on this 34-row checklist.
Supported means a passing or failing property can use that form, and a
close-out gate requires it. Partial means only some forms of that tutorial
heading are drawn or caught. Not supported means we do not draw it, or the
honest close is `retired`.

| Tutorial feature | pi-pbt | What that means |
| --- | --- | --- |
| `number` | supported | `integer()` / `double()`. |
| `bigint` | not supported | No generator. |
| `boolean` | supported | `boolean()`. |
| `string` | supported | `string()`. |
| `Object` | not supported | Not a field bag we can construct. |
| array of `number` / `string` / `boolean` | supported | `array(integer())`, `array(string())`, or `array(boolean())`. A bare element is rejected. |
| `enum` | supported | `constantFrom` of every member in the same file, or one relative import. A second hop is invisible. |
| union | partial | Primitive, string-literal, numeric-literal, mixed-literal, and literal-plus-primitive unions are drawn. An undrawable arm is `retired`. |
| type alias | partial | A drawable alias such as `type Mode = 'xs' \| 'sm'` is expanded and the existing gate applies. An alias of `Date` or a class is not expanded. |
| optional parameter | supported | `oneof` with `constant(undefined)`. |
| default parameter | supported | A `number`, `string`, or `boolean` default must be both omitted and passed as a drawn value. |
| rest parameter | supported | `func(...xs)` where `xs` is `array()` of `number`, `string`, or `boolean`. A bare list is rejected. |
| function type | partial | A real `(...) =>` is a named function or an arrow. `Function` and `ESObject` are `retired`. |
| overload | partial | Every primitive arm, or every arity, must be called. A non-primitive arm is `retired`. |
| class or `struct` with fields | supported | `new` or `record` of every field, when declared in `--scope`. |
| `static` field or method | partial | A public static method with a `number`, `string`, or `boolean` parameter must be called as `Class.method(drawn)`. A static field stays outside. |
| getter / setter | not supported | Not a drawn value. |
| inheritance, `super`, abstract class | not supported | Not constructed. |
| `private` / `protected` field | not supported | The suite must not edit `--scope` to expose it. |
| object literal | partial | Only a known field bag. `record()` itself is `{w, h}`. |
| `Record<K, V>` | not supported | Not a generic map. |
| interface | partial | A field bag in `--scope` can be constructed. A method-only contract is not a value. |
| generic function | supported | The call must be `name<string>`, `name<number>`, or `name<boolean>`. An inferred call is rejected. |
| generic constraint | partial | A constraint of those three is drawn. Any other constraint is `retired`. |
| generic class | not supported | `CustomStack<string>` is not constructed. |
| generic default | not supported | A defaulted type parameter is not expanded. |
| nullish `T` or `null` / `undefined` | supported | `arkNullish`. The call must not use `!` or `??` to drop the arm. |
| named import of `--func` | supported | A passing or failing property must import the file and call that symbol. |
| import alias or namespace | supported | `id as ident` and `ids.id(...)` count. A sibling call does not. |
| default export | supported | A default-exported function must be imported as the default and called. A class or value default is not this binding. |
| dynamic `import()` | not supported | The result is `ESObject`. |
| `throw` / `try` | partial | A throw in `--func`, a same-file callee, a direct import, or up to three hops must be caught. A fourth hop is invisible. |
| `@interface` annotation | not supported | Not a value. |
| ArkUI | not supported | Declarative UI is not this runner. |

`@Sendable` is in the kit overview, not in this tutorial. A Sendable parameter
is `retired`. Do not add it to this count.

## When a tree is a HAP

A directory is a **HAP root** when it contains `oh-package.json5` and
`build-profile.json5` (here or in a parent). The project-local `hvigorw`
wrapper is optional: the toolchain is whichever `hvigorw` resolves first
(project wrapper, `PATH`, or a DevEco install). `hvigorw` only on `PATH` does
not make the current directory a HAP root.

`build-run` on a HAP root sets `PBT_HAP=1` for that process. Do not export
`PBT_HAP` in a shell profile: a leftover `1` forces HAP mode on later C++
campaigns. To force HAP mode on a free-form `pi-pbt` / `-p` run whose tree is
ArkTS but missing the manifests, set `PBT_HAP=1` **on that command only**.

## How to run it

`--repo` is the module under test (scan tree, `--scope`, default `pbt-out/`).
`--workdir` is where `--build-cmd` runs. They must be the **same git
checkout**. `--workdir` is often the HAP project; `--repo` may be a parent
component that contains both the `.ets` / `.ts` SUT and that HAP.

When both paths are the HAP itself:

```bash
pi-pbt build-run \
  --build-cmd "hvigorw assembleHap --mode module -p module=entry@ohosTest -p product=default --no-daemon" \
  --repo /path/to/hap \
  --workdir /path/to/hap \
  --scope entry/src/main/ets/common/Utils.ets \
  --func deepCopyInter \
  --lang en
```

When the SUT file lives outside the HAP directory but in the same checkout:

```bash
pi-pbt build-run \
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
`arkts-build-run` cards. pi-pbt does **not** silently rewrite
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
| For `| null`, `| undefined`, or an optional parameter: draw that arm (`oneof` + `constant(null)` / `constant(undefined)`) and pass the parameter with no `as`, `!`, or `??` | Skipping the arm, or narrowing it with `as` / `ESObject` |
| For an `enum` declared in `--scope`: `constantFrom` of every member, callback typed as that enum, no `as` | `integer()` / `string()` cast to the enum, or a member list that omits one |
| For `string \| number` and the other unions of `string` / `number` / `boolean`: `oneof` of every arm, no `as` | Drawing only one arm, or casting to one arm |
| If the scoped function body `throw`s or names `BusinessError`, or calls a same-file helper that does: call it inside `arkThrew` or `try`/`catch` and return a boolean | Happy path only, or an exception that escapes `check()` |
| A generic: call `name<string>`, `name<number>`, or `name<boolean>` (or a union of those), no `as` | An inferred `name(x)`, or `name<ESObject>` / `any` / `@Sendable` |
| For a `struct` declared in `--scope`: `record` of every field (or `new StructName`), callback typed as that struct, no `as` | `as Rect` / `ESObject` on a plain object, or a `record` that omits a field |
| For a function-typed parameter: pass a named `.ets` function or an arrow, no `as` | Skipping the callback, or `as ESObject` / `as (x) =>` |
| For overloads declared with `;` in `--scope`: call every signature (each primitive arm or each arity), no `as` | Drawing only the first overload |
| For a union of HAR primitives and a named field bag in `--scope`: `oneof` of every arm, no `as` | Drawing only one arm |
| For a Sendable parameter (`@Sendable` class, `SendableMap` / `ISendable`, `collections.Array`): **retire** | `integer()`, or `new Box(n)` on this thread |
| For a branded SDK type (`Length`, `Resource`, a `@kit.*` / `@ohos.*` import not declared in `--scope`): **retire** | `integer()`, or `as Length` |
| For a union with an undrawable arm (`string | Date`): **retire** | Drawing only the drawable arm |
| For overloads whose arms are not HAR primitives and not distinct arities: **retire** | One JS call, or `as Length` on one arm |
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
| `test/component_test` and many `examples/` HAPs | Often pinned to an SDK this machine does not have. Gate fails. Do not retarget in pi-pbt. |
| OpenHarmony GN `build.sh` / `ohos_unittest` | C++ product path. A `.ets` `--scope` is not that path. |

## Evidence

`hdc shell aa test` and the full `OHOS_REPORT_RESULT` line. `strings` on
`modules.abc` shows the SUT was linked, not that a property ran. Native line
coverage is not available for ohosTest.

Do not `ohpm install fast-check`. The ArkTS framework is PandaCheck.

## Current progress

This is the HAP ohosTest path with PandaCheck instead of fast-check. Open issue: [#366](https://github.com/fermat-hkrc/FMBot-PBT/issues/366).

### Supported

- Detect a HAP root and set `PBT_HAP=1`. Run Hypium + PandaCheck in `src/ohosTest`.
- Gate on the operator's `hvigorw` command. Sign, install, `hdc shell aa test`.
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
  parameter must be a named `.ets` function or an arrow
  ([#537](https://github.com/fermat-hkrc/FMBot-PBT/issues/537)); overloads declared
  with `;` must all be called
  ([#539](https://github.com/fermat-hkrc/FMBot-PBT/issues/539)); a mixed union of those
  arms must draw every arm
  ([#541](https://github.com/fermat-hkrc/FMBot-PBT/issues/541)); a Sendable
  parameter must retire — `new Box(n)` on this thread is not Sendable
  ([#543](https://github.com/fermat-hkrc/FMBot-PBT/issues/543)); a branded SDK type
  (`Length`, `Resource`) must retire — `as Length` is not that type
  ([#545](https://github.com/fermat-hkrc/FMBot-PBT/issues/545)); a union with an
  undrawable arm must retire — drawing only `string()` is not `string | Date`
  ([#547](https://github.com/fermat-hkrc/FMBot-PBT/issues/547)); an overload whose
  arm is not a HAR primitive must retire — one JS call is not that set
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
  arm, and a union of string literals, numeric literals, or both via `constantFrom` or `oneof` of every literal, and a literal mixed with an open primitive via `oneof` of `constant` and that primitive's generator, and a `[number, number]` parameter via `property2` and a `[x, y]` literal, and a generic instantiated as `name<string>` / `name<number>` /
  `name<boolean>`, and a `struct` declared in `--scope` via `record` of every
  field or `new StructName`, and a function-typed parameter via a named `.ets`
  function or an arrow, and overloads via a call per semicolon signature, and a
  mixed union of those arms via `oneof` of every arm. A Sendable parameter is
  not drawable: retire that `--func`. A branded SDK type (`Length`, `Resource`)
  is not drawable: retire that `--func`. A union with an undrawable arm
  is not drawable: retire that `--func`. An overload whose arm is not a HAR
  primitive is not drawable: retire that `--func`. A `passing`
  or `failing` property must pass that value with no `as`. Example:
  `export function add(a: number, b: number): number`.

### Not supported

Two lists. The first is the HAP runner. The second is **ArkTS in the SUT** that
`fast-check.har` cannot feed. Closing either means new pi-pbt work.

#### Runner / tree

| Gap | What happens now | What to improve |
|---|---|---|
| koala / `@koalaui` / `#generated` | Honest close is all `retired`; `report.json` validates. Cannot compile into ohosTest. | A koala/Panda runner, or a full OH-image path, not HAP ohosTest. |
| Unexported framework `.ts` (e.g. `state_mgmt` `utils.ts`) | ohosTest cannot import the class. Existing UTs `declare class` the image global. Honest close is all `retired`; `report.json` validates. | Either retire (current honest result) or a way to compile that file **without** editing it. |
| SDK / hvigor / `runtimeOS` mismatch | Gate fails, or the toolchain is deferred. pi-pbt does not rewrite manifests. | Fail loud with the expected SDK fields. Operator preflight, not a silent retarget. |
| `test/component_test` and many `examples/` HAPs | Often API 12 / HarmonyOS 5.x; this SDK 26 tree does not assemble them. | Same: fail loud. Do not rewrite `compatibleSdkVersion`. |
| OpenHarmony GN `build.sh` | Separate C++ product path. An `.ets` `--scope` is not that path. | Keep the split. Do not spawn GN for ArkTS `--scope`. |
| Native line coverage | `coverage_gaps` is not linked for ohosTest. Evidence is `aa test` plus abc strings. | ArkTS coverage if the toolchain can produce it; do not fake C++ lcov. |
| Sibling export / private-field walk | `passing` / `failing` is rejected unless the suite imports and calls `--func`. Calling a sibling is not that symbol. | Retire if ohosTest cannot bind the symbol. |
| Host `es2panda` / PandaCheck | Removed. Do not bring it back as a stand-in for HAP. | Only if a real Panda test tree is the SUT, as its own runner. |
| Kea GUI | `pi-pbt kea` is a different mode (black-box UI). | Keep it separate from source-level ArkTS. |

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
| Union of unsupported types | `passing` / `failing` is rejected. `retired` is the honest close. Drawing only `string()` is not `string \| Date`. | A named `.ets` adapter for every arm. Do not stretch HAR. |
| branded/opaque SDK types | `passing` / `failing` is rejected. `retired` is the honest close. `as Length` is not that type. | Typed constructors. Do not stretch HAR. |
| Overloads whose arms are not HAR primitives | `passing` / `failing` is rejected. `retired` is the honest close. One JS call is not that set. | One property per remaining signature. Do not stretch HAR. |
| `throw` outside the scoped function body | A same-file callee, a direct import, and up to three hops must be caught or the `--func` retired. A fourth hop is still invisible. | Catch at the call the suite actually makes, or retire. |
| `getStringByNameSync` | `passing` / `failing` is rejected unless every `string.json` name is drawn. `string()` only hits the throw path. | `constantFrom` of those names. Assert finite. `NaN` is a failing result, not a successful number. |
| Generic constrained to an unsupported type | `passing` / `failing` is rejected. `retired` is the honest close. `name<string>` is not `T extends SomeStruct`. | Retire that `--func`. |
| Callback whose type is not a function (`Function`, `ESObject`) | `passing` / `failing` is rejected. `retired` is the honest close. A named stub is not `Function` or `ESObject`. | Retire that `--func`. |
| `@Component` / `@Observed` / UI | Needs the UI runtime. | Not HAP fast-check. Kea or a UI runner. |
| koala / `@koalaui` | Different compiler. HAR never loads. | Panda/koala runner, not ohosTest. |
| Native / NAPI / `ESObject`-only APIs | `ESObject` compiles and leaves ArkTS typing. | Typed bindings, or retire. |

### Improve next

1. SDK preflight: name the missing `compileSdkVersion` / `runtimeOS` / hvigor
   plugin and stop, without editing the SUT tree.
2. Symbol-level import: `passing` / `failing` must bind the `--func`, not only
   a value import of the file that happens to define it.
3. Koala / Panda: a documented non-HAP runner, or keep retiring koala `--scope`.
4. **ArkTS SUT types:** do not stretch TS fast-check inside `.ets`. Nullish is
   `arkNullish` in the suite, not a fast-check fork. An enum declared in
   `--scope` is `constantFrom` of its members. A union of `string` / `number` /
   `boolean` is `oneof` of every arm. A scoped function that throws is caught
   with `arkThrew` or `try`/`catch`. A generic is `name<string>`,
   `name<number>`, or `name<boolean>`. A struct declared in `--scope` is
   `record` of its fields or `new StructName`. A function-typed parameter is a
   named `.ets` function or an arrow. Overloads declared with `;` need a call per
   signature. A mixed union of those arms is `oneof` of every arm. A Sendable
   parameter is retired ([#543](https://github.com/fermat-hkrc/FMBot-PBT/issues/543)).
   A branded SDK type (`Length`, `Resource`) is retired ([#545](https://github.com/fermat-hkrc/FMBot-PBT/issues/545)).
   A union with an undrawable arm is retired ([#547](https://github.com/fermat-hkrc/FMBot-PBT/issues/547)).
   An overload whose arm is not a HAR primitive is retired ([#549](https://github.com/fermat-hkrc/FMBot-PBT/issues/549)).
   A throw in a same-file callee is caught ([#551](https://github.com/fermat-hkrc/FMBot-PBT/issues/551)).
   A generic constrained to an unsupported type is retired. `name<string>` is not `T extends SomeStruct`.
   A callback typed `Function` or `ESObject` is retired. A named stub is only legal for `(...) =>`.
   A passing or failing property must import and call `--func`. Importing the file and calling a sibling is not that symbol.
   An enum imported by one relative hop is `constantFrom` of every member in that file. A second hop is still invisible.
   A union of string literals is `constantFrom` of every literal. `string()` is not that union.
   A union of numeric literals is `constantFrom` of every literal. `integer()` is not that union.
   A union of string and numeric literals is `oneof` of `constant` or `constantFrom` for every literal. `string()` or `integer()` is not that union.
   A `[number, number]` parameter is `property2(integer(), integer(), (x: number, y: number) => func([x, y]))`. `tuple()` returns a `Pair`, not that type. Any other tuple is retired.
   A literal mixed with an open primitive (`'xs' | number`, `0 | string`) is `oneof` of `constant` or `constantFrom` and that primitive's generator. `string()` alone is not that union.
   A throw three hops away is caught or retired. A fourth hop is still invisible.
   Missing `hvigorw` / SDK already defers the build gate. A mismatch is named and does not rewrite the tree.
   Long term: an ArkTS PBT library with the same `check`/`property` shape, no
   `any`, no `<const T>`. Scan the scoped signature and only allow arbitraries
   that match it.

## Related

- [Installation: HAP / ArkTS apps](../installation.md#harmonyos-hap--arkts-apps)
  ([中文](../installation.zh.md#harmonyos-hap--arkts-应用))
- [Chinese ArkTS page](arkts.zh.md)
- [Subcommands](../subcommands.md) — `build-run --build-cmd "hvigorw assembleHap --mode module -p module=entry@ohosTest -p product=default --no-daemon"`
- [Kea GUI testing](../kea.md) — black-box UI on a phone, not source-level ArkTS
- The bundled `arkts-build-run` skill: DevEco, SDK alignment, PandaCheck HAR, sign, and `aa test`
