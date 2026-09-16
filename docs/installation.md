# Installing PBT Agent

[中文版 / Chinese version](installation.zh.md)

pi-pbt is an AI agent: point it at a repository and it reads the code, works out
**what should always be true of it**, writes tests that check those rules against
large numbers of generated inputs, and reports the bugs it finds with a minimal
reproducer for each. That style of testing is called **property-based testing**
(PBT).

Installing it means downloading **one executable file**: no Node.js, no Bun, no
dependencies to install, and nothing to put next to it.

## 1. System requirements

pi-pbt has no dependencies of its own. But it **really compiles and runs the
test code it writes**, so the toolchain for the language under test must be
present:

| You test | Install first |
|---|---|
| Any repo | `git` (it reads the commit history and diffs) |
| C / C++ | `cmake` ≥ 3.16, a C++17 compiler (`clang`/`g++`), `make` or `ninja`. The test libraries (GoogleTest/RapidCheck) are downloaded by CMake, so the first build needs network access. |
| Python | `python3` ≥ 3.9 with `pip` (`hypothesis` is installed into the environment it finds) |
| Rust | `cargo` (`proptest` is added as a dev-dependency) |
| Go | `go` toolchain |
| Java | JDK + Maven/Gradle (jqwik) |
| OpenHarmony components | A provisioned OpenHarmony source tree and its official build prerequisites. Prefer native `host_product` tests; a device-product runner is needed only for components with no host target. |
| A HarmonyOS app on a real phone | `hdc` (HarmonyOS device connector) and a Kea2 checkout with its virtualenv — see [`pi-pbt kea`](#a-harmonyos-app-on-a-phone-pi-pbt-kea) |

Debian/Ubuntu example for C++ targets:

```bash
sudo apt-get install -y git cmake ninja-build clang
```

Arch Linux:

```bash
sudo pacman -S --needed git cmake ninja clang
```

## 2. Download and install

Obtain the archive for your platform from wherever you received pi-pbt (a
Releases page, an internal mirror, or a direct handoff). Each ships with a
`.sha256` checksum alongside it. The published files are:

| Platform | File | Download |
|---|---|---|
| Linux x64 | `pi-pbt-linux-x64.zip` | ~37 MiB |
| Linux arm64 (aarch64) | `pi-pbt-linux-arm64.zip` | ~37 MiB |
| macOS Apple Silicon | `pi-pbt-macos-arm64.zip` | ~26 MiB |

Not sure which Linux one you need? `uname -m` — `x86_64` takes the x64 file,
`aarch64` the arm64 one.

Every archive extracts to a same-named directory containing `pi-pbt`, an
`install.sh` installer, and adjacent `tools/fd` and `tools/rg`. Run the bundled
installer to install the main executable and place its tools where the embedded
pi tool manager checks them:

```bash
PLATFORM=linux-x64   # or linux-arm64 / macos-arm64
sha256sum -c "pi-pbt-${PLATFORM}.zip.sha256"   # optional integrity check
unzip "pi-pbt-${PLATFORM}.zip"
cd "pi-pbt-${PLATFORM}"
./install.sh
```

Running `fd` or `rg` directly from a regular shell may still report command not
found, which is expected; no additional `PATH` change is required. Extracting
on Windows drops Unix permissions, so if the files travelled through a Windows
machine, run `chmod +x` on `pi-pbt`, `install.sh`, `tools/fd`, and `tools/rg`
first.

The arm64 and macOS builds are cross-compiled on an x64 Linux machine; they are
validated by format (the release job asserts each is really an aarch64 ELF /
Mach-O arm64), not executed on the target. Command-line use is unaffected; if
the interactive interface misbehaves, build from source on that machine
(`npm run build:binary`).

macOS additionally needs the Gatekeeper quarantine cleared:

```bash
xattr -d com.apple.quarantine pi-pbt
```

Verify:

```bash
pi-pbt --help          # prints usage
pi-pbt --list-models   # lists available models once a provider is configured
```

On first run pi-pbt creates its own config directory `~/.pi-pbt/agent/` (login
credentials, model config, history) — deliberately separate from an upstream
`pi` install's `~/.pi/agent/`, so the two do not interfere.

### Install the `pi-pbt-dev` skill (optional)

The distribution carries a skill that lets your everyday coding agent (Claude
Code, opencode, Codex) run **full or incremental PBT campaigns through pi-pbt**.
The skill files ship inside the distribution itself (embedded in the binary), so
installing it is one command:

```bash
pi-pbt skill-install                 # auto-detects which agents are installed
pi-pbt skill-install --host claude   # or opencode / codex / codeagent / chrys / project
pi-pbt skill-install --dir /custom/path  # a custom skill directory
pi-pbt skill-uninstall               # remove it again (same flags)
```

It copies the `pi-pbt-dev` skill (protocol + helper scripts) into the host
agent's skill directory — a thin shell that locates the `pi-pbt` binary on PATH
at run time, so nothing else is needed. Re-run `skill-install --force` after
updating pi-pbt to refresh it; `skill-uninstall` removes it from every location
it was installed into. Then just tell your agent "test the changes I just made" —
full tutorial in `docs/skill-pi-pbt-dev.md`.

## 3. Configure a model

### First, the part that matters most: use the strongest model you have

This matters more than any other setting. The hardest step of the whole process
is **coming up with the properties** — working out, from the code, what must
hold no matter what input it gets. There is no lookup answer for that; it is
pure reasoning:

- A **strong model** produces properties with real teeth ("encoding then decoding
  must return the original value", "no interleaving of calls can drive the balance
  negative"). Only properties like that can catch real bugs.
- A **weak model** produces empty ones ("calling it does not crash"). The tests
  pass, the report looks fine, and nothing was found — and it is hard to tell
  from the output that nothing was really tested.

Deciding what counts as the correct answer, and working out whether a failure is
a bug in the code or in the test, lean on reasoning just as heavily. So **do not
economize here**: reasoning you save is bugs you miss.

### How to configure it

pi-pbt uses [pi](https://github.com/earendil-works/pi) as its engine, so models
and providers are configured exactly as in pi — with one difference: **the
config directory is `~/.pi-pbt/agent/` instead of `~/.pi/agent/`**. Wherever
pi's docs say `~/.pi/agent/<file>`, read `~/.pi-pbt/agent/<file>`.

The two quickest paths:

**API key via environment variable** — works immediately:

```bash
export ANTHROPIC_API_KEY=sk-ant-...   # or OPENAI_API_KEY, GEMINI_API_KEY, ...
pi-pbt --list-models
```

**Interactive login** — start `pi-pbt`, then use `/login` to sign in (including
subscription accounts) and `/model` (Ctrl+L) to pick a model.

For everything else, follow pi's documentation directly:

- [Providers & authentication](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/providers.md) — built-in providers, API keys, subscription login
- [Models & `models.json`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/models.md) — adding custom models or providers that speak an OpenAI/Anthropic/Google-compatible API (file goes in `~/.pi-pbt/agent/models.json`)
- [Custom providers](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/custom-provider.md) — custom APIs or OAuth

Example `~/.pi-pbt/agent/models.json` for a self-hosted OpenAI-compatible proxy:

```json
{
  "providers": {
    "myproxy": {
      "api": "openai-completions",
      "baseUrl": "https://my-proxy.example.com/v1",
      "apiKey": "sk-...",
      "models": [
        { "id": "claude-opus-4-8", "name": "Claude Opus 4.8", "reasoning": true,
          "contextWindow": 1000000, "maxTokens": 32000 }
      ]
    }
  }
}
```

## 4. Start testing

**Every way to run it, and which one suits which stage** — development
alongside another coding agent, or an MR pipeline — is one table in
[subcommands.md](subcommands.md#every-way-to-run-it). The three different
things called "skill" are disentangled in the same document. What follows here
is the first run.


### Interactive

`cd` into the repository under test and start it:

```bash
cd /path/to/your/repo
pi-pbt
```

Then just ask:

```text
Run property-based testing on this repository: identify the best targets, write
properties, run them, and produce bug reports.
```

It works through four steps — scan the code for targets, decide what to test,
write and run the tests, review the results — and writes what it produces to
`pbt-out/`: the plan (`PLAN.md`), the list of properties (`PROPERTIES.md`), a
summary (`REPORT.md`), and one `bug_reports/*.md` per confirmed bug, each with a
minimal reproducer.

All subcommands (`build-run`, `hook-run`, `watch`, `replay`, `scan`, …) are listed
in [subcommands.md](subcommands.md) ([中文](subcommands.zh.md)).

### From the command line (CI, scripts)

```bash
pi-pbt -p "Run property-based testing on this repository; write results to pbt-out/."
```

`-p` prints the run and exits; it never waits for input, so it is safe under CI
runners, git hooks, and `nohup`.

You do not pick a mode per project type: there is one entry point, and it works
the rest out itself. When the target does not build with a plain
`cmake`/`cargo`/`pytest` — a component of a large OS or of a monorepo — it stands
that component up first. OpenHarmony C++ components follow their official build
path, checking native `host_product` tests first. A device product is only a
fallback when the component has no host target and the workspace already has a
runner configured for that artifact; not every component supports host builds.
Detection uses the repository's own contents (e.g. an `@ohos/` `bundle.json`).

To pin the entry point explicitly in a script, lead the first message with
`/skill:pbt-workflow`:

```bash
pi-pbt -p "/skill:pbt-workflow 对当前仓库做性质测试(PBT),目标是找出 bug。产物写到 pbt-out/。"
```

### How deep it digs: effort tiers

A campaign runs at one of three tiers. The tier fixes the wall-clock budget, how
many properties are written, how many inputs each property is run against,
whether an all-passing first batch is allowed to end the campaign, and whether a
hard-to-build target may be swapped for an easier one.

| | `quick` | `standard` | `thorough` |
|---|---|---|---|
| Wall-clock | ≈10 min | ≈30 min | no cap |
| Properties per target | 3–5 | 5–8 | as many as the behavior warrants |
| Inputs per property | framework default (~100) | ≥1000 | ≥10000 |
| First batch all passes | may stop | strengthen and re-run ≥1 round | ≥2 rounds |
| Metamorphic / differential property | optional | ≥1 required | both required where applicable |
| Swap to an easier target when the build is hard | allowed | allowed, declared in the report | forbidden |
| Contract-surface sweep (coverage-gap driven) | none | 1 round | until covered / budget spent |

Defaults differ by entry point, because they answer different questions:
`hook-run` and `watch` fire on **every commit** and default to `quick`, so a
commit gate stays fast; everything else defaults to `standard`.

Set it per run with `--effort`, or for a whole environment with `PBT_EFFORT`:

```bash
pi-pbt hook-run <sha> --repo /path/to/repo --effort thorough
pi-pbt watch --repo /path/to/repo --effort standard
PBT_EFFORT=thorough pi-pbt -p "/skill:pbt-workflow ..."
```

Use `thorough` when finding the bug matters more than finishing quickly — a
release candidate, a security-relevant module, or a component whose build is the
expensive part. It has no time-box and no tool-call cap, and it forbids the
agent from quietly picking an easier target when the real one is hard to
compile.

This is separate from how strong a model you use. The tier controls how much
searching the agent does; the model controls how good its properties are. A weak
model at `thorough` still writes weak properties — see the model section above.

### How wide it runs: parallelism

pi-pbt sizes every test runner and build it drives to the machine's **currently
idle cores** — total cores minus the load already on the box. It sets each
framework's own standard variable before running a command, so this reaches
whatever the agent invokes, including through a Makefile or a wrapper script:

| Variable set for you | Effect |
|---|---|
| `CTEST_PARALLEL_LEVEL` | `ctest` runs that many tests at once (it is serial by default) |
| `CMAKE_BUILD_PARALLEL_LEVEL`, `MAKEFLAGS` | `cmake --build` / `make` job count |
| `RUST_TEST_THREADS` | `cargo test` thread count |
| `GOFLAGS` (`-p=N`) | `go test` package parallelism |
| `PYTEST_ADDOPTS` (`-n N`) | `pytest` workers — only when the `pytest-xdist` plugin is installed |
| `PBT_TEST_JOBS` | the resolved number, for runners with no variable of their own (`ninja -j"$PBT_TEST_JOBS"`, gradle, maven) |

This cuts both ways on purpose. `ctest` and `pytest` get *faster* (they default
to one test at a time); `cargo test`, `go test` and builds get *throttled* (they
default to every core and would otherwise fight whatever else the machine is
doing). Any value you set yourself is left alone.

Override with `PBT_TEST_JOBS`: a number pins it, `max` uses every core, `auto`
(the default) is the idle count. Inside a container the cgroup CPU quota is
respected, not the host's core count.

`PBT_TEST_JOBS=1` is also the triage tool: parallel runs can surface flakiness
that belongs to the *tests* (two cases claiming the same port, an xdist-unsafe
fixture). A campaign is required to re-run any failure serially before calling
it a bug, and to say in the report which run the verdict came from.

## Code coverage reports

A campaign also measures **which code actually executed**, using each language's
own coverage tooling, and leaves that toolchain's native HTML report under
`pbt-out/code-coverage/` with an `index.html` linking them together.

| Language | Tool used | Requires |
|---|---|---|
| C/C++ | `gcovr` (or `lcov` + `genhtml`) over a `--coverage` build | `gcovr` or `lcov` |
| Rust | `cargo llvm-cov` | `cargo-llvm-cov` |
| Python | `coverage.py` via `pytest-cov` | `pytest-cov` |
| Go | `go tool cover` | the Go toolchain |
| JavaScript/TypeScript | the report your runner already writes (c8, vitest, jest) | that runner's coverage provider |
| Java | JaCoCo's own report, adopted if the build produced one | JaCoCo configured in the build |

Instrumentation is switched on by adding each toolchain's flags to the
environment *before* the campaign builds anything (coverage is a build-time
decision — it cannot be added afterwards). Whatever you set yourself is left
alone. If a tool is missing, or a build is not instrumented, that row simply
says so in the report: **coverage never fails a campaign.**

When an OpenHarmony component has no `host_product` target and a configured
device-product runner is used instead, that cross-compiled run is *not*
instrumented — the profile data lands in the runner's target filesystem — and
the report states that rather than reporting zero coverage.

### What the report adds to `REPORT.md`

`COVERAGE.md` records which functions the campaign **claims** to have written
properties for. Coverage records what **ran**. The report crosses the two:

- *executed* — the claim is backed by execution
- *claimed but never executed* — the property does not exercise the function it
  is filed under; this is the finding worth acting on
- *no evidence* — the symbol was in no report at all, usually an uninstrumented
  target rather than an untrue claim

Turn the whole thing off with `PBT_CODE_COVERAGE=0`.

### Build first, then test: `build-run` (bring your own build command)

For a project whose build you already know how to drive — an OpenHarmony
component, a big monorepo, anything with an official build entry — hand that
command to pi-pbt and let it gate the campaign.

For OpenHarmony, prefer the native `host_product` path. The following ArkUI
`ace_engine` preflight has been run successfully against the real tree. It uses
an **existing** unittest group, so it proves the SUT can compile before the
campaign writes anything:

```bash
export PBT_OH_WORKSPACE=/path/to/openharmony
pi-pbt build-run \
  --workdir "$PBT_OH_WORKSPACE" \
  --repo "$PBT_OH_WORKSPACE/foundation/arkui/ace_engine" \
  --build-cmd "./build.sh --export-para PYCACHE_ENABLE:true --product-name host_product --build-target base_unittest --ccache --no-prebuilt-sdk" \
  --scope frameworks/base/geometry \
  --lang zh
```

`host_product` builds native Linux x86_64 tests under
`out/host/host_product` (not `out/host_product`), so no emulator or device is
needed for supported components. It does **not** cover every OpenHarmony
component. Only when the component has no host target should you use a device
product, and only after the workspace has a runner configured for that product's
artifact. Do not present a device build as a universal replacement for the host
path.

The initial `--build-cmd` is a preflight and must name a target that exists
**before** the campaign, such as `base_unittest`. Do not use `pbt`,
`<name>_pbt_test`, or another target that the campaign has not generated yet.
After the campaign creates and registers its tests in the component's own GN
test family, rebuilds may reuse the same `build.sh` command with the new `pbt`
target.

CMake remains fully supported by `build-run`, but it is a separate ordinary
project example, not an OpenHarmony-equivalent build path. For example, if that
project's existing `CMakeLists.txt` defines `calc_test`, gate on it:

```bash
pi-pbt build-run \
  --workdir /path/to/cmake-project \
  --repo /path/to/cmake-project \
  --build-cmd "cmake -S . -B build && cmake --build build --target calc_test -j$(nproc)" \
  --scope src/calc.cpp \
  --lang en
```

The build command is **your prepared input**, not something the agent figures
out. `build-run` executes it in `--workdir` first (full log in
`<out>/build.log`); if it fails, PBT never starts and the process exits `3` —
fix the tree or the command and re-run. If it succeeds, the campaign starts in
`--repo` under a build contract: rebuilds reuse exactly your command (only an
existing target may be replaced by the newly generated test target), and a
failing rebuild is a STOP condition recorded in `REPORT.md`. The agent will not
switch build systems or derive an alternative way to compile. `--scope` limits
the campaign to that path; omit it only for a full-repo run.

`--scope` and `--func` are prompt-only limits (not a sandbox):

```bash
pi-pbt build-run --build-cmd "…" \
  --scope path/to/file.cpp \
  --func Foo::Bar
```

`--scope <path>` is a file or directory (omit only for a full-repo run).
`--func <name>` (optional) names **one symbol** in that path. It **requires
`--scope`** and must appear **after** `--scope` on the command line. Without
`--func`, every PBT-worthy function in `--scope` is in play.

`--out`, `--lang`, `--effort` (default `standard`), `--provider`,
`--model`, and `--tui` work as on `hook-run`.

To watch the same flow live inside an interactive session, say:

```
/skill:pbt-build-run 构建命令: <your build command>
构建目录: <workdir>
```

The gate and the whole campaign then run right there in front of you, under
the same build contract; the subcommand remains the headless/CI form.

### CI / git-hook integration

To test **one specific commit's change set**, use this subcommand:

```bash
pi-pbt hook-run <sha> --repo /path/to/repo --lang zh
```

It reports its verdict as an exit code, so it drops straight into CI as a check:

| | Meaning |
|---|---|
| `0` | Clean: a report that satisfies the schema, and no bugs. |
| `1` | Bugs found (`bug_reports/` non-empty, or `totals.bugs > 0`). |
| `2` | No `REPORT.md`, no `report.json`, or a `report.json` that fails the schema — it died, timed out, or attested to nothing. |
| `3` | The recorded build failed: nothing was tested. |

**The sha does not check anything out.** It selects the change set
(`git diff-tree <sha>`) and goes into the prompt; the code that gets compiled
and tested is whatever is in `--repo`'s working tree. Putting the tree on that
commit is the caller's job. Running against a different revision neither errors
nor warns: it tests the code that is there against the diff of a commit that is
not.

Two prerequisites follow from that:

- **From a `post-commit` hook, nothing extra is needed** — the commit you just
  made is already the working tree. Do NOT add a checkout there; it would leave
  the developer on a detached HEAD after every commit.
- **In CI, check the commit out into the disposable workspace, and make sure
  its PARENT is present.** The default shallow clone (`fetch-depth: 1`) has no
  parent, so `git diff-tree <sha>` returns an empty change set and `git show
  <sha>` treats the shallow boundary as a root commit — the campaign then sees
  either nothing or the whole repository as new:

  ```yaml
  - uses: actions/checkout@v4
    with:
      fetch-depth: 2          # the commit AND its parent; 0 for full history
  ```

  ```bash
  git -C /path/to/worktree checkout --detach <sha>   # only in a disposable checkout
  pi-pbt hook-run <sha> --repo /path/to/worktree
  ```

### Watching a repo: test every new commit

Two ways, depending on whether you want to *watch* it work.

**Interactive — in front of you (recommended whenever you have pi-pbt open).**
In an interactive `pi-pbt` session, say:

```
/skill:pbt-watch
```

It watches the current repo (or one you name) in the background and, the moment
a new commit lands, runs the whole thing **right there in front of you** —
finding targets, planning, writing and running tests, reviewing — then keeps
watching for the next commit. Watching does not tie up the session: the command
returns immediately and you can keep asking for other things meanwhile; after
each round it carries on by itself, with no command to type to resume. Nothing
is pushed off somewhere you cannot see it, and you can just tell it to stop. For
an OpenHarmony module, a pre-built source environment is detected and reused
automatically. (This also shows up live in the dashboard, below.)

**Unattended — on a machine nobody is watching.** On a CI mirror, a server with
no pi-pbt open, or where a git hook cannot be installed, run it resident in the
background instead. It checks for new commits on a timer and tests each one it
finds, with the results going to its log:

```bash
# watch local commits on the current repo, every 30s (put it in tmux/systemd)
pi-pbt watch --repo /path/to/repo --lang zh --provider xai-oauth --model grok-4.5

# watch pushes to the remote tracking branch instead
pi-pbt watch --repo /path/to/repo --fetch --branch master --interval 60 --lang zh --provider xai-oauth --model grok-4.5
```

The starting point is the latest commit at startup (commits that already existed
are not tested retroactively); every new commit is then tested, with the verdict
(the exit codes above) recorded in the log. All `hook-run` flags (`--out`,
`--workdir`, `--spec`, `--lang`, `--scan-root`, `--effort`, `--provider`,
`--model`, `--tui`) pass through. `--tui` (or `PBT_HOOK_TUI=1`) runs it in the full
interactive interface in the same terminal (good for demos, not for CI: it stops
and waits for `/quit` after each round).

### Delegation from another coding agent (MCP)

If your day-to-day agent is Claude Code or Codex, it can delegate PBT to pi-pbt
over the [Model Context Protocol](https://modelcontextprotocol.io) and **keep
developing while the campaign runs**. `pi-pbt mcp` serves MCP over stdio from
the same single binary:

```bash
# Claude Code
claude mcp add --transport stdio pi-pbt -- pi-pbt mcp

# Codex
codex mcp add pi-pbt -- pi-pbt mcp
```

The server exposes six tools:

| Tool | What it does |
|---|---|
| `pbt_start` | start a campaign on **one immutable commit** (default: current `HEAD`, resolved to a full sha); returns a `run_id` immediately. `build_cmd` / `build_workdir` supply a prepared build command and run the campaign in place (see below) |
| `pbt_status` | state / phase / queue position / artifact URIs for a run or watch |
| `pbt_report` | verdict (`passed` / `bugs_found` / `failed`), `REPORT.md` summary, bug-report index, patch presence |
| `pbt_cancel` | cancel a queued or running campaign (idempotent) |
| `pbt_watch_start` | start one campaign per new commit on a branch |
| `pbt_watch_stop` | stop a watch (by default cancelling its in-flight run) |

The isolation contract is the point: **pi-pbt never tests your changing working
tree.** Every run snapshots the requested commit into a detached `git worktree`
and campaigns there, so you can keep editing, committing, and even rewriting the
same files while it runs — commit a checkpoint, call `pbt_start`, continue
coding.

Nothing committed yet? Pass `include_uncommitted: true` to `pbt_start` and the
current uncommitted state (tracked modifications plus untracked non-ignored
files) is frozen into an immutable snapshot commit first, then tested exactly
like any other revision. The snapshot is minted with a throwaway git index —
your index, `HEAD`, refs, stash, and files are untouched, and you can keep
typing while it runs. A clean tree simply tests `HEAD`. It is mutually
exclusive with `revision`, and the run reports `snapshot: true` plus the
`base_revision` it was taken on.

**One exception, and it is deliberate: `build_cmd`.** Large components
(OpenHarmony, AOSP, a monorepo subtree) cannot be built from a detached
worktree — their build systems resolve paths, generated headers, and ccache
state against the real checkout. Passing `build_cmd` to `pbt_start` therefore
runs the campaign **in place, in the repository itself**, with no worktree
created. The command runs first and a non-zero exit stops the campaign before
any agent work, so a broken build costs you seconds rather than an exploratory
build hunt. Set `build_workdir` when the build must run from a directory above
the module under test, e.g. the OpenHarmony source root while the scope is one
component. Use it when you know your build command; leave it off to keep the
worktree isolation. When the run finishes, any worktree is removed; what survives lives
under `~/.pi-pbt/runs/<run-id>/` (override with `PI_PBT_RUNS_DIR`): `run.json`,
`events.jsonl`, both logs, the campaign artifacts, and a `changes.patch` holding
everything the campaign wrote in its worktree. All of it is also readable
through MCP resources at `pbt://runs/<run-id>/...` (`REPORT.md`,
`PROPERTIES.md`, `bug_reports/<slug>.md`, `changes.patch`, …).

Heavy campaigns are serialized: one child at a time by default
(`PBT_MCP_MAX_CONCURRENT` raises it); further `pbt_start` calls queue. A watch
defaults to `supersede: true` — a newer commit cancels an in-flight run that has
not reached Review, while a run already in Review finishes first and
intermediate commits coalesce to the newest one, so the watch always converges
on the latest code without stacking campaigns.

Progress is pushed two ways. Standard MCP logging notifications work in every
client, and polling `pbt_status` is always reliable. Claude Code sessions can
additionally receive live channel events (campaign phase changes and the final
verdict, carrying only run IDs and status — never repository content). Channels
are a Claude Code research preview, so this part needs an explicit opt-in at
launch:

```bash
claude --dangerously-load-development-channels server:pi-pbt
```

Without that flag everything except the push still works.

### A HarmonyOS app on a phone (`pi-pbt kea`)

Everything above tests **source code**. `pi-pbt kea` tests something different:
an **already installed HarmonyOS app on a USB-connected phone**, as a black box
through its GUI. It drives the Kea2 engine over `hdc` — Kea2 explores the app
while checking properties that must hold whatever the user does — and reports
crashes, ANRs and property violations.

Three things must be in place before it runs:

| | |
|---|---|
| Phone | connected over USB and unlockable without a PIN; `hdc` on `PATH` (`hdc list targets` shows the serial) |
| Kea2 | a Kea2 checkout with its virtualenv, i.e. `<kea_home>/.venv/bin/kea2` (or `venv/bin/kea2`) exists |
| Decompile signals | `<decompile_home>/mined_all/<package>/signals.json`, mined from the app beforehand. Absent → it aborts rather than explore blind |

Configuration is a `kea.config.yml` in the folder you run it from (the "SUT
folder"). Minimal:

```yaml
package: com.example.app
kea_home: /path/to/Kea2
decompile_home: /path/to/harmony-decompile
```

Then:

```bash
cd /path/to/sut-folder     # the folder holding kea.config.yml
pi-pbt kea                 # interactive
pi-pbt kea -p              # headless (CI, nohup)
pi-pbt kea -c              # continue: reuse the previous run's directory
pi-pbt kea --config other.yml --lang zh
```

Those four flags are all it takes; everything else is configured in the file:

| Key | Default | Meaning |
|---|---|---|
| `package` | **required** | bundle name of the app under test |
| `kea_home` | `~/github/Kea2` | Kea2 checkout (its virtualenv provides the `kea2` binary) |
| `decompile_home` | **required** | directory holding `mined_all/<package>/signals.json` |
| `device` | the connected one | device serial, when more than one phone is attached |
| `out` | `pbt-out` | artifact directory, relative to the SUT folder |
| `depth` | from the effort tier | `fast` (short pass) or `deep` (the full property packs) |
| `events`, `running_minutes`, `throttle` | 15 / 6 / 500 (fast), 70 / 12 / 200 (deep) | exploration budget: max steps, wall-clock minutes, ms between events |
| `mode_a_packs` | the built-in packs | `deep` only: Python modules holding the property packs to run |
| `stamp_runs` | `true` | give each run its own timestamped directory; `false` writes flat into `out` |
| `provider`, `model`, `lang` | — | same meaning as the global options |

Depth is the same "how deep do we dig" knob as the [effort tiers](#how-deep-it-digs-effort-tiers)
above, so it is not set twice: `depth:` in the config wins, then `PBT_KEA_DEPTH`,
then the tier (`quick` → `fast`, `standard`/`thorough` → `deep`).

What a run leaves behind:

```text
pbt-out/
  LATEST                                    # points at the newest run directory
  runs/<package>_modeB_<depth>_<stamp>/
    layout.json      one UI dump of the app
    kea-run/         Kea2's own output (res_*/result_*.json)
    LAST_RUN.json    parsed counters: executions, failures, per property
    REPORT.md        the verdict
```

A missing config file, a config without `package:`, or missing decompile
signals stop it with an explanatory message and exit code `1`, before the phone
is touched.

### Environment variables

| Variable | Effect |
|---|---|
| `PBT_LANG=zh` / `en` | Chinese or English narration and artifacts; also `--lang` on subcommands. `en` still injects an English directive when the campaign prompt is Chinese |
| `PBT_SCAN_ROOT=/path` | the repo to scan, for setups where the working directory is a clean copy (git hooks, CI) |
| `PBT_OH_WORKSPACE=/path` | a pre-built full OpenHarmony source environment (source tree + toolchain + built dependencies) to reuse, instead of working out how to build the component standalone. **Detected automatically** when the repo sits inside such an environment (a parent directory with both `.repo/` and `out/`) — set it explicitly only to override; the startup log prints which one is in use |
| `PBT_HOOK_TUI=1` | same as `hook-run --tui` / `watch --tui`: run in the interactive interface (needs a terminal; waits for `/quit`) |
| `PBT_EFFORT=quick\|standard\|thorough` | how deep a campaign digs (see [effort tiers](#how-deep-it-digs-effort-tiers)); also `--effort` on `hook-run` / `watch`, which takes priority. Default: `quick` for `hook-run`/`watch`, `standard` elsewhere. Also sets the `kea` GUI exploration depth when `kea.config.yml` does not pin `depth:` (`quick` → short run, otherwise the long one) |
| `PBT_KEA_DEPTH=fast\|deep` | [`pi-pbt kea`](#a-harmonyos-app-on-a-phone-pi-pbt-kea) exploration depth, overriding the effort tier; the config's `depth:` still wins |
| `PBT_KEA_MODE_A_PACKS="pkg.a pkg.b"` | the property packs `pi-pbt kea` runs at `deep`, when the config sets no `mode_a_packs` |
| `PBT_TEST_JOBS=8` | how wide tests and builds run (see [parallelism](#how-wide-it-runs-parallelism)); a number, `max`, or `auto` (default: the machine's idle cores). `PBT_TEST_JOBS=1` forces serial — required when reconfirming a failure before filing it as a bug |
| `PBT_CODE_COVERAGE=0` | turn off native code-coverage instrumentation and reporting (see [code coverage](#code-coverage-reports)); on by default, and it degrades silently when a toolchain is missing |
| `PBT_BARE=1` | run as plain pi: no orchestration extension, no bundled skills, no campaign guards. Exists for A/B-measuring the harness's own contribution (benchmarking); not for normal use |
| `PI_PBT_RUNS_DIR=/path` | where [`pi-pbt mcp`](#delegation-from-another-coding-agent-mcp) persists run state and artifacts (default `~/.pi-pbt/runs`; must be outside the repo under test) |
| `PBT_OUT_DIR=/path` | where the campaign's artifacts (`PLAN.md`, `PROPERTIES.md`, `COVERAGE.md`, `REPORT.md`, `bug_reports/`) live. **Set automatically** by `build-run` and `hook-run` from their `--out`, so you normally never set it yourself; it exists so the artifact-driven checks read the same directory the campaign writes. Leave it unset for a plain `pi-pbt -p` campaign, which uses `<cwd>/pbt-out`. Do NOT export it in a shell profile: a stale value follows every later campaign, and an inherited one that points somewhere else is exactly the split-brain it was added to prevent |
| `PBT_MCP_MAX_CONCURRENT=2` | how many MCP-delegated campaigns may run at once (default 1; extra runs queue) |
| `PBT_PHASE_MODELS='{"plan":"anthropic/claude-opus-5"}'` | route a different model per campaign phase (`scan`, `plan`, `test`, `review`), as `<provider>/<modelId>`. Unset means one model throughout, which is the default. The phase is read off the artifacts under `pbt-out/`, never self-reported by the agent; an unknown model or unconfigured provider silently leaves the campaign on its current model. There is no built-in routing table on purpose: `scan` and `review` are where the campaign decides what the contract is and whether a failure is real, so downgrading them buys tokens at the price of a false pass |

## 5. Dashboard: watch it work, live

pi-pbt can serve a web dashboard showing every run on the machine as it happens
— whether you started it by hand, a git hook did, or a repo watch did — with the
conversation, token usage and produced files, plus the history of past runs. No
extra npm install is required; pi-pbt installs and runs it itself:

```bash
pi-pbt dashboard install       # one-time: pins the dashboard package under
                               # ~/.pi-pbt/dashboard and wires the bridge
                               # extension into ~/.pi-pbt/agent/extensions/
pi-pbt dashboard --port 8000   # run resident (put it in tmux/systemd)
```

For example:

```bash
tmux new-session -d -s dash 'pi-pbt dashboard --port 8000 >> ~/pbt-dash.log 2>&1'
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8000/   # expect 200 (takes ~20s)
```

Then open **http://localhost:8000** in a browser. On a remote host, tunnel the
port first: `ssh -N -f -L 8000:localhost:8000 <host>`.

How it works: the install step wires things up once, so **every pi-pbt run on
the machine shows up automatically** — one launched by a git hook, by CI, or by
a repo watch appears the moment it starts (named `pbt_hook_<sha>`), with no
per-run configuration. History is read from `~/.pi-pbt/agent/sessions/` (not
`~/.pi/agent/sessions/`) — pi-pbt keeps its own config and history separate from
a plain `pi` install.

Notes:
- The sidebar groups runs by **pinned folders** — pin the repo directories you
  care about (folder menu in the UI) or their runs are only reachable through
  the full list.
- Verify it is alive with `curl localhost:8000` (some hosts don't show the
  ports in `ss`/`netstat`).
- After updating pi-pbt, run `pi-pbt dashboard install` again to refresh the
  separately installed, exact-pinned dashboard package.
- Only one dashboard can run per user: it occupies the chosen HTTP port and
  the bridge gateway port **9999**. `--port` changes only the HTTP port. Stop
  the existing instance before restarting; `pi-pbt dashboard` reports either
  occupied port before invoking the upstream server.

## 6. Troubleshooting

- **It improvises instead of following its built-in workflow** — the built-in
  workflow files did not load (the startup log shows `skills=0`). Delete the
  cache directory `~/.pi-pbt/cache/` and run again to let it rebuild; if it is
  still 0, the home directory is most likely not writable or the disk is full.
- **`command not found: cmake` partway through** — install the toolchain for
  your target language (§1) and re-run. It will not skip the build and pretend
  the code was tested.
- **It hangs in CI** — it should not: it never waits for input and force-exits
  when done. If you see a hang, file an issue with the log.
- **macOS blocks the binary** — run the `xattr -d com.apple.quarantine` step
  from §2.

### Empty responses and campaign recovery

If a PBT campaign receives an empty/thinking-only response with `stop` or an output-budget `length` stop,
pi-pbt tries **one** continuation. It shortens older tool-result text only for
subsequent model requests, retaining user instructions, tool-call/result IDs
and recent messages. Individual recent successful outputs over 16,000 characters
are also shortened, retaining 4,000 characters at each end; the saved session remains unchanged. This is
**not** pi's `/compact` and does not summarize or reconstruct missing facts.
The retry reads existing artifacts and prioritizes report completion. A
context estimate at 80% of the configured model window also selects this
close-out path rather than another coverage/deepening sweep. No model-specific
token limit is assumed.

`pbt-out/recovery.json` records the attempt, removed character count and outcome.
`response-resumed` means only that the model responded again, **not** that the
campaign passed; the normal artifact/schema gates still decide that. A second
empty response stops recovery (`empty-after-retry`). Cancellation and provider
errors are not retried by this mechanism. Unperformed sweeps must be recorded
as incomplete, never counted as executed. If recovery fails, inspect the session
and model/provider configuration; repeated “continue” prompts do not guarantee
recovery.

### Local PBT dependencies and enterprise networks

C/C++ campaigns (native source/build markers or an OpenHarmony component) inventory
known project locations first, then the installed OS package database (dpkg, pacman, RPM, or Homebrew). It writes
`pbt-out/dependencies.json` and passes the findings to the agent before testing.
For C++ this currently covers RapidCheck and GoogleTest: installed package
names, versions, reported file paths and architecture, **not** a guarantee of
successful linking. Detection is bounded, read-only and makes no network calls.

The acquisition order is: **project dependency → installed system package →
configured system repositories/internal mirrors → public downloads with
permission**. For example, an installed `librapidcheck-dev` is evidence to
inspect and use, not a reason to clone RapidCheck again. The CLI itself does
not run apt/pacman/dnf/brew installation commands or sudo during inventory.
Installation remains an explicitly permitted agent/operator action.

OpenHarmony GN builds must still use their project-compatible source/target,
e.g. `third_party/rapidcheck`. An amd64 system library must not be linked into
an ARM target, and even host_product may use a different ABI/toolchain from the
OS package. The inventory marks this distinction instead of claiming “installed”
means “usable by this target.” Missing inventory or a query failure does not
prove that the dependency is absent.
