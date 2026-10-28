# Installing PBT Agent

[中文版 / Chinese version](installation.zh.md)

pi-pbt is an AI agent: point it at a repository and it reads the code, works out
**what should always be true of it**, writes tests that check those rules against
large numbers of generated inputs, and reports the bugs it finds with a minimal
reproducer for each. That style of testing is called **property-based testing**
(PBT).

Install the archive for your platform. The main executable is self-contained
(no Node.js or Bun runtime needed); the archive also includes the installer and
search tools. The project under test still needs its own toolchain (§1).

## 1. System requirements

pi-pbt has no dependencies of its own. But it **really compiles and runs the
test code it writes**, so the toolchain for the language under test must be
present:

| You test | Install first |
|---|---|
| Any repo | `git` (it reads the commit history and diffs) |
| C / C++ | `cmake` ≥ 3.16 for CMake projects, a C++17 compiler (`clang`/`g++`), and the project's own build tool (`make`, `ninja`, GN, etc.). See dependency preparation below for GoogleTest/RapidCheck. |
| Python | `python3` ≥ 3.9 with `pip` (`hypothesis` is installed into the environment it finds) |
| Rust | `cargo` (`proptest` is added as a dev-dependency) |
| Go | `go` toolchain |
| Java | JDK + Maven/Gradle (jqwik) |
| Kotlin, incl. Multiplatform | JDK + the project's Gradle wrapper (kotest-property in the existing test source set, `commonTest` for KMP, resolved from Maven Central); run only targets whose toolchains are available and record each tested task |
| HarmonyOS HAP | DevEco CLI (`hvigorw`, `ohpm`) and the app SDK — installed for you by the `arkts-build-run` install card when missing (distro package with `sudo`, tarball otherwise). A HAP root is a directory with `oh-package.json5` + `build-profile.json5`; `hvigorw` may be the project wrapper, on `PATH`, or under the DevEco install. |
| OpenHarmony components | A provisioned OpenHarmony source tree and its official build prerequisites. Prefer native `host_product` tests; a device-product runner is needed only for components with no host target. |
| A HarmonyOS app on a real phone | `hdc` (HarmonyOS device connector) and a Kea2 checkout with its virtualenv — see the experimental [`pi-pbt kea` guide](kea.md) |

Debian/Ubuntu example for C++ targets:

```bash
sudo apt-get install -y git cmake ninja-build clang
```

Arch Linux:

```bash
sudo pacman -S --needed git cmake ninja clang
```

### Prepare PBT dependencies

For C++ campaigns, pi-pbt records locally available RapidCheck and GoogleTest
information in `pbt-out/dependencies.json`. The campaign works down this order
and stops at the first rung that works: **the project's own dependency, an
already installed system package, a configured system or enterprise-internal
repository, then vendoring the framework's source into the harness**. The order
avoids unnecessary network access and duplicate copies.

A missing dependency does not stop a campaign. The agent is expected to acquire
one without asking: system package managers run directly when the campaign is
already root and under `sudo -n` when elevation is needed, never interactively,
so a password prompt can never hang a headless run; language package managers
run in user scope; and a source vendor needs no root at all — usually the rung
that works on a locked-down box. A vendored framework is pinned to a recorded
revision and verified before it is compiled, so a campaign never builds whatever
an upstream default branch happens to hold. It will not add a new package
repository to your system. If every rung genuinely fails
(air-gapped, no sudo, no source), the campaign still runs to completion with a
self-written driver. The workflow requires that driver to carry
`PBT-FALLBACK: <why>` at the top of the test file and the same reason in
`REPORT.md`; the runtime enforces it wherever it can recognise hand-rolled
randomness (`std::mt19937`, `random.randint`, `Math.random()` and the
equivalents in each supported language), which is the shape a substituted
generator actually takes.

The inventory distinguishes package paths and architecture from actual target
compatibility. A library installed for the host ABI cannot be linked into an ARM
target, and an OpenHarmony `host_product` build may still require its own
toolchain-compatible source, such as `third_party/rapidcheck`. Treat
`dependencies.json` as preparation evidence, not proof that a dependency will
link successfully.

## 2. Download and install

Obtain the archive for your platform from wherever you received pi-pbt (a
Releases page, an internal mirror, or a direct handoff). Each ships with a
`.sha256` checksum alongside it. The published files are:

| Platform | File | Download |
|---|---|---|
| Linux x64 | `pi-pbt-linux-x64.tar.xz` | ~46 MiB |
| Linux arm64 (aarch64) | `pi-pbt-linux-arm64.tar.xz` | ~42 MiB |
| macOS Apple Silicon | `pi-pbt-macos-arm64.tar.xz` | ~33 MiB |
| Windows x64 (**experimental**) | `pi-pbt-windows-x64.tar.xz` | ~47 MiB |

Every archive is a `.tar.xz`. On Linux and macOS extract it with `tar -xJf`
(`xz` ships with every mainstream Linux distribution and macOS); on Windows see
below.

Not sure which Linux one you need? `uname -m` — `x86_64` takes the x64 file,
`aarch64` the arm64 one.

### Windows (experimental)

`pi-pbt-windows-x64.tar.xz` extracts to `pi-pbt-windows-x64\` holding
`pi-pbt.exe` and `tools\fd.exe` / `tools\rg.exe`; there is no installer.
Extract it with 7-Zip, WinRAR or Windows 11's File Explorer, or with `tar -xf`
where that `tar` reads xz. The `tar.exe` built into Windows 10 and Windows
Server 2022 does not: it stops with an unsupported-format error. Put
that directory on `PATH` (or call `pi-pbt.exe` by path) and run it from
PowerShell or cmd. Configuration lives in `%USERPROFILE%\.pi-pbt\agent\`
(same files as on Linux). What works and what does not:

| | Windows |
|---|---|
| Python, Java, Kotlin, JS/TS campaigns (Hypothesis, jqwik, kotest-property, fast-check) | yes — the project's own test runner is invoked as it would be from a terminal |
| ArkTS / HAP (hvigor + ohpm) | yes, with Huawei's official DevEco Studio installed (pi-pbt detects it under `Program Files\Huawei\DevEco Studio` or `DEVECO_HOME`; it never installs it) and Git Bash for the `arkts-build-run` helper scripts |
| C / C++ (RapidCheck, CMake, GN) and OpenHarmony components | no — no MSVC or OpenHarmony build cards |
| `build-run` gate | yes — the command runs through the platform shell and its output is captured to `build.log` |
| Dashboard, Kea, native coverage (`llvm-cov`/`gcov`) | no |

The SOP's shell snippets are written for a POSIX shell; on Windows the agent
uses PowerShell/cmd equivalents, or Git Bash when present. Report Windows
findings as issues — the platform is experimental.

Every archive extracts to a same-named directory containing `pi-pbt`, an
`install.sh` installer, and adjacent `tools/fd` and `tools/rg`. Run the bundled
installer to install the main executable and place its tools where the embedded
pi-pbt can discover them: beside the installed executable in `/usr/local/bin/tools`
by default, or `$PI_CODING_AGENT_DIR/bin` when that override is explicitly set.

```bash
PLATFORM=linux-x64   # or linux-arm64 / macos-arm64 (Windows: see above, no install.sh)
sha256sum -c "pi-pbt-${PLATFORM}.tar.xz.sha256"   # optional integrity check
tar -xJf "pi-pbt-${PLATFORM}.tar.xz"            # needs xz; install it if tar reports it missing
cd "pi-pbt-${PLATFORM}"
./install.sh
```

**v0.2.0 replacement build (2026-10-28):** v0.2.0 was rebuilt in place with
smaller archives: every platform now ships `.tar.xz` instead of `.zip`, and the
binary embeds only bundled, minified code. If you downloaded a v0.2.0 `.zip`
earlier, it still works; the current files are the `.tar.xz` ones above,
checked with their own `.sha256`.

**v0.1.18 replacement build (2026-09-16):** the current archives include the
installer correction above. If you downloaded v0.1.18 earlier, download the archive
and its `.sha256` again and rerun `./install.sh`. The version number is unchanged;
use the current archive checksum to distinguish the replacement. Refresh any
installed host-agent skill with `pi-pbt skill-install --force`.

Running `fd` or `rg` directly from a regular shell may still report command not
found, which is expected; no additional `PATH` change is required. Extracting
on Windows drops Unix permissions, so if the files travelled through a Windows
machine, run `chmod +x` on `pi-pbt`, `install.sh`, `tools/fd`, and `tools/rg`
first.

The arm64 and macOS builds are cross-compiled on an x64 Linux machine; they are
validated by format (the release job asserts each is really an aarch64 ELF /
Mach-O arm64), not executed on the target. This does not prove runtime behavior on those platforms. If a build fails on
your target, report the OS/architecture, `pi-pbt --version`, and startup log to
your distribution maintainer.

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
updating pi-pbt to refresh it; `skill-uninstall` targets detected user-level hosts; repeat your original
`--host` or `--dir` selection for project/custom installations. Then just tell your agent "test the changes I just made" —
see the [host-agent skill guide](skill-pi-pbt-dev.md).

### Upgrade an existing installation

1. Stop active campaigns. Download the new archive **and its matching checksum**,
   extract into a fresh directory, verify it, then run that archive's `./install.sh`.
2. If you installed `pi-pbt-dev` into a host agent, **refresh that copied skill**
   after replacing the binary. Bundled campaign skills update with the binary;
   the copy installed into another agent does not.

   ```bash
   pi-pbt skill-install --force                       # detected user-level hosts
   pi-pbt skill-install --host claude --force         # one previously selected host
   pi-pbt skill-install --host project --force        # run in each relevant project
   pi-pbt skill-install --dir /custom/skills --force   # reuse the original custom path
   ```

   Choose the command matching your original installation; do not run every
   example. `--force` replaces the installed skill files, so back up any local
   edits first. Reload/restart the host agent so it reads the refreshed skill.
3. Verify the executable on PATH and model configuration:

   ```bash
   command -v pi-pbt
   pi-pbt --version
   pi-pbt --list-models
   ```

Keep `~/.pi-pbt/agent/` (credentials, models and history); upgrading does not
require deleting it. Same-version replacement archives such as the reissued
v0.1.18 must be distinguished by the current download checksum, not the version
string alone.

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

For GLM or DeepSeek through CLIProxyAPI, use the
[verified channel-specific models.json examples](models-glm-deepseek.md).
They list the source commit, context windows, output caps and a practical
request budget. Do not copy a generic 1M context value onto every model or proxy.

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

It works through four steps — scan, plan, test, review. One such run is called
a **campaign**.

While a campaign runs you can ask a side question without interrupting it or
polluting its context: `/btw <question>` (alias `/side`) opens a parallel
sub-session with read/bash/edit tool access in its own overlay, `/btw:ask` a
read-only one, `/btw:tangent` one that does not inherit the conversation. The
bundled [pi-btw](https://github.com/dbachelder/pi-btw) package provides it;
it loads in interactive sessions only, never in `-p` runs.

pi-pbt also bundles [pi-subagents](https://github.com/nicobailon/pi-subagents),
which provides the `subagent` tool (fresh-context child sessions, parallel
fan-out, background runs; `/subagents-doctor` checks its setup). It loads in
interactive sessions with its skills and prompts, and in `-p` runs as the
extension only. Campaigns use it for two things on their own. **Scan workers**:
the `scan_modules` tool gives the Scan's reading to read-only children in
parallel (fresh context; `read`, `grep`, `find`, `ls` only) — in a change
campaign every changed function, its callers and callees and the related
functions the workers report, each of which must be read or excused before
properties are written; in a scope campaign the scope's source files — and
records what each read in `pbt-out/SCAN_COVERAGE.md`. And the **independent
close-out review**. Before a campaign may file its result, it runs one
read-only reviewer child (fresh context; `read`, `grep`, `find`, `ls` only) over
`PROPERTIES.md`, every `bug_reports/*.md`, REPORT.md's Design Caveats and the
source lines they cite, asking whether each filed bug's contract evidence
supports it, whether any passing property's oracle is weakened, and whether each
caveat (and each property retired on a citation) rests on a line that, quoted
verbatim, really excludes the behavior. pi-pbt writes the verdict to `pbt-out/REVIEW.md`,
and the result is refused until every finding in it has been answered. The
reviewer uses the same model as the campaign unless you route it elsewhere with
pi-subagents' `agentOverrides.pbt-closeout-reviewer` setting (see its
configuration docs). Set `PBT_CLOSEOUT_REVIEW=off` to skip the review, at the
cost of that independent check. A result that was not independently reviewed
(turned off, reviewer unavailable, or the reviewer failed twice) says so: the
machine-readable result carries `"review": {"status": "not-reviewed", "reason": ...}`
and REPORT.md an `**Independent review:** not reviewed (...)` line.

### Read the results

A completed **source-code PBT campaign** produces the following artifacts in
`pbt-out/` by default. With `--out`, use the directory you selected; MCP-managed
runs expose their artifact locations through `pbt_status` and `pbt_report`.

| Artifact | Who uses it and why |
|---|---|
| `REPORT.html` | Customer-facing overview: result summary and links to every confirmed bug. Start here. |
| `bug_reports/*.html` | One customer-facing page per confirmed bug: overview, discovery, reproduction and recommended fix. |
| `REPORT.md` | Technical Markdown campaign narrative: scope, results, limitations and reproduction commands. |
| `bug_reports/*.md` | Markdown counterpart of each individual bug report. |
| `report.json` | The machine-readable campaign result for CI, MCP and other integrations. It contains `schemaVersion`, tested revision, build result, properties, bugs and totals. Each bug links to the property that found it and its reproduction commands. |
| `PROPERTIES.md` / `PLAN.md` | The property ledger and campaign progress; useful for reviewing what was attempted. |
| `REVIEW.md` | The independent close-out review: each finding against a filed bug or a passing oracle, and the campaign's response to it. Whether the review happened is also recorded in `report.json` (`review`). |
| `COVERAGE.md` / `COVERAGE_STATUS.md` | Function/testing progress records, not native line-coverage percentages. See [coverage tracking](coverage-tracking.md). |

`report.json` is an output, **not a configuration file to create before running**.
The agent writes the technical `REPORT.md`, the `bug_reports/*.md` pages and
`report.json`; once the JSON contract validates, pi-pbt automatically generates
`REPORT.html` and the individual HTML bug pages from it (a Markdown overview or
bug page is rendered only where the agent left none — its own Markdown is never
overwritten). A truncated or
missing report is an incomplete run, not “zero bugs”. See the
[report schema](report-schema.md) for fields and the [reproduction guide](reproducing.md)
for replaying failures and committing generated tests. This report contract also applies to
v0.1.18 replacement archives: refresh the archive when you need the corrected installer,
then rely on the generated report rather than inferring a clean result from process output.

Generated tests live in the project's test tree, not in `pbt-out/`. With the
MCP worktree mode, retrieve `changes.patch` before applying/reviewing the tests;
in-place runs already leave them in your checkout. Kea GUI runs have a separate
[artifact format](kea.md#artifacts-and-failures).

Diagnostic files are conditional: `dependencies.json` describes discovered C++
dependencies, `guards.jsonl` records blocked calls, `recovery.json` records
an attempted recovery, and `compaction.jsonl` records each context compaction
during the campaign (one line per outcome: reason, tokens before, whether the
campaign-state block was attached, or the error). Their absence does not by
itself make a run incomplete.

All subcommands (`build-run`, `hook-run`, `watch`, `replay`, `scan`, …) are listed
in [subcommands.md](subcommands.md) ([中文](subcommands.zh.md)).

### From the command line (unattended scripts)

```bash
pi-pbt -p "Run property-based testing on this repository; write results to pbt-out/."
```

`-p` prints the run and exits without waiting for terminal input. It is suitable
for unattended generation, but its exit code is **not a campaign verdict**. For
a CI check that fails on bugs or incomplete reports, use [hook-run](#ci--git-hook-integration).

You do not pick a mode per project type: there is one entry point, and it works
the rest out itself. When the target does not build with a plain
`cmake`/`cargo`/`pytest` — a component of a large OS or of a monorepo — it stands
that component up first. OpenHarmony C++ components follow their official build
path, checking native `host_product` tests first. A device product is only a
fallback when the component has no host target and the workspace already has a
runner configured for that artifact; not every component supports host builds.
Detection uses the repository's own contents (e.g. an `@ohos/` `bundle.json`).
HarmonyOS ArkTS apps are detected separately — see [HAP / ArkTS](#harmonyos-hap--arkts-apps).

### HarmonyOS HAP / ArkTS apps

Dedicated page: [ArkTS PBT (HarmonyOS HAP)](pbt/arkts.md)
([中文](pbt/arkts.zh.md)).

pi-pbt treats a directory as a **HAP root** when it contains `oh-package.json5`
and `build-profile.json5` (in the project or a parent directory). The `hvigorw`
wrapper is optional: the toolchain is whichever `hvigorw` resolves first — the
project wrapper, `hvigorw` on `PATH` (the DevEco package symlinks it into
`/usr/bin`), or a DevEco install at `$DEVECO_HOME`, `/opt/devecostudio` or
`~/devecostudio`. When none exists, the campaign is told so and the bundled
`arkts-build-run` skill's install card installs DevEco from the
[devecostudio-linux](https://github.com/alex3236/devecostudio-linux) packages:
with `sudo` (or as root) the distro package, otherwise the tarball into the home
directory. The project's `compileSdkVersion` / `runtimeOS` are compared with the
installed SDK and the campaign is told what to align (an extra SDK, or raising
the project's SDK numbers with a targeted edit).

On a HAP root, free-form `pi-pbt` / `-p` and `build-run` use Hypium +
PandaCheck in `ohosTest`. The verdict is `hdc shell aa test`
(`OHOS_REPORT_RESULT`). Host `es2panda`/`ark` (`PBT_PANDA=1`,
`PANDACHECK_RESULT`) is a separate runner, and only when the tree is not a
HAP root. Prefer pinning the build. `hvigorw` here is whichever resolves —
the DevEco package puts it on `PATH`; use `./hvigorw` only when the project
ships the wrapper:

```bash
pi-pbt build-run --build-cmd "hvigorw assembleHap --mode module -p module=entry@ohosTest -p product=default --no-daemon"
```

When `hvigorw` or the SDK is missing, `build-run` does not run `--build-cmd`:
`build.log` records why, and the campaign is told to install first. An SDK
align note (`runtimeOS`, API flavour) is a warning. It does not skip the
operator command.

`build-run` then sets `PBT_HAP=1` for that process.

OpenHarmony application modules that are ArkTS (`oh-package.json5` +
`build-profile.json5`, `.ets` sources) and also carry an `@ohos/` `bundle.json`
are HAP roots too — the manifests decide, not a project-local `hvigorw`. To force
HAP mode on a tree that lacks the manifests, set `PBT_HAP=1` **on that command
only**:

```bash
PBT_HAP=1 pi-pbt -p "Run property-based testing on this repository; write results to pbt-out/."
```

Do not export `PBT_HAP` in a shell profile: a leftover `1` forces HAP mode on
later C++ campaigns.

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
`hook-run` checks one commit and defaults to `quick`; `watch` also defaults to
`quick`, but caps each polling batch as described below. Other campaign entries
default to `standard`.

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

### Code coverage reports

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
| Kotlin | Kover's own HTML report (`build/reports/kover/html`), adopted if the build produced one | the Kover Gradle plugin applied in the build |

Instrumentation is switched on by adding each toolchain's flags to the
environment *before* the campaign builds anything (coverage is a build-time
decision — it cannot be added afterwards). Whatever you set yourself is left
alone. If a tool is missing, or a build is not instrumented, that row simply
says so in the report: **coverage never fails a campaign.**

Coverage requires an instrumented target and profiles readable by the collector.
For device/emulator runs, arrange profile retrieval to the host; do not infer
zero coverage from missing files. `host_product` makes local execution simpler,
but successful tests alone do not prove instrumentation produced usable data.

### What the report adds to `REPORT.md`

`COVERAGE.md` records which functions the campaign **claims** to have written
properties for. Coverage records what **ran**. The report crosses the two:

- *executed* — the claim is backed by execution
- *claimed but never executed* — the property does not exercise the function it
  is filed under; this is the finding worth acting on
- *no evidence* — the symbol was in no report at all, usually an uninstrumented
  target rather than an untrue claim

**Evidence levels.** The `coverage_gaps` tool a campaign calls during its
contract-surface sweep now reports one of three levels and says which: **line**
(native instrumentation ran), **file** (no instrumentation, but the changed or
candidate functions' symbols were checked against the test binaries — a
function no binary links cannot have executed, whatever the ledger claims), or
**none**, with the reason (GN/Bazel ignore the exported flags; coverage turned
off; no reporter; no test binary found). A "no run yet" answer is not a sweep
round; the other three answers are, and the level goes into `COVERAGE_STATUS.md`
and the report's Summary. File-level evidence costs nothing and works on every
architecture and every effort tier.

Turn the whole thing off with `PBT_CODE_COVERAGE=0`.

### Build first, then test: `build-run` (bring your own build command)

For a project whose build you already know how to drive — an OpenHarmony
component, a big monorepo, anything with an official build entry — hand that
command to pi-pbt and let it gate the campaign.
For live logs, jump to [build-run live JSON logs](#build-run-live-json-logs) in this section.

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

There is no single `--build-cmd` for the whole OpenHarmony tree. Reuse this
**shape**: `--workdir` the provisioned workspace, `--repo` **this** component,
and a preflight target that **already exists in this component** (for example
`base_unittest` on `ace_engine`). Do not paste `ace_engine` / `base_unittest`
into another part. After `out/` is warm, ninja of that one existing target is
incremental; a full product rebuild every campaign is not required.

**Speeding up repeated OH builds.** `--ccache` is already hb's default, so it
only states the intent. The parameter that measurably helps is `--fast-rebuild`,
which skips the prepare/preloader/loader/gn phases and starts at ninja: the same
no-op incremental `base_unittest` build measured **27 s** without it and **13 s**
with it.

Keep it out of the preflight above. OH's own help restricts it to runs with "no
change for gn related script", and a campaign changes exactly those files when it
adds `test/pbt/<subpath>/BUILD.gn` and registers the target in `bundle.json`
`build.test`. Use it only for a rebuild whose gn inputs are unchanged, and drop
it again for the first rebuild after the campaign generates its test target:

```bash
./build.sh --export-para PYCACHE_ENABLE:true --product-name host_product \
  --build-target base_unittest --ccache --no-prebuilt-sdk --fast-rebuild
```

Two parameters to leave alone: `--jobs` is deprecated in hb, and
`--load-test-config` (default on) is what reads `bundle.json`'s `test` field,
so disabling it hides the generated PBT target.
`--gn-args enable_notice_collection=false`, which OH lists as a speed-up, is
aimed at full image builds; on this unittest preflight it measured 26 s against
the 27 s baseline and forces a gn regen, so it is not worth adding.

For a HarmonyOS tree driven by `build_system.sh` rather than `build.sh`, check
`--help` on that script before reusing these flags: the parameter set is not
verified here, and an unsupported flag fails the preflight instead of speeding
it up.

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

**HarmonyOS HAP / ArkTS.** Gate on a HAP root (`oh-package.json5` +
`build-profile.json5`). That sets `PBT_HAP=1` and selects Hypium + PandaCheck
in `ohosTest`, then `aa test`. It does not select host `ark`. `hvigorw` may be
the project wrapper, on `PATH`, or under DevEco — see
[HAP / ArkTS](#harmonyos-hap--arkts-apps).
When the only `hvigorw` is inside a DevEco install, `build-run` puts its
directory on `PATH` for that run (and sets `DEVECO_SDK_HOME` if unset), so the
bare `hvigorw …` command below works.

Treat the following as **install prerequisites**. If they are done, `build-run`
should start. If one is skipped, the gate fails naming that step. Full list:
[ArkTS PBT — Before you run](pbt/arkts.md#before-you-run-prerequisites).

- Writable DevEco SDK (`local.properties` `sdk.dir` / `DEVECO_SDK_HOME`). A
  system `/opt/devecostudio/sdk` that is not writable will fail hvigor 00308018.
- `compileSdkVersion` matches that SDK. Align by hand; pi-pbt does not rewrite it.
- `src/ohosTest/module.json5` exists on the module in `--build-cmd`.
- `ohpm install` succeeds (`@ohos/hypium` resolves).
- HAP: `assembleHap … @ohosTest`. HAR (`harTasks`): `genOnDeviceTestHap … @ohosTest`.
  `assembleHap` is not a task on a HAR-only project.

On a fresh clone run `ohpm install --all` first. If `signingConfigs` points at
certificates that are not on this machine (common in open-source HAPs), the
gate stops at `SignHap` after a successful compile and prints a `HINT`: remove
`signingConfig` from the product you build so hvigor emits `*-unsigned.hap`;
the campaign signs with the SDK debug certificate. pi-pbt never edits it for you:

```bash
pi-pbt build-run \
  --workdir /path/to/hap-app \
  --repo /path/to/hap-app \
  --build-cmd "hvigorw assembleHap --mode module -p module=entry@ohosTest -p product=default --no-daemon" \
  --lang en
```

The build command is **your prepared input**, not something the agent figures
out. `build-run` executes it in `--workdir` first (full log in
`<out>/build.log`); if it fails, PBT never starts and the process exits `3` —
fix the tree or the command and re-run.

`--workdir` and `--repo` must belong to **one checkout**: either the same path,
or one containing the other (the build root normally sits above the module, as
in the OpenHarmony example above). Two sibling checkouts are rejected before the
build runs, with an error naming both paths. This is deliberate — provisioned
workspaces often differ only by a hash suffix, and building in one while testing
the other silently wastes the whole build, since nothing produced there can be
tested, covered or reported here. The same rule applies to the build commands a
campaign runs: one that `cd`s into an unrelated checkout is blocked. This is a **preflight** gate, not a
post-campaign bug verdict: after preflight succeeds, direct `build-run` uses the
ordinary pi process exit. Read its reports, or use MCP's `pbt_report` for a
managed result. If it succeeds, the campaign starts in
`--repo` under a build contract: rebuilds reuse exactly your command (only an
existing target may be replaced by the newly generated test target), and a
failing rebuild is a STOP condition recorded in `REPORT.md`. The agent will not
switch build systems or derive an alternative way to compile. `--scope` limits
the campaign to that path; omit it only for a full-repo run.

`--scope` and `--func` limit the campaign (not a sandbox). With no `--diff`/`--commit`, `--func` **is** the change surface:

```bash
pi-pbt build-run --build-cmd "…" \
  --scope path/to/file.cpp \
  --func Foo::Bar
```

`--scope <path>` is a file or directory (omit only for a full-repo run).
`--func <name>` (optional) names **one symbol** in that path. It **requires
`--scope`** and must appear **after** `--scope` on the command line. Without
`--func`, every PBT-worthy function in `--scope` is in play. Without `--diff` or
`--commit`, that symbol is the change surface (not `git HEAD` of `--repo`).

`--prompt "<text>"` (short form `-p`) and `--prompt-file <path>` append your own
context to the campaign prompt: domain rules, which of two similar helpers is the
real entry point, which directory is legacy, which invariant is worth asserting,
what to ignore. They are repeatable (values merge in command-line order) and the
block is appended after the SOP entry — it narrows scope and adds oracles, and
cannot override the build contract. Use it when the tree alone does not tell the
agent what you already know:

```bash
pi-pbt build-run --build-cmd "…" --scope src/decoder.cpp \
  -p "decode() is the only public entry; the legacy_* helpers are dead code"
```

In `build-run` / `hook-run`, `-p` is this extra-context flag, **not** pi's print
switch: those subcommands already run headless. `watch` accepts
`--prompt` / `--prompt-file` and forwards them to every `hook-run` it spawns.

`--out`, `--lang`, `--effort` (default `standard`), `--provider`,
`--model`, `--thinking`, `--mode text|json|json-full`, and `--tui` work as on `hook-run`.
`--mode` defaults to `text`; selecting `json` changes the stdout format, not the
campaign or its exit behavior.

#### build-run live JSON logs

**Requires pi-pbt v0.1.19 or newer**; check `pi-pbt --version` first. Add
**`--mode json`** to `build-run`; it already implies headless output, so no print
flag is needed (`-p` here is the extra-context flag above, not pi's print
switch). After the build
passes, model updates and tool-call/results stream as JSONL (one JSON event per
line), without waiting for the final answer or tailing session files. The default
`--mode text` is not a complete live tool log.

This example reuses this section's OH `host_product` build. Replace the workspace
path and configure a model first, then run in Bash or zsh. For other projects,
keep their own `--build-cmd` instead of copying the OH build:

```bash
set -o pipefail
export PBT_OH_WORKSPACE=/path/to/openharmony
LOG_DIR=$(mktemp -d)
printf 'Logs: %s\n' "$LOG_DIR"

pi-pbt build-run \
  --workdir "$PBT_OH_WORKSPACE" \
  --repo "$PBT_OH_WORKSPACE/foundation/arkui/ace_engine" \
  --build-cmd "./build.sh --export-para PYCACHE_ENABLE:true --product-name host_product --build-target base_unittest --ccache --no-prebuilt-sdk" \
  --scope frameworks/base/geometry \
  --lang en \
  --mode json \
  2>"$LOG_DIR/stderr.log" | tee "$LOG_DIR/events.jsonl"
```

- JSON events appear in the terminal and are saved to `$LOG_DIR/events.jsonl`.
- Preflight build output and operational diagnostics go to `$LOG_DIR/stderr.log`.
  To watch the build in another terminal, use `tail -f <printed-log-directory>/stderr.log`.
- The full preflight log also remains in `<out>/build.log`. With no `--out` in
  this example, it is `$PBT_OH_WORKSPACE/foundation/arkui/ace_engine/pbt-out/build.log`.
- Do not add `2>&1` to the pipeline: it mixes plain diagnostics into JSON.
  Events can contain source code and tool arguments/results; review sensitive
  content before publishing. `events.jsonl` is **not** the final `report.json`.
- `set -o pipefail` prevents `tee` from hiding failure; JSON mode does not change
  build-run's exit semantics. Explicit `--tui` conflicts with JSON; inherited
  `PBT_HOOK_TUI=1` is overridden by JSON mode.

#### Interactive skill alternative

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

`--mode json` is a **compact event log**: one line per thing the agent did —
what it said (`message_end`, with its tool calls, stop reason and token usage),
which tool started with which arguments (`tool_execution_start`), what came back
(`tool_execution_end`), and the retry, compaction and settle transitions around
them. The token-level deltas pi streams (`text_delta`, `thinking_delta`,
`toolcall_delta`), the partial tool output (`tool_execution_update`,
`bash_execution_update`) and the duplicated full snapshots (`turn_end` and
`agent_end` re-send messages already emitted) are dropped, and long strings
inside tool arguments and results are clamped so a single file write cannot bury
the log. Thinking blocks are replaced by a `thinkingChars` count.

`--mode json-full` is pi's raw session stream with none of that filtering — every
delta and every full-message snapshot. Use it when you are consuming the complete
SDK protocol rather than reading a log; expect one line per streamed token.

**Telling the campaign what changed.** `build-run` takes `--diff <file>` (a
unified diff, e.g. an MR export) or `--commit <sha>`; `hook-run` already has its
commit. The in-session entries read the same thing from the invocation prose: a
`.diff`/`.patch` file name that exists in the repo, a commit sha or `a..b`
range, "本次提交"/`HEAD`, "未提交改动", or an MR number whose `<n>.diff` export is
in the repo. From it pi-pbt derives the **change surface** — the functions the
diff actually touched, with error-handling changes marked — writes it to
`pbt-out/CHANGE_SURFACE.md`, hands it to the agent as the primary target, and at
close-out refuses to finish while a changed function has no property or an
error-handling change has only success-path properties. That is the difference
between testing the module and testing the change.

**Turning coverage off, and adding your own instructions.** Both campaign
subcommands accept `--no-coverage` (skip native-coverage instrumentation and
reporting entirely); your own instructions — "only these three functions", "do
not touch qemu" — go through `--prompt` / `-p` / `--prompt-file`, documented
under `build-run` above. The in-session entries have no command line, so they read the same two
settings from the invocation prose: an explicit `effort: quick` /
`effort设为 quick`, and `coverage: off` / `不要覆盖率`. A flag or the
`PBT_EFFORT` / `PBT_CODE_COVERAGE` environment variable always wins over prose.

Coverage is worth turning off when the build system does not consume
`CFLAGS`/`CXXFLAGS`/`LDFLAGS` at all. GN and Bazel take their compile flags from
their own configuration, so the exported flags never reach the compiler and
`pbt-out/code-coverage/` stays empty; pi-pbt now says so in the campaign's
system prompt rather than leaving an empty directory to be interpreted. Do not
flip the build system's own global coverage switch to compensate — on
OpenHarmony that is `use_clang_coverage`, which rebuilds the whole tree with
ccache defeated.

For automation that needs the SDK event stream, add `--mode json` (the default
is `--mode text`). JSON mode reserves stdout for SDK events; preflight output,
scan diagnostics, and other operational messages go to stderr, while
`build-run` still keeps the complete preflight in `<out>/build.log`. The stream
can contain prompts, model output, and tool inputs/results, including sensitive
repository content, so store and publish it accordingly. It is a live event
stream, not the campaign `report.json` and not the session files under
`~/.pi-pbt/agent/sessions/`.

`--mode json` and explicit `--tui` are incompatible. An explicit JSON mode does,
however, override an inherited `PBT_HOOK_TUI=1`. Neither output mode changes the
exit codes above, and `build-run` still returns `3` only when its preflight
fails. To capture both channels without corrupting stdout JSON:

```bash
set -o pipefail
pi-pbt hook-run <sha> --repo /path/to/repo --mode json \
  2>stderr.log | tee events.jsonl
```

Do not use `2>&1`: that mixes diagnostics into the JSON stream. See the
[subcommand reference](subcommands.md#machine-readable-sdk-event-stream) for
`build-run` syntax and details.

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

### Watching a repo: test new commits

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
background instead. It checks for new commits on a timer and tests the selected batch
with results going to its log:

```bash
# watch local commits on the current repo, every 30s (put it in tmux/systemd)
pi-pbt watch --repo /path/to/repo --lang zh --provider xai-oauth --model grok-4.5

# watch pushes to the remote tracking branch instead
pi-pbt watch --repo /path/to/repo --fetch --branch master --interval 60 --lang zh --provider xai-oauth --model grok-4.5
```

The starting point is the latest commit at startup (commits that already existed
are not tested retroactively). The CLI watcher tests at most the **newest 10**
commits per poll, oldest first within that batch; older commits in a burst are
dropped and logged. It is a monitor, not an every-commit guarantee or a CI gate.
Each child run's verdict is recorded in the log. All `hook-run` options listed here (`--out`,
`--workdir`, `--spec`, `--lang`, `--scan-root`, `--effort`, `--provider`,
`--model`, `--tui`) pass through. `--tui` (or `PBT_HOOK_TUI=1`) runs it in the full
interactive interface in the same terminal (good for demos, not for CI: it stops
and waits for `/quit` after each round).

### Delegation from another coding agent (MCP)

If your day-to-day agent is Claude Code or Codex, it can delegate PBT to pi-pbt
over the [Model Context Protocol](https://modelcontextprotocol.io). You can
**keep editing the same checkout only in worktree mode**, without `build_cmd`. `pi-pbt mcp` serves MCP over stdio from
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
| `pbt_report` | verdict (`passed` / `bugs_found` / `failed`), report summary, structured result from `report.json`, bug-report index, patch presence |
| `pbt_cancel` | cancel a queued or running campaign (idempotent) |
| `pbt_watch_start` | start one campaign per new commit on a branch |
| `pbt_watch_stop` | stop a watch (by default cancelling its in-flight run) |

Without `build_cmd`, the run tests an immutable commit in a detached
`git worktree`, so you can continue editing the original checkout. With
`build_cmd`, it tests **in place**: do not edit that checkout until the run ends.
The build-command mode is described below.

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
created. Do not edit the checkout until the run finishes. The command runs first and a non-zero exit stops the campaign before
any agent work, so a broken build costs you seconds rather than an exploratory
build hunt. Set `build_workdir` when the build must run from a directory above
the module under test, e.g. the OpenHarmony source root while the scope is one
component. Use it when you know your build command; leave it off to keep the
worktree isolation. When the run finishes, any worktree is removed; what survives lives
under `~/.pi-pbt/runs/<run-id>/` (override with `PI_PBT_RUNS_DIR`): `run.json`,
`events.jsonl`, both logs, the campaign artifacts, and a `changes.patch` holding
everything the campaign wrote in its worktree. All of it is also readable
through MCP resources at `pbt://runs/<run-id>/...` (`REPORT.md`, `report.json`,
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

### A HarmonyOS app on a phone (`pi-pbt kea`, experimental)

`pi-pbt kea` black-box tests an already installed HarmonyOS app on a
USB-connected phone through its GUI. This mode is experimental; see the
[bilingual Kea guide](kea.md) for prerequisites, configuration, commands,
artifacts, and Kea-specific environment variables.

### Environment variables

| Variable | Effect |
|---|---|
| `PBT_LANG=zh` / `en` | Chinese or English narration and artifacts; also `--lang` on subcommands. `en` still injects an English directive when the campaign prompt is Chinese |
| `PBT_SCAN_ROOT=/path` | the repo to scan, for setups where the working directory is a clean copy (git hooks, CI) |
| `PBT_OH_WORKSPACE=/path` | a pre-built full OpenHarmony source environment (source tree + toolchain + built dependencies) to reuse, instead of working out how to build the component standalone. **Detected automatically** when the repo sits inside such an environment (a parent directory with both `.repo/` and `out/`) — set it explicitly only to override; the startup log prints which one is in use |
| `PBT_HAP=1` | Set **automatically** by `build-run` when `--repo`/`--workdir` is a HAP root (`oh-package.json5` + `build-profile.json5`). ohosTest uses Hypium + PandaCheck (`import { check, property } from 'PandaCheck'`) and `aa test`. Host `ark` is `PBT_PANDA=1`, and only when the tree is not a HAP root. Do not set in a shell profile |
| `PBT_HOOK_TUI=1` | same as `hook-run --tui` / `watch --tui`: run in the interactive interface (needs a terminal; waits for `/quit`) |
| `PBT_EFFORT=quick\|standard\|thorough` | how deep a campaign digs (see [effort tiers](#how-deep-it-digs-effort-tiers)); also `--effort` on `hook-run` / `watch`, which takes priority. Default: `quick` for `hook-run`/`watch`, `standard` elsewhere. The experimental Kea mode also uses this tier when its own depth is not configured; see [the Kea guide](kea.md). |
| `PBT_TEST_JOBS=8` | how wide tests and builds run (see [parallelism](#how-wide-it-runs-parallelism)); a number, `max`, or `auto` (default: the machine's idle cores). `PBT_TEST_JOBS=1` forces serial — required when reconfirming a failure before filing it as a bug |
| `PBT_CODE_COVERAGE=0` | turn off native code-coverage instrumentation and reporting (see [code coverage](#code-coverage-reports)); on by default, and it degrades silently when a toolchain is missing |
| `PBT_CLOSEOUT_REVIEW=off` | skip the independent close-out review (`pbt-out/REVIEW.md`) a campaign otherwise must run, through the bundled pi-subagents, before it may write `report.json`. On by default; also `0` / `false`. The result then records `"review": {"status": "not-reviewed", "reason": "disabled"}` |
| `PBT_CLOSEOUT_REVIEW_TIMEOUT=<minutes>` | time limit for each reviewer child of the close-out review (default 20). The review runs as parallel children, one per bug report, one per batch of three passing properties and one per batch of three caveats; a child that fails or times out is re-run alone on the next `closeout_review` call. How many run at once adapts: one more after each success, half after a failure, and half as soon as a child reports a rate-limit backoff (pi-subagents' provider diagnostics). The review is incremental: a later call reviews only the bug reports, passing properties and caveats that changed, or whose cited test, source or documentation files changed, and keeps the others with their findings and responses |
| `PBT_SCAN_DELEGATION=off` | keep the Scan in the main agent instead of `scan_modules`' read-only workers (bundled pi-subagents): no `pbt-out/SCAN.json` / `SCAN_COVERAGE.md`, and a change campaign is not held to having read every function of its change surface. On by default; also `0` / `false` |
| `PBT_SCAN_TIMEOUT=<minutes>` | time limit for each scan worker's first attempt (default 10; its retry gets twice that). Workers run with the same adaptive window as the close-out review; a unit still unread after the retry gets one more round, together with the related functions the workers reported (change campaigns) |
| `PBT_VERDICTS=off` | do not record per-property verdicts (and so carry nothing over). After a delivered campaign (a valid `report.json` written by that campaign that passes delivery's checks), pi-pbt records each property's status, the commit tested, the effort tier, a failing case's seed, its ledger entry, its close-out review, and a fingerprint of the ledger entry, the test file, the SUT file and the files the entry cites, under `~/.pi-pbt/verdicts/<repository>/` (`PI_PBT_VERDICTS_DIR` moves it). Also `0` / `false` |
| `PBT_CARRY=off` | run every property again. By default a full campaign starting without a `PROPERTIES.md` carries over the passing properties an earlier campaign verified at the same or a higher effort tier, when nothing they are bound to has changed (byte for byte: the entry, the test file, the SUT file, every cited file, the SUT's whole module — the nearest directory with a build file — minus tests and documentation, every module their imports reach, and the build configuration and lockfiles up to the repository top; toolchain versions are not bound), run with the same test target and build command (a `build-run` with another `--build-cmd` carries nothing), and written in the campaign's output language: they are written into the ledger with a `- Carried over:` line, keep their close-out review, are not run again, and are marked `carriedOver` in `report.json`, `**Carried over:**` in `REPORT.md` and in the HTML report. A test written outside the repository — in `hook-run`'s clean-room workdir, a managed run's work directory, or a `build-run` workdir above the module — is bound the same way, with what it imports and the rest of its module in the workdir (build files and linked sources; other tests excluded); those files are kept by content in the verdict store (`blobs/`), put back into the next campaign's workdir before the check, and removed again when they do not carry. Not under a hard `--scope`. Also `0` / `false` |
| `PBT_CARRY_SAMPLE=<share>` | the spot-check of carried-over verdicts (default `0.1`; `0` turns it off): each heading that could be carried is instead re-run with this probability, in the ledger as `approved`. A re-run that does not pass again means the fingerprints missed something in this repository: carry-over is then off for it until a campaign that carries nothing has re-verified the records |
| `PBT_BARE=1` | run as plain pi: no orchestration extension, no bundled skills, no campaign guards. Exists for A/B-measuring the harness's own contribution (benchmarking); not for normal use |
| `PI_PBT_RUNS_DIR=/path` | where [`pi-pbt mcp`](#delegation-from-another-coding-agent-mcp) persists run state and artifacts (default `~/.pi-pbt/runs`; must be outside the repo under test) |
| `PBT_OUT_DIR=/path` | where the campaign's artifacts (`PLAN.md`, `PROPERTIES.md`, `COVERAGE.md`, `REPORT.md`, `report.json`, `bug_reports/`) live. **Set automatically** by `build-run` and `hook-run` from their `--out`, so you normally never set it yourself; it exists so the artifact-driven checks read the same directory the campaign writes. Leave it unset for a plain `pi-pbt -p` campaign, which uses `<cwd>/pbt-out`. Do NOT export it in a shell profile: a stale value follows every later campaign, and an inherited one that points somewhere else is exactly the split-brain it was added to prevent |
| `PBT_MCP_MAX_CONCURRENT=2` | how many MCP-delegated campaigns may run at once (default 1; extra runs queue) |
| `PBT_PHASE_MODELS='{"plan":"anthropic/claude-opus-5-5"}'` | route a different model per campaign phase (`scan`, `plan`, `test`, `review`), as `<provider>/<modelId>`. Unset means one model throughout, which is the default. The phase is read off the artifacts under `pbt-out/`, never self-reported by the agent; an unknown model or unconfigured provider silently leaves the campaign on its current model. There is no built-in routing table on purpose: `scan` and `review` are where the campaign decides what the contract is and whether a failure is real, so downgrading them buys tokens at the price of a false pass |

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

- **The agent says it will call a tool, then stops** — an explicit immediate
  promise to call an available tool, without a real tool call, triggers up to
  **two corrective retries per campaign**. The retry asks for actual arguments
  and a tool call, not another promise. User advice, quoted examples, real tool
  calls, cancellations and provider errors do not trigger this rule. If the
  model instead returns an empty response, the separate recovery runs **once**. Check whether `REPORT.md`
  and `report.json` are complete; a promise to continue is not a completed run.
  Look in the selected output directory (`pbt-out/` by default) for
  `recovery.json`, and in `~/.pi-pbt/agent/sessions/` for the session log. A
  recovery attempt is not guaranteed, and cancellation or provider errors are
  not retried by this mechanism. If it still stops, check the configured model's
  context limit and the provider's error/usage limits, then start a new run with
  a smaller scope or a suitable model. Keep the logs when reporting the issue;
  repeatedly sending “continue” is not a fix.

### Interrupted campaigns

A final provider error or cancellation is an interruption, not a completed
campaign. Headless pi-pbt exits with code `2` and preserves tests, artifacts and
`interruption.json` in the campaign output directory. The diagnostic records
the provider error, selected model/thinking level, saved session path, working
directory and the campaign's `PBT_*` settings, with resume arguments. Fix
provider quota or configuration before resuming. Do not reset the repository or
discard the generated tests. Continue the campaign with:

```bash
pi-pbt resume --out pbt-out                          # one attempt
pi-pbt resume --out pbt-out --attempts 6 --wait 1800 # keep resuming quota/rate-limit/provider stops
```

`resume` relaunches the saved session with the recorded provider, model,
thinking level, working directory and campaign settings (an allow-list of
`PBT_*` variables such as `PBT_SCOPE`, `PBT_BUILD_CMD`, `PBT_EFFORT`).
Variables you set in the calling shell win over the recorded ones, except
`PBT_OUT_DIR`, which stays the directory holding `interruption.json`. A build
command that looks like it embeds a credential is not recorded; `resume` then
asks you to export it again. A HAP campaign's DevEco `PATH`/`DEVECO_SDK_HOME`
is detected again rather than stored. The campaign's source root (for a
`hook-run`, the repository rather than the clean-room workdir) and a
function-scoped target are restored through `PBT_SOURCE_ROOT` and
`PBT_FUNCTION_TARGET`, a scope stated in the prompt through `PBT_RESUME_SCOPE`,
and the campaign start through `PBT_RESUME_STARTED_AT` (so the continuation
appends to the existing `guards.jsonl`). `resume` sets these for the
continuation; do not set them by hand. The language and change source the
original prompt implied come back as `PBT_LANG` and `PBT_CHANGE_SOURCE`. A
resumed `--cov-mode full` sweep continues where it stopped instead of starting
the sweep over. A headless `-p` run with `PBT_OUT_DIR` set renders its coverage
footers there instead of `./pbt-out`. It refuses a configuration error
(for example a rejected thinking level, also when it followed a quota error)
unless you pass `--thinking <level>`, or
`--force` after fixing the provider configuration. With `--attempts`, it only
retries quota, rate-limit and provider interruptions, waits `--wait` seconds
between attempts, and stops when a run ends without a new interruption. Its exit
code is that of the last resumed run, or `2` when nothing could be resumed. The
continuation is a headless `-p` run on the same session, not a new `hook-run`;
validate the result with `pi-pbt finalize`. No automatic model switch or
thinking reduction is performed. Transient failures handled successfully by SDK
retries do not leave an interruption diagnostic.

Unattended campaigns need more patience with rate limits than pi's interactive
defaults. When `~/.pi-pbt/agent/settings.json` has no `retry` key, pi-pbt writes
one on launch: 7 agent-level retries from a 4 s backoff (about 8.5 minutes in
total), and 3 provider-level retries (`retry.provider.maxRetries`) that wait the
server's `Retry-After`, capped at 2 minutes (`retry.provider.maxRetryDelayMs`).
A `retry` key you set yourself is never changed. One that is still exactly an
earlier pi-pbt default is upgraded to the current one.

Before a full headless campaign returns success, pi-pbt revalidates report.json,
source/regression/Gradle evidence and derives the HTML again. Missing or invalid
delivery exits `2`, even when a file operation bypassed a tool-call pattern.
This is distinct from hook-run exit `1` for a completed bugs-found campaign.


### Finalize existing campaign evidence without a model call

From the SUT repository root, run `pi-pbt finalize --out pbt-out --lang en`
(or `--lang zh`). It validates schema, source/regression and Gradle XML evidence,
checks demotion contradictions, regenerates HTML and refreshes only its owned
coverage footer. It does not author tests, alter report.json or rerun the SUT.
Exit `0` means validated delivery, even when the report contains bugs; exit `2`
means missing/invalid evidence or rendering failure. No provider credentials or
LLM request is needed for this step. It cannot turn an interrupted test run into
verified results by filling missing facts.
