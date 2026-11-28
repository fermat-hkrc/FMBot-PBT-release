# 用 Claude Code 或 codeagent 运行

已经有能用的 Claude Code（包括基于它改的公司内部版本）或 codeagent，又不方便在 fmbot-pbt 里配置模型密钥时，可以让 fmbot-pbt 通过 ACP（Agent Client Protocol）驱动 agent 完成测试任务。构建、拦截、引导、记录工具和报告仍由 fmbot-pbt 负责，模型和密钥用 agent 自己的配置。

English: [acp.md](acp.md)

## 准备

Claude Code（`--agent claude`，默认）：

1. Claude Code 已安装并能正常对话：在项目目录运行一次 `claude` 确认。
2. 安装 ACP 适配器：`npm install -g @agentclientprotocol/claude-agent-acp`。它应当在 `PATH` 中；npm 全局 bin 不在 `PATH` 上时，fmbot-pbt 也会在常见全局安装位置（fmbot-pbt 自己所在目录、Windows 的 `%APPDATA%\npm`、`~/.npm-global/bin`、`/usr/local/bin`、`/opt/homebrew/bin`）找到它，找不到时会在启动前给出警告。
3. 使用公司内部版本时，把 `CLAUDE_CODE_EXECUTABLE` 设为它的 `claude` 可执行文件路径。

codeagent（`--agent codeagent`）：

1. codeagent 已安装（它的历史目录是 `~/.cac`）。
2. 不需要 export 任何东西。fmbot-pbt 在 `PATH`、`~/.cac`、`~/.codeagent`、Windows 程序目录（`%LOCALAPPDATA%\Programs\codeagent`、`%LOCALAPPDATA%\codeagent`、`%ProgramFiles%\codeagent`）和上面的全局安装位置里找到 `codeagent` 可执行文件，以 `"<路径>" --acp` 启动它，并从找到的位置推导 `CLI_INSTALL_DIR`：可执行文件所在目录，或它在 `bin` 子目录中时取该目录的父目录。手工导出的 `CLI_INSTALL_DIR` 或 `PBT_ACP_AGENT_COMMAND` 优先生效。哪里都找不到时，运行会在 agent 启动前停止，并列出探测过的全部位置。

## 运行

```bash
fmbot-pbt build-run --engine acp --target <目录> --build-cmd "<构建命令>"
fmbot-pbt build-run --engine acp --agent codeagent --target <目录> --build-cmd "<构建命令>"
```

`hook-run` 同样支持 `--engine acp`。结果、输出目录和退出码与默认引擎相同，见[子命令参考](subcommands.zh.md)。

| 参数 | 环境变量 | 说明 |
|---|---|---|
| `--engine acp` | `PBT_ENGINE` | 用 ACP agent 运行；默认 `pi` |
| `--agent claude\|codeagent` | `PBT_ACP_AGENT` | 驱动哪个 agent；默认 `claude` |
| `--agent-command "<命令>"` | `PBT_ACP_AGENT_COMMAND` | 启动 agent ACP 服务的命令；默认自动检测（见上） |
| `--model <模型>` | | 交给 Claude Code 的模型名（`ANTHROPIC_MODEL`）；不给时用 Claude Code 的默认模型 |

## 与默认引擎的区别

- 只支持无人值守运行：不支持 `--tui` 和 `--mode`，`--thinking` 不生效。
- 模型由 agent 决定，`PBT_PHASE_MODELS` 不生效；上下文压缩由 agent 自己处理。
- 扫描和收尾审查各开一个只读的 agent 会话。
- 报告不含 API 性能一节。
- codeagent 直接启动（`"<路径>" --acp`），`CLI_INSTALL_DIR` 从安装位置推导；hook、MCP 工具和拦截的接法与 Claude Code 完全一致。
