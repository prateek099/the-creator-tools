# the-creator-tools

Full-stack monorepo for **Creator Tools** — a FastAPI + React application with JWT authentication, PostgreSQL, and Docker-based local development. Deployed to [Render](https://render.com) via `render.yaml`.

| | |
|---|---|
| **Backend** | Python 3.10+, FastAPI, SQLAlchemy 2.0, Poetry |
| **Frontend** | React 18, TypeScript, Vite, React Query |
| **Database** | PostgreSQL 16 (SQLite in local dev without Docker) |
| **Auth** | JWT — access token (30 min) + refresh token (7 days) |
| **Deploy** | Docker → Render |

---

## Repositories

| Repo | Description | Link |
|------|-------------|------|
| **the-creator-tools** | This monorepo — docker-compose, render.yaml, CLAUDE.md rules | [github.com/prateek099/the-creator-tools](https://github.com/prateek099/the-creator-tools) |
| **ct-backend** | FastAPI backend — auth, users API, logging, middleware | [github.com/prateek099/ct-backend](https://github.com/prateek099/ct-backend) |
| **ct-frontend** | React frontend — login page, auth context, protected routes | [github.com/prateek099/ct-frontend](https://github.com/prateek099/ct-frontend) |

`ct-backend` and `ct-frontend` are **git submodules** of this repo — each has its own independent commit history and can be developed and deployed separately.

---

## Architecture

```
                        ┌─────────────────────┐
                        │   Browser / Client   │
                        └────────┬────────────┘
                                 │ HTTP
                    ┌────────────▼─────────────┐
                    │   ct-frontend            │
                    │   React + Vite           │
                    │   :5173 (dev)            │
                    │   :80   (prod / nginx)   │
                    └────────────┬─────────────┘
                                 │ /api/v1/*  (proxied)
                    ┌────────────▼─────────────┐
                    │   ct-backend             │
                    │   FastAPI + Uvicorn      │
                    │   :8000                  │
                    │                          │
                    │  Middleware stack:        │
                    │  CORS → RequestID →       │
                    │  Timing → ReqLogging     │
                    └────────────┬─────────────┘
                                 │ SQLAlchemy
                    ┌────────────▼─────────────┐
                    │   PostgreSQL 16          │
                    │   :5432                  │
                    └──────────────────────────┘
```

**Local dev:** Vite dev server proxies `/api` → `http://backend:8000` inside Docker so the frontend calls the backend without CORS issues.

---

## Prerequisites

| Tool | Version | Install |
|------|---------|---------|
| Git | any | [git-scm.com](https://git-scm.com) |
| Docker Desktop | ≥ 4.x | [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/) |
| Python | ≥ 3.10 | Only for the no-Docker path |
| Poetry | 1.8.x | Only for the no-Docker path |
| Node.js | ≥ 20 | Only for the no-Docker path |

---

## Clone

```bash
# Clone the monorepo and both submodules in one command
git clone --recurse-submodules https://github.com/prateek099/the-creator-tools.git
cd the-creator-tools
```

> If you already cloned without `--recurse-submodules`:
> ```bash
> git submodule update --init --recursive
> ```

---

## Quick Start — With Docker (recommended)

The fastest way to run the full stack. Requires Docker Desktop running.

**1 — Set up environment**

```bash
cp ct-backend/.env.example ct-backend/.env
```

Open `ct-backend/.env` and set at minimum:

```dotenv
DEBUG=true
JWT_SECRET_KEY=<run: openssl rand -hex 32>
# Windows: python -c "import secrets; print(secrets.token_hex(32))"
```

**2 — Start all services**

```bash
docker compose up --build
```

That's it. Three services start together:

| Service | URL | Description |
|---------|-----|-------------|
| `frontend` | http://localhost:5173 | React + Vite dev server (HMR) |
| `backend` | http://localhost:8000 | FastAPI (hot-reload via uvicorn) |
| `backend` docs | http://localhost:8000/docs | Swagger UI (requires `DEBUG=true`) |
| `db` | localhost:5432 | PostgreSQL 16 |

**3 — Verify**

```bash
curl http://localhost:8000/health
# {"status":"ok","app":"Creator Tools API","env":"development"}
```

---

## Quick Start — Without Docker

Runs backend on SQLite (no Postgres needed). Two terminals required.

**Terminal 1 — Backend**

```bash
cd ct-backend
cp .env.example .env          # Windows: Copy-Item .env.example .env

# Mac / Linux
curl -sSL https://install.python-poetry.org | POETRY_VERSION=1.8.4 python3 -
export PATH="$HOME/.local/bin:$PATH"

# Windows PowerShell
$env:POETRY_VERSION = "1.8.4"
(Invoke-WebRequest -Uri https://install.python-poetry.org -UseBasicParsing).Content | python -

poetry install
poetry run uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

**Terminal 2 — Frontend**

```bash
cd ct-frontend
npm install
npm run dev
```

Frontend → http://localhost:5173  
Backend → http://localhost:8000

---

## Common Docker Commands

```bash
# Start all services (foreground)
docker compose up

# Start all services (background)
docker compose up -d

# Rebuild images after dependency changes
docker compose up --build

# Stop all services (keep DB data)
docker compose down

# Stop and wipe the database volume
docker compose down -v

# Tail logs for a specific service
docker compose logs -f backend
docker compose logs -f frontend

# Run a command inside a running container
docker compose exec backend poetry run pytest
docker compose exec backend poetry run alembic upgrade head
docker compose exec frontend npm run test

# Check running containers
docker compose ps
```

---

## Debugging (VS Code)

Backend supports live breakpoints via `debugpy` when `DEBUG=true`.

1. Start the stack: `docker compose up`
2. Open VS Code in this folder
3. Go to **Run & Debug** → select **"🐛 Attach to ct-backend (Docker)"** → press **F5**
4. Set breakpoints in `ct-backend/app/` — they trigger on the next API call

Port `5678` on the host is mapped to `debugpy` inside the container.

---

## Git & Submodule Reference

### Working on this monorepo

```bash
# Stage and commit monorepo changes
git add <file>
git commit -m "type: description"
git push origin main

# Create a feature branch
git checkout -b feat/my-change
git push origin feat/my-change
```

### Working on a submodule

```bash
# Enter the submodule and work normally
cd ct-backend
git checkout -b feat/my-backend-feature
# make changes, run tests, commit
git push origin feat/my-backend-feature
cd ..

# After the submodule branch is merged, bump the pointer in the monorepo
git submodule update --remote ct-backend   # pull latest main
git add ct-backend
git commit -m "chore: bump ct-backend to latest"
git push origin main
```

### Useful submodule commands

```bash
# Show which commit each submodule currently points to
git submodule status

# Update all submodules to latest commit on their tracked branch
git submodule update --remote --merge

# Initialize submodules after a fresh clone (if --recurse-submodules was forgotten)
git submodule update --init --recursive
```

---

## Project Structure

```
the-creator-tools/
├── ct-backend/                  # Git submodule → github.com/prateek099/ct-backend
│   ├── app/
│   │   ├── main.py              # FastAPI app factory
│   │   ├── api/routes/          # auth.py, users.py
│   │   ├── core/                # config, database, security, logging, exceptions
│   │   ├── middleware/          # request_id, timing, logging
│   │   ├── models/              # SQLAlchemy User model
│   │   ├── schemas/             # Pydantic schemas
│   │   └── services/            # auth_service, user_service
│   ├── tests/
│   ├── Dockerfile               # multi-stage: base / development / production
│   ├── pyproject.toml
│   └── README.md
│
├── ct-frontend/                 # Git submodule → github.com/prateek099/ct-frontend
│   ├── src/
│   │   ├── api/                 # client.ts (axios + interceptors), useAuth, useUsers
│   │   ├── components/          # Navbar, ProtectedRoute
│   │   ├── context/             # AuthContext
│   │   ├── pages/               # LoginPage, UsersPage
│   │   └── types/               # auth.ts, user.ts
│   ├── Dockerfile               # multi-stage: development (Vite) / production (nginx)
│   ├── nginx.conf               # SPA routing + /api proxy
│   └── package.json
│
├── .claude/rules/               # Claude Code scoped rules (loaded per file type)
│   ├── backend.md               # rules for ct-backend/**/*.py
│   ├── frontend.md              # rules for ct-frontend/**/*.{ts,tsx}
│   └── deployment.md            # rules for Dockerfiles, docker-compose, render.yaml
│
├── .vscode/
│   ├── launch.json              # "Attach to ct-backend (Docker)" debug config
│   └── extensions.json          # recommended VS Code extensions
│
├── docker-compose.yml           # local dev — hot-reload, debugpy, Postgres
├── docker-compose.prod.yml      # production-like build test
├── render.yaml                  # Render deployment — backend + frontend + DB
├── CLAUDE.md                    # Claude Code project memory and conventions
└── README.md                    # this file
```

---

## API Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/health` | — | Health check |
| `POST` | `/api/v1/auth/register` | — | Register a new user |
| `POST` | `/api/v1/auth/login` | — | Login → access + refresh tokens |
| `POST` | `/api/v1/auth/refresh` | — | Exchange refresh token for new pair |
| `GET` | `/api/v1/auth/me` | Bearer | Current user profile |
| `GET` | `/api/v1/users/` | Bearer | List all users |
| `POST` | `/api/v1/users/` | Bearer | Create a user |
| `GET` | `/api/v1/users/{id}` | Bearer | Get user by ID |
| `DELETE` | `/api/v1/users/{id}` | Bearer | Delete user by ID |

---

## Deployment (Render)

The `render.yaml` at the repo root defines the full deployment:

- **ct-backend** — Docker web service, health check at `/health`, DB credentials injected automatically
- **ct-frontend** — Docker web service running nginx, `VITE_API_URL` set to the backend URL
- **ct-db** — Render managed PostgreSQL 16

To deploy:
1. Connect this repo to your Render account
2. Render detects `render.yaml` and creates all services automatically
3. Set `JWT_SECRET_KEY` and other secrets in the Render dashboard
4. Push to `main` — Render auto-deploys on every push

---

## Troubleshooting

**`docker daemon not running`**

```bash
open -a Docker        # Mac — start Docker Desktop
docker info           # verify it's running
```

**`Submodule directory is empty after clone`**

```bash
git submodule update --init --recursive
```

**`port already in use`**

```bash
# Mac / Linux
lsof -ti:8000 | xargs kill   # backend
lsof -ti:5173 | xargs kill   # frontend
lsof -ti:5432 | xargs kill   # postgres

# Windows PowerShell
netstat -ano | findstr :8000
taskkill /PID <pid> /F
```

**`ct-backend/.env not found`**

```bash
cp ct-backend/.env.example ct-backend/.env
```

**`/docs returns 404`**

Set `DEBUG=true` in `ct-backend/.env` and restart: `docker compose restart backend`
