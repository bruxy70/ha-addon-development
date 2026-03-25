---
name: python-backend
description: Python backend development for Home Assistant apps. Use when writing or modifying Python/FastAPI backend code, database models, API routes, authentication, or service layers for HA apps (add-ons). Also use when discussing SQLAlchemy, Pydantic, JWT auth, or REST API patterns.
---

# Python Backend — Non-Obvious Reference

This skill covers Python backend patterns for HA apps using FastAPI + SQLAlchemy + SQLite. Only non-obvious details and common pitfalls — standard Python/FastAPI knowledge applies for everything else.

## 1. FastAPI with Ingress

HA apps served through ingress MUST handle the dynamic base path:

```python
import os

ingress_entry = os.environ.get("INGRESS_ENTRY", "/")

app = FastAPI(
    root_path=ingress_entry,
    docs_url="/docs",       # Available at {ingress_entry}/docs
    redoc_url=None,
)
```

**Static files and templates must also respect the base path:**
```python
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates

app.mount("/static", StaticFiles(directory="static"), name="static")
templates = Jinja2Templates(directory="templates")

# In templates, use: {{ request.scope.get('root_path', '') }}/static/...
```

## 2. SQLite for HA Apps

**Database location:** Always use `/data/` for persistence across updates.

```python
DATABASE_URL = "sqlite:///data/myapp.db"
# NOT /config/data/ unless users need direct file access
```

**SQLite concurrency — the main gotcha:**
```python
from sqlalchemy import create_engine

engine = create_engine(
    DATABASE_URL,
    connect_args={"check_same_thread": False},  # Required for FastAPI
    pool_pre_ping=True,
)
```

**WAL mode for concurrent reads:**
```python
from sqlalchemy import event

@event.listens_for(engine, "connect")
def set_sqlite_pragma(dbapi_conn, connection_record):
    cursor = dbapi_conn.cursor()
    cursor.execute("PRAGMA journal_mode=WAL")
    cursor.execute("PRAGMA foreign_keys=ON")
    cursor.close()
```

**Single-writer limitation:** SQLite allows only one writer at a time. For high-write workloads, serialize writes through a queue or use `PRAGMA busy_timeout`.

## 3. Database Migrations

Use Alembic for schema migrations:

```python
# alembic/env.py — key settings for SQLite
context.configure(
    connection=connection,
    target_metadata=target_metadata,
    render_as_batch=True,    # Required for SQLite ALTER TABLE limitations
    compare_type=True,
)
```

**SQLite ALTER TABLE limitations:**
- Cannot DROP columns (before SQLite 3.35.0)
- Cannot ALTER column types
- Cannot add columns with non-constant defaults
- Alembic's `render_as_batch=True` works around these by recreating the table

## 4. Authentication Patterns

### JWT for web sessions:
```python
from datetime import datetime, timedelta
from jose import JWTError, jwt

SECRET_KEY = os.environ.get("JWT_SECRET", secrets.token_urlsafe(32))
ALGORITHM = "HS256"

def create_access_token(data: dict, expires_delta: timedelta = timedelta(hours=24)):
    to_encode = data.copy()
    to_encode["exp"] = datetime.utcnow() + expires_delta
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

async def get_current_user(token: str = Depends(oauth2_scheme)):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id: int = payload.get("sub")
        if user_id is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception
    return get_user(user_id)
```

### API key for external integrations (e.g., N8N, webhooks):
```python
from fastapi.security import HTTPBearer

bearer_scheme = HTTPBearer()

async def verify_api_key(credentials = Depends(bearer_scheme)):
    api_key = credentials.credentials
    if not secrets.compare_digest(api_key, stored_key_hash):
        raise HTTPException(status_code=401, detail="Invalid API key")
    return api_key
```

**Always use `secrets.compare_digest()` for token comparison** — prevents timing attacks.

### Supervisor token for HA API access:
```python
SUPERVISOR_TOKEN = os.environ.get("SUPERVISOR_TOKEN")
# This is auto-injected and rotated — never store or cache it long-term
```

## 5. Pydantic Models — Gotchas

```python
from pydantic import BaseModel, Field, validator

class ItemCreate(BaseModel):
    name: str = Field(..., min_length=1, max_length=200)
    price: int = Field(..., ge=0)

    @validator('name')
    def name_stripped(cls, v):
        v = v.strip()
        if not v:
            raise ValueError('Name cannot be empty')
        return v

    class Config:
        from_attributes = True  # Pydantic v2 (was orm_mode in v1)
```

**Pydantic v1 vs v2 differences (HA apps may use either):**
- `orm_mode = True` → `from_attributes = True`
- `.dict()` → `.model_dump()`
- `.parse_obj()` → `.model_validate()`
- `@validator` → `@field_validator` (v2) but `@validator` still works

## 6. Error Handling

```python
from fastapi import HTTPException
from fastapi.responses import JSONResponse

# Specific errors
@app.exception_handler(ValueError)
async def value_error_handler(request, exc):
    return JSONResponse(status_code=400, content={"detail": str(exc)})

# Database integrity errors
from sqlalchemy.exc import IntegrityError

@app.exception_handler(IntegrityError)
async def integrity_error_handler(request, exc):
    return JSONResponse(status_code=409, content={"detail": "Duplicate entry"})
```

**Never expose internal error details in production** — log them, return a generic message.

## 7. Background Tasks

For periodic work (cleanup, sync, scheduled jobs):

```python
from contextlib import asynccontextmanager
import asyncio

async def periodic_cleanup():
    while True:
        await asyncio.sleep(3600)  # Every hour
        # Do cleanup work

@asynccontextmanager
async def lifespan(app: FastAPI):
    task = asyncio.create_task(periodic_cleanup())
    yield
    task.cancel()

app = FastAPI(lifespan=lifespan)
```

**Do NOT use `time.sleep()` in async context** — blocks the event loop.

## 8. CORS for Development

```python
from fastapi.middleware.cors import CORSMiddleware

if os.environ.get("ENVIRONMENT") == "development":
    app.add_middleware(
        CORSMiddleware,
        allow_origins=["http://localhost:3000"],
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )
```

**In production through ingress, CORS is not needed** — same origin.

## 9. Logging

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
)
logger = logging.getLogger(__name__)

# Use structured data, not f-strings with sensitive data
logger.info("User %s logged in", user_id)       # ✅
logger.info(f"User {user.email} logged in")      # ❌ PII in logs
```

**HA app logs are visible in the UI** — never log passwords, tokens, or PII.

## 10. Graceful Shutdown

```python
import signal

def handle_sigterm(*args):
    logger.info("Received SIGTERM, shutting down...")
    # Close database connections, flush buffers
    sys.exit(0)

signal.signal(signal.SIGTERM, handle_sigterm)
```

HA Supervisor sends SIGTERM when stopping an app. If the process doesn't exit within 10 seconds, SIGKILL follows.
