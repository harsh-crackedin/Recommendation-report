# CrackedIn Extension Sync Design Report

**Status:** Final implementation design  
**Product:** CrackedIn  
**Primary platform:** Chrome/Chromium extension  
**First coding platform:** LeetCode  
**Future platforms:** Codeforces, GeeksforGeeks, code-course platforms, CrackedIn native practice  
**Backend:** Existing FastAPI backend  
**Existing app database:** SQLite  
**Extension corpus database:** Supabase Postgres  
**Primary goal:** Build a user-owned coding corpus from real coding-platform submissions and keep it continuously updated.

---

## 1. Executive Summary

CrackedIn will add a browser extension that connects to the user's existing CrackedIn account, uses the user's browser session to access LeetCode submission data, imports the user's historical submission corpus, and keeps the corpus updated through live capture and periodic repair.

The extension is a new client inside the existing CrackedIn project. It communicates with extension-specific FastAPI endpoints. The existing SQLite database continues to own app users, authentication, chat, current recommendations, and existing app data. Supabase Postgres stores the extension sync corpus: connected coding profiles, per-user solve graphs, individual submissions, code blobs, catalog metadata, and sync cursors.

The extension never reads, stores, or uploads the user's LeetCode session cookie. LeetCode requests are made from the user's browser context with `credentials: 'include'`, allowing Chrome to attach the cookie automatically. CrackedIn receives only sanitized submission data and submitted code returned by LeetCode APIs.

The final architecture is:

```text
Chrome Extension
  ├── Popup UI
  ├── One Manifest V3 service worker
  └── Content scripts for live capture
        │
        │ sanitized batches
        ▼
CrackedIn FastAPI Backend
  ├── Existing SQLite auth/user database
  ├── Extension-specific API routes
  ├── Validation and normalization layer
  ├── Supabase/Postgres writer
  └── Catalog worker
        │
        ▼
Supabase Postgres Extension Corpus
```

---

## 2. Product Goal

The extension exists to create a complete coding corpus for each CrackedIn user.

The corpus includes:

- historical LeetCode submissions,
- accepted attempts,
- failed attempts,
- submitted source code,
- verdict/status,
- language,
- runtime and memory metrics,
- testcase/error details where available,
- problem slug/title/difficulty/topics,
- future submissions captured after onboarding,
- missed submissions recovered through delta repair.

This corpus enables CrackedIn to later build AI features such as code review, mistake pattern detection, spaced repetition, weakness analysis, problem recommendations, and improvement tracking. The raw corpus is the first deliverable; intelligence tables and LLM analysis are intentionally deferred until the raw data pipeline is stable.

---

## 3. Repo-Level Design

The extension will be added inside the current CrackedIn repository.

Final repo structure:

```text
CrackedIn/
├── api/
│   ├── main.py
│   ├── routes/
│   │   └── extension_sync.py
│   ├── services/
│   │   └── extension_sync/
│   │       ├── __init__.py
│   │       ├── auth.py
│   │       ├── supabase_db.py
│   │       ├── profiles.py
│   │       ├── ingest_submissions.py
│   │       ├── ingest_code.py
│   │       ├── sync_state.py
│   │       ├── catalog_events.py
│   │       ├── normalization.py
│   │       ├── validators.py
│   │       └── platforms/
│   │           ├── __init__.py
│   │           ├── leetcode.py
│   │           ├── codeforces.py
│   │           └── gfg.py
│   └── models/
│       └── extension_sync.py
│
├── web/
│   └── existing CrackedIn web app
│
├── extension/
│   ├── manifest.json
│   ├── package.json
│   ├── src/
│   │   ├── popup/
│   │   ├── service-worker/
│   │   ├── content-scripts/
│   │   ├── api-client.ts
│   │   ├── storage.ts
│   │   ├── sync-state.ts
│   │   └── platform-adapters/
│   │       ├── leetcode.ts
│   │       ├── codeforces.ts
│   │       └── gfg.ts
│   └── tests/
│
├── db/
│   └── existing SQLite schema and migrations
│
├── supabase/
│   ├── migrations/
│   │   └── extension_sync_schema.sql
│   └── seed/
│       └── seed_coding_problems.py
│
└── docs/
    └── qkraken_extension_design_report.md
```

The extension is a separate client, but the backend remains the existing FastAPI application. Extension-specific logic is isolated under `/api/extension/*` routes and `api/services/extension_sync/*` service modules.

---

## 4. System Components

| Component | Location | Responsibility |
|---|---|---|
| CrackedIn web app | `web/` | Existing user login, onboarding, dashboard, extension connection page |
| FastAPI backend | `api/` | Auth, extension API, validation, normalization, Supabase writes |
| SQLite database | `data/interview_prep.db` | Existing CrackedIn users, auth, chat, current app data |
| Supabase Postgres | Supabase | Extension corpus and sync-state data |
| Chrome extension popup | `extension/src/popup` | User actions, connect flow, sync progress, pause/resume |
| Chrome extension service worker | `extension/src/service-worker` | Historical sync, code fetch, retry queue, delta repair, backend upload |
| Content scripts | `extension/src/content-scripts` | Live LeetCode page observation and submission detection |
| Platform adapters | Extension + backend | Platform-specific fetch and normalization boundaries |
| Catalog worker | Backend | Fills missing problem statements and catalog metadata |

---

## 5. Identity Model

CrackedIn has two identities in this feature:

1. CrackedIn app identity.
2. External coding-platform identity.

The CrackedIn app identity remains primary.

```text
CrackedIn SQLite users.id
  ↓
Supabase user_data.user_coding_profiles.app_user_id
  ↓
platform = leetcode | codeforces | gfg | qkraken_practice
  ↓
external platform handle / user ID
  ↓
submissions and solve graph
```

The existing SQLite `users.id` is an integer. Supabase extension tables will therefore use:

```sql
app_user_id INTEGER NOT NULL
```

Supabase will not directly foreign-key to SQLite. The FastAPI backend enforces ownership by deriving `app_user_id` from the authenticated CrackedIn token and applying it to every Supabase query and write.

Request bodies must not be trusted for user ownership. The backend always derives the user from the extension access token.

---

## 6. Extension Authentication Design

The extension uses CrackedIn extension-scoped authentication.

### 6.1 Connection Flow

```text
User logs into CrackedIn web app
  ↓
User chooses Connect Extension
  ↓
Backend creates a short-lived one-time auth code
  ↓
Extension receives the auth code
  ↓
Extension exchanges the code for extension tokens
  ↓
Extension stores tokens in chrome.storage.local
  ↓
Extension calls /api/extension/me
  ↓
Extension checks LeetCode auth state
  ↓
User confirms the detected LeetCode account
  ↓
Sync begins
```

### 6.2 Token Model

| Token | Lifetime | Storage | Purpose |
|---|---:|---|---|
| One-time auth code | 1-5 minutes | Temporary only | Initial extension connection |
| Extension access token | 15-60 minutes | `chrome.storage.local` | Authorize `/api/extension/*` calls |
| Extension refresh token | 30-90 days or until revoked | `chrome.storage.local` | Renew access token |
| Main web JWT | Existing web app behavior | Web app | Normal CrackedIn web usage |
| LeetCode cookie | Browser cookie jar only | Browser only | LeetCode requests from extension context |

### 6.3 Extension Access Token Claims

```json
{
  "sub": "123",
  "typ": "extension_access",
  "sid": "extension_session_abc",
  "scopes": [
    "extension:me",
    "coding_profile:write",
    "coding_submission:write",
    "sync_state:write"
  ],
  "exp": 1710000000
}
```

### 6.4 SQLite Session Tables

Add these tables to the existing SQLite database.

```sql
CREATE TABLE IF NOT EXISTS extension_auth_codes (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    code_hash TEXT NOT NULL UNIQUE,
    expires_at TEXT NOT NULL,
    used_at TEXT,
    created_at TEXT DEFAULT (datetime('now'))
);
```

```sql
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
```

These tables keep extension sessions tied to the existing CrackedIn user system.

---

## 7. LeetCode Trust Model

The extension uses the user's existing logged-in LeetCode browser session.

The extension makes requests like:

```ts
fetch("https://leetcode.com/graphql", {
  method: "POST",
  credentials: "include",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ query, variables })
});
```

Chrome attaches the LeetCode cookie automatically because the request is made to `leetcode.com`. The extension code does not read the cookie value.

CrackedIn receives:

- problem metadata,
- submission IDs,
- verdict/status,
- language,
- timestamps,
- runtime/memory metrics,
- submitted code returned by `submissionDetails`,
- testcase/error details when available.

CrackedIn does not receive:

- LeetCode password,
- LeetCode session cookie,
- CSRF token,
- raw request headers,
- unrelated browsing history,
- unrelated page HTML,
- unrelated network events,
- browser storage dumps.

---

## 8. Extension Permission Model

Chrome extension permissions:

```json
{
  "permissions": [
    "storage",
    "alarms"
  ],
  "host_permissions": [
    "https://leetcode.com/*",
    "https://CrackedIn/*"
  ]
}
```

Additional platform host permissions will be added when each platform ships.

For Codeforces:

```json
"https://codeforces.com/*"
```

For GeeksforGeeks:

```json
"https://www.geeksforgeeks.org/*"
```

The extension popup and service worker use CrackedIn backend APIs. LeetCode platform access is done from the extension service worker and content scripts.

---

## 9. Extension Architecture

The extension has three runtime layers.

```text
Chrome Extension
├── Popup
├── One Manifest V3 service worker
└── Content scripts
```

### 9.1 Popup

The popup is only a control and display surface.

It handles:

- CrackedIn connection status,
- LeetCode login status,
- detected platform account display,
- start sync action,
- pause sync action,
- resume sync action,
- manual repair/resync action,
- progress display,
- error and auth-required states.

The popup does not run sync logic. It sends commands to the service worker and reads persisted state from `chrome.storage.local`.

### 9.2 Manifest V3 Service Worker

The extension has one Manifest V3 service worker. Inside that single service worker, CrackedIn runs multiple resumable jobs.

```text
One Chrome service worker
  ├── auth_check_job
  ├── historical_metadata_sync_job
  ├── foreground_code_fetch_job
  ├── background_code_backfill_job
  ├── failed_code_retry_job
  ├── delta_repair_job
  └── live_capture_event_handler
```

The service worker persists state after every batch because Chrome can stop it when idle.

The service worker owns:

- LeetCode `userStatus` check,
- historical metadata sync,
- foreground code fetch,
- background code backfill,
- failed-code retry queue,
- backend uploads,
- local cursor updates,
- backend sync-state updates,
- pause/resume handling,
- delta repair,
- live-capture event processing.

### 9.3 Content Scripts

Content scripts run on supported platform pages.

For LeetCode, the content script runs on:

```text
https://leetcode.com/problems/*
```

The content script observes the page submission flow, detects the final submission/check event, extracts the submission ID, and sends it to the service worker.

The service worker then fetches canonical submission details from LeetCode and uploads them to CrackedIn.

---

## 10. Platform Adapter Architecture

The extension is platform-aware from day one.

### 10.1 Extension-Side Adapter Interface

```ts
export interface PlatformAdapter {
  platform: "leetcode" | "codeforces" | "gfg" | "qkraken_practice";

  checkAuth(): Promise<AuthStatus>;
  getExternalProfile(): Promise<ExternalProfile>;

  fetchMetadataBatch(cursor: SyncCursor): Promise<MetadataBatch>;
  fetchCodeBatch(ids: string[]): Promise<CodeBatch>;

  supportsLiveCapture: boolean;
  supportsDeltaRepair: boolean;
}
```

### 10.2 Backend-Side Adapter Interface

```python
class PlatformIngestAdapter:
    platform: str

    def normalize_submission(self, raw: dict) -> NormalizedSubmission:
        ...

    def normalize_code(self, raw: dict) -> NormalizedCode:
        ...

    def normalize_verdict(self, raw_status: str) -> str:
        ...
```

The first fully implemented adapter is LeetCode. Codeforces and GFG are added later under the same interface.

---

## 11. Backend API Design

All extension APIs live under:

```text
/api/extension
```

Routes are platform-generic where possible.

### 11.1 Auth Endpoints

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/api/extension/auth/start` | Create one-time extension auth code from web session |
| `POST` | `/api/extension/auth/exchange` | Exchange one-time code for extension tokens |
| `POST` | `/api/extension/auth/refresh` | Refresh extension access token |
| `POST` | `/api/extension/auth/revoke` | Revoke extension session |
| `GET` | `/api/extension/me` | Return connected CrackedIn user and extension session state |

### 11.2 Platform Endpoints

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/api/extension/platforms/{platform}/connect` | Create/update connected coding profile |
| `GET` | `/api/extension/platforms/{platform}/sync-state` | Read sync state |
| `PATCH` | `/api/extension/platforms/{platform}/sync-state` | Update phase/cursor/progress |
| `POST` | `/api/extension/platforms/{platform}/submissions` | Ingest metadata batch |
| `POST` | `/api/extension/platforms/{platform}/code` | Ingest code/details batch |
| `POST` | `/api/extension/platforms/{platform}/live` | Ingest live-captured submission |
| `POST` | `/api/extension/platforms/{platform}/delta` | Ingest delta-recovered submissions |
| `POST` | `/api/extension/platforms/{platform}/pause` | Pause sync |
| `POST` | `/api/extension/platforms/{platform}/resume` | Resume sync |
| `DELETE` | `/api/extension/platforms/{platform}` | Disconnect platform profile |
| `DELETE` | `/api/extension/platforms/{platform}/data` | Delete imported platform corpus |

---

## 12. Backend Responsibilities

The backend is the authority for:

- extension token validation,
- user ownership,
- platform profile ownership,
- payload validation,
- schema validation,
- verdict normalization,
- runtime and memory parsing,
- submission deduplication,
- Supabase writes,
- sync-state persistence,
- catalog event publishing,
- delete/export controls.

The extension collects and sanitizes. The backend validates and canonicalizes.

```text
Extension
  collect → sanitize → batch upload

Backend
  authenticate → validate → normalize → dedupe → store
```

---

## 13. Supabase Database Design

Supabase Postgres stores the extension corpus.

```text
Supabase Postgres
├── catalog
│   └── coding_problems
├── user_data
│   ├── user_coding_profiles
│   ├── user_coding_problems
│   └── user_coding_submissions
├── code_blobs
│   └── user_coding_submission_code
└── system
    └── sync_state
```

All user-owned tables use:

```sql
app_user_id INTEGER NOT NULL
```

This maps to the existing CrackedIn SQLite `users.id`.

### 13.1 `catalog.coding_problems`

Shared problem catalog. One row per `(platform, slug)`.

Purpose:

- problem title,
- difficulty,
- topic tags,
- problem statement,
- constraints,
- examples,
- hints,
- similar problems,
- raw platform metadata,
- catalog sync status.

Primary key:

```sql
PRIMARY KEY (platform, slug)
```

The extension does not own this table. The backend inserts stubs and the catalog worker fills full metadata.

### 13.2 `user_data.user_coding_profiles`

Links one CrackedIn user to one external coding account.

Important fields:

- `app_user_id`,
- `platform`,
- `external_user_id`,
- `external_handle`,
- `display_name`,
- `profile_url`,
- `is_premium`,
- `is_connected`,
- `sync_enabled`,
- `last_full_sync_at`,
- `last_delta_sync_at`,
- `last_live_capture_at`,
- `last_sync_error`,
- `raw_profile`.

Unique key:

```sql
UNIQUE (app_user_id, platform)
```

### 13.3 `user_data.user_coding_problems`

Per-user-per-problem solve graph. This is the main dashboard read table.

Important fields:

- `app_user_id`,
- `platform`,
- `slug`,
- `status`,
- `first_attempt_at`,
- `first_ac_at`,
- `latest_ac_at`,
- `latest_attempt_at`,
- `attempt_count`,
- verdict counts,
- `best_runtime_ms`,
- `best_memory_kb`,
- `best_lang`,
- `primary_submission_id`.

Primary key:

```sql
PRIMARY KEY (app_user_id, platform, slug)
```

This table avoids recomputing a user's solve graph from raw submissions on every dashboard load.

### 13.4 `user_data.user_coding_submissions`

Every individual attempt. This is the atomic submission metadata table.

Important fields:

- `platform`,
- `submission_id`,
- `app_user_id`,
- `profile_id`,
- `slug`,
- `verdict`,
- `raw_status`,
- `status_code`,
- `lang`,
- `lang_verbose`,
- `runtime_ms`,
- `runtime_percentile`,
- `memory_kb`,
- `memory_percentile`,
- `total_correct`,
- `total_testcases`,
- failure detail fields,
- `ts`,
- `captured_at`,
- `capture_source`,
- `code_capture_method`,
- `has_code`,
- `raw_meta`.

Primary key:

```sql
PRIMARY KEY (platform, submission_id)
```

The ingestion contract is idempotent. Re-uploading the same submission updates the row and does not create duplicates.

### 13.5 `code_blobs.user_coding_submission_code`

Submitted code is isolated from the hot submission table.

Important fields:

- `platform`,
- `submission_id`,
- `code`,
- `code_lang`,
- `code_hash`,
- `storage_tier`,
- `s3_key`,
- `fetched_at`.

Primary key:

```sql
PRIMARY KEY (platform, submission_id)
```

`storage_tier` and `s3_key` are included now so old code can be archived later without changing the hot submission metadata table.

### 13.6 `system.sync_state`

Per-user-per-platform sync cursor mirrored with `chrome.storage.local`.

Important fields:

- `app_user_id`,
- `platform`,
- `phase`,
- `cursor`,
- `last_progress_at`,
- `last_full_sync_at`,
- `retry_count`,
- `last_error`.

Primary key:

```sql
PRIMARY KEY (app_user_id, platform)
```

---

## 14. Core Sync State Shape

The extension persists sync state in `chrome.storage.local` and mirrors important state to Supabase.

```ts
type SyncState = {
  appUserId: number;
  platform: "leetcode" | "codeforces" | "gfg" | "qkraken_practice";

  phase:
    | "auth_checking"
    | "A_bulk"
    | "B_fg"
    | "B_bg"
    | "live"
    | "delta_repair"
    | "paused"
    | "auth_required"
    | "failed"
    | "done";

  cursor: {
    offset?: number;
    lastPhaseASubmissionIds?: string[];
    foregroundCodeQueue?: string[];
    backgroundCodeQueue?: string[];
    failedCodeQueue?: FailedCodeItem[];
    metadataFetched?: number;
    codeFetched?: number;
    codeFailed?: number;
    uploadedTotal?: number;
  };

  pauseRequested: boolean;
  lastProgressAt: number;
  lastFullSyncAt?: number;
  lastDeltaSyncAt?: number;
  lastLiveCaptureAt?: number;
  error?: string;
};
```

Failed code queue item:

```ts
type FailedCodeItem = {
  platform: string;
  submissionId: string;
  slug?: string;
  reason: string;
  retryCount: number;
  nextRetryAt: number;
  lastError?: string;
};
```

---

## 15. Stage 1: First-Time Historical Sync

Stage 1 imports the user's historical LeetCode corpus.

Stage 1 has three jobs inside the single extension service worker:

```text
Stage 1A: Bulk metadata sync
Stage 1B: Foreground last-100 code fetch
Stage 1C: Background code backfill + failed-code retry
```

### 15.1 Stage 1A: Bulk Metadata Sync

Goal:

```text
Fetch every historical LeetCode submission metadata record.
```

LeetCode source:

```graphql
submissionList(offset: N, limit: 20)
```

Use GraphQL aliases to fetch 10 pages per request:

```graphql
query bulkPages {
  p0: submissionList(offset: 0, limit: 20) { hasNext submissions { id titleSlug statusDisplay langName runtime memory timestamp } }
  p1: submissionList(offset: 20, limit: 20) { hasNext submissions { id titleSlug statusDisplay langName runtime memory timestamp } }
  p2: submissionList(offset: 40, limit: 20) { hasNext submissions { id titleSlug statusDisplay langName runtime memory timestamp } }
  p3: submissionList(offset: 60, limit: 20) { hasNext submissions { id titleSlug statusDisplay langName runtime memory timestamp } }
  p4: submissionList(offset: 80, limit: 20) { hasNext submissions { id titleSlug statusDisplay langName runtime memory timestamp } }
  p5: submissionList(offset: 100, limit: 20) { hasNext submissions { id titleSlug statusDisplay langName runtime memory timestamp } }
  p6: submissionList(offset: 120, limit: 20) { hasNext submissions { id titleSlug statusDisplay langName runtime memory timestamp } }
  p7: submissionList(offset: 140, limit: 20) { hasNext submissions { id titleSlug statusDisplay langName runtime memory timestamp } }
  p8: submissionList(offset: 160, limit: 20) { hasNext submissions { id titleSlug statusDisplay langName runtime memory timestamp } }
  p9: submissionList(offset: 180, limit: 20) { hasNext submissions { id titleSlug statusDisplay langName runtime memory timestamp } }
}
```

Loop:

```text
offset = sync_state.cursor.offset || 0

while true:
  build 10-page aliased submissionList query
  fetch from LeetCode with credentials: include
  flatten submissions
  upload metadata batch to CrackedIn backend
  persist local sync cursor
  mirror backend sync state

  if any page hasNext=false:
    build last-100 foreground code queue
    build remaining background code queue
    phase = B_fg
    break
  else:
    offset += 200
```

Backend writes:

| Table | Write |
|---|---|
| `user_data.user_coding_profiles` | Upsert connected LeetCode profile |
| `catalog.coding_problems` | Stub unknown slug if needed |
| `user_data.user_coding_submissions` | Upsert metadata, `has_code=false` |
| `user_data.user_coding_problems` | Refresh affected solve-graph aggregates |
| `system.sync_state` | Update phase and cursor |

Backend also publishes `submission.received` events for unknown or pending problem slugs.

### 15.2 Stage 1B: Foreground Last-100 Code Fetch

Goal:

```text
Fetch code/details for the 100 most recent submissions before marking core sync as usable.
```

LeetCode source:

```graphql
submissionDetails(submissionId: ID)
```

Use GraphQL aliases to fetch 10 details per request:

```graphql
query codeBatch {
  s0: submissionDetails(submissionId: 123) { code lang { name } runtime memory runtimePercentile memoryPercentile totalCorrect totalTestcases question { titleSlug } }
  s1: submissionDetails(submissionId: 124) { code lang { name } runtime memory runtimePercentile memoryPercentile totalCorrect totalTestcases question { titleSlug } }
}
```

Loop:

```text
queue = sync_state.cursor.foregroundCodeQueue

while queue not empty:
  take next 10 submission IDs
  fetch aliased submissionDetails
  upload code batch to CrackedIn backend
  update codeFetched count
  persist cursor

phase = B_bg
show core sync complete
```

Backend writes:

| Table | Write |
|---|---|
| `code_blobs.user_coding_submission_code` | Upsert code blob and hash |
| `user_data.user_coding_submissions` | Set `has_code=true`, update detail fields |
| `user_data.user_coding_problems` | Refresh aggregate if needed |
| `system.sync_state` | Update code progress |

At the end of foreground code fetch, the dashboard is usable and code-aware tools can access the recent corpus.

### 15.3 Stage 1C: Background Code Backfill

Goal:

```text
Fetch remaining historical code without blocking the user.
```

Behavior:

- Runs from the same service worker.
- Uses `backgroundCodeQueue`.
- Fetches 10 submissions per LeetCode detail request.
- Uploads details to CrackedIn backend.
- Persists cursor after every batch.
- Uses a slower cadence to reduce platform pressure.
- Continues across popup closes and service worker restarts.
- Pauses when browser closes and resumes when the extension wakes again.

### 15.4 Failed-Code Retry Queue

Submissions that fail during code fetch are added to `failedCodeQueue`.

Retry causes:

- LeetCode timeout,
- rate limiting,
- temporary network failure,
- malformed response,
- backend upload failure,
- auth expiry,
- service worker interruption.

Retry strategy:

| Retry count | Next action |
|---:|---|
| 1 | Retry after short backoff |
| 2 | Retry after longer backoff |
| 3 | Retry during later background cycle |
| 4+ | Keep in persistent retry queue and surface as recoverable |

The metadata row remains stored even when `has_code=false`. The dashboard and solve graph do not block on missing code.

---

## 16. Stage 2: Live Capture

Stage 2 captures new submissions made after historical sync.

### 16.1 Live Capture Flow

```text
User opens LeetCode problem page
  ↓
Content script is active
  ↓
User submits code
  ↓
LeetCode judging/check request is observed
  ↓
Content script extracts submission ID
  ↓
Content script sends NEW_SUBMISSION message to service worker
  ↓
Service worker calls submissionDetails(submissionId)
  ↓
Service worker uploads official details to CrackedIn backend
  ↓
Backend upserts submission, code blob, and solve graph
```

### 16.2 Canonical Code Rule

The canonical submitted code is the server-confirmed code returned by:

```graphql
submissionDetails(submissionId).code
```

Client-captured editor code can be included as secondary/debug context later, but the official stored code is the server-confirmed LeetCode detail response.

### 16.3 Live Capture Backend Writes

| Table | Write |
|---|---|
| `user_data.user_coding_submissions` | Upsert live attempt with `capture_source='live_capture'` |
| `code_blobs.user_coding_submission_code` | Store official submitted code |
| `user_data.user_coding_problems` | Refresh affected problem aggregate |
| `user_data.user_coding_profiles` | Update `last_live_capture_at` |
| `system.sync_state` | Keep phase/live heartbeat |

---

## 17. Stage 3: Delta Repair

Stage 3 catches submissions missed by live capture.

Delta repair runs from the extension service worker.

Triggers:

- `chrome.alarms`,
- browser startup,
- popup open,
- manual repair action,
- auth recovery,
- after live-capture failure.

LeetCode sources:

```text
recentAcSubmissionList(username, limit)
+
first few pages of submissionList(offset, limit: 20)
```

The general `submissionList` pages are required because recent accepted submissions alone do not include failed submissions.

Flow:

```text
Service worker wakes
  ↓
Checks CrackedIn token
  ↓
Checks LeetCode auth
  ↓
Fetches recent accepted submissions
  ↓
Fetches first N pages of general submissionList
  ↓
Compares IDs against local/backend known IDs
  ↓
Fetches details for missing IDs
  ↓
Uploads recovered submissions
  ↓
Updates last_delta_sync_at
```

Backend writes use `capture_source='delta_repair'`.

---

## 18. Catalog Worker

The backend has one Stage 1-3 worker: the catalog worker.

Event type:

```json
{
  "type": "submission.received",
  "app_user_id": 123,
  "platform": "leetcode",
  "slug": "two-sum",
  "submission_id": 123456789
}
```

Catalog worker flow:

```text
Receive submission.received event
  ↓
Check catalog.coding_problems for platform + slug
  ↓
If statement_md is missing or sync_status='pending'
  ↓
Fetch platform problem details
  ↓
Convert statement HTML to markdown
  ↓
Update catalog.coding_problems
```

For LeetCode, the worker fetches problem details using `question(titleSlug)`.

The catalog worker is idempotent. Multiple events for the same slug do not create duplicate catalog rows.

---

## 19. Event Queue

Supabase `pgmq` is used as the backend event queue.

Stage 1-3 use one producer and one consumer.

Producer:

```text
FastAPI ingestion service
```

Consumer:

```text
Catalog worker
```

Queue event:

```text
submission.received
```

The event is published after the backend saves or updates a submission whose problem catalog row is missing or incomplete.

---

## 20. Data Normalization

### 20.1 Platform Names

Use these platform values:

```text
leetcode
codeforces
gfg
qkraken_practice
```

### 20.2 Problem Identity

LeetCode canonical problem key:

```text
titleSlug
```

Universal identity:

```text
platform + slug
```

### 20.3 Submission Identity

Universal submission identity:

```text
platform + submission_id
```

### 20.4 Verdict Normalization

| Raw LeetCode status | Normalized verdict |
|---|---|
| `Accepted` | `AC` |
| `Wrong Answer` | `WA` |
| `Time Limit Exceeded` | `TLE` |
| `Memory Limit Exceeded` | `MLE` |
| `Runtime Error` | `RE` |
| `Compile Error` | `CE` |
| Other/unknown | `OTHER` |

Store both `verdict` and `raw_status`.

### 20.5 Runtime and Memory

The backend parses runtime and memory into normalized numeric fields:

```text
runtime_ms
memory_kb
```

The original raw fragment remains in `raw_meta` when useful for debugging.

### 20.6 Code Hash

The backend computes SHA-256 for submitted code:

```text
code_hash = sha256(code)
```

This supports deduplication, change detection, and future archival.

---

## 21. Backend Ingestion Contracts

### 21.1 Connect Platform

```http
POST /api/extension/platforms/leetcode/connect
Authorization: Bearer <extension_access_token>
```

```json
{
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

Backend derives `app_user_id` from the extension token.

### 21.2 Metadata Batch

```http
POST /api/extension/platforms/leetcode/submissions
Authorization: Bearer <extension_access_token>
```

```json
{
  "batch": [
    {
      "submission_id": "123456789",
      "slug": "two-sum",
      "title": "Two Sum",
      "raw_status": "Accepted",
      "lang": "cpp",
      "lang_verbose": "C++",
      "runtime_display": "4 ms",
      "memory_display": "12.5 MB",
      "timestamp": 1710000000
    }
  ],
  "cursor_offset": 200,
  "last_batch": false
}
```

Backend response:

```json
{
  "ok": true,
  "records_processed": 200,
  "unknown_slugs_queued": 3
}
```

### 21.3 Code Batch

```http
POST /api/extension/platforms/leetcode/code
Authorization: Bearer <extension_access_token>
```

```json
{
  "batch": [
    {
      "submission_id": "123456789",
      "slug": "two-sum",
      "code": "class Solution { ... }",
      "code_lang": "cpp",
      "runtime_ms": 4,
      "runtime_percentile": 91.2,
      "memory_kb": 12800,
      "memory_percentile": 72.5,
      "total_correct": 57,
      "total_testcases": 57,
      "raw_detail": {}
    }
  ]
}
```

Backend response:

```json
{
  "ok": true,
  "records_processed": 10,
  "code_rows_upserted": 10
}
```

---

## 22. Progress UI Design

The progress bar is state-driven. The service worker writes state. The popup reads state.

Progress phases:

| Phase | Popup message |
|---|---|
| `auth_checking` | Connecting to CrackedIn and LeetCode |
| `A_bulk` | Finding your LeetCode submissions |
| `B_fg` | Fetching recent submitted code |
| `B_bg` | Core sync complete. Background sync continuing |
| `live` | Sync complete. Live capture enabled |
| `delta_repair` | Checking for missed submissions |
| `paused` | Sync paused |
| `auth_required` | Log in to LeetCode to continue |
| `failed` | Sync paused due to an error. Retry available |
| `done` | Full sync complete |

Metadata phase starts indeterminate because total submissions are unknown. Once metadata discovery completes, code-fetch progress becomes determinate.

Suggested weight for visual progress:

| Phase | Weight |
|---|---:|
| Metadata sync | 35% |
| Foreground code sync | 60% |
| Finalization | 5% |

Popup copy:

```text
You can close this popup or switch tabs. Sync will continue while your browser remains open. If the browser closes, sync pauses and resumes later.
```

---

## 23. Pause, Resume, and Retry

Pause is user-controlled.

Pause behavior:

```text
User clicks Pause
  ↓
Popup sends PAUSE_SYNC to service worker
  ↓
Service worker sets pauseRequested=true
  ↓
Current safe batch finishes
  ↓
Cursor is persisted
  ↓
Backend sync state becomes paused
```

Resume behavior:

```text
User clicks Resume
  ↓
Service worker checks CrackedIn token
  ↓
Service worker checks platform auth
  ↓
Service worker loads cursor
  ↓
Sync continues from saved phase
```

Retry behavior:

- backend writes are idempotent,
- metadata batches can be safely replayed,
- code batches can be safely replayed,
- failed code fetches remain in `failedCodeQueue`,
- network failures use backoff,
- auth failures move sync to `auth_required`.

---

## 24. Browser and Tab Behavior

| User action | Historical sync | Live capture |
|---|---|---|
| User switches tabs | Continues or resumes | Works if platform tab remains open |
| User closes popup | Continues or resumes | Not affected |
| User closes LeetCode tab | Continues or resumes | Stops until page opens again |
| User closes browser | Pauses | Stops |
| Browser reopens | Resumes from cursor | Works after platform page opens |
| User logs out of LeetCode | Moves to `auth_required` | Stops |

The service worker cannot rely on memory. It must persist state after every batch.

---

## 25. Data Security and Privacy Requirements

CrackedIn stores coding-submission data intentionally provided through the extension sync flow.

The extension uploads:

- platform profile handle and ID,
- problem slug/title,
- submission ID,
- submission timestamp,
- verdict/status,
- language,
- runtime/memory metrics,
- testcase/error details where available,
- submitted code,
- sanitized raw metadata needed for debugging.

The extension does not upload:

- LeetCode password,
- LeetCode session cookie,
- CSRF token,
- raw browser headers,
- unrelated network traffic,
- unrelated page HTML,
- browsing history,
- local/session storage dumps.

Data controls:

- pause sync,
- resume sync,
- disconnect platform,
- delete imported platform data,
- export imported platform data.

---

## 26. CrackedIn Backend Configuration

Add environment variables:

```env
SUPABASE_DB_URL=
SUPABASE_SERVICE_ROLE_KEY=
SUPABASE_PROJECT_URL=
EXTENSION_JWT_SECRET=
EXTENSION_ACCESS_TOKEN_MINUTES=60
EXTENSION_REFRESH_TOKEN_DAYS=90
EXTENSION_AUTH_CODE_TTL_SECONDS=300
```

The Supabase service role key is backend-only. It is never exposed to the extension.

---

## 27. Implementation Plan

### 27.1 Backend Foundation

1. Add Supabase Postgres client under `api/services/extension_sync/supabase_db.py`.
2. Add Supabase migration directory.
3. Create four Supabase schemas: `catalog`, `user_data`, `code_blobs`, `system`.
4. Create six extension sync tables using `app_user_id INTEGER`.
5. Add SQLite tables for `extension_auth_codes` and `extension_sessions`.
6. Add extension token creation, decoding, refresh, and revocation logic.
7. Add `api/routes/extension_sync.py`.
8. Mount extension router in `api/main.py`.

### 27.2 Backend Ingestion

1. Implement `/api/extension/me`.
2. Implement `/api/extension/platforms/{platform}/connect`.
3. Implement metadata batch ingestion.
4. Implement code batch ingestion.
5. Implement verdict normalization.
6. Implement runtime/memory parsing.
7. Implement `user_coding_problems` aggregate refresh.
8. Implement `system.sync_state` read/update.
9. Implement platform disconnect.
10. Implement imported data deletion.

### 27.3 Catalog Worker

1. Add pgmq queue setup.
2. Publish `submission.received` on unknown/pending slugs.
3. Implement catalog worker loop.
4. Fetch LeetCode problem statement/details.
5. Convert HTML to markdown.
6. Update `catalog.coding_problems`.
7. Add retry/dead-letter handling for failed catalog jobs.

### 27.4 Extension Foundation

1. Add `extension/` project.
2. Add Manifest V3 config.
3. Add popup app.
4. Add service worker.
5. Add `chrome.storage.local` state helpers.
6. Add CrackedIn API client.
7. Add extension auth exchange.
8. Add LeetCode platform adapter.
9. Add progress UI.

### 27.5 Stage 1 Sync

1. Implement LeetCode `userStatus` check.
2. Implement profile detection and confirmation.
3. Implement Phase A metadata sync.
4. Implement GraphQL aliasing for `submissionList`.
5. Upload metadata batches to backend.
6. Persist cursor after every batch.
7. Build foreground and background code queues.
8. Implement Phase B foreground last-100 code fetch.
9. Implement GraphQL aliasing for `submissionDetails`.
10. Upload code batches.
11. Implement Phase B background backfill.
12. Implement failed-code retry queue.

### 27.6 Stage 2 Live Capture

1. Add LeetCode problem-page content script.
2. Detect submission/check request.
3. Extract submission ID.
4. Send event to service worker.
5. Fetch official `submissionDetails`.
6. Upload live submission/code to backend.
7. Update `last_live_capture_at`.

### 27.7 Stage 3 Delta Repair

1. Add `chrome.alarms` repair trigger.
2. Add startup/popup-open repair trigger.
3. Fetch recent accepted submissions.
4. Fetch recent general submission pages.
5. Compare IDs with known local/backend IDs.
6. Fetch missing details.
7. Upload recovered submissions with `capture_source='delta_repair'`.
8. Update `last_delta_sync_at`.

### 27.8 Future Platform Integration

1. Add Codeforces extension adapter.
2. Add Codeforces backend normalization adapter.
3. Add GFG extension adapter.
4. Add GFG backend normalization adapter.
5. Reuse the same Supabase schema and `/api/extension/platforms/{platform}/*` endpoints.

---

## 28. Testing Plan

### 28.1 Auth Tests

- Extension auth code is created for logged-in user.
- Auth code expires after TTL.
- Auth code can be used only once.
- Extension access token authorizes extension routes.
- Refresh token rotates or refreshes correctly.
- Revoked session cannot refresh.

### 28.2 LeetCode Auth Tests

- Logged-in LeetCode user returns signed-in status.
- Logged-out LeetCode user returns auth-required state.
- Detected LeetCode account is shown before sync.
- Account switch requires confirmation.

### 28.3 Stage 1 Metadata Tests

- First metadata batch inserts submissions.
- Replayed metadata batch does not duplicate rows.
- Cursor resumes after service worker interruption.
- Unknown slugs create catalog stubs.
- `user_coding_problems` aggregates are correct.

### 28.4 Stage 1 Code Tests

- Last 100 code queue is built correctly.
- Code batch inserts code blobs.
- `has_code` updates to true.
- Failed detail fetch enters retry queue.
- Replayed code batch is idempotent.

### 28.5 Live Capture Tests

- Content script loads on LeetCode problem pages.
- Submit/check request is detected.
- Submission ID is extracted.
- Service worker fetches official details.
- Backend stores live submission and code.

### 28.6 Delta Repair Tests

- Missed accepted submission is recovered.
- Missed failed submission is recovered.
- Existing IDs are skipped.
- Missing IDs are fetched and upserted.
- `last_delta_sync_at` updates.

### 28.7 Privacy Tests

- Extension never reads cookie values.
- Extension payloads contain no request headers.
- Extension payloads contain no CSRF token.
- Extension uploads only structured coding data.

### 28.8 Browser Lifecycle Tests

- Popup close does not lose progress.
- Tab switch does not stop historical sync.
- Service worker restart resumes from cursor.
- Browser close pauses sync.
- Browser reopen resumes sync.

---

## 29. Deployment Plan

### 29.1 Backend Deployment

1. Deploy Supabase schema migration.
2. Add production environment variables.
3. Deploy FastAPI extension routes.
4. Enable extension auth endpoints.
5. Enable Supabase write client.
6. Enable catalog queue and worker.
7. Monitor ingestion logs and queue failures.

### 29.2 Extension Deployment

1. Build extension package.
2. Test locally through Chrome developer mode.
3. Validate LeetCode sync on test accounts.
4. Validate CrackedIn auth exchange.
5. Validate popup progress and resume behavior.
6. Submit extension to Chrome Web Store.
7. Use explicit privacy wording for coding-submission data collection.

---

## 30. Final Implementation Summary

CrackedIn will implement the extension sync system as a browser-extension client plus extension-specific APIs inside the existing FastAPI backend.

The extension has one Manifest V3 service worker that runs multiple resumable jobs: historical metadata sync, foreground code fetch, background code backfill, failed-code retry, delta repair, and live-capture event handling.

The content script handles Stage 2 live submission detection on LeetCode pages. The service worker fetches official submission details and sends sanitized data to CrackedIn.

The backend authenticates the extension user through extension-scoped tokens tied to the existing SQLite user account. The backend validates and normalizes all payloads, writes the coding corpus to Supabase Postgres, updates sync state, and publishes catalog events.

Supabase stores the extension corpus in six tables across four schemas. Existing SQLite remains the source of truth for CrackedIn users and current app data.

The system is platform-aware from day one. LeetCode ships first, while Codeforces, GFG, and CrackedIn native practice integrate through the same platform adapter and database model.

This is the final architecture for implementing CrackedIn's extension-based coding corpus sync.
