# 用 Claude Code 运行

已经有能用的 Claude Code（包括基于它改的公司内部版本），又不方便在 fmbot-pbt 里配置模型密钥时，可以让 fmbot-pbt 通过 ACP（Agent Client Protocol）驱动 Claude Code 完成测试任务。构建、拦截、引导、记录工具和报告仍由 fmbot-pbt 负责，模型和密钥用 Claude Code 自己的配置。

English: [acp.md](acp.md)

## 准备

1. Claude Code 已安装并能正常对话：在项目目录运行一次 `claude` 确认。
2. 安装 ACP 适配器：`npm install -g @agentclientprotocol/claude-agent-acp`，确认 `claude-agent-acp` 在 `PATH` 中。
3. 使用公司内部版本时，把 `CLAUDE_CODE_EXECUTABLE` 设为它的 `claude` 可执行文件路径。

## 运行

```bash
fmbot-pbt build-run --engine acp --target <目录> --build-cmd "<构建命令>"
```

`hook-run` 同样支持 `--engine acp`。结果、输出目录和退出码与默认引擎相同，见[子命令参考](subcommands.zh.md)。

| 参数 | 环境变量 | 说明 |
|---|---|---|
| `--engine acp` | `PBT_ENGINE` | 用 ACP agent 运行；默认 `pi` |
| `--agent claude` | `PBT_ACP_AGENT` | 目前支持 `claude` |
| `--agent-command "<命令>"` | `PBT_ACP_AGENT_COMMAND` | 启动适配器的命令，默认 `claude-agent-acp` |
| `--model <模型>` | | 交给 Claude Code 的模型名（`ANTHROPIC_MODEL`）；不给时用 Claude Code 的默认模型 |

## 与默认引擎的区别

- 只支持无人值守运行：不支持 `--tui` 和 `--mode`，`--thinking` 不生效。
- 模型由 Claude Code 决定，`PBT_PHASE_MODELS` 不生效；上下文压缩由 Claude Code 处理。
- 扫描和收尾审查各开一个只读的 Claude Code 会话。
- 报告不含 API 性能一节。
