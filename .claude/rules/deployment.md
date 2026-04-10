---
paths:
  - "ct-backend/Dockerfile"
  - "ct-frontend/Dockerfile"
  - "ct-frontend/nginx.conf"
  - "docker-compose*.yml"
  - "render.yaml"
  - ".dockerignore"
---

# Deployment Rules — Docker + Poetry + Render

## Docker — Backend (Poetry)
- **Multi-stage build**: `base` → `development` → `builder` → `production`
- Use `python:3.12-slim` as base image
- Export deps with `poetry export --without-hashes -f requirements.txt`
- **Never install Poetry in production image** — only `requirements.txt`
- Run as non-root user (`appuser`) in production
- Healthcheck must hit `GET /health` endpoint

## Stage targets
| Target | Used by | Features |
|--------|---------|----------|
| `development` | `docker-compose.yml` | All deps, `--reload`, debugpy |
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
  - /app/.venv            # preserve venv inside container
```

## Render (`render.yaml`)
- Backend: `type: web`, `runtime: docker`, `healthCheckPath: /health`
- Frontend: `type: web`, `runtime: docker`
- DB credentials via `fromDatabase` — never hardcode
- All secrets in Render dashboard environment variables

## .dockerignore (always exclude)
```
.env
__pycache__/
*.pyc
.git/
tests/
*.md
node_modules/
dist/
```

## Checklist before deploying
- [ ] `DEBUG=false` in production env
- [ ] `DATABASE_URL` points to Render managed DB
- [ ] CORS `allow_origins` updated to production frontend URL
- [ ] `VITE_API_URL` points to production backend URL
- [ ] Health check passes locally before pushing
