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
| `fmbot-pbt arkts-sign` / `fmbot-pbt arkts-test` | local HAP signing / correlated device verdicts, no model call |
| `fmbot-pbt watch` | watch a branch and test each new commit |
| `fmbot-pbt resume` | continue an interrupted run |
| `fmbot-pbt finalize` | validate and render existing results without a model call |
| `fmbot-pbt scan` / `fmbot-pbt coverage` | list functions / summarize test progress, no model call |
| `fmbot-pbt dashboard` | local web dashboard |
| `fmbot-pbt skill-install` | install the `pi-pbt-dev` skill for a coding agent |
| `fmbot-pbt kea` | HarmonyOS on-device UI testing (experimental); see the [Kea guide](kea.md) |

Common flags: `--lang zh` for Chinese output; `--effort quick|standard|thorough` for depth; `--provider`, `--model`, `--thinking` to choose the model; `--target <dir|file>` for the code under test; `--out <dir>` for the output directory.

## `hook-run`

```text
fmbot-pbt hook-run <sha> [--target <dir|file> [--func <name>]] [--whole] [--base <rev>] [--spec <file>]...
  [--build-cmd "<command>" [--build-dir <dir>]] [--out <dir>] [--scratch-dir <dir>]
  [--effort …] [--lang zh] [--prompt <text> | --prompt-file <path>]
  [--provider <p>] [--model <m>] [--thinking <level>] [--mode text|json|json-full] [--no-coverage]
```

| Input | Flag | Meaning |
|---|---|---|
| Documents (optional) | `--spec <file>` | a requirement document whose explicit statements decide the verdict; repeatable; without it the intent is inferred from the code |
| Code | `<sha>` | the commit under test; a merge commit is compared with its first parent |
| | `--base <rev>` | the merge request's base commit; tests every change from it to `<sha>` instead |
| | `--target <dir\|file>`, `--func <name>` | the code under test, a directory or file, default the current directory; its git repository is found from it and must have `<sha>` checked out (use `fetch-depth: 0` in CI). A subdirectory or file narrows the run to it; `--func` to one function of the file. `--whole` tests everything under `--target` at `<sha>`, not only the change; the checkout must be at `<sha>`; not with `--base` |
| Build command (optional) | `--build-cmd "<command>"` | build the checkout with this command before testing; the campaign rebuilds with this command only. Without it, build in CI first and fmbot-pbt reuses the output |
| | `--build-dir <dir>` | where the build command runs, default the repository of `--target`; it must contain that repository or lie inside it |
| Other | `--scratch-dir <dir>` | a scratch directory that is deleted and recreated; default `pbt-hook-work` next to the repository. It is not the build directory |

With `--build-cmd` the checkout must already be at `<sha>` (never checked out for you; otherwise exit `2`); a failing build exits `3` before testing, output in `<out>/build.log`. `--out` and `--scratch-dir` may not contain the checkout, the build directory, the current directory or a `--spec` document, symbolic links included.

| Output | Meaning |
|---|---|
| `<out>/REPORT.md`, `REPORT.html` | the test report; `--out` defaults to `pbt-out-hook` next to the repository |
| `<out>/report.json` | the result for programs |
| `<out>/bug_reports/` | one report per bug |
| exit code `0` / `1` / `2` / `3` | no bug / bugs found / report missing or change set not computable / build failed |

## `build-run`

```text
fmbot-pbt build-run --build-cmd "<command>" [--build-dir <dir>] [--target <dir|file> [--func <name>]]
  [--diff <file> | --commit <sha>] [--spec <file>]... [--out <dir>] [--isolate] [--effort …] [--lang zh]
  [--prompt <text> | --prompt-file <path>] [--no-auto-resume | --recover-missing-cases [--recover-max-cases <n>] [--case-recovery-recipe <file.json>]] [--auto-resume-attempts <n>] [--auto-resume-wait <seconds>]
  [--provider <p>] [--model <m>] [--thinking <level>] [--mode text|json|json-full] [--no-coverage]
```

Runs your build command first and tests only if it succeeds. `--engine acp` runs it in [Claude Code](acp.md). Supervised headless Pi source runs can resume the same saved session after quota, rate-limit or provider interruptions, including confirmed provider stream aborts. `--auto-resume-attempts <n>` caps relaunches per invocation (1–10, default 2); the initial build does not count or repeat. `--auto-resume-wait <seconds>` sets the first wait (1–300 whole seconds, default 15); later waits double up to 300 seconds. Put each retry value immediately after its flag; a missing value stops the command before the build. Ctrl-C cancels the wait. In an active headless Pi campaign without a supervisor, Ctrl-C lets Pi finish its JSON output and run shutdown hooks; a second Ctrl-C or five seconds without finishing forces exit `130` and preserves the interruption. The retry options are rejected with `--no-auto-resume`, `--tui`, in-session skills, ACP, `--isolate`, pre-signed HAPs or bare mode. The [retry guards](#resume-and-offline-completion) still apply; delivery blockers are not retryable. On exhaustion, exit `2` with the saved interruption. When offline completion is ready, exit `2` with the `finalize` command instead of relaunching. With non-terminal stdin, eligible headless Pi source runs are supervised by default. With terminal stdin, `--auto-resume-attempts`, `--auto-resume-wait` or `--recover-missing-cases` opts into supervision without shell redirection. The worker and its build gate get EOF on stdin, while output stays connected as before. To use the default retry budget without specifying a recovery flag, redirect stdin with `< /dev/null` in POSIX shells or Git Bash, or `< NUL` in `cmd.exe`. On POSIX, omit the recovery flags and keep stdin attached if your build needs terminal input. For supervised runs, Ctrl-Z stops the supervisor and its worker group (including the initial build); `fg` or `bg` continues them. Ctrl-Z does not pause separately detached tools or work already running on a device. On native Windows, including when launched from Git Bash, the build command does not inherit stdin even without supervision. Use a build command that doesn't prompt.

| Input | Flag | Meaning |
|---|---|---|
| Documents (optional) | `--spec <file>` | a requirement document whose explicit statements decide the verdict; repeatable; without it the intent is inferred from the code |
| Code | `--target <dir\|file>`, `--func <name>` | the code under test, a directory or file, default the current directory; its git repository is found from it and receives the new tests (outside git the directory is the repository). A subdirectory or file narrows the run to it; `--func` to one function of the file |
| | `--diff <file>`, `--commit <sha>` | the change: a patch file or a commit; without them the run tests `--target`. A relative patch path is looked up in the current directory, then the repository of `--target`, then `--build-dir`; if it is in none, the error lists the paths checked |
| Build command | `--build-cmd "<command>"` | required for a source campaign, naming a target that already exists and builds |
| | `--build-dir <dir>` | where the build command runs, default the current directory; it must be in the same source tree as the repository of `--target` |

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

HarmonyOS apps: see the [ArkTS guide](pbt/arkts.md). `--isolate` requires `--target <file.ets|file.ts> --func <name>` (including `.ts` utilities under `src/main/ets/`) and generates `<out>/isolate-hap`. It copies the product graph into `entry/src/common` (no hard links or symlinks), retargets the suite import, and replaces `--build-cmd` with the generated HAP's ohosTest build after `ohpm install`. Copies and original dependencies are read-only and hash-checked; a modified or missing graph cannot attest passing/failing. The in-session skill uses this same preparation. Both isolate setup and the HAP build use an installed `hvigorw` from `PATH` or DevEco, not the SUT's project wrapper. The generated launcher forwards to the resolved executable in place; its tool directory is added to the campaign's `PATH` for `hvigorw` and `ohpm`. If the installed toolchain is missing, the usual deferred-build installation flow applies. Bare OHPM/HAR imports (for example `@hmos/xyz`) stop isolation before framework setup or compilation, with exit `3`. The diagnostic checks ancestor module/root `oh_modules` within the repository of `--target` and distinguishes a missing installation from an installed package whose HAR/module identity cannot yet be projected. In either case, omit `--isolate` to test in the original HAP. Do not stub or flatten the dependency. `PandaCheck` and `@ohos/hypium` remain available because the generated HAP supplies them. System-API STOPs are unchanged.

Keeping the JSON event log (stdout carries only events, diagnostics go to stderr; the result is still `report.json`):
```bash
set -o pipefail
LOG_DIR=/tmp/pbt-logs && mkdir -p "$LOG_DIR"
fmbot-pbt build-run --mode json \
  --build-cmd "./build.sh --product-name host_product --build-target base_unittest" \
  2>"$LOG_DIR/stderr.log" | tee "$LOG_DIR/events.jsonl"
```

For headless Pi ArkTS source runs, `--recover-missing-cases` opts into close-out for missing named receipts. If missing named receipts are the sole blocker, the worker attempts eligible cases sequentially under the existing locks and rechecks delivery without restarting the model. Set `--recover-max-cases <n>` to change the cap on eligible attempts (default `3`), independently of provider retries. Add [`--case-recovery-recipe <file.json>`](arkts-device.md#recovery-after-test-edits) to authorize one fresh build/sign attempt for stale gaps after eligible current routes are repaired, if the same cap and deadline still allow it. Without that option, close-out is device-only. See the [limits and stop conditions](arkts-device.md#device-results).
Device options, system signing, `arkts-sign`/`arkts-test`, pre-signed execution and lock recovery: [ArkTS device guide](arkts-device.md). Supplied archives execute their existing suite unchanged, not newly authored properties; both adapters return the same receipt-gated `0/1/2` verdict. See [recording failures](arkts-device.md#filing-failures-and-close-out) for how a headless whole-repo run can continue unrelated tests after an exhausted correction and still exit `2` for incomplete delivery.

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
| `build_cmd`, `build_dir` | a build command and its directory, as `hook-run --build-cmd`; the test then runs in the original checkout, which must be at `revision`; do not edit it until the run ends |

Without `build_cmd` the test runs in a separate git worktree. Results are kept in `~/.pi-pbt/runs/<run_id>/`; the generated tests are in its `changes.patch`.

## Argument and path checks

`build-run`, `hook-run` and `watch` check every input before building; anything unexpected stops them (`build-run`/`hook-run` exit `2`) naming the flag, the path and the reason, instead of leaving the model to guess. Refused: unknown flags; the removed `--repo`, `--scope` (use `--target`), `--workdir` (`build-run`: `--build-dir`; `hook-run`/`watch`: `--scratch-dir`), `--build-workdir` (use `--build-dir`) and `hook-run --target change|scope` (use `--whole`); a `--build-dir` that is missing or not readable and writable, a `./script` build command not in it or not executable; a `--target` that is missing or unreadable, a repository that cannot take the new tests, a `hook-run`/`watch` target outside git; an `--out` or `--scratch-dir` that cannot be created or written; a `--diff` that cannot be found. Write access is proven by creating and deleting a file. MCP `pbt_start` and `pbt_watch_start` refuse unknown fields the same way; the old `repo`, `scope` and `build_workdir` name `target` and `build_dir`.

## `watch`

```text
fmbot-pbt watch [--target <dir|file>] [--base <rev>] [--spec <file>]... [--interval <sec>] [--fetch] [--branch <name>] [--out <dir>] [--lang zh] [--effort …]
```

Runs `hook-run` for each new commit, oldest first, with any `hook-run` flag given here, validated before watching. At most the newest 10 new commits per poll are tested; use `hook-run` when every commit must be tested. With `--build-cmd` only the checked-out head is tested, and `--fetch` is refused.

## Skills in a session

In an interactive `fmbot-pbt` session, `/skill:pbt-build-run`, `/skill:pbt-hook-run` and `/skill:pbt-watch` take the arguments of `build-run`, `hook-run` and `watch` and run the same preparation, build gate and verdict (`0`–`3`); the test runs visibly in the session. Stop a watch with the `stop_watch` tool.

Differences from the commands: the session keeps its working directory; `--mode json|json-full` is refused; `--provider`, `--model` and `--thinking` switch the session's model for the rest of the session; `--provider` needs `--model`.

## Other commands

| Command | Meaning |
|---|---|
| `fmbot-pbt resume [--out <dir>] [--attempts <n>] [--wait <sec>]` | continue an interrupted run |
| `fmbot-pbt finalize [--out <dir>] [--lang zh]` | validate and render existing results; `0` valid, `2` invalid |
| `fmbot-pbt scan [--dir <path>] [--out <dir>] [--lang <language>]` | list functions into `FUNCTION_INDEX.md`; `--lang arkts` lists named `export function` definitions in `.ets` (including component files) as inventory only, not PBT candidates. An incomplete ArkTS scan reports diagnostics and exits `1`. |
| `fmbot-pbt coverage [--list \| --json]` | summarize test progress in `pbt-out/` |
| `fmbot-pbt dashboard [--port 8000]` | local web dashboard |
| `fmbot-pbt skill-install [--host <agent>] [--dir <path>] [--force]` | install the `pi-pbt-dev` skill; see the [skill guide](skill-pi-pbt-dev.md) |

## Resume and offline completion

In a Pi `build-run --tui` or `hook-run --tui`, an interrupted campaign with a saved session stays paused. Type `/continue` to continue in the same session with the same model and thinking settings. This does not rerun the build gate or schedule automatic resumes. While paused, switching or forking sessions is blocked; after `/reload`, quit and use the shell recovery command. If you need to change settings, address missing ArkTS case receipts, or the saved evidence is ready to finalize, follow the command's advice. `/quit` leaves the evidence saved and releases the checkout lease so you can use `resume` or `finalize` from the shell. `resume` acquires checkout/output ownership in the resumed runner itself before starting the agent; an overlapping resume or finalize exits 2. Its internal launch-only ownership marker is consumed before tools/subagents start, so descendants retain campaign context without competing with the parent's lease. CLI and in-session campaigns receive fresh runner/parent PIDs and exact lease ticket paths in their context. Common host cleanup commands targeting fmbot-pbt/hdc or the runner's own PIDs, including `Get-Process ... | Stop-Process` without trailing arguments, are blocked correctively: never kill these processes to recover a device lock. `resume` launches the saved campaign once by default. `--attempts` limits launches; `--wait` sets the delay between them. On POSIX, Ctrl-Z suspends the resumed runner along with the command; the shell's `fg` or `bg` continues both. Automatic retries stay with the original campaign and recheck the saved interruption and session under the runner's lease. Among aborts, only a positively identified provider stream abort can auto-retry; error text alone is not proof. `interruption.json` records its `abortOrigin`; operator signals, session cancel requests, policy stops, and unknown or legacy aborts do not auto-retry. Within a launch, Pi can retry HTTP `503` before `interruption.json` is written. An explicit model-access denial or credential rejection stops retries. Get model access or correct the credentials, then use `resume --force`; changing `--thinking` fixes neither. `resume-progress.json` tracks aborts across invocations against the saved campaign identity, recorded artifacts (including device indexes/receipts in either output layout) and referenced source/test bytes. Three consecutive aborts without changes stop further requests (exit 2): the original abort plus two resumed aborts, not three failed resumes. Inspect cancellation, proxy timeouts and provider context limits: an abort alone does not prove an oversized request. After investigation, `--force` authorizes one retry; it does not clear evidence or change models. Changed recorded work or a different campaign resets this comparison.

At the first abort or incomplete delivery, the terminal and `interruption.json` name the blockers or recommend `fmbot-pbt finalize --out <dir>` **after the active campaign ends**. Copy the full command: it points to the owning output directory and includes `--lang en` or `--lang zh` when the campaign recorded its language. The check is read-only. An explicit `resume` can still launch; if an interrupted attempt leaves the evidence ready, automatic retries stop with exit `2` and the offline command. Before resuming, headless `build-run` also verifies the saved transcript's bytes under the retry worker's lease. If only eligible missing receipts remain, the hint identifies the runs. Finalize acquires output/checkout ownership, validates fresh reports, design results, indexed device receipts, retained failures and the recorded close-out review (including its original opt-out/waiver), then renders without a model call or another device run. It checks `<out>`, `<out>/pbt-out`, and fresh reports in `pbt-out/` under the campaign's recorded working directory, provided the record tools bound the report to that campaign. It does not look in the directory from which you launch `finalize` unless the saved campaign ran there. Live, hook/session and offline delivery share design/result and device-evidence validation across the owned layouts: an empty nested ledger or a green report cannot hide unfinished owning designs or retained failures in another report/recovery sidecar. Unrelated ancestor artifacts are not borrowed. Success marks matching checkpoints completed and preserves matching interruptions as `interruption.completed-<hash>.json`; unrelated or invalid records are never cleared. A report with a campaign ID requires a matching checkpoint; use the original `--out` directory, not the checkout fallback path. A standalone report without a campaign ID or checkpoint can still be validated and rendered without closing a campaign. Exit `0` means valid delivery, not “no bugs”; `2` means incomplete. Schema-valid all-retired/failed-build reports require no device execution; actual unknown registrations still block, and the hook's build-failure verdict remains `3`. When a new campaign records into a reused output directory, the record tools keep the previous `report.json` and `REPORT.md` under `report.previous-*/`. They start a new report without inheriting old verdicts or review decisions. Checkout fallback delivery requires both fresh report files and a matching campaign checkpoint, including for ACP campaigns that have no Pi transcript.

The ArkTS conclusion and `COVERAGE.md` checks recognize execution from current, campaign-owned device receipts, including after resume. A named case from a campaign suite, including one authored directly in `List.test.ets`, proves execution only when its receipt matches the current source and test HAP. Registering other suites does not count as running them. A valid no-run report in an owned output layout allows coverage notes, even when `COVERAGE.md` is in another owned layout. Notes may use LF or CRLF; coverage rows must stay unchanged. Missing cases and untriaged failures still block delivery; a build or an `arkts-test` invocation alone does not prove execution. Missing receipts remain unknown execution, not passes. Malformed registrations (including blank target, bundle or module) and conflicting records for a current run ID block delivery. `finalize` checks all owned indexes before writing and leaves those runs' receipts untouched. Identical repeats in one index are not a conflict. Under exclusive campaign ownership, `finalize` can record a valid, unambiguous registration with a missing receipt as aborted (`code: 2`, `reason: parent-exit`) once it is at least 30 minutes old, only when its recorded host owner has exited (legacy indexes may lack an owner), no device lock exists, and preserved logs contain no failure evidence. It retains the index/logs and records `abortedRuns` in `findings-unrecorded.json`; recent/live/unknown-owner, malformed and failing evidence stays blocking. This does not replace required verified device execution. Otherwise inspect the original run before an explicitly requested `arkts-test --supersedes-run <id>`; never prune its index. Absent/already-resolved links produce retained warnings, while mismatched identities, future/self references and genuinely failing targets remain errors. Lock-denied attempts are non-execution, never a verified result; stale-lock recovery uses the shared [device runner](arkts-device.md). A prior green receipt is not permission to reuse changed source, tests, executables or environment. For new/changed properties, select affected classes/cases and widen only when shared inputs require it. `finalize` delivers saved results; it does not claim fresh execution or new coverage.

## Depth

| | `quick` | `standard` | `thorough` |
|---|---|---|---|
| Time budget | about 10 minutes | about 30 minutes | unlimited |
| Properties per test subject | 3–5 | 5–8 | as needed |
| Inputs per property | about 100 | ≥ 1000 | ≥ 10000 |

A **test subject** is usually one function or method, or a group of related operations tested in sequence, such as opening an account, depositing, withdrawing, and closing it. The count applies to each subject, not to an entire repository or module, and does not refer to a build target. `hook-run` and `watch` default to `quick`, other entries to `standard`. Every test entry reads user-provided requirement, API or design documents before mapping them to the code and deriving properties. Intent is inferred from code only where documents are absent or silent.

## Campaign phase

Model routing, design thinking, compaction and managed-run status share phase inference. Proposed property designs reopen Plan, even after a report. Otherwise `REPORT.md` means Review; before it exists, a complete `PLAN.md` checklist selects the first unchecked phase of the first unfinished module. Completing Test therefore enters Review before the report is written. Fenced examples and quotations do not count. Absent, partial or unreadable checklists fall back to artifact presence. A carried-over ledger without a plan does not finish Plan. Before any artifacts exist, model routing waits and managed runs display Scan.

## Environment variables

| Variable | Meaning |
|---|---|
| `PBT_TLS_EXCEPTIONS=off` | turn off [enterprise certificate exceptions](network.md) and package-tool TLS compatibility |
| `NODE_USE_SYSTEM_CA=0` | opt out of the default system CA addition; explicit runtime CA flags are respected |
| `NODE_EXTRA_CA_CERTS=/path/ca.pem` | add PEM CA certificates; set before starting fmbot-pbt; inherited by child processes |
| `PBT_LANG` | `zh` or `en`, output language |
| `PBT_ARKTS_DEVICE_OPTIONS` | JSON device/result/signing policy shared by `build-run`, its skill and device helpers; prefer CLI flags |
| `HDC`, `JAVA` | optional provisioned executable paths for device transport / local SDK signing |
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
