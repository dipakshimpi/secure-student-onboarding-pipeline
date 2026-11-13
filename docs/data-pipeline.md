# Data Pipeline — Pub/Sub & BigQuery

**Author:** Dipak Shimpi — Cloud & DevOps Engineer

## Overview

Validated student onboarding records are streamed through Google Cloud Pub/Sub into BigQuery for analytics. The pipeline enforces schema contracts end-to-end, preventing schema drift from silently breaking downstream consumers.

## Data Flow

```text
Student Onboarding Request
        |
        v
Django REST API
        |
        | validation + DCYN
        v
Google Cloud Pub/Sub
        |
        | topic schema (AVRO)
        v
BigQuery Subscription
        |
        v
D1 — Staged / Enforced
```

## Pub/Sub Schema Contract

The Pub/Sub topic uses an AVRO schema named `student-onboarding-schema`.

| Field | Pub/Sub Type | BigQuery Type |
|---|---|---|
| `student_id` | string | STRING |
| `full_name` | string | STRING |
| `email` | string | STRING |
| `owner_email` | string | STRING |
| `course` | string | STRING |
| `status` | string | STRING |
| `created_at` | timestamp-micros | TIMESTAMP |
| `schema_version` | nullable string | STRING |

The `created_at` field uses an AVRO `timestamp-micros` logical type, stored as a BigQuery `TIMESTAMP`. The contract is explicit rather than relying on implicit string conversion.

## Schema Versioning

Each onboarding event includes `schema_version = "1.0"`.

The schema version provides an explicit contract between the Django producer, Pub/Sub, and BigQuery. If the event structure changes in the future, compatibility can be checked before updating the downstream table contract.

## BigQuery Subscription

Terraform creates the subscription `student-onboarding-bigquery`, connecting:

```text
student-onboarding (Pub/Sub topic)
        |
        v
d1_staged_enforced.student_onboarding (BigQuery table)
```

Configuration:
- `use_topic_schema = true` — subscription uses the Pub/Sub topic schema.
- `drop_unknown_fields = false` — unknown fields are not silently discarded, making schema mismatches visible.

## Expected Data Contract

A successfully accepted onboarding event:

```json
{
  "student_id": "STU1001",
  "full_name": "Aarav Sharma",
  "email": "aarav@example.com",
  "owner_email": "aarav@example.com",
  "course": "Cloud & DevOps Engineering",
  "status": "accepted",
  "created_at": "2026-11-13T15:30:00Z",
  "schema_version": "1.0"
}
```

The API does not publish the event until DRF serializer and DCYN validation both succeed.

## Schema Change Safety

For future schema changes, compatibility should be checked before deployment:

```text
Developer changes schema
        ↓
CI validation
        ↓
Application schema check
        ↓
BigQuery compatibility check
        ↓
PASS → Deploy  |  FAIL → Block
```

## Security Controls

| Risk | Control |
|---|---|
| Schema mismatch | Pub/Sub AVRO topic schema |
| Silent unknown fields | `drop_unknown_fields = false` |
| Uncontrolled analytics access | `roles/bigquery.dataViewer` |
| Cross-row data exposure | BigQuery row-level security |
| Invalid records reaching pipeline | DRF + DCYN pre-validation |

## End-to-End Query

An end-to-end validation query is provided in [`e2e-query.sql`](../e2e-query.sql) to verify that records accepted by the API have correctly landed in BigQuery D1.

