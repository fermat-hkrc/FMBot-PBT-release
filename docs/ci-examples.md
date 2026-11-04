# CI examples

[中文](ci-examples.zh.md) · [Usage guide](usage.md#mode-2-merge-request-gate)

Both examples assume pi-pbt is installed on the runner with a model configured. Replace `./build.sh` with the
project's own build command, or drop the step if no build is needed.

## GitHub Actions

```yaml
name: pbt
on: pull_request

jobs:
  pbt:
    runs-on: [self-hosted]
    timeout-minutes: 90
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}
          fetch-depth: 0

      - run: ./build.sh

      - run: >
          pi-pbt hook-run ${{ github.event.pull_request.head.sha }}
          --base ${{ github.event.pull_request.base.sha }}
          --repo . --out "$RUNNER_TEMP/pbt-out"

      - if: always()
        uses: actions/upload-artifact@v4
        with:
          name: pbt-report
          path: |
            ${{ runner.temp }}/pbt-out/REPORT.md
            ${{ runner.temp }}/pbt-out/report.json
            ${{ runner.temp }}/pbt-out/bug_reports/
```

## GitLab CI

```yaml
pbt:
  stage: test
  tags: [pi-pbt]
  timeout: 90m
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  variables:
    GIT_DEPTH: "0"
  script:
    - ./build.sh
    - pi-pbt hook-run "$CI_COMMIT_SHA" --base "$CI_MERGE_REQUEST_DIFF_BASE_SHA" --repo . --out pbt-report
  artifacts:
    when: always
    paths:
      - pbt-report/REPORT.md
      - pbt-report/report.json
      - pbt-report/bug_reports/
```

## Exit codes

| Exit code | Meaning | Pipeline |
|---|---|---|
| `0` | no bug found | pass |
| `1` | bugs found | fail |
| `2` | report missing or incomplete | fail |
| `3` | build failed, nothing tested | fail |
