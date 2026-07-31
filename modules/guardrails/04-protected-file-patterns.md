## GUARDRAIL — Protected File Pattern Deny List

Apply during Phase 5 Step 1 (strategy determination) and again at Step 3 (implementation). If any proposed file change matches a protected pattern, require explicit user confirmation before touching that file.

---

### Deny list — always require confirmation

These file patterns must never be modified without an explicit user confirmation separate from the standard Phase 5 Step 2 fix-plan approval:

**Secrets and credentials:**
```
.env
.env.*
*.pem
*.key
*.p12
*.pfx
*.crt (when writing, not reading)
secrets.*
credentials.*
**/secrets/**
**/.ssh/**
```

**CI/CD and build pipelines:**
```
.github/workflows/**
.circleci/**
.gitlab-ci.yml
Jenkinsfile
Makefile          (requires review — may have deploy targets)
*.sh              (shell scripts — confirm they are not deployment scripts)
```

**Infrastructure and IaC:**
```
terraform.tfstate
terraform.tfstate.backup
*.tfvars
**/infra/**
**/terraform/**
**/helm/values*.yaml   (production values files)
**/k8s/overlays/prod/**
```

**Release and version management:**
```
CHANGELOG*
CHANGELOG.md
**/VERSION
version.go
version.py
__version__.py
package.json       (version field only — not the full file)
pyproject.toml     (version field only)
pom.xml            (version tag only)
```

**Lock files (high blast radius):**
```
package-lock.json
yarn.lock
poetry.lock
Gemfile.lock
go.sum
```

### Confirmation prompt for protected files

When a proposed change targets a protected file:

```
⚠️ PROTECTED FILE DETECTED IN FIX PLAN

  File    : <repo>/<path>
  Pattern : <matched deny-list pattern>
  Change  : <description of the proposed modification>

  This file is classified as sensitive. Modifications to it carry elevated risk.

  Confirm this change is intentional and necessary:
    [Y] Yes — include this file in the fix as described
    [N] No  — exclude this file; I will handle it manually
```

**Each protected file requires its own Y/N response.** A blanket Y on the fix plan (Phase 5 Step 2) does not cover protected files.

### Security-sensitive code patterns — extra label

If the fix modifies files matching these patterns (even if not on the deny list), automatically add the `security-review-required` label to the PR:

```
**/auth/**
**/authentication/**
**/authorization/**
**/rbac/**
**/permissions/**
**/crypto/**
**/encryption/**
**/payment/**
**/billing/**
```
