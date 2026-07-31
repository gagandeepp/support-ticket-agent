## GUARDRAIL — Secret & PII Scrubber

Apply this guardrail before writing any content to GitHub (commit messages, PR bodies), Jira (comments), Confluence (pages), or SharePoint (files). Also apply to DB query results before including them in the RCA.

---

### Secret patterns to detect

Scan all outbound content against these patterns. Replace any match with `[REDACTED:<TYPE>]`.

| Type | Pattern | Example replacement |
|---|---|---|
| AWS access key | `AKIA[A-Z0-9]{16}` or `ASIA[A-Z0-9]{16}` | `[REDACTED:AWS_KEY]` |
| GitHub token | `ghp_[A-Za-z0-9]{36}` or `github_pat_[A-Za-z0-9_]{82}` | `[REDACTED:GITHUB_TOKEN]` |
| New Relic key | `NRAK-[A-Z0-9]{27}` | `[REDACTED:NR_KEY]` |
| Bearer token | `Bearer [A-Za-z0-9\-._~+/]+=*` | `[REDACTED:BEARER_TOKEN]` |
| Basic auth | `Basic [A-Za-z0-9+/=]{20,}` | `[REDACTED:BASIC_AUTH]` |
| DB connection string | `(postgresql\|mysql\|mongodb\|Server=)[^@\s]*@` | `[REDACTED:CONNECTION_STRING]` |
| Private key block | `-----BEGIN (RSA \|EC \|OPENSSH )?PRIVATE KEY-----` | `[REDACTED:PRIVATE_KEY]` |
| Generic API key param | `(api_key\|apikey\|api-key\|token\|secret)=[A-Za-z0-9\-_]{16,}` (case-insensitive) | `[REDACTED:API_KEY]` |

### PII patterns to detect

| Type | Pattern | Example replacement |
|---|---|---|
| Email address | `[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}` | `[REDACTED:EMAIL]` |
| Credit card | `\b(?:\d{4}[\s\-]?){3}\d{4}\b` | `[REDACTED:CARD_NUMBER]` |
| US SSN | `\b\d{3}-\d{2}-\d{4}\b` | `[REDACTED:SSN]` |
| DB column names (in query results) — mask values in columns named: `password`, `passwd`, `secret`, `token`, `api_key`, `ssn`, `card_number`, `credit_card`, `cvv`, `dob`, `date_of_birth` | column value | `[REDACTED:SENSITIVE_FIELD]` |

### Scrubbing rules

1. **Scan before write** — run the scrubber on the full text of every artifact before it leaves the agent: PR body, commit message, Jira comment, RCA markdown, Confluence page body, SharePoint file content.
2. **Never log secret values** — if a secret is detected, record in the audit trail only that a redaction occurred and in which artifact, not the secret value itself.
3. **Hard stop on secret in outbound content** — if a pattern matches and the content is about to be pushed to GitHub or published externally:

```
⛔ SECRET DETECTED IN OUTBOUND CONTENT — PUBLISH BLOCKED

  Artifact  : <PR body | Jira comment | RCA doc | Confluence page>
  Pattern   : <type of secret detected>
  Location  : line <N> of the artifact (context: "...surrounding text...")

  The secret has been redacted as [REDACTED:<TYPE>] in the artifact.

  Action required:
    A) Review the redacted artifact, confirm the redaction is correct, and proceed.
    B) Edit the artifact manually to remove the sensitive data before proceeding.

  The original unredacted content will NOT be published under any circumstance.
```

**STOP. Show the user the redacted artifact for review before re-attempting the publish.**

4. **DB result truncation** — never include more than 10 rows of raw DB query results in the RCA. Summarise patterns instead of listing raw values.
5. **Log content summarisation** — raw log lines from observability platforms must be paraphrased in the RCA, not copy-pasted verbatim, to avoid accidentally including session tokens or sensitive payloads that appear in log messages.
