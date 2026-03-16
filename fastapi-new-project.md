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
pip install 'fastapi[all]' 'uvicorn[standard]'
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
│   ├── schemas/
│   └── main.py
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
```

### `app/core/dependencies.py`
```python
# Add shared FastAPI dependencies here (e.g. get_current_user)
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
from fastapi import FastAPI
from app.api.v1.routes import router
from app.core.config import settings

app = FastAPI(
    title=settings.app_name,
    docs_url=None if settings.environment == "production" else "/docs",
    redoc_url=None if settings.environment == "production" else "/redoc",
)

app.include_router(router, prefix="/api/v1")


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
```

### `.env.example`
```
APP_NAME=FastAPI App
ENVIRONMENT=development
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
```markdown
# FastAPI Project

## Setup

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

## Run

```bash
uvicorn app.main:app --reload
```

## Endpoints

- `GET /` — API status
- `GET /health` — Health check
- `GET /docs` — Swagger UI (development only)
```

---

## Step 4 — Verify

After creating all files, confirm the following:

- [ ] `.env` exists and is NOT committed (check `.gitignore`)
- [ ] `requirements.txt` is populated from `pip freeze`
- [ ] Running `uvicorn app.main:app --reload` starts the server without errors
- [ ] `GET /` returns `{"message": "API is running"}`
- [ ] `GET /health` returns `{"status": "ok"}`
- [ ] `GET /docs` loads the Swagger UI

---

## Architecture Reminders

- Routers are thin — one service call, 5-15 lines max, no business logic
- Services own all logic and DB calls — never import `fastapi` in a service
- All secrets live in `.env` and are accessed via `settings` only — never `os.environ`
- Custom exceptions live in `core/exceptions.py` — services raise them, routers convert them to `HTTPException`
- Swagger `/docs` is disabled in production automatically via `settings.environment`
