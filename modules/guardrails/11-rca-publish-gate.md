## GUARDRAIL — RCA Publish Gate

Apply in Phase 8 Step 1 before publishing the RCA document to any knowledgebase (Confluence or SharePoint). Prevent premature, incomplete, or sensitive documents from being published.

---

### Check 1 — RCA completeness

Before publishing, verify the RCA document contains all mandatory sections.

**Required sections (case-insensitive heading match):**
```
Timeline
Root Cause
Impact
Fix Summary
Prevention
```

**If any section is missing:**
```
⚠️ INCOMPLETE RCA DOCUMENT

  Missing sections:
    • <section name>
    • <section name>

  Publishing an incomplete RCA may mislead future readers.

  Options:
    A) Generate missing sections — agent will draft placeholder content.
    B) Publish as-is — I accept the incomplete document.
    C) Skip publish — do not publish to knowledgebase; attach to PR only.
```

**STOP. Wait for user response.**

---

### Check 2 — Security ticket publish restriction

If the ticket type is `Security Vulnerability` (detected in guardrail `06-ticket-validity-check.md`), block publication unless explicitly overridden.

```
⛔ SECURITY TICKET — PUBLISH BLOCKED BY DEFAULT

  Ticket : <TICKET-ID>
  Type   : Security Vulnerability

  RCA documents for security tickets may contain sensitive disclosure details.
  Publishing to a shared knowledgebase may expose vulnerability details before
  the fix is deployed or customers are notified.

  Options:
    A) Confirm publish — I have verified the fix is deployed and it is safe to disclose.
    B) Skip publish — attach RCA to PR as a draft; I will publish manually.
```

**STOP. Wait for user response. Default is B.**

---

### Check 3 — Secret/PII scrub before publish

Run the full secret/PII scrubber (`@modules/guardrails/01-secret-pii-scrubber.md`) over the RCA document body before any publish call.

**If secrets or PII are detected:**
```
⛔ SECRETS / PII DETECTED IN RCA DOCUMENT — PUBLISH BLOCKED

  Findings:
    Line <N> : <pattern type> — <redacted preview>

  The document has NOT been published.

  Options:
    A) Auto-redact — replace detections with [REDACTED] and re-check.
    B) Abort publish — do not publish; I will clean the document manually.
```

**STOP on detection. Only proceed after A or the document is clean.**

---

### Check 4 — Production environment gate

If the PRODUCTION FLAG is set (from `@modules/guardrails/03-production-environment-gates.md`), apply Gate G5 before publishing:

```
⚠️ PRODUCTION ENVIRONMENT — CONFIRMATION REQUIRED (Gate G5)

  Ticket     : <TICKET-ID>
  Environment: PRODUCTION
  Action     : Publishing RCA to knowledgebase (<platform>)

  This action involves a production system. Mistakes may have immediate user impact.

  Proceed? [Y / N]
```

**STOP. Wait for explicit Y.**

---

### Check 5 — Knowledgebase target verification

Before publishing, verify the target space/site exists and the agent has write permission.

**Confluence:**
```
GET <confluence.baseUrl>/rest/api/space/<space.key>
```
Expect `200`. If `404` or `403`:
```
⚠️ CONFLUENCE TARGET UNAVAILABLE

  Space : <space.key>
  Error : <404 Not Found | 403 Forbidden>

  Options:
    A) Enter a different space key.
    B) Skip publish — attach RCA to PR instead.
```

**SharePoint:**
```
GET https://graph.microsoft.com/v1.0/sites/<siteId>/drives
Authorization: Bearer <token>
```
Expect `200`. If not:
```
⚠️ SHAREPOINT TARGET UNAVAILABLE

  Site  : <siteId>
  Error : <status code>

  Options:
    A) Skip publish — attach RCA to PR instead.
```

**STOP in both cases. Wait for user response.**

---

### Post-publish audit

After successful publication, log to the audit trail:
```
PUBLISHED_RCA: {
  "ticket": "<TICKET-ID>",
  "platform": "<confluence | sharepoint>",
  "url": "<published document URL>",
  "timestamp": "<ISO 8601>",
  "scrubChecks": "passed",
  "securityTicket": <true | false>
}
```
