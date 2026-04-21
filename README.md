# MCP-plugin — Checkmarx Plugin Marketplace for Claude Code

This repository is a **Claude Code plugin marketplace** published by Checkmarx. It currently hosts one plugin:

| Plugin                                                           | Description                                                                                |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| [`checkmarx-security-mcp`](./plugins/checkmarx-security-mcp/)    | Connects Claude to the Checkmarx One platform via the `security-mcp` server (SAST / SCA / KICS / secret detection / remediation). |

## Install

From the Claude Code CLI:

```bash
# Register this repo as a marketplace (one-time)
/plugin marketplace add cx-hitesh-madgulkar/MCP-plugin

# Install a plugin from it
/plugin install checkmarx-security-mcp@MCP-plugin

# Enable
/plugin enable checkmarx-security-mcp
```

> `/plugin` is only available in the Claude Code **CLI**. The VS Code extension does not expose this command — use the terminal.

See each plugin's own README for its setup, environment variables, and usage:

- [checkmarx-security-mcp](./plugins/checkmarx-security-mcp/README.md)

## Repository layout

```
MCP-plugin/
├─ .claude-plugin/
│  └─ marketplace.json                ← marketplace manifest (lists plugins)
├─ plugins/
│  └─ checkmarx-security-mcp/
│     ├─ .claude-plugin/
│     │  └─ plugin.json               ← plugin metadata
│     ├─ .mcp.json                    ← MCP server config (HTTP + Bearer auth)
│     └─ README.md
└─ README.md                          ← (this file)
```

## Contributing a new plugin

1. Create `plugins/<your-plugin>/` with `.claude-plugin/plugin.json`, `.mcp.json` (if it wraps an MCP server), and `README.md`.
2. Add an entry for it to `plugins` in [`.claude-plugin/marketplace.json`](./.claude-plugin/marketplace.json).
3. Open a PR.
