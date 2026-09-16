# pi-pbt-dev skill — User Tutorial

> Let your code agent run property-based testing (PBT) on your code: a FULL
> campaign over the whole project, or an INCREMENTAL campaign over the changes
> you just made. This tutorial takes you from zero to your first bug report.
> Chinese version: [skill-pi-pbt-dev.zh.md](skill-pi-pbt-dev.zh.md).

## 1. What this skill is

`pi-pbt-dev` is the way to wire pi-pbt into your everyday code agent. Once
installed, you can tell the agent "test the changes I just made" and it will run
a PBT campaign over the changed scope and present the findings to you.

Two modes, chosen by the agent from your intent:

| Mode | You say | What gets tested |
|---|---|---|
| **Full** | "run a PBT campaign on this project" | whole project / a named module |
| **Incremental** | "test the code I just changed" | only the files changed recently |

The architecture is a **thin shell**: the skill contains only the protocol text
(`SKILL.md`) and three helper scripts; the actual PBT engine is the `pi-pbt`
binary (installed separately). Updating the skill updates text; updating the
engine updates the binary — they do not interfere.

### Process-backed skill versus MCP delegation

This tutorial covers the **installed skill** path. The host agent starts
`scripts/run-pbt.mjs`, which starts one `pi-pbt -p` child process and waits for it
to exit before reading the result. Progress lines make that wait visible, but the
contract is synchronous: this is not a background delegation, and the same agent
turn does not continue editing while the campaign runs.

MCP is a separate, skill-free integration for delegation. Its `pbt_start` tool
returns a `run_id` immediately and a pi-pbt run manager owns the independent
campaign process, so the host agent can continue development and later call
`pbt_status` / `pbt_report`. Normal MCP runs test an immutable commit in a
detached worktree; `include_uncommitted: true` first freezes the current changes
into a snapshot, while a supplied `build_cmd` deliberately runs in place. The
MCP path is introduced here before later references to `pbt_report` and
`pbt://...` result resources; see
[Delegation from another coding agent](installation.md#delegation-from-another-coding-agent-mcp)
for setup and the isolation rules. Install the skill for the synchronous workflow
described below; configure MCP when you specifically want asynchronous
delegation. They can coexist.

## 2. Prerequisites

| Item | Requirement | Notes |
|---|---|---|
| Node.js | >= 18 | the skill's helper scripts are written in Node (the `pi-pbt` binary itself needs no Node) |
| `pi-pbt` binary | installed | see [Step 1](#3-step-1-install-the-pi-pbt-engine) |
| git | recommended | git projects get an "objective scope layer" (changed files listed automatically); non-git works too — the agent lists the files from its session memory and asks you to confirm |
| Toolchain for the language under test | as needed | Python needs `python3`+`pip`, C/C++ needs `cmake`+compiler, Rust needs `cargo`… (pi-pbt actually compiles and runs the tests it writes) |
| LLM API | configured | campaigns are driven by a large model — see [Step 1](#3-step-1-install-the-pi-pbt-engine) |

## 3. Step 1: Install the pi-pbt engine

Official release platforms are **Linux x64 / Linux arm64 / macOS arm64** —
download the matching zip from your distribution channel (details in the
[installation guide](installation.md#2-download-and-install)). Each archive
extracts to a same-named directory; run its installer so the executable and its
bundled `fd`/`rg` tools are installed together:

```bash
PLATFORM=linux-x64   # or linux-arm64 / macos-arm64
sha256sum -c "pi-pbt-${PLATFORM}.zip.sha256"   # optional integrity check
unzip "pi-pbt-${PLATFORM}.zip"
cd "pi-pbt-${PLATFORM}"
./install.sh
```

On macOS, use `shasum -a 256 -c` if `sha256sum` is unavailable, and clear
Gatekeeper quarantine as described in the installation guide.

**Windows**: no official Windows binary yet. Alternative path — build from source
and install the packed tarball:

```bash
git clone <pi-pbt repo> && cd <pi-pbt repo>
npm ci --ignore-scripts
npm run build
npm pack --ignore-scripts
npm i -g ./pi-pbt-<version>.tgz     # gives you the pi-pbt command
```

> **Note**: `npm i -g pi-pbt` is currently **not available** — the package is
> not published on the npm registry; use the official paths above.

### Configure a model

pi-pbt's config directory is `~/.pi-pbt/agent/` (created automatically on first
run). The two fastest paths:

```bash
# Path 1: API key via environment variable — works immediately
export ANTHROPIC_API_KEY=sk-ant-...   # or OPENAI_API_KEY / GEMINI_API_KEY / ...
pi-pbt --list-models                   # a model list means you are configured
```

```text
# Path 2: interactive login
pi-pbt            # then /login to sign in, /model (Ctrl+L) to pick a model
```

Custom models/proxies (OpenAI-compatible) go in `~/.pi-pbt/agent/models.json`:

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

Verify:

```bash
pi-pbt --version
pi-pbt --list-models
```

> **Use your strongest model.** The hardest step of PBT is deriving "what must
> always hold" from the code — pure reasoning. A weak model writes empty
> properties like "calling it does not crash": the tests pass, the report looks
> fine, and nothing was really tested.

### How the skill finds pi-pbt (no configuration needed by default)

The skill's scripts resolve the engine in this order:

1. the `PI_PBT_BIN` environment variable (if set and the file exists)
2. `pi-pbt` on `PATH` (npm global install, or the release binary placed on PATH)

Only when neither matches does the skill error out and point you at the install
instructions. So the default needs **no configuration at all**; set `PI_PBT_BIN`
explicitly only when the binary lives off PATH (e.g. a zip extracted into a
custom directory):

```bash
export PI_PBT_BIN=/path/to/pi-pbt      # Windows: set PI_PBT_BIN=C:\path\to\pi-pbt.exe
```

## 4. Step 2 — Install the skill

The skill ships **inside the pi-pbt distribution** (embedded in the binary, or
under `skills/` in the npm package), so installing it is one command — the files
are copied out of pi-pbt into your agent's skill directory:

```bash
pi-pbt skill-install                # auto-detects which agents are installed
pi-pbt skill-install --host claude  # or opencode / codex / codeagent / chrys
pi-pbt skill-install --host project # project-level (shares with collaborators)
pi-pbt skill-install --dir /custom/path  # any skill directory
pi-pbt skill-uninstall              # remove it again (same flags, idempotent)
```

Supported host locations:

| Host | Location |
|---|---|
| Claude Code | `~/.claude/skills/pi-pbt-dev/` |
| opencode | `~/.config/opencode/skills/pi-pbt-dev/` |
| Codex CLI | `~/.agents/skills/pi-pbt-dev/` (user) or `<repo>/.agents/skills/pi-pbt-dev/` (project) |
| codeagent | `~/.cac/skills/pi-pbt-dev/` (user) or `<repo>/.cac/skills/pi-pbt-dev/` (project) |
| chrys | `~/.chrys/skills/pi-pbt-dev/` (user) or `<repo>/.chrys/skills/pi-pbt-dev/` (project) |
| Project-level (`--host project`, shares with collaborators) | `<repo>/.claude/skills/`, `<repo>/.agents/skills/`, `<repo>/.cac/skills/`, `<repo>/.chrys/skills/` (all four) |

Re-run with `--force` after updating pi-pbt to refresh an older install.
`skill-uninstall` uses the same target selection as installation: pass the same
`--host` or `--dir` to remove that copy. With no flags it removes the skill from
each currently detectable user-level host location; it does not remember a
historical list of installation targets.

**Codex triggering**: mention `$pi-pbt-dev` explicitly, or browse with `/skills`;
implicitly, the model picks the skill from its description (same as Claude
Code/opencode). Codex deprecated custom slash commands — skills are its only
officially recommended extension path, and this skill's structure (frontmatter +
`scripts/`) is natively compatible. Codex has no native tool-registration API,
but this skill already invokes node scripts via Bash, so that is unaffected.

Smoke test (inside any git repository):

```bash
node <skill>/scripts/diff-scope.mjs   # should print a JSON list of changed files
```

## 5. Step 3 (optional) — Project configuration

**Auto-trigger**: append to the project root `AGENTS.md` (or `CLAUDE.md`) so the
agent runs incremental PBT automatically when a round of edits finishes:

```markdown
## PBT auto-trigger (pi-pbt-dev)

When a round of code changes is complete (write/edit of production code finished,
continuous edits have gone quiet), call the `pi-pbt-dev` skill to run incremental
PBT on the changes:

- changes touching only docs/comments/tests/config → skip
- changes touching production code → follow the skill's mode decision (scope
  must be confirmed, see the skill contract)
- bug summaries found by the campaign must be presented in the conversation,
  and the user asked whether to fix them
```

**Artifacts**: add `pbt-out/` to `.gitignore`:

```bash
echo "pbt-out/" >> .gitignore
```

## 6. Usage tutorial — three scenarios

### Scenario 1 — Full campaign

```text
You:  Run a full PBT campaign on this project and look for hidden bugs.
agent: (loads pi-pbt-dev, decides full mode)
       1. tells you the campaign started and roughly how long it will take
          (the default `standard` effort tier: about 30 min; say "快速验证" for
          quick / "深入测" for thorough)
       2. chooses a fresh round directory such as pbt-out/2026-09-16T10-30-00,
          then runs:
          node <skill>/scripts/run-pbt.mjs --prompt "对 <repo path> 做性质测试(PBT)战役，目标是找出 bug。产物写到 pbt-out/。" --out-dir pbt-out/2026-09-16T10-30-00 --cwd <repo path> --lang zh --effort standard --timeout-sec 1800
       3. waits for that child process to exit; meanwhile reports stage progress
          (start → planning → testing (N properties) → done)
       4. node <skill>/scripts/summarize.mjs pbt-out/2026-09-16T10-30-00
          → read the summary
       5. presents: N bugs found / all clean
```

> During the wait the agent reports stage progress (`[pbt-progress]` lines), so
> you are not left with no feedback for minutes.

### Scenario 2 — Incremental, git project

```text
You:  Test the changes I just made to src/validators.py.
agent: (decides incremental mode)
       1. node <skill>/scripts/diff-scope.mjs   → objective changed-file list +
          changed functions (uncommitted working-tree changes)
       2. cross-checks against session memory: any files this session did not
          touch? If so, asks whether to include them
       3. once the scope is confirmed, prefers the function-level prompt when
          the diff yields function names (narrower LLM context, same depth):
          node <skill>/scripts/run-pbt.mjs --prompt "对下列最近修改的函数做 PBT 战役，目标是找出 bug：for src/validators.py, Function validate_… 产物写到 pbt-out/。" --out-dir pbt-out/2026-09-16T10-30-00 --cwd <repo path> --lang zh --effort standard --timeout-sec 1800
          (falls back to the file-level prompt if no function names were extracted)
       4. presents the result
```

### Scenario 3 — Incremental, non-git project

```text
You:  Test the code I just changed.
agent: (decides incremental; diff-scope reports "not a git repo" → session-memory path)
       1. lists the changed files from session memory (e.g. src/validators.py,
          src/utils.py) and asks you: "I will run PBT on: … — correct?"
       2. after you confirm (or adjust the list):
          node <skill>/scripts/run-pbt.mjs --prompt "对下列文件做 PBT 战役：src/validators.py src/utils.py … 产物写到 pbt-out/。" --out-dir pbt-out/2026-09-16T10-30-00 --cwd <repo path> --lang zh --timeout-sec 1800
       3. presents the result
```

> **Scope confirmation gate**: for incremental runs (especially the memory-based
> list on non-git projects) the campaign starts only after you confirm the scope.
> This is deliberate — missing or mistaking the changed files defeats the point
> of incremental testing.

## 7. Reading the results

The skill chooses a fresh directory for each campaign, such as
`pbt-out/2026-09-16T10-30-00/`, and passes it as `--out-dir`. The campaign writes
to the repository's top-level `pbt-out/` while it is running; `run-pbt.mjs` then
moves the canonical artifacts into that round directory before `summarize.mjs`
reads them.

| File | Contents |
|---|---|
| `REPORT.md` | human-readable summary: modules tested, bugs found, and reproduction details |
| `PROPERTIES.md` | property ledger: formal statement and status (passing / failing / retired) of every property tested |
| `bug_reports/bug_001_*.md` | one file per bug: Law, minimal input, expected/actual, root cause, regression-test location |

The source-code campaign contract also produces `report.json`, a machine-readable
view of the same facts: schema version, tested revision and tier, build evidence,
every property and bug, exact per-bug reproduction commands, and recomputed
totals. Use it when a script, CI job, dashboard, or MCP client needs stable fields
rather than Markdown parsing.

`run-pbt.mjs` archives this run's `report.json` beside its `REPORT.md` in
`--out-dir`; consume that pair, not the mutable repository-root files. Before
starting, old root `REPORT.md`, `report.json` and `bug_reports/` are preserved
under `pbt-out/.previous-result-*/`, not attributed to the new run. An existing
nonempty `--out-dir` is refused: choose a fresh round directory. If a failed run
did not produce JSON, its round directory has no JSON — there is no fallback to
a previous campaign. `summarize.mjs` still reads the Markdown artifacts; it is
not the JSON schema validator.

Managed `hook-run` and MCP campaigns validate `REPORT.md`/`report.json` together;
invalid or missing JSON makes the run incomplete. The full MCP document is
`pbt://runs/<id>/report.json`, while `pbt_report` returns a summary. See the
[report contract](report-schema.md) for fields and validation rules.


**Version note:** the archival fix above is in current source, not the already
published v0.1.18 embedded skill. With that older helper, use a fresh checkout/output
for each campaign and copy the current `report.json` beside its matching
`REPORT.md` immediately after the run; never pair a past round with a later
root-level JSON. Updating the binary alone does not refresh an installed skill:
run `pi-pbt skill-install --force` after installing a release containing the fix.

The summary your agent presents looks like:

```text
**状态:** 发现 1 个 bug | **性质:** 5 passing / 1 failing | **测试:** 6 total

### Bug #1: validate_port(0) returns None instead of raising ValueError（Medium）
- 函数: validate_port in example_lib/validators.py
- 最小反例: validate_port(0)
- 期望: ValueError raised / 实际: Returns None
- 根因: 下界判断只排除了负数，0 漏过
```

When a bug is found you can ask the agent to fix it, then run another
incremental campaign over the fixed files to verify.

## 8. Troubleshooting

| Symptom | Fix |
|---|---|
| `pi-pbt: command not found` | reinstall the engine from [Step 1](#3-step-1-install-the-pi-pbt-engine), and make sure it is on PATH |
| `pi-pbt --list-models` empty / 401 | check the API key and `models.json` as described in [Step 1](#3-step-1-install-the-pi-pbt-engine) |
| campaign timeout (`run-pbt` exit code 3) | use the tier-aligned cap: quick 900 s, standard 1800 s, thorough 3600 s; test a smaller scope or raise `--timeout-sec` deliberately |
| summary says "incomplete" (no REPORT.md) | the campaign agent produced no human report; inspect `<out-dir>.log` and retry. For `hook-run`/MCP, also check that `report.json` exists and satisfies the schema |
| agent does not auto-trigger | check the AGENTS.md snippet is present; an explicit "test the changes I just made" always works |

## 9. Cost and advice

- Each campaign is one full LLM session (minutes, token cost). For very small
  changes (one or two edits) or docs-only changes you can ask to skip — the
  agent's mode decision respects your choice.
- Incremental testing focuses on the changed surface and is cheaper than full;
  use incremental day-to-day and a full campaign periodically (e.g. before a
  release).
- Use a fresh `--out-dir` for every round (for example,
  `pbt-out/2026-09-16T10-30-00/`). The helper does not invent that name for you;
  a reused non-empty directory is refused without deleting its contents. Choose
  a new directory to avoid mixing results.

## 10. Upgrading

| Component | How to update |
|---|---|
| skill (protocol/scripts) | replace the skill directory with the new version |
| pi-pbt engine | re-download the release binary (or rebuild and install the tarball); after updating, re-run `pi-pbt --list-models` to confirm the configuration still works |
