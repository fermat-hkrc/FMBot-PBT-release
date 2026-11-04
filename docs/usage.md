# Usage guide

[中文](usage.zh.md)

pi-pbt has two modes:

| | Mode 1: while you develop | Mode 2: merge-request gate |
|---|---|---|
| Used by | a developer explicitly starting tests in pi-pbt or a coding agent | the CI pipeline |
| Tests | the current change, uncommitted changes included | every change in the merge request |
| Result | a report and generated code in the project's test tree | the exit code passes or fails the pipeline; the report is a pipeline artifact |

First install pi-pbt and configure a model: [installation guide](installation.md).

## Inputs and outputs

| Input | Meaning |
|---|---|
| Documents (optional) | requirement docs, API docs, comments; properties are extracted from them. Without documents the intent is inferred from the code |
| Code | the repository under test; for a merge-request gate, also the range of the change |
| Build command (optional) | how to build the code and the tests; without it pi-pbt works it out from the project |

| Output | Meaning |
|---|---|
| Property list | `PROPERTIES.md`: each property stated formally, with what it is based on |
| Test code | written into the project's own test tree, ready to commit |
| Reports | `REPORT.md`, `report.json`, and one report per bug with a minimal reproducer |

## Mode 1: while you develop

Prepare a project that builds and runs. Start `pi-pbt` in the project directory, explicitly invoke the bundled skill,
and supply the inputs for this run:

```text
/skill:pbt-build-run
repo: /path/to/project
workdir: /path/to/project
scope: src/parser
Requirement document: docs/parser-contract.md; read it first and derive properties from it
Build command: cmake -S . -B build && cmake --build build --target parser_test
Change source: uncommitted changes
effort: standard
lang: en
```

Replace paths and the build command with the project's actual values. If there is no document, explicitly say
“no separate requirement document; derive properties from source contracts.” Choose `quick`, `standard` or `thorough`
explicitly; see [depth](subcommands.md#depth).

The terminal equivalent is below. For a host agent, run `pi-pbt skill-install` once, then invoke `/pi-pbt-dev` in
Claude Code or `$pi-pbt-dev` in Codex to execute it:

```bash
pi-pbt build-run --repo /path/to/project --workdir /path/to/project \
  --scope src/parser --spec docs/parser-contract.md --effort standard --lang en \
  --build-cmd "cmake -S . -B build && cmake --build build --target parser_test"
```

Repeat `--spec` for several documents; omit it when none exists. Testing uses the current checkout: do not edit that
copy until it finishes; keep developing in another checkout. Read `pbt-out/REPORT.md`, review the generated tests in
the project's test tree, then commit them with the code. MCP delegation parameters: [reference](subcommands.md#mcp).

## Mode 2: merge-request gate

1. Install pi-pbt on the runner and configure a model.
2. Check out the merge request; build it with the project's own command if needed.
3. Run the test, with `--base` naming the merge request's base commit and `--spec` passing a requirement document if there is one:

   ```bash
   pi-pbt hook-run <MR head commit> --base <base commit> --spec docs/requirements.md --effort standard --repo . --out /tmp/pbt-out
   ```

4. Decide by exit code: `0` pass; `1` bugs found; `2` report missing or incomplete; `3` build failed.
5. Upload `REPORT.md`, `report.json` and `bug_reports/` from `/tmp/pbt-out` as pipeline artifacts.

Complete GitHub Actions and GitLab CI configurations: [CI examples](ci-examples.md).

## Results

| File | Content |
|---|---|
| `REPORT.md` / `REPORT.html` | the test report: what was tested, what was found, how to reproduce it |
| `bug_reports/` | one report per bug, with a minimal reproducing input |
| `report.json` | the result for programs; format in the [report schema](report-schema.md) |

Reproducing a failure and keeping the generated tests: [reproduction guide](reproducing.md). Every command and
flag: [reference](subcommands.md).
