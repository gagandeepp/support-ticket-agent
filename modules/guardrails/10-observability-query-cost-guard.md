## GUARDRAIL — Observability Query Cost Guard

Apply in Phase 3 before executing any query against New Relic, Azure Application Insights, or AWS CloudWatch. Prevent runaway query costs, scan windows that are too broad, and queries that could return multi-GB result sets.

---

### Thresholds by platform

| Platform | Max time window | Max result rows | Max scan size |
|---|---|---|---|
| New Relic (NRQL) | 7 days | 2 000 | N/A (managed by NR) |
| Azure App Insights (Kusto) | 30 days | 10 000 | 100 GB (estimated) |
| AWS CloudWatch Logs Insights | 7 days | 10 000 | 5 GB (estimated) |

---

### Check 1 — Time window validation

Extract the time window from the query before execution.

**New Relic (NRQL):**
- Parse `SINCE <duration> AGO` or `SINCE <timestamp> UNTIL <timestamp>`.
- If `SINCE` is absent, default window is 60 minutes — ✓ safe.

**Azure App Insights (Kusto):**
- Parse `| where timestamp > ago(<duration>)` or `starttime`/`endtime` parameters.

**AWS CloudWatch Logs Insights:**
- Inspect `startTime` and `endTime` parameters passed to `start-query`.

**If the time window exceeds the threshold:**
```
⚠️ QUERY TIME WINDOW TOO BROAD

  Platform    : <platform>
  Query window: <parsed duration>
  Maximum     : <threshold>

  A wider window increases query cost and latency significantly.

  Options:
    A) Narrow to <threshold> — agent will rewrite the query automatically.
    B) Proceed with original window — I accept the cost.
    C) Skip this query — continue without observability data from <platform>.
```

**STOP. Wait for user response. If A, rewrite the time clause and show the revised query.**

---

### Check 2 — Result row cap

Append a row limit to every query before execution:

| Platform | Clause to append if absent |
|---|---|
| New Relic | `LIMIT 2000` |
| Azure App Insights | `\| take 10000` |
| CloudWatch | `limit 10000` field in query params |

If a `LIMIT` / `take` / `limit` clause already exists and exceeds the maximum, reduce it and log:
```
ℹ️ Result limit reduced from <N> to <max> to cap query cost.
```

---

### Check 3 — Estimated scan size (Azure and CloudWatch only)

For Azure App Insights and CloudWatch, attempt to estimate the scan size before committing:

**Azure App Insights:**
- If the workspace supports `QueryPack` cost estimation, use it.
- Otherwise, default to advisory: if the time window is > 7 days, display a cost warning.

**AWS CloudWatch:**
- CloudWatch charges per GB scanned. Estimate = `(retention GB/day) × (window days)`.
- If `observability.cloudwatch.serviceLogGroupMap` provides a `retentionDays` hint, use it.
- If estimated scan > 5 GB:
  ```
  ⚠️ CLOUDWATCH SCAN ESTIMATED AT ~<N> GB

    Log group : <logGroupName>
    Window    : <duration>
    Estimated : ~<N> GB scanned (~$<cost> USD at standard pricing)

    Proceed? [Y / N]
  ```

---

### Check 4 — Concurrent query cap

Never run more than 3 observability queries in parallel within a single phase execution.

If Phase 3 would trigger > 3 platform+log group combinations simultaneously, queue them in batches of 3 and log:
```
ℹ️ Batching observability queries: <N> total, running 3 at a time.
```

---

### Failure handling

If a query times out or returns a rate-limit error (429 / quota exceeded):
```
⚠️ OBSERVABILITY QUERY FAILED

  Platform : <platform>
  Error    : <timeout | 429 rate limit | quota exceeded>

  Options:
    A) Retry with a narrower time window (<half the original window>).
    B) Skip — continue without data from <platform>.
```

**STOP. Wait for user response.** (If A, halve the time window and retry once. If it fails again, escalate to option B automatically.)
