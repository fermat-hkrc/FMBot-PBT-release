# ArkTS signing and device verdicts

[中文](arkts-device.zh.md) · [Command reference](subcommands.md) · [Installation](installation.md)

## Source campaigns

```bash
fmbot-pbt build-run --build-cmd "<your build command>" --target <source file> --func <function> \
  --sign-profile system --result-source auto
```

`build-run` and `/skill:pbt-build-run` share these options, validation and execution:

| Flag | Meaning |
|---|---|
| `--result-source auto\|stdout\|hilog` | default `auto`; complete stdout or correlated device results |
| `--sign-profile normal\|system` | SDK debug signing policy; `normal` pairs `normal` with `hos_normal_app`, `system` pairs `system_basic` with `hos_system_app`; default `normal` |
| `--device-target <hdc>` | select a device |
| `--test-bundle`, `--test-module`, `--test-class` | explicit bundle/module/`Suite#case` selection |
| `--ability-grace-ms`, `--test-ability` | launch recovery grace (20000; 0 disables) and ability name (`TestAbility`) |
| `--idle-timeout-ms <ms>` | stop after this long without observed test progress (300000; 0 disables the idle cutoff) |
| `--hap <entry.hap>`, `--test-hap <test.hap>` | existing-artifact mode below; no new property generation |

The engine passes settings to the helpers through `PBT_ARKTS_DEVICE_OPTIONS` (JSON).
Prefer CLI flags rather than setting it manually. `HDC` and `JAVA` may select provisioned executables.
Source campaigns identify the source owner from `build-profile.json5`. If the supplied `ohosTest` build selects a consuming HAP, register the suite in that HAP and import the original library export; keep the build's full module/target selector. After rebuilding the latest source and tests, pass the fresh, device-authorized project-signed pair unchanged to `fmbot-pbt arkts-test --class <PbtSuite> --hap <signed-app.hap> --test-hap <signed-ohosTest.hap>` (or select `Suite#case`), not the whole unit suite. Omit `--hap` only for a standalone test HAP or a verified matching app already installed. On an installed signer mismatch, stop; do not uninstall or change signers to force the update. Supply archive flags to this verification command, not to the initial `build-run`, where they select existing-artifact mode. The runner installs and fingerprints the specified test HAP; a test HAP older than its current source/test inputs is inconclusive. Keep it and the source tree for offline finalization. Import the real scoped declaration through its verified package export or source path; never copy it.
CLI and in-session preparations hold checkout/output ownership under `~/tmp/fmbot-pbt-campaign-locks/`. Overlapping campaigns fail with exit 2 and owner details before build or cleanup; independent checkouts may run. Only proven-exited host owners are automatically reclaimed; live, inaccessible or foreign-host owners are never stolen.

For a generated ohosTest target whose build needs normalized OHM URLs, supply your normal working product build, not a pinned test command. The product and companion app build unchanged. Only the generated test invocation uses a temporary project with the setting and PandaCheck dependency. `arkts-preflight.json` records the declared and observed settings (`testOhmUrls`) and the named device case; this proves readiness, not property coverage. Edit tests in the checkout and use the verified test build after edits: it stages current inputs and prints the new HAP directory. If signing is needed, use `fmbot-pbt arkts-sign` on that HAP and keep the result under the campaign output. Rebuilding and device verification both check the staged inputs by content; a newer timestamp cannot make old tests current. A warm direct app build keeps its original options when forcing packaging. For a script or compound command, make the supplied command produce a fresh companion HAP. An incompatible SDK, loader error, or missing named completion stops the scan. Builds that already enable the setting keep their existing route.

With `--isolate --isolate-test-sdk`, `testLink.compiler` in `arkts-preflight.json` points to the ohosTest project and loader configurations saved **before** the companion app build. Its `linkedAbcSha256` must match `ets/modules.abc` inside the signed test HAP before either HAP is installed. A mismatch stops preflight even if an older package could pass the same named case. A local signer failure is reported as `environment/signing`; a device install rejection is `environment/device`. Neither starts a campaign.

## Filing failures and close-out

Caller citations accept Windows paths, spaces and qualified methods. UI callbacks (`onPaste`, `onClick`, TextInput `onChange`/`onSubmit`) and `@Entry`/`@Component` build bodies are app-facing producers when the cited code actually calls the target. A transform must exclude the actual witness to justify rejection; stripping commas does not exclude `'1E'`.
Record the failing property first, then use `record_bug` and its bundled worked example. The limits are three refusals for one finding and twelve unresolved refusals overall. Refusals for only Chinese punctuation or spacing in `record_bug` have their own limits of three and twelve; a mixed refusal uses the regular budget. Both share the five-minute no-progress deadline. At a limit, the campaign stops inconclusively (exit 2) unless it can park the finding as described below. The timer starts at the first refusal and restarts only when an accepted correction clears an outstanding key while others remain. Unrelated or repeated accepted writes do not extend it. `rejected-records.jsonl` keeps requests, reasons and prior evidence; `recording-stop.json` and `interruption.json` explain a hard stop. Do not withdraw or retire a real failure merely to satisfy a gate.
In a headless Pi `build-run` targeting the whole repository without a change filter, the third refusal can park correction if the refusals point to the same saved finding. This includes a recorded failing witness, or a summary blocked solely by one passing property's saved failure-to-pass notes. The property, notes and evidence stay unchanged while unrelated tests continue. The parked checkpoint blocks verified delivery, including via `finalize`; the run exits 2 with its partial results. To verify the affected work, start a fresh campaign in a separate output directory.
Run-correlated device failures are retained automatically in `report.json.deviceFailures` and `findings-unrecorded.json`, even when property/bug validation refuses them. An observation received before a timeout still needs triage, even after a verified rerun supersedes the inconclusive attempt. These observations are not automatically confirmed SUT bugs. Link validated properties with `deviceObservationIds`; correcting a test requires preserved failure history and re-verification, not silent retirement. Relative property paths resolve against the recorded repository.
A `reference` oracle needs independently cited contract evidence (specification, comment or caller assertion), not the implementation's own key table/regex. Record this in the design's evidence before approval.
`campaign-state.json` is saved before tools execute. If a parent dies without `interruption.json`, `fmbot-pbt resume --out <out>` uses that exact checkpoint/session, preserving scope, model and freshness. An empty/unmatched class writes an inconclusive receipt/interruption and exits 2, never a successful zero-test result.
If the agent aborts after recording complete evidence, end the active campaign and use `fmbot-pbt finalize --out <out>` for receipt-verified offline completion. It preserves the interruption history, marks the matching checkpoint completed, and makes no model or device request. Old abandoned missing receipts can be recorded as aborted under exclusive ownership, never as passes; contradictory receipts and untriaged failures remain blocking; see [resume and offline completion](subcommands.md#resume-and-offline-completion).
Headless runs finalize after SDK settlement, with no pending work, and still run the delivery gates before exiting. A `REPORT.md` file alone never proves completion; interactive/RPC sessions remain open.

## Local signing

```bash
fmbot-pbt arkts-sign app-unsigned.hap app-signed.hap --bundle com.example.app --udid DEVICE_UDID \
  --sign-profile system
```

Default is `normal`/`hos_normal_app`; `--sign-profile system` selects `system_basic`/`hos_system_app`.
The former `--sign-apl` and `--sign-feature` flags now exit `2` before building or signing. Use `--sign-profile normal` or `--sign-profile system` for the same policy.
The local SDK debug profile retains the UDID and merges requested permissions, existing ACLs and repeatable `--acl` values.
The input HAP and SUT signing scripts remain unchanged; no CPKI request or fallback to a different signing policy occurs.

The device must trust the SDK debug certificate. This does not grant retail-device authorization or supply missing system-SDK APIs.
An error `202` can still require a different device, certificate, permission or SDK.

SDK precedence: `--sdk`, `local.properties` `sdk.dir`, `DEVECO_SDK_HOME`, then `DEVECO_HOME`/standard DevEco locations.
Windows discovery includes `Program Files/Huawei/DevEco Studio/sdk`. The resolved path is printed; use `--project <hap-root>` outside the project root.
Java comes from `JAVA`, the detected DevEco `jbr`, or PATH. HAP metadata needs `unzip` or ZIP-capable `tar` (included in Windows).

## Device results

```bash
fmbot-pbt arkts-test --target DEVICE --bundle com.example.app --module entry_test \
  --class DisplayPbt --test-hap current-signed-ohosTest.hap --result-source auto --out pbt-out/device
```

The binary uses its embedded runtime, without a model call. Shell adapters `sign-hap.sh` and `aa-test.sh` call the same implementations using Node.
`auto` accepts a complete stdout result (including a validated native completion report) or a fresh hilog summary tied to this run. `stdout` disables hilog capture; `hilog` still requires a device-log summary.
The runner filters hilog on the device so unrelated log history does not hold up test records. If `aa test` closes with a usable summary, it gives correlated hilog a short drain after both events, within the original hard deadline. At the cutoff it checks all records already received in time, including conflicting summaries, without waiting for more. It never clears device logs or accepts partial/status-only records as a terminal verdict.

New test runners emit a UUID marker before Hypium starts (the bundled command card describes it; generated isolates include it).
A different run's marker on the same PID ends attribution without discarding records from before that boundary.
Older pre-signed runners can instead be attributed through a newly observed, verified bundle PID.
Missing or ambiguous attribution remains inconclusive, even when a terminal summary exists elsewhere in the log.

| Exit | Meaning |
|---|---|
| `0` | positive, consistent terminal counts with no failure/error |
| `1` | complete terminal counts with a real failure/error |
| `2` | missing, conflicting, uncorrelated or invalid results, transport failure, or a stall |

A Hypium failure in this run's stdout or correlated hilog contradicts an all-pass summary (exit `2`), even if only the custom assertion message (`stream=message: …, Error in …`) survives. Keep an unnamed failure for triage; do not attribute it to the selected or passing case. If stdout already proves a failure, an identical hilog summary alone does not require PID confirmation (exit `1`).

`--class Suite#case` selects cases; nested suites use their full Hypium path, such as `Outer.Inner#case`. `--timeout-ms` is the per-case limit (default 120000); `--run-timeout-ms` bounds the test wait (1800000). Cases with optional Hypium timeout/tag arguments and cases defined directly in `List.test.ets` require the same execution receipts. A registration-only index is not itself a property's test file.
During a campaign (`PBT_OUT_DIR`), all adapters put `stdout.log`, fresh relevant `hilog.log`, `diagnostic.log` and `result.json` in `<out>/device-run/<run-id>/`, regardless of a helper's `--out`. Receipts include completed Hypium suite/case statuses, SHA-256 source/test/registration/HAP fingerprints, ISO `startedAt`/`endedAt` and observed failure details. Every case in a reported test file needs matching fresh evidence; separate case selections accumulate only while those inputs remain unchanged. The default suite must be called on a reachable path from `List.test.ets`'s synchronous exported `testsuite()`. A dead registration, class selection alone or unrelated smoke summary cannot clear pending PBT execution; missing case/artifact evidence leaves delivery inconclusive (exit 2). `device-run/runs.jsonl` registers each invocation before launch. Standalone calls outside a campaign honor `--out`, otherwise use a new `~/tmp/pbt-aa-*` directory.

Use a string literal or same-file string `const` for each Hypium name. Register each `Suite#case` once across the files called by `List.test.ets`. A smoke receipt does not verify an unresolved name, a registration inside a callback such as `forEach`, or a case that may register more than once in a loop.

### Named results without hilog

Generated isolates install the test-only reporter automatically. For an existing Hypium 1.0.24/1.0.25 source harness, copy `references/arkts/runner/FmbotCaseReporter.js` from the bundled `pbt-build-run` skill beside `src/ohosTest/ets/test/List.test.ets`. In that file, import `{ installFmbotCaseReporter }` from `./FmbotCaseReporter` and call it inside the exported `testsuite()` before registering the suites. Rebuild and sign the test HAP once. Do not patch production code or rewrite a supplied pre-signed archive.

The reporter records each named pass, failure or error in Hypium's terminal callback. Small reports fit in the native `aa test` finish message and need no hilog. For larger reports, the finish message carries the counts; preceding `aa` output pages carry whole case records, with a tagged hilog mirror when capture is enabled. The host checks the run UUID and a fresh invocation ticket, counts identical mirrors once, and saves the received outcomes in `case-report.json` beside `result.json`.

Missing pages or names leave a `namedCaseGap`, even when the aggregate verdict is valid. Spaces and horizontal tabs are part of the exact suite and case name. Other control characters or line breaks prevent reliable comparison with text logs; the runner retains the callbacks but exits `2`. Conflicting counts (including a zero-test or all-ignored summary), contradictory named results, or a malformed finish also exit `2`. A legacy HAP without the reporter still uses stdout/hilog and cannot recover a name that neither retained.

### Legacy named-case gaps

A passing class summary does not verify every case by name. If a valid terminal summary has more non-ignored outcomes than `result.json.cases` accounts for, the runner warns and records `namedCaseGap` with `unnamedPasses` and `unnamedFailuresOrErrors`. For example, seven passes with five named completions gives `unnamedPasses: 2`; the two cases stay unverified. The exit code still describes the suite summary, not the completeness of the named evidence. Delivery groups missing cases by suite, test artifact and reason, with the exact case IDs and affected property IDs in the details. Copy the full `arkts-test --class 'Suite#case'` command line for that case only. Keep `PBT_OUT_DIR` to save the receipt to the campaign and, when shown, `PBT_SCOPE_FILE` to fingerprint the scoped source even before property results are recorded. On Windows the command uses PowerShell. If the HAP or target is unknown or ambiguous, confirm the intended values and replace any placeholders. Confirm the matching application HAP is installed, or pass its verified archive with `--hap`. Recheck delivery before choosing another alternative. Wait for an active run to end, keep its original logs and receipt, and triage a targeted failure rather than rerunning it to get a pass. No rerun is launched by this diagnostic.

To permit automatic repair, pass `--recover-missing-cases` when starting a headless Pi ArkTS source `build-run`. A [terminal launch](subcommands.md#build-run) needs no stdin redirection, but the build command must not prompt for terminal input. The campaign is prompted to pass both signed archives (`--hap` and `--test-hap`) to its original `arkts-test` invocation, not to `build-run`, and to keep them unchanged through close-out. A passing class receipt must identify those archives and the device. Without both archives the diagnostic remains manual.

After the model settles, the opted-in worker first checks missing `result.json` files under the same campaign lease. As with `finalize`, it can record an unambiguous attempt as **aborted**, not passed, only if it is at least 30 minutes old, its host owner has exited or was not recorded, no target lock exists, and retained files show no test activity, results or failures. It leaves the original index and logs intact, takes no device attempt, and does not fill any named receipt gap. Recent, locked, conflicting or otherwise uncertain attempts still block delivery.

The worker can then run eligible missing selectors sequentially. `--recover-max-cases N` caps eligible attempts (positive integer, default `3`, requires `--recover-missing-cases`). Missing or ambiguous routes, or routes whose snapshot the runner cannot reproduce, do not use the budget or block verified routes. The time limits remain two minutes per test wait and five minutes total, including setup. A separate, bounded device-state check can run after the deadline to release the lock if the app is no longer running.

The worker retains the campaign lease, acquires the normal device lock for each case without reclaiming it, and rechecks inputs, named execution and all delivery gates. Without a recovery recipe, it only uses current partial class receipts and never rebuilds or signs. Another delivery blocker prevents recovery; a changed route, input drift, a failed or inconclusive attempt, or lack of verified progress stops it. The worker never starts another model round or retries a failed test. Inconclusive tests remain manual unless the stall option below applies. Any remaining gaps keep delivery incomplete (exit `2`), with receipts and manual commands preserved. Recovery is off by default and is not available with `--no-auto-resume`, a TUI, in-session skills, ACP, isolates, or pre-signed execution.

Add `--recover-stalled-run` **alongside** `--recover-missing-cases` to authorize one sequential superseding run at close-out. The latest registered run must be an unambiguous `stalled` receipt from this campaign. Its local runner must have exited; its receipt and retained stdout/hilog must be readable and show no failure. Under a newly acquired device lock (never a reclaimed lock), the worker checks that no `aa` client or target bundle is active, installs the same signed app and test HAPs, and rechecks the target and input snapshot before launching the original `Suite` or `Suite#case`. This uses one attempt and the same five-minute deadline as other case recovery. A second stall, unknown device state or mismatched evidence stops recovery. Both receipts and logs remain; only a verified named result on the new `supersedesRun` receipt can resolve the original gap. Standalone `arkts-test` does not retry.

After three consecutive launches on one target stall without correlated test activity, automatic recovery pauses for that target. Failed installs do not break the streak. An unpinned automatic resume waits rather than guessing a device; a resumed worker also checks the target selected by `arkts-test` before installation or launch. Check the listed run IDs and `hdc list targets`; restart an emulator only after confirming it is idle. Then explicitly run `fmbot-pbt resume --out <out>`. Nothing reboots automatically, and the original receipts remain inconclusive.

To resolve a stall, pass and fail results both require the requested suite and case fields and unchanged inputs at completion. If a later recovery build adds an input to the superseding run’s module, close-out stops before using the rebuilt pair.

To clear a `--class` stall, the replacement receipt must name a completion for each requested suite or case and account for every test in its summary. An out-of-scope completion blocks recovery even if the rerun lost its summary. After the report is rendered, an unresolved scope mismatch stops headless close-out and `resume` exits 2. Rerun the original selection explicitly and keep both receipts; `finalize` does not waive the stall.

Once the campaign report is rendered, `resume` lists missing named cases and unresolved stalls instead of starting another model or device run, even if no replacement was attempted. Supply matching receipts and run `fmbot-pbt finalize --out <out>` offline. Each receipt clears only the obligations it verifies; remaining gaps still mean exit 2.

### Recovery after test edits

Pass `--case-recovery-recipe recovery.json` **in addition to** `--recover-missing-cases` to authorize one build/sign attempt during close-out. Use an output filesystem that supports hard links, so the worker can publish the complete build proof without overwriting existing evidence. For example, a project-specific recipe might be:

```json
{
  "command": "bash ./rebuild-and-sign.sh",
  "inputs": ["rebuild-and-sign.sh", "AppScope", "shared/src"]
}
```

The supervisor reads this file at launch; it does not take commands from the model, receipts or saved campaign state. The command runs in `--build-dir` (inside the ArkTS project), with Bash on POSIX or `cmd.exe` on Windows and no stdin. Use a trusted, host-only recipe that rebuilds the registered harness and its app from current inputs, waits for all build and signing work, and writes the fresh signed app and test archives to the new paths in `PBT_CASE_RECOVERY_APP_HAP` and `PBT_CASE_RECOVERY_TEST_HAP`. Use the matching signing policy. Do not re-sign cached test bytecode, change the original archives, install on the device, or launch tests in the recipe.

`inputs` is the recipe's additional build input list: files or directories relative to `--build-dir`, or absolute paths. Include all other dependencies, resources, scripts, and build/sign configuration the recipe uses; exclude outputs and generated caches. The worker also snapshots the harness's `src`, reported source files, and known project/module build files. Unreadable or aliased inputs stop recovery. Both output identities must match the original modules, the test bytecode must change, and the input snapshot must stay unchanged through each new receipt. Successful device installation is still required.

The worker first repairs eligible current routes. If case budget remains, it can run the authorized recipe in the same close-out for gaps without a current route. The stale gaps must identify one harness, with unambiguous passing class runs from the same campaign, the original app/test bytes, and an edit to a registered suite or its registration. Both paths share the attempt cap and the five-minute budget, including the build. The worker rechecks all delivery obligations after the build and requires a fresh exact receipt for each missing case, including previously passing cases on the stale pair. It also recomputes the snapshots for just-repaired current routes before and after the build and after each passing rebuilt case, so newly added inputs can stop close-out even if every recorded file still matches. Any further drift stops the batch without a second build. The build log, input snapshot and rebuilt pair stay under `<out>/case-recovery-build-*/` for offline verification; keep them with the receipts. Once a passing receipt verifies the rebuilt pair, live diagnostics and `finalize` give the exact pair and device for remaining planned cases, even in suites the worker did not reach. These hints are not execution evidence. Use both original archive paths, not copies. The runner checks the same build inputs again, including configuration files that were absent at build time. Earlier failures and receipts are retained. Plain `resume` and offline `finalize` do not execute recipes.

### Troubleshooting device results

If installation fails, read `install.log` and the receipt's `reason`. The runner keeps the installer's message separate from an archive fingerprint mismatch. `9568344` alone does not identify the cause. When the installer or a fresh installer log for that bundle also identifies a privilege-extension or signature capability-list rejection, use authorized release signing or a device provisioned for that bundle. Rebuilding the same archive or changing SDK debug profiles does not grant the missing capability. No Hypium test ran.

If a recovery-built pair's `build-artifact.json` is missing or unreadable, rebuild and sign a current pair instead of editing the proof. Damaged build history does not block unrelated HAPs.

When investigating a receipt gap, retain and sanitize copies of `stdout.log` and `hilog.log` before sharing. Compare the named completions with the saved receipt using only the run's stdout and attributable hilog records (UUID marker or verified PID and freshness); `hilog.log` can also contain other processes' lines. A missing terminal case record and a record lost by the parser are distinct problems. Never fill a gap from another process, a selector or a total.

The idle clock starts at `aa test`, not during installation. New run markers, Hypium case milestones, advancing case indices and terminal counts can reset it. Duplicate or mirrored milestones, countdowns and unrelated logs do not. Increase `--idle-timeout-ms` for tests that legitimately go longer without a milestone; `0` leaves only the hard deadline. Progress never extends `--run-timeout-ms`.

When output is buffered, use the run ID in `device-run/runs.jsonl` to read that run's `progress.json`. It records the last observed activity (or none), idle milliseconds as of `updatedAt`, and the evidence directory. The runner updates it on new activity and at heartbeats. A stall preserves the logs and records `reason: stalled` with exit 2, not a test failure. It does not retry and does not authorize taking the device away from a live or unknown owner.

An ordinary, explicitly requested rerun can link one earlier `empty-selection` without a flag. The runner checks both the campaign output and its `pbt-out` layout before choosing a match. A case start, even without a completion, contradicts an empty selection. Both registrations must belong to the same running campaign checkpoint, target, bundle, module and single `Suite` or `Suite#case` selector. The earlier runner must have closed with the device bundle idle, its local host process must have exited, and both retained logs must be readable and free of failures. The rerun must install the same fingerprinted test HAP on unchanged inputs and account for every terminal count with a passed case in that selection. A zero exit with no case records is not recovery. Only the **new** receipt gets `supersedesRun`; this link does not attest any missing property cases or trigger another test.

If that proof is missing or multiple earlier registrations match, inspect the original evidence; never edit receipts. After correcting an inconclusive attempt, explicitly rerun with `--supersedes-run <old-run-id>` if needed. Only a verified nonempty run on the same target/bundle/module and recorded class selection resolves that attempt; original receipts remain, and genuine failing runs cannot be superseded. New index entries include `testClass`; handled setup failures after registration write an atomic inconclusive receipt when the output is writable. New registrations also record host ownership. A killed process may still leave no receipt: offline `finalize` can record a missing attempt older than 30 minutes as aborted under exclusive ownership, provided the recorded owner exited, no device lock exists and retained logs contain no failure evidence (legacy indexes may lack an owner). It never prunes the index or invents a verified result. Absent/already-resolved supersede links are retained warnings; mismatches and genuinely failing targets remain errors. No standalone retry is automatic.
If a failing Hypium case has no matching PandaCheck detail, `failures` records the observed `Suite#case` and its status from `stdout.log` or `hilog.log` beside the receipt. Any count that cannot be attributed stays in one aggregate observation with `testCase: null`; the requested `--class` is not evidence. Unreported seeds and counterexamples stay `null`. Multiple witnesses from one execution count only once toward the failure total; witnesses from a passed case are retained but do not count toward it. Each observation still requires triage. `PANDACHECK_RESULT` alone is not a suite verdict.

After `--ability-grace-ms`, at most one external `aa start -a <ability> -b <bundle> -m <module>` repairs a missing launch, only when `aa test` is still open, no correlated test activity exists and a successful recognized ability dump proves absence. Unknown dumps or existing activity disable recovery; the original deadline still applies. This is never a second `aa test`.

A per-target lock in `~/tmp/fmbot-pbt-device-locks/` blocks overlapping local runs and reports PID, liveness, start time, command and output.
After interruption/timeout it stays when the app is alive or its state is unknown. Inspect the exact `owner.json` for the old run ID/output. Never kill fmbot-pbt or hdc processes or manually delete the lock: a process listing may include this runner and its resume parent. Campaign context exposes their fresh PIDs and campaign lease paths, and common self-kill commands are blocked correctively.
On a later explicit invocation, the shared runner archives a lock at least 30 minutes old only after proving its host owner exited and a recognized device process listing shows that owner's bundle absent. Competing reclaimers are serialized; the archived owner remains inspectable. When the old attempt belongs to the same campaign/selection and is inconclusive, the new invocation links it automatically only if its receipt and retained stdout/hilog contain no failure evidence. Missing receipts do not bypass this check; unreadable logs also prevent automatic linking. Only a verified nonempty result resolves a linked attempt. A `device-locked` receipt records non-execution and does not accumulate delivery debt, but never counts as verified execution. Live, unknown and foreign-host owners remain protected. Never automatically retry an inconclusive or failing test; separate hosts still need external device scheduling.

## Pre-signed archives

```bash
fmbot-pbt build-run --hap app-signed.hap --test-hap test-signed.hap --device-target DEVICE --out pbt-out
# Direct deterministic runner:
fmbot-pbt arkts-test --hap app-signed.hap --test-hap test-signed.hap --target DEVICE --out pbt-out/device
```

Either archive flag selects existing-suite execution: supplied files are installed byte-for-byte and SHA-256 checked.
No build command (even if supplied), signing, model call or property authoring occurs. Omit an archive only if its module is already installed.
HAP metadata supplies bundle/module defaults; one connected device is auto-selected, otherwise supply the target. `--isolate` is incompatible.

This mode does not assert that new source/`--target` changes are in the frozen binary, and invents no campaign `REPORT.md`.
Evidence for `build-run` is `<out>/device-run/<run-id>/result.json` plus logs; CLI exit is `0/1/2`, gated on that exact run's receipt.
To generate new properties, omit the archive flags and run a normal source campaign.

The in-session adapter streams execution during preparation, returns `started: false` (no model workflow queued) with the suite exit code/evidence path, and exposes that code through `campaign_result`.
Both adapters use the same implementation. Source-campaign verification has the case/artifact receipt requirements above.
