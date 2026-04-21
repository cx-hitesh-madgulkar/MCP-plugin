# Checkmarx Security MCP Plugin

A Claude Code plugin that connects Claude to the **Checkmarx Security MCP server** (`security-mcp`), exposing Checkmarx One security workflows directly inside your IDE / CLI session.

> Upstream server: [CheckmarxDev/security-mcp](https://github.com/CheckmarxDev/security-mcp)

---

## Overview

`security-mcp` is a Golang MCP server built by Checkmarx that acts as the bridge between AI assistants and the Checkmarx One platform (SAST, SCA, KICS, Container Security, Secret Detection). This plugin is the thin wrapper that makes the server installable as a Claude Code plugin.

Once enabled, Claude can:

- Inspect your Checkmarx projects, applications, and scans
- Plan and trigger scans on demand
- Summarize findings and risk posture across tenants
- Generate remediation guidance for vulnerable **packages**, **container images**, and **source code** (including secrets).

---

## Features

- **Project & application management** — `listProjects`, `listApplications`, `resolveProject`, `createProject`, `deleteProject`, `getProjectConfig`, `createApplication`, `associateProject`, `getApplicationDetails`
- **Scan lifecycle** — `planScan`, `triggerScan`, `getScanDetails`, `listScans`, `getLatestScans`
- **Findings & risk** — `listFindings`, `getFindingDetails`, `getFindingsSummary`, `getTenantVulnerabilitiesSummary`, `getRiskSummary`, `listRiskResults`, `updateRiskResultStatus`
- **Overview dashboards** — `getApplicationsOverviewCount`, `getApplicationsOverviewAggregate`, `listProjectsOverview`, `getProjectsOverviewAggregate`
- **Remediation** — `packageRemediation`, `imageRemediation`, `codeRemediation` (incl. secret remediation for api_key, password, token, private_key, database_url, aws_access_key, jwt_secret, generic)

---

## Setup

### 1. Prerequisites

- A Checkmarx One account and a valid API key / access token.
- A reachable `security-mcp` server endpoint (a Checkmarx-hosted URL for your tenant, or one you run yourself).

### 2. Set environment variables **before** starting Claude

The plugin reads two variables:

| Variable             | Required | Example                                                 | Notes                                                |
| -------------------- | -------- | ------------------------------------------------------- | ---------------------------------------------------- |
| `CHECKMARX_MCP_URL`  | ✅       | `https://<region>.ast.checkmarx.net/<tenant>/api/security-mcp` | Full URL of the `security-mcp` server                |
| `CHECKMARX_API_KEY`  | ✅       | `Bearer eyJhbGciOi...`                                  | **Must include the literal `Bearer ` prefix**         |

> ⚠️ **`Bearer ` prefix is required.** The plugin passes `CHECKMARX_API_KEY` straight through as the `Authorization` header — it does **not** add `Bearer ` for you. Your env var must include the word `Bearer` followed by a space and then the token.

**Windows (PowerShell / cmd):**

```powershell
setx CHECKMARX_MCP_URL "https://<region>.ast.checkmarx.net/<tenant>/api/security-mcp"
setx CHECKMARX_API_KEY "Bearer <your-checkmarx-token>"
```

> `setx` only affects **new** terminals and processes. Fully quit Claude Code / VS Code (check Task Manager for lingering `claude.exe` / `Code.exe` processes) and reopen before continuing.

**macOS / Linux:** add to `~/.zshrc` or `~/.bashrc`:

```bash
export CHECKMARX_MCP_URL="https://<region>.ast.checkmarx.net/<tenant>/api/security-mcp"
export CHECKMARX_API_KEY="Bearer <your-checkmarx-token>"
```

Then reopen your terminal.

### 3. Install and enable the plugin

```bash
/plugin marketplace add cx-hitesh-madgulkar/MCP-plugin
/plugin install checkmarx-security-mcp@MCP-plugin
/plugin enable checkmarx-security-mcp
```

Make sure the toggle in `/plugin` → **Manage Plugins** is **on** (not grey).

### 4. Verify

```
/reload-plugins
```

Look for `1 plugin MCP server` (not `0`). Then:

```
/mcp
```

You should see a `checkmarx` entry. If it shows as connected, you're done — the Checkmarx tools (`listProjects`, `triggerScan`, `codeRemediation`, etc.) are now available to Claude.

---

## Authentication

The `security-mcp` server expects a **Bearer token** on every request and verifies its signature against the Checkmarx IAM JWKS:

- You supply the token via `CHECKMARX_API_KEY` (including the `Bearer ` prefix).
- The plugin forwards the full value as the `Authorization` header on each HTTP request.
- No credentials are stored in the plugin — only read from the environment at runtime.

Getting a token:

1. Sign in to Checkmarx One.
2. **Account Settings → API Keys** → create a new key.
3. Copy the value and prefix it with `Bearer ` when setting the env var.

---

## Usage

Just ask Claude in natural language. Examples:

- *"List my Checkmarx projects and show which ones have any CRITICAL findings."*
- *"Trigger a SAST scan on project `payment-api` and wait for it to finish."*
- *"Summarize tenant vulnerabilities for the last week, grouped by severity."*
- *"I got a secret-detection hit for an AWS access key in a Python file — give me remediation steps."*
- *"For `lodash@4.17.20` (npm), is there a safer alternative version?"*
- *"Show me the latest 5 scans for application `web-frontend` and drill into failures."*

Claude picks the right MCP tool and formats the result.

---

## Troubleshooting

| Symptom                                                | Likely cause                                                                                                            |
| ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------- |
| `/reload-plugins` reports `0 plugin MCP servers`       | Plugin toggle is **off**, or `.mcp.json` contains a schema error. Check `/plugin` Manage Plugins.                       |
| `checkmarx` missing from `/mcp` output                 | Env vars not exported in the shell that launched Claude. Fully restart your terminal / VS Code after `setx`.            |
| `401 Unauthorized` on every tool call                  | Wrong / expired token, or **`CHECKMARX_API_KEY` missing the `Bearer ` prefix**.                                         |
| Red dot on `checkmarx` in `/mcp`                       | Connection / auth failure. Run `claude --debug` to see the underlying error.                                            |
| Tools listed but all return empty results              | Token scoped to a different tenant than the resources you're querying.                                                  |
| `Failed to parse` error pointing at some other `.mcp.json` | Stray `.mcp.json` somewhere on your system (often in home dir or current working dir). Delete or fix it — not this plugin. |

Quick env var sanity check in a fresh terminal:

```powershell
# PowerShell
$env:CHECKMARX_MCP_URL
$env:CHECKMARX_API_KEY
```

Both should print non-empty values. If either is empty, re-run `setx` and open **another** new terminal.

---

## Support

- **Issues / feature requests:** [security-mcp repository](https://github.com/CheckmarxDev/security-mcp/issues)
- **Design docs:** [MCP Server Detailed Design](https://checkmarx.atlassian.net/wiki/spaces/AID/pages/8464105625/MCP+Server+Detailed+Design) (internal)
- **Checkmarx support:** https://checkmarx.com/support/
