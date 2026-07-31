## GUARDRAIL — Database Query Safety

Apply in Phase 3 before executing any database query. Enforce read-only access, query complexity limits, and result volume caps regardless of environment.

---

### Check 1 — Read-only enforcement

Before running any query, verify the connection string or DSN points to a read-only replica or uses a read-only user.

**Detection heuristics (in order):**
1. Config `database.connections[].label` contains `readonly`, `read-only`, `replica`, or `ro` → ✓ likely safe.
2. Connection string user segment (e.g., `user=readonly_user`) → ✓ likely safe.
3. Neither condition met → treat as unknown.

**If connection is unknown or appears writable:**
```
⚠️ DATABASE CONNECTION MAY NOT BE READ-ONLY

  Connection : <connection.label>
  Host       : <host extracted from DSN, credentials redacted>

  This agent cannot verify the connection is read-only.
  Running a query on a writable connection carries risk of
  accidental writes if the query is malformed.

  Options:
    A) Proceed — I confirm this connection is read-only.
    B) Skip    — Do not query this database; continue without DB findings.
    C) Abort   — Stop the workflow.
```

**STOP. Wait for user response.**

---

### Check 2 — Disallowed statement detection

Scan the query string for non-SELECT statement keywords before execution.

**Blocked keywords (case-insensitive, whole-word match):**
```
INSERT  UPDATE  DELETE  DROP  TRUNCATE  ALTER  CREATE
REPLACE MERGE   EXEC    EXECUTE  CALL   GRANT  REVOKE
```

**If any blocked keyword is found:**
```
⛔ DISALLOWED SQL STATEMENT DETECTED

  Connection : <connection.label>
  Keyword    : <matched keyword>
  Query      : <redacted first 200 chars>

  This agent only runs SELECT queries.
  The query has NOT been executed.

  Options:
    A) Rewrite — I will revise the query to SELECT-only.
    B) Skip    — Do not query this database; continue without DB findings.
```

**STOP. Do not execute the query until the user responds.**

---

### Check 3 — Query complexity cap

Before execution, apply a complexity check to prevent runaway queries.

**Rules:**
- Prepend `LIMIT 100` (or engine equivalent) if no LIMIT / TOP / ROWNUM clause is present.
  - PostgreSQL / MySQL / SQLite: `LIMIT 100`
  - SQL Server: `SELECT TOP 100`
  - Oracle: `FETCH FIRST 100 ROWS ONLY` or `ROWNUM <= 100`
- If the query already has `LIMIT N` with N > 500, reduce to 500 and log a note:
  ```
  ℹ️ Result limit reduced from <N> to 500 to cap query cost.
  ```
- If the query contains a Cartesian product (JOIN without ON / WHERE) or `SELECT *` from a table with no WHERE clause, warn:
  ```
  ⚠️ POTENTIALLY EXPENSIVE QUERY

    Pattern  : <Cartesian join | unrestricted SELECT *>
    Estimated: Unknown row count

    Recommend adding a WHERE clause or explicit column list.
    Proceed with limit-capped query? [Y / N]
  ```

---

### Check 4 — Result volume cap and truncation

After query execution, enforce result volume limits before including results in the workflow.

| Metric | Limit | Action if exceeded |
|---|---|---|
| Rows returned | 100 | Truncate to 100, log count |
| Total result size | 50 KB | Truncate rows until under limit |
| Single cell value | 1 KB | Truncate cell value, append `…[truncated]` |

**Append to DB findings block when truncated:**
```
ℹ️ Results truncated: <N> rows returned, showing first <M>.
   Full results available by re-running with a narrower WHERE clause.
```

---

### Check 5 — Credentials in query results (secret scrubber handoff)

After receiving query results, pass all string values through the secret/PII scrubber pattern table from `@modules/guardrails/01-secret-pii-scrubber.md` before including in any output, log, or document.

Never log the raw query results to the audit trail — log only the row count and column names.
