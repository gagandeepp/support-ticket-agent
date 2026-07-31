## Step 5 — External Metrics Push — MongoDB

Push the per-run metrics document to MongoDB after the local files have been written. This step runs only when `metrics.mongodb.enabled = true` in config.

This step is **non-blocking** — if the push fails for any reason (credential missing, network unreachable, write permission denied), log the failure to the audit trail, output a warning block, and continue. Never STOP the workflow for a metrics push failure.

---

### Pre-condition

Only execute this module if both conditions are true:
1. `metrics.mongodb.enabled = true` in `support-ticket-resolver-config.json`
2. The per-run metrics file `.claude/metrics/run-<TICKET_ID>-<RUN_TS>.json` was written successfully in Step 4 of `@modules/09-agent-metrics.md`

If either condition is false, skip silently (no warning output).

---

### Step 5a — Pre-flight: credential check

Read the env var named by `metrics.mongodb.connectionStringEnvVar`. If unset or empty:

```
⚠️ MONGODB METRICS PUSH SKIPPED — credential missing

  Config key : metrics.mongodb.connectionStringEnvVar = "<varName>"
  Env var    : <varName> is not set or empty.

  To enable metrics push, set the variable:
    export <varName>="mongodb://user:password@host:27017/?authSource=admin"

  Local metrics file is still available at:
    .claude/metrics/run-<TICKET_ID>-<RUN_TS>.json
```

Log `METRICS_PUSH_MONGODB_FAILED` with `error: "credential_missing"` and return. Do not attempt the connection.

---

### Step 5b — Pre-flight: connectivity check

Before inserting, verify the connection with a ping:

```bash
METRICS_MONGODB_URL="$(printenv <connectionStringEnvVar>)"
mongosh "${METRICS_MONGODB_URL}" \
  --quiet \
  --eval "
    const pong = db.runCommand({ ping: 1 });
    print(JSON.stringify(pong));
  "
```

Interpret the result:

| Result | Action |
|---|---|
| `{ "ok": 1 }` | Proceed to Step 5c |
| `MongoServerError: Authentication failed` | Log failure, output warning, return |
| `MongoNetworkError` / connection timeout | Log failure, output warning, return |
| Any non-zero exit code | Log failure, output warning, return |

On any connectivity failure:

```
⚠️ MONGODB METRICS PUSH FAILED — connection error

  Host  : <host extracted from connection string — omit credentials>
  Error : <mongosh error message>

  Local metrics file is still available at:
    .claude/metrics/run-<TICKET_ID>-<RUN_TS>.json

  To debug: mongosh "<connectionStringEnvVar value>" --eval "db.runCommand({ping:1})"
```

Log `METRICS_PUSH_MONGODB_FAILED` with the error and return.

---

### Step 5c — Insert the metrics document

Write a temporary JS script to avoid shell quoting issues with the embedded JSON:

```bash
METRICS_MONGODB_URL="$(printenv <connectionStringEnvVar>)"
DATABASE="<metrics.mongodb.database>"
COLLECTION="<metrics.mongodb.collection>"
METRICS_FILE=".claude/metrics/run-<TICKET_ID>-<RUN_TS>.json"
TMP_JS=$(mktemp /tmp/metrics-push-XXXXXX.js)

cat > "$TMP_JS" << JSEOF
use('${DATABASE}');
const doc = $(cat "${METRICS_FILE}");
doc._id = doc.runId;
doc.schemaVersion = 1;
const result = db['${COLLECTION}'].insertOne(doc);
print(JSON.stringify({ insertedId: result.insertedId.toString(), ok: 1 }));
JSEOF

mongosh "${METRICS_MONGODB_URL}" --quiet "$TMP_JS"
EXIT_CODE=$?
rm -f "$TMP_JS"
```

**`doc._id = doc.runId`** — sets `_id` to the `runId` string (e.g. `APP-4821-2024-01-15T14-32-07Z`). This makes inserts idempotent: re-pushing the same run produces a duplicate-key error (see below) rather than a second document.

**`doc.schemaVersion = 1`** — forward-compatibility marker for future query migrations.

---

### Step 5d — Interpret the result

**Success** — mongosh exits 0 and output contains `"ok": 1`:

```
✅ METRICS PUSHED — MongoDB

  Database   : <database>
  Collection : <collection>
  Document   : _id = <runId>
  Inserted   : <insertedId>
```

Log `METRICS_PUSH_MONGODB_SUCCESS` to the audit trail with `insertedId` and `collection`.

---

**Duplicate key** — output contains `E11000 duplicate key error`:

```
ℹ️ MONGODB METRICS — duplicate run skipped

  Document _id = <runId> already exists in <database>.<collection>.
  This run's metrics were already pushed (possibly from a prior retry).
  No action needed.
```

Log `METRICS_PUSH_MONGODB_FAILED` with `error: "duplicate_key"`. This is not an error requiring attention — treat it as a no-op success.

---

**Write permission denied** — output contains `not authorized` or `Unauthorized`:

```
⚠️ MONGODB METRICS PUSH FAILED — write permission denied

  Database   : <database>
  Collection : <collection>
  Error      : <mongosh error message>

  The MongoDB user named in <connectionStringEnvVar> needs insertOne
  permission on <database>.<collection>.

  Minimum role: { role: "readWrite", db: "<database>" }
  Or a custom role: { privileges: [{ resource: { db: "<database>",
    collection: "<collection>" }, actions: ["insert"] }] }

  Local metrics file is still available at:
    .claude/metrics/run-<TICKET_ID>-<RUN_TS>.json
```

Log `METRICS_PUSH_MONGODB_FAILED` with the error and return.

---

**Any other non-zero exit or unexpected output:**

```
⚠️ MONGODB METRICS PUSH FAILED

  Exit code : <N>
  Output    : <first 200 chars of mongosh output>

  Local metrics file is still available at:
    .claude/metrics/run-<TICKET_ID>-<RUN_TS>.json
```

Log `METRICS_PUSH_MONGODB_FAILED` with the error and return.

---

### Document shape in MongoDB

After a successful insert, the document in `<database>.<collection>` looks like:

```json
{
  "_id": "APP-4821-2024-01-15T14-32-07Z",
  "schemaVersion": 1,
  "runId": "APP-4821-2024-01-15T14-32-07Z",
  "ticketId": "APP-4821",
  "runTs": "2024-01-15T14:32:07Z",
  "outcome": "completed",
  "durationSeconds": 342,
  "phases": { ... },
  "tokenUsage": {
    "budget": 200000,
    "totalSpent": 45234,
    "left": 154766,
    "byPhase": { ... },
    "tokenBudgetLowFired": false
  },
  "apiCalls": { ... },
  "guardrailsTriggered": [...],
  "circuitBreakers": [],
  "checkpointRestored": false,
  "prUrl": "https://github.com/acme/payments/pull/512",
  "rcaUrl": "https://acme.atlassian.net/wiki/...",
  "abortReason": null
}
```

---

### Useful queries after data accumulates

```js
// Success rate
db.agent_runs.aggregate([
  { $group: { _id: "$outcome", count: { $sum: 1 } } }
])

// Average resolution time for completed runs
db.agent_runs.aggregate([
  { $match: { outcome: "completed" } },
  { $group: { _id: null, avgSeconds: { $avg: "$durationSeconds" } } }
])

// Slowest 5 runs
db.agent_runs.find({ outcome: "completed" })
  .sort({ durationSeconds: -1 })
  .limit(5)
  .project({ ticketId: 1, durationSeconds: 1, prUrl: 1 })

// Token usage trend (last 30 runs)
db.agent_runs.find()
  .sort({ runTs: -1 })
  .limit(30)
  .project({ ticketId: 1, runTs: 1, "tokenUsage.totalSpent": 1, "tokenUsage.left": 1 })

// Most expensive phases by average token delta
db.agent_runs.aggregate([
  { $project: {
    phase1Tokens:  "$tokenUsage.byPhase.1",
    phase3Tokens:  "$tokenUsage.byPhase.3",
    phase3bTokens: "$tokenUsage.byPhase.3b",
    phase4Tokens:  "$tokenUsage.byPhase.4",
    phase5Tokens:  "$tokenUsage.byPhase.5"
  }},
  { $group: {
    _id: null,
    avgPhase1:  { $avg: "$phase1Tokens" },
    avgPhase3:  { $avg: "$phase3Tokens" },
    avgPhase3b: { $avg: "$phase3bTokens" },
    avgPhase4:  { $avg: "$phase4Tokens" },
    avgPhase5:  { $avg: "$phase5Tokens" }
  }}
])

// Runs where token budget dropped low
db.agent_runs.find({ "tokenUsage.tokenBudgetLowFired": true })
  .project({ ticketId: 1, "tokenUsage.totalSpent": 1, "tokenUsage.left": 1, outcome: 1 })

// Guardrails triggered most often
db.agent_runs.aggregate([
  { $unwind: "$guardrailsTriggered" },
  { $group: { _id: "$guardrailsTriggered", count: { $sum: 1 } } },
  { $sort: { count: -1 } }
])

// Integrations with highest error rates
db.agent_runs.aggregate([
  { $project: {
    jiraErrors:     "$apiCalls.jira.errors",
    nrErrors:       "$apiCalls.newRelic.errors",
    githubErrors:   "$apiCalls.github.errors"
  }},
  { $group: {
    _id: null,
    totalJiraErrors:   { $sum: "$jiraErrors" },
    totalNrErrors:     { $sum: "$nrErrors" },
    totalGithubErrors: { $sum: "$githubErrors" }
  }}
])
```

---

### Suggested indexes

Create these once after the collection is provisioned:

```js
db.agent_runs.createIndex({ outcome: 1, runTs: -1 })
db.agent_runs.createIndex({ ticketId: 1 })
db.agent_runs.createIndex({ "tokenUsage.totalSpent": -1 })
db.agent_runs.createIndex({ runTs: -1 })
```
