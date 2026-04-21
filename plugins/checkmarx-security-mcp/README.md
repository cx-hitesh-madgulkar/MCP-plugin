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
- **Remediation** — `packageRemediation`, `imageRemediation`, `codeRemediation` (incl. secret remediation: api_key, password, token, private_key, database_url, aws_access_key, jwt_secret, generic)
- **Transport** — HTTP (default), SSE, or stdio
- **Auth** — OAuth2 against Checkmarx IAM, with JWT signature verification against the tenant's JWKS

---

## Setup

### 1. Prerequisites

- A Checkmarx One tenant and a valid API key (or OAuth2 client credentials).
- A reachable `security-mcp` server endpoint. You can either:
  - Point at a Checkmarx-hosted MCP endpoint for your tenant, **or**
  - Run the server yourself (see [upstream repo](https://github.com/CheckmarxDev/security-mcp) — `docker build -t checkmarx/security-mcp . && docker run -p 8089:8089 checkmarx/security-mcp`).

### 2. Install the plugin (via the MCP-plugin marketplace)

From the Claude Code CLI:

```bash
# Register the marketplace once
/plugin marketplace add cx-hitesh-madgulkar/MCP-plugin

# Install this plugin from it
/plugin install checkmarx-security-mcp@MCP-plugin
```

### 3. Configure environment variables

The plugin reads two variables to wire up the MCP connection:

| Variable             | Required | Example                                                 | Notes                                           |
| -------------------- | -------- | ------------------------------------------------------- | ----------------------------------------------- |
| `CHECKMARX_MCP_URL`  | ✅       | `https://mcp.checkmarx.net/api/security-mcp`            | Base URL of the `security-mcp` server           |
| `CHECKMARX_API_KEY`  | ✅       | `Bearer eyJhbGciOi...`                                  | **Must include the `Bearer ` prefix** (see note) |

> ⚠️ **`Bearer ` prefix is required.** The plugin sets `Authorization: ${CHECKMARX_API_KEY}` verbatim — it does **not** add `Bearer ` for you. Your env var must contain the full header value, including the literal string `Bearer ` followed by a space and the token. Example: `Bearer eyJhbGciOi...`.

Set them in the shell that launches Claude:

```bash
export CHECKMARX_MCP_URL="https://mcp.checkmarx.net/api/security-mcp"
export CHECKMARX_API_KEY="Bearer <your-checkmarx-one-api-key>"
```

### 4. Verify

```bash
/mcp
```

You should see a `checkmarx` server listed as connected, and its tools (e.g. `listProjects`, `triggerScan`, `codeRemediation`) available to Claude.

---

## Authentication

The `security-mcp` server expects a **Bearer token** on every request and verifies it against the Checkmarx IAM JWKS:

- Supply your Checkmarx One API key via `CHECKMARX_API_KEY` (including the `Bearer ` prefix).
- The plugin forwards the full value as the `Authorization` header.
- On the server side, `JWT_VERIFY=true` (default) ensures the token is signature-verified against your tenant's realm under `CX_IAM_URL`.
- OAuth2 scopes used by the server: `ast-api,basic,email,iam-api,profile,roles`.

No credentials are ever stored in the plugin — only read from the environment at runtime.

---

## Usage

Once enabled, just ask Claude in natural language. Examples:

- *"List my Checkmarx projects and show which ones have any CRITICAL findings."*
- *"Trigger a SAST scan on project `payment-api` and wait for it to finish."*
- *"Summarize tenant vulnerabilities for the last week, grouped by severity."*
- *"I got a secret-detection hit for an AWS access key in a Python file — give me remediation steps."*
- *"For `lodash@4.17.20` (npm), is there a safer alternative version?"*
- *"Show me the latest 5 scans for application `web-frontend` and drill into failures."*

Claude will pick the right MCP tool (`listFindings`, `triggerScan`, `codeRemediation`, etc.) and format the result for you.

---

## Troubleshooting

| Symptom                                                | Likely cause                                                                                         |
| ------------------------------------------------------ | ---------------------------------------------------------------------------------------------------- |
| `checkmarx` server missing from `/mcp`                 | Env vars not exported in the shell that launched Claude                                              |
| `401 Unauthorized` on every tool call                  | Expired / wrong API key, **missing `Bearer ` prefix in `CHECKMARX_API_KEY`**, or untrusted issuer     |
| `JWT_VERIFY is enabled but no trust anchor configured` | Server-side: set `CX_IAM_URL` or `JWT_ALLOWED_ISSUERS`                                               |
| Connection hangs / times out                           | Server URL unreachable, or behind a proxy that blocks SSE/HTTP streams                               |
| Tools listed but all return empty results              | API key scoped to a different tenant than the resources you're querying                              |

To debug server-side, run with `LOG_LEVEL=debug` and check `/health` on `HEALTH_SERVICE_PORT` (default `4321`).

---

## Support

- **Issues / feature requests:** open an issue in the [security-mcp repository](https://github.com/CheckmarxDev/security-mcp/issues).
- **Design docs:** [MCP Server Detailed Design](https://checkmarx.atlassian.net/wiki/spaces/AID/pages/8464105625/MCP+Server+Detailed+Design) (internal).
- **Checkmarx support:** https://checkmarx.com/support/
