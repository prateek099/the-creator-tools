# the-creator-tools

Full-stack monorepo for **Creator Tools** — a FastAPI + React application with JWT authentication, AI-powered content generation, LLM usage tracking, PostgreSQL, and Docker-based local development. Deployed to [Render](https://render.com) via `render.yaml`.

| | |
|---|---|
| **Backend** | Python 3.10+, FastAPI, SQLAlchemy 2.0, Poetry |
| **Frontend** | React 18, TypeScript, Vite, React Query |
| **Database** | SQLite (local dev) / PostgreSQL 16 (Docker + production) |
| **Auth** | JWT — access token (30 min) + refresh token (7 days), stored in browser cookies |
| **AI** | OpenAI GPT-4o-mini — video ideas, scripts, titles, SEO descriptions |
| **Deploy** | Docker → Render |

---

## Repositories

| Repo | Description | Link |
|------|-------------|------|
| **the-creator-tools** | This monorepo — docker-compose, render.yaml, CLAUDE.md rules | [github.com/prateek099/the-creator-tools](https://github.com/prateek099/the-creator-tools) |
| **ct-backend** | FastAPI backend — auth, AI routes, LLM tracking, users API | [github.com/prateek099/ct-backend](https://github.com/prateek099/ct-backend) |
| **ct-frontend** | React frontend — login, auth context, tool pages, logout widget | [github.com/prateek099/ct-frontend](https://github.com/prateek099/ct-frontend) |

`ct-backend` and `ct-frontend` are **git submodules** of this repo — each has its own independent commit history and can be developed and deployed separately.

---

## Architecture

```
                        ┌─────────────────────┐
                        │   Browser / Client   │
                        └────────┬────────────┘
                                 │ HTTP  (Bearer token via cookie)
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
                    │  Auth → AI Routes →      │
                    │  LLM Tracker             │
                    │                          │
                    │  Middleware stack:        │
                    │  CORS → RequestID →       │
                    │  Timing → ReqLogging     │
                    └──────┬──────────┬────────┘
                           │          │
              SQLAlchemy   │          │  OpenAI API
                    ┌──────▼──┐  ┌────▼────────┐
                    │ SQLite  │  │  OpenAI     │
                    │ / Postgres  │  GPT-4o-mini│
                    └─────────┘  └─────────────┘
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
OPENAI_API_KEY=sk-...         # required for AI generation endpoints
YOUTUBE_API_KEY=AIza...       # required for YouTube channel fetch
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

## API Endpoints

### Auth (public)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/health` | Health check |
| `POST` | `/api/v1/auth/register` | Register a new user → returns user profile |
| `POST` | `/api/v1/auth/login` | Login with email + password → `{name, access_token, refresh_token}` |
| `POST` | `/api/v1/auth/refresh` | Exchange refresh token for a new token pair |
| `POST` | `/api/v1/login` | Workflow login (base64-encoded credentials) → same token response |

### Auth (protected — Bearer required)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/auth/me` | Current user profile |
| `GET` | `/api/v1/users/` | List all users |
| `POST` | `/api/v1/users/` | Create a user |
| `GET` | `/api/v1/users/{id}` | Get user by ID |
| `DELETE` | `/api/v1/users/{id}` | Delete user by ID |

### AI Tools (Bearer required — usage recorded to `llm_usage` table)

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/video-idea-gen` | Generate 10 video ideas from a prompt + optional channel context |
| `POST` | `/api/v1/script-generator` | Generate a full YouTube script |
| `POST` | `/api/v1/title-suggestor` | Suggest 10 video titles with CTR analysis |
| `POST` | `/api/v1/seo-description` | Generate SEO description, hashtags, and tags |
| `POST` | `/api/v1/yt/channel` | Fetch YouTube channel metadata and recent videos |

---

## Authentication — Sending the Bearer Token

After login, the server returns an `access_token`. All protected endpoints require it in the `Authorization` header.

### JavaScript / Axios (frontend — handled automatically)

The frontend `client.ts` reads the token from the `access_token` cookie and attaches it on every request automatically. No manual work needed.

```ts
// src/api/client.ts — interceptor already does this for every call
config.headers.Authorization = `Bearer ${Cookies.get("access_token")}`;
```

### curl

```bash
# 1. Login and capture the token
TOKEN=$(curl -s -X POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"you@example.com","password":"yourpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

# 2. Use it on any protected endpoint
curl -X POST http://localhost:8000/api/v1/video-idea-gen \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Python tutorials for beginners"}'
```

### Python (requests)

```python
import requests

BASE = "http://localhost:8000/api/v1"

# Login
resp = requests.post(f"{BASE}/auth/login", json={"email": "you@example.com", "password": "yourpassword"})
token = resp.json()["access_token"]

headers = {"Authorization": f"Bearer {token}"}

# Call an AI endpoint
ideas = requests.post(f"{BASE}/video-idea-gen",
    headers=headers,
    json={"prompt": "Python tutorials for beginners"}
).json()
```

### Workflow login (base64-encoded, used by the UI)

```bash
python3 -c "import base64; print(base64.b64encode(b'your@email.com').decode())"
python3 -c "import base64; print(base64.b64encode(b'yourpassword').decode())"

curl -X POST http://localhost:8000/api/v1/login \
  -H "Content-Type: application/json" \
  -d '{"username": "<base64_email>", "password": "<base64_password>"}'
```

---

## Database Schema

Tables are created automatically on server startup via `Base.metadata.create_all()`. No migrations needed for local dev.

### `users`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | INTEGER | PK, indexed | Auto-increment primary key |
| `name` | VARCHAR(100) | NOT NULL | Display name |
| `email` | VARCHAR(255) | UNIQUE, NOT NULL, indexed | Login identifier |
| `hashed_password` | VARCHAR(255) | NOT NULL | bcrypt hash |
| `is_active` | BOOLEAN | NOT NULL, default `true` | Account enabled flag |

### `llm_usage`

One row is written for every OpenAI API call, regardless of success or failure.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | INTEGER | PK, indexed | Auto-increment primary key |
| `user_id` | INTEGER | FK → users.id, nullable, indexed | Null for demo / system calls |
| `username` | VARCHAR(100) | NOT NULL, indexed | User's name, or `"SYSTEM_USAGE"` for demo tokens |
| `endpoint` | VARCHAR(100) | NOT NULL, indexed | Calling route: `video_idea_gen`, `script_generator`, `title_suggestor`, `seo_description` |
| `model` | VARCHAR(100) | NOT NULL | OpenAI model used, e.g. `gpt-4o-mini` |
| `system_prompt` | TEXT | nullable | System-level prompt sent to OpenAI |
| `user_prompt` | TEXT | NOT NULL | User-facing prompt sent to OpenAI |
| `response_text` | TEXT | nullable | Raw response string from OpenAI (before JSON parse) |
| `prompt_tokens` | INTEGER | nullable | Tokens consumed by the prompt |
| `completion_tokens` | INTEGER | nullable | Tokens consumed by the completion |
| `total_tokens` | INTEGER | nullable | Total tokens billed |
| `duration_ms` | INTEGER | nullable | Wall-clock ms from call start to response |
| `status` | VARCHAR(20) | NOT NULL | `"success"` or `"error"` |
| `error_message` | TEXT | nullable | Exception message if `status = "error"` |
| `created_at` | DATETIME | NOT NULL, indexed, server default | UTC timestamp of the call |

---


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



## Handling Git Submodules

Each submodule (`ct-backend`, `ct-frontend`) is a fully independent git repo. The parent repo only stores a **commit SHA pointer** — you work inside the submodule like any normal repo, then bump the pointer in the parent separately.

### Making changes to a submodule

```bash
# 1. Enter the submodule
cd ct-backend          # or ct-frontend

# 2. Always work on a branch — never in detached HEAD
git checkout main      # or: git checkout -b feat/my-feature

# 3. Make changes, stage, commit normally
git add <files>
git commit -m "feat: my change"

# 4. Push the submodule to its own remote
git push origin main   # or feat/my-feature

# 5. Go back to monorepo root and bump the pointer
cd ..
git add ct-backend
git commit -m "chore: bump ct-backend to latest"
git push origin main
```

> Step 5 is always required after pushing inside the submodule — otherwise others who pull the parent get the old version.

### Pulling latest (someone else pushed changes)

```bash
# Pull parent + update all submodule pointers in one command
git pull --recurse-submodules

# Or if you already pulled the parent:
git submodule update --remote --merge
```

### Working on this monorepo (non-submodule files)

```bash
# Stage and commit monorepo changes (docker-compose, render.yaml, etc.)
git add <file>
git commit -m "type: description"
git push origin main

# Create a feature branch
git checkout -b feat/my-change
git push origin feat/my-change
```

### Quick status check

```bash
git submodule status
# + means ahead of the recorded pointer
# - means not initialized
# (clean) means in sync
```

### Common pitfalls

| Mistake | What happens | Fix |
|---------|-------------|-----|
| Push submodule but forget to bump parent | CI and others still get the old code | Always `git add <submodule> && git commit` in the parent after pushing inside |
| Work in detached HEAD inside a submodule | Commits exist but aren't on any branch — lost after `submodule update` | Always `git checkout main` (or a branch) before making changes |
| Pull parent without updating submodules | Submodule source stays at old SHA despite the pointer changing | Use `git pull --recurse-submodules` |
| Commit submodule pointer before pushing the submodule | Others get "object not found" when they run `submodule update` | Push the submodule remote **before** bumping the parent pointer |

---

## Project Structure

```
the-creator-tools/
├── ct-backend/                  # Git submodule → github.com/prateek099/ct-backend
│   ├── app/
│   │   ├── main.py              # FastAPI app factory
│   │   ├── api/
│   │   │   ├── deps.py          # get_current_user, get_optional_user, require_valid_token
│   │   │   └── routes/          # auth, users, login, video_idea_gen, script_generator,
│   │   │                        # title_suggestor, seo_description, youtube
│   │   ├── api_wrappers/        # open_ai.py (openai_call + openai_wrapper), youtube.py
│   │   ├── core/                # config (dotenv), database, security, logging, exceptions, messages
│   │   ├── middleware/          # request_id, timing, logging
│   │   ├── models/              # user.py, llm_usage.py
│   │   ├── prompts/             # ideas.py, script.py, title.py, seo.py (LLM prompt templates)
│   │   ├── schemas/             # Pydantic schemas (incl. ai.py with unified ChannelContext)
│   │   └── services/            # auth_service, user_service, llm_tracker
│   ├── tests/                   # includes test_ai_routes.py + test_deps.py
│   ├── Dockerfile
│   ├── pyproject.toml
│   └── README.md
│
├── ct-frontend/                 # Git submodule → github.com/prateek099/ct-frontend
│   ├── src/
│   │   ├── api/                 # client.ts (axios + cookie Bearer), useAuth, useWorkflow
│   │   ├── components/
│   │   │   ├── auth/            # ProtectedRoute
│   │   │   ├── layout/          # AppShell, Sidebar, TopBar, Navbar, PageHeader
│   │   │   ├── pipeline/        # PipelineStepper, BackgroundGenerationBanner,
│   │   │   │                    # NoIdeaSelectedCard, ContextBanner
│   │   │   └── shared/          # Icon, UserCard, UserWidget
│   │   ├── context/             # AuthContext, WorkflowContext
│   │   ├── pages/
│   │   │   ├── idea/            # IdeaSidebar
│   │   │   ├── script/          # FlavorPicker, ReviewPanel
│   │   │   └── *.tsx            # Route-level pages (one per route)
│   │   └── types/               # auth.ts, user.ts, workflow.ts, api.ts (ApiError helper)
│   ├── Dockerfile
│   ├── nginx.conf
│   └── package.json
│
├── .claude/rules/               # Claude Code scoped rules
├── .vscode/                     # VS Code launch + extensions
├── docker-compose.yml           # local dev — hot-reload, debugpy, Postgres
├── render.yaml                  # Render deployment
├── CLAUDE.md                    # Claude Code project memory
└── README.md                    # this file
```

---

## Deployment (Render)

The `render.yaml` at the repo root defines the full deployment:

- **ct-backend** — Docker web service, health check at `/health`, DB credentials injected automatically
- **ct-frontend** — Docker web service running nginx, `VITE_API_URL` set to the backend URL
- **ct-db** — Render managed PostgreSQL 16

To deploy:
1. Connect this repo to your Render account
2. Render detects `render.yaml` and creates all services automatically
3. Set `JWT_SECRET_KEY`, `OPENAI_API_KEY`, `YOUTUBE_API_KEY` in the Render dashboard
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
