# Hearth — Submission Checklist

Use this checklist to verify that all required deliverables are present before submitting. Each item can be confirmed independently via file tree inspection.

---

## Backend

- [ ] `backend/main.py` — FastAPI application entry point; defines API routes and WebSocket endpoint
- [ ] `backend/risk_engine.py` — Weighted risk scoring logic (evictions 40%, shelter 30%, mental health 20%, encampment 10%)
- [ ] `backend/simulator.py` — Real-time block event simulator that feeds the WebSocket stream
- [ ] `backend/requirements.txt` — Python package dependencies for pip installation
- [ ] `backend/render.yaml` — Render deployment configuration (service name, start command, environment)

---

## Frontend

### Components (`frontend/src/components/`)

Verify that all 12 component files are present inside `frontend/src/components/`:

- [ ] Component 1
- [ ] Component 2
- [ ] Component 3
- [ ] Component 4
- [ ] Component 5
- [ ] Component 6
- [ ] Component 7
- [ ] Component 8
- [ ] Component 9
- [ ] Component 10
- [ ] Component 11
- [ ] Component 12

> To count them quickly: `ls frontend/src/components/ | wc -l` (macOS/Linux) or `(Get-ChildItem frontend\src\components).Count` (PowerShell). Expected result: **12**.

### Configuration & Build Files

- [ ] `frontend/.env.example` — Environment variable template with all required keys documented
- [ ] `frontend/vercel.json` — Vercel deployment configuration (routing, build settings)
- [ ] `frontend/package.json` — Node.js dependencies and npm scripts (`dev`, `build`, `preview`)

---

## Documentation

- [ ] `README.md` — Setup guide (clone → install → configure → run), tech stack overview, and Render + Vercel deployment notes
- [ ] `PITCH.md` — Investor/judge pitch covering the problem, solution, and impact metrics

---

## Assets

- [ ] `frontend/public/hero-bg.jpg` — Hero background image for the landing banner

> **Note:** This image is **not included in the repository** and must be supplied by the developer. Place any high-resolution JPEG at `frontend/public/hero-bg.jpg` before running or deploying the frontend. Without it, the hero section will render without a background image.

---

## Final Verification

Before submitting, confirm all of the following:

- [ ] Backend starts without errors: `uvicorn main:app --reload` (from repo root)
- [ ] Frontend starts without errors: `npm run dev` (from `frontend/`)
- [ ] WebSocket connection establishes and risk scores update in real time
- [ ] All environment variables in `.env.example` have been filled in `.env`
- [ ] Backend CORS settings include the deployed Vercel frontend URL
- [ ] Frontend `VITE_API_URL` points to the deployed Render backend URL
