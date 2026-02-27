# FastAPI Cursor Rules
> Directives for every file generated in this project.
> No ORMs. No SQLAlchemy. Raw SQL only. Production quality always.

---

## 🧠 Core Mental Model

| Layer | Responsibility | Rule |
|-------|---------------|------|
| Router | Define the endpoint | 5–15 lines max. One service call. Nothing else. |
| Service | Do the work | All logic, all DB calls, all external APIs live here |
| Schema | Define the shape | Every input and output must be a typed Pydantic model |
| Config | Manage secrets | Every value from `settings`. Never `os.environ` directly |
| Exceptions | Signal failures | Services raise custom exceptions. Routers convert them to HTTP errors |

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
- [ ] JWT / identity extracted via `Depends()` — never inline
- [ ] Calls exactly one service function
- [ ] `response_model` declared on every route decorator
- [ ] Only catches custom exceptions from `core/exceptions.py`
- [ ] Converts custom exceptions to `HTTPException` with the correct status code
- [ ] No SQL, no DB calls, no external API calls
- [ ] No business logic or conditionals beyond routing

**Status code ownership:**
- `401` → missing or invalid identity token
- `403` → plan or usage limit failure
- `404` → resource not found
- `422` → Pydantic validation (automatic, never manual)
- `500` → unexpected DB or service failure

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

**SQL rule:**
- Simple CRUD → Supabase `.table()` chaining is acceptable
- Complex queries (joins, aggregations, multi-step logic) → write a named Postgres function and call via `.rpc()`
- All SQL functions saved to `sql/functions/` for version control
- All DB calls stay in the service — never in the router

---

## 🟨 Schema Checklist

When generating a schema, verify:

- [ ] Every request body has its own schema
- [ ] Every response shape has its own schema
- [ ] Request and response schemas are always separate classes — never reused
- [ ] All fields use explicit Python types — no bare `Any`
- [ ] Non-obvious fields have a `Field(description=...)` for Swagger
- [ ] No business logic or methods inside the schema class

---

## ⚙️ Config Checklist

When adding configuration or secrets:

- [ ] All values declared as typed fields in the `Settings` class
- [ ] `os.environ.get()` never used anywhere in the codebase
- [ ] New secret added to `.env.example` with a placeholder value immediately
- [ ] `.env` is in `.gitignore`
- [ ] App will crash at startup if a required value is missing — this is correct behaviour

---

## ❌ Exceptions Checklist

When handling errors:

- [ ] Custom exception types defined in `core/exceptions.py` for every known failure category
- [ ] All custom exceptions inherit from a single `AppBaseException`
- [ ] Services raise custom exceptions — never `HTTPException`
- [ ] Routers catch custom exceptions and convert to `HTTPException`
- [ ] No bare `except Exception` without a `logger.error()` call and re-raise
- [ ] `RuntimeError` is acceptable as a fallback for fast iteration — but never catch it in the router; let it bubble up as a 500 with a full traceback in logs

**Standard exception types to always have available:**
`ResourceNotFound`, `UsageLimitExceeded`, `ExternalAPIError`, `DatabaseError`, `Unauthorized`

**RuntimeError fallback rule:** raise it to signal an intentional failure you haven't typed yet. Never catch `RuntimeError` — if you catch it you'll swallow real unexpected errors silently.

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

---

## 🚀 `main.py` Rules

- Only contains app factory function and router registration
- Zero business logic
- CORS middleware configured here and nowhere else
- Swagger `/docs` disabled when `settings.environment == "production"`

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

Every import that didn't ship with Python must be in `requirements.txt` with a pinned version.

- [ ] Any new `import` that requires an install has a matching entry in `requirements.txt`
- [ ] Versions are always pinned exactly — `fastapi==0.111.0`, never `fastapi` or `fastapi>=0.111.0`
- [ ] When adding a package, check its sub-dependencies and pin those too if they are critical to runtime behaviour
- [ ] Never rely on a package being available because it was installed locally — if it's not in `requirements.txt`, it doesn't exist in production
- [ ] After adding any new import to any file, pause and confirm the package is listed before moving on

**When to update `requirements.txt`:**
| Trigger | Action |
|---------|--------|
| New `import` added to any file | Add pinned package to `requirements.txt` immediately |
| Package upgraded locally | Update the pinned version in `requirements.txt` |
| Package removed from codebase | Remove it from `requirements.txt` |
| New override section activated (e.g. Firebase) | Add all packages that override requires |

---

## ✅ Pre-Commit Checklist

Before any feature is considered done:

- [ ] Router under 15 lines, one service call, correct `response_model`
- [ ] Service raises custom exceptions, no FastAPI imports
- [ ] Schemas are separate for request and response
- [ ] No hardcoded secrets anywhere
- [ ] All errors caught explicitly with logging
- [ ] Complex SQL saved to `sql/functions/`
- [ ] New routes registered in `main.py`
- [ ] Every new `import` has a pinned entry in `requirements.txt`

---

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

**Checklist when using Firebase Auth:**
- [ ] Firebase Admin SDK initialised at startup, not per-request
- [ ] `verify_id_token()` called inside the dependency — never inside a router or service
- [ ] `ExpiredIdTokenError` and `InvalidIdTokenError` caught and converted to `401` separately
- [ ] `uid` used as the user identifier passed to all service functions
- [ ] Service account JSON referenced via `settings.firebase_credentials_path`
- [ ] `firebase_credentials_path` in `.gitignore` and `.env.example`
