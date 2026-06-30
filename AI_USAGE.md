# AI Usage

This document describes how AI assistance was used during the implementation of this assignment.

## Tool used

Claude (Anthropic) via Claude Code CLI — an interactive coding assistant running in the terminal alongside the project.

## How it was used

AI was used as a **reference and review tool** throughout implementation — I drove the design, scope, and testing; AI helped with specific implementation details which I then reviewed, corrected, and approved. Concretely:

### Architecture decisions
I consulted the AI on non-obvious architectural choices before implementing them — for example:
- SQLite in WAL mode vs. Redis vs. in-memory queue for job state
- Whether to use an ORM (SQLAlchemy) or raw `aiosqlite`
- The retry/backoff strategy and how to distinguish retriable from non-retriable errors
- Whether to run the worker in-process or as a separate container

In each case I described the constraints and options; the AI explained trade-offs; I made the final call. Those decisions are recorded in `DECISIONS.md`.

### Code generation
I defined the structure and interface of each module; AI helped fill in implementation details for specific stdlib and third-party APIs I looked up in parallel. Areas where AI was most useful for implementation details:
- `app/worker/runner.py` — atomic job claiming via `RETURNING`, retry loop, backoff
- `app/worker/steps/transform.py` — streaming CSV/JSON filter/select/apply with operator dispatch and "did you mean?" column hints
- `app/worker/steps/convert.py` — streaming CSV→JSON, JSON→CSV
- `app/worker/steps/compress.py` — gzip via `asyncio.to_thread` with 256 KB chunked streaming
- `app/worker/steps/notify.py` — HTTP webhook with 4xx/5xx error classification
- `tests/conftest.py` — isolated-storage monkeypatching pattern and `wait_for_job` helper
- `docker-compose.yml` — two-service setup with healthcheck dependency and shared volume

In each case I reviewed the generated code, ran the tests, and made corrections. Several bugs were found during review (see **Debugging** below).

### Debugging
The AI helped diagnose a subtle test isolation bug: the worker imported `DB_PATH` with `from app.db.database import DB_PATH`, which copies the value at import time, making later `monkeypatch.setattr(db_mod, "DB_PATH", ...)` calls ineffective. The fix was to change to `import app.db.database as _db` and reference `_db.DB_PATH` so the worker always reads the current module attribute. This was not immediately obvious from the test failure (TimeoutError) and would have taken significant time to diagnose manually.

### What I drove
- The overall file → queue → worker → storage pipeline design
- All decisions in `DECISIONS.md` (AI presented options; I chose)
- Feature scoping (e.g. choosing 6 filter operators rather than full SQL expressions)
- Renaming `intermediate/` → `output/` for clarity
- Replacing the external httpbin.org webhook target with a local `/webhook` endpoint to eliminate test flakiness from external dependencies
- Reviewing and approving each code change before it was written to disk

## Honest assessment

Using AI as a reference tool compressed the implementation time significantly — it's faster to review and correct generated implementations than to look up stdlib documentation from scratch for every unfamiliar API. That said, I took responsibility for the final code: I reviewed every change, ran the full test suite, and decided what to keep, modify, or reject. Several AI-generated snippets were incorrect on the first pass and required fixes before they passed tests.

The design decisions — what to build, what trade-offs to make, what is in scope — were mine.
