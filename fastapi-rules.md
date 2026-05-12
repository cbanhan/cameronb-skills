---
name: fastapi-rules
description: FastAPI coding rules, architecture standards, and security requirements. Covers routers, services, schemas, config, exceptions, SQL, auth, input validation, and pre-commit checklist. No ORMs — raw SQL or Supabase only. Security is built into every layer. Use when building or reviewing any FastAPI project.
---

# FastAPI Rules
> Directives for every file generated in this project.
> No ORMs. No SQLAlchemy. Raw SQL only. Production quality always. Security first — always.

---

## 🧠 Core Mental Model

| Layer | Responsibility | Rule |
|-------|---------------|------|
| Router | Define the endpoint | 5–15 lines max. One service call. Auth via `Depends()`. Nothing else. |
| Service | Do the work | All logic, all DB calls, all external APIs live here |
| Schema | Define the shape | Every input and output must be a typed Pydantic model with `extra = "forbid"` |
| Config | Manage secrets | Every value from `settings`. Never `os.environ` directly |
| Exceptions | Signal failures | Services raise custom exceptions. Routers convert them to HTTP errors |

**Security principle:** Validate all input server-side. Fail closed — deny by default, grant explicitly. Never trust client-supplied data, IDs, or roles.

---

## 📁 Structure & Naming

```
app/
├── api/v1/routes.py
├── services/{domain}_service.py
├── core/config.py
├── core/dependencies.py
├── core/exceptions.py
├── models/schemas/{domain}_schema.py
└── main.py

sql/
├── tables/
├── functions/
└── migrations/
```

**Naming rules:**
- Files: `{domain}_service.py`, `{domain}_schema.py`, `{domain}_routes.py`
- Functions: `snake_case` verbs — `get_`, `create_`, `delete_`, `update_`
- Schema classes: `PascalCase` nouns — `UserResponse`, `SearchRequest`
- SQL migrations: numeric prefix — `001_`, `002_` for explicit ordering

---

## 🟦 Router Checklist

When generating a router or endpoint, verify:

- [ ] Function body is 5–15 lines max
- [ ] Auth extracted via `Depends(get_current_user)` — never inline
- [ ] Calls exactly one service function
- [ ] `response_model` declared on every route decorator
- [ ] Only catches custom exceptions from `core/exceptions.py`
- [ ] Converts custom exceptions to `HTTPException` with the correct status code
- [ ] No SQL, no DB calls, no external API calls
- [ ] No business logic or conditionals beyond routing
- [ ] Rate limiting applied on auth, upload, and sensitive endpoints

**Status code ownership:**
- `401` → missing or invalid identity token
- `403` → plan or usage limit failure
- `404` → resource not found (also used for unauthorized resources — never expose 403 on owned data)
- `422` → Pydantic validation (automatic, never manual)
- `500` → unexpected DB or service failure

**Auth pattern (`core/dependencies.py`):**
```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, jwt
from core.config import settings

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

---

## 🟩 Service Checklist

When generating a service function, verify:

- [ ] Accepts plain Python types only — no `Request`, no FastAPI objects
- [ ] Returns a plain dict or Pydantic model — never a raw DB cursor or response object
- [ ] Raises only custom exceptions from `core/exceptions.py`
- [ ] Never imports from `fastapi`
- [ ] Never raises `HTTPException`
- [ ] All secrets accessed via `settings`, never hardcoded
- [ ] Logs the error with context before re-raising
- [ ] Function is single-responsibility and under ~50 lines
- [ ] SQL query logic and data transformation logic are in separate functions
- [ ] Ownership enforced inside the SQL `WHERE` clause — not checked after fetching
- [ ] UUIDs used for all resource identifiers — never sequential integers
- [ ] Parameterized queries only — no string formatting into SQL

**SQL rule:**
- Simple CRUD → Supabase `.table()` chaining is acceptable
- Complex queries (joins, aggregations, multi-step logic) → write a named Postgres function and call via `.rpc()`
- All SQL functions saved to `sql/functions/` for version control
- All DB calls stay in the service — never in the router

**Ownership pattern:**

Railway/Postgres:
```python
async def get_document(document_id: uuid.UUID, user_id: str) -> dict:
    async with get_db_connection() as conn:
        row = await conn.fetchrow(
            "SELECT id, title, content FROM documents WHERE id = $1 AND owner_id = $2",
            document_id,
            user_id,
        )
    if row is None:
        raise ResourceNotFound("Document not found")
    return dict(row)
```

Supabase:
```python
async def get_document(document_id: uuid.UUID, user_id: str) -> dict:
    response = (
        supabase.table("documents")
        .select("id, title, content")
        .eq("id", str(document_id))
        .eq("owner_id", user_id)
        .maybe_single()
        .execute()
    )
    if response.data is None:
        raise ResourceNotFound("Document not found")
    return response.data
```

**Dynamic ORDER BY — whitelist explicitly (cannot be parameterized):**
```python
ALLOWED_SORT_COLUMNS = {"created_at", "title", "updated_at"}
ALLOWED_SORT_ORDERS = {"asc", "desc"}

if sort_by not in ALLOWED_SORT_COLUMNS:
    raise ValueError("Invalid sort column")
```

---

## 🟨 Schema Checklist

When generating a schema, verify:

- [ ] Every request body has its own schema
- [ ] Every response shape has its own schema
- [ ] Request and response schemas are always separate classes — never reused
- [ ] All fields use explicit Python types — no bare `Any`
- [ ] Non-obvious fields have a `Field(description=...)` for Swagger
- [ ] No business logic or methods inside the schema class
- [ ] `extra = "forbid"` on all request schemas to block mass assignment
- [ ] `role`, `is_admin`, `is_superuser` are never in user-editable schemas
- [ ] UUIDs used for IDs (`uuid.UUID` type), not `int`
- [ ] String fields use `min_length`, `max_length`, and `strip_whitespace` where appropriate

**Schema pattern:**
```python
from pydantic import BaseModel, Field, validator
import re, uuid

class DocumentUpdateRequest(BaseModel):
    title: str = Field(..., min_length=1, max_length=200, strip_whitespace=True)
    content: str | None = Field(None, max_length=50_000)

    @validator("title")
    def title_no_script_tags(cls, v):
        if re.search(r"[<>\"';&]", v):
            raise ValueError("Title contains invalid characters")
        return v

    class Config:
        extra = "forbid"

class DocumentResponse(BaseModel):
    id: uuid.UUID
    title: str
```

---

## ⚙️ Config Checklist

When adding configuration or secrets:

- [ ] All values declared as typed fields in the `Settings` class
- [ ] `os.environ.get()` never used anywhere in the codebase
- [ ] New secret added to `.env.example` with a placeholder value immediately
- [ ] `.env` is in `.gitignore`
- [ ] App will crash at startup if a required value is missing — this is correct behaviour

**Security-critical fields that must always be in `Settings`:**
```python
secret_key: str          # 256+ bit random hex — generate with: secrets.token_hex(32)
algorithm: str = "HS256" # Never derive from token header
access_token_expire_minutes: int = 15
allowed_origins: list[str] = []
allowed_redirect_hosts: list[str] = []
upload_dir: str = "/tmp/uploads"
```

---

## ❌ Exceptions Checklist

When handling errors:

- [ ] Custom exception types defined in `core/exceptions.py` for every known failure category
- [ ] All custom exceptions inherit from a single `AppBaseException`
- [ ] Services raise custom exceptions — never `HTTPException`
- [ ] Routers catch custom exceptions and convert to `HTTPException`
- [ ] No bare `except Exception` without a `logger.error()` call and re-raise
- [ ] Generic error responses only — no stack traces, no DB details exposed to clients
- [ ] `RuntimeError` is acceptable as a fallback for fast iteration — but never catch it in the router

**Standard exception types:**
`ResourceNotFound`, `UsageLimitExceeded`, `ExternalAPIError`, `DatabaseError`, `Unauthorized`, `ValidationError`

**Global catch-all in `main.py`:**
```python
@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    logger.exception("Unhandled exception: %s %s", request.method, request.url)
    return JSONResponse(status_code=500, content={"detail": "Internal server error"})
```

---

## 🗄️ SQL Rules

- No ORMs, no SQLAlchemy — raw SQL or Supabase client only
- Simple CRUD → Supabase `.table()` chaining is acceptable
- Complex queries (joins, aggregations, multi-step logic) → named Postgres function called via `.rpc()`
- Write and test all SQL in Supabase's SQL editor first, then save to the `sql/` folder
- Save every SQL function to `sql/functions/{function_name}.sql`
- Save every table definition to `sql/tables/{table_name}.sql`
- Name migration files with a numeric prefix: `001_`, `002_`
- Never edit a deployed migration — always create a new one
- DB user should have `SELECT`, `INSERT`, `UPDATE`, `DELETE` only — no `DROP`, `CREATE`, `GRANT`
- Never return raw database error messages to the client

---

## 🚀 `main.py` Rules

- Only contains app factory function and router registration
- Zero business logic
- CORS middleware configured here and nowhere else — never `allow_origins=["*"]` in production
- Security headers middleware applied here
- Swagger `/docs` disabled when `settings.environment == "production"`

**Security headers middleware:**
```python
from starlette.middleware.base import BaseHTTPMiddleware
from fastapi.middleware.cors import CORSMiddleware

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

app.add_middleware(SecurityHeadersMiddleware)
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.allowed_origins,
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
)
```

**Rate limiting:**
```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

# Apply strict limits on auth, reset, upload endpoints
@router.post("/login")
@limiter.limit("5/minute")
async def login(request: Request, credentials: LoginRequest): ...
```

---

## 🔒 JWT Rules

| Vulnerability | Prevention |
|---|---|
| `alg: none` attack | Whitelist algorithms explicitly in `jwt.decode()` |
| Algorithm confusion | Never derive algorithm from the token header |
| Weak secret | 256+ bit cryptographically random secret from `settings` |
| Missing expiration | Always set `exp` claim |
| Token in response body | Set in `httpOnly`, `Secure`, `SameSite=Strict` cookie for browser clients |

```python
def create_access_token(user_id: str) -> str:
    return jwt.encode(
        {
            "sub": user_id,
            "iat": datetime.utcnow(),
            "exp": datetime.utcnow() + timedelta(minutes=settings.access_token_expire_minutes),
            "jti": str(uuid.uuid4()),
        },
        settings.secret_key,
        algorithm=settings.algorithm,
    )

def verify_token(token: str) -> dict:
    return jwt.decode(token, settings.secret_key, algorithms=[settings.algorithm])
```

> **Firebase override:** If this project uses the Firebase Auth override, replace `verify_token` with `firebase_admin.auth.verify_id_token()`. See the Firebase override section below.

---

## 📤 File Upload Rules

- [ ] Validate magic bytes — never rely on `Content-Type` header alone
- [ ] Enforce file size server-side
- [ ] Rename file to UUID — never use original filename
- [ ] Store outside webroot or in an isolated bucket/CDN domain
- [ ] Serve with `Content-Disposition: attachment` and `X-Content-Type-Options: nosniff`

```python
import magic, uuid
from pathlib import Path

ALLOWED_MIME_TYPES = {
    "image/jpeg": b"\xff\xd8\xff",
    "image/png": b"\x89PNG\r\n\x1a\n",
    "application/pdf": b"%PDF",
}
MAX_FILE_SIZE = 5 * 1024 * 1024

async def handle_upload(content: bytes, original_filename: str) -> str:
    if len(content) > MAX_FILE_SIZE:
        raise ValidationError("File too large")
    detected_mime = magic.from_buffer(content, mime=True)
    if detected_mime not in ALLOWED_MIME_TYPES:
        raise ValidationError("File type not allowed")
    if not content.startswith(ALLOWED_MIME_TYPES[detected_mime]):
        raise ValidationError("File content does not match declared type")
    extension = detected_mime.split("/")[1]
    safe_filename = f"{uuid.uuid4()}.{extension}"
    (Path(settings.upload_dir) / safe_filename).write_bytes(content)
    return safe_filename
```

---

## 🌐 SSRF & Path Traversal Rules

**SSRF — validate any user-supplied URL before fetching:**
```python
import ipaddress, socket
from urllib.parse import urlparse

SSRF_BLOCKED_NETWORKS = [
    ipaddress.ip_network("10.0.0.0/8"),
    ipaddress.ip_network("172.16.0.0/12"),
    ipaddress.ip_network("192.168.0.0/16"),
    ipaddress.ip_network("127.0.0.0/8"),
    ipaddress.ip_network("169.254.0.0/16"),  # Cloud metadata — covers Railway, AWS, GCP, Azure
    ipaddress.ip_network("::1/128"),
    ipaddress.ip_network("fc00::/7"),
]

def validate_webhook_url(url: str) -> str:
    parsed = urlparse(url)
    if parsed.scheme not in ("http", "https"):
        raise ValidationError("Only HTTP/HTTPS allowed")
    resolved_ip = ipaddress.ip_address(socket.gethostbyname(parsed.hostname))
    if resolved_ip.is_private or resolved_ip.is_loopback or resolved_ip.is_link_local:
        raise ValidationError("Requests to internal addresses are not allowed")
    for network in SSRF_BLOCKED_NETWORKS:
        if resolved_ip in network:
            raise ValidationError("Requests to internal addresses are not allowed")
    return url
```

**Path traversal — resolve and confirm path stays inside base dir:**
```python
BASE_DIR = Path(settings.upload_dir).resolve()

def safe_file_path(filename: str) -> Path:
    target = (BASE_DIR / filename).resolve()
    if not str(target).startswith(str(BASE_DIR) + "/"):
        raise ValidationError("Invalid file path")
    return target
```

---

## 🔑 Password Rules

- Hash with **Argon2id** — never MD5, SHA1, or plain SHA256
- Minimum 8 characters, no maximum under 128
- Never log, return, or store plaintext passwords

```python
from passlib.context import CryptContext
pwd_context = CryptContext(schemes=["argon2"], deprecated="auto")

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)
```

---

## 📐 When to Split & Scale

| Signal | Action |
|--------|--------|
| `routes.py` has 5+ unrelated domains | Split into `{domain}_routes.py` files |
| Service function exceeds ~50 lines | Break into smaller focused functions |
| Same SQL query in 2+ services | Promote to a shared Supabase SQL function |
| Operation takes >2s | Move to an async task queue |
| Writing to 2+ tables in one operation | Wrap in a Postgres transaction or SQL function |

---

## 📦 Dependency Rules (`requirements.txt`)

- [ ] Any new `import` that requires an install has a matching entry in `requirements.txt`
- [ ] Versions are always pinned exactly — `fastapi==0.111.0`, never `fastapi>=0.111.0`
- [ ] After adding any new import to any file, pause and confirm the package is listed before moving on

| Trigger | Action |
|---------|--------|
| New `import` added to any file | Add pinned package to `requirements.txt` immediately |
| Package upgraded locally | Update the pinned version in `requirements.txt` |
| Package removed from codebase | Remove it from `requirements.txt` |
| New override section activated | Add all packages that override requires |

---

## ✅ Pre-Commit Checklist

Before any feature is considered done:

- [ ] Router under 15 lines, one service call, correct `response_model`
- [ ] Auth via `Depends(get_current_user)` on every protected route
- [ ] Ownership enforced in the SQL `WHERE` clause — not after fetching
- [ ] Service raises custom exceptions, no FastAPI imports
- [ ] Schemas use `extra = "forbid"`, no admin fields in user-editable schemas
- [ ] UUIDs used for all resource identifiers
- [ ] Parameterized queries only — no string formatting into SQL
- [ ] Dynamic `ORDER BY` columns whitelisted explicitly
- [ ] JWT algorithm whitelisted in `jwt.decode(algorithms=[...])`
- [ ] No hardcoded secrets anywhere — all via `settings`
- [ ] Generic error responses — no stack traces or DB details exposed
- [ ] Security headers middleware applied in `main.py`
- [ ] CORS restricted to known origins from `settings`
- [ ] Rate limiting on all auth and sensitive endpoints
- [ ] Complex SQL saved to `sql/functions/`
- [ ] New routes registered in `main.py`
- [ ] Every new `import` has a pinned entry in `requirements.txt`
- [ ] Swagger `/docs` disabled in production

---

## 🔧 Project-Specific Overrides

> Only activate the section relevant to this project.

---

### 🔥 Override: Firebase Auth

Activate when this project uses Firebase Authentication for user identity.

**What changes:**
- `core/dependencies.py` uses `firebase_admin.auth.verify_id_token()` instead of a generic `verify_jwt()`
- `settings.jwt_secret` is removed — Firebase manages token signing, not us
- `settings.firebase_credentials_path` is added — path to service account JSON
- User identifier in services is `decoded["uid"]`, not a custom `id` field
- Firebase Admin SDK is initialised once at app startup in `core/firebase.py`
- The service account JSON file is always in `.gitignore` — never committed
- Skip the JWT signing/verification section above — Firebase handles it

**Checklist when using Firebase Auth:**
- [ ] Firebase Admin SDK initialised at startup, not per-request
- [ ] `verify_id_token()` called inside the dependency — never inside a router or service
- [ ] `ExpiredIdTokenError` and `InvalidIdTokenError` caught and converted to `401` separately
- [ ] `uid` used as the user identifier passed to all service functions
- [ ] Service account JSON referenced via `settings.firebase_credentials_path`
- [ ] `firebase_credentials_path` in `.gitignore` and `.env.example`
