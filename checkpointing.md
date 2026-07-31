## Checkpointing and Resume

A checkpoint file is written after each phase completes. On session start, check for an existing checkpoint for the current ticket and resume from the last completed phase rather than restarting from Phase 1.

---

### Checkpoint file location

`.claude/checkpoint-<TICKET_ID>.json`

Example: for ticket `APP-4821` → `.claude/checkpoint-APP-4821.json`

The `.claude/` directory is gitignored. Do not commit checkpoint files.

---

### Checkpoint schema

```json
{
  "ticketId": "<TICKET_ID>",
  "startedAt": "<ISO-8601 timestamp>",
  "lastCompletedPhase": "<0 | 1 | 2 | 3 | 3b | 4 | 5 | 6 | 7>",
  "phases": {
    "0":  { "completedAt": "<ISO-8601>", "status": "complete" },
    "1":  { "completedAt": "<ISO-8601>", "status": "complete", "summary": "<ticket title, services, error signature>" },
    "2":  { "completedAt": "<ISO-8601>", "status": "complete", "summary": "<service→repo mappings>" },
    "3":  { "completedAt": "<ISO-8601>", "status": "complete", "summary": "<log investigation summary one-liner>" },
    "3b": { "completedAt": "<ISO-8601>", "status": "complete", "summary": "<KB findings one-liner>" },
    "4":  { "completedAt": "<ISO-8601>", "status": "complete", "summary": "<root cause one-liner>" },
    "5":  { "completedAt": "<ISO-8601>", "status": "complete", "summary": "<fix branch name and key files changed>" },
    "6":  { "completedAt": "<ISO-8601>", "status": "complete", "summary": "<RCA document path or Confluence URL>" },
    "7":  { "completedAt": "<ISO-8601>", "status": "complete", "summary": "<PR URL>" }
  }
}
```

Only phases that have completed appear under `"phases"`. `lastCompletedPhase` holds the highest phase key completed so far. Phase `"3b"` sorts after `"3"` and before `"4"`.

---

### On session start — check for existing checkpoint

After Phase 0 pre-flight passes and the ticket ID is known, run:

```bash
cat .claude/checkpoint-<TICKET_ID>.json 2>/dev/null
```

**If the file does not exist:** proceed normally from Phase 1.

**If the file exists:**
1. Parse `lastCompletedPhase` and `phases`.
2. Log `CHECKPOINT_RESTORED` to the audit trail immediately after parsing:
   ```json
   {
     "phase": "system",
     "event": "CHECKPOINT_RESTORED",
     "detail": {
       "lastCompletedPhase": "<lastCompletedPhase>",
       "startedAt": "<startedAt>",
       "path": ".claude/checkpoint-<TICKET_ID>.json"
     },
     "ts": "<ISO 8601>",
     "tokens": { "budget": <N>, "spent": <N>, "left": <N> }
   }
   ```
3. If `startedAt` is more than 7 days ago:
   ```
   ⚠️ STALE CHECKPOINT FOUND

     Ticket    : <TICKET_ID>
     Started   : <startedAt>
     Last phase: <lastCompletedPhase>

   This checkpoint is more than 7 days old. Resume may produce
   inconsistent results if the ticket, repo, or environment has
   changed.

   Options:
     [R] Resume from Phase <N+1> using stale checkpoint
     [D] Discard checkpoint and restart from Phase 1
   ```
   Wait for user choice. Do not proceed until answered.
4. If the checkpoint is fresh (≤ 7 days):
   - Restore phase summaries from `phases[N].summary` into context.
   - Skip all phases up to and including `lastCompletedPhase`.
   - Announce:
     ```
     ♻️ RESUMING — <TICKET_ID>

       Checkpoint found. Last completed phase: <lastCompletedPhase>
       Restoring context from checkpoint and continuing from Phase <N+1>.

       Completed phases:
         ✅ Phase 0  — pre-flight
         ✅ Phase 1  — <phases.1.summary>
         ✅ Phase 2  — <phases.2.summary>
         ... (only completed phases listed)
     ```

---

### After each phase completes — write checkpoint

Immediately after a phase produces its output block, update the checkpoint file.

Read the existing file (or start from the base schema if it does not exist), add the completed phase entry, update `lastCompletedPhase`, and write back:

```bash
# Read current checkpoint, update in-memory, write back with Write tool
```

Immediately after the Write succeeds, log `CHECKPOINT_WRITTEN` to the audit trail:

```json
{
  "phase": "system",
  "event": "CHECKPOINT_WRITTEN",
  "detail": {
    "phase": "<completed-phase-key>",
    "path": ".claude/checkpoint-<TICKET_ID>.json"
  },
  "ts": "<ISO 8601>",
  "tokens": { "budget": <N>, "spent": <N>, "left": <N> }
}
```

Phase key mapping:

| Phase | Key in `phases` |
|-------|----------------|
| Phase 0 | `"0"` |
| Phase 1 | `"1"` |
| Phase 2 | `"2"` |
| Phase 3 | `"3"` |
| Phase 3b | `"3b"` |
| Phase 4 | `"4"` |
| Phase 5 | `"5"` |
| Phase 6 | `"6"` |
| Phase 7 | `"7"` |

The `summary` field should be a single line capturing the key output (e.g., the first line of the output block). It is used to restore context on resume — keep it information-dense.

---

### On successful completion — delete checkpoint

After Phase 7 and Phase 8 finish successfully, delete the checkpoint file:

```bash
rm .claude/checkpoint-<TICKET_ID>.json
```

Log the deletion. If deletion fails, warn but do not block the final output.
