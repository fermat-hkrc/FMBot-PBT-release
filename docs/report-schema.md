# Campaign result contract (`pbt-out/report.json`)

A completed campaign writes customer-facing and technical result artifacts:

- `REPORT.html` is the customer-facing overview. It links to every individual bug page.
- `bug_reports/<slug>.html` is one customer-facing page per confirmed bug.
- `REPORT.md` is the technical Markdown campaign narrative.
- `bug_reports/<slug>.md` is the Markdown version of the individual bug report.
- `report.json` is the structured source for CI, dashboards, and the generated customer reports.
- `api-perf.json` is fmbot-pbt's own measurement of the model provider: one record per
  main-session call (time to the response headers, time to first token, duration, tokens,
  final HTTP status), tool time per
  tool, and each delegated scan or close-out review child (duration, usage, errors, rate
  limits). `REPORT.html` shows its summary, the CLI teardown (or the TUI on exit) appends
  it to `REPORT.md` as a marked `## API performance` section, and the TUI footer shows
  calls, time to first token, output rate and errors while the session runs. It is not
  part of the `report.json` contract and never changes a verdict. pi-ai retries a 429 or
  5xx inside the request without an event for the failed attempts, so their wait shows
  as a long time to the response headers rather than as an error.

A campaign never writes these files. It records each fact with a record tool
(`record_run`, `record_property`, `record_bug`, `record_caveat`,
`record_remark`, `record_summary`; see [Record tools](#record-tools) below), and fmbot-pbt writes
`report.json` and renders `REPORT.md`, `REPORT.html` and every
`bug_reports/<slug>.md` / `.html` from it after each record. A write or edit of
any of them is refused.

`hook-run` requires both files in the same recognized artifact root. A missing
or invalid `report.json` makes the check incomplete (exit `2`), even if
`REPORT.md` exists. See the
[`hook-run` exit-code table](subcommands.md#hook-run).

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

The record tools write this file; the comments below are explanatory and do not
appear in a real `report.json`.

```jsonc
{
  "schemaVersion": 5,                    // bumped when a field changes meaning
  "generator": { "tool": "fmbot-pbt", "version": "0.1.17" },
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
      "contractEvidence": "documented matrix4.h:88 — `Invert assumes a non-singular matrix`",
      "documentationConflict": "(none)",
      "callerEvidence": "/srv/oh/foundation/arkui/ace_engine/frameworks/core/transform.cpp:40 `matrix.Invert();`",
      "reproduction": {
        "build": "./build.sh --product-name host_product --build-target geometry_pbt_test",
        "run": "RC_PARAMS='reproduce=BQSThRncAQAA' out/host/host_product/tests/geometry_pbt_test --gtest_filter=GeometryPbt.Inverse",
        "workdir": "/srv/oh",
        "seed": null,                     // exact RapidCheck replay is in run
        "path": null                      // fast-check replay path; null elsewhere
      }
    }
  ],
  "caveats": [],                          // design notes; see Schema 5
  "remarks": [],                          // why the result is weaker than it should be
  "summary": {
    "conclusion": "One medium bug: Invert reports success for a singular matrix.",
    "changedCode": "frameworks/base/geometry/matrix4.cpp",
    "coverageEvidence": { "level": "file-level", "detail": "coverage_gaps: matrix4.cpp covered by geometry_pbt_test" },
    "followUp": "Fix b1 and keep its regression test."
  },
  "modules": [{ "name": "geometry", "tests": 1, "bugs": 1, "oracles": ["round_trip"] }],
  "testFiles": [{ "file": "/srv/oh/foundation/arkui/ace_engine/test/pbt/base/geometry_pbt_test.cpp", "tests": 1 }],
  "reproduction": {
    "build": "./build.sh --product-name host_product --build-target geometry_pbt_test",
    "run": "out/host/host_product/tests/geometry_pbt_test",
    "workdir": "/srv/oh"
  },
  "totals": { "properties": 1, "passing": 0, "failing": 1, "bugs": 1 },
  "review": { "status": "reviewed", "targets": { "bug:b1": { "sha": "…", "share": "bug:b1", "summary": "…" } }, "findings": [], "lastFinding": 0 }   // see "Independent review"
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
- **Every bug carries its evidence**: `law` (the invariant violated, plain language), `rootCause` (`file:line` + why), `offendingCode` (`file` absolute and inside `run.repository`, `line` one-based, `snippet` the defective line(s) verbatim), `rawOutput` (the framework's failure text) and `regressionTest` (absolute path inside `run.repository`, or `null` while none is written). None of them may still carry the skeleton's `<...>` template phrase. The `offendingCode` snippet is **re-read from the tree** at each record, at every close-out, and in the final gates (`hook-run`, the run manager — which judges it before a worktree is torn down): matching is whitespace-insensitive per line and tolerates the snippet starting within two lines of `line`; a paraphrase, an elision or a stale line number is refused with the line the file actually has there. `regressionTest: null` is accepted only at write time; the close-out and the final gates require a readable file, so a page never ships saying "not written yet".
- `reportPath` is unique across bugs and ends in `.md` — one bug, one file. Two bugs sharing a path render to the same `<slug>.html`, and the second silently replaces the first (observed on ace_engine `scroll_container`: b2 overwrote b1's page).
- The conclusion (`record_summary`) is refused while the campaign's own records state a finding written out of it: a Design Caveat describing a failure, a passing `PROPERTIES.md` entry with a `Counterexample:`, a Formal admitting the documented value or its sign twin, a writer differential with permuted arguments, or an enumerated domain that omits an enumerator the entry names. The refusal is corrective — file the bug or quote the documentation that declares the input invalid, then call `record_summary` again.
- A bug's contract may be documented or inferred. The bug's `contractEvidence` says which (`documented <file:line>` + quote, or `inferred (<signature / type / caller / symmetry>)`); a failing, shrunk property is a bug either way — documentation retires it only by declaring the failing input invalid.
- `bugs[].contractBasis` is optional (`inferred` | `documented-exclusion` | `documented-limitation` | `documented-and-violated`). It records **what the comment actually claims**, which is the difference between a finding and a test bug:
  - `documented-exclusion` — the cited text declares the input **invalid / out of domain** (`调用方须保证…`, `never valid`, `precondition`, `must not be persisted`). The generator was wrong; no bug is filed.
  - `documented-limitation` — the cited text **admits a known gap on an input the API accepts** (`暂不支持闰年`, `cannot handle the NONE suffix`, `not implemented yet`). That is the bug being documented: it is filed, `documentedBehavior` quotes the author's text verbatim with its `file:line`, and the severity is graded one step below what the impact alone would earn (critical→high, high→medium, medium→low), stated as such in `REPORT.md`'s `**Severity:**` line. The validator enforces the one contradiction it can decide — a `documented-limitation` bug cannot also be `critical`: "the author disclosed this" and "the top severity" cannot both be the headline. The step itself stays a judgement the report states, because the impact-based grade is not recoverable from the final field.
  - `documented-and-violated` — the cited text **states the behavior is handled** while the code violates it. The comment is the contract; it is the strongest evidence a bug can carry.
  - A comment is evidence, not ground truth. When the code contradicts it, the report says so explicitly instead of treating the comment as authoritative.
- `bugs[].documentedBehavior` is required whenever `contractBasis` is not `inferred`: the author's text verbatim plus its `file:line`. "We read your comment and it documents the limitation rather than excluding the input" has to be quotable, or the report cannot answer the rebuttal it exists to answer.
- `reportPath` stays inside the artifact directory: no absolute paths, no `..`.

## Rendered pages

`src/pbt-report-render.ts` (REPORT.md) and `src/pbt-html-report.ts` (REPORT.html
and the bug pages) are the only report layouts in the repository: there is no
template file and no sample report to keep in step. They render only what a
validated `report.json` carries and overwrite every page on each record; do not
hand-write or edit them. Every page follows the campaign output language:

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
that block is what tells a developer reading the page that fmbot-pbt read their
comment and is disagreeing with it on purpose. `report.json` keys remain English
and stable as the machine-readable contract. HTML output escapes all report
values before placing them in the page. The report generator also rejects a
`reportPath` that escapes the artifact directory.

### Evidence the page embeds, and what it still does not

The page embeds the offending source excerpt, the root cause, the framework's failure output, the
fix as code and the regression test path. What it still does not embed is the
property-test code itself and the shrinking trace — the test file and target
are named, the reproduction command re-runs the case, and the shrunk witness
is the counterexample. Adding those would be a further contract extension; do
not infer or fabricate them while rendering.

## Managed runs

Under the MCP run manager's default worktree mode, the campaign runs in
`<runDir>/worktree`, captures its repository changes in `changes.patch`, and
then deletes that worktree. Before deletion, fmbot-pbt rewrites worktree paths in
`report.json`, `REPORT.md`, `PROPERTIES.md`, `COVERAGE.md`, and bug reports to
the original repository path. In that repository, check out `run.revision` and
then apply `changes.patch` before using paths to generated tests. See the
[MCP run-artifact description](subcommands.md#mcp).

An in-place managed run (selected when `pbt_start` receives `build_cmd`) has no
relocation step: its absolute paths already refer to the repository where the
campaign ran.

## Independent review

`review` is the close-out review itself. fmbot-pbt alone writes it — the
`closeout_review` tool from the reviewer children's results, `respond_review`
with the campaign's answer to a finding — and renders `pbt-out/REVIEW.md` from
it; nothing reads that page back.

| Field | Content |
|---|---|
| `status` | `reviewed`, or `not-reviewed` with `reason` and `detail` |
| `reason` | `disabled` (`PBT_CLOSEOUT_REVIEW=off`), `reviewer-unavailable` (the bundled pi-subagents did not accept the reviewer in that session) or `reviewer-failed` (the reviewer failed twice and the campaign went on without it) |
| `targets` | each reviewed target by key — `bug:<id>`, `caveat:<id>`, `property:<name>`, `retired:<name>` — with `sha` (the fingerprint of the record, ledger entry and cited files its review read), `share`, `summary`, optional `runId`, `model`, `caveat` (the verdict on a caveat's citation), `raised`, `contested` |
| `findings[]` | `id` (`R<n>`, never reused), `target` (a key of `targets`), `kind` (`unsupported-bug` / `weakened-oracle` / `unsupported-caveat`), `subject` (what the reviewer named), `problem`, `evidence`, `requestedAction`, `response` once the campaign answered |
| `lastFinding` | the highest finding number assigned |
| `reviewedNow` | the targets the latest call reviewed; the rest were carried over unchanged |

A record or cited file that changes after its review makes that target due
again; the conclusion (`record_summary`) is refused until the review covers the
records as they are and every finding has a `response`. A `reviewer-failed`
review keeps the findings of the shares that finished, and they need a response
too.

The field is optional in the schema: read a missing `review` as "unknown",
never as "reviewed". The rendered report states it.

## Compatibility

`schemaVersion` is the compatibility boundary. A consumer must check it before
using fields and reject a version it does not know. Adding an optional field
does not require a bump; changing a field's meaning or removing one does.

### Schema 5

Schema 5, the only schema, makes `report.json` the only source of the result
(`docs/report-single-source.md`). Besides the run, build, properties and bugs
above, it carries:

| Field | Content |
|---|---|
| `summary` | `conclusion`, `changedCode`, `coverageEvidence` (`level`: `line-level` / `file-level` / `none`, `detail`), `followUp` |
| `properties[].history[]` | written by fmbot-pbt, oldest first: `{ kind: "status", status, at, counterexample? }` for each new status or new witness, `{ kind: "reverified", at, command, output }` for the same counterexample re-run on the real implementation. A property that failed and is now passing or retired needs a `reverified` entry after its last failure, or a failing `discrimination` |
| `properties[].discrimination` | `baseline`, `command`, `result` (`fails` / `passes`), `output`: the property run against the pre-change baseline |
| `modules[]` | `name`, `tests`, `bugs`, `oracles[]` (the Modules Tested table) |
| `caveats[]` | `id`, `claim`, `docEvidence` (`file` absolute, `line`, `quote` verbatim), `classification` (`input-domain-restriction` / `does-not-constrain`), optional `note` |
| `remarks[]` | `category` (`environment` / `build` / `budget` / `coverage` / `tooling` / `other`), `text`: why the result is weaker than it should be; not a caveat, not reviewed |
| `testFiles[]` | `file` (absolute), `tests` |
| `reproduction` | the whole suite: `build` (or null), `run`, `workdir` (absolute) |
| `bugs[].contractEvidence`, `bugs[].documentationConflict` | required; `contractBasis` becomes required |
| `bugs[].callerEvidence` or `bugs[].appEntry` | the in-tree call, or the app-facing entry (`capi/`, `napi/`, `OH_*`), that passes the witness, `file:line` + quote; one is required unless the function is itself app-facing |
| `bugs[].sharedContract` | for a sibling differential: the production line stating both siblings implement the same contract |
| `bugs[].formal` | the property's Formal statement, filled from the ledger; must equal it |

### Record tools

`report.json` is written through record tools, one validated record per call;
a refused call changes nothing and names the field to fix:

| Tool | Writes | Filled by fmbot-pbt |
|---|---|---|
| `record_run` | `run`, `build` | `generator`, `schemaVersion` |
| `record_property` | one property's result, a re-verification (`reverified`), a baseline run (`discrimination`) | name, `formal`, oracle, function and test file from its `PROPERTIES.md` entry; `history` |
| `record_bug` | one bug | `formal` and the counterexample from its property; that property's `bugId`; a `reportPath` from the id and summary |
| `record_caveat` | one design note; its quote is checked at the cited line | — |
| `record_remark` | one remark | — |
| `record_summary` | `summary`, `modules`, `testFiles`, `reproduction`; REPORT.md is rendered from then on | — |

`totals` and `review` are always computed by fmbot-pbt. In a Chinese campaign
every prose field (a bug's summary, law, impact, expected, actual, root cause
and fix; a note's claim; a remark; the summary) must be Chinese and pass the
typography and jieba dictionary checks, or the record is refused. Fields that
quote documentation, evidence, framework output or code are exempt.

fmbot-pbt renders `REPORT.md` and every `bug_reports/<slug>.md` from it and
overwrites them. `REPORT.md` appears only once `summary` is recorded, because
its presence marks a campaign as closing. Headings and labels are plain words
in the campaign language (`结论` / `Conclusion`, `设计说明` / `Design notes`);
nothing parses them. The marked coverage, code coverage and API
performance sections are kept across re-renders. Evidence fields quote source
verbatim, so a quoted `TODO:` / `FIXME:` is accepted there; only a skeleton
`<...>` phrase is refused. The caller-reachability and sibling-differential
checks run on the bug records. A design note's `docEvidence` must not be the
campaign's own output (a `pbt-out/` file or a log).

### Kotlin / Gradle execution evidence

When Gradle XML exists for an authored Kotlin suite, each executed property's
`testCase` identifies `fully.qualified.Class#method` from `gradle_test_results`.
The report gate checks that case's actual status and rejects omitted failures
in that suite. The XML must be newer than the suite's last edit. Shared KMP
source does not prove all targets ran; `testTarget` records the actual task.

### Multiple witnesses of one defect

`bug.propertyId` selects the primary failing reproducer and retains its exact
counterexample/backlink checks. Additional failing properties may reference the
same `bugId` with different witnesses. Bug counts and failing-property counts
need not be equal. A live Gradle XML failure cannot become `retired` merely to
deduplicate a defect; correct or disable an invalid test and rerun first.
