# Reference

[中文](subcommands.zh.md) · The two modes step by step: [usage guide](usage.md)

## Commands

| Command | Purpose |
|---|---|
| `pi-pbt` | interactive session; ask for a test in plain language |
| `pi-pbt -p "<request>"` | one unattended run; the exit code is not a verdict |
| `pi-pbt mcp` | MCP server for Claude Code and Codex (mode 1) |
| `pi-pbt hook-run <sha>` | test one commit or one merge request; the exit code is the verdict (mode 2) |
| `pi-pbt build-run --build-cmd "<command>"` | run your build command first, then test |
| `pi-pbt watch` | watch a branch and test each new commit |
| `pi-pbt resume` | continue an interrupted run |
| `pi-pbt finalize` | validate and render existing results without a model call |
| `pi-pbt scan` / `pi-pbt coverage` | list functions / summarize test progress, no model call |
| `pi-pbt dashboard` | local web dashboard |
| `pi-pbt skill-install` | install the `pi-pbt-dev` skill for a coding agent |
| `pi-pbt kea` | HarmonyOS on-device UI testing (experimental); see the [Kea guide](kea.md) |

Common flags: `--lang zh` for Chinese output; `--effort quick|standard|thorough` for depth; `--provider`, `--model`,
`--thinking` to choose the model; `--scope <path>` to limit the target; `--out <dir>` for the output directory.

## `hook-run`

```text
pi-pbt hook-run <sha> [--base <rev>] [--spec <file>]... [--repo <path>] [--out <dir>] [--workdir <dir>]
  [--scope <path>] [--effort …] [--lang zh] [--prompt <text> | --prompt-file <path>]
  [--provider <p>] [--model <m>] [--thinking <level>] [--mode text|json|json-full] [--no-coverage]
```

| Input | Flag | Meaning |
|---|---|---|
| Documents (optional) | `--spec <file>` | a requirement document whose explicit statements decide the verdict; repeatable; without it the intent is inferred from the code |
| Code | `<sha>` | the commit under test; a merge commit is compared with its first parent |
| | `--base <rev>` | the merge request's base commit; tests every change from it to `<sha>` instead |
| | `--repo <path>` | the working tree with `<sha>` checked out, default the current directory; use `fetch-depth: 0` in CI |
| | `--scope <path>` | test only this file or directory |
| Build command | — | build with the project's own command in CI first; pi-pbt reuses the build output |

| Output | Meaning |
|---|---|
| `<out>/REPORT.md`, `REPORT.html` | the test report; `--out` defaults to `pbt-out-hook` next to `--repo` |
| `<out>/report.json` | the result for programs |
| `<out>/bug_reports/` | one report per bug |
| exit code `0` / `1` / `2` / `3` | no bug / bugs found / report missing or change set not computable / build failed |

## `build-run`

```text
pi-pbt build-run --build-cmd "<command>" [--spec <file>]... [--workdir <dir>] [--repo <path>] [--out <dir>]
  [--scope <path>] [--func <name>] [--diff <file> | --commit <sha>] [--isolate]
  [--effort …] [--lang zh] [--prompt <text> | --prompt-file <path>]
  [--provider <p>] [--model <m>] [--thinking <level>] [--mode text|json|json-full] [--no-coverage]
```

Runs your build command first and tests only if it succeeds; the run builds with that command only.

| Input | Flag | Meaning |
|---|---|---|
| Documents (optional) | `--spec <file>` | a requirement document whose explicit statements decide the verdict; repeatable; without it the intent is inferred from the code |
| Code | `--repo <path>` | the repository under test, default the current directory |
| | `--scope <path>`, `--func <name>` | test only this file or directory, and one function in it |
| | `--diff <file>`, `--commit <sha>` | the change: a patch file or a commit; without them the run tests `--scope` |
| Build command | `--build-cmd "<command>"` | required, naming a target that already exists and builds |
| | `--workdir <dir>` | where the build command runs, default the current directory |

| Output | Meaning |
|---|---|
| `<out>/build.log` | the build log; `--out` defaults to `<repo>/pbt-out` |
| `<out>/REPORT.md`, `report.json`, `bug_reports/` | the test results, as for `hook-run` |
| new tests in the project's test tree | registered in an existing test target; after generation the same command can build that target |
| exit code | `3` means the build failed; after a successful build the result is in `report.json` |

For OpenHarmony components prefer the native `host_product`; output is in `out/host/host_product`. Use a device product
only when the component has no host target and a runner is configured for that device product.
There is no single `--build-cmd` for the whole OpenHarmony tree: point `--repo` at this component and build an
existing unittest of it. Example:

```bash
export PBT_OH_WORKSPACE=/path/to/openharmony
pi-pbt build-run \
  --workdir "$PBT_OH_WORKSPACE" \
  --repo "$PBT_OH_WORKSPACE/foundation/arkui/ace_engine" \
  --build-cmd "./build.sh --export-para PYCACHE_ENABLE:true --product-name host_product --build-target base_unittest --ccache --no-prebuilt-sdk" \
  --scope frameworks/base/geometry
```

A plain CMake project:

```bash
pi-pbt build-run --build-cmd "cmake -S . -B build && cmake --build build --target calc_test" --scope src/calc.cpp
```

HarmonyOS apps: see the [ArkTS guide](pbt/arkts.md).

Keeping the JSON event log (stdout carries only events, diagnostics go to stderr; the result is still `report.json`):

```bash
set -o pipefail
LOG_DIR=/tmp/pbt-logs && mkdir -p "$LOG_DIR"
pi-pbt build-run --mode json \
  --build-cmd "./build.sh --product-name host_product --build-target base_unittest" \
  2>"$LOG_DIR/stderr.log" | tee "$LOG_DIR/events.jsonl"
```

## `mcp`

```bash
claude mcp add --transport stdio pi-pbt -- pi-pbt mcp
codex mcp add pi-pbt -- pi-pbt mcp
```

| Tool | Purpose |
|---|---|
| `pbt_start` | start a test; returns a `run_id` at once |
| `pbt_status` | progress |
| `pbt_report` | verdict, report summary and bug reports |
| `pbt_cancel` | cancel a test |
| `pbt_watch_start` / `pbt_watch_stop` | watch a branch and test each new commit |

`pbt_start` parameters:

| Parameter | Meaning |
|---|---|
| `repo` | repository under test (required) |
| `spec` | requirement documents (optional), one path or a list; without them the intent is inferred from the code |
| `include_uncommitted` | include uncommitted changes |
| `revision` | commit to test, default `HEAD` |
| `base` | base commit; tests every change from it to `revision`; not with `build_cmd` |
| `scope` | limit the target |
| `effort`, `lang`, `provider`, `model` | as on the command line |
| `prompt` | extra instructions for the run |
| `build_cmd`, `build_workdir` | a build command and its directory; the test then runs in the original checkout, so do not edit it until the run ends |

Without `build_cmd` the test runs in a separate git worktree. Results are kept in `~/.pi-pbt/runs/<run_id>/`; the
generated tests are in its `changes.patch`.

## `watch`

```text
pi-pbt watch [--repo <path>] [--base <rev>] [--spec <file>]... [--interval <sec>] [--fetch] [--branch <name>] [--out <dir>] [--lang zh] [--effort …]
```

Runs `hook-run` for each new commit. When one poll finds more than 10 new commits only the newest 10 are tested; use
`hook-run` when every commit must be tested.

## Other commands

| Command | Meaning |
|---|---|
| `pi-pbt resume [--out <dir>] [--attempts <n>] [--wait <sec>]` | continue an interrupted run |
| `pi-pbt finalize [--out <dir>] [--lang zh]` | validate and render existing results; `0` valid, `2` invalid |
| `pi-pbt scan [--dir <path>] [--out <dir>]` | list functions into `FUNCTION_INDEX.md`; see [coverage tracking](coverage-tracking.md) |
| `pi-pbt coverage [--list \| --json]` | summarize test progress in `pbt-out/` |
| `pi-pbt dashboard [--port 8000]` | local web dashboard |
| `pi-pbt skill-install [--host <agent>] [--dir <path>] [--force]` | install the `pi-pbt-dev` skill; see the [skill guide](skill-pi-pbt-dev.md) |

## Depth

| | `quick` | `standard` | `thorough` |
|---|---|---|---|
| Time budget | about 10 minutes | about 30 minutes | unlimited |
| Properties per target | 3–5 | 5–8 | as needed |
| Inputs per property | about 100 | ≥ 1000 | ≥ 10000 |

`hook-run` and `watch` default to `quick`, other entries to `standard`.

## Environment variables

| Variable | Meaning |
|---|---|
| `PBT_LANG` | `zh` or `en`, output language |
| `PBT_EFFORT` | default depth |
| `PBT_TEST_JOBS` | test and build parallelism: a number, `max`, or `auto` (default, idle cores) |
| `PBT_CODE_COVERAGE=0` | turn native code coverage off |
| `PBT_CLOSEOUT_REVIEW=off` | skip the independent review (run by the bundled pi-subagents); the report says it was not reviewed |
| `PBT_SCAN_DELEGATION=off` | do not use parallel scan subagents |
| `PBT_VERDICTS=off` / `PBT_CARRY=off` | do not record / do not reuse earlier verdicts |
| `PBT_OH_WORKSPACE` | the OpenHarmony source tree; detected automatically when run inside it |
| `PBT_PHASE_MODELS` | a model per phase, e.g. `{"plan":"anthropic/claude-opus-5-5"}` |
| `PBT_MCP_MAX_CONCURRENT` | MCP tests running at once, default 1 |
| `PI_PBT_RUNS_DIR` | MCP results directory, default `~/.pi-pbt/runs` |
