# Data Validation — DCYN Schema Rules

**Author:** Dipak Shimpi — Cloud & DevOps Engineer

## Overview

Student onboarding data passes through deterministic validation before entering the downstream analytics pipeline. Invalid records are rejected at the API layer — they never reach Pub/Sub or BigQuery.

The validation flow:

```text
Student Onboarding Form
        ↓
Django REST Framework API
        ↓
DRF ModelSerializer
        ↓
DCYN Validation
        ↓
Accept / Reject
        ↓
Pub/Sub → BigQuery D1
```

## DCYN Validation

DCYN stands for **Deconstructed Clean Yes/No**.

The incoming onboarding payload is broken down into individual validation decisions. Each rule produces only a Yes or No result. The final decision is deterministic — all required checks must pass.

| Field / Rule | Decision |
|---|---|
| Student identifier is present and valid | Yes / No |
| Full name is valid | Yes / No |
| Email address is valid | Yes / No |
| Guardian consent satisfies the required rule | Yes / No |
| Learning difficulty is an allowed value | Yes / No |
| Age band is an allowed value | Yes / No |
| Emergency contact satisfies the required format | Yes / No |
| Data-sharing consent satisfies the required rule | Yes / No |

```text
All required checks = Yes
        ↓
DCYN = Yes → Accept

Any required check = No
        ↓
DCYN = No → Reject
```

This prevents validation from depending on individual human interpretation.

## Django REST Framework Validation

The DRF `ModelSerializer` provides the validation boundary for incoming data.

Explicit rules applied:

- Required fields
- Field length limits
- Allowed values
- Boolean values
- Email format
- Emergency contact format
- Whitespace normalisation
- DCYN validation

The record is accepted only when serializer validation and DCYN both succeed.

```text
Incoming JSON
     ↓
Serializer
     ↓
Field validation
     ↓
DCYN
     ↓
 ┌──────────┴──────────┐
 ↓                     ↓
YES                    NO
 ↓                     ↓
Accept                Reject
```

## Schema Contract

The application schema is treated as a contract between the API and downstream analytics.

```text
Django
   ↓
Validated Record
   ↓
Pub/Sub
   ↓
BigQuery D1
```

The application-side data structure must remain compatible with the BigQuery D1 schema. A schema change is treated as an intentional engineering change and must be validated before deployment.

## Failure Handling

When validation fails, the API returns:

```text
HTTP 400 Bad Request
```

The frontend displays a user-facing message without exposing internal validation details:

> Please check the entered information and try again.

When validation succeeds:

> Onboarding submitted successfully.

## Validation Responsibility

The frontend collects input but is not the authoritative validation boundary.

The backend enforces:

- Required values
- Field length limits
- Email format
- Allowed learning-difficulty values
- Allowed age-band values
- Emergency contact format
- Guardian consent
- Data-sharing consent
- DCYN decision

This prevents invalid frontend input from being treated as trusted data.

## Implementation Files

```text
backend/
├── student_onboarding/
│   ├── dcyn.py
│   ├── models.py
│   ├── pubsub.py
│   ├── serializers.py
│   └── views.py
└── config/
    └── urls.py
```

The frontend submits the onboarding payload to:

```text
POST /api/student-onboarding/
```

