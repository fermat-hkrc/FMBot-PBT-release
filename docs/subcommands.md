# Reference

[中文](subcommands.zh.md) · The two modes step by step: [usage guide](usage.md)

## Commands

| Command | Purpose |
|---|---|
| `fmbot-pbt` | interactive session; ask for a test in plain language |
| `fmbot-pbt -p "<request>"` | one unattended run; the exit code is not a verdict |
| `fmbot-pbt mcp` | MCP server for Claude Code and Codex (mode 1) |
| `fmbot-pbt hook-run <sha>` | test one commit or one merge request; the exit code is the verdict (mode 2) |
| `fmbot-pbt build-run --build-cmd "<command>"` | run your build command first, then test |
| `fmbot-pbt watch` | watch a branch and test each new commit |
| `fmbot-pbt resume` | continue an interrupted run |
| `fmbot-pbt finalize` | validate and render existing results without a model call |
| `fmbot-pbt scan` / `fmbot-pbt coverage` | list functions / summarize test progress, no model call |
| `fmbot-pbt dashboard` | local web dashboard |
| `fmbot-pbt skill-install` | install the `pi-pbt-dev` skill for a coding agent |
| `fmbot-pbt kea` | HarmonyOS on-device UI testing (experimental); see the [Kea guide](kea.md) |

Common flags: `--lang zh` for Chinese output; `--effort quick|standard|thorough` for depth; `--provider`, `--model`,
`--thinking` to choose the model; `--scope <path>` to limit the target; `--out <dir>` for the output directory.

## `hook-run`

```text
fmbot-pbt hook-run <sha> [--base <rev>] [--spec <file>]... [--repo <path>] [--out <dir>] [--workdir <dir>]
  [--build-cmd "<command>" [--build-workdir <dir>]] [--scope <path> [--func <name>]] [--target change|scope]
  [--effort …] [--lang zh] [--prompt <text> | --prompt-file <path>]
  [--provider <p>] [--model <m>] [--thinking <level>] [--mode text|json|json-full] [--no-coverage]
```

| Input | Flag | Meaning |
|---|---|---|
| Documents (optional) | `--spec <file>` | a requirement document whose explicit statements decide the verdict; repeatable; without it the intent is inferred from the code |
| Code | `<sha>` | the commit under test; a merge commit is compared with its first parent |
| | `--base <rev>` | the merge request's base commit; tests every change from it to `<sha>` instead |
| | `--repo <path>` | the working tree with `<sha>` checked out, default the current directory; use `fetch-depth: 0` in CI |
| | `--scope <path>`, `--func <name>` | test only this file or directory, and one function in it (`--func` needs `--scope`) |
| | `--target scope` | test everything under `--scope` at `<sha>`, not only the change (`--scope .` is the whole repository); the checkout must be at `<sha>`; not with `--base` |
| Build command (optional) | `--build-cmd "<command>"` | build the checkout with this command before testing; the campaign rebuilds with this command only. Without it, build in CI first and fmbot-pbt reuses the output |
| | `--build-workdir <dir>` | where the build command runs, default `--repo`; it must contain `--repo` or lie inside it |
| | `--workdir <dir>` | a scratch directory that is deleted and recreated; default `pbt-hook-work` next to `--repo`. It is not the build directory |

With `--build-cmd` the checkout must already be at `<sha>` (never checked out for you; otherwise exit `2`); a failing build exits `3` before testing, output in `<out>/build.log`. `--out` and `--workdir` may not contain the checkout, the
build directory, the current directory or a `--spec` document, symbolic links included.

| Output | Meaning |
|---|---|
| `<out>/REPORT.md`, `REPORT.html` | the test report; `--out` defaults to `pbt-out-hook` next to `--repo` |
| `<out>/report.json` | the result for programs |
| `<out>/bug_reports/` | one report per bug |
| exit code `0` / `1` / `2` / `3` | no bug / bugs found / report missing or change set not computable / build failed |

## `build-run`

```text
fmbot-pbt build-run --build-cmd "<command>" [--spec <file>]... [--workdir <dir>] [--repo <path>] [--out <dir>]
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
fmbot-pbt build-run \
  --workdir "$PBT_OH_WORKSPACE" \
  --repo "$PBT_OH_WORKSPACE/foundation/arkui/ace_engine" \
  --build-cmd "./build.sh --export-para PYCACHE_ENABLE:true --product-name host_product --build-target base_unittest --ccache --no-prebuilt-sdk" \
  --scope frameworks/base/geometry
```

A plain CMake project:

```bash
fmbot-pbt build-run --build-cmd "cmake -S . -B build && cmake --build build --target calc_test" --scope src/calc.cpp
```

HarmonyOS apps: see the [ArkTS guide](pbt/arkts.md). `--isolate` requires `--scope <file.ets> --func <name>` and generates `<out>/isolate-hap`. It copies the product graph into `entry/src/common` (no hard links or symlinks), retargets the suite import, and replaces `--build-cmd` with the generated HAP's ohosTest build after `ohpm install`. Copies and original dependencies are read-only and hash-checked; a modified or missing graph cannot attest passing/failing. The in-session skill uses this same preparation.

Keeping the JSON event log (stdout carries only events, diagnostics go to stderr; the result is still `report.json`):

```bash
set -o pipefail
LOG_DIR=/tmp/pbt-logs && mkdir -p "$LOG_DIR"
fmbot-pbt build-run --mode json \
  --build-cmd "./build.sh --product-name host_product --build-target base_unittest" \
  2>"$LOG_DIR/stderr.log" | tee "$LOG_DIR/events.jsonl"
```

## `mcp`

```bash
claude mcp add --transport stdio fmbot-pbt -- fmbot-pbt mcp
codex mcp add fmbot-pbt -- fmbot-pbt mcp
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
| `base` | base commit; tests every change from it to `revision`; not with `target: "scope"` |
| `scope`, `target` | limit the target; `target: "scope"` tests everything under `scope`, not only the change (`.` is the whole repository; one run per module is easier to read) |
| `effort`, `lang`, `provider`, `model` | as on the command line |
| `prompt` | extra instructions for the run |
| `build_cmd`, `build_workdir` | a build command and its directory, as `hook-run --build-cmd`; the test then runs in the original checkout, which must be at `revision`; do not edit it until the run ends |

Without `build_cmd` the test runs in a separate git worktree. Results are kept in `~/.pi-pbt/runs/<run_id>/`; the
generated tests are in its `changes.patch`.

## `watch`

```text
fmbot-pbt watch [--repo <path>] [--base <rev>] [--spec <file>]... [--interval <sec>] [--fetch] [--branch <name>] [--out <dir>] [--lang zh] [--effort …]
```

Runs `hook-run` for each new commit, oldest first, with any `hook-run` flag given here, validated before watching.
At most the newest 10 new commits per poll are tested; use `hook-run` when every commit must be tested. With
`--build-cmd` only the checked-out head is tested, and `--fetch` is refused.

## Skills in a session

In an interactive `fmbot-pbt` session, `/skill:pbt-build-run`, `/skill:pbt-hook-run` and `/skill:pbt-watch` take the
arguments of `build-run`, `hook-run` and `watch` and run the same preparation, build gate and verdict (`0`–`3`); the
test runs visibly in the session. Stop a watch with the `stop_watch` tool.

Differences from the commands: the session keeps its working directory; `--mode json|json-full` is refused;
`--provider`, `--model` and `--thinking` switch the session's model for the rest of the session; `--provider` needs `--model`.

## Other commands

| Command | Meaning |
|---|---|
| `fmbot-pbt resume [--out <dir>] [--attempts <n>] [--wait <sec>]` | continue an interrupted run |
| `fmbot-pbt finalize [--out <dir>] [--lang zh]` | validate and render existing results; `0` valid, `2` invalid |
| `fmbot-pbt scan [--dir <path>] [--out <dir>]` | list functions into `FUNCTION_INDEX.md`; see [coverage tracking](coverage-tracking.md) |
| `fmbot-pbt coverage [--list \| --json]` | summarize test progress in `pbt-out/` |
| `fmbot-pbt dashboard [--port 8000]` | local web dashboard |
| `fmbot-pbt skill-install [--host <agent>] [--dir <path>] [--force]` | install the `pi-pbt-dev` skill; see the [skill guide](skill-pi-pbt-dev.md) |

## Depth

| | `quick` | `standard` | `thorough` |
|---|---|---|---|
| Time budget | about 10 minutes | about 30 minutes | unlimited |
| Properties per test subject | 3–5 | 5–8 | as needed |
| Inputs per property | about 100 | ≥ 1000 | ≥ 10000 |

A **test subject** is usually one function or method, or a group of related operations tested in sequence, such as
opening an account, depositing, withdrawing, and closing it. The count applies to each subject, not to an entire
repository or module, and does not refer to a build target.

`hook-run` and `watch` default to `quick`, other entries to `standard`.

Every test entry reads user-provided requirement, API or design documents before mapping them to the code and deriving properties. Intent is inferred from code only where documents are absent or silent.

## Environment variables

| Variable | Meaning |
|---|---|
| `PBT_TLS_EXCEPTIONS=off` | turn off [enterprise certificate exceptions](network.md) and package-tool TLS compatibility |
| `NODE_USE_SYSTEM_CA=0` | opt out of the default system CA addition; explicit runtime CA flags are respected |
| `NODE_EXTRA_CA_CERTS=/path/ca.pem` | add PEM CA certificates; set before starting fmbot-pbt; inherited by child processes |
| `PBT_LANG` | `zh` or `en`, output language |
| `PBT_EFFORT` | default depth |
| `PBT_TEST_JOBS` | test and build parallelism: a number, `max`, or `auto` (default, idle cores) |
| `PBT_CODE_COVERAGE=0` | turn native code coverage off |
| `PBT_CLOSEOUT_REVIEW=off` | skip the independent review (run by the bundled pi-subagents); the report says it was not reviewed |
| `PBT_SCAN_DELEGATION=off` | do not use parallel scan subagents |
| `PBT_VERDICTS=off` / `PBT_CARRY=off` | do not record / do not reuse earlier verdicts |
| `PBT_OH_WORKSPACE` | the OpenHarmony source tree; detected automatically when run inside it |
| `PBT_PHASE_MODELS` / `PBT_DESIGN_THINKING` | a model per phase, e.g. `{"plan":"anthropic/claude-opus-5-5"}` / the Scan and Plan thinking level, default `xhigh` (falling back to `max`, then `high`), never below the session level; `keep` uses the session level |
| `PBT_MCP_MAX_CONCURRENT` | MCP tests running at once, default 1 |
| `PI_PBT_RUNS_DIR` | MCP results directory, default `~/.pi-pbt/runs` |
