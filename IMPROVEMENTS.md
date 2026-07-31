# Support Ticket Resolver Agent — Improvements & Loopholes

Analysis of the current agent implementation. Issues are grouped by category and ordered by severity within each group.

---

## Critical Security Loopholes

### 1. Unpinned supply-chain dependencies in `.mcp.json`

The MCP servers are launched with:
- `npx -y @atlassian/mcp-atlassian` — `-y` auto-accepts any package version, no version pin
- `ghcr.io/github/github-mcp-server` — no image digest pinned

A compromised package release or image tag can silently run malicious code with access to all env vars (Jira token, GitHub PAT, DB URLs, etc.). Pin both to an exact version/digest.

### 3. No validation on the initial ticket ID input

The entry point accepts a raw ticket ID from the user (e.g., `APP-4821`) without sanitizing it before interpolation into Jira API URLs and NRQL queries. A malicious input like `APP-4821'; DROP TABLE--` could be exploited if any downstream tool constructs queries via string concatenation.

### 4. Database credentials embedded in env var URLs

`.env.example` shows patterns like `PAYMENTS_DB_READONLY_URL=postgres://user:password@host/db`. Passwords inside connection strings get logged, echoed in shell history, and exposed in process listings. Use separate host/user/password env vars instead.

---

## High-Priority Architectural Gaps

### 10. No rollback mechanism after fix branch is pushed

If Phase 7 (PR creation) succeeds but the PR is later rejected or the fix is found to be wrong, there is no automated path to close the PR and delete the branch. The agent treats push as terminal.

### 25. No eval / test harness for prompt modules

There are zero tests for the agent's prompt modules. A regression in Phase 4's RCA methodology or Phase 5's fix plan format would go undetected until it silently produces incorrect output on a real ticket. A set of fixture tickets with expected output blocks would enable prompt regression testing.

---

## Configuration & Schema Issues

### 11. No JSON schema for `support-ticket-resolver-config.json`

A typo like `"enabled": "true"` (string instead of boolean) silently disables an integration — including safety guardrails such as the production environment gate. There is no validation step. Adding a JSON Schema and a preflight schema-validation step would catch this before the workflow starts.

### 12. No version field in the config file

If a future version of the agent adds a new required config key, there is no way to detect that the config is stale and missing required fields.

### 13. `serviceRepoMap` has no auto-persistence path

Phase 2 discovers new service→repo mappings and prompts the user to confirm them, but there is no mechanism to write confirmed mappings back to `support-ticket-resolver-config.json`. Every run that encounters an unmapped service must re-discover and re-confirm it.

### 14. `support-ticket-resolver-config.json` reveals org topology

Even with placeholder values, the file structure exposes which observability platforms, databases, and services exist. This file should not be checked into a public repo. Add it to `.gitignore` and provide only a `.example` version.

---

## Missing Integrations / Capability Gaps

### 16. No Grafana / Prometheus support

Absent for teams running self-hosted observability stacks.

### 17. No PagerDuty / OpsGenie integration

Common incident management tools. The agent has no way to acknowledge an alert, update an incident, or read on-call context — all of which would improve RCA quality by correlating the ticket with an ongoing alert and surfacing the on-call engineer.

### 18. No CVE / dependency vulnerability check

When the RCA identifies a vulnerable or outdated dependency as the root cause, the agent has no tool to query a vulnerability database (Snyk, OSV, GitHub Advisory Database) to confirm the CVE and fetch the safe version to pin.

---

## Operational & Quality Gaps

### 22. No dry-run / simulation mode

Users cannot test the agent against a staging Jira project or a sandbox repo without making real changes. A `--dry-run` flag that executes all read-only steps but skips writes (branch push, PR creation, Jira comment, KB publish) is essential for onboarding and testing.

### 23. No multi-ticket or batch mode

The agent handles exactly one ticket per run. During an incident storm (multiple related tickets), it cannot process a set of tickets and correlate findings across them.

### 24. No SLA / priority-based behavior differentiation

P0/SEV1 tickets need faster turnaround and different escalation paths than P3 tickets, but the agent treats all priorities identically except for the production environment gates.

---

## Summary Table

| # | Issue | Severity |
|---|---|---|
| 1 | Unpinned MCP supply-chain deps | Critical |
| 3 | No validation on ticket ID input | High |
| 4 | DB passwords in connection string URLs | High |
| 25 | No eval/test harness for prompt modules | High |
| 10 | No rollback after branch push | Medium |
| 11 | No JSON schema validation for config | Medium |
| 12 | No config version field | Medium |
| 13 | No auto-persistence of discovered repo mappings | Medium |
| 14 | Config file reveals org topology | Medium |
| 17 | No PagerDuty/OpsGenie integration | Medium |
| 18 | No CVE/dependency vulnerability check | Medium |
| 22 | No dry-run mode | Medium |
| 16 | No Grafana/Prometheus support | Low |
| 23 | No multi-ticket/batch mode | Low |
| 24 | No SLA/priority-based differentiation | Low |
