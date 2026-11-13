# Infrastructure — Cloud Resources & Terraform

**Author:** Dipak Shimpi — Cloud & DevOps Engineer

## Overview

All Google Cloud infrastructure is provisioned and managed through Terraform. This ensures the environment is repeatable, version-controlled, and auditable.

## Infrastructure Components

- Google Cloud Storage — D0 Raw Landing
- BigQuery — D1 Staged/Enforced (student onboarding table)
- Google App Engine — backend application
- Dedicated service accounts
- Least-privilege Identity and Access Management
- Conditional IAM bindings
- BigQuery row-level security
- Storage versioning and lifecycle configuration

## D0 — Raw Landing Layer

D0 is the raw data landing layer. It provides a controlled Cloud Storage location for raw incoming objects before validation.

```text
Raw Input
    |
    v
D0 — Cloud Storage
    |
    | validation / processing
    v
D1 — BigQuery
```

D0 and D1 are separate layers so that raw input is never treated as trusted, validated data.

### Storage Configuration

- Uniform bucket-level access
- Public access prevention
- Object versioning
- Google-managed encryption at rest
- Lifecycle rule — objects moved to Nearline after 30 days

### IAM

A dedicated service account is used for ingestion:

- Role: `roles/storage.objectCreator`
- Conditionally scoped to objects within the D0 bucket

This allows the ingestion workload to create objects without granting delete or bucket management permissions.

## D1 — Staged / Enforced Layer

D1 is the validated and enforced data layer. It is implemented using BigQuery and provides an explicit schema and controlled access boundary for validated student onboarding data.

### BigQuery Configuration

- **Dataset:** `d1_staged_enforced`
- **Location:** `asia-south1`
- **Table:** `student_onboarding`

### Table Schema

| Field | Type | Mode | Purpose |
|---|---|---|---|
| `student_id` | STRING | REQUIRED | Unique student identifier |
| `full_name` | STRING | REQUIRED | Student name |
| `email` | STRING | REQUIRED | Student email address |
| `owner_email` | STRING | REQUIRED | Owner identity used by the row-level access policy |
| `course` | STRING | REQUIRED | Course associated with the student |
| `status` | STRING | REQUIRED | Current onboarding status |
| `created_at` | TIMESTAMP | REQUIRED | Record creation timestamp |
| `schema_version` | STRING | NULLABLE | Version of the onboarding event schema |

The table uses an explicit schema. `schema_version` is nullable to support controlled schema evolution.

### IAM

A dedicated analytics service account is used for D1 data access:

- Role: `roles/bigquery.dataViewer`

This provides read access to BigQuery data without granting administrative control over the dataset or infrastructure.

The ingestion and analytics service accounts are separate to keep responsibilities isolated.

### Row-Level Security

D1 includes a BigQuery row-level access policy (`owner_access`):

```sql
owner_email = SESSION_USER()
```

This restricts rows according to the identity of the current BigQuery session. The policy is managed through Terraform.

## Security Controls

| Risk | Control |
|---|---|
| Public bucket access | Public access prevention |
| Inconsistent ACLs | Uniform bucket-level access |
| Accidental object loss | Object versioning |
| Uncontrolled storage growth | Lifecycle rule |
| Excess ingestion permissions | `roles/storage.objectCreator` |
| Missing required fields | Required BigQuery schema fields |
| Unexpected data structure | Explicit table schema |
| Schema mismatch | Pub/Sub topic schema |
| Silent unknown fields | `drop_unknown_fields = false` |
| Uncontrolled analytics permissions | Dedicated analytics service account |
| Unnecessary write access | `roles/bigquery.dataViewer` |
| Cross-row data exposure | BigQuery row-level access policy |
| Manual infrastructure changes | Terraform |

## Terraform Resources

D0 module manages:
- `google_storage_bucket`
- `google_storage_bucket_iam_member`

D1 module manages:
- `google_bigquery_dataset`
- `google_bigquery_table`
- `google_bigquery_dataset_iam_member`
- `google_bigquery_row_access_policy`

Root Terraform manages:
- Service accounts
- Pub/Sub schema, topic, and BigQuery subscription
- App Engine application

## Validation

```bash
terraform fmt
terraform validate
terraform plan
```

