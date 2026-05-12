---
name: fastapi-new-project
description: Scaffolds a fresh FastAPI project from scratch. Use this when a user wants to start a new FastAPI project, set up the folder structure, or bootstrap a new API. Trigger on phrases like "new fastapi project", "setup fastapi", "scaffold an API", or "start a new API".
---

# FastAPI New Project Setup

You are setting up a brand new FastAPI project. Follow every step below in order. Do not skip any step.

---

## Step 1 — Run Terminal Commands

Run each of the following commands one at a time in the project terminal:

```
python3 -m venv .venv
source .venv/bin/activate
pip install 'fastapi[all]' 'uvicorn[standard]' 'slowapi' 'python-jose[cryptography]' 'passlib[argon2]'
pip freeze > requirements.txt
```

---

## Step 2 — Create the Folder Structure

Create the following files and directories exactly as shown:

```
├── app/
│   ├── api/
│   │   └── v1/
│   │       └── routes.py
│   ├── services/
│   │   └── example_service.py
│   ├── core/
│   │   ├── config.py
│   │   ├── dependencies.py
│   │   └── exceptions.py
│   ├── models/
│   └── schemas/
│   └── main.py
├── sql/
│   ├── tables/
│   ├── functions/
│   └── migrations/
├── .venv/
├── .env
├── .env.example
├── .gitignore
├── requirements.txt
└── README.md
```

---

## Step 3 — Write the File Contents

### `app/core/config.py`
```python
from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    app_name: str = "FastAPI App"
    environment: str = "development"
    secret_key: str
    algorithm: str = "HS256"
    access_token_expire_minutes: int = 15
    allowed_origins: list[str] = []
    allowed_redirect_hosts: list[str] = []
    upload_dir: str = "/tmp/uploads"

    class Config:
        env_file = ".env"


settings = Settings()
```

### `app/core/exceptions.py`
```python
class AppBaseException(Exception):
    pass


class ResourceNotFound(AppBaseException):
    pass


class Unauthorized(AppBaseException):
    pass


class DatabaseError(AppBaseException):
    pass


class ExternalAPIError(AppBaseException):
    pass


class UsageLimitExceeded(AppBaseException):
    pass


class ValidationError(AppBaseException):
    pass
```

### `app/core/dependencies.py`
```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, jwt
from app.core.config import settings

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")


async def get_current_user(token: str = Depends(oauth2_scheme)) -> dict:
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, settings.secret_key, algorithms=[settings.algorithm])
        user_id: str = payload.get("sub")
        if user_id is None:
            raise credentials_exception
        return {"user_id": user_id}
    except JWTError:
        raise credentials_exception
```

### `app/api/v1/routes.py`
```python
from fastapi import APIRouter

router = APIRouter()
```

### `app/services/example_service.py`
```python
# Services contain all business logic and DB calls.
# Routers call services — services never import from fastapi.

def get_example() -> dict:
    return {"message": "example service response"}
```

### `app/main.py`
```python
import logging
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
from starlette.middleware.base import BaseHTTPMiddleware
from app.api.v1.routes import router
from app.core.config import settings

logger = logging.getLogger(__name__)

limiter = Limiter(key_func=get_remote_address)


class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        response = await call_next(request)
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains; preload"
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
        response.headers["Content-Security-Policy"] = (
            "default-src 'self'; script-src 'self'; frame-ancestors 'none'; base-uri 'self';"
        )
        return response


app = FastAPI(
    title=settings.app_name,
    docs_url=None if settings.environment == "production" else "/docs",
    redoc_url=None if settings.environment == "production" else "/redoc",
)

app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)
app.add_middleware(SecurityHeadersMiddleware)
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.allowed_origins,
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
)

app.include_router(router, prefix="/api/v1")


@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    logger.exception("Unhandled exception: %s %s", request.method, request.url)
    return JSONResponse(status_code=500, content={"detail": "Internal server error"})


@app.get("/")
def root():
    return {"message": "API is running"}


@app.get("/health")
def health_check():
    return {"status": "ok"}
```

### `.env`
```
APP_NAME=FastAPI App
ENVIRONMENT=development
SECRET_KEY=changeme-generate-with-secrets.token_hex(32)
ALLOWED_ORIGINS=["http://localhost:3000"]
```

### `.env.example`
```
APP_NAME=FastAPI App
ENVIRONMENT=development
SECRET_KEY=
ALLOWED_ORIGINS=[]
```

### `.gitignore`
```
.venv/
__pycache__/
*.pyc
.env
*.egg-info/
dist/
.DS_Store
```

### `README.md`
````markdown
# FastAPI Project

## Setup

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

## Environment

Copy `.env.example` to `.env` and fill in all values before running.
Generate a secret key with: `python3 -c "import secrets; print(secrets.token_hex(32))"`

## Run

```bash
uvicorn app.main:app --reload
```

## Endpoints

- `GET /` — API status
- `GET /health` — Health check
- `GET /docs` — Swagger UI (development only)
````

---

## Step 4 — Verify

After creating all files, confirm the following:

- [ ] `.env` exists and is NOT committed (check `.gitignore`)
- [ ] `SECRET_KEY` in `.env` is set to a real value — run `python3 -c "import secrets; print(secrets.token_hex(32))"`
- [ ] `requirements.txt` is populated from `pip freeze`
- [ ] Running `uvicorn app.main:app --reload` starts the server without errors
- [ ] `GET /` returns `{"message": "API is running"}`
- [ ] `GET /health` returns `{"status": "ok"}`
- [ ] `GET /docs` loads the Swagger UI

---

## Architecture Reminders

- Routers are thin — one service call, 5–15 lines max, no business logic
- Services own all logic and DB calls — never import `fastapi` in a service
- All secrets in `.env`, accessed via `settings` only — never `os.environ`
- Custom exceptions in `core/exceptions.py` — services raise them, routers convert to `HTTPException`
- SQL files go in `sql/tables/`, `sql/functions/`, `sql/migrations/` — never inline complex queries
- Swagger `/docs` is disabled in production automatically via `settings.environment`
