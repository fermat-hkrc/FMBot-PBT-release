# Kea GUI testing (experimental)

[中文版 / Chinese version](kea.zh.md) · [Back to installation](installation.md)

`pi-pbt kea` is an **experimental** mode for testing an already installed
HarmonyOS app on a USB-connected phone. It treats the app as a black box and
drives the Kea2 engine over `hdc`: Kea2 explores the GUI while checking
properties that must hold whatever the user does, and reports crashes, ANRs, and
property violations.

## Prerequisites

Three things must be in place before it runs:

| | |
|---|---|
| Phone | connected over USB and unlockable without a PIN; `hdc` on `PATH` (`hdc list targets` shows the serial) |
| Kea2 | a Kea2 checkout with its virtualenv, i.e. `<kea_home>/.venv/bin/kea2` (or `venv/bin/kea2`) exists |
| Decompile signals | `<decompile_home>/mined_all/<package>/signals.json`, mined from the app beforehand. Absent → it aborts rather than explore blind |

## Configuration

Configuration is a `kea.config.yml` in the folder you run it from (the "SUT
folder"). Minimal:

```yaml
package: com.example.app
kea_home: /path/to/Kea2
decompile_home: /path/to/harmony-decompile
```

Then run it from the SUT folder, or point `--config` at a file elsewhere:

```bash
cd /path/to/sut-folder     # the folder holding kea.config.yml
pi-pbt kea                 # interactive TUI
pi-pbt kea -p              # headless/print mode (CI, nohup)
pi-pbt kea -c              # interactive continuation in the latest run directory
pi-pbt kea -p -c           # headless continuation
pi-pbt kea --config other.yml --lang zh
```

`-p` selects headless mode; `-c` is independent and reuses the directory named by
`<configured out>/LATEST` (`pbt-out/LATEST` by default) when that directory still exists. `--config` selects the YAML
file, and `--lang` overrides its output language. These are the command-line
flags; everything else is configured in the file:

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
| `provider`, `model`, `lang` | — | model provider, model ID, and artifact/narration language; CLI `--lang` overrides `lang` |

Depth is the same "how deep do we dig" knob as the
[effort tiers](installation.md#how-deep-it-digs-effort-tiers), so it is not set
twice: `depth:` in the config wins, then `PBT_KEA_DEPTH`, then the tier
(`quick` → `fast`, `standard`/`thorough` → `deep`).

## Artifacts and failures

A stamped run leaves behind:

```text
pbt-out/
  LATEST                                    # path to the newest run directory
  runs/<package>_modeB_<depth>_<stamp>/
    layout.json      one UI dump of the app
    kea/             fast only: generated prop_*.py files
    kea-run/         Kea2's own output (res_*/result_*.json)
    LAST_RUN.json    parsed Kea counters: executions, failures/errors, per property
    REPORT.md        human Kea verdict, including crash/ANR and infra-flake/tarpit notes
```

`LAST_RUN.json` is the machine-readable execution summary for this Kea mode. A
separate source-code PBT campaign writes `report.json`: its validated schema
links source properties, bugs, build evidence, and exact reproduction commands,
and `hook-run` uses it as a CI gate. Kea's workflow does **not** specify that
source report. **Current experimental limitation:** the CLI nevertheless shares
hook-run's final report gate, so it can exit `2` for missing/invalid `report.json`
even after producing GUI results. Do not use this exit code alone as a reliable
GUI verdict, and do not fabricate a source report to satisfy it.
For Kea, inspect `LAST_RUN.json` together with its `REPORT.md` and Kea2 result
files. With `stamp_runs: false`, the same entries are written directly under the
configured `out` directory and no stamped `runs/...` directory is created.

A missing config file, a config without `package:`, or missing decompile signals
stop launch with an explanatory message and exit code `1`, before the phone is
touched. After launch, inspect the Kea results and stderr together; the shared source-report
gate limitation above can affect the final exit code.

## Environment variables

| Variable | Effect |
|---|---|
| `PBT_KEA_DEPTH=fast\|deep` | exploration depth, overriding the effort tier; the config's `depth:` still wins |
| `PBT_KEA_MODE_A_PACKS="pkg.a pkg.b"` | the property packs run at `deep`, when the config sets no `mode_a_packs` |

The general `PBT_EFFORT=quick|standard|thorough` setting supplies the depth only
when `kea.config.yml` does not set `depth:` and `PBT_KEA_DEPTH` is unset:
`quick` selects the short `fast` run; `standard` and `thorough` select `deep`.
