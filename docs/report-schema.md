# Campaign result contract (`pbt-out/report.json`)

A completed campaign writes two result artifacts:

- `REPORT.md` is the human-readable narrative.
- `report.json` is the structured result for CI, dashboards, and other programs.

They should describe the same verified facts, but they are separate campaign
outputs. pi-pbt validates `report.json`; it does **not** currently render
`REPORT.md` from the JSON. Do not treat one file as an automatically generated
copy of the other.

`hook-run` requires both files in the same recognized artifact root. A missing
or invalid `report.json` makes the check incomplete (exit `2`), even if
`REPORT.md` exists. See the
[`hook-run` exit-code table](installation.md#ci--git-hook-integration).

Before `report.json`, consumers had to infer a verdict from Markdown files. The
JSON makes the result explicit. The shipped validator checks it when the
campaign writes the file and again when `hook-run` applies its gate; validation
returns all detected problems so the campaign can correct them together.

## The two invariants

Everything else is bookkeeping. These two are why the file is worth writing.

**A bug names the property that found it, and the property names the bug back.**
`bugs[].propertyId` and `properties[].bugId` must agree. "Which test caught
this, and how do I run just that one" is then a lookup, not a reading
comprehension exercise. A failing property with no `bugId` is un-triaged: file
the bug or retire the property.

**A bug carries a pasteable reproduction.** `reproduction.build`, `.run`,
and `.workdir` record the commands and directory actually used, with `run`
narrowed to the reported property or case. Framework replay data belongs in
`.seed` and, for fast-check, `.path`; these fields are not a universal replay
format. A command containing `...` or a `<placeholder>` is rejected. See
[Reproducing a campaign](reproducing.md) for framework-specific details.

## Shape

JSON does not support comments; the comments below are explanatory and must be
removed in a real `report.json`.

```jsonc
{
  "schemaVersion": 1,                    // bumped when a field changes meaning
  "generator": { "tool": "pi-pbt", "version": "0.1.17" },
  "run": {
    "date": "2026-09-14",
    "revision": "76296ab70ebeb03798762e396289c5625eb5733a",   // the commit tested — makes the report reproducible
    "repository": "/srv/oh/foundation/arkui/ace_engine",   // absolute
    "target": "frameworks/base/geometry", // module or scope under test
    "tier": "quick"                       // quick | standard | thorough
  },
  "build": {
    "command": "./build.sh --product-name host_product --build-target geometry_pbt_test",
    "workdir": "/srv/oh",
    "result": "success"                   // success | failure | not-applicable
  },
  "properties": [
    {
      "id": "p1",                         // stable within the campaign
      "name": "matrix4 inverse round-trips",
      "formal": "∀ m. invertible(m) ⇒ m * inverse(m) ≈ identity",
      "oracle": "round_trip",
      "sourceFile": "/srv/oh/foundation/arkui/ace_engine/frameworks/base/geometry/matrix4.cpp",
      "functionName": "Matrix4::Invert",
      "testFile": "/srv/oh/foundation/arkui/ace_engine/test/pbt/base/geometry_pbt_test.cpp",
      "testTarget": "geometry_pbt_test",
      "status": "failing",                // passing | failing | retired
      "counterexample": "m = {{1,2},{2,4}}",  // required when failing
      "bugId": "b1"                       // null when the property passed
    }
  ],
  "bugs": [
    {
      "id": "b1",
      "propertyId": "p1",                 // required — the property that found it
      "summary": "Invert returns identity for a singular matrix",
      "severity": "high",                 // low | medium | high | critical
      "expected": "false / error for a singular matrix",
      "actual": "identity, silently",
      "counterexample": "m = {{1,2},{2,4}}",
      "reportPath": "bug_reports/invert-singular.md",   // relative to this file
      "reproduction": {
        "build": "./build.sh --product-name host_product --build-target geometry_pbt_test",
        "run": "RC_PARAMS='reproduce=BQSThRncAQAA' out/host/host_product/tests/geometry_pbt_test --gtest_filter=GeometryPbt.Inverse",
        "workdir": "/srv/oh",
        "seed": null,                     // exact RapidCheck replay is in run
        "path": null                      // fast-check replay path; null elsewhere
      }
    }
  ],
  "totals": { "properties": 1, "passing": 0, "failing": 1, "bugs": 1 }
}
```

`totals` is checked against the arrays. `passing` and `failing` count only
those statuses; a `retired` property remains in `totals.properties` but belongs
to neither count.

## Paths

Two conventions, enforced in both directions so a consumer never has to guess:

- **Repository paths are absolute** — `run.repository`, `build.workdir`,
  `properties[].sourceFile`, `properties[].testFile`, and
  `bugs[].reproduction.workdir`. A consumer should not need an implicit base
  directory to locate the tested checkout or run the commands.
- **Artifact files are relative to `report.json` itself** — `bugs[].reportPath`.
  A managed run may collect or relocate the artifact directory after the
  campaign, so an absolute bug-report path could go stale. Resolve it against
  the directory that contains the `report.json` you are reading.

## Reading it

```bash
# Did this campaign find anything?
jq '.totals.bugs' pbt-out/report.json

# For each bug: which command and replay data belong to it?
jq -r '.bugs[] | "\(.id) \(.summary)\n  run:  \(.reproduction.run)\n  seed: \(.reproduction.seed // "n/a")\n  path: \(.reproduction.path // "n/a")"' \
  pbt-out/report.json

# Every property that is still red
jq -r '.properties[] | select(.status == "failing") | "\(.name) — \(.counterexample)"' pbt-out/report.json
```

## Rules the validator enforces beyond shape

- `run.tier` is one of `quick`, `standard`, `thorough`.
- `run.revision` is the commit the campaign tested, as a **resolved full object ID** (40 hex, or 64 for SHA-256) — not `HEAD`, not a branch name, not an abbreviation. A managed run snapshots an immutable revision while the checkout moves on; applying `changes.patch` to an unknown revision reproduces something else.
- `build.command` may be `null` only when `build.result` is `not-applicable`; `build.workdir` is absolute. The runtime validator is the JSON contract here even though the current TypeScript interface still declares `build.command` as a string. The command itself is placeholder-checked like `reproduction.build` — a clean campaign has no per-bug reproduction to catch a templated `<build command as run>`.
- **At least one property must have been executed** (`passing` or `failing`). An all-empty report — no properties, no bugs, zero totals — would otherwise satisfy every other rule and gate green while the campaign tested nothing. The one exception is `build.result: "failure"`: nothing could run, so zero executed properties is the honest report, and rejecting it would hide the build failure (`hook-run` exit `3`) behind "no usable report" (`2`).
- A `passing` property carries `counterexample: null`. A witness only means something while the property fails; `retired` keeps its witness as the record of why it was dropped.
- A bug's linked property must be `failing` — a bug comes from a failing property.
- `build.result: "failure"` is a valid report — and a **failed run**: `hook-run` exits 3 and a managed run is `failed`. Nothing was tested.
- `reproduction.build` may be `null` only when `build.result` is `not-applicable` (for example, when the test command itself is the only step). Never `"N/A"`.
- `reproduction.seed` is a framework replay value or `null` for a deterministic command. RapidCheck's exact shrunk replay is its printed `reproduce=...` token, which belongs in the recorded run command; a RapidCheck seed alone reruns the generated sequence but does not identify the shrunk case. `reproduction.path` is optional and is used with the seed for fast-check replay. See [Replaying the same case](reproducing.md#replaying-the-same-case).
- `reproduction.build` / `.run` must not contain `...`, `<placeholder>`, `TODO`, `TBD`, `FIXME`, `XXX`, `PLACEHOLDER` or `N/A`.
- Every bug's property names it back, and their counterexamples agree — one witness per link.
- `reportPath` stays inside the artifact directory: no absolute paths, no `..`.

## Managed runs

Under the MCP run manager's default worktree mode, the campaign runs in
`<runDir>/worktree`, captures its repository changes in `changes.patch`, and
then deletes that worktree. Before deletion, pi-pbt rewrites worktree paths in
`report.json`, `REPORT.md`, `PROPERTIES.md`, `COVERAGE.md`, and bug reports to
the original repository path. In that repository, check out `run.revision` and
then apply `changes.patch` before using paths to generated tests. See the
[MCP run-artifact description](installation.md#delegation-from-another-coding-agent-mcp).

An in-place managed run (selected when `pbt_start` receives `build_cmd`) has no
relocation step: its absolute paths already refer to the repository where the
campaign ran.

## Compatibility

`schemaVersion` is the compatibility boundary. A consumer must check it before
using fields and reject a version it does not know. Adding an optional field
does not require a bump; changing a field's meaning or removing one does.
