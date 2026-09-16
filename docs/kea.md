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

Depth is the same "how deep do we dig" knob as the
[effort tiers](installation.md#how-deep-it-digs-effort-tiers), so it is not set
twice: `depth:` in the config wins, then `PBT_KEA_DEPTH`, then the tier
(`quick` → `fast`, `standard`/`thorough` → `deep`).

## Artifacts and failures

A run leaves behind:

```text
pbt-out/
  LATEST                                    # points at the newest run directory
  runs/<package>_modeB_<depth>_<stamp>/
    layout.json      one UI dump of the app
    kea-run/         Kea2's own output (res_*/result_*.json)
    LAST_RUN.json    parsed counters: executions, failures, per property
    REPORT.md        the verdict
```

A missing config file, a config without `package:`, or missing decompile signals
stop it with an explanatory message and exit code `1`, before the phone is
touched.

## Environment variables

| Variable | Effect |
|---|---|
| `PBT_KEA_DEPTH=fast\|deep` | exploration depth, overriding the effort tier; the config's `depth:` still wins |
| `PBT_KEA_MODE_A_PACKS="pkg.a pkg.b"` | the property packs run at `deep`, when the config sets no `mode_a_packs` |

The general `PBT_EFFORT=quick|standard|thorough` setting supplies the depth only
when `kea.config.yml` does not set `depth:` and `PBT_KEA_DEPTH` is unset:
`quick` selects the short `fast` run; `standard` and `thorough` select `deep`.
