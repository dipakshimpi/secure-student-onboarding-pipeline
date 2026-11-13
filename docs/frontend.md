# Frontend — Student Onboarding

**Author:** Dipak Shimpi — Cloud & DevOps Engineer

## Overview

The frontend provides a lightweight student onboarding form. It collects student information and sends it to the Django REST Framework backend for authoritative validation and processing.

The frontend is intentionally simple:

- HTML — form structure
- CSS — presentation and layout
- JavaScript — API communication and response handling

No React, Node.js, or build toolchain is required.

## Application Flow

```text
Student
   |
   v
Frontend Form
   |
   v
JavaScript (fetch API)
   |
   v
Django REST Framework API
   |
   v
Serializer + DCYN Validation
   |
   +---- Invalid → HTTP 400 (user sees error message)
   |
   +---- Valid → Pub/Sub → BigQuery D1
```

The frontend is not the validation boundary. The Django backend performs all authoritative checks.

## Files

```text
frontend/
├── index.html   — Onboarding form and field structure
├── style.css    — Layout and styling
└── script.js    — Form collection, JSON POST, response display
```

## User-Facing Responses

| Scenario | Message |
|---|---|
| Successful submission | "Onboarding submitted successfully." |
| Validation failure | "Please check the entered information and try again." |
| Backend unavailable | "Unable to submit onboarding details. Please try again later." |
| Unexpected server error | "Something went wrong. Please try again." |

Internal validation details are never exposed to the user.

## API

```text
POST /api/student-onboarding/
Content-Type: application/json
```

Local development endpoint: `http://127.0.0.1:8000/api/student-onboarding/`

The deployed backend is hosted on Google App Engine.

## Local Setup

Use two terminals:

### Terminal 1 — Backend

```powershell
cd backend
python -m pip install -r requirements.txt
python manage.py runserver
```

Available at `http://127.0.0.1:8000/`

### Terminal 2 — Frontend

```powershell
cd frontend
python -m http.server 5500
```

Available at `http://127.0.0.1:5500/`

The frontend and backend run as separate local servers. Django is configured to allow cross-origin requests from the local frontend.

## Deployment

The Django backend is deployed to Google App Engine using `backend/app.yaml`. The deployed backend publishes accepted onboarding events to the `student-onboarding` Pub/Sub topic.

