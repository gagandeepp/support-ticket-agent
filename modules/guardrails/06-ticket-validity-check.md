## GUARDRAIL — Ticket Validity & Type Check

Apply immediately after the Ticket Summary block is produced in Phase 1. Validate that the ticket is an appropriate target for this workflow before investing further investigation effort.

---

### Check 1 — Project key match

Verify `fields.project.key` from the Jira response matches `jira.project` in config.

**If mismatch:**
```
⚠️ PROJECT KEY MISMATCH

  Ticket project : <fields.project.key>
  Config project : <jira.project>

  This agent is configured for project "<jira.project>".
  Processing tickets from other projects may produce incorrect service/repo mappings.

  Proceed anyway? [Y / N]
```

---

### Check 2 — Ticket status

Verify `fields.status.name` is actionable. Flag if the ticket is already terminal.

| Status (case-insensitive) | Action |
|---|---|
| `Open`, `In Progress`, `Reopened`, `To Do` | ✓ Proceed |
| `Done`, `Closed`, `Resolved` | ⚠️ Warn — see prompt below |
| `Won't Fix`, `Duplicate`, `Invalid` | ⚠️ Warn — see prompt below |

**If terminal status:**
```
⚠️ TICKET ALREADY CLOSED

  Ticket : <TICKET-ID>
  Status : <status>
  Resolved at : <resolutiondate, if available>

  This ticket is marked as <status>. Running a full investigation may be unnecessary.

  Options:
    A) Proceed — re-investigate (useful for regression analysis or post-mortems).
    B) Abandon — close this run; no fix or PR will be created.
```

---

### Check 3 — Issue type

Verify `fields.issuetype.name` is incident-appropriate.

| Issue type | Action |
|---|---|
| `Bug`, `Incident`, `Problem`, `Service Request`, `Defect` | ✓ Proceed |
| `Story`, `Epic`, `Task`, `Sub-task`, `Feature`, `Improvement` | ⚠️ Warn |
| `Security Vulnerability` | ⚠️ Warn — extra care required |

**If non-incident type:**
```
⚠️ UNEXPECTED ISSUE TYPE

  Ticket    : <TICKET-ID>
  Type      : <issuetype.name>

  This agent is optimised for Bug / Incident tickets.
  Processing a "<type>" ticket may not produce a meaningful RCA or fix.

  Proceed? [Y / N]
```

**If type is `Security Vulnerability`:**
```
⚠️ SECURITY VULNERABILITY TICKET

  Ticket : <TICKET-ID>

  Security tickets may contain sensitive disclosure information.
  The RCA document and PR body will be treated with elevated sensitivity:
    • RCA will NOT be published to the knowledgebase without explicit confirmation.
    • PR will be created as a DRAFT.
    • Labels will include "security" and "do-not-merge-without-review".

  Proceed with these restrictions? [Y / N]
```

---

### Check 4 — Duplicate or linked tickets

Scan `fields.issuelinks` for `duplicates` or `is duplicated by` relationships.

**If duplicates found:**
```
⚠️ DUPLICATE TICKET DETECTED

  Ticket    : <TICKET-ID>
  Duplicate : <linked ticket ID> — "<linked ticket summary>"

  This ticket may already be tracked under a different ID.
  Investigate the linked ticket first before creating a separate fix.

  Proceed with <TICKET-ID> anyway? [Y / N]
```
