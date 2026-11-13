# CI Validation — Fail-Closed Build Gate

**Author:** Dipak Shimpi — Cloud & DevOps Engineer

## Overview

The GitHub Actions CI pipeline provides an automated, fail-closed validation boundary for every change made to this project.

The pipeline checks Python code, CI configuration, Terraform configuration, frontend source files, and the repository for accidentally committed secrets. If any required validation fails, the workflow stops with a failure — the change cannot progress.

This removes dependence on manual review as the final protection against mistakes.

## Pipeline

```text
Pull Request / Push
        |
        v
  Checkout Repository
        |
        v
   Python Quality
   /          \
Flake8      Black Check
        |
        v
     YAML Lint
        |
        v
 Terraform fmt
        |
        v
 Terraform init
        |
        v
 Terraform validate
        |
        v
 Frontend Files
        |
        v
     Gitleaks
        |
        v
       PASS
```

The workflow is a single sequential validation job. A failed step fails the entire job.

## Checks

### 1. Python Code Quality

```bash
flake8 backend
```

Checks the Python source for common code-quality and style problems. A failure stops the CI job.

### 2. Python Formatting

```bash
black --check backend
```

Verifies Python code follows the Black formatting standard. The workflow does not auto-modify files — the developer must format before the check passes.

### 3. YAML Lint

```bash
yamllint .github/workflows/ci.yml
```

Validates the CI configuration file. Prevents malformed workflow YAML from passing validation.

### 4. Terraform Formatting

```bash
terraform fmt -check -recursive
```

Checks that all Terraform files are formatted to the Terraform standard. CI does not silently modify source.

### 5. Terraform Initialization

```bash
terraform init -backend=false
```

Initialises providers and modules without accessing remote state. Validates the dependency graph before any deployment.

### 6. Terraform Validation

```bash
terraform validate
```

Checks that the Terraform configuration is syntactically valid and internally consistent.

### 7. Frontend Source Verification

Verifies that expected frontend source files exist:

- `frontend/index.html`
- `frontend/script.js`
- `frontend/style.css`

### 8. Secret Detection

```yaml
- name: Scan repository for hardcoded secrets
  uses: gitleaks/gitleaks-action@v3
```

Gitleaks scans the full repository for accidentally committed credentials and secret patterns. A detected secret fails the CI job.

## Fail-Closed Logic

```text
Validation Check
       |
       v
    Pass?
    /   \
  Yes    No
   |      |
   v      v
 Next    FAIL
 Check   Job
```

A failed step causes the job to fail. The remaining steps do not run. If configured as a required status check, a failed workflow also blocks the pull request from being merged.

## Security Controls

| Risk | Automated Control |
|---|---|
| Python code-quality problems | Flake8 |
| Incorrect Python formatting | Black --check |
| Invalid CI YAML | yamllint |
| Unformatted Terraform | terraform fmt -check |
| Terraform initialization problems | terraform init -backend=false |
| Invalid Terraform configuration | terraform validate |
| Missing frontend files | File verification |
| Hardcoded credentials | Gitleaks |
| Continuation after failure | Fail-closed sequential job |

## Least-Privilege Permissions

```yaml
permissions:
  contents: read
```

The workflow requests only read access to repository contents. No write permissions are granted to the CI workflow.

## Terraform Security Boundary

The CI pipeline validates Terraform configuration but does not apply infrastructure changes. Deployment remains a separate controlled operation. This prevents a normal code push from automatically modifying the cloud environment.

## Tools Used

- GitHub Actions
- `actions/checkout@v6`
- `actions/setup-python@v6` — Python 3.13
- Flake8
- Black
- yamllint
- `hashicorp/setup-terraform@v4` — Terraform 1.14.8
- Gitleaks

## Expected Result

A valid change passes all checks:

```text
Flake8                 PASS
Black                  PASS
YAML Lint              PASS
Terraform Format       PASS
Terraform Init         PASS
Terraform Validate     PASS
Frontend Files         PASS
Gitleaks               PASS
                        ----
                        PASS
```

