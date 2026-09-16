# GLM / DeepSeek 配置：以 CLIProxyAPI 渠道为准

[English](models-glm-deepseek.md) · [返回安装指南](installation.zh.md#3-配置模型)

本页给通过 **CLIProxyAPI 的 Devin 渠道**使用 GLM、DeepSeek 的 `models.json` 示例。
不是智谱/DeepSeek 官方直连接口的通用参数，也不是任意企业代理的保证。

## 1. 先核对渠道和模型 ID

以下数值取自 CLIProxyAPI 提交
[`8335eac`](https://github.com/router-for-me/CLIProxyAPI/commit/8335eac731946bd4eff18f500653f93736df53d6)
的[有效 Devin 注册表](https://github.com/router-for-me/CLIProxyAPI/blob/8335eac731946bd4eff18f500653f93736df53d6/internal/registry/models/devin_models.json#L64-L189)，核对日期为 2026-09-16。
注册器会加 `devin/` 前缀；部署若配置了别名，以该部署 `/v1/models` 返回的 ID 为准。

| 渠道模型 ID | `contextWindow` | 注册表最大输出 tokens | 本页示例 `maxTokens` |
|---|---:|---:|---:|
| `devin/glm-5-2` | 200000 | 128000 | 16384 |
| `devin/glm-5-3` | 1048576 | 128000 | 16384 |
| `devin/glm-5-3-flash` | 1000000 | 128000 | 16384 |
| `devin/deepseek-v4-flash` | 1048576 | 128000 | 16384 |
| `devin/deepseek-v4-1-flash` | 1048576 | 128000 | 16384 |
| `devin/deepseek-v4-pro` | 1048576 | 128000 | 16384 |

**不能只按模型显示名照抄窗口。** 例如这里的 GLM-5.2 渠道登记是 200000，不能因为
某个厂商直连页面写 1M，就把此代理路线填成 1000000。旧版 CLIProxyAPI、本地自定义
别名或网关策略可能更低。`/v1/models` 能证明可用 ID，但标准返回值不一定提供 token
上限；仍须核对同一部署的注册表/配置。没有数据时，不要把测试夹具或其他模型的值当规格。

## 2. 两个 token 字段不是一回事

- `contextWindow`：这条路线允许的上下文窗口，影响历史预算和自动压缩时机。不能
  虚报，否则客户端可能在服务端已受限时仍不收缩上下文。
- `maxTokens`：pi 0.85.1 常规 agent 请求的**默认输出预算**，不只是说明文字。
  SDK 会结合剩余上下文进一步收紧它。它不应超过渠道最大输出，也不应误填为上下文长度。
- 本页用 **16384** 作为起步输出预算，不声称渠道上限只有 16384。需要更长的推理/回答，
  可在该路线上限以内调大，同时观察耗时、费用和 `length` 截断。工具任务不需要每次都
  请求注册表允许的最大输出。

## 3. 可复制示例

前提：你已配置 CLIProxyAPI 的 Devin 凭据，`/v1/models` 中确实能看到下面两个 ID。
示例使用本机端口 `8317`；远程企业网关应替换 `baseUrl`，密钥必须是代理的访问密钥，
不是未经配置就把厂商密钥直接塞给代理。

保存到 `~/.pi-pbt/agent/models.json`。已有文件请合并 `providers`，不要覆盖其他模型。

```json
{
  "providers": {
    "cliproxy-devin": {
      "api": "openai-completions",
      "baseUrl": "http://127.0.0.1:8317/v1",
      "apiKey": "$CLIPROXY_API_KEY",
      "compat": {
        "supportsStore": false,
        "supportsDeveloperRole": false,
        "supportsReasoningEffort": true,
        "maxTokensField": "max_completion_tokens",
        "thinkingFormat": "openai"
      },
      "models": [
        {
          "id": "devin/glm-5-2",
          "name": "GLM-5.2 via CLIProxyAPI Devin",
          "reasoning": true,
          "input": ["text"],
          "contextWindow": 200000,
          "maxTokens": 16384,
          "thinkingLevelMap": {
            "off": "none",
            "minimal": null,
            "low": null,
            "medium": null,
            "high": "high",
            "xhigh": null,
            "max": null
          }
        },
        {
          "id": "devin/deepseek-v4-pro",
          "name": "DeepSeek V4 Pro via CLIProxyAPI Devin",
          "reasoning": true,
          "input": ["text"],
          "contextWindow": 1048576,
          "maxTokens": 16384,
          "thinkingLevelMap": {
            "off": null,
            "minimal": null,
            "low": null,
            "medium": null,
            "high": "high",
            "xhigh": null,
            "max": "max"
          }
        }
      ]
    }
  }
}
```

这里走 OpenAI Chat Completions → Devin 转换：代理读取 `reasoning_effort`，将
`max_completion_tokens`（或 `max_tokens`）映射为输出预算，并转发 function tools。
因此本例用 `thinkingFormat: "openai"`，**不**照搬直连 ZAI/DeepSeek 的 `thinking`
对象配置。只有在换成对应厂商直连或已验证支持其参数的路线时才换兼容字段。

```bash
export CLIPROXY_API_KEY='你的代理访问密钥'
curl --fail --silent --show-error \
  -H "Authorization: Bearer $CLIPROXY_API_KEY" \
  http://127.0.0.1:8317/v1/models
pi-pbt --list-models
pi-pbt --provider cliproxy-devin --model devin/glm-5-2 --thinking high
# 或
pi-pbt --provider cliproxy-devin --model devin/deepseek-v4-pro --thinking high
```

`--list-models` 只核对配置和模型可发现性，不证明网关能完成工具调用。首次用新路线时，
先对小型测试仓库跑一轮，确认响应包含实际 tool calls、工具结果能继续回传且报告完成。
不要在问题未明时把 `contextWindow` 调到 1M 来“解决”空响应。

## 4. 官方直连 / 其他代理

若不是 Devin 渠道，不要原样使用上表的 `devin/...` ID 或 token 数值。
核对实际服务商与模型版本，再改 `baseUrl`、ID、兼容参数和上限。

- [CLIProxyAPI 渠道 ID 规范化与目录加载](https://github.com/router-for-me/CLIProxyAPI/blob/8335eac731946bd4eff18f500653f93736df53d6/internal/registry/devin_models.go)
- [CLIProxyAPI 请求参数与 tools 转换](https://github.com/router-for-me/CLIProxyAPI/blob/8335eac731946bd4eff18f500653f93736df53d6/internal/translator/openai/interactions/chat-completions/openai_interactions_request.go#L230-L288)
- [智谱官方 GLM-5.2 规格](https://docs.bigmodel.cn/cn/guide/models/text/glm-5.2)
- [DeepSeek 官方模型与限制](https://api-docs.deepseek.com/quick_start/pricing)
- [pi 的 models.json 字段说明](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/models.md)

本页示例经过 pi 配置解析测试；没有用你的代理密钥进行在线调用测试。配置中的密钥使用
环境变量引用，示例不携带凭据。
