---
paths:
  - "ct-backend/**/*.py"
---

# Backend Rules — FastAPI + Poetry

## Stack
- Python 3.12, FastAPI, SQLAlchemy 2.0, Alembic, Pydantic v2
- Dependency management via **Poetry only** — never use `pip install` directly

## Project Layout
```
ct-backend/
├── app/
│   ├── main.py              # FastAPI app + debugpy + CORS
│   ├── api/routes/          # One file per resource (users.py, items.py …)
│   ├── core/
│   │   ├── config.py        # Settings via pydantic-settings
│   │   └── database.py      # SQLAlchemy engine + get_db dependency
│   ├── models/              # SQLAlchemy ORM models
│   ├── schemas/             # Pydantic request/response schemas
│   └── services/            # Business logic — no DB code in routes
└── tests/                   # Mirror app/ structure
```

## Code Style
- **Black** formatter (line-length 88), **isort** with `profile = "black"`
- Type hints required on all function signatures
- Use `async def` for all route handlers
- Google-style docstrings for public functions

## API Patterns
- Routes live in `app/api/routes/`, registered via `APIRouter` in `main.py`
- Always use Pydantic schemas for request body + response model
- Return explicit HTTP status codes: 201 create, 204 delete, 404 not found
- Inject DB session: `db: Session = Depends(get_db)`
- Never put business logic in route handlers — delegate to `services/`

## Error Handling
- Raise `HTTPException` with clear `detail` messages
- Never expose stack traces or internal errors to clients

## Testing
- Run: `poetry run pytest`
- Use `TestClient` (httpx) for route tests
- Override `get_db` with in-memory SQLite for isolation
- Target: 85%+ coverage

## Debugging
- `debugpy` auto-starts when `DEBUG=true` env var is set
- Listens on `0.0.0.0:5678` — attach from VS Code
