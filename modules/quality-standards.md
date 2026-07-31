## Quality Standards

---

### Code change rules

- Every code change must be justified by log evidence or code analysis — no speculative fixes.
- PRs must be minimal and focused — do not bundle unrelated improvements.
- All secrets must be read from environment variables — never hardcode credentials.
- Database connections must be read-only; refuse to connect if `readOnly` is not `true` in the connection config.
- Announce phase transitions clearly so the user can follow progress.

---

### RCA completeness requirements

Run this check before publishing the RCA (Phase 8 Step 1). All sections must be present and non-empty.

| Section | Minimum content required |
|---|---|
| Root Cause | At least one sentence naming the specific failure. Must not be only a category label (e.g., "Code Defect") or "unknown." |
| Contributing Factors | At least one factor listed, or explicit statement "none identified." |
| Timeline | At least two timestamped events (incident start + at least one causal event). |
| Blast Radius | States which services, users, or data were affected and to what extent. |
| Detection Gap | Explains specifically why this was not caught before production. |
| Fix Description | References the specific code change — file name, function, or configuration key. |
| Prevention | At least one concrete follow-up action (new test, alert rule, config change, or monitoring improvement). |

**If any section fails the check:**

```
⛔ RCA COMPLETENESS CHECK FAILED — CANNOT PUBLISH

  Missing or incomplete sections:
    • <section name>: <reason — e.g., "empty", "only contains category label", "only 1 timeline event">
    ...

  Complete these sections before publishing. Schema: modules/quality-standards.md
```

Do not publish until all sections pass.

---

### PR format requirements

**Title format:** `fix(<scope>): <imperative description> [<TICKET-ID>]`

Valid examples:
- `fix(payment-service): handle null response from StripeClient [APP-4821]`
- `fix(auth): revert session token expiry regression [APP-4900]`
- `fix(analytics): correct timezone offset in aggregation query [APP-5102]`

**Required PR body sections:**

| Section header | Required content |
|---|---|
| `## Summary` | What the fix does and why (2–5 sentences). |
| `## Root Cause` | One-liner from the RCA. |
| `## Changes` | Bullet list of files or functions modified. |
| `## Test Plan` | Step-by-step instructions for verifying the fix. At minimum: how to reproduce the original bug and confirm it is resolved. |
| `## RCA Link` | URL to the published RCA page, or "see attached `RCA-<ID>.md`" if publishing failed. |
| `## Related Ticket` | Full Jira ticket URL. |

**If any required section is missing:**

```
⛔ PR FORMAT CHECK FAILED — CANNOT CREATE PR

  Missing required sections:
    • ## Test Plan
    • ## RCA Link

  Add these sections to the PR body and retry.
```

---

### Placeholder and TODO detection

Before creating a PR or publishing an RCA, scan all content for markers that indicate incomplete work.

| Pattern | Category |
|---|---|
| `TODO`, `FIXME`, `HACK`, `XXX` (as standalone comment markers) | Incomplete work |
| `<placeholder>`, `<REPLACE>`, `<YOUR_VALUE>`, `<add ...>` | Template placeholder |
| `TBD`, `TBC`, `to be determined`, `to be confirmed` (outside code) | Deferred decision |
| `...` used as a content substitute between headings (not inside code blocks) | Truncation placeholder |
| A section heading followed immediately by the next heading with no body | Empty section |

**If any pattern is found:**

```
⛔ PLACEHOLDER CONTENT DETECTED — CANNOT PUBLISH / CREATE PR

  Found in: <RCA document | PR body>
  Matches:
    • <location>: "<matched text>"
    ...

  Resolve all placeholders before proceeding.
```

**Exception:** patterns inside fenced code blocks (` ``` `) are intentional and must not be flagged.

---

### Output style

- The RCA document must be understandable to both technical and non-technical readers. Avoid jargon without explanation.
- Never include raw credentials, internal hostnames, PII, or unredacted stack traces in outputs visible to stakeholders. Apply `@modules/guardrails/01-secret-pii-scrubber.md` before all external-facing output.
- Phase transition announcements must include the phase number and name so the user can orient themselves at a glance.
