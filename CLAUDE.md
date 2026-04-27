# the-creator-tools

Monorepo for Creator Tools. FastAPI backend + React frontend, deployed via Docker on Render.

## Repos

| Submodule | Stack | GitHub | Path |
|-----------|-------|--------|------|
| ct-backend | Python 3.10+, FastAPI, Poetry, SQLAlchemy 2.0, JWT auth, loguru | [ct-backend](https://github.com/prateek099/ct-backend) | `ct-backend/` |
| ct-frontend | React 18, TypeScript, Vite, React Query, JWT auth | [ct-frontend](https://github.com/prateek099/ct-frontend) | `ct-frontend/` |
| the-creator-tools | Monorepo — docker-compose, render.yaml, CLAUDE.md rules | [the-creator-tools](https://github.com/prateek099/the-creator-tools) | root |

## Clone

```bash
# Clone main repo with both submodules in one command
git clone --recurse-submodules https://github.com/prateek099/the-creator-tools.git
cd the-creator-tools

# If already cloned without submodules
git submodule update --init --recursive
```

## Common Commands

### Start everything (local dev)
```bash
cp ct-backend/.env.example ct-backend/.env
docker compose up              # all services with hot-reload
docker compose up --build      # rebuild after dependency changes
docker compose down            # stop (keep DB volume)
docker compose down -v         # stop + wipe DB
```

### Backend
```bash
docker compose exec backend poetry run pytest                                         # run all tests
docker compose exec backend poetry run pytest tests/path/to/test_file.py::test_name -vv  # single test
docker compose exec backend poetry add <pkg>                                          # add package
docker compose exec backend poetry run alembic upgrade head                           # run migrations
```

### Frontend
```bash
docker compose exec frontend npm run test                  # run all tests
docker compose exec frontend npm run test -- <pattern>     # run tests matching pattern
docker compose exec frontend npm install <pkg>             # add package
```

## Architecture

- Backend: FastAPI REST API on `http://localhost:8000`
  - JWT auth (access token 30 min, refresh token 7 days)
  - Middleware: RequestID → Timing → RequestLogging → CORS
  - Structured logging via loguru (pretty locally, JSON in prod)
  - Rate limiting via slowapi (200 req/min default)
  - Global exception handlers — all errors return `{"error": {"code": ..., "detail": ...}}`
- Frontend: Vite dev server on `http://localhost:5173`
  - Vite proxies `/api` → `http://backend:8000` inside Docker
  - Auth context + ProtectedRoute + auto token refresh interceptor
- DB: Postgres-only at runtime. Local dev uses the docker-compose `db` service (Postgres 16 on `localhost:5432`); prod is Postgres 15 on Render. Tests still use SQLite (`tests/conftest.py`) for isolation and speed.
- Debug: debugpy on port `5678` when `DEBUG=true` — attach via VS Code

## Conventions

- Never commit `.env` files — use `.env.example` as template
- Generate `JWT_SECRET_KEY` with `openssl rand -hex 32`
- All secrets via Render environment variables in production
- PostgreSQL connection string uses `postgresql+psycopg://` (psycopg3, NOT psycopg2)
- PRs require passing tests before merge
- Backend uses `bcrypt` directly for password hashing (passlib removed — incompatible with bcrypt 5.x)

## Recurring backend patterns

These conventions span many files — follow them when adding new routes/services.

- **Auth dependencies** (`app/api/deps.py`): every protected route picks one of `get_current_user`, `get_optional_user`, or `require_admin`. `get_optional_user` returns `None` for the demo token (`sub='demo'`) — that is how unauthenticated demo access works, so do not block on `user is None` unless you mean to.
- **All LLM calls go through `track_openai_call()`** (`app/services/llm_tracker.py`). Never call `openai_call()` directly from a route — it bypasses the LLMUsage audit row (user, endpoint, tokens, duration, status) which is persisted in a `finally` block even on failure.
- **Prompt overrides**: AI routes fetch a DB-backed admin prompt override before building prompts. The pattern is consistent across `video_idea_gen`, `script_generator`, `title_suggestor`, `seo_description` — copy from one of these when adding a new generation route.
- **Errors**: raise `AppError` subclasses (`UnauthorizedError`, `ForbiddenError`, `BadRequestError`, `NotFoundError`). The global handler serialises to `{"error": {"code", "detail"}, "request_id"}`. Do not return ad-hoc `HTTPException`s with custom shapes.

## Recurring frontend patterns

- **React Query mutations** for the AI pipeline live in `src/api/useWorkflow.ts` (and sibling `useX.ts` files). Don't scatter `client.post` calls across components — add a hook.
- **401 auto-refresh** is handled in `src/api/client.ts` (queues concurrent requests, swaps tokens, retries). Don't add manual refresh logic in components.
- **Workflow autosave** listens via `useMutationState` in `src/context/WorkflowContext.tsx`. New pipeline steps should hook into that listener rather than implementing their own save.
- **Error rendering**: use `getApiErrorMessage(err, fallback)` from `src/types/api.ts` to extract `error.detail` from the standard backend error shape.

## Further reading

- [docs/SYSTEM.md](docs/SYSTEM.md) — deep architecture: layering, services, roadmap.
- [docs/project-flow.md](docs/project-flow.md) — end-to-end request trace (frontend → backend → LLM → DB) using the `/idea` flow.

## Submodule Workflow

The [Makefile](Makefile) wraps the common submodule operations — prefer it over the manual commands.

```bash
make sync                                   # pull both submodules to latest main
make push-backend  msg="feat: ..."          # commit+push ct-backend, then bump pointer
make push-frontend msg="feat: ..."          # commit+push ct-frontend, then bump pointer
make push-all      msg="feat: ..."          # both submodules + bump pointer
make branch-backend  b=feat/my-feature      # create+push feature branch in ct-backend
make branch-frontend b=feat/my-feature      # create+push feature branch in ct-frontend
```

Manual flow (when Make isn't handy or you need finer control):

```bash
# Update both submodules to latest from GitHub
git submodule update --remote --merge

# Work inside a submodule
cd ct-backend
git checkout -b feat/my-feature
# make changes, commit, push
git push origin feat/my-feature
cd ..

# Bump the submodule pointer in the monorepo
git add ct-backend
git commit -m "chore: bump ct-backend to latest"
git push origin main

# Check which commit each submodule points to
git submodule status
```

## Git Reference

```bash
# Stage and commit
git add <file>
git commit -m "type: short description"

# Push changes
git push origin main

# Create a feature branch
git checkout -b feat/my-feature

# Pull latest
git pull origin main

# Delete branch after merging
git branch -d feat/my-feature
git push origin --delete feat/my-feature
```
