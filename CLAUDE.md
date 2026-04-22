# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Commands

### Backend
```bash
python run.py                        # start API server (port 8000) — use this, not uvicorn directly
python setup/init_db.py              # create data/v2.db (idempotent)
python setup/seed.py                 # populate DB with demo data (fictional persona)
python setup/seed.py --force         # re-seed a DB that already has data
```

### Tests
```bash
pytest                               # all tests
pytest tests/unit/ -v                # unit tests only
pytest tests/integration/ -v         # integration tests
pytest tests/unit/test_db_store.py   # single test file
pytest -k "test_score"               # tests matching pattern
pytest --cov=. --cov-report=term-missing  # with coverage
```

### Dashboard (React/Vite)
```bash
cd dashboard && npm install          # first time only
cd dashboard && npm run dev          # dev server on port 5173
```

### Linting / type-checking
```bash
ruff check .                         # lint
ruff format .                        # format
mypy .                               # type check (strict=false, see pyproject.toml)
```

---

## Architecture

### Request → Response Flow

A request to the chat endpoint (`POST /api/chat`) illustrates how every layer connects:

1. **FastAPI** (`api/routes/chat.py`) receives the message, pulls singletons from `app.state` via `api/dependencies.py`
2. The route calls `Orchestrator.run(goal)` for action-oriented requests, or streams directly from `LLMClient.stream()` for conversational turns
3. **Orchestrator** (`agents/orchestrator.py`) runs a Claude tool-use loop (≤10 iterations). Each tool call dispatches to a worker agent or DB query
4. Workers (`agents/job_scout.py`, `agents/resume_writer.py`, etc.) call `LLMClient.chat()` for LLM work and `Store` methods for DB work
5. The **Pipeline** (`pipeline.py`) coordinates the full tailor → fill → persist flow when an application is triggered

### Singleton Lifecycle

All expensive objects are created once in `api/main.py`'s `lifespan()` and stored on `app.state`:

```
app.state.store              # Store (SQLite)
app.state.llm                # LLMClient (Anthropic SDK)
app.state.pipeline           # Pipeline
app.state.orchestrator       # Orchestrator
app.state.job_scout          # JobScout
app.state.profile_builder    # ProfileBuilder
app.state.email_tracker      # EmailTracker
app.state.conversation_memory
app.state.resume_writer
```

Routes access these via `api/dependencies.py` helpers (`get_store`, `get_pipeline`, etc.) — never construct them in route handlers.

### Configuration

`config.py` exports a module-level singleton `settings = Settings()`. Import it anywhere with `from config import settings`. All paths (`db_path`, `resumes_dir`, etc.) are properties derived from `settings.data_dir`. The critical quirk: `env_ignore_empty=True` prevents empty OS env vars (e.g., `ANTHROPIC_API_KEY=""`) from silently overriding `.env` values — without this, Pydantic Settings would prefer the empty OS value.

### Database Layer

`db/store.py` is the only file that runs SQL. Everything goes through `Store` methods. Key patterns:
- `Store.get_full_profile()` returns a `FullProfile` — all tables merged into one object. Use this for LLM context, not individual table getters
- `FullProfile.to_context_string()` renders the profile as a compact text block ready to paste into any prompt
- `email`, `phone`, and `address` are Fernet-encrypted at rest. `db/encryption.py` handles this transparently inside `Store`
- SQLite WAL mode is on; one long-lived connection per `Store` instance is fine for this single-user app

### LLM Client

`llm/client.py` wraps the Anthropic SDK. Three call patterns:
- `await llm.chat(messages, system, tools, max_tokens)` → `str | ToolUseResult`
- `async for chunk in llm.stream(messages, system)` → yields text chunks (used by SSE chat endpoint)
- `await llm.with_image(messages, image_b64, system)` → vision call (used by form filler)

Default model is `claude-opus-4-5` (`DEFAULT_MODEL`). Use `FAST_MODEL = "claude-haiku-4-5"` for cheap classification tasks (fit scoring, email classification).

`load_prompt(name)` loads `prompts/{name}.md` and falls back to an inline default in `_INLINE_DEFAULTS` if the file is missing — agents never break silently.

### Agents

Each agent loads its system prompt at instantiation from `prompts/{name}.md`. Editing the `.md` file and restarting the server changes behavior without touching code.

| Agent | File | LLM calls |
|-------|------|-----------|
| Orchestrator | `agents/orchestrator.py` | 1 per chat turn (tool loop) |
| JobScout | `agents/job_scout.py` | 1 per job scored |
| ResumeWriter | `agents/resume_writer.py` | 2 per application (tailor + cover letter) |
| FormFillerAgent | `agents/form_filler.py` | thin wrapper around `filler/universal.py` |
| ProfileBuilder | `agents/profile_builder.py` | 1 per resume extraction or question |
| EmailTracker | `agents/email_tracker.py` | 1 per email classified |

### Resume Tailoring — FROZEN vs FREE Policy

`agents/resume_writer.py` pre-builds two blocks in Python *before* the LLM call, from actual DB data:
- **Education block** — degree, institution, year, GPA assembled from `profile.education` rows. Claude copies verbatim; never fills placeholders.
- **Experience headers** — `### Title — Company (dates)` assembled from `profile.experience` rows. Claude writes the bullet points only, from the `[SOURCE: ...]` text provided.

This prevents fabrication of school name, GPA, employer name, job title, or dates. The pattern mirrors how the contact line is already pre-built. Do not revert to template-style placeholders (`[Degree, Major — Institution]`) — that's what caused the fabrication.

### Form Filler

`filler/universal.py` runs a screenshot → DOM snapshot → Claude → action loop (up to `max_steps=30`). Key behavioral constraints baked into the code:
- LinkedIn Easy Apply URLs are rejected *before* a browser is opened (URL string check in `fill()`)
- `done=true` is not accepted before step 4; even after step 4, a JavaScript query counts remaining required fields — if any exist, the loop continues
- The step number is passed to Claude each turn so it knows its position in the process

`DEV_MODE=true` returns a `FillResult` stub with placeholder screenshots without opening a browser.

### Job Source Filtering

`scrapers/jsearch.py` has a `_BLOCKED_APPLY_DOMAINS` frozenset (28 domains). Jobs whose apply URL resolves to a blocked domain are dropped in `_parse()` before reaching the DB. This includes LinkedIn, Indeed, ZipRecruiter, Glassdoor, and known ghost-job aggregators.

### Background Tasks

Long-running operations (shadow apply, job search) return a `task_id` immediately and run via FastAPI `BackgroundTasks`. `api/tasks.py` holds an in-memory `TaskRegistry`. The dashboard polls `GET /api/tasks/{task_id}` until `status != "running"`.

### Windows-Specific Quirks

- **Run with `python run.py`**, not `uvicorn api.main:app --reload`. `run.py` passes `loop="browser.proactor_loop:factory"` which ensures `ProactorEventLoop` in the uvicorn reload subprocess — required for Playwright's `asyncio.create_subprocess_exec()`.
- `browser/proactor_loop.py` installs a custom exception handler that silences `WinError 87` from Windows IOCP when socket fd numbers get high (Playwright opens many pipes). The handler only suppresses that specific error.
- `pyproject.toml` sets `--basetemp=tests/.tmp` so pytest doesn't use `AppData\Local\Temp`, which has permission issues on Windows.

### MCP Servers

Three servers in `mcp_servers/` are started in `api/main.py`'s lifespan (when the `mcp` package is installed). Each wraps `Store` methods and `LLMClient` behind `@server.tool()` decorated functions. They fall back to `_StubServer` if the SDK is missing — tools remain callable programmatically so tests work without the full MCP infrastructure.

### Testing Patterns

`tests/conftest.py` provides:
- `store` — fresh SQLite DB per test, real `Store` with a generated Fernet key
- `mock_llm` — `MagicMock` with `AsyncMock` on `.chat`, `.stream`, `.with_image`; override `.chat` per test for specific responses
- `mock_stagehand` — returns a `FillResult(status="shadow_complete")` without a browser
- `seeded_store` — `store` pre-loaded with `sample_profile` + `sample_job`

The `env_setup` fixture is `autouse=True, scope="session"` — it runs before any import and sets `DEV_MODE=true`, preventing real API calls in tests.

### Dashboard

React + Vite + Tailwind. All `.js` files in `src/` are treated as JSX (configured in `vite.config.js`). Vite proxies `/api/*` to `localhost:8000`. State is managed with React Query (`@tanstack/react-query`). The four main views are `OnboardView`, `DiscoverView`, `ApplyView`, and `ChatView` in `src/components/`.

---

## Key Files to Know

| File | Why it matters |
|------|----------------|
| `config.py` | All settings. `from config import settings` everywhere. |
| `db/store.py` | Only file with SQL. All models defined here too. |
| `api/main.py` | Singleton creation, lifespan, route mounting. |
| `api/dependencies.py` | FastAPI `Depends()` helpers — how routes get store/llm/etc. |
| `pipeline.py` | Full application flow: tailor → fill → persist. |
| `agents/resume_writer.py` | Pre-builds frozen education/experience blocks before LLM call. |
| `filler/universal.py` | Playwright + Claude vision loop. 1500+ lines — read the section headers. |
| `llm/client.py` | `load_prompt()`, `DEFAULT_MODEL`, all three call patterns. |
| `prompts/` | Edit `.md` files here to change agent behavior. No code change needed. |
| `browser/proactor_loop.py` | Windows ProactorEventLoop factory + WinError 87 suppression. |
| `tests/conftest.py` | All shared fixtures — read before writing any test. |
