# ResumeIQ — AI Resume Analyzer

A full-stack, production-styled AI resume analysis tool. Upload a PDF resume and get:

- **ATS Score** (0-100), with a full section-by-section breakdown
- **Skill detection** (technical + soft skills) and missing-skill suggestions
- **Optional job-description matching** (paste a JD, get a keyword-match %)
- **AI-generated feedback** — summary, strengths, weaknesses, prioritized suggestions
  (uses Gemini or OpenAI if you provide a key; falls back to a transparent
  rule-based engine if you don't, so the app works with **zero API keys**)
- **Downloadable HTML report** (open in browser → Print → Save as PDF)
- **History** of past uploads with delete support

---

## Tech Stack

| Layer      | Tech |
|------------|------|
| Frontend   | React 18, Vite, Tailwind CSS, Axios, Recharts, Framer Motion, lucide-react |
| Backend    | Python, FastAPI, Uvicorn |
| PDF/NLP    | pdfplumber (+ PyMuPDF fallback) |
| Database   | SQLite (swap to PostgreSQL later — the `models/database.py` layer is the only file you'd touch) |
| AI         | Gemini API or OpenAI API (optional) |
| Deployment | Docker + docker-compose |

---

## Project Structure

```
AI-Resume-Analyzer/
├── backend/
│   ├── main.py                  # FastAPI app entry point
│   ├── routes/
│   │   └── resume_routes.py     # /upload /analyze /report /history /resume/{id}
│   ├── services/
│   │   ├── resume_parser.py     # PDF -> text -> sections
│   │   ├── skill_detector.py    # keyword-based skill matching
│   │   ├── ats_score.py         # the scoring engine (8 weighted parameters)
│   │   ├── ai_suggestions.py    # Gemini/OpenAI + rule-based fallback
│   │   └── report_generator.py  # builds the downloadable HTML report
│   ├── models/
│   │   ├── database.py          # sqlite3 data layer
│   │   └── schemas.py           # Pydantic response models
│   ├── data/skills.json         # the skills database (edit this to extend detection)
│   ├── uploads/                 # uploaded PDFs land here
│   ├── requirements.txt
│   └── .env.example
├── frontend/
│   ├── src/
│   │   ├── pages/               # Landing, Upload, Analysis, History, NotFound
│   │   ├── components/          # Navbar, Card, ScoreCircle, SkillBadge, UploadDropzone, Loader
│   │   ├── context/ToastContext.jsx
│   │   ├── api/api.js           # all backend calls live here
│   │   ├── App.jsx / main.jsx
│   ├── tailwind.config.js       # dark SaaS color system baked in
│   └── package.json
├── docker-compose.yml
└── README.md (this file)
```

---

## Quick Start (local, no Docker)

### 1. Backend

```bash
cd backend
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt

cp .env.example .env             # optional: add GEMINI_API_KEY or OPENAI_API_KEY
uvicorn main:app --reload --port 8000
```

Backend now running at `http://localhost:8000` — interactive docs at `http://localhost:8000/docs`.

### 2. Frontend

```bash
cd frontend
npm install
cp .env.example .env             # defaults to http://localhost:8000, edit if needed
npm run dev
```

Frontend now running at `http://localhost:5173`.

### 3. Try it

Open `http://localhost:5173`, click **Analyze Resume**, drop in any PDF resume.
Works immediately with no API keys — you'll get the rule-based analysis.
Add a `GEMINI_API_KEY` (free tier available at [ai.google.dev](https://ai.google.dev))
to backend/.env to unlock real LLM-generated feedback instead.

---

## Quick Start (Docker)

```bash
cp backend/.env.example backend/.env   # add API keys if you want them
docker-compose up --build
```

- Backend: `http://localhost:8000`
- Frontend: `http://localhost:5173`

---

## API Reference

See [`API_DOCS.md`](./API_DOCS.md) for full endpoint documentation, or just visit
`http://localhost:8000/docs` once the backend is running — FastAPI generates
interactive, testable docs automatically from the code.

---

## How the ATS Score is Calculated

See [`ARCHITECTURE.md`](./ARCHITECTURE.md) for the full breakdown of the 8
weighted scoring parameters and the overall pipeline architecture.

---

## What's intentionally simplified

This is a learning/portfolio project, so a few things were scoped down from
"enterprise SaaS" to keep the codebase readable and runnable without a cloud
account:

- **No authentication/login** — single-user, local-first. Adding auth would
  mean a `users` table + JWT middleware in `main.py`; the schema already
  isolates data by `resume_id` so this slots in cleanly later.
- **Report export is HTML, not a generated PDF binary** — avoids heavy
  system dependencies (wkhtmltopdf/weasyprint). The browser's own
  Print → Save as PDF produces an identical result with zero extra libraries.
- **Missing-skills logic is keyword-based**, not a trained ML model — this
  keeps it fast, dependency-light, and easy for you to extend by editing
  `backend/data/skills.json`.
- **Refreshing the analysis page loses the in-memory result** — a next
  step would be a `GET /api/analysis/{resume_id}` endpoint (the data is
  already persisted in SQLite, it's just not wired to a route yet).

---

## Next Steps / Ideas to Extend

- Add a `GET /api/analysis/{resume_id}` endpoint + wire the Analysis page to
  fetch it on load (fixes the refresh issue above)
- Add authentication (FastAPI + JWT) for true multi-user support
- Swap SQLite → PostgreSQL for production
- Add spaCy-based fuzzy skill matching on top of the keyword matcher
- True PDF export via a headless-browser render of the HTML report
