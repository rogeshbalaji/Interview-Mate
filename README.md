# InterviewMate AI — adaptive mock interview simulator

Practice HR, technical (software, electrical/control), project-viva and mixed interviews with an adaptive interviewer, a transparent five-category scoring rubric with exact transcript evidence, retry comparison, a seven-day plan and a downloadable report.

## Run it

Requirements: Python 3.11+ with `pypdf`, `python-docx`, `reportlab` (`pip install pypdf python-docx reportlab`). No Node build step.

```bash
cd backend
cp ../.env.example .env       # the server reads .env from the folder you start it in; set IM_SECRET_KEY
python -m interviewmate.server    # http://127.0.0.1:8000
```
Open the site → **Start Practicing** → create an account. On the dashboard you can load a clearly labelled **DEMO** session. Sample files are in `samples/`.

Tests (standard library only): `python -m unittest discover -s tests` (41 tests).

## What was built (and the stack)

Python standard-library HTTP server + SQLite + vanilla-JS single-page app (ES modules, no framework). This deliberately differs from the suggested React/TypeScript/FastAPI/PostgreSQL stack: it needs no package installs beyond the three document libraries and runs anywhere. Schema is in `backend/interviewmate/db.py` (versioned); SQLite can be swapped for PostgreSQL by porting `db.py`.

Services: documents (PDF/DOCX/TXT parsing), retrieval, adaptive question selection, sessions, evaluation, coaching, speech (STT/TTS), report (HTML + PDF), analytics, plan, demo — under `backend/interviewmate/services/`. API: [docs/API.md](docs/API.md).

### Implemented and tested
- Auth (email + password, signed tokens), per-user ownership checks on every record, account/session/document deletion, data export, rate limiting, upload type/size/content validation.
- Resume + job-description extraction with user correction, resume↔JD comparison; missing fields are marked unknown, never invented; instruction-like lines in documents are dropped and reported (prompt-injection defence).
- Five modes, adaptive follow-ups (vague → clarify, wrong → diagnostic, "I don't know" → move on), difficulty adaptation, no repeated questions, pause/resume/repeat/skip/end with a correct timer.
- Scoring: 35/25/20/10/10 weighted total; categories without reliable evidence are `insufficient_evidence` and weights are renormalised and flagged *partial*. Every quote is verified as an exact substring of the transcript; model output with invented quotes is downgraded, never repaired.
- Retry mode (second − first), progress analytics, seven-day plan, print-friendly HTML and PDF report, rubric/fairness page with flagging.
- Provider failure handling: missing/failed LLM falls back to the offline evaluator with a notice; the LLM is only used if configured **and** the user consents.
- Verified in headless Chromium: register → consent → upload → review → mixed interview → pause/resume → coaching → results → retry → PDF/print → plan, with no console errors; no horizontal overflow at 390px on every page.

### Limits you should know about
- **Offline evaluator is heuristic** (rubric concept/keyword matching, English only). It is a practice aid; accuracy is far below a real LLM or human. Configure an LLM (`IM_LLM_*`) for richer evaluation — that path is tested only against a local fake OpenAI-compatible server, not a real provider.
- **Voice:** Whisper-compatible STT and TTS adapters are tested against a local fake server only. Browser recording (MediaRecorder) and the browser-voice fallback could not be exercised with real microphone/speakers here. Typed answers always work.
- **Retrieval is lexical (BM25-style)**, not vector embeddings; the `retrieve` interface is isolated so embeddings can replace it.
- **Coding mode is review-only**: no code is executed (no sandbox is provided); the UI states this. Syntax highlighting is a lightweight built-in highlighter.
- **Camera** is an opt-in self-preview only: no analysis, nothing stored, never used in scores.
- No OAuth, email verification or password reset. Seven-day-plan checkmarks are stored in the browser only.
- Single-process server on SQLite: fine for a demo or small group; use a reverse proxy with HTTPS and a real database for more.
- Not tested: real PDF layouts beyond simple text PDFs (scanned PDFs need OCR and are rejected with a message), multi-user load.

## Privacy defaults
Documents and answers stay on the server and go to an external provider only when one is configured and the user opts in (Settings & Privacy). Raw audio is processed in memory and never stored. Scores are practice indicators, not hiring predictions; a rubric does not make an AI unbiased, and the rubric page says so.

## Deployment
Run `python -m interviewmate.server` under a process manager (systemd/Docker) behind HTTPS (nginx/Caddy) with `IM_SECRET_KEY` set, `IM_HOST=127.0.0.1`, and `IM_DB_PATH` on a persistent, backed-up volume. Keep `.env` out of version control. Optional: `IM_LLM_PROVIDER=openai_compatible` with `IM_LLM_BASE_URL/API_KEY/MODEL`; `IM_STT_*` and `IM_TTS_*` for voice.
