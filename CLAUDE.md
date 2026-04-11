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
docker compose exec backend poetry run pytest               # run tests
docker compose exec backend poetry add <pkg>                # add package
docker compose exec backend poetry run alembic upgrade head # run migrations
```

### Frontend
```bash
docker compose exec frontend npm run test     # run tests
docker compose exec frontend npm install <pkg> # add package
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
- DB: Postgres on `localhost:5432` (SQLite default in dev)
- Debug: debugpy on port `5678` when `DEBUG=true` — attach via VS Code

## Conventions

- Never commit `.env` files — use `.env.example` as template
- Generate `JWT_SECRET_KEY` with `openssl rand -hex 32`
- All secrets via Render environment variables in production
- PostgreSQL connection string uses `postgresql+psycopg://` (psycopg3, NOT psycopg2)
- PRs require passing tests before merge
- Backend uses `bcrypt` directly for password hashing (passlib removed — incompatible with bcrypt 5.x)

## Submodule Workflow

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
