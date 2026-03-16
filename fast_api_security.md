---
name: FastAPI-Security-Skill
description: This skill helps AI write secure FastAPI applications. Use this when working on any FastAPI project or when a user requests a security scan or audit of their API endpoints. Supports Railway/Postgres (raw SQL) and Supabase client patterns.
---

# FastAPI Endpoint Security Guide

## Overview

Approach every FastAPI endpoint from a **bug hunter's perspective**. Apply defense in depth — never rely on a single control.

**Core Principles:**
- Validate all input server-side (Pydantic is your first line, not your only line)
- Fail closed: deny by default, grant explicitly
- Least privilege: minimize DB user permissions, scope tokens tightly
- Never trust client-supplied data, IDs, or roles
- All DB calls live in the service layer — never in the router

**DB patterns supported:**
- **Railway/Postgres** — raw SQL via `asyncpg` or `psycopg2`. No ORMs, no SQLAlchemy.
- **Supabase** — `.table()` chaining for simple CRUD, `.rpc()` for complex queries.

---

## Authentication & Authorization

### Dependency-Based Auth Pattern

Use FastAPI's `Depends()` for auth — never inline auth logic in route handlers.

```python
# core/dependencies.py
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

### Router — Thin, One Service Call

```python
# api/v1/documents_routes.py
@router.get("/documents/{document_id}", response_model=DocumentResponse)
async def get_document(
    document_id: uuid.UUID,
    current_user: dict = Depends(get_current_user),
):
    try:
        return await document_service.get_document(
            document_id=document_id,
            user_id=current_user["user_id"],
        )
    except ResourceNotFound:
        raise HTTPException(status_code=404, detail="Document not found")
```

### Service — Ownership Verified at the Data Layer

Return 404 for both missing and unauthorized resources — prevents enumeration.

**Railway/Postgres (raw SQL):**
```python
# services/document_service.py
async def get_document(document_id: uuid.UUID, user_id: str) -> dict:
    async with get_db_connection() as conn:
        row = await conn.fetchrow(
            # Ownership check is part of the WHERE clause — single query, no extra round trip
            "SELECT id, title, content, owner_id FROM documents WHERE id = $1 AND owner_id = $2",
            document_id,
            user_id,
        )
    if row is None:
        raise ResourceNotFound("Document not found")
    return dict(row)
```

**Supabase:**
```python
async def get_document(document_id: uuid.UUID, user_id: str) -> dict:
    response = (
        supabase.table("documents")
        .select("id, title, content, owner_id")
        .eq("id", str(document_id))
        .eq("owner_id", user_id)  # Ownership enforced here
        .maybe_single()
        .execute()
    )
    if response.data is None:
        raise ResourceNotFound("Document not found")
    return response.data
```

### Authorization Checklist

- [ ] Every protected route uses `Depends(get_current_user)` — never inline
- [ ] Ownership enforced inside the WHERE clause of the DB query — not after fetching
- [ ] 404 returned for both missing and unauthorized resources (never 403 on owned resources)
- [ ] Org/tenant ID included in WHERE clause for multi-tenant apps
- [ ] Role permissions validated server-side — never derived from request body
- [ ] Parent resource ownership checked (comment → post → user)

### Common Pitfalls

- **IDOR**: Ownership must be in the SQL `WHERE` clause, not checked after fetching the row
- **Mass Assignment**: Use explicit Pydantic schemas — never pass `**request.dict()` to a DB insert
- **Privilege Escalation**: Strip `role`, `is_admin`, `is_superuser` from user-editable schemas
- **Horizontal Access**: User A accessing User B's resource — always filter by `owner_id = $user_id`

---

## Input Validation with Pydantic

Pydantic validates shape and types automatically — add field constraints and use `extra = "forbid"` to block unexpected fields.

```python
# models/schemas/document_schema.py
from pydantic import BaseModel, Field, validator
import re

class DocumentUpdateRequest(BaseModel):
    title: str = Field(..., min_length=1, max_length=200, strip_whitespace=True)
    content: str | None = Field(None, max_length=50_000)

    @validator("title")
    def title_no_script_tags(cls, v):
        if re.search(r"[<>\"';&]", v):
            raise ValueError("Title contains invalid characters")
        return v

    class Config:
        extra = "forbid"  # Blocks mass assignment — unknown fields are rejected
```

### Mass Assignment Prevention

```python
# VULNERABLE — user can inject {"role": "admin"} in the request body
await conn.execute("INSERT INTO users(id, name, role) VALUES($1, $2, $3)", ...)

# SECURE — schema only exposes fields the user is allowed to set
class UserUpdateRequest(BaseModel):
    name: str
    email: EmailStr
    # role, is_admin are NOT in this schema

# Service maps schema fields explicitly to the query
async def update_user(user_id: str, data: UserUpdateRequest) -> dict:
    async with get_db_connection() as conn:
        row = await conn.fetchrow(
            "UPDATE users SET name = $1, email = $2 WHERE id = $3 RETURNING id, name, email",
            data.name,
            data.email,
            user_id,
        )
    return dict(row)
```

### Use UUIDs, Not Sequential IDs

```python
# In your SQL table definition (sql/tables/documents.sql)
# id UUID PRIMARY KEY DEFAULT gen_random_uuid()

# In your Pydantic schema — accept UUID type, not int
class DocumentResponse(BaseModel):
    id: uuid.UUID
    title: str
```

---

## SQL Injection Prevention

Always use parameterized queries. Never format or concatenate user input into SQL strings.

**Railway/Postgres (asyncpg — positional `$N` placeholders):**
```python
# VULNERABLE
await conn.execute(f"SELECT * FROM users WHERE email = '{email}'")

# SECURE
await conn.fetchrow("SELECT * FROM users WHERE email = $1", email)
```

**Supabase (client handles parameterization automatically):**
```python
# SECURE — Supabase client parameterizes all .eq() / .filter() calls
response = supabase.table("users").select("*").eq("email", email).execute()
```

### ORDER BY / Dynamic Column Names

These cannot be parameterized — whitelist them explicitly.

```python
ALLOWED_SORT_COLUMNS = {"created_at", "title", "updated_at"}
ALLOWED_SORT_ORDERS = {"asc", "desc"}

async def list_documents(user_id: str, sort_by: str = "created_at", order: str = "asc") -> list[dict]:
    if sort_by not in ALLOWED_SORT_COLUMNS:
        raise ValueError("Invalid sort column")
    if order not in ALLOWED_SORT_ORDERS:
        raise ValueError("Invalid sort order")

    # Safe to interpolate because both values came from whitelists
    async with get_db_connection() as conn:
        rows = await conn.fetch(
            f"SELECT id, title, created_at FROM documents WHERE owner_id = $1 ORDER BY {sort_by} {order}",
            user_id,
        )
    return [dict(r) for r in rows]
```

### Additional Defenses

- DB user should have `SELECT`, `INSERT`, `UPDATE`, `DELETE` only — no `DROP`, `CREATE`, `GRANT`
- Never return raw database error messages to the client

---

## JWT Security

> **Firebase override:** If this project uses the Firebase Auth override from `fastapi.coding-rules.md`, this section is replaced by `firebase_admin.auth.verify_id_token()`. Skip JWT signing/verification below and follow the Firebase checklist instead.

| Vulnerability | Prevention |
|---|---|
| `alg: none` attack | Whitelist algorithms explicitly in `jwt.decode()` |
| Algorithm confusion | Never derive algorithm from the token header |
| Weak secret | 256+ bit cryptographically random secret from `settings` |
| Missing expiration | Always set `exp` claim |
| Token in response body | Set in `httpOnly`, `Secure`, `SameSite=Strict` cookie for browser clients |

```python
# services/auth_service.py
from jose import JWTError, jwt
from datetime import datetime, timedelta
import uuid
from core.config import settings

def create_access_token(user_id: str) -> str:
    return jwt.encode(
        {
            "sub": user_id,
            "iat": datetime.utcnow(),
            "exp": datetime.utcnow() + timedelta(minutes=settings.access_token_expire_minutes),
            "jti": str(uuid.uuid4()),  # Unique ID — enables per-token revocation
        },
        settings.secret_key,
        algorithm=settings.algorithm,  # Comes from config, not the token header
    )

# core/dependencies.py
def verify_token(token: str) -> dict:
    # CRITICAL: whitelist algorithm — never derive from token header
    return jwt.decode(token, settings.secret_key, algorithms=[settings.algorithm])
```

### JWT Checklist

- [ ] Algorithm explicitly listed in `jwt.decode(algorithms=[...])` — not derived from token
- [ ] `alg: none` rejected (whitelisting handles this automatically)
- [ ] `settings.secret_key` is 256+ bits of random data — never a password or phrase
- [ ] `exp` claim always set and validated
- [ ] Tokens in `httpOnly` cookies for browser clients — never return raw tokens in JSON
- [ ] Refresh token rotation: invalidate old refresh token on use

---

## File Upload Security

```python
# services/upload_service.py
import magic  # python-magic
import uuid
from pathlib import Path
from core.exceptions import ValidationError

ALLOWED_MIME_TYPES = {
    "image/jpeg": b"\xff\xd8\xff",
    "image/png": b"\x89PNG\r\n\x1a\n",
    "application/pdf": b"%PDF",
}
MAX_FILE_SIZE = 5 * 1024 * 1024  # 5MB

async def handle_upload(content: bytes, original_filename: str) -> str:
    # 1. File size
    if len(content) > MAX_FILE_SIZE:
        raise ValidationError("File too large")

    # 2. Validate MIME type via magic bytes — never trust Content-Type header
    detected_mime = magic.from_buffer(content, mime=True)
    if detected_mime not in ALLOWED_MIME_TYPES:
        raise ValidationError("File type not allowed")

    # 3. Confirm magic bytes match
    if not content.startswith(ALLOWED_MIME_TYPES[detected_mime]):
        raise ValidationError("File content does not match declared type")

    # 4. Rename to UUID — discard original filename entirely
    extension = detected_mime.split("/")[1]
    safe_filename = f"{uuid.uuid4()}.{extension}"

    # 5. Store outside webroot (or upload to isolated S3/R2 bucket)
    save_path = Path(settings.upload_dir) / safe_filename
    save_path.write_bytes(content)

    return safe_filename
```

### Upload Checklist

- [ ] Validate magic bytes — never rely on `Content-Type` header alone
- [ ] Enforce file size server-side
- [ ] Rename file to UUID — never use original filename
- [ ] Store outside webroot or in an isolated bucket/CDN domain
- [ ] Serve with `Content-Disposition: attachment` and `X-Content-Type-Options: nosniff`
- [ ] Strip EXIF metadata from images before storing

---

## SSRF Prevention

Any service function that fetches a user-supplied URL is a SSRF risk.

```python
# services/webhook_service.py
import ipaddress
import socket
from urllib.parse import urlparse
from core.exceptions import ValidationError

SSRF_BLOCKED_NETWORKS = [
    ipaddress.ip_network("10.0.0.0/8"),
    ipaddress.ip_network("172.16.0.0/12"),
    ipaddress.ip_network("192.168.0.0/16"),
    ipaddress.ip_network("127.0.0.0/8"),
    ipaddress.ip_network("169.254.0.0/16"),  # Cloud metadata (AWS/GCP/Azure/Railway)
    ipaddress.ip_network("::1/128"),
    ipaddress.ip_network("fc00::/7"),
]

def validate_webhook_url(url: str) -> str:
    parsed = urlparse(url)

    if parsed.scheme not in ("http", "https"):
        raise ValidationError("Only HTTP/HTTPS allowed")

    try:
        resolved_ip = ipaddress.ip_address(socket.gethostbyname(parsed.hostname))
    except Exception:
        raise ValidationError("Could not resolve hostname")

    if resolved_ip.is_private or resolved_ip.is_loopback or resolved_ip.is_link_local:
        raise ValidationError("Requests to internal addresses are not allowed")

    for network in SSRF_BLOCKED_NETWORKS:
        if resolved_ip in network:
            raise ValidationError("Requests to internal addresses are not allowed")

    return url
```

**Railway-specific:** The `169.254.0.0/16` block also covers Railway's internal metadata service — always include it.

---

## Path Traversal Prevention

```python
# services/file_service.py
from pathlib import Path
from core.exceptions import ValidationError

BASE_DIR = Path(settings.upload_dir).resolve()

def safe_file_path(filename: str) -> Path:
    target = (BASE_DIR / filename).resolve()
    # Ensure resolved path stays inside BASE_DIR
    if not str(target).startswith(str(BASE_DIR) + "/"):
        raise ValidationError("Invalid file path")
    return target
```

---

## Open Redirect Prevention

```python
# services/auth_service.py
from urllib.parse import urlparse
from core.config import settings

ALLOWED_REDIRECT_HOSTS = set(settings.allowed_redirect_hosts)  # From config, not hardcoded

def validate_redirect_url(url: str) -> str:
    # Prefer relative paths only
    if url.startswith("/") and not url.startswith("//"):
        return url

    parsed = urlparse(url)
    if parsed.hostname not in ALLOWED_REDIRECT_HOSTS:
        raise ValidationError("Invalid redirect URL")

    return url
```

---

## Security Headers Middleware

```python
# main.py
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

# CORS — never use allow_origins=["*"] in production
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.allowed_origins,  # From config
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
)

# Disable Swagger in production
if settings.environment == "production":
    app = FastAPI(docs_url=None, redoc_url=None)
```

---

## Rate Limiting

```python
# main.py
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

# api/v1/auth_routes.py
@router.post("/login", response_model=TokenResponse)
@limiter.limit("5/minute")
async def login(request: Request, credentials: LoginRequest):
    ...

@router.post("/reset-password", response_model=MessageResponse)
@limiter.limit("3/hour")
async def reset_password(request: Request, data: PasswordResetRequest):
    ...
```

Apply strict limits on: login, token refresh, password reset, email/SMS verification, file upload.

---

## Error Handling

Never expose internal errors or DB details to clients. All error handling follows the pattern in `fastapi.coding-rules.md` — services raise custom exceptions, routers convert them to `HTTPException`.

```python
# core/exceptions.py
class AppBaseException(Exception):
    pass

class ResourceNotFound(AppBaseException):
    pass

class Unauthorized(AppBaseException):
    pass

class ValidationError(AppBaseException):
    pass

class DatabaseError(AppBaseException):
    pass

# main.py — catch-all for unexpected errors
@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    logger.exception("Unhandled exception: %s %s", request.method, request.url)
    return JSONResponse(
        status_code=500,
        content={"detail": "Internal server error"},  # Generic — no stack traces, no DB errors
    )
```

---

## Secrets Management

```python
# core/config.py
from pydantic import BaseSettings

class Settings(BaseSettings):
    environment: str
    secret_key: str              # 256+ bit random hex — `secrets.token_hex(32)`
    algorithm: str = "HS256"
    access_token_expire_minutes: int = 15

    # Railway Postgres
    database_url: str | None = None

    # Supabase (if used)
    supabase_url: str | None = None
    supabase_key: str | None = None

    allowed_origins: list[str] = []
    allowed_redirect_hosts: list[str] = []
    upload_dir: str = "/tmp/uploads"

    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"

settings = Settings()
```

- Never use `os.environ.get()` anywhere — always `settings.*`
- Add every new secret to `.env.example` with a placeholder immediately
- App must crash at startup if a required value is missing
- `.env` is always in `.gitignore`

---

## Password Security

- Minimum 8 characters (12+ recommended), no maximum (or 128+ chars)
- Hash with **Argon2id** or **bcrypt** — never MD5, SHA1, or plain SHA256
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

## General Security Checklist

- [ ] All protected routes use `Depends(get_current_user)` — never inline
- [ ] Ownership enforced in the SQL `WHERE` clause — not after fetching
- [ ] UUIDs used for all resource identifiers
- [ ] Pydantic schemas use `extra = "forbid"` to block mass assignment
- [ ] Parameterized queries only — no string formatting into SQL
- [ ] Dynamic `ORDER BY` columns whitelisted explicitly
- [ ] JWT algorithm whitelisted in `jwt.decode(algorithms=[...])`
- [ ] File uploads validate magic bytes and are renamed to UUID
- [ ] User-supplied URLs validated against SSRF blocklist (including `169.254.x.x`)
- [ ] Security headers applied via middleware in `main.py`
- [ ] CORS restricted to known origins from `settings`
- [ ] Rate limiting on all auth and sensitive endpoints
- [ ] Generic error responses — no stack traces or DB details exposed
- [ ] All secrets in environment variables via `settings` — never hardcoded
- [ ] Swagger `/docs` disabled in production
