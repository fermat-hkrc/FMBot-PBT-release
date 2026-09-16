# Reproducing a campaign, and living with the tests it wrote

A campaign's value is what survives it. This page covers the three things you
do after one finishes: see a reported failure again, run the generated tests
without pi-pbt, and keep them as part of the project.

## 1. Reproduce a reported failure

Every bug in [`pbt-out/report.json`](report-schema.md) carries the commands as
they were actually run:

```bash
jq -r '.bugs[] | "\(.id) \(.summary)\n  build: \(.reproduction.build)\n  run:   \(.reproduction.run)\n  cwd:   \(.reproduction.workdir)\n  seed:  \(.reproduction.seed // "deterministic")"' \
  pbt-out/report.json
```

`REPORT.md` should carry equivalent commands in its `## Reproduction`
section, and each `pbt-out/bug_reports/<slug>.md` should repeat the commands for
its own bug. These Markdown files are written by the campaign; pi-pbt does not
currently render them from `report.json`, so use the JSON as the checked
machine-readable contract when the files disagree.

Run the commands from `workdir`. For an MCP run in the default worktree mode,
pi-pbt has rewritten paths from the deleted worktree to the original repository.
Check out `run.revision`, apply the managed run's `changes.patch`, and then run
the recorded commands. See the [MCP run-artifact description](installation.md#delegation-from-another-coding-agent-mcp).
An in-place managed run (one started with `build_cmd`) needs no path rewrite or
patch relocation. `run` is required to be narrowed to one property or test case,
not the whole suite.

### Replaying the same case

Property frameworks expose different replay identifiers. A **seed** initializes
the generated sequence; it does not always identify the final shrunk example.
Use the framework's exact replay value when it has one:

| Framework | Replay the recorded case |
|---|---|
| RapidCheck (C++) | Use the failure's printed `reproduce=...` token: `RC_PARAMS='reproduce=<token>' ./<test> --gtest_filter=<Case>`. `RC_PARAMS='seed=<seed>'` repeats the generated sequence and shrinking process, but the seed alone is not the exact shrunk-case identifier. The recorded `reproduction.run` should already contain the exact token. |
| Hypothesis (Python) | Prefer the printed `@reproduce_failure(...)` decorator for the exact example. `pytest -k <test> --hypothesis-seed=<seed>` repeats generation from a seed but is not a durable substitute for the decorator or a regression example. |
| proptest (Rust) | Keep the generated entry under `proptest-regressions/`, then run `PROPTEST_CASES=1 cargo test <name>`. The regression entry is the durable replay record. |
| fast-check (JS/TS) | Use both fields: `fc.assert(prop, { seed: <seed>, path: "<path>" })`. `reproduction.path` follows the shrink path and is required for the exact shrunk replay. |
| gopter (Go) | Use the seed option supported by the project's gopter test runner; the exact flag is project-defined. Keep the concrete shrunk input as a regression test when possible. |

`report.json` has generic `seed` and optional `path` fields, so a framework value
that does not fit those fields—most notably RapidCheck's `reproduce=...` token—
must remain in the pasteable `reproduction.run` command. The shrunk
`counterexample` is also recorded. Convert it to an ordinary deterministic unit
test when the framework has no durable replay artifact or when long-term
regression protection matters more than replaying the shrink process.

## 2. Run the generated tests without pi-pbt

A successful campaign is expected to put tests into the repository's normal
test layout and runner. That is an output requirement, not a reason to assume
every project has the same build command. Use the commands recorded in
`report.json`; each property names its generated file and target in
`properties[].testFile` and `properties[].testTarget`.

The following are command families, not pasteable universal recipes:

| Project family | Typical build or run entry |
|---|---|
| OpenHarmony (GN) | Use the component's official `build.sh` target and the exact product/workdir recorded by the campaign. `host_product` is preferred when that component supports it; it is not available for every component. See [Build first, then test](installation.md#build-first-then-test-build-run-bring-your-own-build-command). |
| CMake / ctest | `cmake --build <build-dir> --target <recorded-target>`, then `ctest --test-dir <build-dir> -R <recorded-case>` if the project registered the test with ctest. |
| Cargo | `cargo test <recorded-name>`; a separate `cargo build --tests` is optional rather than universally required. |
| pytest | `pytest <recorded-file-or-node-id>`; there may be no separate build step. |
| Go | `go test <recorded-package> -run <recorded-name>`; do not replace the package with `./...` unless the report actually did so. |
| npm / jest / vitest | Use the project's package script and the file/test filter recorded by the campaign. Script names and argument forwarding differ by repository. |

For a project with an official or non-obvious build entry, start the campaign
with [`pi-pbt build-run`](installation.md#build-first-then-test-build-run-bring-your-own-build-command)
so the resulting report records a known command rather than an inferred one.

Two environment variables are worth knowing:

- `PBT_TEST_JOBS=1` pins the run serial. Campaigns run tests in parallel, and a
  failure that only appears in parallel is a test-isolation defect rather than a
  bug in the code — the campaign is required to reconfirm serially before
  filing anything, and you should too.
- Native code-coverage instrumentation is on by default when the required
  toolchain is available, but it never controls whether the test itself can run.
  A plain reproduction does not need coverage. Set `PBT_CODE_COVERAGE=0` if you
  are launching another pi-pbt campaign and want to disable instrumentation; see
  [Code coverage reports](installation.md#code-coverage-reports).

## 3. Keep the tests

### What to commit

Commit the tests and their build wiring. They are ordinary tests:

- the generated test file(s) named in `properties[].testFile`
- the build-file change that registers them — one additive block, or one more
  source on an existing target
- for OpenHarmony: the `test/pbt/` tree plus the one line added to
  `bundle.json` `build.test`
- for proptest: the `proptest-regressions/` files, so a found counterexample
  stays a permanent regression

### What not to commit

`pbt-out/` is campaign scratch — `PLAN.md`, `PROPERTIES.md`, `COVERAGE.md`,
`INVARIANTS.md`, `FUNCTION_INDEX.md`, `report.json`, `REPORT.md`,
`bug_reports/`, `code-coverage/`. Keep it out of the repository:

```gitignore
pbt-out/
```

Attach `REPORT.md` and `report.json` to the PR, the issue, or your CI artifacts
instead. The exception is deliberate: if you want a bug report to live in the
repository, copy that one file somewhere the project keeps such things, rather
than committing the whole directory.

Incremental campaigns can reuse several prior artifacts, including
`FUNCTION_INDEX.md`, `SCAN.md`, `COVERAGE.md`, and `COVERAGE_STATUS.md`, when
those files remain in the output location. `INVARIANTS.md` is also useful
context, but it is not the only reusable artifact. Keep the campaign directory
outside version control if you want a later run to continue its registry; a
`hook-run` invocation cleans its selected `--out` directory before starting, so
an external archive or managed-run artifact store is needed across those runs.

### Regression from here on

Once committed, the tests are the project's, and they run in CI with everything
else. Nothing about them needs pi-pbt: the property framework is a normal test
dependency, and the assertions are normal assertions. A later campaign on the
same module reads `COVERAGE.md`/`FUNCTION_INDEX.md` if they are still around and
extends the tree rather than duplicating it — but the tests you committed keep
working whether or not another campaign ever runs.

When a property test starts failing months later, that is the point of it. Use
its `formal` statement from the report to decide whether the code broke or the
law was wrong; the same triage the campaign applied (`## Bugs Found` vs Design
Caveats) applies to you.
