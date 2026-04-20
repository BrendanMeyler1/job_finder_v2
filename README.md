# Job Finder v2

An AI-powered job application assistant built for **GRAD 5900** at Northeastern University. The system demonstrates graduate-level mastery of the Model Context Protocol (MCP), multi-agent Manager/Worker design, Human-in-the-Loop (HITL) control, and persistent memory — while functioning as a real, working tool that researches jobs, tailors resumes, fills application forms, and tracks recruiter replies.

---

## What It Does

1. **Upload your resume** — Claude extracts your profile (name, contact, education, experience, skills) from a PDF or DOCX. Answer a few follow-up questions to fill gaps.
2. **Discover jobs** — Search by role and location. JSearch aggregates LinkedIn, Indeed, Google Jobs, ZipRecruiter, and Glassdoor in a single API call. Greenhouse and Lever boards are also queried directly. Each listing is scored for fit against your profile.
3. **Shadow-apply** — One click tailors your resume to the job's language, generates a cover letter, opens the application form in a headless browser, fills every field using Claude vision + Playwright, takes screenshots at each step, and stops before submitting.
4. **Review and approve** — See exactly what was filled in, diff your tailored resume against the original, read the cover letter and custom question answers. Edit anything in Chat, then approve to submit the real application.
5. **Track replies** — Connect your Outlook inbox and the server checks for recruiter emails every 30 minutes, classifying them as interview request / rejection / offer / follow-up and updating the application status automatically.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         React Dashboard                             │
│  /onboard   /discover   /apply   /chat                              │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ HTTP + SSE
┌──────────────────────────▼──────────────────────────────────────────┐
│                    FastAPI  (api/)                                   │
│  /api/profile  /api/jobs  /api/apply  /api/chat  /api/applications  │
│  /api/email    /api/tasks                                           │
│                                                                     │
│  Middleware: RequestLogging · CORS · GlobalExceptionHandler         │
│  Background: TaskRegistry (asyncio) · Email sync loop (30 min)     │
└──────┬──────────────────┬───────────────────┬───────────────────────┘
       │                  │                   │
┌──────▼──────┐  ┌────────▼───────┐  ┌────────▼────────┐
│ Orchestrator│  │  MCP Servers   │  │    Pipeline     │
│  (Manager)  │  │  ┌───────────┐ │  │  tailor → fill  │
│             │  │  │ profile   │ │  │  → screenshot   │
│ Tool-use    │  │  ├───────────┤ │  │  → persist      │
│ loop ≤8     │  │  │  jobs     │ │  └─────────────────┘
│ iterations  │  │  ├───────────┤ │
└──────┬──────┘  │  │  files    │ │  ┌─────────────────┐
       │ delegates└──┴───────────┘ │  │ ConversationMem │
┌──────▼──────────────────────┐   │  │ Rolling summary │
│          Workers            │   │  └─────────────────┘
│  ┌───────────────────────┐  │   │  ┌─────────────────┐
│  │ JobScout              │  │   │  │ SQLite (WAL)    │
│  │  JSearch+Greenhouse   │  │   │  │  user_profile   │
│  │  +Lever · fit scoring │  │   │  │  job_listings   │
│  ├───────────────────────┤  │   │  │  applications   │
│  │ ResumeWriter          │  │   │  │  chat_messages  │
│  │  tailor + cover ltr   │  │   │  │  email_events   │
│  ├───────────────────────┤  │   │  │  (Fernet PII)   │
│  │ FormFillerAgent       │──┘   │  └─────────────────┘
│  │  Playwright + Vision  │      │
│  ├───────────────────────┤      │
│  │ ProfileBuilder        │      │
│  ├───────────────────────┤      │
│  │ EmailTracker          │      │
│  │  Outlook IMAP         │      │
│  └───────────────────────┘      │
└─────────────────────────────────┘
```

---

## Setup

### Prerequisites

- Python 3.11+
- Node.js 20+
- Anthropic API key (required)
- JSearch RapidAPI key (optional — 200 free requests/month)
- Outlook email + App Password (optional — for email tracking)

### 1. Clone and install Python dependencies

```bash
git clone https://github.com/BrendanMeyler1/GRAD-5900
cd GRAD-5900/job_finder_v2
pip install -r requirements.txt
```

### 2. Install Playwright browsers

```bash
playwright install chromium
```

### 3. Configure environment

```bash
cp .env.example .env
# Edit .env — at minimum, set ANTHROPIC_API_KEY
```

Key variables:

```env
# Required
ANTHROPIC_API_KEY=sk-ant-...

# Job discovery (optional — 200 free req/month)
JSEARCH_API_KEY=...

# Email tracking via Outlook IMAP (optional)
OUTLOOK_EMAIL=you@outlook.com
OUTLOOK_APP_PASSWORD=...   # Microsoft App Password, NOT your login password
                           # Generate at: account.microsoft.com → Security → App passwords

# Browser
HEADLESS=false             # Set true to hide the browser during form fills
SLOW_MO=50                 # Milliseconds between Playwright actions

# Development
DEV_MODE=true              # Mock browser + scrapers; LLM still real
LOG_LEVEL=INFO
```

### 4. Initialize the database

```bash
python setup/init_db.py
```

### 5. (Optional) Seed demo data

Loads a fictional but realistic profile, 8 scored job listings, and 2 shadow-review applications so every tab of the dashboard has content immediately:

```bash
python setup/seed.py
```

> **Note:** Seed data uses a fictional persona (John Smith). If you have already uploaded your real resume, run `python setup/init_db.py --reset` first to clear the DB, then re-upload your resume before seeding — or skip seeding entirely.

### 6. Install dashboard dependencies

```bash
cd dashboard
npm install
```

---

## Running

### Backend

```bash
# From project root
python run.py
```

> **Do not use `uvicorn api.main:app --reload` directly on Windows.** Playwright requires `ProactorEventLoop`, but uvicorn 0.40+ returns `SelectorEventLoop` in its reload subprocess. `run.py` passes a custom loop factory that keeps `ProactorEventLoop` in every process. See `browser/proactor_loop.py`.

The server performs a startup validation pass:
- Checks `ANTHROPIC_API_KEY` is set and the Anthropic SDK can instantiate
- Creates all required data directories (`data/`, `data/resumes/`, `data/generated/`, etc.)
- Runs `init_db` if the database does not yet exist
- Logs a startup summary: profile complete, jobs in queue, pending applications

Interactive API docs: **http://localhost:8000/docs**

### Dashboard

```bash
cd dashboard
npm run dev
# Opens http://localhost:5173
```

Vite proxies `/api/*` to `localhost:8000` automatically — no CORS configuration needed in development.

---

## Usage Guide

### Step 1: Upload your resume

On first launch the onboarding screen appears. Drop your PDF or DOCX resume onto the upload zone. Claude extracts your name, contact info, education, experience, and skills. Answer 3–4 quick questions to fill remaining gaps (target role, salary range, remote preference).

### Step 2: Discover jobs

In the **Discover** tab, type a role and location and click **Find**. JobScout fans out to JSearch (aggregates LinkedIn, Indeed, Google Jobs, ZipRecruiter, Glassdoor), Greenhouse boards, and Lever simultaneously. Results are deduplicated, filtered to remove known aggregator and ghost-job domains, and scored for fit against your profile. Fit badges appear as scores compute: green (70+), amber (40–69), red (below 40).

### Step 3: Shadow-apply to top matches

Click **Shadow Apply** on any listing. The pipeline:
1. Tailors your resume to the job's language (Claude) — see [Resume Tailoring Policy](#resume-tailoring-policy) below
2. Writes a 200–260 word cover letter (Claude)
3. Opens the application form in a Playwright-controlled browser
4. Fills every field using a Claude vision loop: screenshot → plan actions → execute → repeat
5. Captures screenshots at each step
6. **Stops before the submit button** — puts the application in "Review"

### Step 4: Review and approve

In the **Apply** tab, find the application in the Review column. The review panel has three tabs:
- **Screenshots** — step-by-step fill images (lightbox, prev/next)
- **Resume Diff** — original vs. tailored side-by-side (green = added, red = removed)
- **Cover Letter & Q&A** — editable cover letter + custom question answers (edit before submitting)

Click **Approve & Submit** (requires an inline confirmation click) to submit the real application. Or **Edit in Chat** to have Claude refine anything first.

### Step 5: Track replies

With Outlook configured, the server checks your inbox every 30 minutes. Emails from companies you applied to are classified automatically:
- `interview_request` → status updated to "Interview Scheduled"
- `rejection` → status updated to "Rejected"
- `offer` → status updated to "Offer Received"
- `followup_needed` → flagged in the Apply view

A pill appears on each application card showing the most recent email category.

### Chat

The **Chat** tab is a full-screen conversation with Claude that always has your complete context: profile, job scores, application statuses, email alerts. The Orchestrator handles multi-step goals:

> *"Find data analyst jobs in Boston, score them, and shadow apply to the top two."*

It decomposes this into `search_jobs → score_fit → tailor_resume × 2 → fill_form × 2`, runs each step via tool use, and reports back with a summary and links to the Review queue.

---

## Resume Tailoring Policy

The resume writer is designed to **never fabricate**. It has two strictly separated categories:

### FROZEN — copied verbatim from your source resume
Recruiters verify these in background checks. Changing them ends candidacies.

- Name, email, phone, city/state, LinkedIn URL, GitHub URL
- Employer names (exact spelling, never abbreviated or upgraded)
- Job titles (exact as written — "Intern" stays "Intern")
- Employment dates (exact months and years — never inferred)
- School name, degree, major, GPA, graduation date

### FREE TO TAILOR — where the optimization happens

- **Professional Summary** — rewritten to open with a role identity statement, include 2–3 JD keywords, and close with a specific value statement for this company
- **Skills section** — reordered and renamed using JD terminology, but only for skills you actually have
- **Bullet points** — reframed to lead with the most JD-relevant behavior, using JD vocabulary where accurate. No new tasks, metrics, or responsibilities are invented.

The education block and experience section headers (title, company, dates) are pre-built in Python from DB data before the LLM call — they are not template placeholders for Claude to fill in.

---

## Job Source Filtering

LinkedIn Easy Apply jobs, Indeed reposts, and other aggregator listings are blocked at the scraper level before they reach the database. These produce ghost jobs, duplicate postings, and anonymous-recruiter situations where your application disappears into a void.

Blocked domains include: `linkedin.com`, `indeed.com`, `ziprecruiter.com`, `dice.com`, `glassdoor.com`, `monster.com`, `careerbuilder.com`, `simplyhired.com`, and others. Only jobs with a direct company application URL are stored.

LinkedIn application URLs are also rejected by the form filler with an explicit error rather than attempting to open a browser and landing on a LinkedIn login wall.

---

## Running Tests

```bash
# All tests
pytest

# Unit tests only
pytest tests/unit/ -v

# Integration tests
pytest tests/integration/ -v

# E2E shadow run (requires full DB + Playwright)
pytest tests/e2e/ -v

# With coverage report
pytest --cov=. --cov-report=term-missing
```

Tests use an isolated SQLite DB per test (`tests/.tmp/`), a mock LLM that returns predefined responses (no Anthropic API calls), and a mock form filler (no real browser). The full unit + integration suite runs in under 30 seconds.

---

## Architecture Deep-Dive

### MCP Servers

Three MCP servers expose structured tool access to agents without custom API code in agents:

| Server | Tools | Purpose |
|--------|-------|---------|
| `profile_server` | `get_profile`, `update_profile`, `add_qa_note`, `get_resume_text`, `get_profile_completeness` | Profile read/write |
| `jobs_server` | `list_jobs`, `get_job`, `update_job_status`, `list_applications`, `get_application`, `get_application_memory` | Jobs + applications DB |
| `files_server` | `read/write_tailored_resume`, `read/write_cover_letter`, `list_screenshots`, `get_fill_log` | Generated document access |

When the `mcp` package is not installed (e.g., in CI), each server falls back to a `_StubServer` whose tools are callable programmatically — tests run without the full SDK.

### Multi-Agent System (Manager/Worker)

The **Orchestrator** is the Manager agent. It receives user goals from chat and delegates to Worker agents via Claude's native tool-use API. The loop runs up to 8 iterations; a plain-text response from Claude (no tool calls) signals completion.

```
User: "Find Python backend jobs in Boston, shadow apply to the best match"
  ↓
Orchestrator → search_jobs("Python backend", "Boston", 20)
  → JobScout: fans out to JSearch + Greenhouse + Lever, deduplicates, scores, returns ranked list
Orchestrator → tailor_resume(job_id="abc123")
  → ResumeWriter: tailors resume + writes cover letter, saves PDFs
Orchestrator → run_shadow_application(job_id="abc123")
  → Pipeline: Playwright fill loop, screenshots, status → shadow_review
Orchestrator: "Shadow applied to Stripe (fit: 84). Check the Review queue."
```

### Universal Form Filler (Playwright + Claude Vision)

`filler/universal.py` uses a screenshot-driven loop instead of per-ATS CSS selectors that break on redesigns:

1. Navigate to apply URL; detect and skip LinkedIn Easy Apply URLs immediately
2. Click the "Apply" / "Apply Now" CTA to reach the actual form
3. Take a full-page screenshot + DOM accessibility snapshot
4. Ask Claude: "What fields are visible? What do I fill next?" → JSON action plan
5. Execute actions (`fill`, `select`, `check`, `upload`, `click Next`, `scroll`)
6. Minimum 4 steps enforced before `done=true` is accepted; required-field count verified via JavaScript before stopping
7. Shadow mode: halt with submit button visible. Live mode: click submit, confirm success page.

Handles Greenhouse, Lever, Workday, Jobvite, and custom ATS without any site-specific code.

### Memory System

| Layer | Mechanism | Lifespan |
|-------|-----------|----------|
| Short-term context | Last 20 chat messages sent to Claude on every turn | Session |
| Rolling summary | Claude summarizes every 30 messages; summary + last 20 = context window | Persistent (SQLite) |
| Per-company form notes | What fields worked, what failed, custom question labels per company | Persistent (SQLite) |
| Application state | `pending → shadow_running → shadow_review → submitted / rejected / offer_received` | Persistent (SQLite) |

### Human-in-the-Loop Pause Points

1. **Fit threshold** — Jobs below 40 are visually de-emphasized. User decides to proceed.
2. **Shadow review** — Every application pauses at `shadow_review`. User inspects screenshots, resume diff, and cover letter before any submission.
3. **Approve confirmation** — "Approve & Submit" shows an inline "Are you sure?" before the live application is sent. No accidental submissions.

### PII Encryption

`email`, `phone`, and `address` fields are encrypted at rest using Fernet symmetric encryption. The key is stored in `DB_ENCRYPTION_KEY` (auto-generated to `data/encryption.key` on first run). No plaintext PII is written to SQLite.

### Windows-Specific Notes

- **`python run.py` instead of uvicorn CLI**: Uses a custom `ProactorEventLoop` factory (`browser/proactor_loop.py`) so Playwright can spawn browser subprocesses in the reload worker process on Windows.
- **`WinError 87` suppressed**: Windows IOCP raises `WinError 87` on `CreateIoCompletionPort` when socket file descriptor numbers get high (Playwright + Chromium open many pipes). The error is recoverable and is suppressed via a targeted exception handler on the event loop — only this specific error is silenced; all other socket errors remain visible.
- **pytest temp dir**: Tests direct temp files to `tests/.tmp/` (set in `pyproject.toml`) to avoid Windows `AppData\Local\Temp` permission errors.

---

## API Reference

Full interactive docs at **http://localhost:8000/docs** (Swagger UI) and **http://localhost:8000/redoc**.

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/profile` | Full profile (all tables merged) |
| `PUT` | `/api/profile` | Update top-level profile fields |
| `POST` | `/api/profile/resume` | Upload PDF/DOCX, extract and save profile |
| `GET` | `/api/profile/completeness` | Completion percentage + missing fields |
| `GET` | `/api/jobs` | Filtered job list with fit scores |
| `POST` | `/api/jobs/search` | Trigger background job search |
| `POST` | `/api/apply/{job_id}/shadow` | Start shadow application (background task) |
| `POST` | `/api/apply/{app_id}/approve` | Approve → submit live |
| `GET` | `/api/apply/{app_id}/screenshot/{n}` | Serve screenshot image |
| `POST` | `/api/chat` | SSE streaming chat |
| `GET` | `/api/chat/history` | Last 50 messages |
| `POST` | `/api/email/sync` | Trigger Outlook IMAP scan |
| `GET` | `/api/tasks/{task_id}` | Background task status (polls until done) |
| `GET` | `/api/health` | Liveness probe |

---

## DEV_MODE

Set `DEV_MODE=true` in `.env` to run the full pipeline without a real browser or external scrapers:

- **JSearch** returns 5 hardcoded job listings
- **Form filler** returns a mock `FillResult` with placeholder PNG screenshots (no browser launched)
- **LLM** still hits the real Anthropic API — prompts, tailoring, and scoring are fully exercised

This lets the entire dashboard flow work on day one without any external API keys except Anthropic.

```bash
# Quick start with mocked externals
DEV_MODE=true python run.py
```

---

## Prompt Tuning

Every agent's behavior is controlled by a Markdown file in `prompts/`. Edit the file, restart the server, and the change takes effect immediately — no code changes needed.

| File | Controls |
|------|----------|
| `prompts/orchestrator.md` | Goal decomposition style, available tools, safety rules |
| `prompts/resume_writer.md` | Tailoring rules, FROZEN vs FREE policy, ATS formatting |
| `prompts/cover_letter.md` | Letter structure, voice, banned phrases, no-fabrication rule |
| `prompts/form_filler.md` | Field-filling strategy, stopping conditions, EEO defaults |
| `prompts/fit_scorer.md` | Scoring rubric, interview likelihood thresholds |
| `prompts/profile_builder.md` | Onboarding question style, extraction format |
| `prompts/email_classifier.md` | Classification categories, urgency thresholds |
| `prompts/chat_system.md` | Chat persona, what context Claude sees, response style |

---

## Logs

Structured JSON logs write to `data/logs/app.log` (rotating, 10 MB × 5 files = 50 MB max). Human-readable format streams to stdout.

```bash
# All errors
grep '"level": "ERROR"' data/logs/app.log | python -m json.tool

# LLM cost for a session
grep '"input_tokens"' data/logs/app.log | python -c \
  "import sys,json; d=[json.loads(l) for l in sys.stdin]; \
   print(sum(x.get('input_tokens',0) for x in d), 'input tokens')"

# Trace one application end-to-end
grep '"app_id": "YOUR_APP_ID"' data/logs/app.log | python -m json.tool
```

---

## Course Concepts Demonstrated (GRAD 5900)

| Concept | Implementation |
|---------|----------------|
| **MCP as "USB for AI"** | 3 MCP servers (`profile_server`, `jobs_server`, `files_server`) expose data via standard tool interface. Agents never query the DB directly. |
| **Custom MCP Server** | `mcp_servers/jobs_server.py` bridges Claude to SQLite without custom API glue in agent code |
| **Manager/Worker Agents** | `orchestrator.py` (Manager) delegates via tool use to `job_scout`, `resume_writer`, `form_filler`, `profile_builder`, `email_tracker` (Workers) |
| **Collaborative multi-agent** | Workers share state via MCP tools — decoupled, independently testable, swappable |
| **Human-in-the-Loop** | 3 mandatory pause points: fit review, shadow review with diffs, approve confirmation. No submission path bypasses human review. |
| **Long-term Memory** | SQLite persistence + rolling conversation summary + per-company form notes survive server restart |
| **State persistence** | Checkpoint-style application states: `pending → shadow_running → shadow_review → submitted / rejected / offer_received` |
| **Prompt tunability** | All agent behavior in `prompts/*.md` — production-configurable without code changes |
| **Email integration** | Outlook IMAP + Claude classification → auto-updates application status |
| **Safety constraints** | Resume FROZEN fields prevent LLM from fabricating education, employer, or date; LinkedIn/aggregator domain blocklist prevents ghost-job submissions |
