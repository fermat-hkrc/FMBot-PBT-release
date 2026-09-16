# pi-pbt subcommands

[中文版](subcommands.zh.md)

Every campaign launches through the **`pi-pbt` binary**. A bare
`pi-pbt -p "…"` is one free-form headless run; it is not a CI verdict. Use
`hook-run` when the process exit code must decide a job, and use `watch` only as
a best-effort resident monitor.

`--repo` / `--workdir` default to the current directory. Omit them when you
already `cd` into the module.

| I want to… | Command |
|---|---|
| Interactive campaign | `pi-pbt` |
| One headless run (script / nohup, not a gate) | `pi-pbt -p "…"` |
| Run a known build before a campaign | `pi-pbt build-run --build-cmd "…"` |
| Gate **one git commit** (hook / CI) | `pi-pbt hook-run <sha>` |
| Monitor new commits (best effort) | `pi-pbt watch` |
| Simulate a developer (no PBT) | `pi-pbt replay --pr <n>` |
| List functions, no LLM | `pi-pbt scan` |
| PBT every scan candidate | `pi-pbt test-all` |
| Print coverage from `pbt-out/` | `pi-pbt coverage` |
| Findings dashboard | `pi-pbt dashboard` |
| MCP for Claude Code / Codex | `pi-pbt mcp` |
| Version | `pi-pbt --version` |

Provider/model selection is not limited to `build-run`: `--provider` and
`--model` are accepted by free-form `pi-pbt`, `build-run`, `hook-run`, `watch`,
and `test-all`; MCP `pbt_start` / `pbt_watch_start` expose matching `provider`
and `model` fields. `replay` accepts `--model` only. Kea reads `provider` and
`model` from `kea.config.yml`. Omit overrides to use `~/.pi-pbt/agent/`.

`kea` is **experimental** GUI testing on a HarmonyOS phone; see the
[Kea guide](kea.md) for prerequisites, configuration and commands.

---

## Every way to run it

One binary, thirteen entry points. Pick by what you want to happen; the details
of each are below.

| | Command | What it does |
|---|---|---|
| **Run a campaign** | `pi-pbt` | Interactive. Describe the target in chat. |
| | `pi-pbt -p "<prompt>"` | One campaign, headless. For scripts and unattended generation — **not a CI gate**: the process exit code is pi's, not a verdict, so bugs found do not fail the job. Use `hook-run` for that. |
| | `pi-pbt build-run --build-cmd "<cmd>"` | Your build command is the gate: it runs first, and a non-zero exit stops the campaign before any agent work. The campaign then runs **in place** — no worktree — because large components cannot build from a detached copy. |
| | `pi-pbt hook-run <sha>` | One commit's change set. **The exit code is the verdict**: `0` clean, `1` bugs found, `2` no report or a report that fails the schema, `3` the build failed. The sha selects the diff and the prompt — **the caller checks the tree out**. |
| | `pi-pbt watch` | Polls a branch and runs one campaign per new commit, oldest first — **newest 10 per poll**, older ones dropped (and logged). |
| | `pi-pbt test-all` | Reads `pbt-out/FUNCTION_INDEX.md` and campaigns over every candidate function in it. |
| **Let another agent drive** | `pi-pbt mcp` | MCP server on stdio: six tools, plus the artifacts as `pbt://` resources. |
| | `pi-pbt skill-install` / `skill-uninstall` | Installs the bundled `pi-pbt-dev` skill into a host agent's skill directory. |
| **Local only, no model calls** | `pi-pbt scan` | Ripgrep function scan → `FUNCTION_INDEX.md`. No LLM, no agent. |
| | `pi-pbt coverage` | Prints the coverage report from existing `pbt-out/` artifacts. Never starts an agent. |
| | `pi-pbt dashboard` | Web dashboard for runs on this machine. |
| **GUI PBT (experimental)** | `pi-pbt kea --config kea.config.yml` | HarmonyOS real-device GUI PBT via Kea2 — properties over app UI state on a phone, not source-level properties. Demo slice today. |
| **Not PBT** | `pi-pbt replay --pr <n>` | Simulates a developer implementing one sub-requirement and committing it. Hard-isolated: emits no PBT artifact. |

Two that are easy to confuse:

- **`build-run` vs `hook-run`.** `build-run` is for a target whose build you
  already know — you hand it the command, and it gates the campaign on that
  command succeeding. `hook-run` is for CI on one commit — it cares about the
  exit code. They do **not** compose in one invocation: `hook-run` takes no
  `--build-cmd`, and the campaign decides for itself how to compile. If your
  repository needs a specific build command, either build it in the CI step
  before `hook-run` (a warm build tree is what the campaign then reuses) or use
  `build-run` and read its exit code instead.
- **`-p` vs `mcp`.** `-p` runs the campaign in this process and blocks.
  `mcp` hands the campaign to a separate process and returns a run id
  immediately, so the caller keeps working while it runs.

## Skills: three different things share the word

| | What | How you get it | How it is invoked |
|---|---|---|---|
| **1. Bundled skills** | 15 SOP documents that drive the campaign itself — `pbt-workflow` (the four phases), `pbt-oracles`, `pbt-patterns`, `openharmony-build-run`, … | Ship **inside the binary**, next to it as `dist/skills/`. You never install them. | The agent loads them automatically. A few are also callable by name in a `pi-pbt` session: `/skill:pbt-workflow`, `/skill:pbt-build-run`, `/skill:pbt-watch`, `/skill:kea-harmony-app`. |
| **2. The `pi-pbt-dev` skill** | Teaches **your everyday coding agent** (Claude Code, Codex, opencode, codeagent, chrys) to run PBT by calling the `pi-pbt` binary. | `pi-pbt skill-install` — copies it out of the distribution into the host agent's skill directory. | Your host agent picks it up; ask it in plain language ("run PBT on this change"). Codex needs `$pi-pbt-dev` named explicitly. |
| **3. MCP tools** | Not a skill at all — six tools plus `pbt://` resources. The alternative to 2 when you want the campaign in a **separate process**. | `claude mcp add --transport stdio pi-pbt -- pi-pbt mcp` | The host agent calls `pbt_start` / `pbt_status` / `pbt_report`. |

**Which of 2 and 3 do I want?** The skill makes your agent run the campaign and
wait for it. MCP hands the campaign to a separate process and returns a run id
immediately, so you keep working while it runs. They do not conflict — install
both if you like.

**You do not install the bundled skills.** They are versioned with the binary,
and a mismatched hand-copied `SKILL.md` on a host is the classic way a campaign
starts behaving differently from what the code enforces.

Full detail: [skill-pi-pbt-dev.md](skill-pi-pbt-dev.md) for the host-agent
skill, [installation.md](installation.md) for MCP registration.

## Which mode at which stage

A campaign is not fast — measured on one ArkUI ace_engine module at the `quick`
tier it takes 20–30 minutes. That single fact decides most of this table.

### While developing, next to another coding agent

You are writing code in Claude Code / Codex / opencode and want the change
tested. A campaign takes 20–30 minutes, so the question is whether you can keep
typing while it runs. **That depends on one flag**, not on which tool you use:

| Mode | Where the campaign runs | Can you keep editing? |
|---|---|---|
| `pbt_start` **without** `build_cmd` | a detached `git worktree` on an immutable commit | **Yes** |
| `pbt_start` **with** `build_cmd` | **in your checkout, in place** | **No** — the run emits `do not edit the tree until this run finishes` |
| `pi-pbt-dev` skill (in-session) | your agent's own session, blocking | No — you are waiting on it |

In-place is not a flaw: a large component resolves paths, generated headers and
ccache state against the real checkout and cannot build from a detached copy.
But it means "delegate and keep coding" is only true in worktree mode. With a
build command, delegate and go do something else — or work in a second checkout.

#### The concrete flow in Claude Code

Register once:

```bash
claude mcp add --transport stdio pi-pbt -- pi-pbt mcp
```

Then, in the session, ask in plain language; the agent calls the tool:

```
Start a pi-pbt run on /path/to/repo, scope src/parser, include_uncommitted true
```

`pbt_start` returns immediately with a `run_id`. **Pass
`include_uncommitted: true` while developing** — the default is `HEAD`, so
without it you test the code you have not changed. It freezes the working tree
into a snapshot commit; your index, `HEAD`, refs and files are untouched.

Keep coding. Later:

```
Check that pbt run          →  pbt_status  (phase, status, queue position)
Read the result             →  pbt_report  (verdict, REPORT.md snippet, bug URIs)
```

#### What to do with the result

`pbt_report` is a summary. The artifacts are MCP **resources**:

| Resource | For |
|---|---|
| `pbt://runs/<id>/REPORT.md` | the human report |
| `pbt://runs/<id>/report.json` | the machine-readable result |
| `pbt://runs/<id>/bug_reports/<file>.md` | one file per bug |
| `pbt://runs/<id>/changes.patch` | **the generated tests** |

`changes.patch` is the part people miss. In worktree mode the campaign wrote its
tests inside a worktree that is **deleted when the run ends**, so the generated
tests are never in your checkout — the patch is how they get there. It is a
resource, and also a file on disk under the run directory:

```bash
# ~/.pi-pbt/runs/<run_id>/changes.patch  (PI_PBT_RUNS_DIR overrides the root)
cd /path/to/repo
git apply --stat ~/.pi-pbt/runs/<run_id>/changes.patch   # what it touches
git apply        ~/.pi-pbt/runs/<run_id>/changes.patch   # then review and commit
```

Apply it at the revision the run reports (`report.json`'s `run.revision`): the
patch is a binary diff against exactly that commit. Ask the agent for it in
plain language if you prefer — "apply the changes.patch from that run to my
checkout" — it can read the resource and write the files.

In-place runs (`build_cmd` set) produce no patch — the tests are already in your
checkout.

#### Using `build-run` from another agent

Three routes, none of which need the skill:

- **MCP**: `pbt_start` with `build_cmd` (and `build_workdir` when the build runs
  above the module, e.g. an OS source root while the scope is one component).
- **The `pi-pbt-dev` skill**: tell the agent the build command; the skill routes
  it to `pi-pbt build-run`.
- **Directly**: it is a CLI — any agent that can run a shell command can run
  `pi-pbt build-run --build-cmd "<cmd>"`.

Two things that keep the loop tight: scope the campaign (`scope` / `--scope`, or
a function-scoped prompt) instead of pointing it at a whole repository, and keep
`INVARIANTS.md` from the previous campaign around — the next one reads it and
does not rediscover what you already confirmed.

### In an MR / CI pipeline

| Situation | Use | Why |
|---|---|---|
| The gate on a merge request | **`hook-run <sha>`** | Built for exactly this, and **the exit code is the verdict** — `0` clean, `1` bugs found, `2` no report or a report that fails the schema, `3` the build failed. Nothing else to interpret. Check the commit out first: the sha selects the diff and the prompt, not the tree (below). |
| The repository needs a specific build command | **Build it in the step before** | `hook-run` has no `--build-cmd`; the campaign compiles the SUT itself and reuses a warm build tree. Run your CI's own build first, then `hook-run`. Where the build must *gate* the campaign, use `build-run --build-cmd` and read its exit code instead. |
| Machine-readable result for a bot or dashboard | **`pbt-out/report.json`** | Schema-validated, with every bug linked to the property that found it and pasteable reproduction commands. See [report-schema.md](report-schema.md). |
| What to attach to the MR | `REPORT.md` + `report.json` | Not the whole `pbt-out/` — that is campaign scratch, see [reproducing.md](reproducing.md). |

Do **not** put these in a pipeline: `watch` (long-lived by design), `dashboard`
(a UI), the interactive mode, and `replay` (a developer simulation, not PBT).

### After the campaign, whichever stage

The tests it wrote are ordinary tests in the project's own harness. Commit them
and their build-file registration, and from then on they run in CI with
everything else — no pi-pbt involved. [reproducing.md](reproducing.md) covers
what to commit, what not to, and how to replay a reported failure.

## `pi-pbt` / `pi-pbt -p`

Interactive TUI, or headless one-shot. The agent **chooses** how to compile
unless you use `build-run`.

```bash
cd /path/to/repo
pi-pbt -p "Run property-based testing; write results to pbt-out/."
```

Pin the SOP:

```bash
pi-pbt -p "/skill:pbt-workflow 对当前仓库做性质测试,产物写到 pbt-out/。"
```

---

## `build-run` — your build command gates PBT

```text
pi-pbt build-run --build-cmd "<command>"
  [--workdir <dir>] [--repo <path>] [--out <dir>]
  [--scope <path>] [--func <name>]
  [--lang zh] [--scan-root <dir>]
  [--effort quick|standard|thorough]
  [--provider <p>] [--model <m>] [--tui]
```

1. Runs `--build-cmd` in `--workdir` (log: `<repo>/pbt-out/build.log`).
2. Non-zero → **exit 3**, PBT never starts.
3. Zero → campaign in `--repo`. Rebuilds may **only reuse that command**
   (replacing its existing target with the generated test target when needed).
   No build-system switch and no derived alternative build.

`--scope` is a **path** (file or directory). `--func` is one **symbol**; it
**requires `--scope` and must appear after `--scope`**. Without `--func`, every
PBT-worthy function in `--scope` is in play.

`--workdir` = where the build runs. `--repo` = module under test (`pbt-out/`,
`--scope`). Same directory (typical CMake): `cd` there and omit both.

**OpenHarmony: `host_product` first.** This real ArkUI `ace_engine` preflight
builds the existing `base_unittest` group before the campaign writes tests:

```bash
export PBT_OH_WORKSPACE=/path/to/openharmony
pi-pbt build-run --provider xai --model grok-4.6 --lang en \
  --workdir "$PBT_OH_WORKSPACE" \
  --repo "$PBT_OH_WORKSPACE/foundation/arkui/ace_engine" \
  --build-cmd "./build.sh --export-para PYCACHE_ENABLE:true --product-name host_product --build-target base_unittest --ccache --no-prebuilt-sdk" \
  --scope frameworks/base/geometry
```

The host output is `out/host/host_product`, not `out/host_product`. The initial
preflight must use a target that already exists; only after the campaign has
generated and registered its tests may the command switch to the new `pbt`
target. `host_product` is not universal: use a device product only when the
component has no host target and the workspace already has a runner configured
for the resulting artifact.

**Ordinary CMake project.** This is a separate example of `build-run`'s general
build support, not an OpenHarmony-equivalent path. If the existing
`CMakeLists.txt` defines `calc_test`, gate on it:

```bash
cd /path/to/cmake-project
pi-pbt build-run --provider xai --model grok-4.6 --lang en \
  --build-cmd "cmake -S . -B build && cmake --build build --target calc_test -j$(nproc)" \
  --scope src/calc.cpp
```

If the project has other broken default targets, explicitly naming the known-good
existing target keeps the preflight bounded. After test generation, the campaign
may replace `calc_test` with the registered PBT target while preserving the same
CMake command structure.

Details: [installation.md](installation.md#build-first-then-test-build-run-bring-your-own-build-command).

---

## `hook-run` — one commit (CI / git hook)

```text
pi-pbt hook-run <sha>
  [--repo <path>] [--out <dir>] [--workdir <dir>] [--spec <file>]
  [--lang zh] [--scan-root <dir>]
  [--effort quick|standard|thorough]
  [--scope <path>] [--run-id <id>] [--tui]
```

Cleans workdir + `pbt-out`, scans, runs PBT on that commit's change set.
Default effort **quick**.

Exit codes — this is the whole interface for a CI job:

| | Meaning |
|---|---|
| `0` | Clean: a report that satisfies the schema, and no bugs. |
| `1` | Bugs found (`bug_reports/`, or `totals.bugs > 0`). |
| `2` | No `REPORT.md`, no `report.json`, or a `report.json` that fails the schema — the campaign died or attested to nothing. |
| `3` | The recorded build failed: nothing was tested. |

**The sha does not check anything out.** It selects the change set
(`git diff-tree <sha>`) and goes into the prompt (`git show <sha>`); the code
the campaign compiles and tests is whatever is in `--repo`'s working tree. So
the caller is responsible for putting the tree on that commit — in CI, the
checkout step already has:

```bash
# From a post-commit hook: nothing extra — the new commit IS the working tree.
pi-pbt hook-run <sha> --repo /path/to/repo --lang zh

# In CI: check it out into the disposable workspace, with its PARENT present
# (fetch-depth: 2 or 0 — a depth-1 clone makes `git diff-tree <sha>` empty).
git -C /path/to/worktree checkout --detach <sha>
pi-pbt hook-run <sha> --repo /path/to/worktree
```

Running it against a tree at a different revision is not an error and will not
warn — it will faithfully test the code that is there, against the diff of a
commit that is not.

If the repo is an OpenHarmony module, set `PBT_OH_WORKSPACE` or sit inside a
tree with `.repo/` + `out/` (auto-detect).

---

## `watch` — every new commit

```text
pi-pbt watch
  [--repo <path>] [--interval <sec>] [--fetch] [--branch <name>]
  [--out <dir>] [--workdir <dir>] [--spec <file>]
  [--lang zh] [--scan-root <dir>]
  [--cov-mode incremental|full] [--effort …] [--tui]
```

Spawns `hook-run` per new commit, oldest first. Default effort **quick**.

The baseline is the head at startup: commits that already existed are not
tested. A poll that finds more than **10** new commits — a burst, a rebase, or
a queue that built up while the previous 20–30 minute campaign ran — tests the
**newest 10** and drops the older ones, logging what it dropped. So `watch` is
a monitor for a branch that moves at a human pace; it is not a guarantee that
every commit was tested. When that guarantee is what you need, call `hook-run`
per commit from the hook or the pipeline, where nothing is dropped.

```bash
pi-pbt watch --repo /path/to/repo --lang zh --interval 30
pi-pbt watch --repo /path/to/repo --fetch --branch master --interval 60
```

---

## `replay` — fake a developer (not PBT)

```text
pi-pbt replay (--pr <n> | --commit <sha>)
  [--repo <path>] [--project owner/repo]
  [--lang zh] [--scan-root <dir>] [--model <name>]
```

Implements a change only. Sets `PBT_REPLAY=1` so **no** PBT campaign, no
PLAN/REPORT. PBT belongs on `hook-run` of the resulting commit.

```bash
pi-pbt replay --pr 8721 --lang zh --scan-root /path/to/workspace
```

---

## `scan` / `test-all` / `coverage`

No LLM for `scan` / `coverage`.

```text
pi-pbt scan [--dir <path>] [--out <dir>] [--lang <id>] [--module <name>]
pi-pbt test-all [--dir <path>] [--out <dir>] [--provider <p>] [--model <m>]
pi-pbt coverage
```

`scan` writes `pbt-out/FUNCTION_INDEX.md`. Language detection only looks two
directory levels down; on a large OpenHarmony module `pi-pbt scan` at the repo
root can print `no functions found`. Point `--dir` at the source subtree:

```bash
cd /path/to/telephony_core_service
pi-pbt scan --dir utils/codec --out ./pbt-out
```

`test-all` PBT-campaigns every candidate in that index (scans first if missing).

---

## `dashboard` / `mcp` / `skill-install`

```text
pi-pbt dashboard [install] [--port 8000]
pi-pbt mcp
pi-pbt skill-install [--host claude|opencode|codex|codeagent|chrys|project] [--dir <path>] [--force]
pi-pbt skill-uninstall [--host claude|opencode|codex|codeagent|chrys|project] [--dir <path>]
```

MCP: [installation.md](installation.md#delegation-from-another-coding-agent-mcp).
Dashboard: [installation guide](installation.md#5-dashboard-watch-it-work-live).

---

## Env (also CLI on some commands)

| Env | Meaning |
|---|---|
| `PBT_LANG` | Output language (`zh` / `en`); also `--lang` |
| `PBT_OH_WORKSPACE` | Full OH tree (`.repo/` + `out/`) used by official `host_product` or configured device-product builds. Unrelated ordinary CMake projects do not set it. |
| `PBT_EFFORT` | `quick` / `standard` / `thorough` |
| `PBT_SCAN_ROOT` | Scan directory; also `--scan-root` |
| `PBT_HOOK_TUI=1` | TUI for `hook-run` / `watch` |

Hook-run: [CI / git-hook integration](installation.md#ci--git-hook-integration). Coverage:
[coverage-tracking.md](coverage-tracking.md).
