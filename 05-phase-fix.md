### PHASE 5 — Fix Implementation

---

#### Step 1 — Determine fix strategy

> **Guardrail — Protected Files:** As you identify files to change, check each one against `@modules/guardrails/04-protected-file-patterns.md`. Flag any match before presenting the plan in Step 2. Each protected file requires its own explicit confirmation separate from the plan-level Y/N.

Based on the RCA from Phase 4, determine the fix strategy and identify every file, config, or dependency that needs to change:

- **Code fix**: patch the defective logic in the appropriate repo.
- **Config fix**: update environment variables, feature flags, or config files.
- **Dependency fix**: update a library version or pin a safe version.
- **Infrastructure fix**: update k8s manifests, Helm values, or IaC.

---

#### Step 2 — Present fix plan and ask for confirmation

> **Guardrail — Diff Scope (Check A):** Before displaying the plan, evaluate its scope against `@modules/guardrails/07-diff-scope-enforcement.md`. Append any warning block to the plan if thresholds are exceeded. Require hard-confirmation phrase if the hard threshold is exceeded.

> **Guardrail — Production Gate (G3):** If the PRODUCTION FLAG is set, apply Gate G3 from `@modules/guardrails/03-production-environment-gates.md` in addition to the standard Y/N. Both confirmations are required.

Before writing a single line of code or touching any file, output the full fix plan and wait for explicit user approval.

```
📋 PROPOSED FIX PLAN — <TICKET-ID>

  Strategy : <code fix | config fix | dependency fix | infrastructure fix>
  Branch   : fix/<ticket-id>-<short-description>
  Root Cause (from Phase 4): <one-liner>

  ── Files to change ────────────────────────────────────────────────────────

  1. <repo>/<path/to/file>
     Change  : <what will be modified — be specific: function name, line range, config key>
     Reason  : <why this change directly addresses the root cause>
     Risk    : <low | medium | high> — <one sentence on what could go wrong>

  2. <repo>/<path/to/file>
     Change  : <description>
     Reason  : <reason>
     Risk    : <level> — <description>

  ── Files to create ────────────────────────────────────────────────────────

  3. <repo>/<path/to/new-file>
     Purpose : <what this file does and why it needs to exist>
     Risk    : low

  ── Tests to add / update ──────────────────────────────────────────────────

  4. <repo>/<path/to/test-file>
     Covers  : <the exact scenario that triggered this incident>
     Type    : <unit | integration | e2e>

  ── Dependencies to update ─────────────────────────────────────────────────

  5. <repo>/package.json (or pom.xml / go.mod / requirements.txt)
     Change  : <package> <old-version> → <new-version>
     Reason  : <CVE / bug fix / breaking-change avoidance>

  ── What will NOT be changed ───────────────────────────────────────────────
  <list any related areas that look tempting but are intentionally excluded>

  ── Estimated blast radius of this fix ────────────────────────────────────
  <which services, endpoints, or deployments will be affected by deploying this fix>

❓ Proceed with this fix plan?

  [Y] Yes — implement all changes as described above
  [N] No  — abandon the fix; I will handle it manually
  [M] Modify — I want to change the scope before proceeding
               (describe what to add, remove, or change)
```

**STOP. Do not write any code, create any branch, or modify any file until the user responds.**

- If the user responds **Y**: proceed to Step 3.
- If the user responds **N**: output a summary of the RCA and fix plan as a reference, then stop. Do not open a PR.
- If the user responds **M** or provides modifications: revise the plan, re-present it, and ask for confirmation again before proceeding.

---

#### Step 3 — Implement the approved fix

> **Guardrail — Audit Trail:** Log `FIX_PLAN_APPROVED` and `FIX_IMPLEMENTED` events via `@modules/guardrails/12-audit-trail.md` at the appropriate moments.

For each affected repo, in the order listed in the approved plan:

a. Clone or checkout the repo using the configured GitHub token.
b. Create the fix branch: `fix/<ticket-id>-<short-description>`.
c. Implement each file change exactly as described in the approved plan. Do NOT introduce unrelated changes.
d. Add or update unit/integration tests that cover the bug scenario.
e. Update relevant inline documentation or comments where the WHY is non-obvious.
f. Run available linting and test suites if CI hooks are accessible. If tests fail, STOP and report before continuing.
g. After implementation, apply `@modules/guardrails/13-fix-regression-verification.md` to run the affected test suite and detect regressions before pushing.
h. After tests pass, run `@modules/guardrails/07-diff-scope-enforcement.md` **Check B** to compare the actual diff against the approved plan. STOP if unexpected files or excess lines are detected.

---

#### Step 4 — Fix summary

```
🔧 FIX SUMMARY
  Strategy : <code fix | config fix | dependency fix | infrastructure fix>
  Branch   : fix/<ticket-id>-<slug>
  Approved : Yes (user confirmed at Step 2)
  Changes  :
    - <repo>/<file>: <what changed and why>
    - ...
  Tests Added/Updated:
    - <test file>: <what is now covered>
  Linting / Tests: ✓ passed | ✗ failed (<detail>) | — not available
```
