## GUARDRAIL — Fix Regression Verification

Apply in Phase 5 Step 3 after implementing the fix, before the fix branch is pushed. Verify the fix does not introduce regressions in the existing test suite or known-good behaviours.

---

### Step 1 — Detect test framework

Identify the test framework(s) in the repository by inspecting files:

| Framework | Detection signal |
|---|---|
| pytest | `pytest.ini`, `pyproject.toml` with `[tool.pytest]`, `conftest.py` |
| Jest | `jest.config.*`, `package.json` with `"jest"` key |
| Vitest | `vitest.config.*`, `package.json` with `"vitest"` key |
| Go test | `*_test.go` files present |
| JUnit / Maven | `pom.xml` with `<surefire>` or `src/test/java` |
| RSpec | `spec/` directory with `*_spec.rb` files |
| Mocha | `package.json` with `"mocha"` key |

**If no test framework is detected:**
```
ℹ️ No test framework detected in <repo>.
   Skipping automated regression check.
   Manual verification is recommended before merging.
```

Log `TESTS_RUN: { result: "skipped" }` and continue.

---

### Step 2 — Run affected tests

Run only the tests relevant to the changed files (not the full suite) to keep execution time bounded.

**Scoping strategy (in order of preference):**
1. If the framework supports file-scoped runs, run tests in the same directories as changed files.
2. If the framework supports coverage-based selection, use it.
3. If neither is supported, run the full test suite with a 5-minute timeout.

**Commands by framework:**

| Framework | Scoped run command |
|---|---|
| pytest | `pytest <changed_dirs> -x --timeout=300` |
| Jest | `jest --findRelatedTests <changed_files> --passWithNoTests` |
| Vitest | `vitest run --reporter=verbose <changed_files>` |
| Go test | `go test ./... -run <package>` |
| Maven | `mvn test -Dtest=<TestClass>` |
| RSpec | `rspec <spec_files>` |

**Enforce a hard timeout:** kill the test process after 5 minutes (300 seconds) regardless of framework. If killed:
```
⚠️ TEST SUITE TIMED OUT (>5 min)

  Framework : <framework>
  Elapsed   : >300s

  Options:
    A) Continue anyway — skip regression check (not recommended for production tickets).
    B) Run a narrower subset — specify test file(s) to run.
    C) Abort — do not push until tests pass.
```

---

### Step 3 — Evaluate results

**Pass condition:** all previously passing tests still pass. New test failures are regressions.

**If tests pass:**
```
✓ Regression check passed — <N> tests ran, 0 failures.
```
Log `TESTS_RUN: { result: "pass", testsRun: N }` and continue.

**If tests fail:**
```
⛔ REGRESSION DETECTED — FIX HALTED

  Failed tests:
    • <TestClass/file>::<test name> — <failure message (first 200 chars)>
    • ...

  These tests passed before the fix was applied.

  The branch has NOT been pushed.

  Options:
    A) Fix the regression — I will investigate and patch the failing test(s).
    B) Mark as known-failure — I accept these test failures (requires justification).
    C) Abandon this fix — revert all changes and start over.
```

**STOP. Wait for user response.**

- If **A**: agent returns to Phase 5 Step 3 to patch the regression. Re-run verification after patching.
- If **B**: user must provide a written justification (minimum 20 characters). Log `REGRESSION_DETECTED` event with justification. Continue.
- If **C**: run `git checkout -- .` to revert changes and log `WORKFLOW_ABORTED`.

---

### Step 4 — Production ticket extra check

If the PRODUCTION FLAG is set (from `@modules/guardrails/03-production-environment-gates.md`):
- Require the full test suite to pass (not just scoped tests), even if it takes longer.
- Override the scoped-test strategy and run all tests.
- If the full suite exceeds 10 minutes, surface:
  ```
  ⚠️ FULL SUITE RUNNING FOR PRODUCTION TICKET

    Elapsed : <N> min
    Status  : <N> passed, <N> remaining

    This is expected for production tickets. Waiting...
  ```
  Continue waiting (do not kill after 5 min). Apply a 30-minute hard cap instead.

---

### Step 5 — Type-check and lint (if applicable)

After test verification, run a type-check and lint pass if the project has:
- TypeScript: `tsc --noEmit`
- Python: `mypy <changed_files>` or `ruff check <changed_files>`
- Go: `go vet ./...`

**On failure:**
```
⚠️ TYPE CHECK / LINT FAILED

  Tool    : <tsc | mypy | ruff | go vet>
  Errors  : <N> issues found

  (first 5 errors shown)
  <file>:<line>: <message>
  ...

  Fix these before pushing? [Y / N]
```

If N, log as `LINT_SKIPPED` and continue. Type/lint failures do not block the push — they are advisory.
