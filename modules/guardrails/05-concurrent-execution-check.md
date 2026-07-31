## GUARDRAIL — Concurrent Execution & Duplicate PR Check

Apply at two points: end of Phase 1 (check for prior agent runs on this ticket) and start of Phase 7 (check for existing PRs before creating a new one).

---

### Check A — Prior agent activity on the Jira ticket (Phase 1)

After fetching the ticket, scan its comment history for evidence of a previous agent run.

**If `mcp.atlassian.enabled = true`:**
```
<toolPrefix>__getJiraIssue({
  issueIdOrKey: "<ticket-id>",
  fields: ["comment"]
})
```

**If `mcp.atlassian.enabled = false`:**
```
GET <jira.baseUrl>/rest/api/3/issue/<ticket-id>?fields=comment
```

Scan `fields.comment.comments[].body` for the string `support-ticket-resolver agent` or `fix/<ticket-id>`.

**If found:**
```
⚠️ PRIOR AGENT RUN DETECTED

  Ticket    : <TICKET-ID>
  Evidence  : Comment from "<author>" on <date> containing agent signature.
  Comment   : "<excerpt>"

  This ticket appears to have been processed by this agent previously.

  Options:
    A) Continue — re-investigate and potentially create a new fix branch.
    B) Abandon — the prior run may have already handled this ticket.
       Check the existing comment for a PR link before deciding.
```

**STOP. Wait for user response before continuing.**

---

### Check B — Existing fix branch or PR (Phase 7, before PR creation)

Before pushing the branch or creating the PR, check GitHub for conflicts.

**Step 1 — check for existing branch:**
```
GET https://api.github.com/repos/<org>/<repo>/branches/fix/<ticket-id>-<slug>
Authorization: Bearer <GITHUB_TOKEN>
```
If `200` returned, the branch already exists on the remote.

**Step 2 — check for existing open PR:**
```
GET https://api.github.com/repos/<org>/<repo>/pulls?state=open&head=<org>:fix/<ticket-id>
Authorization: Bearer <GITHUB_TOKEN>
```
Filter results where `head.ref` starts with `fix/<ticket-id>`.

**If either exists:**
```
⚠️ EXISTING BRANCH / PR DETECTED

  Ticket : <TICKET-ID>
  Found  :
    Branch : fix/<ticket-id>-<slug> already exists on remote
    PR     : #<number> — "<title>" (opened <date>, status: open)
             <PR URL>

  Options:
    A) Reuse — push to the existing branch and update the existing PR.
    B) New   — create a new branch fix/<ticket-id>-v2 and a new PR.
    C) Abort — do not create a branch or PR; link the existing PR to the Jira ticket instead.
```

**STOP. Wait for user response. Never force-push to an existing branch.**

---

### Force-push absolute prohibition

Regardless of user response, the agent must NEVER run:
```bash
git push --force
git push --force-with-lease
git push -f
```
If the only way to update an existing branch is a force-push, tell the user and ask them to handle it manually.
