## GUARDRAIL — Production Environment Protection

Apply whenever the ticket's environment field is `production`, `prod`, `prd`, or any case-insensitive variant. Tighten all confirmation requirements and enforce additional safeguards.

---

### Detection

If Phase 1 extracts `Environment: production` (or equivalent), set a **PRODUCTION FLAG** for the remainder of the workflow. This flag triggers additional confirmation gates at every phase that writes, queries, or publishes.

### Mandatory confirmation gates (when PRODUCTION FLAG is set)

Each gate below requires an explicit Y/N response before the action proceeds. A prior Y for one gate does NOT imply Y for subsequent gates.

| Gate | Phase | Action requiring confirmation |
|---|---|---|
| **G1** | 3 | Querying observability platforms for a production service |
| **G2** | 3 | Querying any database connection (even if environment is in `allowedEnvironments`) |
| **G3** | 5 | Implementing the fix (in addition to the standard Phase 5 Step 2 confirmation) |
| **G4** | 7 | Pushing the fix branch and creating the PR |
| **G5** | 8 | Publishing the RCA to the knowledgebase |

### Gate prompt template

```
⚠️ PRODUCTION ENVIRONMENT — CONFIRMATION REQUIRED (Gate <N>)

  Ticket    : <TICKET-ID>
  Environment: PRODUCTION
  Action    : <description of what is about to happen>

  This action involves a production system. Mistakes may have immediate user impact.

  Proceed? [Y / N]
```

**STOP at each gate. Do not proceed until the user explicitly responds Y.**

### Additional production-specific rules

**Database queries:**
- Even if `production` is listed in `database.allowedEnvironments`, display this warning before any query:
  ```
  ⚠️ You are about to query a PRODUCTION database.
     Query: <exact query string>
     Connection: <connection.label>
  ```
  Require confirmation before executing.

**Fix branch base:**
- If the ticket's environment is production, verify the fix branch base is `github.defaultBaseBranch` (typically `main` or `master`).
- If the base branch is anything else (e.g., `hotfix/*`, `release/*`), surface an additional warning: `⚠️ Fix branch is targeting a non-default base. Confirm this is intentional.`

**PR labels:**
- Automatically add the label `production-incident` to any PR raised for a production ticket, in addition to the standard labels.

**Never allowed for production tickets:**
- Force-pushing any branch.
- Skipping test execution in Phase 5 Step 3f.
- Using `--no-verify` on any git operation.
