# CI 配置示例

[English](ci-examples.md) · [使用指南](usage.zh.md#模式二mr-门禁)

两个示例都假设 runner 上已安装 fmbot-pbt 并配置好模型。`./build.sh` 换成项目自己的构建命令，不需要构建时删掉这一步。

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
          fmbot-pbt hook-run ${{ github.event.pull_request.head.sha }}
          --base ${{ github.event.pull_request.base.sha }}
          --repo . --out "$RUNNER_TEMP/pbt-out" --lang zh

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
  tags: [fmbot-pbt]
  timeout: 90m
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  variables:
    GIT_DEPTH: "0"
  script:
    - ./build.sh
    - fmbot-pbt hook-run "$CI_COMMIT_SHA" --base "$CI_MERGE_REQUEST_DIFF_BASE_SHA" --repo . --out pbt-report --lang zh
  artifacts:
    when: always
    paths:
      - pbt-report/REPORT.md
      - pbt-report/report.json
      - pbt-report/bug_reports/
```

## 退出码

| 退出码 | 含义 | 流水线 |
|---|---|---|
| `0` | 未发现缺陷 | 通过 |
| `1` | 发现缺陷 | 失败 |
| `2` | 报告缺失或不完整 | 失败 |
| `3` | 构建失败，未执行测试 | 失败 |
