# CrackedIn Extension Sync - Stage 1 Implementation Plan

**Product:** CrackedIn  
**Feature:** Browser extension based LeetCode corpus sync  
**Primary scope:** Stage 1 only - first-time historical sync  
**Auth model:** JWT stored in an HttpOnly cookie  
**Database state:** Supabase Postgres tables and PGMQ queue are already created and verified  
**Future compatibility:** Keep architecture ready for Stage 2 live capture and Stage 3 delta repair, but do not implement them in this pass.

---

## 0. Purpose

This document is the implementation instruction plan for building the CrackedIn Chrome/Chromium extension sync system on top of the existing InterviewPrep.ai repo.

The implementation agent should follow this file in order. Do not re-litigate architecture decisions. Do not implement alternatives. Do not add token storage in `chrome.storage.local`. Do not add extension session tables. The final auth path is JWT in an HttpOnly cookie.

The Stage 1 outcome is:

```text
A Chrome extension connected to CrackedIn that can:
1. authenticate to the CrackedIn backend through an HttpOnly JWT cookie,
2. verify the user's LeetCode browser session,
3. fetch all historical LeetCode submission metadata,
4. fetch submitted code for the latest 100 submissions in the foreground,
5. continue remaining code backfill in the background,
6. retry failed code/detail fetches safely,
7. upload sanitized batches to FastAPI,
8. store the corpus in Supabase Postgres using the finalized sync schema,
9. update extension-local and backend sync state after every batch,
10. publish `submission.received` events to PGMQ for missing problem catalog data.
```

---

## 1. Locked Decisions

| Area | Final decision |
|---|---|
| Product name | CrackedIn |
| Repo strategy | Extend the current InterviewPrep.ai repo |
| Extension location | Add a new `extension/` directory inside the repo |
| Backend strategy | Add extension-specific routes/modules to the existing FastAPI backend |
| Existing app data | Keep current SQLite-backed app behavior for existing routes |
| Extension corpus DB | Use Supabase Postgres |
| User ID for extension tables | Use Postgres `public.users.id` UUID |
| SQLite bridge | `public.users.sqlite_id` exists and is maintained by `pg_user_sync` |
| Extension auth | JWT stored in an HttpOnly cookie |
| Extension token storage | No JWT in `chrome.storage.local`; no Bearer token in extension storage |
| Extension sessions table | Do not create/use `system.extension_sessions` for Stage 1 |
| Auth dependency | Extension routes use Postgres-native auth from `api/auth_pg.py`, extended to read cookie |
| Extension DB writes | Extension never writes directly to Supabase; backend writes to Supabase |
| LeetCode auth | Use the user's existing browser session from the extension context |
| LeetCode cookie handling | Never read, store, or upload LeetCode cookies |
| Chrome background execution | One Manifest V3 service worker with multiple resumable jobs |
| Metadata source | LeetCode `submissionList(offset, limit)` |
| Code source | LeetCode `submissionDetails(submissionId)` |
| Batching | GraphQL aliasing |
| Foreground code fetch | Latest 100 submissions by timestamp |
| Remaining code | Background backfill |
| Failed code | Persistent failed-code retry queue in extension storage |
| Event queue | Existing Supabase PGMQ queue `extension_events` |
| Stage 1 backend worker | Catalog worker only |
| Intelligence/code analysis | Deferred |
| Stage 2 live capture | Readiness hooks only; do not implement now |
| Stage 3 delta repair | Readiness hooks only; do not implement now |

---

## 2. Current Postgres State

The Supabase/Postgres work is considered complete for Stage 1.

Expected existing objects:

```text
public.users
  id UUID
  sqlite_id INTEGER

catalog.coding_problems
user_data.user_coding_profiles
user_data.user_coding_problems
user_data.user_coding_submissions
code_blobs.user_coding_submission_code
system.sync_state

PGMQ queue:
  extension_events
```

Implementation agents should verify these objects but should not recreate them blindly.

Verification SQL:

```sql
SELECT
  table_schema,
  table_name
FROM information_schema.tables
WHERE table_schema IN ('catalog', 'user_data', 'code_blobs', 'system', 'public')
ORDER BY table_schema, table_name;
```

Verify PGMQ queues:

```sql
SELECT * FROM pgmq.list_queues();
```

Expected queue:

```text
extension_events
```

---

## 3. High-Level Runtime Architecture

```text
Chrome Extension
├── Popup UI
│   ├── shows CrackedIn connection state
│   ├── shows LeetCode connection state
│   ├── starts Stage 1 sync
│   ├── shows progress
│   └── exposes pause/resume
│
├── One Manifest V3 service worker
│   ├── calls CrackedIn API with credentials: "include"
│   ├── validates extension cookie via /api/extension/me
│   ├── validates LeetCode session via userStatus
│   ├── runs Phase A metadata sync
│   ├── runs Phase B foreground latest-100 code fetch
│   ├── runs Phase C background code backfill
│   ├── runs failed-code retry queue
│   ├── writes local cursor after every batch
│   └── uploads sanitized data to FastAPI
│
└── Stage 2 placeholder content script
    └── reserved for future live capture; do not implement in Stage 1

FastAPI Backend
├── existing app routes
├── api/auth_pg.py
│   └── resolves HttpOnly extension JWT cookie to Postgres public.users.id UUID
├── /api/extension/* routes
├── Supabase/Postgres writer services
├── validation/normalization layer
├── PGMQ publisher
└── catalog worker

Supabase Postgres
├── public.users
├── catalog.coding_problems
├── user_data.user_coding_profiles
├── user_data.user_coding_problems
├── user_data.user_coding_submissions
├── code_blobs.user_coding_submission_code
└── system.sync_state
```

---

## 4. Authentication Design - JWT in HttpOnly Cookie

### 4.1 Auth principle

The extension does not store or read tokens. The browser attaches the CrackedIn extension cookie automatically.

Extension API calls must use:

```ts
fetch(url, {
  method: "GET",
  credentials: "include",
});
```

The backend reads the JWT from the HttpOnly cookie, verifies it, resolves the Postgres user UUID, and authorizes the request.

### 4.2 Cookie name

Use one cookie name consistently:

```text
crackedin_ext_jwt
```

### 4.3 Cookie settings

Use environment-driven settings:

```env
EXTENSION_AUTH_COOKIE_NAME=crackedin_ext_jwt
EXTENSION_AUTH_COOKIE_DOMAIN=.crackedin.app
EXTENSION_AUTH_COOKIE_SECURE=true
EXTENSION_AUTH_COOKIE_SAMESITE=none
EXTENSION_AUTH_COOKIE_MAX_AGE_DAYS=60
```

Cookie must be:

```text
HttpOnly
Secure in production
SameSite=None for Chrome extension fetch compatibility
Path=/api/extension
```

Example Set-Cookie behavior:

```python
response.set_cookie(
    key="crackedin_ext_jwt",
    value=jwt_token,
    httponly=True,
    secure=True,
    samesite="none",
    path="/api/extension",
    domain=".crackedin.app",
    max_age=60 * 24 * 60 * 60,
)
```

### 4.4 JWT payload

The cookie JWT should use the Postgres UUID as `sub`.

```json
{
  "sub": "postgres-user-uuid",
  "typ": "extension_cookie",
  "scopes": [
    "extension:read",
    "coding_profile:write",
    "coding_submission:write",
    "sync_state:write"
  ],
  "iat": 1710000000,
  "exp": 1715184000
}
```

### 4.5 No extension session table

Do not create or use:

```text
extension_auth_codes
extension_sessions
system.extension_sessions
refresh_token_hash
one-time extension auth code table
```

Stage 1 uses cookie-only JWT authentication. Logout/disconnect clears the cookie. Expiry is controlled by JWT `exp` and cookie `Max-Age`.

### 4.6 Required backend endpoints for auth

```text
POST /api/extension/auth/connect
GET  /api/extension/me
POST /api/extension/auth/logout
```

#### `POST /api/extension/auth/connect`

This endpoint is called from the logged-in CrackedIn web app. It creates the extension JWT cookie.

Implementation responsibilities:

1. Authenticate the current web user.
2. Ensure/sync a `public.users` row exists in Postgres.
3. Create JWT with `sub = public.users.id`.
4. Set `crackedin_ext_jwt` HttpOnly cookie.
5. Return safe user info.

Pseudo-code:

```python
@router.post("/auth/connect")
def connect_extension(
    response: Response,
    sqlite_user_id: int = Depends(get_current_user_id),
):
    pg_user = ensure_pg_user_for_sqlite_user(sqlite_user_id)

    token = create_extension_cookie_jwt(
        pg_user_id=pg_user["id"],
        scopes=[
            "extension:read",
            "coding_profile:write",
            "coding_submission:write",
            "sync_state:write",
        ],
    )

    set_extension_cookie(response, token)

    return {
        "ok": True,
        "user": {
            "id": str(pg_user["id"]),
            "email": pg_user.get("email"),
            "display_name": pg_user.get("display_name"),
        },
    }
```

#### `GET /api/extension/me`

This endpoint validates the cookie and returns the current Postgres user.

```python
@router.get("/me")
def extension_me(
    user_id: UUID = Depends(get_current_extension_pg_user_id),
):
    return get_extension_me(user_id)
```

#### `POST /api/extension/auth/logout`

This clears the cookie.

```python
@router.post("/auth/logout")
def logout_extension(response: Response):
    clear_extension_cookie(response)
    return {"ok": True}
```

---

## 5. Required Repo Structure

Add or update these files.

```text
interview-prep-ai/
├── api/
│   ├── auth_pg.py                         # extend to read extension cookie JWT
│   ├── main.py                            # mount extension router
│   ├── config.py                          # add extension cookie + Supabase config
│   ├── routes/
│   │   └── extension_sync.py              # /api/extension/* routes
│   └── services/
│       └── extension_sync/
│           ├── __init__.py
│           ├── supabase_db.py
│           ├── cookie_auth.py             # cookie helpers and JWT creation
│           ├── pg_user_bridge.py          # ensure Postgres user from sqlite_id if needed
│           ├── models.py
│           ├── validators.py
│           ├── normalization.py
│           ├── profiles.py
│           ├── ingest_submissions.py
│           ├── ingest_code.py
│           ├── aggregates.py
│           ├── sync_state.py
│           ├── catalog_events.py
│           ├── catalog_worker.py
│           └── platforms/
│               ├── __init__.py
│               ├── base.py
│               └── leetcode.py
│
├── extension/
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   ├── manifest.json
│   └── src/
│       ├── shared/
│       │   ├── constants.ts
│       │   ├── types.ts
│       │   └── storage.ts
│       ├── api/
│       │   └── crackedinApi.ts
│       ├── platforms/
│       │   ├── types.ts
│       │   └── leetcode.ts
│       ├── service-worker/
│       │   ├── index.ts
│       │   ├── state.ts
│       │   ├── scheduler.ts
│       │   ├── graphql/
│       │   │   ├── aliasing.ts
│       │   │   └── leetcodeQueries.ts
│       │   └── jobs/
│       │       ├── authCheckJob.ts
│       │       ├── metadataSyncJob.ts
│       │       ├── foregroundCodeJob.ts
│       │       ├── backgroundCodeJob.ts
│       │       └── failedCodeRetryJob.ts
│       ├── popup/
│       │   ├── popup.html
│       │   ├── popup.ts
│       │   └── popup.css
│       └── content-scripts/
│           └── leetcodeLivePlaceholder.ts
│
└── docs/
    └── extension-stage1-implementation.md
```

Do not add `db/extension_auth_migration.sql`. Cookie-only auth does not require SQLite extension auth tables.

---

## 6. Backend Implementation Tasks

### Task B1 - Update backend config

Update `api/config.py`.

Add:

```python
EXTENSION_DATABASE_URL = os.getenv("EXTENSION_DATABASE_URL", "")
SUPABASE_PROJECT_URL = os.getenv("SUPABASE_PROJECT_URL", "")
SUPABASE_SERVICE_ROLE_KEY = os.getenv("SUPABASE_SERVICE_ROLE_KEY", "")

EXTENSION_PGMQ_QUEUE = os.getenv("EXTENSION_PGMQ_QUEUE", "extension_events")
CATALOG_WORKER_ENABLED = os.getenv("CATALOG_WORKER_ENABLED", "true").lower() == "true"
LEETCODE_GRAPHQL_URL = os.getenv("LEETCODE_GRAPHQL_URL", "https://leetcode.com/graphql")

EXTENSION_AUTH_COOKIE_NAME = os.getenv("EXTENSION_AUTH_COOKIE_NAME", "crackedin_ext_jwt")
EXTENSION_AUTH_COOKIE_DOMAIN = os.getenv("EXTENSION_AUTH_COOKIE_DOMAIN", "")
EXTENSION_AUTH_COOKIE_SECURE = os.getenv("EXTENSION_AUTH_COOKIE_SECURE", "true").lower() == "true"
EXTENSION_AUTH_COOKIE_SAMESITE = os.getenv("EXTENSION_AUTH_COOKIE_SAMESITE", "none")
EXTENSION_AUTH_COOKIE_MAX_AGE_DAYS = int(os.getenv("EXTENSION_AUTH_COOKIE_MAX_AGE_DAYS", "60"))
EXTENSION_JWT_SECRET = os.getenv("EXTENSION_JWT_SECRET", JWT_SECRET)
EXTENSION_JWT_ALGORITHM = os.getenv("EXTENSION_JWT_ALGORITHM", JWT_ALGORITHM)
```

Acceptance criteria:

- Backend boots with extension env vars.
- Local dev can set `EXTENSION_AUTH_COOKIE_SECURE=false`.
- Catalog worker can be disabled with `CATALOG_WORKER_ENABLED=false`.

---

### Task B2 - Add Supabase/Postgres connection helper

Create `api/services/extension_sync/supabase_db.py`.

```python
from contextlib import contextmanager
import psycopg
from psycopg.rows import dict_row
from api.config import EXTENSION_DATABASE_URL


@contextmanager
def get_pg():
    if not EXTENSION_DATABASE_URL:
        raise RuntimeError("EXTENSION_DATABASE_URL is not configured")

    conn = psycopg.connect(EXTENSION_DATABASE_URL, row_factory=dict_row)
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()
```

Add dependency:

```text
psycopg[binary]
```

Acceptance criteria:

- Backend can open a Supabase connection.
- Successful operations commit.
- Failed operations rollback.

---

### Task B3 - Add cookie auth helpers

Create `api/services/extension_sync/cookie_auth.py`.

```python
from __future__ import annotations

from datetime import datetime, timedelta, timezone
from uuid import UUID

import jwt
from fastapi import HTTPException, Request, Response, status

from api.config import (
    EXTENSION_AUTH_COOKIE_DOMAIN,
    EXTENSION_AUTH_COOKIE_MAX_AGE_DAYS,
    EXTENSION_AUTH_COOKIE_NAME,
    EXTENSION_AUTH_COOKIE_SAMESITE,
    EXTENSION_AUTH_COOKIE_SECURE,
    EXTENSION_JWT_ALGORITHM,
    EXTENSION_JWT_SECRET,
)
from api.services.extension_sync.supabase_db import get_pg

EXTENSION_SCOPES = [
    "extension:read",
    "coding_profile:write",
    "coding_submission:write",
    "sync_state:write",
]


def create_extension_cookie_jwt(pg_user_id: UUID) -> str:
    now = datetime.now(timezone.utc)
    exp = now + timedelta(days=EXTENSION_AUTH_COOKIE_MAX_AGE_DAYS)
    payload = {
        "sub": str(pg_user_id),
        "typ": "extension_cookie",
        "scopes": EXTENSION_SCOPES,
        "iat": int(now.timestamp()),
        "exp": int(exp.timestamp()),
    }
    return jwt.encode(payload, EXTENSION_JWT_SECRET, algorithm=EXTENSION_JWT_ALGORITHM)


def set_extension_cookie(response: Response, token: str) -> None:
    kwargs = {
        "key": EXTENSION_AUTH_COOKIE_NAME,
        "value": token,
        "httponly": True,
        "secure": EXTENSION_AUTH_COOKIE_SECURE,
        "samesite": EXTENSION_AUTH_COOKIE_SAMESITE,
        "path": "/api/extension",
        "max_age": EXTENSION_AUTH_COOKIE_MAX_AGE_DAYS * 24 * 60 * 60,
    }
    if EXTENSION_AUTH_COOKIE_DOMAIN:
        kwargs["domain"] = EXTENSION_AUTH_COOKIE_DOMAIN
    response.set_cookie(**kwargs)


def clear_extension_cookie(response: Response) -> None:
    kwargs = {
        "key": EXTENSION_AUTH_COOKIE_NAME,
        "path": "/api/extension",
        "secure": EXTENSION_AUTH_COOKIE_SECURE,
        "samesite": EXTENSION_AUTH_COOKIE_SAMESITE,
    }
    if EXTENSION_AUTH_COOKIE_DOMAIN:
        kwargs["domain"] = EXTENSION_AUTH_COOKIE_DOMAIN
    response.delete_cookie(**kwargs)


def decode_extension_cookie(request: Request) -> UUID:
    token = request.cookies.get(EXTENSION_AUTH_COOKIE_NAME)
    if not token:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Extension auth cookie missing")

    try:
        payload = jwt.decode(token, EXTENSION_JWT_SECRET, algorithms=[EXTENSION_JWT_ALGORITHM])
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Extension auth cookie expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid extension auth cookie")

    if payload.get("typ") != "extension_cookie":
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid extension token type")

    try:
        return UUID(str(payload["sub"]))
    except Exception:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid extension user id")


def get_current_extension_pg_user_id(request: Request) -> UUID:
    user_id = decode_extension_cookie(request)

    with get_pg() as conn:
        row = conn.execute(
            "SELECT id FROM public.users WHERE id = %s",
            (str(user_id),),
        ).fetchone()

    if row is None:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Extension user not found")

    return user_id
```

Acceptance criteria:

- Missing cookie returns 401.
- Expired cookie returns 401.
- Invalid JWT returns 401.
- Valid cookie returns `public.users.id` UUID.
- Extension routes never use SQLite auth directly.

---

### Task B4 - Update `api/auth_pg.py`

The repo already has `api/auth_pg.py` for Postgres-native auth. Extend or wrap it so extension routes can use cookie-based auth.

Required exported dependency:

```python
from uuid import UUID
from fastapi import Depends
from api.services.extension_sync.cookie_auth import get_current_extension_pg_user_id

# Extension routes should depend on this:
# user_id: UUID = Depends(get_current_extension_pg_user_id)
```

Acceptance criteria:

- `/api/extension/*` routes use Postgres UUID identity.
- No extension route depends on old SQLite `get_current_user_id` except `/auth/connect`, which is called from the logged-in web app to create the cookie.

---

### Task B5 - Add Postgres user bridge helper

Create `api/services/extension_sync/pg_user_bridge.py`.

Purpose: `/api/extension/auth/connect` is called from the web app while the user is authenticated through the existing app auth. It must ensure a matching Postgres user exists.

```python
from api.services.extension_sync.supabase_db import get_pg


def get_pg_user_by_sqlite_id(sqlite_user_id: int) -> dict | None:
    with get_pg() as conn:
        return conn.execute(
            "SELECT id, email, display_name, sqlite_id FROM public.users WHERE sqlite_id = %s",
            (sqlite_user_id,),
        ).fetchone()


def ensure_pg_user_for_sqlite_user(sqlite_user_id: int) -> dict:
    row = get_pg_user_by_sqlite_id(sqlite_user_id)
    if row:
        return row

    # pg_user_sync should normally create this row.
    # If missing, fail loudly so the bridge sync can be fixed.
    raise RuntimeError(f"No Postgres user found for sqlite_id={sqlite_user_id}")
```

Acceptance criteria:

- Auth connect fails clearly if `pg_user_sync` has not synced the user.
- Extension cookie always uses Postgres UUID.

---

### Task B6 - Add Pydantic models

Create `api/services/extension_sync/models.py`.

```python
from typing import Any, Literal
from pydantic import BaseModel, Field

Platform = Literal["leetcode", "codeforces", "gfg", "crackedin_practice"]


class ExtensionMeResponse(BaseModel):
    user_id: str
    email: str | None = None
    display_name: str | None = None


class ConnectProfileRequest(BaseModel):
    platform: Platform
    external_user_id: str | None = None
    external_handle: str
    display_name: str | None = None
    profile_url: str | None = None
    is_premium: bool = False
    raw_profile: dict[str, Any] | None = None


class SubmissionMetaItem(BaseModel):
    submission_id: int
    slug: str
    title: str
    raw_status: str
    lang: str
    lang_verbose: str | None = None
    runtime_display: str | None = None
    memory_display: str | None = None
    timestamp: int
    status_code: int | None = None
    raw_meta: dict[str, Any] | None = None


class SubmissionMetadataBatchRequest(BaseModel):
    platform: Platform = "leetcode"
    profile_handle: str
    batch: list[SubmissionMetaItem]
    cursor_offset: int | None = None
    last_batch: bool = False


class CodeDetailItem(BaseModel):
    submission_id: int
    slug: str
    code: str
    code_lang: str
    lang_verbose: str | None = None
    raw_status: str | None = None
    runtime_ms: int | None = None
    runtime_percentile: float | None = None
    memory_kb: int | None = None
    memory_percentile: float | None = None
    total_correct: int | None = None
    total_testcases: int | None = None
    last_testcase: str | None = None
    expected_output: str | None = None
    actual_output: str | None = None
    runtime_error: str | None = None
    compile_error: str | None = None
    raw_detail: dict[str, Any] | None = None


class CodeBatchRequest(BaseModel):
    platform: Platform = "leetcode"
    batch: list[CodeDetailItem]


class SyncStatePatchRequest(BaseModel):
    platform: Platform = "leetcode"
    phase: str
    cursor: dict[str, Any] | None = None
    last_error: str | None = None
```

Acceptance criteria:

- No model accepts `user_id` from the extension request body.
- `user_id` always comes from cookie auth dependency.

---

### Task B7 - Add normalization helpers

Create `api/services/extension_sync/normalization.py`.

```python
import hashlib
import re
from datetime import datetime, timezone

VERDICT_MAP = {
    "Accepted": "AC",
    "Wrong Answer": "WA",
    "Time Limit Exceeded": "TLE",
    "Memory Limit Exceeded": "MLE",
    "Runtime Error": "RE",
    "Compile Error": "CE",
    "Output Limit Exceeded": "OTHER",
}


def normalize_verdict(raw_status: str | None) -> str:
    if not raw_status:
        return "OTHER"
    return VERDICT_MAP.get(raw_status.strip(), "OTHER")


def ts_to_datetime(ts: int) -> datetime:
    return datetime.fromtimestamp(int(ts), tz=timezone.utc)


def code_hash(code: str) -> str:
    return hashlib.sha256(code.encode("utf-8")).hexdigest()


def parse_runtime_ms(display: str | None) -> int | None:
    if not display:
        return None
    match = re.search(r"(\d+)", display)
    return int(match.group(1)) if match else None


def parse_memory_kb(display: str | None) -> int | None:
    if not display:
        return None
    raw = display.strip().lower()
    match = re.search(r"([\d.]+)", raw)
    if not match:
        return None
    value = float(match.group(1))
    if "mb" in raw:
        return int(value * 1024)
    if "kb" in raw:
        return int(value)
    return None
```

Acceptance criteria:

- Store normalized verdict and raw status separately.
- Runtime/memory parsing is best-effort.

---

### Task B8 - Add sensitive payload validator

Create `api/services/extension_sync/validators.py`.

```python
from fastapi import HTTPException

FORBIDDEN_RAW_KEYS = {
    "cookie",
    "cookies",
    "authorization",
    "headers",
    "csrf",
    "csrftoken",
    "leetcode_session",
    "localstorage",
    "sessionstorage",
}


def assert_no_sensitive_keys(obj: object) -> None:
    if isinstance(obj, dict):
        for key, value in obj.items():
            if key.lower() in FORBIDDEN_RAW_KEYS:
                raise HTTPException(status_code=400, detail=f"Sensitive key not allowed: {key}")
            assert_no_sensitive_keys(value)
    elif isinstance(obj, list):
        for item in obj:
            assert_no_sensitive_keys(item)


def assert_non_empty_batch(batch: list) -> None:
    if not batch:
        raise HTTPException(status_code=400, detail="Batch cannot be empty")
```

Acceptance criteria:

- Backend rejects uploaded cookies, headers, CSRF, or auth data.
- Backend never stores raw browser request objects.

---

### Task B9 - Add profile service

Create `api/services/extension_sync/profiles.py`.

Use `user_id UUID`, not `app_user_id`.

```sql
INSERT INTO user_data.user_coding_profiles (
  user_id,
  platform,
  external_user_id,
  external_handle,
  display_name,
  profile_url,
  is_premium,
  is_connected,
  sync_enabled,
  raw_profile,
  updated_at
) VALUES (
  %(user_id)s,
  %(platform)s,
  %(external_user_id)s,
  %(external_handle)s,
  %(display_name)s,
  %(profile_url)s,
  %(is_premium)s,
  TRUE,
  TRUE,
  %(raw_profile)s,
  now()
)
ON CONFLICT (user_id, platform) DO UPDATE SET
  external_user_id = EXCLUDED.external_user_id,
  external_handle = EXCLUDED.external_handle,
  display_name = EXCLUDED.display_name,
  profile_url = EXCLUDED.profile_url,
  is_premium = EXCLUDED.is_premium,
  is_connected = TRUE,
  sync_enabled = TRUE,
  raw_profile = EXCLUDED.raw_profile,
  updated_at = now()
RETURNING id;
```

Acceptance criteria:

- One profile per `(user_id, platform)`.
- Reconnect updates existing profile.

---

### Task B10 - Add metadata ingestion service

Create `api/services/extension_sync/ingest_submissions.py`.

Responsibilities:

1. Validate batch.
2. Upsert catalog stubs for unknown slugs.
3. Upsert `user_data.user_coding_submissions` with `has_code=false`.
4. Recompute `user_data.user_coding_problems` aggregates for affected slugs.
5. Publish `submission.received` events for unknown/incomplete catalog rows.
6. Update `system.sync_state`.

Important SQL shape:

```sql
INSERT INTO user_data.user_coding_submissions (
  platform,
  submission_id,
  user_id,
  profile_id,
  slug,
  verdict,
  raw_status,
  status_code,
  lang,
  lang_verbose,
  runtime_ms,
  memory_kb,
  ts,
  capture_source,
  has_code,
  raw_meta,
  updated_at
) VALUES (
  %(platform)s,
  %(submission_id)s,
  %(user_id)s,
  %(profile_id)s,
  %(slug)s,
  %(verdict)s,
  %(raw_status)s,
  %(status_code)s,
  %(lang)s,
  %(lang_verbose)s,
  %(runtime_ms)s,
  %(memory_kb)s,
  %(ts)s,
  'historical_sync',
  FALSE,
  %(raw_meta)s,
  now()
)
ON CONFLICT (platform, submission_id) DO UPDATE SET
  user_id = EXCLUDED.user_id,
  profile_id = EXCLUDED.profile_id,
  slug = EXCLUDED.slug,
  verdict = EXCLUDED.verdict,
  raw_status = EXCLUDED.raw_status,
  status_code = EXCLUDED.status_code,
  lang = EXCLUDED.lang,
  lang_verbose = EXCLUDED.lang_verbose,
  runtime_ms = EXCLUDED.runtime_ms,
  memory_kb = EXCLUDED.memory_kb,
  ts = EXCLUDED.ts,
  raw_meta = EXCLUDED.raw_meta,
  updated_at = now();
```

Acceptance criteria:

- Replaying the same batch is safe.
- Duplicates do not create duplicate rows.
- Metadata sync works before code is fetched.

---

### Task B11 - Add code ingestion service

Create `api/services/extension_sync/ingest_code.py`.

Responsibilities:

1. Validate code batch.
2. Upsert `code_blobs.user_coding_submission_code`.
3. Update `user_data.user_coding_submissions.has_code=true`.
4. Update detail fields if present.
5. Publish catalog events for slugs missing problem statement.

Important SQL shape:

```sql
INSERT INTO code_blobs.user_coding_submission_code (
  platform,
  submission_id,
  code,
  code_lang,
  code_hash,
  storage_tier,
  fetched_at
) VALUES (
  %(platform)s,
  %(submission_id)s,
  %(code)s,
  %(code_lang)s,
  %(code_hash)s,
  'hot',
  now()
)
ON CONFLICT (platform, submission_id) DO UPDATE SET
  code = EXCLUDED.code,
  code_lang = EXCLUDED.code_lang,
  code_hash = EXCLUDED.code_hash,
  fetched_at = now();
```

Acceptance criteria:

- Code can be uploaded after metadata.
- Missing metadata returns a controlled error or stores only after metadata exists.
- `has_code` is true only after code blob insert succeeds.

---

### Task B12 - Add aggregate recompute service

Create `api/services/extension_sync/aggregates.py`.

For each affected `(user_id, platform, slug)`, recompute:

```text
status
first_attempt_at
first_ac_at
latest_ac_at
latest_attempt_at
attempt_count
ac_count
wa_count
tle_count
mle_count
re_count
ce_count
other_count
best_runtime_ms
best_memory_kb
best_lang
primary_submission_id
```

Recommended implementation: after each metadata/code batch, recompute only affected slugs.

Acceptance criteria:

- Dashboard can read from `user_coding_problems` without scanning all submissions.
- Aggregates are correct after replayed batches.

---

### Task B13 - Add sync state service

Create `api/services/extension_sync/sync_state.py`.

Use `system.sync_state`.

```sql
INSERT INTO system.sync_state (
  user_id,
  platform,
  phase,
  cursor,
  last_progress_at,
  last_error,
  updated_at
) VALUES (
  %(user_id)s,
  %(platform)s,
  %(phase)s,
  %(cursor)s,
  now(),
  %(last_error)s,
  now()
)
ON CONFLICT (user_id, platform) DO UPDATE SET
  phase = EXCLUDED.phase,
  cursor = EXCLUDED.cursor,
  last_progress_at = now(),
  last_error = EXCLUDED.last_error,
  updated_at = now();
```

Acceptance criteria:

- Sync cursor is visible to backend/dashboard.
- Extension can recover from local storage loss using backend state if needed.

---

### Task B14 - Add PGMQ event publisher

Create `api/services/extension_sync/catalog_events.py`.

Use existing PGMQ queue `extension_events`.

```python
from api.config import EXTENSION_PGMQ_QUEUE


def publish_submission_received(conn, *, user_id: str, platform: str, slug: str, submission_id: int) -> None:
    conn.execute(
        """
        SELECT pgmq.send(
          %s,
          jsonb_build_object(
            'type', 'submission.received',
            'user_id', %s,
            'platform', %s,
            'slug', %s,
            'submission_id', %s
          )
        )
        """,
        (EXTENSION_PGMQ_QUEUE, str(user_id), platform, slug, submission_id),
    )
```

Acceptance criteria:

- Unknown/incomplete problem slugs publish `submission.received`.
- Messages are sent to `extension_events`.

---

### Task B15 - Add catalog worker

Create `api/services/extension_sync/catalog_worker.py`.

Responsibilities:

1. Read messages from `extension_events`.
2. Process only `type = submission.received`.
3. Check `catalog.coding_problems` for missing `statement_md` or `sync_status != complete`.
4. Fetch problem detail from LeetCode GraphQL.
5. Convert HTML to markdown.
6. Update `catalog.coding_problems`.
7. Delete processed PGMQ message.
8. Leave failed message visible for retry.

PGMQ read shape:

```sql
SELECT * FROM pgmq.read('extension_events', 30, 10);
```

Delete after success:

```sql
SELECT pgmq.delete('extension_events', %(msg_id)s);
```

Acceptance criteria:

- Catalog worker fills missing statements.
- Failed catalog fetches do not delete the message.
- Worker can be disabled with `CATALOG_WORKER_ENABLED=false`.

---

### Task B16 - Add extension routes

Create `api/routes/extension_sync.py`.

Route list:

```text
POST   /api/extension/auth/connect
POST   /api/extension/auth/logout
GET    /api/extension/me
POST   /api/extension/platforms/{platform}/connect
GET    /api/extension/platforms/{platform}/sync-state
PATCH  /api/extension/platforms/{platform}/sync-state
POST   /api/extension/platforms/{platform}/submissions
POST   /api/extension/platforms/{platform}/code
POST   /api/extension/platforms/{platform}/pause
POST   /api/extension/platforms/{platform}/resume
```

Route auth rules:

```text
/auth/connect:
  uses existing web app auth dependency, then sets extension cookie

/auth/logout:
  clears extension cookie

all other /api/extension/* routes:
  use get_current_extension_pg_user_id
```

Example route:

```python
from uuid import UUID
from fastapi import APIRouter, Depends, Response

from api.auth import get_current_user_id
from api.services.extension_sync.cookie_auth import (
    create_extension_cookie_jwt,
    set_extension_cookie,
    clear_extension_cookie,
    get_current_extension_pg_user_id,
)
from api.services.extension_sync.pg_user_bridge import ensure_pg_user_for_sqlite_user
from api.services.extension_sync.models import (
    ConnectProfileRequest,
    SubmissionMetadataBatchRequest,
    CodeBatchRequest,
    SyncStatePatchRequest,
)

router = APIRouter(prefix="/api/extension", tags=["extension"])


@router.post("/auth/connect")
def connect_extension(response: Response, sqlite_user_id: int = Depends(get_current_user_id)):
    pg_user = ensure_pg_user_for_sqlite_user(sqlite_user_id)
    token = create_extension_cookie_jwt(pg_user["id"])
    set_extension_cookie(response, token)
    return {"ok": True, "user_id": str(pg_user["id"])}


@router.post("/auth/logout")
def logout_extension(response: Response):
    clear_extension_cookie(response)
    return {"ok": True}


@router.get("/me")
def extension_me(user_id: UUID = Depends(get_current_extension_pg_user_id)):
    return {"ok": True, "user_id": str(user_id)}
```

Mount in `api/main.py`:

```python
from api.routes import extension_sync
app.include_router(extension_sync.router)
```

Acceptance criteria:

- `/api/extension/me` returns 401 without cookie.
- `/api/extension/me` returns Postgres user UUID with valid cookie.
- Ingestion routes never accept user ID from request body.

---

### Task B17 - Update CORS

FastAPI must allow credentialed requests from the web app and Chrome extension origin.

Example:

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "https://crackedin.app",
        "https://www.crackedin.app",
        "chrome-extension://<production-extension-id>",
    ],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

Do not use wildcard origins with credentials.

Acceptance criteria:

- Extension fetch with `credentials: "include"` works.
- Browser accepts and sends HttpOnly cookie.

---

## 7. Frontend/Web App Tasks

### Task W1 - Add extension connect page

Add a page in the web app:

```text
/extension/connect
```

Behavior:

1. User must be logged in to CrackedIn.
2. Page explains that CrackedIn will connect the browser extension.
3. User clicks `Enable Extension Sync`.
4. Web app calls `POST /api/extension/auth/connect` with credentials.
5. Backend sets HttpOnly extension cookie.
6. Page tells user to open the extension popup.

Example fetch:

```ts
await fetch(`${API_BASE}/api/extension/auth/connect`, {
  method: "POST",
  credentials: "include",
});
```

Acceptance criteria:

- Logged-in user can set extension cookie.
- Logged-out user is redirected to login.
- Extension popup can call `/api/extension/me` after connect.

---

## 8. Extension Implementation Tasks

### Task E1 - Scaffold extension

Create the `extension/` project with TypeScript and Vite.

Required manifest permissions:

```json
{
  "manifest_version": 3,
  "name": "CrackedIn",
  "version": "0.1.0",
  "permissions": ["storage", "alarms"],
  "host_permissions": [
    "https://leetcode.com/*",
    "https://crackedin.app/*",
    "https://api.crackedin.app/*"
  ],
  "background": {
    "service_worker": "src/service-worker/index.js",
    "type": "module"
  },
  "action": {
    "default_popup": "src/popup/popup.html"
  },
  "content_scripts": [
    {
      "matches": ["https://leetcode.com/problems/*"],
      "js": ["src/content-scripts/leetcodeLivePlaceholder.js"],
      "run_at": "document_idle"
    }
  ]
}
```

Do not request the Chrome `cookies` permission.

Acceptance criteria:

- Extension builds.
- Popup opens.
- Service worker starts.

---

### Task E2 - Add CrackedIn API client

Create `extension/src/api/crackedinApi.ts`.

Every backend call must include credentials.

```ts
const API_BASE = "https://api.crackedin.app";

async function apiFetch<T>(path: string, options: RequestInit = {}): Promise<T> {
  const res = await fetch(`${API_BASE}${path}`, {
    ...options,
    credentials: "include",
    headers: {
      "Content-Type": "application/json",
      ...(options.headers || {}),
    },
  });

  if (!res.ok) {
    throw new Error(`CrackedIn API error ${res.status}`);
  }

  return res.json() as Promise<T>;
}

export async function getExtensionMe() {
  return apiFetch<{ ok: boolean; user_id: string }>("/api/extension/me");
}

export async function connectPlatform(payload: unknown) {
  return apiFetch("/api/extension/platforms/leetcode/connect", {
    method: "POST",
    body: JSON.stringify(payload),
  });
}

export async function uploadSubmissionMetadata(payload: unknown) {
  return apiFetch("/api/extension/platforms/leetcode/submissions", {
    method: "POST",
    body: JSON.stringify(payload),
  });
}

export async function uploadCodeBatch(payload: unknown) {
  return apiFetch("/api/extension/platforms/leetcode/code", {
    method: "POST",
    body: JSON.stringify(payload),
  });
}
```

Acceptance criteria:

- No Authorization header is used.
- No JWT is stored in extension storage.
- All calls include `credentials: "include"`.

---

### Task E3 - Add local sync state

Create `extension/src/service-worker/state.ts`.

```ts
export type SyncPhase =
  | "idle"
  | "auth_checking"
  | "A_bulk"
  | "B_fg"
  | "B_bg"
  | "paused"
  | "auth_required"
  | "failed"
  | "done";

export type FailedCodeItem = {
  submission_id: number;
  slug?: string;
  reason: string;
  retry_count: number;
  next_retry_at: number;
  last_error?: string;
};

export type LocalSyncState = {
  platform: "leetcode";
  phase: SyncPhase;
  cursor: {
    offset?: number;
    foregroundCodeQueue?: number[];
    backgroundCodeQueue?: number[];
    failedCodeQueue?: FailedCodeItem[];
    metadataFetched?: number;
    codeDone?: number;
    codeFailed?: number;
  };
  pauseRequested?: boolean;
  lastProgressAt?: number;
  lastError?: string;
};
```

State must be stored in `chrome.storage.local` after every batch.

Acceptance criteria:

- Service worker can die and resume.
- Popup can read progress at any time.

---

### Task E4 - Add LeetCode adapter

Create `extension/src/platforms/leetcode.ts`.

Responsibilities:

1. Check auth with `userStatus`.
2. Fetch metadata pages with `submissionList` aliases.
3. Fetch code details with `submissionDetails` aliases.
4. Never read LeetCode cookies directly.

All LeetCode fetches use:

```ts
fetch("https://leetcode.com/graphql", {
  method: "POST",
  credentials: "include",
  headers: {
    "Content-Type": "application/json",
  },
  body: JSON.stringify({ query, variables }),
});
```

Acceptance criteria:

- Logged-in LeetCode user returns username.
- Logged-out user returns auth-required state.
- Adapter does not use Chrome cookies API.

---

### Task E5 - Implement GraphQL aliasing

Create `extension/src/service-worker/graphql/aliasing.ts`.

Metadata batch: 10 aliased pages per request.

```graphql
query bulkPages {
  p0: submissionList(offset: 0, limit: 20) { hasNext submissions { id title titleSlug statusDisplay lang runtime memory timestamp } }
  p1: submissionList(offset: 20, limit: 20) { hasNext submissions { id title titleSlug statusDisplay lang runtime memory timestamp } }
}
```

Code batch: 10 aliased `submissionDetails` per request.

```graphql
query codeBatch {
  s0: submissionDetails(submissionId: 123) { code lang { name verboseName } runtime memory runtimePercentile memoryPercentile totalCorrect totalTestcases statusDisplay question { titleSlug } }
}
```

Acceptance criteria:

- One metadata request can return up to 200 rows.
- One code request can return up to 10 code blobs.
- Response aliases map back to submission IDs.

---

### Task E6 - Implement Phase A metadata sync job

Create `extension/src/service-worker/jobs/metadataSyncJob.ts`.

Algorithm:

```text
read local state offset, default 0
loop until done or paused:
  build 10-page aliased submissionList query
  fetch LeetCode GraphQL with credentials include
  flatten rows
  upload sanitized rows to /api/extension/platforms/leetcode/submissions
  update local cursor offset += 200
  write state to chrome.storage.local
  if any page hasNext=false:
    sort all known submission IDs by timestamp desc
    foregroundCodeQueue = latest 100
    backgroundCodeQueue = remaining
    phase = B_fg
    persist state
    stop Phase A
```

Acceptance criteria:

- Fetches full historical metadata.
- Can resume from offset after service worker restart.
- Does not fetch code in Phase A.

---

### Task E7 - Implement Phase B foreground code job

Create `extension/src/service-worker/jobs/foregroundCodeJob.ts`.

Algorithm:

```text
while foregroundCodeQueue not empty and not paused:
  take 10 submission IDs
  fetch submissionDetails via aliases
  upload code batch
  remove successful IDs from queue
  move failed IDs to failedCodeQueue
  update codeDone/codeFailed
  persist local state

when queue empty:
  phase = B_bg
  persist state
  show core sync complete
```

Acceptance criteria:

- Latest 100 code blobs are fetched before core sync is marked complete.
- Failed items do not block the whole sync.

---

### Task E8 - Implement Phase C background code backfill

Create `extension/src/service-worker/jobs/backgroundCodeJob.ts`.

Algorithm:

```text
while backgroundCodeQueue not empty and not paused:
  take 10 submission IDs
  fetch submissionDetails via aliases
  upload code batch
  remove successes
  move failures to failedCodeQueue
  persist state
  wait 2 seconds between batches

when background queue empty:
  phase = done
  persist state
```

Acceptance criteria:

- User can use the app while background code backfill continues.
- Backfill is resumable.
- Backfill does not trigger intelligence/code analysis.

---

### Task E9 - Implement failed-code retry job

Create `extension/src/service-worker/jobs/failedCodeRetryJob.ts`.

Retry behavior:

| Retry count | Next retry |
|---:|---|
| 1 | 30 seconds |
| 2 | 2 minutes |
| 3 | 10 minutes |
| 4+ | manual/delta repair later |

Acceptance criteria:

- Temporary failures are retried.
- Persistent failures remain visible in progress state.
- Retry queue survives service worker restart.

---

### Task E10 - Implement popup UI

Popup states:

| State | Message |
|---|---|
| Not connected | Connect CrackedIn |
| Connected, LC missing | Open LeetCode and log in |
| Ready | Sync my LeetCode |
| A_bulk | Finding your LeetCode submissions |
| B_fg | Fetching recent submitted code |
| B_bg | Core sync complete. Background code backfill running |
| paused | Sync paused |
| failed | Sync paused due to error |
| done | Sync complete |

When not connected to CrackedIn, the popup opens:

```text
https://crackedin.app/extension/connect
```

Acceptance criteria:

- Popup never runs sync itself.
- Popup sends messages to service worker.
- Popup reads progress from storage.

---

## 9. Stage 1 Backend Route Contracts

### `POST /api/extension/platforms/leetcode/connect`

Request:

```json
{
  "platform": "leetcode",
  "external_user_id": "12345",
  "external_handle": "leetcode_user",
  "display_name": "leetcode_user",
  "profile_url": "https://leetcode.com/u/leetcode_user/",
  "is_premium": false,
  "raw_profile": {
    "username": "leetcode_user",
    "isSignedIn": true
  }
}
```

Response:

```json
{
  "ok": true,
  "profile_id": 1
}
```

### `POST /api/extension/platforms/leetcode/submissions`

Request:

```json
{
  "platform": "leetcode",
  "profile_handle": "leetcode_user",
  "cursor_offset": 200,
  "last_batch": false,
  "batch": [
    {
      "submission_id": 123456789,
      "slug": "two-sum",
      "title": "Two Sum",
      "raw_status": "Accepted",
      "lang": "cpp",
      "lang_verbose": "C++",
      "runtime_display": "4 ms",
      "memory_display": "12.5 MB",
      "timestamp": 1710000000
    }
  ]
}
```

Response:

```json
{
  "ok": true,
  "inserted_or_updated": 1,
  "unknown_slugs_queued": 1
}
```

### `POST /api/extension/platforms/leetcode/code`

Request:

```json
{
  "platform": "leetcode",
  "batch": [
    {
      "submission_id": 123456789,
      "slug": "two-sum",
      "code": "class Solution { ... }",
      "code_lang": "cpp",
      "runtime_ms": 4,
      "memory_kb": 12800,
      "total_correct": 57,
      "total_testcases": 57
    }
  ]
}
```

Response:

```json
{
  "ok": true,
  "inserted_or_updated": 1
}
```

---

## 10. Stage 2 and Stage 3 Readiness Hooks

Do not implement Stage 2 live capture or Stage 3 delta repair in this pass.

Keep these compatibility points:

1. Database has `capture_source` values that can later include `live_capture` and `delta_repair`.
2. Database has `last_live_capture_at` and `last_delta_sync_at` on profiles.
3. Extension manifest includes a placeholder content script file.
4. Backend routes are platform-generic: `/api/extension/platforms/{platform}/...`.
5. Service worker architecture supports future scheduler jobs.
6. `system.sync_state.phase` can later include `live` and `delta_syncing`.

---

## 11. Security Rules

1. The extension never reads CrackedIn cookies.
2. The extension never reads LeetCode cookies.
3. The extension never stores JWTs in `chrome.storage.local`.
4. Backend cookies are HttpOnly.
5. Backend cookies are Secure in production.
6. Backend derives user ID only from the extension cookie.
7. Request body must never include trusted `user_id`.
8. Supabase service credentials never appear in extension code.
9. LeetCode request headers/cookies/CSRF tokens are never uploaded.
10. Raw payload validators reject sensitive keys.
11. All backend writes filter by authenticated Postgres UUID.
12. CORS allows only trusted web/extension origins with credentials.

---

## 12. Testing Plan

### Backend auth tests

- `/api/extension/me` returns 401 without cookie.
- `/api/extension/me` returns 401 for invalid cookie.
- `/api/extension/me` returns user UUID for valid cookie.
- `/api/extension/auth/logout` clears cookie.
- Ingestion routes reject unauthenticated requests.

### Backend ingestion tests

- Profile connect upserts by `(user_id, platform)`.
- Metadata batch inserts submissions.
- Replaying metadata batch is idempotent.
- Code batch inserts code blob.
- Code batch updates `has_code=true`.
- Aggregates update after metadata batch.
- Sensitive keys are rejected.
- PGMQ event publishes for missing catalog statements.

### Extension tests

- API client uses `credentials: "include"`.
- API client does not use Authorization header.
- Extension storage contains no JWT.
- LeetCode adapter uses `credentials: "include"`.
- Metadata sync resumes from saved offset.
- Foreground code fetch processes latest 100.
- Background queue survives service worker restart.
- Failed-code retry queue persists.

### Manual end-to-end test

1. Log into CrackedIn web app.
2. Open `/extension/connect`.
3. Click `Enable Extension Sync`.
4. Open extension popup.
5. Confirm `/api/extension/me` succeeds.
6. Log into LeetCode in same browser profile.
7. Start Stage 1 sync.
8. Confirm metadata rows appear in `user_data.user_coding_submissions`.
9. Confirm aggregates appear in `user_data.user_coding_problems`.
10. Confirm latest 100 code blobs appear in `code_blobs.user_coding_submission_code`.
11. Confirm PGMQ receives `submission.received` events.
12. Confirm catalog worker updates `catalog.coding_problems`.

---

## 13. Implementation Order

Follow this exact order.

### Phase 0 - Verification

1. Verify Supabase tables exist.
2. Verify `public.users.sqlite_id` exists.
3. Verify PGMQ queue `extension_events` exists.
4. Verify `api/auth_pg.py` exists.

### Phase 1 - Backend auth

1. Add config values for extension cookie auth.
2. Add `cookie_auth.py`.
3. Add `pg_user_bridge.py`.
4. Extend/wrap `api/auth_pg.py`.
5. Add `/api/extension/auth/connect`.
6. Add `/api/extension/me`.
7. Add `/api/extension/auth/logout`.
8. Update CORS.
9. Add tests for cookie auth.

### Phase 2 - Backend ingestion foundation

1. Add Postgres connection helper.
2. Add Pydantic models.
3. Add validators.
4. Add normalization helpers.
5. Add profile service.
6. Add sync state service.
7. Add aggregate service.
8. Add PGMQ publisher.

### Phase 3 - Backend ingestion routes

1. Add platform connect route.
2. Add metadata ingestion route.
3. Add code ingestion route.
4. Add sync-state get/patch routes.
5. Add pause/resume routes.
6. Mount router in `api/main.py`.
7. Add backend tests.

### Phase 4 - Catalog worker

1. Add PGMQ read loop.
2. Add LeetCode problem detail fetch.
3. Add HTML to markdown conversion.
4. Update catalog rows.
5. Delete processed messages.
6. Add worker startup guarded by `CATALOG_WORKER_ENABLED`.

### Phase 5 - Web app connect page

1. Add `/extension/connect` page.
2. Require logged-in CrackedIn user.
3. Call `/api/extension/auth/connect`.
4. Show success and instructions to open popup.

### Phase 6 - Extension scaffold

1. Create extension project.
2. Add manifest.
3. Add popup shell.
4. Add service worker shell.
5. Add API client with credentials include.
6. Add local storage state.

### Phase 7 - LeetCode adapter

1. Add `userStatus` auth check.
2. Add aliased `submissionList` query builder.
3. Add aliased `submissionDetails` query builder.
4. Add response flatteners.
5. Add sanitizers.

### Phase 8 - Stage 1 jobs

1. Implement metadata sync job.
2. Implement foreground code job.
3. Implement background code job.
4. Implement failed-code retry job.
5. Implement pause/resume.
6. Persist cursor after every batch.

### Phase 9 - Popup progress UI

1. Show CrackedIn auth state.
2. Show LeetCode auth state.
3. Show phase progress.
4. Show core sync complete at end of foreground code fetch.
5. Show background backfill progress.
6. Show errors/retry states.

### Phase 10 - End-to-end validation

1. Run auth flow.
2. Run Phase A metadata sync.
3. Run Phase B foreground code fetch.
4. Run background backfill on a small queue.
5. Verify database rows.
6. Verify PGMQ events.
7. Verify no sensitive data stored.

---

## 14. Acceptance Criteria for Stage 1 Complete

Stage 1 is complete only when:

```text
[ ] Extension can authenticate through HttpOnly cookie.
[ ] Extension stores no JWT/access token in chrome.storage.local.
[ ] Extension can detect CrackedIn auth through /api/extension/me.
[ ] Extension can detect LeetCode auth through userStatus.
[ ] User can connect LeetCode profile.
[ ] Phase A imports full historical metadata.
[ ] Phase B foreground imports latest 100 code blobs.
[ ] Background code backfill starts after core sync complete.
[ ] Failed-code retry queue works.
[ ] Backend writes to Supabase using public.users.id UUID.
[ ] Backend never trusts user_id from request body.
[ ] user_coding_problems aggregates are populated.
[ ] system.sync_state is updated after batches.
[ ] PGMQ extension_events receives submission.received events.
[ ] Catalog worker processes events and updates coding_problems.
[ ] Popup displays progress accurately.
[ ] Sync resumes after service worker restart.
[ ] No LeetCode cookies, CSRF tokens, headers, or browser storage are uploaded.
[ ] Stage 2/3 compatibility fields remain untouched but available.
```

---

## 15. Explicit Non-Goals

Do not implement in Stage 1:

1. Live capture during LeetCode submits.
2. Delta repair.
3. Code analysis worker.
4. Weakness intelligence worker.
5. Recommendation precomputation.
6. Embeddings/problem similarity.
7. Direct extension writes to Supabase.
8. Chrome cookies permission.
9. LeetCode cookie reading.
10. JWT storage in extension local storage.
11. Extension auth code exchange.
12. Refresh-token based extension sessions.
13. `system.extension_sessions`.

---

## 16. Implementation Prompt for Repo Agent

Use this prompt when handing implementation to a coding agent:

```text
Implement Stage 1 of the CrackedIn extension sync system using the provided implementation plan.

Important auth update:
- Use JWT stored in an HttpOnly cookie for extension auth.
- Do not store JWTs in chrome.storage.local.
- Do not create extension auth code tables.
- Do not create extension session tables.
- Extension API calls must use fetch(..., { credentials: "include" }).
- Extension routes must resolve the authenticated user to public.users.id UUID through api/auth_pg.py / cookie auth helper.

Database state:
- Supabase extension sync tables already exist.
- PGMQ queue extension_events already exists.
- Use user_id UUID references to public.users(id).

Build order:
1. Verify Postgres tables and PGMQ queue.
2. Add backend cookie auth helpers and /api/extension/auth/connect, /api/extension/me, /api/extension/auth/logout.
3. Add backend Supabase services and extension ingestion routes.
4. Add PGMQ catalog event publisher and catalog worker.
5. Add web /extension/connect page.
6. Scaffold extension.
7. Add CrackedIn API client with credentials include.
8. Add LeetCode adapter with GraphQL aliasing.
9. Implement Phase A metadata sync.
10. Implement Phase B latest-100 code fetch.
11. Implement background code backfill and failed-code retry.
12. Implement popup progress UI.
13. Add tests and run end-to-end validation.

Do not implement Stage 2 live capture or Stage 3 delta repair in this pass. Keep placeholders only.
```
