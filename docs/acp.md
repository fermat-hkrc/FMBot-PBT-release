# Running with Claude Code or codeagent

If you already have a working Claude Code (including an in-house build based on it) or codeagent and cannot configure model keys in fmbot-pbt, fmbot-pbt can drive the agent over ACP (Agent Client Protocol) to run the campaign. The build, guards, steering, record tools and report stay with fmbot-pbt; the model and keys come from the agent's own configuration.

中文：[acp.zh.md](acp.zh.md)

## Prepare

Claude Code (`--agent claude`, the default):

1. Claude Code is installed and can chat: run `claude` once in the project directory.
2. Install the ACP adapter: `npm install -g @agentclientprotocol/claude-agent-acp`. It should be on `PATH`; when the npm global bin is not, fmbot-pbt still finds it in the usual global install locations (next to its own executable, `%APPDATA%\npm` on Windows, `~/.npm-global/bin`, `/usr/local/bin`, `/opt/homebrew/bin`) and warns before starting when it is nowhere.
3. For an in-house build, set `CLAUDE_CODE_EXECUTABLE` to its `claude` executable.

codeagent (`--agent codeagent`):

1. codeagent is installed (its historical home is `~/.cac`).
2. Nothing to export. fmbot-pbt finds the `codeagent` executable on `PATH` or in `~/.cac`, `~/.codeagent`, the Windows program locations (`%LOCALAPPDATA%\Programs\codeagent`, `%LOCALAPPDATA%\codeagent`, `%ProgramFiles%\codeagent`) and the global install locations above, starts it as `"<path>" --acp`, and derives `CLI_INSTALL_DIR` from where it found it: the executable's directory, or that directory's parent when the executable sits in a `bin` subdirectory. An exported `CLI_INSTALL_DIR` or `PBT_ACP_AGENT_COMMAND` always wins. When codeagent is nowhere, the run stops before the agent starts and lists every location it probed.

## Run

```bash
fmbot-pbt build-run --engine acp --target <dir> --build-cmd "<build command>"
fmbot-pbt build-run --engine acp --agent codeagent --target <dir> --build-cmd "<build command>"
```

`hook-run` accepts `--engine acp` too. Results, output directory and exit codes are the same as with the default engine; see the [subcommand reference](subcommands.md).

| Flag | Environment | Meaning |
|---|---|---|
| `--engine acp` | `PBT_ENGINE` | Run in an ACP agent; default `pi` |
| `--agent claude\|codeagent` | `PBT_ACP_AGENT` | Which agent to drive; default `claude` |
| `--agent-command "<cmd>"` | `PBT_ACP_AGENT_COMMAND` | Command that starts the agent's ACP server; default: auto-detected (see above) |
| `--model <model>` | | Model name for Claude Code (`ANTHROPIC_MODEL`); default: Claude Code's own |

## Differences from the default engine

- Headless only: `--tui` and `--mode` are not supported, and `--thinking` has no effect.
- The agent chooses the model, so `PBT_PHASE_MODELS` has no effect; it compacts its own context.
- Scan workers and the close-out review each run in a read-only agent session.
- The report has no API performance section.
- codeagent is started directly (`"<path>" --acp`) with `CLI_INSTALL_DIR` derived from its install location; hooks, MCP tools and guards are wired exactly as for Claude Code.
