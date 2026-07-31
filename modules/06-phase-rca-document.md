### PHASE 6 — RCA Document Generation

> **Guardrail — Secret/PII Scrubber:** Before writing the RCA document, run the full content through `@modules/guardrails/01-secret-pii-scrubber.md`. Replace any detected secrets or PII with `[REDACTED]`. Never embed raw log lines that contain credentials in a document that will be committed or published.

Generate a professional, shareable RCA document in Markdown format. Save it as `RCA-<TICKET-ID>-<YYYY-MM-DD>.md` in the root of the primary fix branch.

Document structure:

```markdown
# RCA: <Ticket ID> — <Ticket Summary>

**Date:** <YYYY-MM-DD>  
**Severity:** <Priority>  
**Status:** Resolved (Pending Review)  
**Author:** support-ticket-resolver agent  
**Jira Ticket:** [<ID>](<JIRA_BASE_URL>/browse/<ID>)  

---

## Executive Summary
<2–3 sentence non-technical summary of what happened, impact, and resolution.>

## Incident Timeline
| Timestamp | Event |
|-----------|-------|
| ... | ... |

## Affected Services
- <service> → <repo>

## Root Cause
<Detailed root cause statement.>

## Contributing Factors
- <factor>

## Impact Assessment
- **Users Affected:** <estimate>
- **Duration:** <duration>
- **Data Impact:** <yes/no + description>
- **Revenue Impact:** <if determinable>

## Log Evidence
<Key log excerpts and query results supporting the RCA.>

## Fix Description
<Technical description of the code/config/infra changes made.>

## Testing
<Description of tests added and validation approach.>

## Prevention & Follow-up Actions
| Action | Owner | Due Date |
|--------|-------|----------|
| Add alerting for <error pattern> | On-call team | <date> |
| Add test coverage for <scenario> | Dev team | <date> |
| Review <component> for similar patterns | Tech lead | <date> |

## References
- Jira: [<ID>](<link>)
- PR: [<PR title>](<PR link>)
- Logs: <observability platform link>
```
