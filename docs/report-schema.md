# Campaign result contract (`pbt-out/report.json`)

A completed campaign writes customer-facing and technical result artifacts:

- `REPORT.html` is the customer-facing overview. It links to every individual bug page.
- `bug_reports/<slug>.html` is one customer-facing page per confirmed bug.
- `REPORT.md` is the technical Markdown campaign narrative.
- `bug_reports/<slug>.md` is the Markdown version of the individual bug report.
- `report.json` is the structured source for CI, dashboards, and the generated customer reports.

When a valid `report.json` is written, pi-pbt renders the top-level
`REPORT.html` and the per-bug `bug_reports/<slug>.html` from it — and renders
them again at close-out, so the HTML always matches the final validated JSON.
Do not hand-maintain those HTML files. The Markdown is different: `REPORT.md`
and `bug_reports/<slug>.md` are authored by the campaign (the technical
narrative with Verdict, Modules Tested, Design Caveats, change surface and
coverage evidence), and the renderer never overwrites an existing one; it only
fills in a Markdown overview or bug page where the campaign left none.

`hook-run` requires both files in the same recognized artifact root. A missing
or invalid `report.json` makes the check incomplete (exit `2`), even if
`REPORT.md` exists. See the
[`hook-run` exit-code table](installation.md#ci--git-hook-integration).

Before `report.json`, consumers had to infer a verdict from Markdown files. The
JSON makes the result explicit. The shipped validator checks it when the
campaign writes the file and again when `hook-run` applies its gate; validation
returns all detected problems so the campaign can correct them together.

## The two invariants

Everything else is bookkeeping. These two are why the file is worth writing.

**A bug names the property that found it, and the property names the bug back.**
`bugs[].propertyId` and `properties[].bugId` must agree. "Which test caught
this, and how do I run just that one" is then a lookup, not a reading
comprehension exercise. A failing property with no `bugId` is un-triaged: file
the bug or retire the property.

**A bug carries a pasteable reproduction.** `reproduction.build`, `.run`,
and `.workdir` record the commands and directory actually used, with `run`
narrowed to the reported property or case. Framework replay data belongs in
`.seed` and, for fast-check, `.path`; these fields are not a universal replay
format. A command containing `...` or a `<placeholder>` is rejected. See
[Reproducing a campaign](reproducing.md) for framework-specific details.

## Shape

JSON does not support comments; the comments below are explanatory and must be
removed in a real `report.json`.

```jsonc
{
  "schemaVersion": 4,                    // bumped when a field changes meaning
  "generator": { "tool": "pi-pbt", "version": "0.1.17" },
  "run": {
    "date": "2026-09-14",
    "revision": "76296ab70ebeb03798762e396289c5625eb5733a",   // the commit tested — makes the report reproducible
    "repository": "/srv/oh/foundation/arkui/ace_engine",   // absolute
    "target": "frameworks/base/geometry", // module or scope under test
    "tier": "quick"                       // quick | standard | thorough
  },
  "build": {
    "command": "./build.sh --product-name host_product --build-target geometry_pbt_test",
    "workdir": "/srv/oh",
    "result": "success"                   // success | failure | not-applicable
  },
  "properties": [
    {
      "id": "p1",                         // stable within the campaign
      "name": "matrix4 inverse round-trips",
      "formal": "∀ m. invertible(m) ⇒ m * inverse(m) ≈ identity",
      "oracle": "round_trip",
      "sourceFile": "/srv/oh/foundation/arkui/ace_engine/frameworks/base/geometry/matrix4.cpp",
      "sourceLine": 88,                  // one-based definition line in sourceFile
      "functionName": "Matrix4::Invert",
      "testFile": "/srv/oh/foundation/arkui/ace_engine/test/pbt/base/geometry_pbt_test.cpp",
      "testTarget": "geometry_pbt_test",
      "status": "failing",                // passing | failing | retired
      "counterexample": "m = {{1,2},{2,4}}",  // required when failing
      "bugId": "b1"                       // null when the property passed
    }
  ],
  "bugs": [
    {
      "id": "b1",
      "propertyId": "p1",                 // required — the property that found it
      "summary": "Invert returns identity for a singular matrix",
      "severity": "medium",               // low | medium | high | critical — one step down for a documented limitation
      "law": "A singular matrix has no inverse: Invert must report failure, never return a matrix.",
      "impact": "Callers may continue calculations using an invalid inverse.",
      "expected": "false / error for a singular matrix",
      "actual": "identity, silently",
      "counterexample": "m = {{1,2},{2,4}}",
      "rootCause": "matrix4.cpp:91 divides by the determinant without testing it for zero, so 1/0 becomes inf and the cofactor loop normalises to identity.",
      "offendingCode": {                  // re-read from the tree; a paraphrase is refused
        "file": "/srv/oh/foundation/arkui/ace_engine/frameworks/base/geometry/matrix4.cpp",
        "line": 91,
        "snippet": "const double inv = 1.0 / det;"
      },
      "fix": "Test the determinant first: `if (NearZero(det)) { return false; }` before `const double inv = 1.0 / det;`.",
      "regressionTest": "/srv/oh/foundation/arkui/ace_engine/test/pbt/base/geometry_pbt_test.cpp",   // or null
      "rawOutput": "Falsifiable after 12 tests and 3 shrinks: m = {{1,2},{2,4}} — Invert() returned true",
      "reportPath": "bug_reports/invert-singular.md",   // relative to this file; one bug per file
      "contractBasis": "documented-limitation",        // optional — see "What retires a finding"
      "documentedBehavior": "Invert assumes a non-singular matrix — matrix4.h:88",  // required with a documented basis
      "reproduction": {
        "build": "./build.sh --product-name host_product --build-target geometry_pbt_test",
        "run": "RC_PARAMS='reproduce=BQSThRncAQAA' out/host/host_product/tests/geometry_pbt_test --gtest_filter=GeometryPbt.Inverse",
        "workdir": "/srv/oh",
        "seed": null,                     // exact RapidCheck replay is in run
        "path": null                      // fast-check replay path; null elsewhere
      }
    }
  ],
  "totals": { "properties": 1, "passing": 0, "failing": 1, "bugs": 1 }
}
```

`totals` is checked against the arrays. `passing` and `failing` count only
those statuses; a `retired` property remains in `totals.properties` but belongs
to neither count.

## Paths

Two conventions, enforced in both directions so a consumer never has to guess:

- **Repository paths are absolute** — `run.repository`, `build.workdir`,
  `properties[].sourceFile`, `properties[].testFile`, and
  `bugs[].reproduction.workdir`. A consumer should not need an implicit base
  directory to locate the tested checkout or run the commands.
- **Artifact files are relative to `report.json` itself** — `bugs[].reportPath`.
  A managed run may collect or relocate the artifact directory after the
  campaign, so an absolute bug-report path could go stale. Resolve it against
  the directory that contains the `report.json` you are reading.

## Reading it

```bash
# Did this campaign find anything?
jq '.totals.bugs' pbt-out/report.json

# For each bug: which command and replay data belong to it?
jq -r '.bugs[] | "\(.id) \(.summary)\n  run:  \(.reproduction.run)\n  seed: \(.reproduction.seed // "n/a")\n  path: \(.reproduction.path // "n/a")"' \
  pbt-out/report.json

# Every property that is still red
jq -r '.properties[] | select(.status == "failing") | "\(.name) — \(.counterexample)"' pbt-out/report.json
```

## Rules the validator enforces beyond shape

- `run.tier` is one of `quick`, `standard`, `thorough`.
- `run.revision` is the commit the campaign tested, as a **resolved full object ID** (40 hex, or 64 for SHA-256) — not `HEAD`, not a branch name, not an abbreviation. A managed run snapshots an immutable revision while the checkout moves on; applying `changes.patch` to an unknown revision reproduces something else.
- `build.command` may be `null` only when `build.result` is `not-applicable`; `build.workdir` is absolute. The runtime validator is the JSON contract here even though the current TypeScript interface still declares `build.command` as a string. The command itself is placeholder-checked like `reproduction.build` — a clean campaign has no per-bug reproduction to catch a templated `<build command as run>`.
- **At least one property must have been executed** (`passing` or `failing`). An all-empty report — no properties, no bugs, zero totals — would otherwise satisfy every other rule and gate green while the campaign tested nothing. The one exception is `build.result: "failure"`: nothing could run, so zero executed properties is the honest report, and rejecting it would hide the build failure (`hook-run` exit `3`) behind "no usable report" (`2`).
- A `passing` property carries `counterexample: null`. A witness only means something while the property fails; `retired` keeps its witness as the record of why it was dropped.
- A bug's linked property must be `failing` — a bug comes from a failing property.
- `build.result: "failure"` is a valid report — and a **failed run**: `hook-run` exits 3 and a managed run is `failed`. Nothing was tested.
- `reproduction.build` may be `null` only when `build.result` is `not-applicable` (for example, when the test command itself is the only step). Never `"N/A"`.
- `reproduction.seed` is a framework replay value or `null` for a deterministic command. RapidCheck's exact shrunk replay is its printed `reproduce=...` token, which belongs in the recorded run command; a RapidCheck seed alone reruns the generated sequence but does not identify the shrunk case. `reproduction.path` is optional and is used with the seed for fast-check replay. See [Replaying the same case](reproducing.md#replaying-the-same-case).
- `reproduction.build` / `.run` must not contain `...`, `<placeholder>`, `TODO`, `TBD`, `FIXME`, `XXX`, `PLACEHOLDER` or `N/A`.
- Every bug's property names it back, and their counterexamples agree — one witness per link.
- `impact` and `fix` are required for every bug so a customer can understand the consequence and the recommended remediation without interpreting test internals. `fix` must show the change **as code** — a fenced block or an inline code span with the corrected line(s) — and must not be a pointer at another bug (`同 b1`, `see b1`): every page stands alone.
- **Every bug carries its evidence** (schema 4): `law` (the invariant violated, plain language), `rootCause` (`file:line` + why), `offendingCode` (`file` absolute and inside `run.repository`, `line` one-based, `snippet` the defective line(s) verbatim), `rawOutput` (the framework's failure text) and `regressionTest` (absolute path inside `run.repository`, or `null` while none is written). None of them may still carry the skeleton's `<...>` template phrase. The `offendingCode` snippet is **re-read from the tree** when the campaign writes `report.json`, at every close-out, and in the final gates (`hook-run`, the run manager — which judges it before a worktree is torn down): matching is whitespace-insensitive per line and tolerates the snippet starting within two lines of `line`; a paraphrase, an elision or a stale line number is refused with the line the file actually has there. `regressionTest: null` is accepted only at write time; the close-out and the final gates require a readable file, so a page never ships saying "not written yet". An edit of the campaign's `report.json` is judged on the file it produces (the edits are applied to the current text, exactly and uniquely, and the result runs the same checks as a write), so one field can be corrected with `edit`; an edit whose text is not found once is refused rather than left to a fuzzy match.
- `reportPath` is unique across bugs and ends in `.md` — one bug, one file. Two bugs sharing a path render to the same `<slug>.html`, and the second silently replaces the first (observed on ace_engine `scroll_container`: b2 overwrote b1's page).
- `report.json` is refused at write time while the campaign's own artifacts record a finding written out of it: a Design Caveat describing a failure, a passing `PROPERTIES.md` entry with a `Counterexample:`, a Formal admitting the documented value or its sign twin, a writer differential with permuted arguments, or an enumerated domain that omits an enumerator the entry names. The refusal is corrective — file the bug or quote the documentation that declares the input invalid, then write the file again.
- A bug's contract may be documented or inferred. `REPORT.md`'s bug entry says which under `**Contract evidence:**` (`documented <file:line>` + quote, or `inferred (<signature / type / caller / symmetry>)`); a failing, shrunk property is a bug either way — documentation retires it only by declaring the failing input invalid.
- `bugs[].contractBasis` is optional (`inferred` | `documented-exclusion` | `documented-limitation` | `documented-and-violated`). It records **what the comment actually claims**, which is the difference between a finding and a test bug:
  - `documented-exclusion` — the cited text declares the input **invalid / out of domain** (`调用方须保证…`, `never valid`, `precondition`, `must not be persisted`). The generator was wrong; no bug is filed.
  - `documented-limitation` — the cited text **admits a known gap on an input the API accepts** (`暂不支持闰年`, `cannot handle the NONE suffix`, `not implemented yet`). That is the bug being documented: it is filed, `documentedBehavior` quotes the author's text verbatim with its `file:line`, and the severity is graded one step below what the impact alone would earn (critical→high, high→medium, medium→low), stated as such in `REPORT.md`'s `**Severity:**` line. The validator enforces the one contradiction it can decide — a `documented-limitation` bug cannot also be `critical`: "the author disclosed this" and "the top severity" cannot both be the headline. The step itself stays a judgement the report states, because the impact-based grade is not recoverable from the final field.
  - `documented-and-violated` — the cited text **states the behavior is handled** while the code violates it. The comment is the contract; it is the strongest evidence a bug can carry.
  - A comment is evidence, not ground truth. When the code contradicts it, the report says so explicitly instead of treating the comment as authoritative.
- `bugs[].documentedBehavior` is required whenever `contractBasis` is not `inferred`: the author's text verbatim plus its `file:line`. "We read your comment and it documents the limitation rather than excluding the input" has to be quotable, or the report cannot answer the rebuttal it exists to answer.
- `reportPath` stays inside the artifact directory: no absolute paths, no `..`.

## Customer-facing HTML report template

`src/templates/pbt-bug-report.html` and `src/templates/pbt-bug-report.md` are
the repository-visible, versioned sources of the per-defect HTML and fallback
Markdown layouts. They have fixed placeholders for the five required sections;
`src/pbt-html-report.ts` only supplies data from validated `report.json`.
`npm run build` stages both at `dist/templates/`, and the Bun binary embeds and
extracts the same files. Do not hand-write generated `bug_reports/*.html`.
Existing agent-authored bug Markdown is preserved rather than overwritten; when
there is no Markdown, the fixed `.md` template is rendered.

`REPORT.html`, `bug_reports/<slug>.html`, and any Markdown the renderer fills in
(`REPORT.md` / `bug_reports/<slug>.md` when the campaign left none) all follow the
campaign output language:

- `PBT_LANG=zh` renders Chinese prose, uses `<html lang="zh-CN">`, and uses the
  per-bug sections **问题概述 / 问题代码 / 发现与验证方法 / 复现步骤 / 修复建议**.
- `PBT_LANG=en` renders English prose, uses `<html lang="en">`, and uses the
  per-bug sections **Issue Synopsis / Offending Code / Detection and Validation
  Methodology / Reproduction Protocol / Remediation Strategy**.

The overview shows the target, test date, tested revision, campaign tier, the
build command and working directory, aggregate property counts, a table of
confirmed bugs (with impact) and a table of every property tested — id, name,
status, the formal statement, the SUT function with its `sourceFile:sourceLine`,
and the defect it found. Each bug row links to `bug_reports/<slug>.html`; when
no bugs are confirmed, the table says so. A per-bug page always places fields
in its five fixed reader-oriented sections:

- **Issue Synopsis / 问题概述**: severity, identifier, summary, the violated
  contract (`law`), expected and observed behavior, impact, minimal
  counterexample, and report context shown in the overview.
- **Offending Code / 问题代码**: `offendingCode.file:line`, the quoted snippet
  in a code block, and `rootCause`.
- **Detection and Validation Methodology / 发现与验证方法**: property metadata,
  formal property, oracle, SUT function, absolute `sourceFile:sourceLine`, test
  file, test target, and any documentation provenance.
- **Reproduction Protocol / 复现步骤**: absolute work directory, actual build
  and narrowed run commands, seed, fast-check path where applicable, and the
  framework's raw failure output (`rawOutput`) in a code block.
- **Remediation Strategy / 修复建议**: `fix` with its fenced blocks rendered as
  code, and the `regressionTest` path.

The renderer renders only what `report.json` carries; it never infers evidence
from freeform Markdown. When `contractBasis` is `documented-limitation`,
`documented-and-violated`, or `documented-exclusion`, the page also carries a
**Documentation** block (Chinese: **文档依据**) with the author's verbatim text —
that block is what tells a developer reading the page that pi-pbt read their
comment and is disagreeing with it on purpose. `report.json` keys remain English
and stable as the machine-readable contract. HTML output escapes all report
values before placing them in the page. The report generator also rejects a
`reportPath` that escapes the artifact directory.

### Evidence the page embeds, and what it still does not

Schema 4 closed the gap this section used to describe: the page embeds the
offending source excerpt, the root cause, the framework's failure output, the
fix as code and the regression test path. What it still does not embed is the
property-test code itself and the shrinking trace — the test file and target
are named, the reproduction command re-runs the case, and the shrunk witness
is the counterexample. Adding those would be a further contract extension; do
not infer or fabricate them while rendering.

## Managed runs

Under the MCP run manager's default worktree mode, the campaign runs in
`<runDir>/worktree`, captures its repository changes in `changes.patch`, and
then deletes that worktree. Before deletion, pi-pbt rewrites worktree paths in
`report.json`, `REPORT.md`, `PROPERTIES.md`, `COVERAGE.md`, and bug reports to
the original repository path. In that repository, check out `run.revision` and
then apply `changes.patch` before using paths to generated tests. See the
[MCP run-artifact description](installation.md#delegation-from-another-coding-agent-mcp).

An in-place managed run (selected when `pbt_start` receives `build_cmd`) has no
relocation step: its absolute paths already refer to the repository where the
campaign ran.

## Compatibility

`schemaVersion` is the compatibility boundary. A consumer must check it before
using fields and reject a version it does not know. Adding an optional field
does not require a bump; changing a field's meaning or removing one does.
