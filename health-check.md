## Health Check

A standalone integration validation command. Run this to verify all enabled integrations are reachable and credentials are valid **without processing any ticket**. No external writes are made. No Jira tickets are fetched. No branches are created.

Invoke by asking the agent:
```
support-ticket-resolver --health-check
```
or:
```
Run a health check for the support ticket resolver.
```

---

### When to use

- After rotating credentials or API keys
- Before an on-call shift or major incident window
- After adding or enabling a new integration in config
- To diagnose why a recent run failed at pre-flight

---

### Step 1 — Load and validate config

Read `support-ticket-resolver-config.json`. Validate it against `support-ticket-resolver-config.schema.json`.

| Result | Action |
|---|---|
| Valid | Proceed |
| Schema violation | Report which field failed and stop — health check cannot proceed with an invalid config |
| File not found | Report: `Config file not found. Copy support-ticket-resolver-config.example.json and fill in your values.` and stop |

---

### Step 2 — Check integrations

For each integration, run the same pre-flight dry-run defined in `@modules/guardrails/00-preflight-credentials.md`. Collect results without stopping on individual failures — continue through all integrations regardless of outcome.

Report each result as one line:

```
✅ PASS     <integration-name>  — <brief confirmation, e.g. "authenticated as user@example.com">
❌ FAIL     <integration-name>  — <error: HTTP status or exception, e.g. "401 Unauthorized — token revoked">
⏭️  SKIPPED  <integration-name>  — disabled in config
```

Integrations to check (in this order):

| Integration | Config flag | Check |
|---|---|---|
| Jira | always on | Authenticate and fetch issue metadata for the configured project (e.g. `GET /rest/api/3/project/<project>`) |
| GitHub | `github.enabled` | Verify token with `GET /user` (REST) or equivalent MCP call |
| New Relic | `observability.newRelic.enabled` | NerdGraph probe: `SELECT 1 FROM Transaction SINCE 1 minute ago LIMIT 1` |
| Azure App Insights | `observability.azureAppInsights.enabled` | KQL probe: `traces \| take 1` |
| AWS CloudWatch | `observability.cloudwatch.enabled` | `aws sts get-caller-identity` + `aws logs describe-log-groups --max-items 1` |
| Confluence | `knowledgebase.enabled` + `platform="confluence"` | Fetch space metadata for configured `spaceKey` |
| SharePoint | `knowledgebase.enabled` + `platform="sharepoint"` | Acquire OAuth2 token, fetch site metadata |
| MongoDB metrics | `metrics.mongodb.enabled` | `mongosh <url> --quiet --eval "db.runCommand({ping:1})"` — verify connectivity and that `insertOne` is permitted on the target collection via `db.runCommand({ hello: 1 })` |

---

### Step 3 — Check local disk state

Report the status of the agent's local data directories:

```bash
# Log directory
ls -lh .claude/logs/ 2>/dev/null | tail -5

# Metrics index
wc -l .claude/metrics/index.jsonl 2>/dev/null

# Active checkpoints
ls .claude/checkpoint-*.json 2>/dev/null
```

Format the output as:

```
📁 Local state
  Logs      : .claude/logs/      — <N files, latest: audit-APP-XXXX-<ts>.jsonl (12 KB)> | <directory empty> | <not created yet>
  Metrics   : .claude/metrics/   — index.jsonl (<N> runs recorded) | <not created yet>
  Checkpoints: <none active> | <APP-4821 (started 2024-01-14, phase 3 — 2 days old)>
```

If `.claude/logs/` or `.claude/metrics/` do not exist, note that they will be created on the first run — this is not an error.

---

### Step 4 — Recent run summary

If `.claude/metrics/index.jsonl` exists and contains at least one entry, show the last 5 runs:

```bash
tail -5 .claude/metrics/index.jsonl | jq -r '[.ts, .ticketId, .outcome, (.durationSeconds | tostring) + "s", "tokens:", (.tokenSpent | tostring) + " spent / " + (.tokenLeft | tostring) + " left"] | join("  ")'
```

Format as a table:

```
📊 Recent runs (last 5)
  Timestamp              Ticket       Outcome     Duration   Tokens
  2024-01-15T14:32:07Z   APP-4821     completed   342s       45234 spent / 154766 left
  2024-01-14T09:15:22Z   SRE-1102     completed   198s       31088 spent / 168912 left
  2024-01-13T16:44:01Z   APP-4790     aborted     87s        12400 spent / 187600 left
```

If the index does not exist or is empty, output: `No previous runs recorded.`

---

### Step 5 — Health check summary

After checking all integrations and local state, output the consolidated result:

```
🏥 HEALTH CHECK — support-ticket-resolver
  Config    : ✅ valid (v<configVersion>)
  Timestamp : <ISO 8601>

  Integrations:
    ✅ PASS     Jira               authenticated as user@example.com
    ✅ PASS     GitHub             token valid, org: acme-corp
    ✅ PASS     New Relic          account 1234567 — NRQL permission confirmed
    ❌ FAIL     Azure App Insights 403 Forbidden — API key lacks "Read telemetry" permission
    ⏭️  SKIPPED  AWS CloudWatch     disabled in config
    ✅ PASS     Confluence         space ENG accessible, 142 pages
    ⏭️  SKIPPED  SharePoint         disabled in config
    ✅ PASS     MongoDB metrics    ping ok — agent_metrics.agent_runs reachable, write permitted

  Local state:
    Logs     : 8 files, latest 45 KB
    Metrics  : 12 runs recorded
    Checkpoints: none active

  Overall: ⚠️  1 integration failed — resolve before running a ticket

  To fix Azure App Insights:
    Azure Portal → Application Insights → <resource> → Configure → API Access
    → Create API key → Required permission: "Read telemetry"
    Then: export <varName>="<new-key>"
```

**Overall status rules:**
- All enabled integrations pass → `✅ Ready — all integrations healthy`
- One or more enabled integrations fail → `⚠️  <N> integration(s) failed — resolve before running a ticket`
- Config invalid → `❌ Config invalid — fix errors above before running`

Do not emit any audit trail events or metrics during a health check — this command is read-only.
