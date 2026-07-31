## GUARDRAIL — Diff Size & Scope Enforcement

Apply at two points in Phase 5: during Step 2 (fix plan presentation) to warn about large plans, and after Step 3 (implementation) to verify the actual diff matches the approved plan.

---

### Thresholds

| Metric | Warning threshold | Hard confirmation threshold |
|---|---|---|
| Files changed | > 5 | > 10 |
| Lines added + removed | > 200 | > 500 |
| Repos touched | > 1 | > 2 |
| Dependency major version bumps | > 0 | — (each requires individual confirmation) |

### Check A — Fix plan scope warning (Phase 5 Step 2)

Before presenting the fix plan for user confirmation, evaluate its scope against the thresholds above.

**If any warning threshold is exceeded:**
```
⚠️ LARGE FIX SCOPE DETECTED

  Files to change : <N>  (threshold: 5)
  Lines estimated : ~<N> (threshold: 200)
  Repos involved  : <N>  (threshold: 1)

  A fix of this size carries elevated risk of:
    • Introducing unrelated regressions
    • Longer review cycles
    • Difficult-to-revert changes

  Review the fix plan carefully. Consider whether all listed changes are
  strictly necessary to address the root cause.
```

Append this warning to the fix plan block. Do not block — the user will see it as part of the standard Phase 5 Step 2 confirmation.

**If any hard confirmation threshold is exceeded, add this to the prompt:**
```
  ❗ This plan exceeds the hard scope limit (>10 files or >500 lines).
     You must explicitly confirm you have reviewed the full plan before proceeding.
     Type "I have reviewed the full plan" to continue.
```

### Check B — Post-implementation diff verification (after Phase 5 Step 3)

After implementing the fix, run:
```bash
git diff --stat origin/<defaultBaseBranch>...fix/<ticket-id>-<slug>
```

Compare the actual diff against the approved plan:

| Condition | Action |
|---|---|
| Diff matches plan (±10%) | ✓ Continue |
| Actual files > approved files | ⚠️ STOP — see prompt below |
| Actual lines > 150% of estimated | ⚠️ STOP — see prompt below |
| Files modified but not in approved plan | ⚠️ STOP — see prompt below |

**If the diff exceeds the approved plan:**
```
⛔ IMPLEMENTATION EXCEEDS APPROVED PLAN — HALTED

  Approved plan  : <N> files, ~<N> lines
  Actual diff    : <N> files, <N> lines

  Unexpected changes detected:
    <file not in approved plan> — <+N/-N lines>
    <file not in approved plan> — <+N/-N lines>

  The branch has NOT been pushed.

  Options:
    A) Review the unexpected changes, confirm they are intentional, and proceed.
    B) Revert the unexpected changes (git checkout -- <files>) and retry.
    C) Abandon this fix and start over.
```

**STOP. Do not push the branch until the user responds.**

### Dependency update rules

- Each dependency major version bump (e.g., `lodash 3.x → 4.x`) requires its own confirmation:
  ```
  ⚠️ MAJOR VERSION BUMP: <package> <old> → <new>
     Breaking changes may exist. Confirm this upgrade is intentional. [Y / N]
  ```
- Minor and patch version bumps within the fix scope do not require individual confirmation.
- Never update more than one major-version dependency per PR without explicit user instruction.
