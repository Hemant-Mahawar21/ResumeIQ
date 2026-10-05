# Architecture

## Request Flow

```
User (browser)
   │
   ▼
React Frontend (Vite dev server, :5173)
   │  axios calls
   ▼
FastAPI Backend (:8000)
   │
   ├── POST /api/upload            → validates + stores PDF, creates DB row
   │
   ├── POST /api/analyze/{id}      → runs the full pipeline:
   │        │
   │        ├─ resume_parser.py    : PDF → raw text → cleaned text → sections
   │        │                        (contact / summary / experience / education /
   │        │                         skills / projects / certifications / other)
   │        │
   │        ├─ skill_detector.py   : keyword-matches text against data/skills.json
   │        │                        → {technical: [...], soft: [...]}
   │        │                        → optional JD diff if job_description provided
   │        │
   │        ├─ ats_score.py        : 8 weighted sub-scores → overall 0-100
   │        │
   │        ├─ ai_suggestions.py   : tries Gemini → OpenAI → rule-based fallback
   │        │                        → summary, strengths, weaknesses, suggestions
   │        │
   │        └─ database.py         : persists text, score, skills, analysis
   │
   ├── GET  /api/report/{id}       → report_generator.py builds standalone HTML
   ├── GET  /api/history           → lists all past resumes + scores
   └── DELETE /api/resume/{id}     → removes file + DB rows
```

## Why this pipeline shape

Each stage is a pure function that takes the previous stage's output and
returns plain dicts — no stage reaches into another stage's internals. That
means:

- You can unit-test `ats_score.calculate_ats_score()` with a hand-built
  dict, no PDF or database required.
- Swapping `sqlite3` for SQLAlchemy+Postgres only touches `models/database.py`.
- Swapping the AI provider only touches `services/ai_suggestions.py`.
- The route layer (`routes/resume_routes.py`) stays thin — it just wires
  stages together and handles HTTP-specific concerns (status codes, file
  size limits).

## ATS Scoring Model

The overall score is a weighted sum of 8 independently-computed sub-scores
(each 0-100), combined using these weights:

| Parameter      | Weight | What it measures |
|----------------|--------|-------------------|
| Skills         | 20%    | Count of recognized technical + soft skills |
| Experience     | 20%    | Word count, bullet-point usage, presence of quantified numbers |
| Keywords       | 15%    | Total skill/keyword density across the whole resume |
| Contact Info   | 10%    | Presence of email, phone, LinkedIn/GitHub link |
| Projects       | 10%    | Word count in the Projects section |
| Education      | 10%    | Whether an education section was detected at all |
| Formatting     | 10%    | Bullet usage, section count, average line length |
| Length         | 5%     | Total word count vs. the 400-800 word "sweet spot" |

This mirrors what ATS-optimization guides commonly cite as the highest-impact
factors — skills and experience dominate, structural/length factors are
present but smaller. It's a heuristic model, not a claim about how any real
commercial ATS vendor scores resumes (those are proprietary and vary widely).

## AI Feedback: Provider + Fallback Chain

```
generate_ai_feedback()
   │
   ├─ AI_PROVIDER env var picks a preferred provider (default: gemini)
   ├─ try preferred provider (if its key is set) → success? return
   ├─ try the other provider (if its key is set) → success? return
   └─ rule-based fallback (always available, always succeeds)
       — every claim ties directly back to a real computed sub-score,
         so it's honest even without an LLM in the loop
```

The `/analyze` response always includes `"ai_powered": true|false` so the
frontend can be transparent with the user about which path actually ran.

## Database Schema

```
resumes
  id | filename | stored_path | upload_date | raw_text | ats_score

skills
  id | resume_id (FK) | skill_name | skill_type ('technical' | 'soft')

analysis
  id | resume_id (FK) | strengths (json) | weaknesses (json)
     | suggestions (json) | section_scores (json) | summary | created_at
```

## Frontend Architecture

- **Pages** (`src/pages/`) are route-level screens: Landing, Upload,
  Analysis, History, NotFound.
- **Components** (`src/components/`) are stateless/reusable: Navbar, Card,
  ScoreCircle (SVG circular progress), SkillBadge, UploadDropzone, Loader.
- **Context** (`src/context/ToastContext.jsx`) provides app-wide toast
  notifications via a simple provider + hook, no external toast library.
- **API layer** (`src/api/api.js`) is the only file that knows about HTTP —
  every page calls a named function (`uploadResume`, `analyzeResume`, ...)
  instead of calling axios directly, so the backend URL/shape only needs
  to be updated in one place.
