# Checkmarx Security MCP Plugin

A Claude Code plugin that connects Claude to the **Checkmarx Security MCP server** (`security-mcp`), exposing Checkmarx One security workflows directly inside your IDE / CLI session.

> Upstream server: [CheckmarxDev/security-mcp](https://github.com/CheckmarxDev/security-mcp)

---

## Overview

`security-mcp` is a Golang MCP server built by Checkmarx that acts as the bridge between AI assistants and the Checkmarx One platform (SAST, SCA, KICS, Container Security, Secret Detection). This plugin is the thin wrapper that makes the server installable as a Claude Code plugin.

Authentication uses **OAuth 2.0 against Checkmarx IAM** — users sign in through their browser on first use; no API keys need to be pasted or managed.

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
- **Transport** — HTTP (default)
- **Auth** — OAuth 2.0 (authorization code + PKCE) against Checkmarx IAM; tokens stored in the OS keychain and refreshed automatically by Claude Code

---

## Setup

### 1. Prerequisites

- A Checkmarx One account (any tenant).
- A reachable `security-mcp` server endpoint — either a Checkmarx-hosted MCP endpoint for your tenant, or one you run yourself (see [upstream repo](https://github.com/CheckmarxDev/security-mcp)).

### 2. Install the plugin

From the Claude Code CLI:

```bash
# Register the marketplace once
/plugin marketplace add cx-hitesh-madgulkar/MCP-plugin

# Install and enable this plugin
/plugin install checkmarx-security-mcp@MCP-plugin
/plugin enable checkmarx-security-mcp
```

### 3. Tell the plugin which server to talk to

The plugin reads **one** environment variable — the URL of the `security-mcp` server:

| Variable             | Required | Example                                                 | Notes                                 |
| -------------------- | -------- | ------------------------------------------------------- | ------------------------------------- |
| `CHECKMARX_MCP_URL`  | ✅       | `https://mcp.checkmarx.net/api/security-mcp`            | Base URL of the `security-mcp` server |

On Windows (persistent, new shells only):

```powershell
setx CHECKMARX_MCP_URL "https://mcp.checkmarx.net/api/security-mcp"
```

On macOS / Linux:

```bash
export CHECKMARX_MCP_URL="https://mcp.checkmarx.net/api/security-mcp"
```

> 🔐 **No API key / token is required.** Authentication is handled by OAuth the first time you use the server.

### 4. Sign in

Open Claude Code and run:

```
/mcp
```

Select `checkmarx` and choose **Authenticate**. Your default browser will open to the Checkmarx IAM login page. After you sign in, Claude Code stores the resulting access + refresh tokens in your OS keychain and the server dot turns green.

You should now see the Checkmarx tools (e.g. `listProjects`, `triggerScan`, `codeRemediation`) available to Claude.

---

## Authentication

This plugin uses the **standard MCP OAuth flow** (RFC 9728 + RFC 8414):

1. Claude Code connects to `${CHECKMARX_MCP_URL}` with no token.
2. The server returns `401 Unauthorized` with a `WWW-Authenticate` header pointing at its `/.well-known/oauth-protected-resource` document.
3. Claude Code follows that document to the Checkmarx IAM realm's authorization and token endpoints (the server fills these in automatically based on the tenant).
4. Claude Code opens a browser, completes an **authorization code flow with PKCE**, and exchanges the code for access + refresh tokens.
5. Tokens are stored in the OS keychain (Keychain on macOS, Credential Manager on Windows, libsecret on Linux) and refreshed transparently.

OAuth client:
- `client_id`: `ide-integration` (public client; no secret needed)
- Scopes: `ast-api`, `iam-api`, `openid`, `offline_access`
- Callback: `http://localhost:8976/callback` (configurable via `callbackPort` in `.mcp.json`)

On the server side, every inbound token is signature-verified against the IAM realm's JWKS. **No long-lived credentials ever touch the plugin or `.mcp.json`.**

### Re-authenticating

If your refresh token expires or is revoked, run `/mcp` → `checkmarx` → **Re-authenticate**.

---

## Usage

Once enabled and authenticated, just ask Claude in natural language. Examples:

- *"List my Checkmarx projects and show which ones have any CRITICAL findings."*
- *"Trigger a SAST scan on project `payment-api` and wait for it to finish."*
- *"Summarize tenant vulnerabilities for the last week, grouped by severity."*
- *"I got a secret-detection hit for an AWS access key in a Python file — give me remediation steps."*
- *"For `lodash@4.17.20` (npm), is there a safer alternative version?"*
- *"Show me the latest 5 scans for application `web-frontend` and drill into failures."*

Claude picks the right MCP tool (`listFindings`, `triggerScan`, `codeRemediation`, etc.) and formats the result for you.

---

## Troubleshooting

| Symptom                                                | Likely cause                                                                                         |
| ------------------------------------------------------ | ---------------------------------------------------------------------------------------------------- |
| `checkmarx` server missing from `/mcp`                 | `CHECKMARX_MCP_URL` not exported in the shell that launched Claude (restart your terminal after `setx`)  |
| Browser doesn't open / OAuth never completes           | Port `8976` is already in use. Change `callbackPort` in `.mcp.json` and re-auth.                     |
| `401 Unauthorized` after successful auth               | Your IAM user lacks the required scopes (`ast-api`, `iam-api`) — contact your Checkmarx tenant admin. |
| Server dot is red with no obvious error                | Run `claude --debug` from a terminal to see the underlying MCP / OAuth error.                        |
| Tools listed but all return empty results              | Your IAM user is scoped to a different tenant than the resources you're querying.                    |
| OAuth loop after token revocation                      | `/mcp` → `checkmarx` → **Clear authentication**, then authenticate again.                            |

To debug on the server side, run `security-mcp` with `LOG_LEVEL=debug` and check `/health` on `HEALTH_SERVICE_PORT` (default `4321`).

---

## Support

- **Issues / feature requests:** open an issue in the [security-mcp repository](https://github.com/CheckmarxDev/security-mcp/issues).
- **Design docs:** [MCP Server Detailed Design](https://checkmarx.atlassian.net/wiki/spaces/AID/pages/8464105625/MCP+Server+Detailed+Design) (internal).
- **Checkmarx support:** https://checkmarx.com/support/
