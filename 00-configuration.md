## Configuration (resolved at agent installation)

Before executing any workflow, read your installed configuration from `support-ticket-resolver-config.json` (located in the project root or the path provided during installation).

> **First-time setup:** Copy `support-ticket-resolver-config.example.json` to `support-ticket-resolver-config.json` and fill in your values. The real config file is gitignored and must never be committed.

---

### Preflight: Config Validation (run before Phase 0)

Run these two checks immediately after reading the config file. Both must pass before the workflow can start.

#### 1. Schema validation

Validate `support-ticket-resolver-config.json` against `support-ticket-resolver-config.schema.json` (located in the project root).

Key errors to surface explicitly:

| Symptom | Example | Why it matters |
|---|---|---|
| Wrong type on `enabled` | `"enabled": "true"` (string) | Silently disables integrations including production environment gates |
| Missing required field | `jira.baseUrl` absent | Phase 1 cannot make Jira calls |
| Invalid `engine` value | `"engine": "oracle"` | Agent would attempt an unsupported DB engine |
| `readOnly: false` on a DB connection | — | Agent must refuse non-read-only DB connections |
| Unknown top-level key | `"extra": {...}` | Indicates config was edited without schema awareness |

If validation fails:

```
⛔ CONFIG VALIDATION FAILED — CANNOT PROCEED

  Errors found in support-ticket-resolver-config.json:

    • jira.enabled: expected boolean, got string ("true")
    • database.connections[0].readOnly: expected true, got false
    • github.org: required field is missing

  Fix these errors and retry. Schema: support-ticket-resolver-config.schema.json
```

**STOP. Do not proceed until the config is valid.**

#### 2. Version check

Read `configVersion` from the config file. The current required version is **`1.0.0`**.

| `configVersion` value | Action |
|---|---|
| `"1.0.0"` | ✓ Proceed |
| Field absent | ✗ Stale config — see below |
| Any other value | ✗ Version mismatch — see below |

If `configVersion` is absent or does not match `1.0.0`:

```
⛔ CONFIG VERSION MISMATCH — CANNOT PROCEED

  Expected configVersion: 1.0.0
  Found:                  <value or "field missing">

  Your config file is out of date. To update:
    1. Compare your support-ticket-resolver-config.json against
       support-ticket-resolver-config.example.json to identify missing or renamed keys.
    2. Add or update the required fields.
    3. Set configVersion to "1.0.0".

  STOP. Do not proceed until configVersion matches.
```

---

### MCP setup

MCP server definitions live in `.mcp.json` at the project root. Two servers are pre-configured:

| Server key | Covers | How to enable |
|---|---|---|
| `Atlassian` | Jira + Confluence | Remote OAuth via claude.ai Settings → Integrations, **or** run `@atlassian/mcp-atlassian` locally |
| `github` | Repos, PRs, commits | Run `ghcr.io/github/github-mcp-server` via Docker (token in `GITHUB_TOKEN`) |

The `mcp` section in `support-ticket-resolver-config.json` controls which servers are active and what tool-name prefix they produce:

```json
"mcp": {
  "atlassian": { "enabled": true,  "toolPrefix": "mcp__claude_ai_Atlassian" },
  "github":    { "enabled": false, "toolPrefix": "mcp__github" }
}
```

**`toolPrefix` values by setup:**

| Setup | `toolPrefix` |
|---|---|
| claude.ai remote Atlassian integration | `mcp__claude_ai_Atlassian` |
| Local `@atlassian/mcp-atlassian` server named `Atlassian` | `mcp__Atlassian` |
| GitHub Docker server named `github` | `mcp__github` |

**Fallback rule:** At every Jira, Confluence, and GitHub call point, the agent checks `mcp.<server>.enabled`. If `true`, it calls `<toolPrefix>__<toolName>`. If `false`, it falls back to the equivalent REST API call via `WebFetch` or `Bash`. Both paths produce identical output blocks.

---

### Tool activation rules

Tool definitions — including integration groups, config activation paths, and per-engine CLI patterns — are maintained in `support-ticket-resolver-tools.json`. Read that file for the canonical mapping.

Every integration section in the config carries an `"enabled"` boolean. **Before invoking any integration-specific tool or API call, check the corresponding `enabled` flag.** If `false` (or the section is absent), skip that integration entirely — do not call its tools, do not ask for credentials, and note the skip in your phase output.

| Config path | Activates | Tools group in manifest |
|---|---|---|
| `jira.enabled` | Jira ticket fetch, search, comment, transition | `jira` |
| `github.enabled` | Repo lookup, branch creation, PR creation | `github` |
| `observability.newRelic.enabled` | New Relic NRQL queries | `observability_newrelic` |
| `observability.azureAppInsights.enabled` | Azure App Insights KQL queries | `observability_azure` |
| `observability.cloudwatch.enabled` | AWS CloudWatch Logs Insights queries | `observability_cloudwatch` |
| `knowledgebase.enabled` + `platform = "confluence"` | Confluence page search & fetch | `knowledgebase_confluence` |
| `knowledgebase.enabled` + `platform = "sharepoint"` | SharePoint document search & fetch | `knowledgebase_sharepoint` |
| `database.enabled` | Read-only queries across one or more DB servers | `database` |

At minimum, `jira.enabled` must be `true` for the workflow to proceed. All other integrations are optional.

### Config schema

```json
{
  "configVersion": "1.0.0",
  "jira": {
    "enabled": true,
    "baseUrl": "<JIRA_BASE_URL>",
    "project": "<DEFAULT_PROJECT_KEY>",
    "apiTokenEnvVar": "JIRA_API_TOKEN",
    "userEmail": "<JIRA_USER_EMAIL>"
  },
  "github": {
    "enabled": true,
    "org": "<GITHUB_ORG>",
    "repos": ["<REPO_1>", "<REPO_2>"],
    "defaultBaseBranch": "main",
    "apiTokenEnvVar": "GITHUB_TOKEN"
  },
  "observability": {
    "newRelic": {
      "enabled": true,
      "accountId": "<NR_ACCOUNT_ID>",
      "apiKeyEnvVar": "NEW_RELIC_API_KEY"
    },
    "azureAppInsights": {
      "enabled": false,
      "appId": "<AI_APP_ID>",
      "apiKeyEnvVar": "AZURE_AI_API_KEY"
    }
  },
  "knowledgebase": {
    "enabled": true,
    "platform": "confluence",
    "confluence": {
      "baseUrl": "<CONFLUENCE_BASE_URL>",
      "spaceKey": "<SPACE_KEY>",
      "apiTokenEnvVar": "CONFLUENCE_API_TOKEN",
      "userEmail": "<CONFLUENCE_USER_EMAIL>"
    },
    "sharepoint": {
      "tenantId": "<AZURE_TENANT_ID>",
      "siteUrl": "<SHAREPOINT_SITE_URL>",
      "clientIdEnvVar": "SHAREPOINT_CLIENT_ID",
      "clientSecretEnvVar": "SHAREPOINT_CLIENT_SECRET"
    }
  },
  "database": {
    "enabled": false,
    "allowedEnvironments": ["staging"],
    "maxQueryTimeoutSeconds": 5,
    "connections": [
      {
        "label": "<FRIENDLY_NAME>",
        "service": "<SERVICE_NAME>",
        "engine": "postgres",
        "host": "<DB_HOST>",
        "port": 5432,
        "database": "<DB_NAME>",
        "readOnly": true,
        "connectionStringEnvVar": "<SERVICE>_DB_READONLY_URL"
      },
      {
        "label": "<FRIENDLY_NAME>",
        "service": "<SERVICE_NAME>",
        "engine": "mysql",
        "host": "<DB_HOST>",
        "port": 3306,
        "database": "<DB_NAME>",
        "readOnly": true,
        "connectionStringEnvVar": "<SERVICE>_DB_READONLY_URL"
      },
      {
        "label": "<FRIENDLY_NAME>",
        "service": "<SERVICE_NAME>",
        "engine": "mssql",
        "host": "<DB_HOST>",
        "port": 1433,
        "database": "<DB_NAME>",
        "readOnly": true,
        "connectionStringEnvVar": "<SERVICE>_DB_READONLY_URL"
      },
      {
        "label": "<FRIENDLY_NAME>",
        "service": "<SERVICE_NAME>",
        "engine": "mongodb",
        "host": "<DB_HOST>",
        "port": 27017,
        "database": "<DB_NAME>",
        "readOnly": true,
        "connectionStringEnvVar": "<SERVICE>_DB_READONLY_URL"
      }
    ]
  }
}
```

Connection field reference:
- `label`                   — human-friendly name shown in RCA output
- `service`                 — matches a service name from Phase 1 affected-services extraction
- `engine`                  — one of: postgres | mysql | mssql | mongodb
- `host` / `port`           — used when constructing CLI args directly
- `database`                — target database / schema name
- `readOnly`                — must be true; agent refuses to connect if false
- `connectionStringEnvVar`  — env var holding the read-only connection string
- `allowedEnvironments`     — global guard; agent checks the ticket's environment against this list before querying any connection
- `maxQueryTimeoutSeconds`  — enforced via CLI flags (psql: statement_timeout, mysql: --connect-timeout, sqlcmd: -t, mongosh: maxTimeMS)

If the config file is missing or malformed, STOP and ask the user to complete installation before proceeding.
