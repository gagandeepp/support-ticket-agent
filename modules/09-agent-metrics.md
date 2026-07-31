## PHASE 8 — Agent Self-Metrics

Collect and persist a structured metrics record for this run immediately after `WORKFLOW_COMPLETED` or `WORKFLOW_ABORTED` is logged to the audit trail. This step runs even when the workflow fails — the metrics record documents the failure for cross-run analysis.

---

### When to execute

Trigger this module at the end of Phase 8, after:
- `WORKFLOW_COMPLETED` is logged (successful run), **or**
- `WORKFLOW_ABORTED` is logged (aborted/failed run)

Do not skip metrics collection even if knowledgebase publish or notifications failed.

---

### Step 1 — Derive per-phase timing

Scan the in-memory audit trail for `PHASE_STARTED` / `PHASE_COMPLETED` event pairs. For each phase, compute:

```
durationSeconds = PHASE_COMPLETED.ts - PHASE_STARTED.ts  (in seconds)
tokenDelta      = PHASE_COMPLETED.tokens.spent - PHASE_STARTED.tokens.spent
```

If a phase has no `PHASE_COMPLETED` event (it was in-progress when the run aborted), record `status: "interrupted"` and omit `durationSeconds`.

If a phase was skipped because a valid checkpoint existed, record `status: "skipped_checkpoint"` and `durationSeconds: 0`.

---

### Step 2 — Derive API call statistics

Scan the in-memory audit trail for events that represent external API calls and tally per integration:

| Integration | Count events | Error events |
|---|---|---|
| `jira` | `TICKET_FETCHED` | `PREFLIGHT_FAILED` (integration=jira) |
| `github` | `BRANCH_PUSHED`, `PR_CREATED`, `REVIEWER_ASSIGNED` | `REPO_RESOLUTION_FAILED` |
| `newRelic` | `OBS_QUERY_COMPLETED` (platform=new_relic) | `OBS_QUERY_FAILED` (platform=new_relic) |
| `azureAppInsights` | `OBS_QUERY_COMPLETED` (platform=azure_app_insights) | `OBS_QUERY_FAILED` (platform=azure_app_insights) |
| `cloudwatch` | `OBS_QUERY_COMPLETED` (platform=cloudwatch) | `OBS_QUERY_FAILED` (platform=cloudwatch) |
| `confluence` | `KB_SEARCH_COMPLETED` (platform=confluence), `RCA_PUBLISHED` (platform=confluence) | `RCA_PUBLISH_FAILED` (platform=confluence) |
| `sharepoint` | `KB_SEARCH_COMPLETED` (platform=sharepoint), `RCA_PUBLISHED` (platform=sharepoint) | `RCA_PUBLISH_FAILED` (platform=sharepoint) |

Retries are counted from the difference between call-start attempts and completions per integration (estimate from audit events if exact retry counts were not individually logged).

---

### Step 3 — Build the run metrics document

Construct the following JSON object:

```json
{
  "runId": "<TICKET_ID>-<RUN_TS>",
  "ticketId": "<TICKET_ID>",
  "runTs": "<RUN_TS>",
  "outcome": "<completed | aborted | failed>",
  "durationSeconds": <total wall-clock seconds from PREFLIGHT_STARTED to WORKFLOW_COMPLETED/ABORTED>,
  "phases": {
    "0":  { "status": "<complete | skipped_checkpoint | interrupted>", "startTs": "<ISO>", "endTs": "<ISO>", "durationSeconds": <N>, "tokenDelta": <N> },
    "1":  { "status": "...", "startTs": "...", "endTs": "...", "durationSeconds": <N>, "tokenDelta": <N> },
    "2":  { "status": "...", "startTs": "...", "endTs": "...", "durationSeconds": <N>, "tokenDelta": <N> },
    "3":  { "status": "...", "startTs": "...", "endTs": "...", "durationSeconds": <N>, "tokenDelta": <N> },
    "3b": { "status": "...", "startTs": "...", "endTs": "...", "durationSeconds": <N>, "tokenDelta": <N> },
    "4":  { "status": "...", "startTs": "...", "endTs": "...", "durationSeconds": <N>, "tokenDelta": <N> },
    "5":  { "status": "...", "startTs": "...", "endTs": "...", "durationSeconds": <N>, "tokenDelta": <N> },
    "6":  { "status": "...", "startTs": "...", "endTs": "...", "durationSeconds": <N>, "tokenDelta": <N> },
    "7":  { "status": "...", "startTs": "...", "endTs": "...", "durationSeconds": <N>, "tokenDelta": <N> },
    "8":  { "status": "...", "startTs": "...", "endTs": "...", "durationSeconds": <N>, "tokenDelta": <N> }
  },
  "tokenUsage": {
    "budget": <N>,
    "totalSpent": <N>,
    "left": <N>,
    "byPhase": {
      "0": <tokenDelta>, "1": <tokenDelta>, "2": <tokenDelta>,
      "3": <tokenDelta>, "3b": <tokenDelta>, "4": <tokenDelta>,
      "5": <tokenDelta>, "6": <tokenDelta>, "7": <tokenDelta>, "8": <tokenDelta>
    },
    "tokenBudgetLowFired": <true | false>
  },
  "apiCalls": {
    "jira":              { "calls": <N>, "errors": <N>, "retries": <N> },
    "github":            { "calls": <N>, "errors": <N>, "retries": <N> },
    "newRelic":          { "calls": <N>, "errors": <N>, "retries": <N> },
    "azureAppInsights":  { "calls": <N>, "errors": <N>, "retries": <N> },
    "cloudwatch":        { "calls": <N>, "errors": <N>, "retries": <N> },
    "confluence":        { "calls": <N>, "errors": <N>, "retries": <N> },
    "sharepoint":        { "calls": <N>, "errors": <N>, "retries": <N> }
  },
  "guardrailsTriggered": ["<guardrail-id>", ...],
  "circuitBreakers":     ["<service>", ...],
  "checkpointRestored":  <true | false>,
  "prUrl":               "<URL | null>",
  "rcaUrl":              "<URL | null>",
  "abortReason":         "<reason | null>"
}
```

Omit integrations from `apiCalls` that are disabled in config (`calls: 0, errors: 0, retries: 0` is valid for enabled-but-unused integrations).

---

### Step 4 — Write the per-run metrics file

```bash
mkdir -p .claude/metrics
```

Write the metrics JSON to:

```
.claude/metrics/run-<TICKET_ID>-<RUN_TS>.json
```

Use the Write tool. Pretty-print with 2-space indentation.

If the write fails, output:
```
⚠️ METRICS WRITE FAILED — .claude/metrics/run-<TICKET_ID>-<RUN_TS>.json
  Error: <OS error>
  Metrics will not be persisted for this run.
```
Continue — do not STOP.

---

### Step 5 — Append to the metrics index

After writing the per-run file, append a single-line JSON record to the index:

```bash
echo '<index-record>' >> .claude/metrics/index.jsonl
```

Index record schema:

```json
{ "ts": "<RUN_TS>", "ticketId": "<TICKET_ID>", "outcome": "<completed|aborted|failed>", "durationSeconds": <N>, "tokenSpent": <N>, "tokenLeft": <N>, "prUrl": "<URL|null>", "rcaUrl": "<URL|null>", "abortReason": "<reason|null>", "runFile": "run-<TICKET_ID>-<RUN_TS>.json" }
```

This index enables `jq`-based analytics across runs without parsing individual run files:

```bash
# Success rate
jq -s 'group_by(.outcome) | map({outcome: .[0].outcome, count: length})' .claude/metrics/index.jsonl

# Average resolution time for completed runs
jq -s '[.[] | select(.outcome=="completed") | .durationSeconds] | add / length' .claude/metrics/index.jsonl

# Slowest runs
jq -s 'sort_by(-.durationSeconds) | .[:5] | .[] | {ticketId, durationSeconds, outcome}' .claude/metrics/index.jsonl

# Token usage trend
jq '{ticketId, tokenSpent, tokenLeft, ts}' .claude/metrics/index.jsonl
```

If the index append fails, output the same `⚠️ METRICS WRITE FAILED` warning scoped to `index.jsonl` and continue.

---

### Step 5 — Push metrics to MongoDB

@modules/10-metrics-push-mongodb.md

---

### Step 6 — Log METRICS_WRITTEN to audit trail

After both writes succeed, emit to the audit trail:

```json
{
  "phase": "8",
  "event": "METRICS_WRITTEN",
  "detail": {
    "path": ".claude/metrics/run-<TICKET_ID>-<RUN_TS>.json",
    "indexPath": ".claude/metrics/index.jsonl"
  },
  "ts": "<ISO 8601>",
  "tokens": { "budget": <N>, "spent": <N>, "left": <N> }
}
```
