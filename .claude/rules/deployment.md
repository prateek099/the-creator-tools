---
paths:
  - "ct-backend/Dockerfile"
  - "ct-frontend/Dockerfile"
  - "ct-frontend/nginx.conf"
  - "docker-compose*.yml"
  - "render.yaml"
---

# Deployment Rules — Docker + Poetry + Render

## Docker — Backend (Poetry)
- **Multi-stage build**: `base` → `development` → `builder` → `production`
- Use `python:3.12-slim` as base (satisfies `python = "^3.10"` constraint)
- Export deps with `poetry export --without-hashes -f requirements.txt`
- **Never install Poetry in production image** — only `requirements.txt`
- Run as non-root user (`appuser`) in production
- Healthcheck must hit `GET /health` endpoint
- PostgreSQL driver: `psycopg[binary]` — no system libpq needed on Debian-slim

## Stage targets

| Target | Used by | Features |
|--------|---------|----------|
| `development` | `docker-compose.yml` | All deps, `--reload`, debugpy on 5678 |
| `production` | `docker-compose.prod.yml` / Render | Slim, no dev deps, appuser |

## Docker — Frontend
- Dev: Node 20 alpine with Vite HMR
- Prod: multi-stage, Nginx alpine serves built `/dist`
- `nginx.conf` proxies `/api/` → `http://backend:8000/api/`
- React Router needs `try_files $uri $uri/ /index.html`

## Volume Mounts (local dev only)
```yaml
volumes:
  - ./ct-backend:/app     # live code sync — no rebuild on save
  - /app/.venv            # preserve Poetry venv inside container
```

## docker-compose service names
- `backend` — FastAPI app (ports 8000, 5678)
- `frontend` — Vite dev / nginx prod (port 5173 dev, 80 prod)
- `db` — Postgres 16 alpine (port 5432)

## Render (`render.yaml`)
- Backend: `type: web`, `runtime: docker`, `healthCheckPath: /health`
- Frontend: `type: web`, `runtime: docker`
- DB credentials via `fromDatabase` — never hardcode
- All secrets in Render dashboard environment variables

## Key environment variables for production

| Variable | Production value |
|----------|-----------------|
| `DEBUG` | `false` |
| `ENVIRONMENT` | `production` |
| `DATABASE_URL` | `postgresql+psycopg://` from Render managed DB |
| `JWT_SECRET_KEY` | `openssl rand -hex 32` output |
| `CORS_ORIGINS` | production frontend URL (e.g. `https://ct-frontend.onrender.com`) |
| `LOG_FORMAT` | `json` |

## Checklist before deploying
- [ ] `DEBUG=false` in production env
- [ ] `DATABASE_URL` points to Render managed DB (`postgresql+psycopg://` prefix)
- [ ] `CORS_ORIGINS` env var updated to production frontend URL
- [ ] `VITE_API_URL` points to production backend URL
- [ ] `JWT_SECRET_KEY` is a securely generated secret (not the default)
- [ ] Health check passes locally before pushing: `curl http://localhost:8000/health`
