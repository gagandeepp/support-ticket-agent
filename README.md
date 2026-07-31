# Support Ticket Resolver Agent

A production-grade Claude Code sub-agent that autonomously resolves Jira support tickets end-to-end: from ticket ingestion through root cause analysis, code fix, pull request creation, and RCA publication — all in a single automated workflow.

---

## What it does

Given a Jira ticket ID, this agent:

1. **Ingests the ticket** — fetches full metadata, extracts affected services, error signatures, and environment details.
2. **Maps services to repos** — resolves each service to a GitHub repository using a config-driven map, falling back to GitHub search.
3. **Investigates logs** — queries New Relic, Azure Application Insights, and/or AWS CloudWatch for error spikes, exceptions, slow transactions, and infrastructure anomalies within the incident time window.
4. **Searches the knowledgebase** — looks up prior incidents, runbooks, and architectural context in Confluence or SharePoint.
5. **Performs RCA** — synthesizes findings using 5 Whys and Fishbone methodology to identify root cause, contributing factors, and blast radius.
6. **Implements a targeted fix** — presents a full change plan for approval, then applies the minimal code/config/dependency/infra change needed to resolve the root cause.
7. **Generates an RCA document** — writes a structured, reviewer-ready Markdown RCA to the fix branch.
8. **Opens a pull request** — pushes the fix branch, creates a PR with the RCA attached, and assigns the Jira ticket's assignee as reviewer.
9. **Publishes the RCA** — uploads the RCA document to Confluence or SharePoint for institutional knowledge.

Every action is gated by a comprehensive guardrail system. The agent never acts destructively without explicit user confirmation.

---

## Architecture

```
support-ticket-resolver.md          ← Main agent orchestrator (YAML + @imports)
support-ticket-resolver-config.json ← Runtime configuration (all integrations)
support-ticket-resolver-tools.json  ← Tool manifest (maps config to allowed tools)
.mcp.json                           ← MCP server declarations
.env.example                        ← Required environment variable stubs

modules/
  00-configuration.md               ← Config schema reference
  01-phase-ticket-ingestion.md      ← Phase 1: Jira fetch + parse
  02-phase-repo-mapping.md          ← Phase 2: service → repo resolution
  03-phase-observability.md         ← Phase 3: NR + AI + CloudWatch queries
  03b-phase-knowledgebase.md        ← Phase 3b: Confluence / SharePoint search
  04-phase-rca.md                   ← Phase 4: root cause analysis
  05-phase-fix.md                   ← Phase 5: fix plan, confirmation, implementation
  06-phase-rca-document.md          ← Phase 6: RCA Markdown generation
  07-phase-pr-creation.md           ← Phase 7: branch push, PR, reviewer assignment
  08-final-output.md                ← Phase 8: knowledgebase publish + resolution report
  error-handling.md                 ← Global error handling rules
  quality-standards.md              ← Output quality constraints
  memory-instructions.md            ← Agent memory behaviour

  guardrails/
    00-preflight-credentials.md     ← Credential validation at startup
    01-secret-pii-scrubber.md       ← Secret/PII detection and redaction
    02-prompt-injection-detection.md← External data injection protection
    03-production-environment-gates.md ← Extra confirmation gates for prod tickets
    04-protected-file-patterns.md   ← Deny list for sensitive file modifications
    05-concurrent-execution-check.md← Duplicate run / duplicate PR detection
    06-ticket-validity-check.md     ← Ticket project, status, type, duplicate checks
    07-diff-scope-enforcement.md    ← Change size warning and hard limits
    08-db-query-safety.md           ← Read-only, SELECT-only, row-cap enforcement
    09-reviewer-identity-verification.md ← GitHub handle verification before assignment
    10-observability-query-cost-guard.md ← Time window and scan cost limits
    11-rca-publish-gate.md          ← Pre-publish completeness and safety checks
    12-audit-trail.md               ← Structured event log throughout the workflow
    13-fix-regression-verification.md   ← Automated regression test run post-fix
```

---

## Integrations

| Integration | Purpose | Protocol |
|---|---|---|
| Jira | Ticket fetch, comment, transition | MCP (Atlassian) or REST API |
| GitHub | Repo lookup, branch push, PR creation, reviewer assignment | MCP (github-mcp-server) or REST API |
| New Relic | Error rate, transaction, infrastructure queries | NerdGraph GraphQL |
| Azure Application Insights | Exception, request, trace queries | REST (`ai.applicationinsights.io`) |
| AWS CloudWatch | Log Insights queries | AWS CLI (`aws logs`) |
| Confluence | Knowledgebase search + RCA publication | MCP (Atlassian) or REST API |
| SharePoint | Knowledgebase search + RCA publication | Microsoft Graph REST |
| Databases | Read-only investigative queries | Direct connection (postgres/mysql/mssql/mongodb) |

Each integration is independently enabled/disabled in `support-ticket-resolver-config.json`. The agent adapts its tool usage at runtime based on the config.

---

## Guardrail system

14 guardrail modules enforce safety, correctness, and auditability throughout the workflow:

| # | Guardrail | When applied |
|---|---|---|
| 00 | Pre-flight credential check | Before Phase 1 — blocks start if any required credential is missing |
| 01 | Secret/PII scrubber | Before any external write or publish |
| 02 | Prompt injection detection | After every external data fetch |
| 03 | Production environment gates | Throughout — extra Y/N gates on prod tickets |
| 04 | Protected file patterns | Phase 5 Step 1 — confirm before touching secrets, CI/CD, IaC |
| 05 | Concurrent execution check | Phase 1 end + Phase 7 start — prevents duplicate PRs |
| 06 | Ticket validity check | After Phase 1 parse — checks status, type, duplicates |
| 07 | Diff scope enforcement | Phase 5 Step 2 (plan) + post-implementation |
| 08 | DB query safety | Before every database query |
| 09 | Reviewer identity verification | Phase 7 Step 1 — confirms handle before PR assignment |
| 10 | Observability query cost guard | Before observability queries — time window + scan cost |
| 11 | RCA publish gate | Phase 8 Step 1 — completeness, security, scrub |
| 12 | Audit trail | Throughout — structured JSON event log |
| 13 | Fix regression verification | Phase 5 Step 3 — runs affected tests before push |

---

## Configuration

All runtime settings live in `support-ticket-resolver-config.json`. Key sections:

### `mcp`
Controls whether Atlassian and GitHub operations use MCP servers (preferred when available) or fall back to REST API calls.

```json
"mcp": {
  "atlassian": { "enabled": true, "toolPrefix": "mcp__claude_ai_Atlassian" },
  "github": { "enabled": false, "toolPrefix": "mcp__github" }
}
```

### `jira`
Jira Cloud instance URL, project key, and credentials.

### `github`
Organisation name, default base branch, token env var, `reviewerMap` (Jira email → GitHub handle), and `serviceRepoMap` (service name → repo/path).

### `observability`
Independent `enabled` flags and credentials for New Relic, Azure App Insights, and CloudWatch. CloudWatch includes a `serviceLogGroupMap` for log group resolution.

### `knowledgebase`
Platform (`confluence` or `sharepoint`) plus connection details. Used for both search (Phase 3b) and RCA publication (Phase 8).

### `database`
`enabled` flag, `allowedEnvironments` list, and an array of `connections` — each with a label, service name, engine type, and a read-only connection string env var.

---

## Tool activation

`support-ticket-resolver-tools.json` documents which tool groups map to which config flags. Only tools for enabled integrations need to be in the agent's `tools:` allowlist. The file also records the `preflightCommands` needed for AWS CloudWatch.

---

## Workflow phases

```
Phase 0  Pre-flight credential validation (all integrations)
Phase 1  Ticket ingestion — Jira fetch, parse, guardrail checks
Phase 2  Repository mapping — config lookup → GitHub search fallback
Phase 3  Observability investigation — NR / App Insights / CloudWatch
Phase 3b Knowledgebase search — Confluence / SharePoint
Phase 4  Root cause analysis — 5 Whys + Fishbone + timeline
Phase 5  Fix implementation — plan → confirm → implement → verify
Phase 6  RCA document generation — structured Markdown to fix branch
Phase 7  Pull request creation — push, PR, reviewer assignment, Jira comment
Phase 8  Final output — knowledgebase publish + resolution report
```

Each phase is a separate Markdown module. Guardrail references are inline at every relevant step using `@modules/guardrails/XX.md` directives.

---

## MCP vs REST API fallback

The agent is designed to work with or without MCP servers configured:

| Operation | MCP path | REST API fallback |
|---|---|---|
| Fetch Jira ticket | `mcp__claude_ai_Atlassian__getJiraIssue` | `GET /rest/api/3/issue/{id}` |
| Add Jira comment | `mcp__claude_ai_Atlassian__addCommentToJiraIssue` | `POST /rest/api/3/issue/{id}/comment` |
| Search Confluence | `mcp__claude_ai_Atlassian__searchConfluenceUsingCql` | `GET /rest/api/content/search?cql=...` |
| Create Confluence page | `mcp__claude_ai_Atlassian__createConfluencePage` | `POST /rest/api/content` |
| Create GitHub PR | `mcp__github__create_pull_request` | `POST /repos/{org}/{repo}/pulls` |
| Assign reviewer | `mcp__github__request_reviewers` | `POST /pulls/{n}/requested_reviewers` |

SharePoint and observability platforms always use REST/CLI — no MCP server is configured for them.

---

## Security model

- All external data (Jira fields, log results, KB pages) is treated as untrusted input and scanned for prompt injection patterns before use.
- Secrets are never logged. Credential values are referenced by env var name only.
- Database access is enforced read-only at the query level (SELECT-only, LIMIT-capped).
- Production tickets trigger five numbered confirmation gates (G1–G5) that must each be explicitly approved.
- Security Vulnerability ticket types cause the PR to be created as a draft and suppress automatic RCA publication.
- All fix changes require user approval of a detailed plan before a single file is touched.
- Force-push to any branch is absolutely prohibited.
