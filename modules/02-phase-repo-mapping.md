### PHASE 2 — Service & Repository Mapping

For each service extracted in Phase 1, attempt resolution in order. Track the result of every step per service.

---

#### Step 1 — Config lookup (primary)

Read `github.serviceRepoMap` from `support-ticket-resolver-config.json`. Each entry has:
- `service` — the service name as it appears in tickets and observability
- `repo`    — the repository name within `github.org`
- `path`    — subdirectory within the repo (use `/` for the root; supports monorepos)

Match the Phase 1 service name against `serviceRepoMap[].service` (case-insensitive exact match first, then substring). Resolve the full repo path as `<github.org>/<repo>/<path>`.

**Result:** matched → record and skip to Step 3. Not matched → proceed to Step 2.

---

#### Step 2 — GitHub API search (fallback)

If a service has no entry in `serviceRepoMap`, search GitHub:
- `GET /search/repositories?q=<service>+org:<org>` — match on name, description, or topic tags.
- Try progressively broader queries if the first returns no results:
  1. Exact service name: `q=<service>+org:<org>`
  2. Service name without suffix (e.g., strip `-service`, `-api`, `-worker`): `q=<stem>+org:<org>`
  3. Keyword from the ticket summary: `q=<keyword>+org:<org>`

**Result:** one or more candidates found → proceed to Step 3 with candidates flagged as **unconfirmed**. Zero results → proceed to Step 3 marked as **unresolved**.

---

#### Step 3 — Repo inspection

For each resolved or candidate repo:
1. Verify the repo exists and is accessible: `GET /repos/{owner}/{repo}` — if this returns 404 or 403, mark as **inaccessible**.
2. Identify primary language and framework.
3. Fetch recent commits on the default branch (last 72 hours): `GET /repos/{owner}/{repo}/commits?since=<72h ago>`.
4. List open PRs that may be related: `GET /repos/{owner}/{repo}/pulls?state=open`.
5. Check for relevant config files at `path` (`docker-compose.yml`, `k8s/`, `helm/`, environment manifests).

**Result:** repo accessible and inspected → record as **confirmed**. 404/403/timeout → mark as **inaccessible**.

---

#### Step 3a — Stack-trace blame (apply when Phase 1 extracted file paths and line numbers)

For each `file:line` pair from the Phase 1 stack trace that belongs to this repo:

1. Fetch blame data via GitHub GraphQL:

```graphql
POST https://api.github.com/graphql
Authorization: Bearer <GITHUB_TOKEN>

query {
  repository(owner: "<org>", name: "<repo>") {
    object(expression: "HEAD") {
      ... on Commit {
        blame(path: "<file-path>") {
          ranges {
            startingLine
            endingLine
            commit {
              oid
              message
              committedDate
              author { name email }
            }
          }
        }
      }
    }
  }
}
```

If using the GitHub MCP, use the equivalent blame tool call with the resolved `owner`, `repo`, and `path`.

2. For each erroring line, extract the blame entry: `{ sha, author_email, committed_date, commit_message }`.
3. Flag as **regression candidate** if `committed_date` falls within the 72 hours before the error first-seen timestamp extracted in Phase 1.
4. Record results in the repo's inspection block for use in Phase 4 RCA.

If the GitHub blame API returns a 403 or the repo does not support GraphQL, skip Step 3a and note it as `blame: unavailable`.

---

#### Step 3b — Regression commit bisect (apply when Phase 1 identified an error first-seen timestamp)

Using the commit list from Step 3.3 and the first-error timestamp from Phase 1:

1. **Candidate window**: all commits whose `committed_date` falls in the range `[first_error_timestamp − 72 h, first_error_timestamp]`.
2. For each candidate commit, record:
   - `sha` (short 8-char prefix)
   - `committed_date`
   - `commit_message` (first line only)
   - `author_email`
   - `conventional_type`: extract the type prefix if the message follows conventional commits (`fix`, `feat`, `refactor`, `chore`, `perf`, `build`, `ci`); otherwise `unknown`
3. **Rank candidates** by proximity to the first-error timestamp (closest before = rank 1).
4. Mark rank-1 commit as **primary regression candidate**; remaining as **secondary candidates**.
5. If rank-1 is a merge commit (message starts with `Merge pull request` or `Merge branch`), also record the PR number from the message so Phase 4 can fetch the PR diff.

**Output for Step 3b:**

```
🔬 REGRESSION BISECT
  Candidate window: <start> → <first_error_timestamp>
  Candidates found: <N>

  Rank 1 (PRIMARY): <sha>  <date>  <message>  [<type>]  author: <email>
  Rank 2:           <sha>  <date>  <message>  [<type>]
  ...
  (if 0 candidates): No commits in the 72-hour window before first error.
```

If the first-error timestamp is unavailable from Phase 1 (ticket did not include timing information), skip Step 3b and note it as `bisect: no timestamp available`.

---

#### ⛔ Step 4 — Unresolvable service: STOP and prompt

If a service exits Step 3 with no confirmed repo (all of Step 1, Step 2, and Step 3 failed or returned nothing usable), **do not proceed to Phase 3**. STOP and output the following prompt to the user:

```
⛔ REPO MAPPING INCOMPLETE — CANNOT PROCEED

The following service(s) could not be mapped to a repository after exhausting
all resolution steps:

  Service: <service-name>
  Step 1 (config lookup):   ✗ No entry in github.serviceRepoMap
  Step 2 (GitHub search):   ✗ <reason: no results | candidates inaccessible | ambiguous>
  Step 3 (repo inspection): ✗ <reason: 404 | 403 | timeout | no candidates>

To continue, please provide one or more of the following:

  A) The exact GitHub repo name for "<service-name>" in org "<github.org>":
       repo name: _______________
       subdirectory (if monorepo, else leave blank): _______________

  B) A GitHub repo URL you want the agent to inspect:
       URL: _______________

  C) Confirmation that this service has no dedicated repo (e.g., serverless,
     third-party, or infra-only) and should be excluded from fix targeting.

Once you provide the information, I will:
  1. Resume Phase 2 with the supplied repo.
  2. Confirm the mapping with you (Step 3 inspection).
  3. Persist the confirmed mapping to github.serviceRepoMap in
     support-ticket-resolver-config.json (Step 6) so future tickets resolve automatically.
```

Wait for the user's response before continuing. Do not guess, infer, or proceed with an unresolved service.

---

#### Step 5 — Output (all services resolved)

Once every service has a confirmed repo, produce the **Service → Repo Map** block:

```
🗂️ SERVICE → REPO MAP
  <service>  →  <org>/<repo><path>  [<language>, <framework>]
               source: config | github-search (user-confirmed)
  ...

📝 NEW MAPPINGS TO PERSIST:
  { "service": "<name>", "repo": "<repo>", "path": "/" }
  ...

🔬 BLAME / REGRESSION ANALYSIS
  Stack-trace blame:
    <file>:<line>  →  <sha>  <date>  "<commit message>"  by <author>
                      regression candidate: yes | no
    ...
    blame: unavailable (403 or GraphQL not supported)  [if applicable]

  Regression bisect:
    Primary candidate: <sha>  <date>  "<message>"  [<type>]
    Secondary:         <sha>  <date>  "<message>"
    ...
    bisect: no timestamp available  [if applicable]
```

If there are any new mappings (source is `github-search` or `user-supplied`, not `config`), proceed to Step 6.

---

#### Step 6 — Persist new mappings to config

For every mapping whose source is **not** `config` (i.e., discovered via GitHub search or supplied by the user), write it back to `support-ticket-resolver-config.json` by appending it to `github.serviceRepoMap`.

**Write procedure:**

1. Read the current `support-ticket-resolver-config.json`.
2. For each new mapping `{ "service": "...", "repo": "...", "path": "..." }`:
   - Check that no existing entry in `github.serviceRepoMap` already has the same `service` value (case-insensitive). If a duplicate exists, skip (do not overwrite — the config entry is authoritative).
   - Append the new object to the `serviceRepoMap` array.
3. Write the updated config back to `support-ticket-resolver-config.json`.
4. Validate the written file against `support-ticket-resolver-config.schema.json` to confirm the write did not corrupt the config.

**Output after persisting:**

```
✅ MAPPINGS PERSISTED — support-ticket-resolver-config.json updated

  Added to github.serviceRepoMap:
    • <service>  →  <repo>  (path: <path>)
    ...

  Future runs will resolve these services from config (Step 1) without re-discovery.
```

If the write fails (file locked, permissions error, schema validation of the written file fails):

```
⚠️ MAPPING PERSISTENCE FAILED

  Could not write to support-ticket-resolver-config.json: <reason>

  The workflow will continue using the in-memory mappings for this run.
  Add the following entries to github.serviceRepoMap manually before the next run:

    { "service": "<name>", "repo": "<repo>", "path": "<path>" }
```

Do not STOP the workflow for a persistence failure — the mappings are already resolved in memory for this run. Log the failure and continue to Phase 3.
