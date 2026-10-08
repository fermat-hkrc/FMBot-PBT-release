# ArkTS PBT (HarmonyOS HAP)

[中文版 / Chinese version](arkts.zh.md)

[Back to installation](../installation.md#harmonyos-hap--arkts-apps) ·
[Subcommands](../subcommands.md)

pi-pbt supports ArkTS as **HarmonyOS HAP ohosTest**, not as a host compiler
for every `.ets` file. A property is Hypium + fast-check in
`src/ohosTest/ets/test/<Name>Pbt.test.ets`, assembled with `hvigorw`, signed,
installed, and run with `hdc shell aa test`. The verdict is that
`OHOS_REPORT_RESULT` line. A successful `hvigorw` only proves compilation.

This is not Kea (`pi-pbt kea` drives a black-box GUI). This is not OpenHarmony
GN C++. This is not koala / PandaCheck.

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

Toolchain, SDK alignment, fast-check HAR, signing, and `aa test` are the
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

fast-check does not come from the ohpm registry. Use
`arkts-build-run/scripts/make-fast-check-har.sh`, then local HAR references in
`oh-package.json5`.

## Current progress

This is the HAP ohosTest path on current `main`. It is not a general ArkTS
engine. Open issue: [#366](https://github.com/fermat-hkrc/FMBot-PBT/issues/366).

### Supported

- Detect a HAP root (`oh-package.json5` + `build-profile.json5`) and set
  `PBT_HAP=1`. Run Hypium + fast-check in `src/ohosTest`.
- Gate on the operator's `hvigorw` command. Sign, install, `hdc shell aa test`.
- `--repo` (scan / `--scope`) separate from `--workdir` (build), same git
  checkout. Custom `--out` may hold `REPORT.md`.
- A HAP that **owns** the scoped `.ets` and can import it: execute that symbol
  and attest `passing` or `failing`.
- Honesty gates already shipped: no `ohosTest/ets/sut/` copy; no `@kit.ArkUI`
  stand-in for koala; `sourceFile` / `functionName` pinned to HARD `--scope` /
  `--func`; `declare class` is not an import; do not rewrite `--scope` so the
  import compiles; a directory `--scope` matches by path prefix and by that
  property's `sourceFile`.
- If this HAP cannot import `--scope` as it stands, **retire**.
- fast-check as a local HAR (`make-fast-check-har.sh`), not the ohpm registry.
- **SUT types the HAR can drive:** exported functions of `number` / `string` /
  `boolean` / arrays of those, and plain objects you can write as `.ets`
  literals. Example: `export function add(a: number, b: number): number`.

### Not supported

Two lists. The first is the HAP runner. The second is **ArkTS in the SUT** that
`fast-check.har` cannot feed. Closing either means new pi-pbt work.

#### Runner / tree

| Gap | What happens now | What to improve |
|---|---|---|
| koala / `@koalaui` / `#generated` | Honest close is all `retired`. Cannot compile into ohosTest. | A koala/Panda runner, or a full OH-image path, not HAP ohosTest. |
| Unexported framework `.ts` (e.g. `state_mgmt` `utils.ts`) | ohosTest cannot import the class. Existing UTs `declare class` the image global. | Either retire (current honest result) or a way to compile that file **without** editing it. |
| SDK / hvigor / `runtimeOS` mismatch | Gate fails, or the toolchain is deferred. pi-pbt does not rewrite manifests. | Fail loud with the expected SDK fields. Operator preflight, not a silent retarget. |
| `test/component_test` and many `examples/` HAPs | Often API 12 / HarmonyOS 5.x; this SDK 26 tree does not assemble them. | Same: fail loud. Do not rewrite `compatibleSdkVersion`. |
| OpenHarmony GN `build.sh` | Separate C++ product path. An `.ets` `--scope` is not that path. | Keep the split. Do not spawn GN for ArkTS `--scope`. |
| Native line coverage | `coverage_gaps` is not linked for ohosTest. Evidence is `aa test` plus abc strings. | ArkTS coverage if the toolchain can produce it; do not fake C++ lcov. |
| All-retired `report.json` | A **no-op** `--build-cmd` (`true` / `exit 0`) already allows every property `retired`. A **successful real** HAP build (`hvigorw …`) with every row `retired` is still rejected as “no property was executed”. | Allow all-retired when the HAP built and import of `--scope` is impossible — not only when the build command was a no-op. |
| Sibling export / private-field walk | A suite can import an already-exported helper from `--scope` and call the unexported method on a live instance. That is not a rewrite, and the import check accepts it. | Decide whether `passing` requires importing the **symbol**, not only the file. |
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
| `@Sendable` / concurrent types | `integer()` is a JS `number`, not a `Box`. `boxId(n)` does not type-check. Building `new Box(n)` in the predicate tests `boxId`, not Sendable (no other-thread object, no `SendableMap`). | ArkTS-shaped arbitraries, or a runner that can construct Sendable values. Until then, retire `--func` whose arguments are only Sendable. |
| Unexported `class` / `private` fields | Cannot `import` or `new` without editing `--scope`. | Compile/import without rewriting the SUT, or retire. |
| Union / optional / `T \| null` | No typed arbitrary. Hand-rolled `oneof` may still fail the `.ets` callback type. | Adapters (`arkNullish()`, `arkEnum`) that `.ets` can name without `ESObject`. |
| `enum`, `struct`, branded/opaque SDK types | HAR will not `new` them. | Typed constructors in the harness, or retire. |
| Overloads | One JS call; ArkTS overload resolution is not in the HAR. | One property per signature, from the scoped symbol. |
| `throw` / `BusinessError` | `check()` wants a `boolean`. try/catch is manual. | Negative/error oracle that Hypium can print. |
| Generics (`foo<T>`) | No `T` in the HAR. | Instantiate in the suite from the scanned signature, or retire. |
| Callback / function-typed params | No ArkTS-legal `func()` arbitrary. | Explicit callback stubs in `.ets`, or retire. |
| `@Component` / `@Observed` / UI | Needs the UI runtime. | Not HAP fast-check. Kea or a UI runner. |
| koala / `@koalaui` | Different compiler. HAR never loads. | Panda/koala runner, not ohosTest. |
| Native / NAPI / `ESObject`-only APIs | `ESObject` compiles and leaves ArkTS typing. | Typed bindings, or retire. |

### Improve next

1. All-retired close-out for a **real** HAP build: when `hvigorw` succeeded and
   every property is `retired` because `--scope` cannot be imported, `report.json`
   must validate. (No-op `--build-cmd` already allows all-retired; that is a
   different case.)
2. SDK preflight: name the missing `compileSdkVersion` / `runtimeOS` / hvigor
   plugin and stop, without editing the SUT tree.
3. Symbol-level import: `passing` / `failing` must bind the `--func`, not only
   a value import of the file that happens to define it.
4. Koala / Panda: a documented non-HAP runner, or keep retiring koala `--scope`.
5. **ArkTS SUT types:** do not stretch TS fast-check inside `.ets`. Add `.ets`
   adapters for nullish/enum/object, then Sendable (or retire those `--func`).
   Long term: an ArkTS PBT library with the same `check`/`property` shape, no
   `any`, no `<const T>`. Scan the scoped signature and only allow arbitraries
   that match it.

## Related

- [Installation: HAP / ArkTS apps](../installation.md#harmonyos-hap--arkts-apps)
  ([中文](../installation.zh.md#harmonyos-hap--arkts-应用))
- [Chinese ArkTS page](arkts.zh.md)
- [Subcommands](../subcommands.md) — `build-run --build-cmd "./hvigorw assembleHap"`
- [Kea GUI testing](../kea.md) — black-box UI on a phone, not source-level ArkTS
- The bundled `arkts-build-run` skill: DevEco, SDK alignment, fast-check HAR, sign, and `aa test`
