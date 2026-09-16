# GLM / DeepSeek configuration through CLIProxyAPI

[中文版](models-glm-deepseek.zh.md) · [Installation](installation.md#3-configure-a-model)

This example targets **CLIProxyAPI's Devin channel**, not the vendor's direct
API or an arbitrary enterprise proxy. Check which route actually serves your
model before copying token limits.

## 1. Channel-specific limits

Source checked on 2026-09-16: CLIProxyAPI commit
[`8335eac`](https://github.com/router-for-me/CLIProxyAPI/commit/8335eac731946bd4eff18f500653f93736df53d6),
[active Devin registry](https://github.com/router-for-me/CLIProxyAPI/blob/8335eac731946bd4eff18f500653f93736df53d6/internal/registry/models/devin_models.json#L64-L189).
The loader prefixes IDs with `devin/`; if your deployment defines aliases, use
its actual `/v1/models` IDs.

| Channel model ID | `contextWindow` | Registry maximum output tokens | Example `maxTokens` |
|---|---:|---:|---:|
| `devin/glm-5-2` | 200000 | 128000 | 16384 |
| `devin/glm-5-3` | 1048576 | 128000 | 16384 |
| `devin/glm-5-3-flash` | 1000000 | 128000 | 16384 |
| `devin/deepseek-v4-flash` | 1048576 | 128000 | 16384 |
| `devin/deepseek-v4-1-flash` | 1048576 | 128000 | 16384 |
| `devin/deepseek-v4-pro` | 1048576 | 128000 | 16384 |

The GLM-5.2 route above is registered as **200000**, even if a direct vendor
endpoint advertises 1M. Do not transfer limits between routes just because the
model display names match. Older proxies, aliases and enterprise policies may
impose lower limits. `/v1/models` confirms IDs but may omit token metadata;
check the registry/configuration of that same deployment. Test fixtures and
other models' defaults are not specifications.

## 2. Context window versus output budget

- `contextWindow` is the route's context capacity. It informs history budgeting
  and automatic compaction; exaggerating it can delay compaction beyond the
  service's actual capacity.
- `maxTokens` is the **default requested output budget** in pi 0.85.1's ordinary
  agent path, not just descriptive metadata. The SDK further reduces it when
  the remaining context is smaller. Never set it above the route's output cap
  or confuse it with the context window.
- **16384** here is a starting output budget, not a claim that the route only
  supports 16384. Increase it within the registered limit when needed for long
  reasoning/answers, watching cost, latency and `length` truncations. Tool runs
  need not request the maximum output on every call.

## 3. Example models.json

Prerequisites: CLIProxyAPI has working Devin credentials and its `/v1/models`
lists both IDs below. Port `8317` is the local example; replace `baseUrl` for
an enterprise gateway. Use the **proxy access key**, not an unconfigured vendor
key. Save this to `~/.pi-pbt/agent/models.json`; merge `providers` into an existing
file rather than overwriting other models.

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

This route translates OpenAI Chat Completions into Devin requests:
`reasoning_effort` selects thinking level, `max_completion_tokens` (or
`max_tokens`) supplies the output budget, and function tools are forwarded.
Use `thinkingFormat: "openai"` here, **not** a direct ZAI/DeepSeek `thinking`
object. Change compatibility fields only for a route that actually supports them.

```bash
export CLIPROXY_API_KEY='your-proxy-access-key'
curl --fail --silent --show-error \
  -H "Authorization: Bearer $CLIPROXY_API_KEY" \
  http://127.0.0.1:8317/v1/models
pi-pbt --list-models
pi-pbt --provider cliproxy-devin --model devin/glm-5-2 --thinking high
# Or:
pi-pbt --provider cliproxy-devin --model devin/deepseek-v4-pro --thinking high
```

`--list-models` checks discovery/configuration, not successful tool use. Test a
new route against a small repository first: verify actual tool calls, subsequent
tool-result replay, and completed reports. Setting contextWindow to 1M is not a
fix for unexplained empty responses.

## 4. Direct vendors and other proxies

Do not reuse `devin/...` IDs or these limits on another route. Verify the exact
provider/model revision before changing the URL, ID, compatibility or limits.

- [CLIProxyAPI ID normalization and catalog loading](https://github.com/router-for-me/CLIProxyAPI/blob/8335eac731946bd4eff18f500653f93736df53d6/internal/registry/devin_models.go)
- [CLIProxyAPI request/tool translation](https://github.com/router-for-me/CLIProxyAPI/blob/8335eac731946bd4eff18f500653f93736df53d6/internal/translator/openai/interactions/chat-completions/openai_interactions_request.go#L230-L288)
- [GLM-5.2 direct-vendor specification](https://docs.bigmodel.cn/cn/guide/models/text/glm-5.2)
- [DeepSeek direct-vendor models and limits](https://api-docs.deepseek.com/quick_start/pricing)
- [pi models.json reference](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/models.md)

The example is tested through pi's configuration loader, not against your live
proxy credentials. Keys are environment references; no credentials ship here.
