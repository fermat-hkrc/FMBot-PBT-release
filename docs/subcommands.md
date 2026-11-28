# Reference

[中文](subcommands.zh.md) · The modes step by step: [usage guide](usage.md)

## Commands

| Command | Purpose |
|---|---|
| `fmbot-pbt` | interactive session; ask for a test in plain language |
| `fmbot-pbt -p "<request>"` | one unattended run; the exit code is not a verdict |
| `fmbot-pbt mcp` | MCP server for Claude Code and Codex (mode 1) |
| `fmbot-pbt hook-run <sha>` | test one commit or one merge request; the exit code is the verdict (mode 2) |
| `fmbot-pbt build-run --build-cmd "<command>"` | run your build command first, then test |
| `fmbot-pbt arkts-sign` / `fmbot-pbt arkts-test` | local HAP signing / correlated device verdicts, no model call |
| `fmbot-pbt watch` | watch a branch and test each new commit |
| `fmbot-pbt resume` | continue an interrupted run |
| `fmbot-pbt finalize` | validate and render existing results without a model call |
| `fmbot-pbt scan` / `fmbot-pbt coverage` | list functions / summarize test progress, no model call |
| `fmbot-pbt dashboard` / `fmbot-pbt setup` | local web dashboard / interactive model setup |
| `fmbot-pbt skill-install` | install the `pi-pbt-dev` skill for a coding agent |
| `fmbot-pbt kea` | HarmonyOS on-device UI testing (experimental); see the [Kea guide](kea.md) |

Common flags: `--help` / `-h` (or `fmbot-pbt help <command>`) for usage; `--lang zh` for Chinese output; `--effort quick|standard|thorough` for [test depth](#depth); `--provider`, `--model`, `--thinking` to choose the model; `--target <dir|file>` for the code under test; `--out <dir>` for the output directory. Omit `--func` unless filtering to a single function.

## `hook-run`

```text
fmbot-pbt hook-run <sha> [--target <dir|file> [--func <name>]] [--whole] [--base <rev>] [--spec <file>]...
  [--build-cmd "<command>" [--build-dir <dir>]] [--out <dir>] [--scratch-dir <dir>]
  [--effort …] [--lang zh] [--cov-mode incremental|full] [--prompt <text> | --prompt-file <path>]
  [--provider <p>] [--model <m>] [--thinking <level>] [--mode text|json|json-full] [--no-coverage]
```

| Input | Flag | Meaning |
|---|---|---|
| Documents (optional) | `--spec <file>` | a requirement document whose explicit statements decide the verdict; repeatable; without it the intent is inferred from the code |
| Code | `<sha>` | the commit under test; a merge commit is compared with its first parent |
| | `--base <rev>` | the merge request's base commit; tests every change from it to `<sha>` instead |
| | `--target <dir\|file>`, `--func <name>` | directory or file under test (default `.`); its git repository must have `<sha>` checked out (use `fetch-depth: 0` in CI). `--func` selects one function |
| | `--whole` | test everything under `--target` at `<sha>`, not only the change; incompatible with `--base` |
| Build command (optional) | `--build-cmd "<command>"` | build the checkout with this command before testing; without it, build in CI first and fmbot-pbt reuses the output |
| | `--build-dir <dir>` | where the build command runs, default the repository of `--target`; must contain that repository or lie inside it |
| Other | `--scratch-dir <dir>` | disposable directory deleted and recreated per run; default `pbt-hook-work` next to the repository |

The checkout must already be at `<sha>` (mismatches exit `2` without switching branches). Omit `--cov-mode` to use `PBT_COVERAGE_MODE` (default `incremental`). A failing build exits `3` before testing, with output in `<out>/build.log`. `--out` and `--scratch-dir` may not contain the checkout, build directory, current directory or `--spec` files.

| Output | Meaning |
|---|---|
| `<out>/REPORT.md`, `REPORT.html` | the test report; `--out` defaults to `pbt-out-hook` next to the repository |
| `<out>/report.json` | the result for programs |
| `<out>/bug_reports/` | one report per bug |
| exit code `0` / `1` / `2` / `3` | no bug / bugs found / report missing or change set not computable / build failed |

## `build-run`

```text
fmbot-pbt build-run --build-cmd "<command>" [--build-dir <dir>] [--target <dir|file> [--func <name>]]
  [--diff <file> | --commit <sha>] [--spec <file>]... [--out <dir>] [--isolate [--isolate-test-sdk <version>]]
  [--effort …] [--lang zh] [--prompt <text> | --prompt-file <path>]
  [--no-auto-resume | --auto-resume-attempts <n> --auto-resume-wait <seconds>]
  [--provider <p>] [--model <m>] [--thinking <level>] [--mode text|json|json-full] [--no-coverage]
```

Runs your build command first and tests only if it succeeds; `--engine acp` runs in [Claude Code or codeagent](acp.md).

| Input | Flag | Meaning |
|---|---|---|
| Documents (optional) | `--spec <file>` | a requirement document whose explicit statements decide the verdict; repeatable; without it the intent is inferred from the code |
| Code | `--target <dir\|file>`, `--func <name>` | directory or file under test (default `.`); new tests are written in its repository. `--func` selects one function |
| | `--diff <file>`, `--commit <sha>` | limit testing to a patch file or commit; without them the run tests `--target` |
| Build command | `--build-cmd "<command>"` | required for a source campaign, naming a target that already exists and builds |
| | `--build-dir <dir>` | where the build command runs, default `.`; must share the source tree of `--target` |

| Output | Meaning |
|---|---|
| `<out>/build.log` | the build log; `--out` defaults to `<repo>/pbt-out` |
| `<out>/REPORT.md`, `report.json`, `bug_reports/` | the test results, as for `hook-run` |
| new tests in the project's test tree | registered in an existing test target; after generation the same command can build that target |
| exit code | `3` means the build failed; after a successful build the result is in `report.json` |

For OpenHarmony components prefer the native `host_product`; output is in `out/host/host_product`. Use a device product only when the component has no host target and a runner is configured for that device product. There is no single `--build-cmd` for the whole OpenHarmony tree: point `--target` at this component (or a directory or file in it), `--build-dir` at the tree root, and build an existing unittest of the component. Example:

```bash
export PBT_OH_WORKSPACE=/path/to/openharmony
fmbot-pbt build-run \
  --build-dir "$PBT_OH_WORKSPACE" \
  --target "$PBT_OH_WORKSPACE/foundation/arkui/ace_engine/frameworks/base/geometry" \
  --build-cmd "./build.sh --export-para PYCACHE_ENABLE:true --product-name host_product --build-target base_unittest --ccache --no-prebuilt-sdk"
```

A plain CMake project:

```bash
fmbot-pbt build-run --build-cmd "cmake -S . -B build && cmake --build build --target calc_test" --target src/calc.cpp
```

HarmonyOS apps: `--isolate` (with `--target <file.ets|file.ts> --func <name>`) tests one function in a generated mini HAP under `<out>/isolate-hap`; omit `--isolate` when the target imports OHPM/HAR packages or system APIs. ArkTS isolation, signing, pre-signed `--hap`/`--test-hap`, and `--recover-missing-cases` options: [ArkTS guide](pbt/arkts.md) and [ArkTS device guide](arkts-device.md).

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
| `target` | the code under test, a directory or file (required); its git repository is found from it |
| `spec` | requirement documents (optional), one path or a list; without them the intent is inferred from the code |
| `include_uncommitted` | include uncommitted changes |
| `revision` | commit to test, default `HEAD` |
| `base` | base commit; tests every change from it to `revision`; not with `whole` |
| `whole` | `true` tests everything under `target`, not only the change (one run per module is easier to read) |
| `effort`, `lang`, `provider`, `model` | as on the command line |
| `prompt` | extra instructions for the run |
| `build_cmd`, `build_dir` | build command and directory, as `hook-run --build-cmd`; runs in the original checkout at `revision` |

Without `build_cmd` the test runs in a separate git worktree. Results are kept in `~/.pi-pbt/runs/<run_id>/` (generated tests in `changes.patch`); read them via the URIs returned by `pbt_report`.

## Argument and path checks

`build-run`, `hook-run`, `watch` and MCP `pbt_start` / `pbt_watch_start` validate flags and paths before building, stopping (`build-run` and `hook-run` exit `2`; `watch` exits non-zero; MCP returns an error) with the flag, path and reason on unknown/removed options (`--repo`, `--scope`, `--workdir`, `--build-workdir`), missing or unwritable directories, missing `--diff` files, or unexecutable local build scripts.

## `watch`

```text
fmbot-pbt watch [--target <dir|file>] [--base <rev>] [--spec <file>]... [--interval <sec>] [--fetch] [--branch <name>] [--out <dir>] [--lang zh] [--effort …]
```

Runs `hook-run` for each new commit, oldest first (up to the newest 10 per poll; `--prompt` and `--prompt-file` are repeatable in order). With `--build-cmd`, only the checked-out `HEAD` is tested and `--fetch` is refused; to test un-checked-out commits in detached worktrees, use MCP `pbt_start` or `pbt_watch_start` without `build_cmd`.

## Skills in a session

In an interactive `fmbot-pbt` session, `/skill:pbt-build-run`, `/skill:pbt-hook-run` and `/skill:pbt-watch` take the arguments of `build-run`, `hook-run` and `watch` and run the same preparation, build gate and verdict (`0`–`3`); stop a watch with `stop_watch`.

Differences from the commands: the session keeps its working directory; `--mode json|json-full` is refused; `--provider`, `--model` and `--thinking` switch the session's model for the rest of the session (`--provider` needs `--model`).

## Other commands

| Command | Meaning |
|---|---|
| `fmbot-pbt resume [--out <dir>] [--attempts <n>] [--wait <sec>]` | continue an interrupted run |
| `fmbot-pbt finalize [--out <dir>] [--lang zh]` | validate and render existing results; `0` valid, `2` invalid |
| `fmbot-pbt scan [--dir <path>] [--out <dir>] [--lang <language>]` | list functions into `FUNCTION_INDEX.md` (pass a module's source directory via `--dir`); exits `1` on missing paths or incomplete `rg` scans. See [coverage tracking](coverage-tracking.md) |
| `fmbot-pbt coverage [--list \| --json] [--diff <sha>]` | summarize test progress in `pbt-out/`; for a commit, count matched definitions or fall back to indexed functions in that file |
| `fmbot-pbt dashboard [--port 8000]` / `fmbot-pbt setup` | local web dashboard / interactive wizard writing credentials and defaults to `~/.pi-pbt/agent/` |
| `fmbot-pbt skill-install [--host <agent>] [--dir <path>] [--force]` | install the `pi-pbt-dev` skill; see the [skill guide](skill-pi-pbt-dev.md) |

## Resume and offline completion

In a paused `--tui` run, `/continue` resumes in the same session and `/quit` saves state for shell recovery. `fmbot-pbt resume [--out <dir>] [--attempts <n>] [--wait <sec>] [--force]` continues an interrupted campaign under exclusive checkout/output ownership (default 1 launch; auto-retries apply only to confirmed provider stream aborts and stop after three unchanged aborts unless `--force` is given).

When recorded evidence is already complete, run `fmbot-pbt finalize --out <original-out>` (or copy the command printed at interruption) after the active campaign ends to validate artifacts and render `REPORT.md` without a model call or test rerun (`0` valid delivery, `2` incomplete; inspect `interruption.json`, `recovery.json` or `REPORT.incomplete.md` for blockers). ArkTS device receipt and lock recovery rules: [ArkTS device guide](arkts-device.md).

## Depth

| | `quick` | `standard` | `thorough` |
|---|---|---|---|
| Time budget | about 10 minutes | about 30 minutes | unlimited |
| Properties per test subject | 3–5 | 5–8 | as needed |
| Inputs per property | about 100 | ≥ 1000 | ≥ 10000 |

A **test subject** is one function or method, or a sequence of related operations; the count applies per subject, not per repository or build target. `hook-run` and `watch` default to `quick`, other entries to `standard`. Above 400 implementation files, a single repo-wide run cannot certify completeness (exit `2`); run separate module campaigns with `--target <module-dir>` and distinct `--out` directories. User-supplied `--spec` documents take precedence; code intent is inferred only where documents are absent or silent.

## Campaign phase

Model routing, design thinking, compaction and managed-run status share phase inference from campaign artifacts (`Scan` before artifacts exist, `Plan` while `PLAN.md` checklists or proposed designs are open, `Test` during execution, and `Review` once tests finish or `REPORT.md` is written).

## Environment variables

| Variable | Meaning |
|---|---|
| `PBT_TLS_EXCEPTIONS=off` | turn off [enterprise certificate exceptions](network.md) and package-tool TLS compatibility |
| `NODE_USE_SYSTEM_CA=0` | opt out of the default system CA addition; explicit runtime CA flags are respected |
| `NODE_EXTRA_CA_CERTS=/path/ca.pem` | add PEM CA certificates; set before starting fmbot-pbt; inherited by child processes |
| `PBT_LANG` | `zh` or `en`, output language |
| `PBT_ARKTS_DEVICE_OPTIONS` / `HDC`, `JAVA` | shared JSON ArkTS device policy (prefer CLI flags) / provisioned `hdc` and `java` paths |
| `PBT_EFFORT` / `PBT_SKIP_SETUP=1` | default depth / skip the automatic first-run interactive model setup prompt |
| `PBT_TEST_JOBS` | test and build parallelism: a number, `max`, or `auto` (default, idle cores) |
| `PBT_CODE_COVERAGE=0` | turn native code coverage off |
| `PBT_CLOSEOUT_REVIEW=off` | skip the independent review (run by the bundled pi-subagents); the report says it was not reviewed |
| `PBT_SCAN_DELEGATION=off` | do not use parallel scan subagents |
| `PBT_VERDICTS=off` / `PBT_CARRY=off` | do not record / do not reuse earlier verdicts |
| `PBT_OH_WORKSPACE` | the OpenHarmony source tree; detected automatically when run inside it |
| `PBT_PHASE_MODELS` / `PBT_DESIGN_THINKING` | model per phase, e.g. `{"plan":"anthropic/claude-opus-5-5"}` / Scan and Plan thinking level (default `xhigh`, `keep` keeps session level) |
| `PBT_MCP_MAX_CONCURRENT` | MCP tests running at once, default 1 |
| `PI_PBT_RUNS_DIR` | MCP results directory, default `~/.pi-pbt/runs` |
