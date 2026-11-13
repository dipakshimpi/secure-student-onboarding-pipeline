# Secure Student Onboarding & Analytics Pipeline

**Author:** Dipak Shimpi — Cloud & DevOps Engineer
**GitHub:** [dipakshimpi](https://github.com/dipakshimpi)

---

## Overview

A cloud-native, fail-closed student onboarding platform that validates, secures, and streams onboarding data into a managed analytics pipeline — with full infrastructure automation and CI enforcement.

Student onboarding data is sensitive. Invalid or malformed records break downstream analytics. Leaked credentials compromise the entire system. This project addresses all three risks through automated controls at every layer of the stack.

---

## Problem Statement

- Student onboarding data is sensitive and must be validated before reaching analytics systems.
- Incorrect or malformed records silently break downstream reporting.
- Credential or configuration leakage must be blocked before code is ever merged.
- Manual review alone is not a reliable safety boundary.

---

## Solution

- **Fail-closed CI pipeline** — blocks insecure code, bad formatting, and invalid configuration before merge.
- **REST API validation** — Django serializers and DCYN rules reject invalid onboarding records at the application layer.
- **Cloud infrastructure** — Google Cloud provisioned end-to-end with Terraform.
- **Secure data pipeline** — validated records streamed through Pub/Sub into BigQuery with RBAC and row-level security.

---

## System Architecture

### CI / Development Boundary

```text
Developer
    ↓
GitHub Repository
    ↓
GitHub Actions — Fail-Closed Validation
    ├── Python code quality   (Flake8)
    ├── Python formatting     (Black --check)
    ├── YAML validation       (yamllint)
    ├── Terraform format      (terraform fmt -check)
    ├── Terraform validation  (terraform validate)
    ├── Frontend file check
    └── Secret scanning       (Gitleaks)
    ↓
PASS → Continue  |  FAIL → Blocked
```

### Data Flow

```text
Student
    ↓
Frontend (HTML / CSS / JavaScript)
    ↓
Django REST Framework API
    ↓
DRF Serializer + DCYN Validation
    ├── Valid   → Pub/Sub Topic
    └── Invalid → HTTP 400 Rejected
    ↓
Pub/Sub (AVRO schema)
    ↓
BigQuery D1 — Staged / Enforced
    ↓
Analytics
```

Terraform provisions the infrastructure and identity boundaries supporting these flows.

---

## Security Controls

| Boundary | Risk | Control |
|---|---|---|
| Development | Hardcoded credentials | Gitleaks secret scan |
| Development | Bad code / formatting | Flake8, Black, yamllint |
| Development | Invalid Terraform | terraform fmt + validate |
| Application | Invalid onboarding data | DRF Serializer + DCYN |
| Data | Schema mismatch | Pub/Sub AVRO schema |
| Data | Unauthorised access | BigQuery row-level security |
| Infrastructure | Excess permissions | Least-privilege IAM + Terraform |

---

## Core Features

- **Fail-closed CI** — any validation failure stops the pipeline; deployment is blocked.
- **DCYN validation** — deterministic Yes/No rules reject invalid records before they enter the pipeline.
- **Schema contract** — Pub/Sub AVRO schema + BigQuery enforced schema prevent silent data drift.
- **Row-level security** — BigQuery access policy scoped by session identity.
- **Terraform IaC** — all cloud resources provisioned and version-controlled.
- **Least-privilege IAM** — separate service accounts for ingestion and analytics with minimal roles.

---

## Tech Stack

| Layer | Technology |
|---|---|
| CI/CD | GitHub Actions |
| Secret scanning | Gitleaks |
| Code quality | Flake8, Black, yamllint |
| Backend API | Django REST Framework (Python) |
| Data validation | DCYN (custom rule engine) |
| Messaging | Google Cloud Pub/Sub (AVRO) |
| Analytics store | Google BigQuery |
| Object storage | Google Cloud Storage |
| Infrastructure | Terraform |
| Hosting | Google App Engine |
| Frontend | HTML, CSS, JavaScript |

---

## Infrastructure Components

Terraform provisions and manages:

- Google Cloud Storage — D0 Raw Landing bucket (versioning, lifecycle, public access prevention)
- BigQuery — D1 Staged/Enforced dataset and table with explicit schema
- Pub/Sub — topic with AVRO schema and BigQuery subscription
- Google App Engine — backend application service
- IAM — dedicated service accounts with least-privilege roles
- BigQuery row-level access policy

**Detailed docs:** [`docs/infrastructure.md`](docs/infrastructure.md), [`docs/data-pipeline.md`](docs/data-pipeline.md)

---

## Project Structure

```text
secure-student-onboarding-pipeline/
│
├── .github/
│   └── workflows/
│       └── ci.yml
│
├── backend/
│   ├── app.yaml
│   ├── manage.py
│   ├── requirements.txt
│   ├── config/
│   │   ├── settings.py
│   │   ├── urls.py
│   │   └── wsgi.py
│   └── student_onboarding/
│       ├── dcyn.py
│       ├── models.py
│       ├── pubsub.py
│       ├── serializers.py
│       └── views.py
│
├── frontend/
│   ├── index.html
│   ├── script.js
│   └── style.css
│
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── versions.tf
│   ├── pubsub-schema.avsc
│   └── modules/
│       ├── d0-raw-landing/
│       └── d1-staged-enforced/
│
├── docs/
│   ├── infrastructure.md
│   ├── data-pipeline.md
│   ├── ci-validation.md
│   ├── data-validation.md
│   └── frontend.md
│
└── e2e-query.sql
```

---

## Local Setup

### Prerequisites

- Python 3.13+
- Terraform 1.x
- Google Cloud SDK (for deployment)

### Run the Backend

```bash
cd backend
pip install -r requirements.txt
python manage.py runserver
```

Backend available at `http://127.0.0.1:8000/`

### Run the Frontend

Open a second terminal:

```bash
cd frontend
python -m http.server 5500
```

Frontend available at `http://127.0.0.1:5500/`

The frontend submits data to `http://127.0.0.1:8000/api/student-onboarding/`

### Infrastructure (Terraform)

```bash
cd terraform
terraform init
terraform plan
terraform apply
```

---

## Why It Matters

- Sensitive student data is validated and secured before it reaches analytics.
- Schema mismatches are caught at the API layer — not discovered after data is already broken downstream.
- Credentials and configuration drift are blocked automatically before any merge.
- Infrastructure is repeatable, version-controlled, and auditable.
- Security is enforced by default, not by memory.

---

## Project Impact

- Reduced data quality risk through deterministic DCYN validation before ingestion.
- Prevented credential exposure through automated secret scanning on every push.
- Eliminated configuration drift with Terraform-managed infrastructure.
- Improved analytics trust by enforcing schema contracts end-to-end.
- Automated the full validation and deployment pipeline with zero manual gate reliance.

---

## Future Enhancements

- Add schema evolution compatibility checks to CI pipeline.
- Implement dead-letter queue handling for Pub/Sub failures.
- Add observability layer (Cloud Monitoring / alerting).
- Extend DCYN rules with configurable rule sets.
- Add automated end-to-end integration tests.

---

## Documentation

| Document | Description |
|---|---|
| [`docs/infrastructure.md`](docs/infrastructure.md) | Cloud Storage, BigQuery, IAM, Terraform resources |
| [`docs/data-pipeline.md`](docs/data-pipeline.md) | Pub/Sub, BigQuery schema, data flow |
| [`docs/ci-validation.md`](docs/ci-validation.md) | CI pipeline, fail-closed logic, all checks |
| [`docs/data-validation.md`](docs/data-validation.md) | DCYN validation, serializer rules |
| [`docs/frontend.md`](docs/frontend.md) | Frontend setup and API communication |
