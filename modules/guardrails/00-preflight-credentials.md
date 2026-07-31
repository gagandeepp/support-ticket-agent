## GUARDRAIL — Phase 0: Pre-flight Credential Validation

Run this before Phase 1. Validate credentials for every enabled integration simultaneously and surface all failures in one block. Do not proceed until all pass or the user explicitly disables failing integrations.

---

### Checks to run

Execute all checks in parallel. Record each result as ✓ / ✗ / — (skipped).

#### Jira (`jira.enabled = true`)
```
# Env var check
<jira.apiTokenEnvVar> must be set and non-empty

# Auth dry-run
GET <jira.baseUrl>/rest/api/3/myself
Authorization: Basic base64(<jira.userEmail>:<JIRA_API_TOKEN>)
```
| Response | Status |
|---|---|
| `200` + `accountId` present | ✓ |
| `401` | ✗ — token invalid or revoked |
| `403` | ✗ — token valid but account lacks Browse Projects permission |
| env var missing | ✗ — credential absent |

#### GitHub (`github.enabled = true`)
```
# Env var check
<github.apiTokenEnvVar> must be set and non-empty

# Auth dry-run
GET https://api.github.com/user
Authorization: Bearer <GITHUB_TOKEN>
```
| Response | Status |
|---|---|
| `200` + `login` present | ✓ |
| `401` | ✗ — token invalid |
| `403` | ✗ — insufficient scopes (need repo, pull_request) |

#### New Relic (`observability.newRelic.enabled = true`)
```
# Env var check
<observability.newRelic.apiKeyEnvVar> must be set and non-empty

# Auth dry-run (NerdGraph)
POST https://api.newrelic.com/graphql
Api-Key: <apiKey>
{ "query": "{ actor { user { name } } }" }
```
| Response | Status |
|---|---|
| `200` + `data.actor.user.name` | ✓ |
| `200` + `errors[].type = UNAUTHORIZED` | ✗ — key invalid |
| `200` + `errors[].type = FORBIDDEN` | ✗ — key valid, missing NRQL scope |
| `401` / `403` | ✗ — authentication failure |

#### Azure App Insights (`observability.azureAppInsights.enabled = true`)
```
# Env var check
<observability.azureAppInsights.apiKeyEnvVar> must be set and non-empty

# Auth dry-run
POST https://api.applicationinsights.io/v1/apps/<appId>/query
X-Api-Key: <apiKey>
{ "query": "traces | take 1" }
```
| Response | Status |
|---|---|
| `200` + `tables` present | ✓ |
| `401` | ✗ — key invalid |
| `403` | ✗ — key lacks Read Telemetry permission |
| `404` | ✗ — appId not found or key has no access |

#### CloudWatch (`observability.cloudwatch.enabled = true`)
```
# Env var check
<accessKeyIdEnvVar> and <secretAccessKeyEnvVar> must both be set and non-empty

# Identity dry-run
aws sts get-caller-identity --region <region>
```
| Response | Status |
|---|---|
| Returns `UserId` + `Account` + `Arn` | ✓ |
| `InvalidClientTokenId` | ✗ — credentials invalid |
| `ExpiredTokenException` | ✗ — session token expired |
| env var(s) missing | ✗ — credentials absent |

#### Confluence (`knowledgebase.enabled = true`, `platform = "confluence"`)
```
# Env var check
<confluence.apiTokenEnvVar> must be set and non-empty

# Auth dry-run
GET <confluence.baseUrl>/rest/api/user/current
Authorization: Basic base64(<confluence.userEmail>:<CONFLUENCE_API_TOKEN>)
```
| Response | Status |
|---|---|
| `200` + `accountId` | ✓ |
| `401` | ✗ — token invalid |
| `403` | ✗ — token lacks Confluence read access |

#### SharePoint (`knowledgebase.enabled = true`, `platform = "sharepoint"`)
```
# Env var check
<clientIdEnvVar> and <clientSecretEnvVar> must both be set and non-empty

# Auth dry-run (token grant)
POST https://login.microsoftonline.com/<tenantId>/oauth2/v2.0/token
  grant_type=client_credentials
  client_id=<SHAREPOINT_CLIENT_ID>
  client_secret=<SHAREPOINT_CLIENT_SECRET>
  scope=https://graph.microsoft.com/.default
```
| Response | Status |
|---|---|
| `200` + `access_token` | ✓ |
| `401` / `invalid_client` | ✗ — client ID or secret invalid |
| `403` | ✗ — app lacks Sites.Read.All permission |

#### Database (per entry in `database.connections`, when `database.enabled = true`)
```
# Env var check (per connection)
<connectionStringEnvVar> must be set and non-empty for each connection
```
Connection string presence only — no live DB connection attempted at pre-flight.

---

### Pre-flight output block

```
🔒 PRE-FLIGHT CREDENTIAL CHECK — <TICKET-ID>

  ✓ Jira              <jira.userEmail> authenticated
  ✓ GitHub            token valid — org <github.org> accessible
  ✗ New Relic         NEW_RELIC_API_KEY not set
  ✗ CloudWatch        AWS_ACCESS_KEY_ID not set
  ✓ Confluence        <confluence.userEmail> authenticated
  — SharePoint        skipped (knowledgebase.platform = "confluence")
  ✓ Database          2 connection string(s) present (live check deferred to Phase 3)

  ❌ 2 integration(s) failed pre-flight.
```

**If any check fails:**

```
⛔ PRE-FLIGHT FAILED — CANNOT PROCEED

  Failed integrations:
    • New Relic   : NEW_RELIC_API_KEY is not set
    • CloudWatch  : AWS_ACCESS_KEY_ID is not set

  Options:
    A) Fix the failing credentials (see per-integration guidance in Phase 3)
       and confirm to retry pre-flight.
    B) Disable the failing integrations in support-ticket-resolver-config.json:
         observability.newRelic.enabled = false
         observability.cloudwatch.enabled = false
       and confirm to proceed without them.
```

**STOP. Do not advance to Phase 1 until the user responds.**

Initialize the audit log once pre-flight passes:
@modules/guardrails/12-audit-trail.md
