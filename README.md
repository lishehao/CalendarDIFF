# CalendarDIFF

A tool that watches a student's course calendar feeds and email, turns changes to deadlines into a reviewable diff, and alerts on them.

[中文说明](README.zh-CN.md)

## Status

Not maintained.

CalendarDIFF was built in early 2026 as an exercise in connecting a full system end to end: source ingestion, parsing, a canonical event store, a review queue, a web UI and notifications. It was built without first checking whether students needed it. It is not in use.

The code runs locally (steps below) and most of the test suite passes, but there is no hosted instance and no plan to develop it further.

## How to run

These steps start the backend and frontend on a fresh clone. They were checked with Python 3.12; the backend image uses Python 3.11 (`Dockerfile`) and the frontend image uses Node 20 (`frontend/Dockerfile`).

### 1. PostgreSQL and Redis

With Docker running:

```bash
docker compose up -d postgres redis
```

This starts PostgreSQL 16 on `5432` (database `deadline_diff`, user and password `postgres`) and Redis 7 on `6379`, which match `.env.example`.

Without Docker, run your own PostgreSQL and Redis, create a database for the app, and point `DATABASE_URL` and `REDIS_URL` in `.env` at them.

### 2. Backend

```bash
cp .env.example .env
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

set -a; source .env; set +a
python -m alembic upgrade head
SERVICE_NAME=backend PORT=8200 ./scripts/start_service.sh
```

Check it from another shell:

```bash
curl http://127.0.0.1:8200/health
```

`./scripts/start_service.sh` with `RUN_MIGRATIONS=true` runs the migration step itself.

### 3. Frontend

```bash
cd frontend
cp .env.local.example .env.local
npm install
npm run dev -- --hostname 127.0.0.1 --port 3000
```

Open `http://127.0.0.1:3000`. `BACKEND_API_KEY` in `frontend/.env.local` must match `APP_API_KEY` in `.env` (both examples use the same placeholder).

`./scripts/dev_stack.sh up` does steps 1 to 3 in one command once `.env` exists and `frontend/node_modules` is installed. It needs Docker.

### 4. Tests

The tests need PostgreSQL and Redis. They create and drop their own database.

```bash
TEST_DATABASE_URL=postgresql+psycopg://postgres:postgres@localhost:5432/deadline_diff_test \
REDIS_URL=redis://127.0.0.1:6379/15 \
python -m pytest -q
```

See Limits for the tests that fail or skip on a fresh clone.

## Design

- **Backend:** one FastAPI process (`services/app_api/main.py`) over PostgreSQL (SQLAlchemy, Alembic migrations) and Redis (job queues and the LLM request limiter). Module boundaries are under `app/modules/`.
- **Frontend:** a Next.js app in `frontend/` that talks to the backend through `BACKEND_BASE_URL`.
- **Ingestion:** Canvas ICS feeds and Gmail (OAuth, read only) are polled as sources. New or changed items go through a parse step and are matched against a canonical event baseline.
- **Review:** the first import becomes a baseline that the user reviews once. Later changes land in a single review queue (`Changes`), where each proposed change is approved, edited or rejected before it touches the baseline. Notifications go out by SMTP when enabled.
- **Agent and MCP:** an internal gateway (`app/modules/agents/gateway.py`) and an MCP server (`services/mcp_server/main.py`) expose read access, proposals and approval tickets.

Deeper docs:

- [PURPOSE.md](PURPOSE.md): what the system is meant to detect and what it ignores
- [docs/architecture.md](docs/architecture.md): runtime model and module boundaries
- [docs/project_structure.md](docs/project_structure.md): repository layout
- [docs/api_surface_current.md](docs/api_surface_current.md): HTTP routes
- [docs/frontend_backend_contracts.md](docs/frontend_backend_contracts.md): contracts the frontend relies on
- [docs/README.md](docs/README.md): index of the rest

The OpenAPI snapshot is `contracts/openapi/public-service.json`; refresh it with `python scripts/update_openapi_snapshots.py`.

## Limits

- **The parse step needs an LLM.** Both ICS and Gmail items are parsed through `app/modules/llm_gateway`, which needs `LLM_API_KEY`, `LLM_MODEL` and `LLM_BASE_URL` in `.env`. Without them the backend still starts, but parsing fails with `LLM_API_KEY is not configured`. The LLM rewrite of agent proposal text (`AGENT_GENERATION_MODE=llm_assisted`) and the BERT Gmail filter under `training/` are optional and off by default.
- **Gmail needs your own Google OAuth client.** Set `GMAIL_OAUTH_CLIENT_SECRETS_FILE` to a client secrets file.
- **Tests that need generated fixtures skip on a fresh clone.** Three tests read `tests/fixtures/private/email_pool/`, which is gitignored, and skip when it is missing. `python -m tools.datasets.export_year_timeline_fixtures` writes those fixtures from the synthetic year timeline, and `tests/test_year_timeline_fixture_export.py` also writes them as a side effect, so a second test run in the same clone no longer skips them. On that second run `test_fake_source_provider_can_overlay_full_sim_email_bucket` fails, because `scripts/fake_source_provider.py` reads the email pool from a hard-coded absolute path (see below).
- **The training test skips** unless `evaluate`, `numpy`, `torch`, `datasets` and `transformers` are installed. They are not in `requirements.txt`.
- **Three tests fail on a fresh clone:**
  - two preflight tests in `tests/test_full_repo_validation_runner.py`, because `run_preflight` also checks for `frontend/node_modules`, the synthetic manifest under `data/` and the generated email pool on disk;
  - `tests/test_oauth_runtime_config.py::test_frontend_app_base_url_falls_back_to_first_public_origin`, which expects the second entry of `PUBLIC_WEB_ORIGINS` while the code returns the first.
- **Hard-coded local paths.** These files contain absolute paths from the original author's machine and will not work elsewhere without editing:
  - `scripts/compare_gmail_cache_modes.py`
  - `scripts/fake_source_provider.py`
  - `scripts/probe_gmail_live_accuracy.py`
  - `scripts/probe_real_llm_usage.py`
  - `specs/frontend/2026-03-23-agent-entry-surfaces/GUIDANCE_PROMPT.md`
  - `specs/frontend/2026-03-23-agent-entry-surfaces/OUTPUT.md`
  - `specs/frontend/2026-03-23-perceived-performance-tuning/GUIDANCE_PROMPT.md`
  - `specs/frontend/2026-03-23-perceived-performance-tuning/OUTPUT.md`
- **`scripts/run_full_repo_validation.py`** expects the author's local setup and does not pass on a fresh clone.
- **Dependency pins:** `fastapi` is held below `0.130` and `starlette` below `1.0`. With newer releases (for example FastAPI 0.162 with Starlette 1.10) the app fails at startup with `OAuth callback route is not registered in input service app`.

## Who wrote what

Coding agents wrote almost all of the code. I set the direction and defined what the system should do: which sources to watch, how a deadline change becomes a reviewable diff, and when to alert.

## License

No license has been chosen.
