### PHASE 7 — Pull Request Creation

> **Guardrail — Concurrent Execution (Check B):** Before pushing any branch or creating a PR, apply `@modules/guardrails/05-concurrent-execution-check.md` Check B to detect an existing fix branch or open PR for this ticket. Stop and present options if found.

> **Guardrail — Production Gate (G4):** If the PRODUCTION FLAG is set, apply Gate G4 from `@modules/guardrails/03-production-environment-gates.md` before pushing. Explicit Y required.

---

#### Step 1 — Resolve assignee from Jira

Fetch the ticket's assignee and map their Jira identity to a GitHub handle before creating the PR.

**If `mcp.atlassian.enabled = true`**:
```
<toolPrefix>__getJiraIssue({ issueIdOrKey: "<ticket-id>", fields: ["assignee"] })
```

**If `mcp.atlassian.enabled = false`** — Jira REST API via WebFetch:
```
GET <jira.baseUrl>/rest/api/3/issue/<ticket-id>?fields=assignee
Authorization: Basic base64(<jira.userEmail>:<JIRA_API_TOKEN>)
```

Extract from the response:
- `fields.assignee.displayName` — full name
- `fields.assignee.emailAddress` — email

   > **Guardrail — Reviewer Identity:** Before assigning any reviewer, apply `@modules/guardrails/09-reviewer-identity-verification.md` to verify the resolved handle is a real user (not a bot), has repo access, and is not the token owner.

**Map Jira email → GitHub handle:**

1. Search GitHub for the email address:
   ```
   GET https://api.github.com/search/users?q=<emailAddress>+in:email
   Authorization: Bearer <GITHUB_TOKEN>
   ```
2. If a single unambiguous result is returned, record `login` as the GitHub handle.
3. If no result or multiple results, fall back to searching by display name:
   ```
   GET https://api.github.com/search/users?q=<displayName>+in:fullname+org:<github.org>
   ```
4. If still unresolved, record the assignee as **unmapped** and skip reviewer assignment (do not block PR creation).

```
⚠️ ASSIGNEE UNMAPPED

  Jira assignee : <displayName> (<emailAddress>)
  GitHub search : no unambiguous match found in org <github.org>

  The PR will be created without a reviewer.
  To assign manually: GitHub PR → Reviewers → search for the user.
  To avoid this in future runs, the user can add a mapping in config
  (see github.reviewerMap below).
```

**Optional config shortcut — `github.reviewerMap`:**

If the config contains a `reviewerMap` array, check it first before hitting the GitHub search API:

```json
"github": {
  ...
  "reviewerMap": [
    { "jiraEmail": "alice@your-org.com", "githubHandle": "alice-gh" },
    { "jiraEmail": "bob@your-org.com",   "githubHandle": "bob-gh" }
  ]
}
```

Exact email match in `reviewerMap` takes priority over the GitHub search API.

---

#### Step 2 — Push branch and open PR

> **Guardrail — Secret/PII Scrubber:** Before composing the PR body, run the draft through `@modules/guardrails/01-secret-pii-scrubber.md`. Replace any detected secrets or PII with `[REDACTED]` before submitting to GitHub.

Choose the path based on `mcp.github.enabled`:

**If `mcp.github.enabled = true`** — use GitHub MCP tools (prefix from `mcp.github.toolPrefix`):
```
<toolPrefix>__push_files(...)
<toolPrefix>__create_pull_request({
  owner: "<org>", repo: "<repo>",
  title: "fix(<ticket-id>): <description>",
  head: "fix/<ticket-id>-<slug>",
  base: "<defaultBaseBranch>",
  body: "<PR body from template below>"
})
```

**If `mcp.github.enabled = false`** — GitHub REST API via WebFetch + Bash:
```bash
# Push branch
git push origin fix/<ticket-id>-<slug>

# Create PR
POST https://api.github.com/repos/<org>/<repo>/pulls
Authorization: Bearer <GITHUB_TOKEN>
{
  "title": "fix(<ticket-id>): <description>",
  "head": "fix/<ticket-id>-<slug>",
  "base": "<defaultBaseBranch>",
  "body": "<PR body>"
}

# Add labels
POST https://api.github.com/repos/<org>/<repo>/issues/<pr_number>/labels
{ "labels": ["bug", "automated-fix", "rca-attached", "<ticket-id>"] }
```

PR body template:

```markdown
## Summary
<1–2 sentence summary of the fix.>

## Jira Ticket
[<TICKET-ID>](<JIRA_BASE_URL>/browse/<TICKET-ID>)

## Root Cause (TL;DR)
<One sentence root cause.>

## Changes
- `<file>`: <description>

## RCA Document
The full Root Cause Analysis is included in this branch: [`RCA-<TICKET-ID>-<DATE>.md`](./RCA-<TICKET-ID>-<DATE>.md)

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests pass
- [ ] Manual verification steps: <steps>

## Checklist
- [ ] Minimal targeted change (no scope creep)
- [ ] No secrets or credentials committed
- [ ] Documentation updated
- [ ] Reviewers assigned
```

---

#### Step 3 — Assign reviewer

Using the GitHub handle resolved in Step 1 (skip if unmapped):

**If `mcp.github.enabled = true`**:
```
<toolPrefix>__request_reviewers({
  owner: "<org>", repo: "<repo>", pull_number: <pr_number>,
  reviewers: ["<githubHandle>"]
})
```

**If `mcp.github.enabled = false`**:
```
POST https://api.github.com/repos/<org>/<repo>/pulls/<pr_number>/requested_reviewers
Authorization: Bearer <GITHUB_TOKEN>
{ "reviewers": ["<githubHandle>"] }
```

If the reviewer request returns `422` (user not a collaborator or handle incorrect):
```
⚠️ REVIEWER ASSIGNMENT FAILED

  GitHub handle : <githubHandle>
  Error         : 422 — <message>

  Possible causes:
    • The user is not a collaborator on <org>/<repo>
    • The resolved handle is incorrect

  The PR was created successfully. Assign a reviewer manually or
  add a correct mapping to github.reviewerMap in config.
```

Continue — do not block on reviewer failure.

---

#### Step 4 — Comment on Jira ticket

Choose the path based on `mcp.atlassian.enabled`:

**If `mcp.atlassian.enabled = true`**:
```
<toolPrefix>__addCommentToJiraIssue({
  issueIdOrKey: "<ticket-id>",
  body: "PR raised: <PR URL>\nReviewer: <displayName> (@<githubHandle>)\nBranch: fix/<ticket-id>-<slug>\nStatus: Pending Review"
})
```

**If `mcp.atlassian.enabled = false`** — Jira REST API via WebFetch:
```
POST <jira.baseUrl>/rest/api/3/issue/<ticket-id>/comment
Authorization: Basic base64(<jira.userEmail>:<JIRA_API_TOKEN>)
Content-Type: application/json
{
  "body": {
    "type": "doc", "version": 1,
    "content": [{ "type": "paragraph", "content": [{ "type": "text",
      "text": "PR raised: <PR URL> | Reviewer: <displayName> (@<githubHandle>) | Branch: fix/<ticket-id>-<slug> | Status: Pending Review"
    }]}]
  }
}
```

---

   > **Guardrail — Audit Trail:** Log `BRANCH_PUSHED`, `PR_CREATED`, `REVIEWER_ASSIGNED` (or `REVIEWER_SKIPPED`), and `JIRA_COMMENT_ADDED` events via `@modules/guardrails/12-audit-trail.md`.

#### Step 5 — Output

```
🚀 PULL REQUEST
  URL      : <PR URL>
  Title    : <PR title>
  Branch   : fix/<ticket-id>-<slug> → <base branch>
  Labels   : [bug, automated-fix, rca-attached, <ticket-id>]
  Reviewer : <displayName> (@<githubHandle>) | ⚠️ unmapped — assigned manually
  Assignee source: reviewerMap (config) | GitHub search | unmapped
  PR created via : MCP (<toolPrefix>) | REST API
  Jira Updated   : Yes — comment added via MCP | REST API
```
