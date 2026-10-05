# API Documentation

Base URL (local dev): `http://localhost:8000`

Interactive docs (auto-generated, testable in-browser): `http://localhost:8000/docs`

All responses are JSON except `/api/report/{id}`, which returns HTML.

---

## `POST /api/upload`

Uploads a PDF resume. Does not analyze it yet.

**Request:** `multipart/form-data`
| Field | Type | Required |
|-------|------|----------|
| file  | PDF file | yes |

**Constraints:** PDF only, max 5MB, must not be empty.

**Response `200`:**
```json
{
  "resume_id": 1,
  "filename": "john_doe_resume.pdf",
  "message": "Resume uploaded successfully. Call /analyze/{resume_id} next."
}
```

**Errors:** `400` (wrong file type / too large / empty), `500` (couldn't save file).

---

## `POST /api/analyze/{resume_id}`

Runs the full analysis pipeline on a previously uploaded resume.

**Path param:** `resume_id` (int)

**Request:** `multipart/form-data` (all optional)
| Field | Type | Required |
|-------|------|----------|
| job_description | string | no — enables JD keyword matching |

**Response `200`:**
```json
{
  "resume_id": 1,
  "ats_score": 78,
  "section_scores": {
    "contact_info": 100, "skills": 85, "experience": 72, "projects": 60,
    "education": 100, "keywords": 80, "formatting": 75, "length": 100
  },
  "skills_found": { "technical": ["Python", "React", "..."], "soft": ["Communication", "..."] },
  "missing_skills": ["Docker", "AWS"],
  "keyword_match_percent": null,
  "strengths": ["...", "..."],
  "weaknesses": ["...", "..."],
  "suggestions": [
    { "priority": "Critical", "title": "...", "reason": "...", "impact": "..." }
  ],
  "summary": "...",
  "word_count": 512,
  "ai_powered": false
}
```

**Errors:** `404` (resume not found), `422` (PDF unreadable or effectively empty).

---

## `GET /api/report/{resume_id}`

Returns a standalone, styled HTML report for a resume that has already
been analyzed. Open it directly in a browser tab; use Print → Save as PDF
to export a PDF copy.

**Errors:** `404` (resume not found), `400` (resume hasn't been analyzed yet).

---

## `GET /api/history`

Lists every uploaded resume, most recent first.

**Response `200`:**
```json
[
  { "id": 1, "filename": "john_doe_resume.pdf", "upload_date": "2026-07-15T10:00:00", "ats_score": 78 }
]
```

---

## `DELETE /api/resume/{resume_id}`

Deletes a resume's file and all associated database rows (skills, analysis).

**Response `200`:**
```json
{ "message": "Resume deleted successfully." }
```

**Errors:** `404` (resume not found).

---

## `GET /health`

Simple liveness check, useful for Docker healthchecks / uptime monitors.

```json
{ "status": "healthy" }
```
