# CrackedIn Extension Sync — Stage 1 Implementation Instructions

**Product:** CrackedIn / InterviewPrep.ai  
**Feature:** Browser extension based LeetCode corpus sync  
**Primary focus:** Stage 1 only — first-time historical sync  
**Future compatibility:** Keep architecture ready for Stage 2 live capture and Stage 3 delta repair, but do not implement them in this pass.  
**Audience:** Implementation LLM, repo agent, backend engineer, extension engineer, intern.  
**Status:** Final implementation instruction file.

---

## 0. Purpose of This Document

This file is the build instruction document for implementing the CrackedIn extension sync system on top of the existing Interview Prep AI repository.

It should be treated as the implementation source of truth for Stage 1. The implementation agent should follow this document in order and should not re-litigate architecture decisions.

The target outcome is:

```text
A Chrome/Chromium extension connected to the existing CrackedIn backend that can:

1. authenticate as an existing CrackedIn user,
2. verify the user's LeetCode browser session,
3. fetch the user's full historical LeetCode submission metadata,
4. fetch submitted code for the latest 100 submissions in foreground,
5. backfill remaining submitted code in background,
6. retry failed code fetches safely,
7. upload sanitized batches to FastAPI,
8. store the corpus in Supabase Postgres using the finalized extension sync schema,
9. update local and backend sync state after every batch,
10. publish catalog events through PGMQ for missing problem statements.
```

---

## 1. Final Decisions

These are locked decisions. Do not propose alternatives during implementation.

| Area | Final decision |
|---|---|
| Product name | CrackedIn |
| Current app repo | Extend the current Interview Prep AI repo |
| Extension location | Add a new `extension/` directory inside the existing repo |
| Backend | Add extension-specific routes/modules to the existing FastAPI backend |
| Existing database | Keep SQLite for current users, auth, chat, interview data, existing recommendations |
| Extension corpus database | Use Supabase Postgres |
| Extension DB writes | Extension never writes directly to Supabase; FastAPI writes to Supabase |
| User identity | Existing SQLite `users.id` is the app user identity |
| Supabase user key | Use `app_user_id INTEGER`, derived from JWT user ID |
| LeetCode auth | Use user's existing browser session through extension requests to LeetCode |
| LeetCode cookie handling | Never read, store, or upload LeetCode cookies |
| Chrome background execution | One Manifest V3 service worker with multiple resumable jobs |
| Stage 1 metadata source | LeetCode `submissionList(offset, limit)` |
| Stage 1 code source | LeetCode `submissionDetails(submissionId)` |
| Batching | Use GraphQL aliasing |
| Foreground code fetch | Fetch latest 100 submissions by timestamp |
| Remaining code | Background backfill |
| Failed code | Persistent failed-code retry queue |
| Event queue | Supabase PGMQ |
| Stage 1 backend worker | Catalog worker only |
| Intelligence/code analysis | Deferred; do not implement in Stage 1 |
| Live capture | Stage 2 readiness only; do not implement now |
| Delta repair | Stage 3 readiness only; do not implement now |

---

## 2. Current Repo Context

The existing repo is assumed to have this shape:

```text
interview-prep-ai/
├── api/                  FastAPI backend
│   ├── main.py           App entrypoint and router mounting
│   ├── auth.py           JWT helpers and current-user dependency
│   ├── config.py         Environment config
│   ├── database.py       SQLite connection helper
│   ├── routes/           Existing FastAPI routers
│   └── services/         Existing backend services
│
├── web/                  Next.js frontend
├── db/                   SQLite schema/init scripts
├── scripts/              Data scripts
├── src/                  Existing scraper/shared scripts
└── data/                 Local SQLite DB, git-ignored
```

The existing backend uses:

```text
FastAPI + SQLite + JWT auth
```

The existing user identity is:

```text
SQLite users.id INTEGER
```

The extension implementation must not require migrating the whole app to Supabase. Supabase is used only for extension corpus data.

---

## 3. Target Repository Structure

Add these folders and files.

```text
interview-prep-ai/
├── api/
│   ├── routes/
│   │   └── extension_sync.py
│   │
│   ├── services/
│   │   └── extension_sync/
│   │       ├── __init__.py
│   │       ├── supabase_db.py
│   │       ├── token_service.py
│   │       ├── models.py
│   │       ├── validators.py
│   │       ├── normalization.py
│   │       ├── profiles.py
│   │       ├── ingest_submissions.py
│   │       ├── ingest_code.py
│   │       ├── aggregates.py
│   │       ├── sync_state.py
│   │       ├── catalog_events.py
│   │       ├── catalog_worker.py
│   │       └── platforms/
│   │           ├── __init__.py
│   │           ├── base.py
│   │           └── leetcode.py
│   │
│   └── main.py
│
├── db/
│   └── extension_auth_migration.sql
│
├── supabase/
│   └── migrations/
│       └── 001_extension_sync_stage1.sql
│
├── extension/
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   ├── manifest.json
│   └── src/
│       ├── shared/
│       │   ├── types.ts
│       │   ├── constants.ts
│       │   └── storage.ts
│       │
│       ├── api/
│       │   └── crackedinApi.ts
│       │
│       ├── platforms/
│       │   ├── types.ts
│       │   └── leetcode.ts
│       │
│       ├── service-worker/
│       │   ├── index.ts
│       │   ├── state.ts
│       │   ├── scheduler.ts
│       │   ├── jobs/
│       │   │   ├── authCheckJob.ts
│       │   │   ├── metadataSyncJob.ts
│       │   │   ├── foregroundCodeJob.ts
│       │   │   ├── backgroundCodeJob.ts
│       │   │   └── failedCodeRetryJob.ts
│       │   └── graphql/
│       │       ├── aliasing.ts
│       │       └── leetcodeQueries.ts
│       │
│       ├── popup/
│       │   ├── popup.html
│       │   ├── popup.ts
│       │   └── popup.css
│       │
│       └── content-scripts/
│           └── leetcodeLivePlaceholder.ts
│
└── docs/
    └── extension-sync-stage1-implementation.md
```

The Stage 2 content script file is only a placeholder. Do not implement live capture in Stage 1.

---

## 4. Runtime Architecture

```text
Chrome Extension
├── Popup
│   ├── shows connection status
│   ├── starts Stage 1 sync
│   ├── shows progress
│   ├── exposes pause/resume
│   └── never performs sync itself
│
├── One Manifest V3 service worker
│   ├── validates CrackedIn extension auth
│   ├── validates LeetCode session via userStatus
│   ├── runs Phase A metadata sync
│   ├── runs Phase B foreground latest-100 code fetch
│   ├── runs Phase C background code backfill
│   ├── runs failed-code retry queue
│   ├── persists local cursor after every batch
│   └── uploads sanitized data to FastAPI
│
└── Content script placeholder
    └── reserved for Stage 2 live capture

FastAPI Backend
├── Existing app routes/auth
├── New /api/extension/* routes
├── SQLite auth/session lookup
├── Supabase Postgres writer
├── ingestion validators
├── normalization layer
├── aggregate updater
├── sync-state manager
├── PGMQ event publisher
└── catalog worker

Storage
├── SQLite
│   ├── existing users
│   ├── existing auth
│   ├── extension auth sessions
│   └── extension auth codes
│
└── Supabase Postgres
    ├── catalog.coding_problems
    ├── user_data.user_coding_profiles
    ├── user_data.user_coding_problems
    ├── user_data.user_coding_submissions
    ├── code_blobs.user_coding_submission_code
    └── system.sync_state
```

---

## 5. Stage 1 Scope

### 5.1 Implement in Stage 1

Build these now:

1. Extension auth connection to CrackedIn.
2. LeetCode auth check from extension service worker.
3. First-time historical metadata sync.
4. GraphQL alias batching for `submissionList`.
5. Backend metadata ingestion.
6. User solve-graph aggregate updates.
7. Latest 100 code fetch in foreground.
8. GraphQL alias batching for `submissionDetails`.
9. Backend code ingestion.
10. Background code backfill queue.
11. Failed-code retry queue.
12. Local sync state in `chrome.storage.local`.
13. Backend sync state in Supabase.
14. PGMQ `submission.received` queue event.
15. Catalog worker for missing problem details.
16. Popup progress display.
17. Pause/resume between batches.
18. Basic tests and manual QA.

### 5.2 Do Not Implement in Stage 1

These are future-ready only:

1. Live capture of future submissions.
2. Delta repair for missed submissions.
3. Code analyzer worker.
4. Weakness intelligence tables.
5. Recommendation worker.
6. Embeddings or problem similarity.
7. S3 archival of code blobs.
8. Multi-platform sync implementation beyond interface readiness.
9. Extension direct Supabase access.
10. Server-side LeetCode private scraping with uploaded cookies.

---

## 6. Stage 1 User Flow

```text
1. User logs into CrackedIn web app.
2. User installs CrackedIn extension.
3. Extension connects to CrackedIn through extension auth flow.
4. Extension checks LeetCode login state with userStatus.
5. If LeetCode is signed in, popup shows detected username.
6. User clicks "Sync my LeetCode".
7. Service worker starts Phase A metadata sync.
8. Backend stores all metadata and builds solve graph.
9. Service worker starts Phase B foreground code fetch for latest 100 submissions.
10. Backend stores latest 100 code blobs.
11. Popup marks core sync complete.
12. Service worker starts background code backfill.
13. Failed code fetches go into retry queue.
14. Catalog worker fills missing problem statements from PGMQ events.
```

---

## 7. Environment Variables

Add these to `.env.example` and deployment config.

```env
# Existing values already used by backend
JWT_SECRET=
JWT_EXPIRATION_HOURS=72
DATABASE_PATH=data/interview_prep.db

# Extension token config
EXTENSION_ACCESS_TOKEN_MINUTES=30
EXTENSION_REFRESH_TOKEN_DAYS=60
EXTENSION_AUTH_CODE_MINUTES=5
EXTENSION_TOKEN_SECRET=

# Supabase/Postgres for extension corpus
EXTENSION_DATABASE_URL=postgresql://...
SUPABASE_PROJECT_URL=https://...
SUPABASE_SERVICE_ROLE_KEY=...

# PGMQ
EXTENSION_PGMQ_QUEUE=extension_events
CATALOG_WORKER_ENABLED=true

# LeetCode catalog worker
LEETCODE_GRAPHQL_URL=https://leetcode.com/graphql
```

If `EXTENSION_TOKEN_SECRET` is not provided, fall back to `JWT_SECRET` only for local development. Production should set a separate secret.

---

## 8. SQLite Migration for Extension Auth

Create:

```text
db/extension_auth_migration.sql
```

```sql
CREATE TABLE IF NOT EXISTS extension_auth_codes (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    code_hash TEXT NOT NULL UNIQUE,
    expires_at TEXT NOT NULL,
    used_at TEXT,
    created_at TEXT DEFAULT (datetime('now'))
);

CREATE INDEX IF NOT EXISTS idx_extension_auth_codes_user
ON extension_auth_codes(user_id);

CREATE INDEX IF NOT EXISTS idx_extension_auth_codes_expiry
ON extension_auth_codes(expires_at);

CREATE TABLE IF NOT EXISTS extension_sessions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    device_id TEXT NOT NULL,
    refresh_token_hash TEXT NOT NULL,
    scopes TEXT NOT NULL,
    user_agent TEXT,
    created_at TEXT DEFAULT (datetime('now')),
    last_used_at TEXT,
    revoked_at TEXT,
    UNIQUE(user_id, device_id)
);

CREATE INDEX IF NOT EXISTS idx_extension_sessions_user
ON extension_sessions(user_id);

CREATE INDEX IF NOT EXISTS idx_extension_sessions_device
ON extension_sessions(device_id);
```

Update `db/init_db.py` migration logic or add a separate migration runner so these tables are created on existing local databases.

---

## 9. Supabase Migration

Create:

```text
supabase/migrations/001_extension_sync_stage1.sql
```

### 9.1 Schemas

```sql
CREATE SCHEMA IF NOT EXISTS catalog;
CREATE SCHEMA IF NOT EXISTS user_data;
CREATE SCHEMA IF NOT EXISTS code_blobs;
CREATE SCHEMA IF NOT EXISTS system;
```

### 9.2 `catalog.coding_problems`

```sql
CREATE TABLE IF NOT EXISTS catalog.coding_problems (
  platform        TEXT NOT NULL,
  slug            TEXT NOT NULL,
  display_id      TEXT,
  title           TEXT NOT NULL,
  difficulty      TEXT,
  topic_tags      TEXT[],
  ac_rate         NUMERIC(5,2),

  statement_md    TEXT,
  constraints_md  TEXT,
  examples_json   JSONB,
  hints           TEXT[],
  similar_slugs   TEXT[],

  sync_status     TEXT DEFAULT 'pending',
  raw_meta        JSONB,

  created_at      TIMESTAMPTZ DEFAULT now(),
  updated_at      TIMESTAMPTZ DEFAULT now(),

  PRIMARY KEY (platform, slug)
);

CREATE INDEX IF NOT EXISTS idx_coding_problems_difficulty
ON catalog.coding_problems(platform, difficulty);

CREATE INDEX IF NOT EXISTS idx_coding_problems_topics
ON catalog.coding_problems USING GIN(topic_tags);

CREATE INDEX IF NOT EXISTS idx_coding_problems_sync_status
ON catalog.coding_problems(sync_status)
WHERE sync_status != 'complete';
```

### 9.3 `user_data.user_coding_profiles`

Use `app_user_id INTEGER` because the existing app user lives in SQLite.

```sql
CREATE TABLE IF NOT EXISTS user_data.user_coding_profiles (
  id                      BIGSERIAL PRIMARY KEY,
  app_user_id             INTEGER NOT NULL,
  platform                TEXT NOT NULL,

  external_user_id        TEXT,
  external_handle         TEXT NOT NULL,
  display_name            TEXT,
  profile_url             TEXT,
  is_premium              BOOLEAN DEFAULT FALSE,

  is_connected            BOOLEAN DEFAULT TRUE,
  sync_enabled            BOOLEAN DEFAULT TRUE,

  connected_at            TIMESTAMPTZ DEFAULT now(),
  last_full_sync_at       TIMESTAMPTZ,
  last_delta_sync_at      TIMESTAMPTZ,
  last_live_capture_at    TIMESTAMPTZ,
  last_sync_error         TEXT,

  raw_profile             JSONB,

  created_at              TIMESTAMPTZ DEFAULT now(),
  updated_at              TIMESTAMPTZ DEFAULT now(),

  UNIQUE (app_user_id, platform)
);

CREATE INDEX IF NOT EXISTS idx_profiles_app_user
ON user_data.user_coding_profiles(app_user_id);

CREATE INDEX IF NOT EXISTS idx_profiles_handle
ON user_data.user_coding_profiles(platform, external_handle);
```

### 9.4 `user_data.user_coding_problems`

```sql
CREATE TABLE IF NOT EXISTS user_data.user_coding_problems (
  app_user_id             INTEGER NOT NULL,
  platform                TEXT NOT NULL,
  slug                    TEXT NOT NULL,

  status                  TEXT NOT NULL,
  first_attempt_at        TIMESTAMPTZ,
  first_ac_at             TIMESTAMPTZ,
  latest_ac_at            TIMESTAMPTZ,
  latest_attempt_at       TIMESTAMPTZ,

  attempt_count           INTEGER DEFAULT 0,
  ac_count                INTEGER DEFAULT 0,
  wa_count                INTEGER DEFAULT 0,
  tle_count               INTEGER DEFAULT 0,
  mle_count               INTEGER DEFAULT 0,
  re_count                INTEGER DEFAULT 0,
  ce_count                INTEGER DEFAULT 0,
  other_count             INTEGER DEFAULT 0,

  best_runtime_ms         INTEGER,
  best_memory_kb          INTEGER,
  best_lang               TEXT,

  primary_submission_id   BIGINT,

  created_at              TIMESTAMPTZ DEFAULT now(),
  updated_at              TIMESTAMPTZ DEFAULT now(),

  PRIMARY KEY (app_user_id, platform, slug),
  FOREIGN KEY (platform, slug)
    REFERENCES catalog.coding_problems(platform, slug)
    ON DELETE RESTRICT
);

CREATE INDEX IF NOT EXISTS idx_user_problems_status
ON user_data.user_coding_problems(app_user_id, status);

CREATE INDEX IF NOT EXISTS idx_user_problems_latest_attempt
ON user_data.user_coding_problems(app_user_id, latest_attempt_at DESC);
```

### 9.5 `user_data.user_coding_submissions`

```sql
CREATE TABLE IF NOT EXISTS user_data.user_coding_submissions (
  platform                TEXT NOT NULL,
  submission_id           BIGINT NOT NULL,

  app_user_id             INTEGER NOT NULL,
  profile_id              BIGINT NOT NULL REFERENCES user_data.user_coding_profiles(id) ON DELETE CASCADE,
  slug                    TEXT NOT NULL,

  verdict                 TEXT NOT NULL,
  raw_status              TEXT,
  status_code             INTEGER,

  lang                    TEXT NOT NULL,
  lang_verbose            TEXT,

  runtime_ms              INTEGER,
  runtime_percentile      NUMERIC(5,2),
  memory_kb               INTEGER,
  memory_percentile       NUMERIC(5,2),

  total_correct           INTEGER,
  total_testcases         INTEGER,

  last_testcase           TEXT,
  expected_output         TEXT,
  actual_output           TEXT,
  runtime_error           TEXT,
  compile_error           TEXT,

  ts                      TIMESTAMPTZ NOT NULL,
  captured_at             TIMESTAMPTZ DEFAULT now(),
  capture_source          TEXT NOT NULL,
  code_capture_method     TEXT,

  has_code                BOOLEAN DEFAULT FALSE,

  raw_meta                JSONB,

  created_at              TIMESTAMPTZ DEFAULT now(),
  updated_at              TIMESTAMPTZ DEFAULT now(),

  PRIMARY KEY (platform, submission_id),
  FOREIGN KEY (platform, slug)
    REFERENCES catalog.coding_problems(platform, slug)
    ON DELETE RESTRICT
);

CREATE INDEX IF NOT EXISTS idx_submissions_app_user_ts
ON user_data.user_coding_submissions(app_user_id, ts DESC);

CREATE INDEX IF NOT EXISTS idx_submissions_app_user_slug
ON user_data.user_coding_submissions(app_user_id, platform, slug, ts DESC);

CREATE INDEX IF NOT EXISTS idx_submissions_verdict
ON user_data.user_coding_submissions(app_user_id, verdict, ts DESC);

CREATE INDEX IF NOT EXISTS idx_submissions_lang
ON user_data.user_coding_submissions(app_user_id, lang);

CREATE INDEX IF NOT EXISTS idx_submissions_has_code_false
ON user_data.user_coding_submissions(app_user_id, platform, ts DESC)
WHERE has_code = false;
```

### 9.6 `code_blobs.user_coding_submission_code`

```sql
CREATE TABLE IF NOT EXISTS code_blobs.user_coding_submission_code (
  platform                TEXT NOT NULL,
  submission_id           BIGINT NOT NULL,

  code                    TEXT NOT NULL,
  code_lang               TEXT NOT NULL,
  code_hash               TEXT,

  storage_tier            TEXT DEFAULT 'hot',
  s3_key                  TEXT,

  fetched_at              TIMESTAMPTZ DEFAULT now(),

  PRIMARY KEY (platform, submission_id),
  FOREIGN KEY (platform, submission_id)
    REFERENCES user_data.user_coding_submissions(platform, submission_id)
    ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS idx_code_storage_tier
ON code_blobs.user_coding_submission_code(storage_tier)
WHERE storage_tier = 'hot';
```

### 9.7 `system.sync_state`

```sql
CREATE TABLE IF NOT EXISTS system.sync_state (
  app_user_id             INTEGER NOT NULL,
  platform                TEXT NOT NULL,

  phase                   TEXT,
  cursor                  JSONB,

  last_progress_at        TIMESTAMPTZ,
  last_full_sync_at       TIMESTAMPTZ,

  retry_count             INTEGER DEFAULT 0,
  last_error              TEXT,

  created_at              TIMESTAMPTZ DEFAULT now(),
  updated_at              TIMESTAMPTZ DEFAULT now(),

  PRIMARY KEY (app_user_id, platform)
);
```

### 9.8 PGMQ Setup

Enable PGMQ in Supabase and create one queue:

```sql
-- Requires pgmq extension enabled in Supabase.
CREATE EXTENSION IF NOT EXISTS pgmq;

SELECT pgmq.create('extension_events');
```

The only Stage 1 event type is:

```json
{
  "type": "submission.received",
  "app_user_id": 123,
  "platform": "leetcode",
  "slug": "two-sum",
  "submission_id": 123456789
}
```

---

## 10. Backend Implementation Order

Implement the backend in this exact order.

---

### Task B1 — Add Extension Config

Modify:

```text
api/config.py
```

Add config fields:

```python
EXTENSION_DATABASE_URL = os.getenv("EXTENSION_DATABASE_URL", "")
SUPABASE_PROJECT_URL = os.getenv("SUPABASE_PROJECT_URL", "")
SUPABASE_SERVICE_ROLE_KEY = os.getenv("SUPABASE_SERVICE_ROLE_KEY", "")

EXTENSION_TOKEN_SECRET = os.getenv("EXTENSION_TOKEN_SECRET", JWT_SECRET)
EXTENSION_ACCESS_TOKEN_MINUTES = int(os.getenv("EXTENSION_ACCESS_TOKEN_MINUTES", "30"))
EXTENSION_REFRESH_TOKEN_DAYS = int(os.getenv("EXTENSION_REFRESH_TOKEN_DAYS", "60"))
EXTENSION_AUTH_CODE_MINUTES = int(os.getenv("EXTENSION_AUTH_CODE_MINUTES", "5"))

EXTENSION_PGMQ_QUEUE = os.getenv("EXTENSION_PGMQ_QUEUE", "extension_events")
CATALOG_WORKER_ENABLED = os.getenv("CATALOG_WORKER_ENABLED", "true").lower() == "true"
LEETCODE_GRAPHQL_URL = os.getenv("LEETCODE_GRAPHQL_URL", "https://leetcode.com/graphql")
```

Acceptance criteria:

- Backend boots when extension env vars are present.
- Local dev can disable catalog worker with `CATALOG_WORKER_ENABLED=false`.

---

### Task B2 — Add Supabase/Postgres Connection Helper

Create:

```text
api/services/extension_sync/supabase_db.py
```

Implementation shape:

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

- Can open a Supabase Postgres connection from backend.
- Transactions commit on success and rollback on failure.

---

### Task B3 — Add Pydantic Models

Create:

```text
api/services/extension_sync/models.py
```

Minimum models:

```python
from typing import Any, Literal
from pydantic import BaseModel, Field

Platform = Literal["leetcode", "codeforces", "gfg", "crackedin_practice"]


class ExtensionMeResponse(BaseModel):
    app_user_id: int
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

- Route files import these models.
- Invalid payloads fail with FastAPI validation errors.

---

### Task B4 — Add Normalization Helpers

Create:

```text
api/services/extension_sync/normalization.py
```

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

- All verdicts are stored as normalized short codes.
- Raw status is preserved separately.
- Runtime/memory parsing is best-effort and safe.

---

### Task B5 — Add Payload Validators

Create:

```text
api/services/extension_sync/validators.py
```

Rules:

1. Reject unsupported platforms.
2. Reject empty batches.
3. Reject suspicious raw payload fields.
4. Reject missing slug/submission IDs.
5. Reject code payloads larger than a configured safety limit.
6. Never accept `app_user_id` from request body.

Example:

```python
FORBIDDEN_RAW_KEYS = {
    "cookie",
    "cookies",
    "authorization",
    "headers",
    "csrf",
    "csrftoken",
    "LEETCODE_SESSION",
    "localStorage",
    "sessionStorage",
}


def assert_no_sensitive_keys(obj: object) -> None:
    if isinstance(obj, dict):
        for key, value in obj.items():
            if key.lower() in {k.lower() for k in FORBIDDEN_RAW_KEYS}:
                raise ValueError(f"Sensitive key not allowed in payload: {key}")
            assert_no_sensitive_keys(value)
    elif isinstance(obj, list):
        for item in obj:
            assert_no_sensitive_keys(item)
```

Acceptance criteria:

- Payloads with cookies/headers are rejected.
- User ID is derived only from JWT dependency.

---

### Task B6 — Add Extension Token Service

Create:

```text
api/services/extension_sync/token_service.py
```

Responsibilities:

1. Create one-time auth code.
2. Hash and store auth code in SQLite.
3. Exchange auth code for access/refresh tokens.
4. Hash refresh token before storage.
5. Refresh access token using refresh token.
6. Revoke extension session.

Token claims for extension access token:

```json
{
  "sub": "123",
  "typ": "extension_access",
  "sid": "device-or-session-id",
  "scopes": [
    "extension:me",
    "coding_profile:write",
    "coding_submission:write",
    "sync_state:write"
  ],
  "exp": 1710000000
}
```

Implementation note:

The first implementation may reuse the existing `get_current_user_id` dependency for the ingestion routes if extension-specific token exchange is not ready. However, the files/tables for extension sessions should still be added now.

Acceptance criteria:

- Extension tokens are revocable.
- Refresh tokens are stored hashed.
- Access tokens are short-lived.

---

### Task B7 — Add Profile Service

Create:

```text
api/services/extension_sync/profiles.py
```

Responsibilities:

1. Upsert connected coding profile.
2. Enforce `(app_user_id, platform)` uniqueness.
3. Mark profile connected/disconnected.
4. Update `last_full_sync_at`, `last_delta_sync_at`, `last_live_capture_at` later.

Example SQL:

```sql
INSERT INTO user_data.user_coding_profiles (
  app_user_id,
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
)
VALUES (%s, %s, %s, %s, %s, %s, %s, true, true, %s, now())
ON CONFLICT (app_user_id, platform)
DO UPDATE SET
  external_user_id = EXCLUDED.external_user_id,
  external_handle = EXCLUDED.external_handle,
  display_name = EXCLUDED.display_name,
  profile_url = EXCLUDED.profile_url,
  is_premium = EXCLUDED.is_premium,
  is_connected = true,
  sync_enabled = true,
  raw_profile = EXCLUDED.raw_profile,
  updated_at = now()
RETURNING id;
```

Acceptance criteria:

- Connecting the same platform twice updates the same profile row.
- A user cannot modify another user's profile because `app_user_id` comes from auth.

---

### Task B8 — Add Catalog Event Publisher

Create:

```text
api/services/extension_sync/catalog_events.py
```

Responsibilities:

1. Publish `submission.received` events to PGMQ.
2. Only publish for unknown or incomplete problem statements.
3. Deduplicate where practical before publishing.

Example function:

```python
import json
from api.config import EXTENSION_PGMQ_QUEUE


def publish_submission_received(conn, app_user_id: int, platform: str, slug: str, submission_id: int) -> None:
    payload = {
        "type": "submission.received",
        "app_user_id": app_user_id,
        "platform": platform,
        "slug": slug,
        "submission_id": submission_id,
    }
    conn.execute(
        "SELECT pgmq.send(%s, %s::jsonb)",
        (EXTENSION_PGMQ_QUEUE, json.dumps(payload)),
    )
```

Acceptance criteria:

- Events are inserted transactionally with ingestion.
- Catalog worker can consume the events.

---

### Task B9 — Add Submission Metadata Ingest Service

Create:

```text
api/services/extension_sync/ingest_submissions.py
```

Responsibilities:

1. Validate metadata batch.
2. Upsert catalog stubs for every slug.
3. Upsert user profile if needed.
4. Upsert `user_coding_submissions` with `has_code=false`.
5. Recompute `user_coding_problems` aggregates for affected slugs.
6. Publish `submission.received` for pending/missing catalog statements.
7. Update sync state.

Pseudo-code:

```python
def ingest_metadata_batch(app_user_id: int, req: SubmissionMetadataBatchRequest) -> dict:
    validate_batch(req)

    with get_pg() as conn:
        profile_id = ensure_profile(conn, app_user_id, req.platform, req.profile_handle)
        affected_slugs = set()
        unknown_event_count = 0

        for item in req.batch:
            upsert_catalog_stub(conn, req.platform, item.slug, item.title)
            affected_slugs.add(item.slug)

            insert_or_update_submission_meta(
                conn=conn,
                app_user_id=app_user_id,
                profile_id=profile_id,
                platform=req.platform,
                item=item,
            )

            if catalog_needs_statement(conn, req.platform, item.slug):
                publish_submission_received(
                    conn,
                    app_user_id,
                    req.platform,
                    item.slug,
                    item.submission_id,
                )
                unknown_event_count += 1

        recompute_user_problem_aggregates(conn, app_user_id, req.platform, affected_slugs)
        update_sync_state_for_metadata(conn, app_user_id, req.platform, req.cursor_offset, req.last_batch)

    return {
        "ok": True,
        "processed": len(req.batch),
        "affected_slugs": len(affected_slugs),
        "catalog_events_published": unknown_event_count,
    }
```

Important SQL behavior:

```sql
ON CONFLICT (platform, submission_id) DO UPDATE
```

But never allow a different `app_user_id` to claim the same `(platform, submission_id)`. If a conflict exists with another user, log and reject or skip safely. In practice, LeetCode submission IDs should belong to one account.

Acceptance criteria:

- Replaying the same metadata batch is safe.
- Submissions are deduplicated by `(platform, submission_id)`.
- Solve graph updates after each batch.

---

### Task B10 — Add Aggregate Recompute Service

Create:

```text
api/services/extension_sync/aggregates.py
```

For each affected `(app_user_id, platform, slug)`, recompute from `user_coding_submissions`.

Rules:

```text
status = 'ac' if ac_count > 0
status = 'tried' if attempt_count > 0 and ac_count = 0
status = 'untouched' is not needed for rows that exist only after attempt

first_attempt_at = MIN(ts)
latest_attempt_at = MAX(ts)
first_ac_at = MIN(ts WHERE verdict='AC')
latest_ac_at = MAX(ts WHERE verdict='AC')
primary_submission_id = latest AC submission if exists, otherwise latest attempt
best_runtime_ms = best AC runtime if available
best_memory_kb = best AC memory if available
best_lang = language of primary/best AC submission
```

Example aggregate SQL shape:

```sql
SELECT
  COUNT(*) AS attempt_count,
  COUNT(*) FILTER (WHERE verdict = 'AC') AS ac_count,
  COUNT(*) FILTER (WHERE verdict = 'WA') AS wa_count,
  COUNT(*) FILTER (WHERE verdict = 'TLE') AS tle_count,
  COUNT(*) FILTER (WHERE verdict = 'MLE') AS mle_count,
  COUNT(*) FILTER (WHERE verdict = 'RE') AS re_count,
  COUNT(*) FILTER (WHERE verdict = 'CE') AS ce_count,
  COUNT(*) FILTER (WHERE verdict NOT IN ('AC','WA','TLE','MLE','RE','CE')) AS other_count,
  MIN(ts) AS first_attempt_at,
  MAX(ts) AS latest_attempt_at,
  MIN(ts) FILTER (WHERE verdict = 'AC') AS first_ac_at,
  MAX(ts) FILTER (WHERE verdict = 'AC') AS latest_ac_at
FROM user_data.user_coding_submissions
WHERE app_user_id = %s AND platform = %s AND slug = %s;
```

Acceptance criteria:

- Dashboard can read from `user_coding_problems` without grouping all submissions.
- Aggregate rows are idempotently updated after metadata/code/live/delta inserts.

---

### Task B11 — Add Code Ingest Service

Create:

```text
api/services/extension_sync/ingest_code.py
```

Responsibilities:

1. Validate code batch.
2. Confirm each submission metadata row already exists.
3. Upsert code blob into `code_blobs.user_coding_submission_code`.
4. Update `user_coding_submissions.has_code=true`.
5. Update richer runtime/memory/testcase/error fields if present.
6. Publish catalog event if statement still missing.
7. Update sync state.

Pseudo-code:

```python
def ingest_code_batch(app_user_id: int, req: CodeBatchRequest) -> dict:
    validate_code_batch(req)

    with get_pg() as conn:
        processed = 0
        missing_meta = []

        for item in req.batch:
            existing = find_submission_for_user(conn, app_user_id, req.platform, item.submission_id)
            if not existing:
                missing_meta.append(item.submission_id)
                continue

            upsert_code_blob(conn, req.platform, item)
            update_submission_detail_fields(conn, app_user_id, req.platform, item)
            processed += 1

        return {"ok": True, "processed": processed, "missing_meta": missing_meta}
```

Code blob SQL:

```sql
INSERT INTO code_blobs.user_coding_submission_code (
  platform,
  submission_id,
  code,
  code_lang,
  code_hash,
  storage_tier,
  fetched_at
)
VALUES (%s, %s, %s, %s, %s, 'hot', now())
ON CONFLICT (platform, submission_id)
DO UPDATE SET
  code = EXCLUDED.code,
  code_lang = EXCLUDED.code_lang,
  code_hash = EXCLUDED.code_hash,
  storage_tier = 'hot',
  fetched_at = now();
```

Acceptance criteria:

- Code ingestion is idempotent.
- Code blobs do not live in the hot submission table.
- `has_code` accurately reflects whether code exists.

---

### Task B12 — Add Sync State Service

Create:

```text
api/services/extension_sync/sync_state.py
```

Backend sync state fields:

```json
{
  "phase": "A_bulk",
  "cursor": {
    "offset": 400,
    "metadataFetched": 600,
    "foregroundCodeQueue": [],
    "backgroundCodeQueueSize": 1200,
    "codeDone": 100,
    "codeFailed": 3
  },
  "last_progress_at": "2026-05-27T12:00:00Z",
  "last_error": null
}
```

Allowed phases:

```text
auth_checking
A_bulk
B_fg
B_bg
paused
auth_required
failed
done
live_ready
```

Stage 1 should finish as:

```text
B_bg complete → done or live_ready
```

Use `live_ready` only if the architecture wants to show that Stage 2 can be enabled later. Do not implement live capture now.

Acceptance criteria:

- Backend state can reconstruct high-level progress.
- Extension local state remains the detailed cursor source.
- User support/debugging can inspect server state.

---

### Task B13 — Add FastAPI Router

Create:

```text
api/routes/extension_sync.py
```

Router prefix:

```python
router = APIRouter(prefix="/api/extension", tags=["extension"])
```

Required endpoints:

```text
GET    /api/extension/me
POST   /api/extension/auth/start
POST   /api/extension/auth/exchange
POST   /api/extension/auth/refresh
POST   /api/extension/auth/revoke

POST   /api/extension/platforms/{platform}/connect
GET    /api/extension/platforms/{platform}/sync-state
PATCH  /api/extension/platforms/{platform}/sync-state
POST   /api/extension/platforms/{platform}/submissions
POST   /api/extension/platforms/{platform}/code
POST   /api/extension/platforms/{platform}/pause
POST   /api/extension/platforms/{platform}/resume
DELETE /api/extension/platforms/{platform}
DELETE /api/extension/platforms/{platform}/data
```

Stage 1 active endpoints:

```text
GET    /api/extension/me
POST   /api/extension/platforms/{platform}/connect
GET    /api/extension/platforms/{platform}/sync-state
PATCH  /api/extension/platforms/{platform}/sync-state
POST   /api/extension/platforms/{platform}/submissions
POST   /api/extension/platforms/{platform}/code
POST   /api/extension/platforms/{platform}/pause
POST   /api/extension/platforms/{platform}/resume
```

The auth exchange endpoints should be implemented if possible. If not, use existing JWT for local MVP and keep the endpoint stubs with clear TODOs.

Example endpoint:

```python
@router.post("/platforms/{platform}/submissions")
def ingest_submissions_endpoint(
    platform: str,
    req: SubmissionMetadataBatchRequest,
    app_user_id: int = Depends(get_current_user_id),
):
    if platform != req.platform:
        raise HTTPException(status_code=400, detail="Platform mismatch")
    return ingest_metadata_batch(app_user_id, req)
```

Acceptance criteria:

- All Stage 1 endpoints are callable with JWT.
- No endpoint trusts `app_user_id` from client payload.

---

### Task B14 — Mount Router in `api/main.py`

Modify:

```text
api/main.py
```

Add import:

```python
from api.routes import extension_sync
```

Add mount:

```python
app.include_router(extension_sync.router)
```

Acceptance criteria:

- `/api/extension/me` appears in OpenAPI docs.
- Existing routes still work.

---

### Task B15 — Add Catalog Worker

Create:

```text
api/services/extension_sync/catalog_worker.py
```

Purpose:

```text
Consume PGMQ submission.received events and fill missing catalog.coding_problems.statement_md.
```

Behavior:

```text
1. Poll pgmq.read('extension_events', ...).
2. For each event:
   - validate type == submission.received
   - read platform + slug
   - check catalog.coding_problems
   - if statement_md exists and sync_status='complete', archive/delete message
   - else fetch problem detail from LeetCode GraphQL
   - convert HTML to markdown
   - update catalog.coding_problems
   - archive/delete message
3. On failure:
   - leave/retry message
   - after repeated failure, mark sync_status='failed'
```

LeetCode problem query shape:

```graphql
query questionData($titleSlug: String!) {
  question(titleSlug: $titleSlug) {
    questionFrontendId
    title
    titleSlug
    difficulty
    topicTags { name slug }
    content
    hints
    similarQuestions
    stats
  }
}
```

Worker pseudo-code:

```python
def run_catalog_worker_once() -> int:
    with get_pg() as conn:
        messages = conn.execute(
            "SELECT * FROM pgmq.read(%s, 30, 10)",
            (EXTENSION_PGMQ_QUEUE,),
        ).fetchall()

        processed = 0
        for msg in messages:
            payload = msg["message"]
            try:
                handle_submission_received(conn, payload)
                conn.execute(
                    "SELECT pgmq.archive(%s, %s)",
                    (EXTENSION_PGMQ_QUEUE, msg["msg_id"]),
                )
                processed += 1
            except Exception as exc:
                # log and let pgmq retry by visibility timeout
                pass

        return processed
```

Mount worker at FastAPI startup only if enabled:

```python
if CATALOG_WORKER_ENABLED:
    start_catalog_worker_thread()
```

Acceptance criteria:

- Unknown problem slugs eventually get statement/difficulty/topics.
- Ingest endpoint does not block on catalog fetch.
- PGMQ is used for catalog events.

---

## 11. Extension Implementation Order

Implement the extension in this exact order.

---

### Task E1 — Create Extension Project

Create:

```text
extension/package.json
extension/tsconfig.json
extension/vite.config.ts
extension/manifest.json
```

Use TypeScript.

Minimal `package.json`:

```json
{
  "name": "crackedin-extension",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "vite build --watch",
    "build": "vite build",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {},
  "devDependencies": {
    "@types/chrome": "latest",
    "typescript": "latest",
    "vite": "latest"
  }
}
```

Acceptance criteria:

- Extension builds into a loadable Chrome extension directory.
- TypeScript typecheck passes.

---

### Task E2 — Add Manifest V3

Create:

```text
extension/manifest.json
```

```json
{
  "manifest_version": 3,
  "name": "CrackedIn LeetCode Sync",
  "version": "0.1.0",
  "description": "Sync your LeetCode submissions into CrackedIn for personalized interview preparation.",
  "permissions": [
    "storage",
    "alarms"
  ],
  "host_permissions": [
    "https://leetcode.com/*",
    "https://*.crackedin.app/*",
    "http://localhost:8000/*"
  ],
  "background": {
    "service_worker": "service-worker/index.js",
    "type": "module"
  },
  "action": {
    "default_popup": "popup/popup.html",
    "default_title": "CrackedIn"
  },
  "content_scripts": [
    {
      "matches": ["https://leetcode.com/problems/*"],
      "js": ["content-scripts/leetcodeLivePlaceholder.js"],
      "run_at": "document_start"
    }
  ]
}
```

Important:

```text
Do not request cookies permission in Stage 1.
Do not read LeetCode cookies directly.
Use fetch(..., credentials: 'include') to LeetCode.
```

Acceptance criteria:

- Chrome shows LeetCode host permission.
- Extension can call `https://leetcode.com/graphql` from service worker.

---

### Task E3 — Shared Types

Create:

```text
extension/src/shared/types.ts
```

```ts
export type Platform = "leetcode" | "codeforces" | "gfg" | "crackedin_practice";

export type SyncPhase =
  | "idle"
  | "auth_checking"
  | "A_bulk"
  | "B_fg"
  | "B_bg"
  | "paused"
  | "auth_required"
  | "failed"
  | "done"
  | "live_ready";

export type FailedCodeItem = {
  platform: Platform;
  submission_id: number;
  slug?: string;
  reason: string;
  retry_count: number;
  next_retry_at: number;
  last_error?: string;
};

export type LocalSyncState = {
  platform: Platform;
  phase: SyncPhase;
  leetcodeUsername?: string;

  metadataOffset?: number;
  metadataFetched: number;
  totalSubmissions?: number;

  foregroundCodeQueue: number[];
  backgroundCodeQueue: number[];
  failedCodeQueue: FailedCodeItem[];

  codeFetched: number;
  codeFailed: number;

  pauseRequested: boolean;
  lastProgressAt?: number;
  lastFullSyncAt?: number;
  error?: string;
};
```

Acceptance criteria:

- Every job uses the same sync state shape.
- State can be serialized to `chrome.storage.local`.

---

### Task E4 — Storage Helper

Create:

```text
extension/src/shared/storage.ts
```

```ts
import type { LocalSyncState } from "./types";

const SYNC_STATE_KEY = "crackedin_sync_state";
const AUTH_TOKEN_KEY = "crackedin_auth";

export async function getSyncState(): Promise<LocalSyncState | null> {
  const result = await chrome.storage.local.get(SYNC_STATE_KEY);
  return result[SYNC_STATE_KEY] ?? null;
}

export async function setSyncState(state: LocalSyncState): Promise<void> {
  await chrome.storage.local.set({
    [SYNC_STATE_KEY]: {
      ...state,
      lastProgressAt: Date.now(),
    },
  });
}

export async function patchSyncState(patch: Partial<LocalSyncState>): Promise<LocalSyncState> {
  const current = await getSyncState();
  const next = {
    ...(current ?? {
      platform: "leetcode",
      phase: "idle",
      metadataFetched: 0,
      foregroundCodeQueue: [],
      backgroundCodeQueue: [],
      failedCodeQueue: [],
      codeFetched: 0,
      codeFailed: 0,
      pauseRequested: false,
    }),
    ...patch,
    lastProgressAt: Date.now(),
  } as LocalSyncState;
  await setSyncState(next);
  return next;
}

export async function getAuthToken(): Promise<string | null> {
  const result = await chrome.storage.local.get(AUTH_TOKEN_KEY);
  return result[AUTH_TOKEN_KEY]?.accessToken ?? null;
}
```

Acceptance criteria:

- Service worker can resume after being killed.
- Popup can render progress by reading storage.

---

### Task E5 — Backend API Client

Create:

```text
extension/src/api/crackedinApi.ts
```

```ts
import { getAuthToken } from "../shared/storage";

const API_BASE = "http://localhost:8000"; // replace through build env later

async function request<T>(path: string, init: RequestInit = {}): Promise<T> {
  const token = await getAuthToken();
  if (!token) throw new Error("Not connected to CrackedIn");

  const res = await fetch(`${API_BASE}${path}`, {
    ...init,
    headers: {
      "Content-Type": "application/json",
      "Authorization": `Bearer ${token}`,
      ...(init.headers ?? {}),
    },
  });

  if (!res.ok) {
    const text = await res.text();
    throw new Error(`CrackedIn API error ${res.status}: ${text}`);
  }

  return res.json() as Promise<T>;
}

export async function connectPlatform(body: unknown) {
  return request("/api/extension/platforms/leetcode/connect", {
    method: "POST",
    body: JSON.stringify(body),
  });
}

export async function uploadSubmissionMetadata(body: unknown) {
  return request("/api/extension/platforms/leetcode/submissions", {
    method: "POST",
    body: JSON.stringify(body),
  });
}

export async function uploadCodeBatch(body: unknown) {
  return request("/api/extension/platforms/leetcode/code", {
    method: "POST",
    body: JSON.stringify(body),
  });
}

export async function patchBackendSyncState(body: unknown) {
  return request("/api/extension/platforms/leetcode/sync-state", {
    method: "PATCH",
    body: JSON.stringify(body),
  });
}
```

Acceptance criteria:

- All backend calls use Bearer auth.
- Extension never talks to Supabase directly.

---

### Task E6 — LeetCode GraphQL Query Builder

Create:

```text
extension/src/service-worker/graphql/leetcodeQueries.ts
```

Auth query:

```ts
export const USER_STATUS_QUERY = `
query userStatus {
  userStatus {
    isSignedIn
    username
    userId
    isPremium
  }
}
`;
```

Metadata aliased query builder:

```ts
export function buildSubmissionListAliasedQuery(offset: number, pages = 10): string {
  const fields = Array.from({ length: pages }, (_, i) => {
    const pageOffset = offset + i * 20;
    return `
      p${i}: submissionList(offset: ${pageOffset}, limit: 20) {
        hasNext
        submissions {
          id
          title
          titleSlug
          statusDisplay
          lang
          langName
          runtime
          memory
          timestamp
        }
      }
    `;
  }).join("\n");

  return `query bulkSubmissionPages { ${fields} }`;
}
```

Code aliased query builder:

```ts
export function buildSubmissionDetailsAliasedQuery(ids: number[]): string {
  const fields = ids.map((id, i) => `
    s${i}: submissionDetails(submissionId: ${id}) {
      id
      code
      runtime
      runtimeDisplay
      runtimePercentile
      memory
      memoryDisplay
      memoryPercentile
      statusDisplay
      lang { name verboseName }
      question { title titleSlug }
      totalCorrect
      totalTestcases
      lastTestcase
      expectedOutput
      codeOutput
      runtimeError
      compileError
    }
  `).join("\n");

  return `query codeBatch { ${fields} }`;
}
```

Acceptance criteria:

- Metadata sync fetches 10 pages per request.
- Code sync fetches 10 submissions per request.
- Builders keep alias labels stable and parseable.

---

### Task E7 — LeetCode Adapter

Create:

```text
extension/src/platforms/leetcode.ts
```

Responsibilities:

1. Send GraphQL requests to LeetCode.
2. Use `credentials: 'include'`.
3. Check `userStatus`.
4. Fetch metadata batches.
5. Fetch code detail batches.
6. Return sanitized objects only.

Example request helper:

```ts
async function leetcodeGraphql<T>(query: string, variables?: Record<string, unknown>): Promise<T> {
  const res = await fetch("https://leetcode.com/graphql", {
    method: "POST",
    credentials: "include",
    headers: {
      "Content-Type": "application/json",
      "Referer": "https://leetcode.com",
    },
    body: JSON.stringify({ query, variables: variables ?? {} }),
  });

  if (res.status === 401 || res.status === 403) {
    throw new Error("LEETCODE_AUTH_REQUIRED");
  }

  if (!res.ok) {
    throw new Error(`LeetCode GraphQL failed: ${res.status}`);
  }

  const json = await res.json();
  if (json.errors?.length) {
    throw new Error(json.errors[0]?.message ?? "LeetCode GraphQL error");
  }

  return json.data as T;
}
```

Acceptance criteria:

- Adapter never reads cookies directly.
- Adapter never returns raw headers or browser request data.
- Adapter output matches backend payload contract.

---

### Task E8 — Service Worker Entrypoint

Create:

```text
extension/src/service-worker/index.ts
```

Responsibilities:

1. Listen for popup commands.
2. Start/resume jobs.
3. Register alarm heartbeat.
4. Resume work on wake.

Message commands:

```ts
export type WorkerMessage =
  | { type: "START_SYNC" }
  | { type: "PAUSE_SYNC" }
  | { type: "RESUME_SYNC" }
  | { type: "GET_STATE" };
```

Skeleton:

```ts
chrome.runtime.onMessage.addListener((message, _sender, sendResponse) => {
  handleMessage(message)
    .then(sendResponse)
    .catch((err) => sendResponse({ ok: false, error: String(err?.message ?? err) }));
  return true;
});

chrome.alarms.create("crackedin-sync-heartbeat", { periodInMinutes: 1 });

chrome.alarms.onAlarm.addListener(async (alarm) => {
  if (alarm.name === "crackedin-sync-heartbeat") {
    await resumeIfNeeded();
  }
});
```

Acceptance criteria:

- Popup can start/pause/resume sync.
- Service worker resumes after idle shutdown.

---

### Task E9 — Metadata Sync Job

Create:

```text
extension/src/service-worker/jobs/metadataSyncJob.ts
```

Algorithm:

```text
1. Set phase = A_bulk.
2. Read metadataOffset from local state; default 0.
3. Build 10-page aliased submissionList query.
4. POST to LeetCode GraphQL.
5. Flatten p0..p9 pages.
6. Upload sanitized rows to backend.
7. Save metadataOffset += 200 after successful backend upload.
8. Append submission IDs to allSubmissionIds ordered by ts DESC.
9. If any page has hasNext=false or fewer than 20 submissions:
   - sort all discovered IDs by timestamp descending
   - first 100 → foregroundCodeQueue
   - rest → backgroundCodeQueue
   - set phase = B_fg
   - persist state
   - return
10. Continue loop until paused/done.
```

Important:

```text
Persist state only after backend upload succeeds.
If worker dies mid-batch, replay the same offset.
Backend upsert makes replay safe.
```

Acceptance criteria:

- Full metadata sync works even if service worker stops and restarts.
- Backend receives clean metadata batches.
- User sees increasing `metadataFetched` count.

---

### Task E10 — Foreground Code Job

Create:

```text
extension/src/service-worker/jobs/foregroundCodeJob.ts
```

Algorithm:

```text
1. Set phase = B_fg.
2. Read foregroundCodeQueue.
3. Take next 10 submission IDs.
4. Build aliased submissionDetails query.
5. Fetch from LeetCode.
6. Upload code batch to backend.
7. Remove successful IDs from foregroundCodeQueue.
8. Failed IDs go to failedCodeQueue.
9. Increment codeFetched/codeFailed.
10. Persist after every batch.
11. When queue empty, set phase = B_bg.
```

Acceptance criteria:

- Latest 100 code details are fetched before foreground completion.
- Core sync can be marked usable after foreground code queue empties.

---

### Task E11 — Background Code Job

Create:

```text
extension/src/service-worker/jobs/backgroundCodeJob.ts
```

Algorithm:

```text
1. Run only when phase = B_bg.
2. Take next 10 IDs from backgroundCodeQueue.
3. Fetch code details.
4. Upload to backend.
5. Persist state.
6. Sleep/cadence between batches.
7. Continue until queue empty or pause requested.
8. When background queue empty and failed queue empty, set phase = done/live_ready.
```

Cadence:

```text
Foreground code: immediate batches.
Background code: 1 batch every ~2 seconds.
After rate limit: backoff 10s → 30s → 60s.
```

Acceptance criteria:

- User can close popup and background job keeps/resumes progress.
- Background backfill does not block app usage.

---

### Task E12 — Failed Code Retry Job

Create:

```text
extension/src/service-worker/jobs/failedCodeRetryJob.ts
```

Rules:

```text
1. Retry only items whose next_retry_at <= now.
2. Batch up to 10 retry IDs.
3. If retry succeeds, remove from failedCodeQueue.
4. If retry fails, increment retry_count and schedule next_retry_at.
5. Do not block background queue forever.
```

Backoff:

```ts
function nextRetryDelayMs(retryCount: number): number {
  if (retryCount <= 1) return 30_000;
  if (retryCount === 2) return 120_000;
  if (retryCount === 3) return 600_000;
  return 1800_000;
}
```

Acceptance criteria:

- Temporary LeetCode/backend failures do not permanently lose code.
- Retry queue survives service worker restarts.

---

### Task E13 — Popup UI

Create:

```text
extension/src/popup/popup.html
extension/src/popup/popup.ts
extension/src/popup/popup.css
```

Popup states:

| Phase | UI message |
|---|---|
| `idle` | Connect CrackedIn and start sync |
| `auth_checking` | Checking CrackedIn and LeetCode connections |
| `A_bulk` | Finding your LeetCode submissions |
| `B_fg` | Fetching recent submitted code |
| `B_bg` | Core sync complete. Background code backfill running |
| `paused` | Sync paused |
| `auth_required` | Log in to LeetCode to continue |
| `failed` | Sync paused due to an error |
| `done` | Historical sync complete |
| `live_ready` | Historical sync complete. Live capture can be enabled later |

Popup must show:

```text
- detected CrackedIn connection status
- detected LeetCode username
- current phase
- metadata count
- code count
- failed retry count
- Start Sync button
- Pause button
- Resume button
```

Progress calculation:

```ts
function calculateProgress(state: LocalSyncState): number | null {
  if (state.phase === "A_bulk" && !state.totalSubmissions) return null;

  const metadataWeight = 35;
  const foregroundCodeWeight = 25;
  const backgroundCodeWeight = 40;

  const metadataProgress = state.totalSubmissions
    ? Math.min(state.metadataFetched / state.totalSubmissions, 1) * metadataWeight
    : 0;

  const totalCode = state.codeFetched + state.backgroundCodeQueue.length + state.foregroundCodeQueue.length;
  const codeProgress = totalCode
    ? Math.min(state.codeFetched / totalCode, 1) * (foregroundCodeWeight + backgroundCodeWeight)
    : 0;

  return Math.round(metadataProgress + codeProgress);
}
```

Acceptance criteria:

- Popup is state-driven.
- Popup does not run sync loops.
- Closing popup does not stop sync.

---

## 12. Stage 1 Sync Algorithms

### 12.1 Phase A Metadata Loop

```text
while phase == A_bulk:
  state = read local sync state
  if pauseRequested: set paused and return

  query = buildSubmissionListAliasedQuery(state.metadataOffset, 10)
  data = leetcodeGraphql(query)
  rows = flatten pages

  upload /api/extension/platforms/leetcode/submissions

  if upload succeeds:
    update local state:
      metadataFetched += rows.length
      metadataOffset += 200
      discoveredSubmissionIds append rows

  if any page has hasNext=false:
    create foregroundCodeQueue latest 100
    create backgroundCodeQueue remaining
    phase = B_fg
    persist
    return
```

### 12.2 Phase B Foreground Code Loop

```text
while phase == B_fg:
  state = read local sync state
  if foregroundCodeQueue empty:
    phase = B_bg
    persist
    return

  ids = foregroundCodeQueue.take(10)
  data = fetch submissionDetails aliases
  upload /api/extension/platforms/leetcode/code

  success ids removed
  failed ids moved to failedCodeQueue
  persist
```

### 12.3 Phase C Background Code Loop

```text
while phase == B_bg:
  state = read local sync state
  if pauseRequested: set paused and return

  retry due failed-code items first if any

  if backgroundCodeQueue empty:
    if failedCodeQueue empty:
      phase = done or live_ready
    persist
    return

  ids = backgroundCodeQueue.take(10)
  fetch submissionDetails aliases
  upload code batch
  update queues
  persist
  wait cadence delay
```

---

## 13. Backend Data Contracts

### 13.1 Connect Profile Request

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

### 13.2 Metadata Batch Request

```json
{
  "platform": "leetcode",
  "profile_handle": "leetcode_user",
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
      "timestamp": 1710000000,
      "raw_meta": {
        "source": "submissionList"
      }
    }
  ],
  "cursor_offset": 200,
  "last_batch": false
}
```

### 13.3 Metadata Batch Response

```json
{
  "ok": true,
  "processed": 200,
  "affected_slugs": 126,
  "catalog_events_published": 3,
  "next_phase": "A_bulk"
}
```

### 13.4 Code Batch Request

```json
{
  "platform": "leetcode",
  "batch": [
    {
      "submission_id": 123456789,
      "slug": "two-sum",
      "code": "class Solution { ... }",
      "code_lang": "cpp",
      "lang_verbose": "C++",
      "raw_status": "Accepted",
      "runtime_ms": 4,
      "runtime_percentile": 91.2,
      "memory_kb": 12800,
      "memory_percentile": 72.5,
      "total_correct": 57,
      "total_testcases": 57,
      "raw_detail": {
        "source": "submissionDetails"
      }
    }
  ]
}
```

### 13.5 Code Batch Response

```json
{
  "ok": true,
  "processed": 10,
  "missing_meta": [],
  "catalog_events_published": 0
}
```

---

## 14. Security Rules

### 14.1 Extension Must Never Upload

```text
- LeetCode cookies
- LEETCODE_SESSION
- csrftoken
- CrackedIn password
- LeetCode password
- request headers
- raw browser request objects
- unrelated network events
- unrelated page HTML
- localStorage dumps
- sessionStorage dumps
```

### 14.2 Backend Must Enforce

```text
- app_user_id comes only from JWT auth dependency
- payload user_id is ignored/rejected
- platform is validated
- batch size is capped
- code size is capped
- forbidden raw keys are rejected
- Supabase writes use service-side credentials only
- extension tokens are revocable
- duplicate submissions are idempotent
```

### 14.3 LeetCode Trust Boundary

```text
The extension may ask LeetCode for data using the user's browser session.
The extension must not read or transmit the LeetCode cookie.
The backend must never ask the user for their LeetCode session token.
```

---

## 15. Stage 2 and Stage 3 Readiness

Do not implement Stage 2 or Stage 3 now, but keep the architecture compatible.

### 15.1 Stage 2 Readiness — Live Capture

Keep these ready:

```text
- content_scripts manifest entry for LeetCode problem pages
- capture_source field supports 'live_capture'
- code_capture_method field exists
- endpoint path pattern supports future single live submission ingestion
- service worker can receive messages from content script
```

Do not implement:

```text
- fetch interception
- Monaco editor capture
- verdict polling capture
- live upload logic
```

### 15.2 Stage 3 Readiness — Delta Repair

Keep these ready:

```text
- capture_source field supports 'delta_repair'
- user_coding_profiles has last_delta_sync_at
- service worker has alarm heartbeat infrastructure
- backend path pattern supports future delta ingestion
```

Do not implement:

```text
- periodic recentAcSubmissionList repair
- first-pages submissionList repair
- missing ID comparison job
```

---

## 16. Testing Plan

### 16.1 Backend Tests

Create tests for:

```text
1. extension route auth rejects unauthenticated calls
2. connect profile creates/upserts user_coding_profiles row
3. metadata ingest rejects forbidden sensitive keys
4. metadata ingest upserts duplicate submission idempotently
5. metadata ingest creates catalog stub
6. metadata ingest publishes PGMQ event for pending catalog statement
7. aggregate recompute creates user_coding_problems row
8. code ingest writes code_blobs row
9. code ingest sets has_code=true
10. code ingest rejects code for another user
11. sync-state patch updates system.sync_state
12. pause/resume update profile/sync flags
```

### 16.2 Extension Unit Tests

Create tests for:

```text
1. GraphQL alias query builder produces 10 pages
2. code alias query builder maps ids to s0..s9
3. LeetCode response flattening handles missing aliases
4. local storage state patch preserves existing fields
5. progress calculation handles unknown total
6. retry backoff schedules correct next_retry_at
7. sanitization removes forbidden fields
```

### 16.3 Manual QA

Run these manually:

```text
1. Install extension locally.
2. Connect CrackedIn user.
3. Log into LeetCode in same browser.
4. Confirm popup detects username.
5. Start sync.
6. Close popup during metadata sync.
7. Reopen popup and confirm progress persists.
8. Pause sync.
9. Resume sync.
10. Force worker restart and confirm cursor resumes.
11. Complete Phase A.
12. Complete latest 100 code fetch.
13. Confirm background backfill continues.
14. Confirm Supabase rows exist in all Stage 1 tables.
15. Confirm PGMQ events are created and catalog worker consumes them.
16. Confirm no cookie/header data is stored in raw_meta/raw_detail.
```

---

## 17. Acceptance Criteria for Stage 1 Completion

Stage 1 is complete only when all are true:

```text
[ ] Extension can authenticate to CrackedIn.
[ ] Extension can detect signed-in LeetCode user.
[ ] User can explicitly start first-time sync.
[ ] Service worker fetches all historical submission metadata.
[ ] Backend stores metadata in Supabase.
[ ] Backend builds user_coding_problems aggregates.
[ ] Extension fetches latest 100 code details in foreground.
[ ] Backend stores code in code_blobs table.
[ ] Popup shows core sync complete after latest 100 code fetch.
[ ] Background code backfill continues after core completion.
[ ] Failed code fetches go to retry queue.
[ ] Sync state is persisted locally and in Supabase.
[ ] Sync can resume after popup close and service worker shutdown.
[ ] PGMQ submission.received events are published.
[ ] Catalog worker fills missing problem statement fields.
[ ] No LeetCode cookies, CSRF tokens, or request headers are uploaded.
[ ] Existing SQLite app features still work.
[ ] Existing recommendations are not broken.
```

---

## 18. Implementation Sequence for an LLM Repo Agent

Use this exact sequence when asking an implementation LLM to make changes.

### Step 1 — Inspect Repo

Instruction:

```text
Inspect the current repo structure. Confirm api/main.py, api/auth.py, api/database.py, api/config.py, db/schema.sql, db/init_db.py, and web/ exist. Do not modify anything yet. Summarize relevant files.
```

### Step 2 — Add Backend Config and DB Helpers

Instruction:

```text
Add extension environment config to api/config.py. Create api/services/extension_sync/supabase_db.py. Add psycopg dependency. Do not create routes yet.
```

### Step 3 — Add Migrations

Instruction:

```text
Create db/extension_auth_migration.sql for SQLite extension auth tables. Create supabase/migrations/001_extension_sync_stage1.sql with the six Supabase schemas/tables and PGMQ queue setup.
```

### Step 4 — Add Backend Models and Normalization

Instruction:

```text
Create api/services/extension_sync/models.py, normalization.py, and validators.py. Add Pydantic request/response models and helper functions. Include sensitive-key rejection.
```

### Step 5 — Add Profile, Ingest, Aggregate, Sync-State Services

Instruction:

```text
Create profiles.py, ingest_submissions.py, ingest_code.py, aggregates.py, and sync_state.py. Implement idempotent upserts, aggregate recompute, code blob storage, and sync-state patching.
```

### Step 6 — Add PGMQ Catalog Event Publisher and Worker

Instruction:

```text
Create catalog_events.py and catalog_worker.py. Publish submission.received events during ingest when problem statements are missing. Implement a catalog worker that consumes PGMQ and fills catalog.coding_problems.
```

### Step 7 — Add Extension FastAPI Router

Instruction:

```text
Create api/routes/extension_sync.py. Add /api/extension routes for me, connect, sync-state, submissions, code, pause, and resume. Use get_current_user_id for auth. Mount router in api/main.py.
```

### Step 8 — Backend Tests

Instruction:

```text
Add backend tests for route auth, profile connect, metadata ingest, code ingest, aggregate recompute, sync-state updates, and sensitive-key rejection.
```

### Step 9 — Create Extension Scaffold

Instruction:

```text
Create extension/ TypeScript project with manifest v3, Vite config, package.json, tsconfig.json, popup, service-worker, shared storage, API client, and LeetCode adapter folders.
```

### Step 10 — Implement Extension Storage and API Client

Instruction:

```text
Implement LocalSyncState, chrome.storage.local helpers, and crackedinApi.ts for /api/extension calls. Do not implement LeetCode sync yet.
```

### Step 11 — Implement LeetCode Adapter and GraphQL Aliasing

Instruction:

```text
Implement userStatus, submissionList alias query builder, submissionDetails alias query builder, GraphQL fetch helper with credentials: include, and response sanitization.
```

### Step 12 — Implement Service Worker Jobs

Instruction:

```text
Implement metadataSyncJob, foregroundCodeJob, backgroundCodeJob, failedCodeRetryJob, scheduler, and service-worker index message handling. Persist state after every backend-successful batch.
```

### Step 13 — Implement Popup UI

Instruction:

```text
Implement popup.html, popup.ts, and popup.css. Popup reads local state, shows progress, starts sync, pauses sync, and resumes sync. Popup does not run sync loops.
```

### Step 14 — Manual End-to-End Run

Instruction:

```text
Run backend locally, load extension in Chrome, connect CrackedIn, detect LeetCode user, start sync, validate Supabase rows, validate PGMQ catalog events, and confirm no sensitive fields are stored.
```

### Step 15 — Cleanup and Documentation

Instruction:

```text
Add docs/extension-sync-stage1-implementation.md summarizing final setup, env vars, run commands, known limits, and how Stage 2/3 will attach later. Do not add intelligence or recommendation workers.
```

---

## 19. What the Implementation Agent Must Not Do

```text
- Do not create a separate backend microservice.
- Do not migrate all existing SQLite tables to Supabase.
- Do not let the extension write directly to Supabase.
- Do not request Chrome cookies permission.
- Do not read LeetCode cookies.
- Do not upload LeetCode cookies, CSRF tokens, or headers.
- Do not implement live capture in Stage 1.
- Do not implement delta repair in Stage 1.
- Do not run LLM/code analysis during background backfill.
- Do not add intelligence tables yet.
- Do not replace existing recommendation logic yet.
- Do not make popup responsible for sync execution.
- Do not store code in user_coding_submissions.
- Do not trust user IDs from client payloads.
```

---

## 20. Final Build Target

At the end of this implementation, the system should behave like this:

```text
CrackedIn user installs the extension.
The extension connects to the existing CrackedIn account.
The extension confirms the user is logged into LeetCode.
The user starts Stage 1 sync.
The service worker fetches all historical submission metadata with GraphQL aliasing.
The backend stores the complete solve graph in Supabase.
The service worker fetches the latest 100 code submissions.
The popup marks core sync as complete.
The service worker continues background code backfill.
Failed code fetches are retried.
The backend publishes PGMQ catalog events.
The catalog worker fills missing problem statements.
The system remains ready for Stage 2 live capture and Stage 3 delta repair, but neither is implemented yet.
```

This is the final Stage 1 implementation blueprint.
