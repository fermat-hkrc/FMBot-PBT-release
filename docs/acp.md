# Running with Claude Code

If you already have a working Claude Code (including an in-house build based on it) and cannot configure model keys in fmbot-pbt, fmbot-pbt can drive Claude Code over ACP (Agent Client Protocol) to run the campaign. The build, guards, steering, record tools and report stay with fmbot-pbt; the model and keys come from Claude Code's own configuration.

中文：[acp.zh.md](acp.zh.md)

## Prepare

1. Claude Code is installed and can chat: run `claude` once in the project directory.
2. Install the ACP adapter: `npm install -g @agentclientprotocol/claude-agent-acp`, and check that `claude-agent-acp` is on `PATH`.
3. For an in-house build, set `CLAUDE_CODE_EXECUTABLE` to its `claude` executable.

## Run

```bash
fmbot-pbt build-run --engine acp --target <dir> --build-cmd "<build command>"
```

`hook-run` accepts `--engine acp` too. Results, output directory and exit codes are the same as with the default engine; see the [subcommand reference](subcommands.md).

| Flag | Environment | Meaning |
|---|---|---|
| `--engine acp` | `PBT_ENGINE` | Run in an ACP agent; default `pi` |
| `--agent claude` | `PBT_ACP_AGENT` | Currently `claude` |
| `--agent-command "<cmd>"` | `PBT_ACP_AGENT_COMMAND` | Command that starts the adapter; default `claude-agent-acp` |
| `--model <model>` | | Model name for Claude Code (`ANTHROPIC_MODEL`); default: Claude Code's own |

## Differences from the default engine

- Headless only: `--tui` and `--mode` are not supported, and `--thinking` has no effect.
- Claude Code chooses the model, so `PBT_PHASE_MODELS` has no effect; Claude Code compacts its own context.
- Scan workers and the close-out review each run in a read-only Claude Code session.
- The report has no API performance section.
