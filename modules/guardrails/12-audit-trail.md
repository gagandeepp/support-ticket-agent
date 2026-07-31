## GUARDRAIL — Audit Trail

Maintain a structured, append-only log of all significant agent actions throughout the workflow. Every event is written to **two destinations simultaneously**: the in-memory log (for the final report) and a persistent file on disk (for cross-run analysis).

---

### Log format

Each entry is a JSON object on a single line:

```json
{
  "phase": "<phase>",
  "event": "<EVENT_TYPE>",
  "detail": { ... },
  "ts": "<ISO 8601>",
  "tokens": {
    "budget": <N>,
    "spent": <N>,
    "left": <N>
  }
}
```

**`phase`** — one of: `preflight`, `1`, `2`, `3`, `3b`, `4`, `5`, `6`, `7`, `8`, `guardrail`, `system`

**`event`** — uppercase snake_case event type (see catalogue below)

**`detail`** — event-specific key/value pairs; never include secret values, only labels/counts

**`ts`** — UTC timestamp at event fire time in ISO 8601 format: `2024-01-15T14:32:07Z`

**`tokens`** — snapshot of token budget at the moment the event fires:
- `budget` — total token budget for this run (default: `200000` for claude-sonnet; override via `tokenBudget` in config if present)
- `spent` — estimated cumulative tokens consumed so far, calculated as `floor(total_conversation_characters / 4)`; include all prior messages, tool outputs, and phase content
- `left` — `budget - spent`; if this drops below `10000`, emit a `TOKEN_BUDGET_LOW` system event immediately

---

### Disk persistence

**Every event must be appended to the log file immediately after being added to the in-memory log.** Use the Bash tool:

```bash
mkdir -p .claude/logs
echo '<single-line JSON event>' >> .claude/logs/audit-<TICKET_ID>-<RUN_TS>.jsonl
```

Where:
- `<TICKET_ID>` — the ticket being resolved (e.g. `APP-4821`)
- `<RUN_TS>` — the ISO 8601 UTC timestamp when the run started, with colons replaced by hyphens (e.g. `2024-01-15T14-32-07Z`); set once at run initialization and reused for every event in the run

**Initialization:** At run start (before Phase 0), create the log file and write the first `PREFLIGHT_STARTED` event. Ensure `.claude/logs/` exists before writing:

```bash
mkdir -p .claude/logs .claude/metrics
```

**On write failure:** If the Bash append fails (disk full, permission denied), output a single warning:
```
⚠️ AUDIT LOG WRITE FAILED — .claude/logs/audit-<TICKET_ID>-<RUN_TS>.jsonl
  Error: <OS error>
  In-memory log will continue. Events since last successful write may be lost from the file.
```
Do not STOP or retry — the in-memory log remains authoritative for the current session.

---

### Event catalogue

#### System / lifecycle events
| Event | Phase | Key detail fields |
|---|---|---|
| `PHASE_STARTED` | any | `phase: string` |
| `PHASE_COMPLETED` | any | `phase: string`, `durationSeconds: number` |
| `SERVICE_CIRCUIT_OPEN` | any | `service: string`, `lastError: string`, `retryCount: number`, `impact: string` |
| `TOKEN_BUDGET_LOW` | any | `left: number`, `threshold: number` |
| `CHECKPOINT_WRITTEN` | system | `phase: string`, `path: string` |
| `CHECKPOINT_RESTORED` | system | `lastCompletedPhase: string`, `startedAt: string`, `path: string` |

Emit `PHASE_STARTED` immediately before beginning any phase work. Emit `PHASE_COMPLETED` immediately after the phase output block is produced. Compute `durationSeconds` as the wall-clock elapsed since the corresponding `PHASE_STARTED` event.

#### Pre-flight / Phase 0
| Event | Key detail fields |
|---|---|
| `PREFLIGHT_STARTED` | `integrations: string[]` |
| `PREFLIGHT_PASSED` | `integrations: string[]` |
| `PREFLIGHT_FAILED` | `integration: string`, `reason: string` |
| `PREFLIGHT_SKIPPED` | `integration: string`, `reason: "disabled in config"` |

#### Phase 1 — Ticket ingestion
| Event | Key detail fields |
|---|---|
| `TICKET_FETCHED` | `ticketId`, `status`, `issueType`, `priority`, `environment` |
| `TICKET_VALIDITY_WARN` | `check: string`, `finding: string` |
| `INJECTION_FLAGGED` | `source: "jira_field"`, `field: string`, `pattern: string` |
| `CONCURRENT_RUN_FOUND` | `priorComment: string` |

#### Phase 2 — Repo mapping
| Event | Key detail fields |
|---|---|
| `REPO_RESOLVED_CONFIG` | `service`, `repo` |
| `REPO_RESOLVED_GITHUB` | `service`, `repo`, `searchQuery` |
| `REPO_RESOLUTION_FAILED` | `service`, `stepsAttempted: string[]` |
| `REPO_USER_PROVIDED` | `service`, `repo` |

#### Phase 3 — Observability
| Event | Key detail fields |
|---|---|
| `OBS_PREFLIGHT_PASSED` | `platform` |
| `OBS_PREFLIGHT_FAILED` | `platform`, `reason` |
| `OBS_QUERY_STARTED` | `platform`, `windowDays`, `limitRows` |
| `OBS_QUERY_COMPLETED` | `platform`, `rowsReturned`, `truncated: bool` |
| `OBS_QUERY_FAILED` | `platform`, `error` |
| `OBS_COST_WARNING` | `platform`, `estimatedGb` |
| `INJECTION_FLAGGED` | `source: "obs_result"`, `platform: string` |

#### Phase 3b — Knowledgebase search
| Event | Key detail fields |
|---|---|
| `KB_SEARCH_STARTED` | `platform`, `query` |
| `KB_SEARCH_COMPLETED` | `platform`, `articlesFound: number` |
| `INJECTION_FLAGGED` | `source: "kb_result"`, `platform: string` |

#### Phase 4 — RCA analysis
| Event | Key detail fields |
|---|---|
| `RCA_ANALYSIS_STARTED` | — |
| `RCA_ROOT_CAUSE_IDENTIFIED` | `summary: string (max 200 chars)` |

#### Phase 5 — Fix implementation
| Event | Key detail fields |
|---|---|
| `FIX_PLAN_PRESENTED` | `filesPlanned: number`, `linesEstimated: number` |
| `FIX_PLAN_APPROVED` | `by: "user"` |
| `FIX_PLAN_REJECTED` | — |
| `FIX_PLAN_MODIFIED` | — |
| `PROTECTED_FILE_CONFIRMED` | `file`, `pattern` |
| `PROTECTED_FILE_EXCLUDED` | `file`, `pattern` |
| `FIX_IMPLEMENTED` | `filesChanged: number`, `linesAdded: number`, `linesRemoved: number` |
| `DIFF_SCOPE_WARNING` | `files: number`, `lines: number` |
| `DIFF_SCOPE_EXCEEDED` | `approvedFiles`, `actualFiles`, `approvedLines`, `actualLines` |
| `TESTS_RUN` | `result: "pass" \| "fail" \| "skipped"`, `testsRun: number` |
| `REGRESSION_DETECTED` | `testName: string`, `before: string`, `after: string` |

#### Phase 7 — PR creation
| Event | Key detail fields |
|---|---|
| `BRANCH_PUSHED` | `branch`, `repo`, `commits: number` |
| `PR_CREATED` | `prNumber`, `prUrl`, `repo` |
| `REVIEWER_ASSIGNED` | `handle`, `tier: "T1" \| "T2" \| "T3"` |
| `REVIEWER_SKIPPED` | `reason` |
| `JIRA_COMMENT_ADDED` | `ticketId`, `commentLength: number` |
| `EXISTING_PR_DETECTED` | `prNumber`, `prUrl` |

#### Phase 8 — Final output
| Event | Key detail fields |
|---|---|
| `RCA_PUBLISHED` | `platform`, `url` |
| `RCA_PUBLISH_FAILED` | `platform`, `error` |
| `RCA_PUBLISH_SKIPPED` | `reason` |
| `SECRET_DETECTED_IN_RCA` | `count: number` |
| `METRICS_WRITTEN` | `path: string`, `indexPath: string` |
| `METRICS_PUSH_MONGODB_SUCCESS` | `database: string`, `collection: string`, `insertedId: string` |
| `METRICS_PUSH_MONGODB_FAILED` | `error: string`, `database: string \| null`, `collection: string \| null` |
| `WORKFLOW_COMPLETED` | `ticketId`, `prUrl`, `rcaUrl`, `durationSeconds: number` |
| `WORKFLOW_ABORTED` | `reason`, `phase` |

#### Guardrail events
| Event | Key detail fields |
|---|---|
| `GUARDRAIL_TRIGGERED` | `guardrail: string`, `check: string` |
| `GUARDRAIL_USER_RESPONSE` | `guardrail: string`, `check: string`, `choice: string` |
| `PRODUCTION_GATE_PASSED` | `gate: "G1" \| "G2" \| "G3" \| "G4" \| "G5"` |
| `PRODUCTION_GATE_REJECTED` | `gate: string` |

---

### Rules

1. **Append-only** — never modify or delete an existing entry.
2. **Dual write** — every event goes to in-memory log AND the `.claude/logs/` file immediately.
3. **No secrets** — never log token values, passwords, API keys, or PII. Log labels, counts, and booleans only.
4. **Every guardrail trigger** must produce a `GUARDRAIL_TRIGGERED` event and a `GUARDRAIL_USER_RESPONSE` event once the user responds.
5. **Every external API call** that succeeds or fails must produce a log entry.
6. **Every phase boundary** must produce a `PHASE_STARTED` event (before) and a `PHASE_COMPLETED` event (after).
7. **Every circuit-breaker open** must produce a `SERVICE_CIRCUIT_OPEN` event.
8. **Timestamps** — use the ISO 8601 UTC format: `2024-01-15T14:32:07Z`.
9. **Token snapshot** — include the `tokens` field on every event. Recompute `spent` at each event by estimating `floor(conversation_character_count / 4)`.

---

### Output in final report

Include the full audit trail in the resolution report under a collapsible section:

```markdown
<details>
<summary>Audit Trail (<N> events)</summary>

\`\`\`json
{ "phase": "preflight", "event": "PREFLIGHT_STARTED", ... }
{ "phase": "1", "event": "TICKET_FETCHED", ... }
...
\`\`\`

Also available on disk: `.claude/logs/audit-<TICKET_ID>-<RUN_TS>.jsonl`

</details>
```

Also attach to the PR body under the same collapsible format.
