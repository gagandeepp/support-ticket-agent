## GUARDRAIL — Reviewer Identity Verification

Apply in Phase 7 Step 1 before adding any reviewer to the pull request. Prevent assigning the wrong person or a machine account as a PR reviewer.

---

### Check 1 — GitHub handle resolution confidence

Phase 7 resolves the Jira assignee to a GitHub handle via three tiers:

| Tier | Source | Confidence |
|---|---|---|
| T1 | `github.reviewerMap` config entry | High |
| T2 | GitHub user search by email (`/search/users?q=<email>+in:email`) | Medium |
| T3 | GitHub user search by display name (`/search/users?q=<name>+in:fullname`) | Low |

**If resolved via T2 or T3, verify before assigning:**

```
GET https://api.github.com/users/<handle>
Authorization: Bearer <GITHUB_TOKEN>
```

Confirm all three:
1. `type` == `"User"` (not `"Bot"` or `"Organization"`)
2. `login` matches search result (not a partial alias)
3. `email` field (if public) matches the Jira assignee's email

**If verification fails any check:**
```
⚠️ REVIEWER IDENTITY UNCERTAIN

  Jira assignee : <displayName> <email>
  Resolved to   : @<handle> (via <tier>)
  Failure       : <type is Bot | email mismatch | name ambiguous>

  Assigning the wrong reviewer silently routes review to an unintended person.

  Options:
    A) Use @<handle> anyway — I confirm this is correct.
    B) Enter correct handle — provide the GitHub username manually.
    C) Skip reviewer — create PR without a reviewer assigned.
```

**STOP. Wait for user response.**

---

### Check 2 — Bot / service account detection

Never assign bots, service accounts, or machine users as reviewers.

**Blocked account patterns:**
```
login contains: -bot, _bot, bot-, bot_, -ci, -service,
                -deploy, -automation, dependabot, renovate,
                github-actions, snyk-bot
type == "Bot"
```

**If the resolved handle matches any pattern:**
```
⚠️ RESOLVED HANDLE APPEARS TO BE A BOT / SERVICE ACCOUNT

  Resolved to : @<handle>
  Pattern     : <matched pattern>

  Bot accounts cannot review PRs meaningfully.
  Skipping reviewer assignment.
```

Log and continue without assigning. Do not stop — this is a non-fatal skip.

---

### Check 3 — Repository access verification

Before assigning, confirm the resolved user has at least `read` access to the repository (required for PR reviewer assignment on private repos).

```
GET https://api.github.com/repos/<org>/<repo>/collaborators/<handle>/permission
Authorization: Bearer <GITHUB_TOKEN>
```

**Acceptable permission levels:** `admin`, `maintain`, `write`, `triage`, `read`

**If `404` (user not a collaborator) or permission is `none`:**
```
⚠️ REVIEWER HAS NO REPOSITORY ACCESS

  Reviewer : @<handle>
  Repo     : <org>/<repo>
  Status   : Not a collaborator (or insufficient permission)

  GitHub will reject the reviewer assignment (422).

  Options:
    A) Skip reviewer — create PR without reviewer; add manually later.
    B) Enter different handle — provide an alternative GitHub username.
```

**STOP. Wait for user response.** (If user selects A, create PR and log non-fatal skip.)

---

### Check 4 — Self-review prevention

If the agent is running as a GitHub App or under a specific token, verify the resolved reviewer is not the same identity as the token owner. Self-review is meaningless and blocked by GitHub on protected branches.

```
GET https://api.github.com/user
Authorization: Bearer <GITHUB_TOKEN>
```

**If `login` from /user matches the resolved reviewer handle:**
```
ℹ️ SELF-REVIEW DETECTED

  The resolved reviewer (@<handle>) matches the token owner.
  Skipping reviewer assignment — PR author cannot review their own PR.
```

Log and continue. Non-fatal.
