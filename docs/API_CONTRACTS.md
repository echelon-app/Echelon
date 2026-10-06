# Backend API

Base URL:
`http://localhost:8000`

## GET /health

Response:

{
  "status": "ok"
}

## GET /api/auth/me

Requires: `Authorization: Bearer <Firebase ID Token>`

Response:
```json
{
  "status": "authenticated",
  "uid": "user123",
  "email": "user@example.com",
  "token_data": { ... }
}
```

## GET /api/profile/me

Requires: `Authorization: Bearer <Firebase ID Token>`

Response: Returns the authenticated user's `StudentProfile` if it exists. Returns `404 Not Found` otherwise.
```json
{
  "major": "Computer Science",
  "class_year": "Sophomore",
  "skills": ["Python", "Java"],
  "interests": ["AI", "Cybersecurity"],
  "coursework": ["Data Structures"],
  "experience": []
}
```

## POST /api/profile/parse

Requires: `Authorization: Bearer <Firebase ID Token>`

Accepts:
multipart/form-data

Fields:
- resume: PDF file
- bio: optional string
- interests: optional string

Returns:

{
  "major": "Computer Science",
  "class_year": "Sophomore",
  "skills": ["Python", "Java"],
  "interests": ["AI", "Cybersecurity"],
  "coursework": ["Data Structures"],
  "experience": []
}

## POST /api/profile

Requires: `Authorization: Bearer <Firebase ID Token>`

Creates or replaces the authenticated user's profile without a resume.

Accepts JSON (`ProfileCreateRequest`):
```json
{
  "major": "Computer Science",
  "graduation_year": 2028,
  "skills": ["Python"],
  "interests": ["AI"],
  "coursework": ["Data Structures"],
  "experience": [],
  "bio": "Optional free-text bio."
}
```
`major` and `graduation_year` are required; the rest are optional. `graduation_year` is stored as the
string `class_year` (e.g. `"2028"`).

Response: the saved `StudentProfile`.

## GET /api/opportunities/recommendations

Requires: `Authorization: Bearer <Firebase ID Token>`

Query parameters:
- `limit` (optional integer, default: `10`, min: `1`, max: `50`): Maximum number of recommendations to return.

Response: `RecommendationsResponse`
```json
{
  "student_id": "user123",
  "opportunities": [
    {
      "opportunity": {
        "id": "opp-001",
        "title": "Systems Software Intern",
        "organization": "Virginia Tech CS Department",
        "opportunity_type": "internship",
        "description": "Work on distributed systems and cloud infrastructure.",
        "source_url": "https://cs.vt.edu/opp/001",
        "source_name": "VT CS",
        "source_age": null,
        "active": true,
        "first_seen_at": "2026-09-20T00:00:00Z",
        "last_seen_at": "2026-09-20T00:00:00Z",
        "skills": ["Python", "Linux"],
        "interests": ["Software Systems"],
        "eligibility": ["Undergraduate"],
        "majors": ["Computer Science"],
        "class_years": ["Junior", "Senior"],
        "school_restrictions": [],
        "eligibility_notes": [],
        "degree_levels": ["BS"],
        "work_authorization_requirements": [],
        "career_tracks": [
          {"track": "software_engineering", "weight": 1.0}
        ],
        "location": "Blacksburg, VA",
        "remote_status": "in_person",
        "time_commitment": "10 hrs/week",
        "compensation": "Paid",
        "deadline": "2026-12-01",
        "apply_url": "https://cs.vt.edu/apply/001",
        "contact_name": null,
        "contact_email": null
      },
      "score": 95,
      "match_reason": "Direct match for your Computer Science major, Linux skills, and systems interest.",
      "matched_traits": ["skills", "career_tracks"],
      "gaps": [],
      "career_track_fit": ["software_engineering"],
      "eligibility_status": "ELIGIBLE",
      "eligibility_notes": []
    }
  ]
    }
  ]
}
```

---

## POST /api/swipes

Requires: `Authorization: Bearer <Firebase ID Token>`

Accepts JSON:
```json
{
  "opportunity_id": "opp-001",
  "direction": "right"
}
```
*Note: `direction` can be "left" or "right". Right swipe saves the opportunity, left unsaves it.*

Response: `SwipeResponse`
```json
{
  "id": "dummy",
  "student_id": "user123",
  "opportunity_id": "opp-001",
  "direction": "right",
  "created_at": "2026-09-20T00:00:00Z"
}
```
*Label: UNIT-TESTED*

---

## GET /api/saved

Requires: `Authorization: Bearer <Firebase ID Token>`

Response: `SavedOpportunitiesResponse`
```json
{
  "student_id": "user123",
  "opportunities": [
    {
      "id": "opp-001",
      "title": "Systems Software Intern",
      ...
    }
  ]
}
```
*Label: UNIT-TESTED*

---

## POST /api/saved/{opportunity_id}

Requires: `Authorization: Bearer <Firebase ID Token>`

Response:
```json
{
  "student_id": "user123",
  "opportunity_id": "opp-001"
}
```
*Label: UNIT-TESTED*

---

## DELETE /api/saved/{opportunity_id}

Requires: `Authorization: Bearer <Firebase ID Token>`

Response:
```json
{
  "status": "success"
}
```
*Label: UNIT-TESTED*

---

## POST /api/agent/chat

Requires: `Authorization: Bearer <Firebase ID Token>`

Career-only chat with the Echelon agent. Uses the student's profile and current career preferences, and saves
any new preferences it learns (these feed career-track weighting in recommendations).

Accepts JSON:
```json
{ "message": "I'm looking for a data science internship in New York." }
```

Response:
```json
{
  "reply": "I've updated your career preferences to focus on data science roles in New York.",
  "preferences_updated": true
}
```
Returns `400` if the student has no profile yet.
*Label: UNIT-TESTED*

---

## Domain Models

These models describe the canonical JSON structure used internally and
returned by API endpoints. They are defined in `backend/app/models/`.

### StudentProfile

`backend/app/models/student.py`

```json
{
  "major":      "Computer Science",
  "class_year": "Sophomore",
  "bio":        "Optional free-text bio.",
  "skills":     ["Python", "Java"],
  "interests":  ["AI", "Cybersecurity"],
  "coursework": ["Data Structures"],
  "experience": ["SWE Intern @ Acme"]
}
```

| Field        | Type           | Required | Notes            |
|--------------|----------------|----------|------------------|
| `major`      | string or null | No       |                  |
| `class_year` | string or null | No       |                  |
| `bio`        | string or null | No       |                  |
| `skills`     | list[string]   | No       | Defaults to `[]` |
| `interests`  | list[string]   | No       | Defaults to `[]` |
| `coursework` | list[string]   | No       | Defaults to `[]` |
| `experience` | list[string]   | No       | Defaults to `[]` |

---

### Opportunity

`backend/app/models/opportunity.py`

```json
{
  "id":               "opp-001",
  "title":            "Research Assistant",
  "organization":     "VT CS Department",
  "opportunity_type": "research",
  "description":      "Work on ML research.",
  "source_url":       "https://example.vt.edu/opp/001",
  "source_name":      "VT CS",
  "source_age":       null,
  "active":           true,
  "first_seen_at":    "2026-09-20T00:00:00Z",
  "last_seen_at":     "2026-09-20T00:00:00Z",

  "skills":           ["Python"],
  "interests":        ["AI"],
  "eligibility":      ["Undergraduate"],
  "majors":           ["Computer Science"],
  "class_years":      ["Sophomore", "Junior"],
  "school_restrictions": [],
  "eligibility_notes":   [],
  "degree_levels":       ["BS"],
  "work_authorization_requirements": [],
  "career_tracks":    [{"track": "ai_ml", "weight": 1.0}],

  "location":         "Blacksburg, VA",
  "remote_status":    "in_person",
  "time_commitment":  "10 hrs/week",
  "compensation":     "Unpaid",
  "deadline":         "2026-12-01",
  "apply_url":        "https://example.vt.edu/apply/001",
  "contact_name":     null,
  "contact_email":    null
}
```

| Field              | Type                      | Required | Notes                                          |
|--------------------|---------------------------|----------|------------------------------------------------|
| `id`               | string                    | **Yes**  | Unique canonical identifier                     |
| `title`            | string                    | **Yes**  | Opportunity title                               |
| `organization`     | string                    | **Yes**  | Organization or sponsoring body                |
| `opportunity_type` | string                    | **Yes**  | e.g. research, internship, job, fellowship    |
| `description`      | string                    | **Yes**  | Opportunity description                         |
| `source_url`       | string                    | **Yes**  | Canonical link to opportunity listing          |
| `source_name`      | string or null            | No       | Origin source name                             |
| `source_age`       | string or null            | No       | Age / posting date indicator                   |
| `active`           | boolean                   | No       | Defaults to `true`                             |
| `first_seen_at`    | datetime or null          | No       | Ingestion timestamp                            |
| `last_seen_at`     | datetime or null          | No       | Refresh timestamp                              |
| `skills`           | list[string]              | No       | Defaults to `[]`                               |
| `interests`        | list[string]              | No       | Defaults to `[]`                               |
| `eligibility`      | list[string]              | No       | Defaults to `[]`                               |
| `majors`           | list[string]              | No       | Target majors                                  |
| `class_years`      | list[string]              | No       | Eligible academic standings                    |
| `school_restrictions` | list[string]           | No       | College/school constraints                     |
| `eligibility_notes`| list[string]              | No       | Explanatory eligibility criteria               |
| `degree_levels`    | list[string]              | No       | Target degrees (BS, MS, PhD)                   |
| `work_authorization_requirements` | list[string] | No    | Visa / work authorization                      |
| `career_tracks`    | list[CareerTrackAffinity] | No       | AI-classified track weights                    |
| `location`         | string or null            | No       | Campus or geographic location                  |
| `remote_status`    | string or null            | No       | `remote`, `hybrid`, or `in_person`             |
| `time_commitment`  | string or null            | No       | Hours per week                                 |
| `compensation`     | string or null            | No       | e.g. Unpaid, Paid, Stipend                     |
| `deadline`         | string or null            | No       | ISO 8601 date string                           |
| `apply_url`        | string or null            | No       | Verified application link                      |
| `contact_name`     | string or null            | No       | Never fabricated - only authentic data         |
| `contact_email`    | string or null            | No       | Never fabricated - only authentic data         |
