# the-creator-tools

Monorepo for Creator Tools. FastAPI backend + React frontend, deployed via Docker on Render.

## Repos
| Submodule | Stack | Path |
|-----------|-------|------|
| ct-backend | Python 3.12, FastAPI, Poetry, SQLAlchemy | `ct-backend/` |
| ct-frontend | React 18, TypeScript, Vite, React Query | `ct-frontend/` |

## Common Commands

### Start everything (local dev)
```bash
docker compose up              # all services with hot-reload
docker compose up --build      # rebuild after dependency changes
docker compose down            # stop (keep DB volume)
docker compose down -v         # stop + wipe DB
```

### Backend
```bash
docker compose exec backend poetry run pytest          # run tests
docker compose exec backend poetry add <pkg>           # add package
docker compose exec backend poetry run alembic upgrade head  # run migrations
```

### Frontend
```bash
docker compose exec frontend npm run test              # run tests
docker compose exec frontend npm install <pkg>         # add package
```

## Debugging
- Backend debugpy runs on port **5678** when `DEBUG=true`
- Attach VS Code via `.vscode/launch.json` → "🐛 Attach to ct-backend (Docker)"
- Set breakpoints in `ct-backend/app/` — they will hit on next API call

## Architecture
- Backend exposes REST API on `http://localhost:8000`
- Frontend Vite dev server on `http://localhost:5173`
- Vite proxies `/api` → `http://backend:8000` inside Docker
- DB: Postgres on `localhost:5432`

## Conventions
- Never commit `.env` files — use `.env.example` as template
- All secrets via Render environment variables in production
- PRs require passing tests before merge

## Submodule Workflow
```bash
# Update both submodules to latest
git submodule update --remote --merge

# Work inside a submodule
cd ct-backend
git checkout -b feature/my-feature
# make changes, commit, push
cd ..
git add ct-backend
git commit -m "chore: bump ct-backend to latest"
```
