# Installation Guide

This guide walks through setting up the `support-ticket-resolver` agent from scratch.

---

## Prerequisites

| Requirement | Minimum version | Purpose |
|---|---|---|
| Claude Code CLI | latest | Runs the agent |
| Node.js | 18+ | Atlassian MCP server (npx) |
| Python | 3.9+ | Agent scripts and tooling |
| Docker | 24+ | GitHub MCP server (optional) |
| AWS CLI | 2.x | CloudWatch queries (if enabled) |
| Git | 2.x | Fix branch operations |

---

## Step 1 — Copy the agent into your Claude agents directory

Claude Code looks for agent definitions in `~/.claude/agents/`. Create a symlink or copy the directory:

```bash
# Option A — symlink (recommended: edits to source are reflected immediately)
ln -s /path/to/support-ticket-resolver-agent/support-ticket-resolver.md \
      ~/.claude/agents/support-ticket-resolver.md

# Option B — copy the file
cp support-ticket-resolver.md ~/.claude/agents/support-ticket-resolver.md
```

> **Note:** The `@modules/...` directives in the agent file are resolved relative to the file's location at runtime. If you use a symlink, all module paths must be accessible from the symlink's directory. Using an absolute path in CLAUDE.md (`@/path/to/modules/...`) is the most robust approach for symlinked setups.

---

## Step 2 — Configure integrations

Copy the example config and edit it for your environment:

```bash
cp support-ticket-resolver-config.json support-ticket-resolver-config.local.json
```

> The agent loads `support-ticket-resolver-config.json` by convention. Rename your local copy to overwrite it, or adjust the `@modules/00-configuration.md` reference to point to your custom file path.

Open `support-ticket-resolver-config.json` and update every placeholder:

### Jira
```json
"jira": {
  "enabled": true,
  "baseUrl": "https://YOUR-ORG.atlassian.net",
  "project": "APP",
  "apiTokenEnvVar": "JIRA_API_TOKEN",
  "userEmail": "you@your-org.com"
}
```

### GitHub
```json
"github": {
  "enabled": true,
  "org": "your-github-org",
  "defaultBaseBranch": "main",
  "apiTokenEnvVar": "GITHUB_TOKEN",
  "reviewerMap": [
    { "jiraEmail": "alice@your-org.com", "githubHandle": "alice-gh" }
  ],
  "serviceRepoMap": [
    { "service": "payment-service", "repo": "payments-api", "path": "/" }
  ]
}
```

`serviceRepoMap` is the most important field to get right. Add one entry per service that could appear in Jira tickets. If a service is missing, the agent will fall back to GitHub search and prompt you if it cannot resolve the repo.

### Observability (enable only what you have)

Set `"enabled": true` for each platform you want the agent to query. Leave the others at `false` — the agent will skip them entirely.

```json
"observability": {
  "newRelic": {
    "enabled": true,
    "accountId": "1234567",
    "apiKeyEnvVar": "NEW_RELIC_API_KEY"
  },
  "azureAppInsights": { "enabled": false, ... },
  "cloudwatch": { "enabled": false, ... }
}
```

For CloudWatch, populate `serviceLogGroupMap` with one entry per service:
```json
"serviceLogGroupMap": [
  {
    "service": "payment-service",
    "logGroups": ["/aws/ecs/payment-service", "/aws/lambda/payment-processor"]
  }
]
```

### Knowledgebase
```json
"knowledgebase": {
  "enabled": true,
  "platform": "confluence",
  "confluence": {
    "baseUrl": "https://YOUR-ORG.atlassian.net/wiki",
    "spaceKey": "ENG",
    "apiTokenEnvVar": "CONFLUENCE_API_TOKEN",
    "userEmail": "you@your-org.com"
  }
}
```

Set `"platform": "sharepoint"` and fill the `sharepoint` block instead if your organisation uses SharePoint.

### Database (optional, disabled by default)
```json
"database": {
  "enabled": false,
  "allowedEnvironments": ["staging"],
  "connections": [
    {
      "label": "payments-db-staging",
      "service": "payment-service",
      "engine": "postgres",
      "readOnly": true,
      "connectionStringEnvVar": "PAYMENTS_DB_READONLY_URL"
    }
  ]
}
```

Only enable this if you want the agent to run investigative SQL queries. `allowedEnvironments` prevents the agent from connecting to production databases.

---

## Step 3 — Set environment variables

Copy `.env.example` and fill in your actual secret values:

```bash
cp .env.example .env
```

Edit `.env`:

```bash
# Jira / Confluence (same Atlassian API token works for both)
JIRA_API_TOKEN=<your-atlassian-api-token>
CONFLUENCE_API_TOKEN=<your-atlassian-api-token>   # can be same token

# GitHub
GITHUB_TOKEN=<your-github-personal-access-token>

# New Relic (if enabled)
NEW_RELIC_API_KEY=NRAK-xxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Azure App Insights (if enabled)
AZURE_AI_API_KEY=<your-app-insights-api-key>

# AWS CloudWatch (if enabled)
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
# AWS_SESSION_TOKEN=...    # only if using STS/SSO

# SharePoint (if enabled)
SHAREPOINT_CLIENT_ID=<azure-app-registration-client-id>
SHAREPOINT_CLIENT_SECRET=<azure-app-registration-client-secret>

# Database read-only URLs (if enabled)
PAYMENTS_DB_READONLY_URL=postgresql://readonly_user:pass@host:5432/payments
USERS_DB_READONLY_URL=mysql://readonly_user:pass@host:3306/users
```

Source the file in your shell session before running the agent:

```bash
source .env
# or: export $(grep -v '^#' .env | xargs)
```

> **Security:** Never commit `.env` to version control. It is already listed in `.gitignore` (add it if missing).

---

## Step 4 — Configure MCP servers

### Atlassian MCP (recommended)

The Atlassian MCP server provides richer Jira and Confluence integration. It requires the `@atlassian/mcp-atlassian` package (run via npx — no global install needed).

The `.mcp.json` in this repo is pre-configured:

```json
{
  "mcpServers": {
    "Atlassian": {
      "command": "npx",
      "args": ["-y", "@atlassian/mcp-atlassian"],
      "env": {
        "JIRA_BASE_URL": "<your-jira-base-url>",
        "JIRA_USER_EMAIL": "<your-email>",
        "JIRA_API_TOKEN": "<env-var-reference>"
      }
    }
  }
}
```

Place `.mcp.json` in the project directory or in `~/.claude/` for global availability. Claude Code automatically reads it and starts the MCP server when the agent runs.

To verify the Atlassian MCP server is working:
```bash
npx -y @atlassian/mcp-atlassian --help
```

If you do not want to use MCP, set `"mcp.atlassian.enabled": false` in the config — the agent will fall back to Jira REST API calls via `WebFetch`.

### GitHub MCP (optional)

The GitHub MCP server runs as a Docker container. It is optional — the agent falls back to GitHub REST API if disabled.

```bash
# Pull the image once
docker pull ghcr.io/github/github-mcp-server

# Verify
docker run --rm ghcr.io/github/github-mcp-server --version
```

Set `"mcp.github.enabled": true` in config to activate. The `.mcp.json` declares the Docker run config.

---

## Step 5 — Verify AWS CLI (CloudWatch only)

If CloudWatch is enabled, ensure the AWS CLI is configured and the credentials have the required permissions:

```bash
# Verify identity
aws sts get-caller-identity

# Verify CloudWatch Logs access
aws logs describe-log-groups --max-items 5 --region us-east-1
```

Required IAM permissions:
```json
{
  "Effect": "Allow",
  "Action": [
    "logs:DescribeLogGroups",
    "logs:StartQuery",
    "logs:GetQueryResults",
    "logs:FilterLogEvents"
  ],
  "Resource": "arn:aws:logs:<region>:<account-id>:log-group:*"
}
```

---

## Step 6 — Register the agent in Claude Code

Claude Code discovers agents from `~/.claude/agents/`. After copying or symlinking the agent file (Step 1), verify it is registered:

```bash
# In Claude Code CLI — list available agents
/agents
```

You should see `support-ticket-resolver` in the list.

---

## Step 7 — Run your first ticket

In Claude Code:

```
support-ticket-resolver APP-4821
```

Or via the orchestrator agent:
```
We have a critical ticket APP-4821 that's been open since yesterday. Can you resolve it?
```

The agent will:
1. Run Phase 0 pre-flight (validate all credentials)
2. Fetch the ticket from Jira
3. Map affected services to repos
4. Query observability platforms
5. Search the knowledgebase
6. Perform RCA
7. Present a fix plan and wait for your approval
8. Implement the fix, run tests
9. Push a PR and assign the reviewer
10. Publish the RCA to the knowledgebase

---

## Token requirements

### Jira API Token
- Create at: https://id.atlassian.com/manage-profile/security/api-tokens
- Scope: the token inherits the permissions of the user account that created it. The user needs read access to the project and write access to add comments.

### GitHub Personal Access Token (Classic)
Required scopes:
- `repo` — read/write access to private repositories
- `read:org` — required for org-level user search
- `read:user` — required for reviewer identity lookup

Or use a **Fine-grained token** with permissions:
- Repository: Contents (Read & Write), Pull requests (Read & Write), Metadata (Read)
- Organisation: Members (Read)

### New Relic User API Key
- Create at: https://one.newrelic.com → User menu → API Keys → Create key → Type: User
- Required role: Full platform user (or custom role with "NRQL queries" permission)

### Azure App Insights API Key
- Create at: Azure Portal → Application Insights → {resource} → Configure → API Access → Create API key
- Required permission: "Read telemetry"

### Azure AD App Registration (SharePoint)
- Register an app in Azure Portal → Entra ID → App registrations
- Add Microsoft Graph API permission: `Sites.ReadWrite.All`
- Create a client secret and note the Client ID and Tenant ID

---

## Updating the agent

To update to a new version of this agent:

```bash
# If you used a symlink — pull the repo and the symlink automatically reflects the update
git pull

# If you copied the file — re-copy it
cp support-ticket-resolver.md ~/.claude/agents/support-ticket-resolver.md
```

Module files are loaded at runtime from the path specified in the agent file, so updating the modules directory is sufficient — no Claude Code restart required.

---

## Troubleshooting

### "Pre-flight credential check failed"
- Check that all required env vars are exported in your current shell session.
- Run `env | grep -E 'JIRA|GITHUB|NEW_RELIC|AZURE|AWS|CONFLUENCE|SHAREPOINT'` to verify they are set.

### "MCP server not found"
- Verify npx can reach `@atlassian/mcp-atlassian`: `npx -y @atlassian/mcp-atlassian --version`
- Check `.mcp.json` is in the correct directory (project root or `~/.claude/`).

### "Repo resolution failed for service X"
- Add the service to `serviceRepoMap` in config with the correct `repo` and `path`.
- Ensure `GITHUB_TOKEN` has `repo` scope and can access the target repository.

### "NRQL query forbidden"
- The API key must belong to a Full Platform User or have the "NRQL queries" role explicitly assigned.
- Check the `accountId` in config matches the account where the key was created.

### "AWS CloudWatch AccessDeniedException"
- Attach the IAM policy shown in Step 5 to the IAM user or role associated with `AWS_ACCESS_KEY_ID`.
- If using STS/SSO, ensure `AWS_SESSION_TOKEN` is exported alongside the key and secret.

### Agent seems to ignore guardrail stop signals
- Guardrails issue STOP prompts — the agent waits for your reply. Type your choice (Y/N/A/B/C as shown) and press Enter.
- If the agent continues past a STOP without a response, check the Claude Code version (requires a version that supports sub-agent instruction files correctly).
