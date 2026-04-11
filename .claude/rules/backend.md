---
paths:
  - "ct-backend/**/*.py"
---

# Backend Rules — FastAPI + Poetry

## Stack
- Python **3.10+** (3.10 · 3.11 · 3.12), FastAPI, SQLAlchemy 2.0, Alembic, Pydantic v2
- `python-jose[cryptography]` — JWT, `bcrypt` (direct, no passlib) — password hashing
- `psycopg[binary] ^3.1` — PostgreSQL driver (psycopg3); SQLite used by default in dev
- `loguru` — structured logging, `slowapi` — rate limiting (200 req/min)
- Dependency management via **Poetry only** — never use `pip install` directly
- `passlib` is **not used** — it is incompatible with `bcrypt >= 4.x`; use `bcrypt` directly

## Project Layout
```
ct-backend/
├── app/
│   ├── main.py                  # App factory — middleware, routes, handlers wired here
│   ├── api/
│   │   ├── deps.py              # get_current_user dependency
│   │   └── routes/
│   │       ├── auth.py          # /register /login /refresh /me
│   │       └── users.py         # CRUD — all protected
│   ├── core/
│   │   ├── config.py            # pydantic-settings (JWT, CORS, logging)
│   │   ├── database.py          # engine + get_db
│   │   ├── security.py          # hash_password, verify_password, JWT helpers
│   │   ├── logging.py           # loguru setup + stdlib intercept
│   │   └── exceptions.py        # AppError subclasses + register_exception_handlers()
│   ├── middleware/
│   │   ├── request_id.py        # X-Request-ID on every request
│   │   ├── timing.py            # X-Process-Time header
│   │   └── logging.py           # per-request structured log
│   ├── models/                  # SQLAlchemy ORM (User has hashed_password, is_active)
│   ├── schemas/                 # Pydantic: auth.py (Login/Register/Token) + user.py
│   └── services/
│       ├── auth_service.py      # register(), login(), refresh_tokens()
│       └── user_service.py      # CRUD — raises AppError subclasses, never HTTPException
└── tests/
    ├── conftest.py              # shared fixtures: client, registered_user, auth_headers
    ├── test_auth.py
    └── test_users.py
```

## Auth Rules
- JWT access token: 30 min, refresh token: 7 days
- All routes (except `/health`, `/auth/*`) require `Depends(get_current_user)`
- `get_current_user` is in `app/api/deps.py` — inject via `Depends`
- Never store tokens in DB — stateless JWT only
- `JWT_SECRET_KEY` must come from env — never hardcode

## Exception Handling
- Raise **AppError subclasses** (`NotFoundError`, `ConflictError`, `UnauthorizedError`, etc.) from services
- **Never raise `HTTPException`** from services — only from routes if truly needed
- Global handlers in `app/core/exceptions.py` convert every error to `{"error": {"code": ..., "detail": ...}}`
- All handlers include `request_id` from `request.headers["X-Request-ID"]`
- Unhandled exceptions are caught, logged with `logger.exception`, return 500

## Logging
- Use `from loguru import logger` everywhere — never `import logging`
- Call `setup_logging()` once in `main.py` before anything else
- Pass structured fields as kwargs: `logger.info("msg", user_id=1, path="/users")`
- `LOG_FORMAT=pretty` locally, `LOG_FORMAT=json` in production
- Do NOT log passwords, tokens, or PII

## Middleware (order matters — last added = outermost)
1. `CORSMiddleware` — must be outermost
2. `RequestIDMiddleware` — assigns X-Request-ID
3. `TimingMiddleware` — measures duration
4. `RequestLoggingMiddleware` — logs method/path/status/duration

## Code Style
- **Black** (line-length 88), **isort** `profile = "black"`
- Type hints required on all signatures, `async def` for all routes
- Google-style docstrings for public functions

## Commenting Guidelines
- All comments in Python code should start with " Prateek: "
- Use inline comments sparingly and only when the code is not self-explanatory
- Ensure comments are concise and add value to the understanding of the code

## API Patterns
- Routes in `app/api/routes/`, registered via `APIRouter`
- Always use Pydantic schemas for request body + `response_model`
- HTTP status codes: 201 create, 204 delete, 404 not found, 409 conflict, 401 unauth

## Testing
- Run: `poetry run pytest`
- Fixtures from `conftest.py`: `client`, `registered_user`, `auth_headers`
- Override `get_db` with in-memory SQLite via `app.dependency_overrides`
- Target: 85%+ coverage

## Debugging
- `debugpy` starts when `DEBUG=true`, listens on `0.0.0.0:5678`
- Attach VS Code via `.vscode/launch.json` → "Attach to ct-backend (Docker)"

