## Phase-Gate Enforcement

Before entering any phase, assert that the required output block from the previous phase is present — either in the current session context or confirmed via the checkpoint file. Never skip a phase gate.

---

### Required output blocks per phase transition

| Entering | Required evidence | Key text to locate |
|----------|------------------|--------------------|
| Phase 1  | Phase 0 pre-flight passed | No credential failure blocks outstanding |
| Phase 2  | Phase 1 complete | `📋 TICKET SUMMARY` block present |
| Phase 3 & 3b | Phase 2 complete | `🗂️ SERVICE → REPO MAP` block present |
| Phase 4  | Phase 3 **and** Phase 3b complete | Both `🔍 LOG INVESTIGATION SUMMARY` **and** `📚 KNOWLEDGEBASE FINDINGS` blocks present |
| Phase 5  | Phase 4 complete | `🧠 ROOT CAUSE ANALYSIS` block present |
| Phase 6  | Phase 5 complete — user approved fix | `🔧 FIX SUMMARY` block present |
| Phase 7  | Phase 6 complete | `📄 RCA DOCUMENT` block present |
| Phase 8  | Phase 7 complete | `🔗 PULL REQUEST` block present |

---

### Gate check procedure

At the start of **each phase**, perform the following check before doing any work:

1. Scan the current session context for the required output block (see table above).
2. If not found in context, check the checkpoint file for `phases[<required-phase>].status = "complete"`.
3. **If evidence is found in either place:** announce the phase and proceed.
4. **If evidence is absent from both:** output the gate failure block below and STOP.

```
🚫 PHASE GATE FAILURE — Cannot enter Phase <N>

  Missing required output from Phase <M>: <block name>

  This guard prevents skipping phases, which could result in:
    - Fixes based on incomplete investigation
    - PRs opened without an approved fix plan
    - RCA documents that contradict actual findings

  Options:
    [A] Re-run Phase <M> now to generate the required output, then
        I will re-check the gate and continue.
    [B] If Phase <M> output exists in a prior session, provide the
        checkpoint path and I will restore it:
          .claude/checkpoint-<TICKET_ID>.json
```

**Do not proceed until the user resolves the failure.**

---

### Restoring from checkpoint

If the required block is absent from context but present in the checkpoint (`phases[N].status = "complete"`):

1. Read `phases[N].summary` from the checkpoint.
2. Reconstruct a minimal context entry:
   ```
   [Restored from checkpoint — Phase <N> completed at <completedAt>]
   <summary>
   ```
3. Treat the restored entry as equivalent to the live output block.
4. Re-run the gate check — it will now pass.
5. Announce the restoration before continuing.
