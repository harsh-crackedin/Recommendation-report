# CrackedIn Extension Sync — Stage 1 Design and Implementation Report

**Status:** Final Stage 1 implementation design  
**Product:** CrackedIn  
**Repository:** `crackedinlabs/interview-prep-ai`  
**Primary client:** Chrome/Chromium extension  
**Stage 1 platform:** LeetCode  
**Future platforms prepared for:** Codeforces, GeeksforGeeks, code-course platforms, CrackedIn native practice  
**Existing backend:** FastAPI  
**Existing app database:** SQLite  
**Extension corpus database:** Supabase Postgres  
**Stage 1 goal:** Import the user's complete historical LeetCode submission metadata, fetch recent submitted code immediately, and backfill remaining code in the background using a resumable extension service-worker pipeline.

---

## 1. Executive Summary

CrackedIn will add a browser extension on top of the existing InterviewPrep.ai/CrackedIn application. The extension connects to the user's existing CrackedIn account, verifies the user's active LeetCode browser session, imports their historical LeetCode submission corpus, and stores it in Supabase Postgres through CrackedIn's FastAPI backend.

Stage 1 focuses only on **first-time historical sync**.

Stage 1 will deliver:

1. A Chrome extension client inside the current repo.
2. Extension authentication using CrackedIn account identity.
3. LeetCode login detection through the user's browser session.
4. Full historical submission metadata sync.
5. Immediate foreground fetch of the latest 100 submitted code blobs.
6. Background backfill of remaining historical code blobs.
7. Persistent sync progress and retry queues.
8. Supabase Postgres storage using the finalized 6-table extension sync schema.
9. Backend catalog enrichment for missing problem metadata.
10. A design that is ready for Stage 2 live capture and Stage 3 delta repair without implementing those stages now.

The final Stage 1 architecture is:

```text
Chrome Extension
  ├── Popup UI
  ├── One Manifest V3 service worker
  │     ├── Stage 1A historical metadata sync job
  │     ├── Stage 1B foreground code fetch job
  │     ├── Stage 1C background code backfill job
  │     └── failed-code retry queue
  └── Platform adapter: LeetCode
        │
        │ sanitized batches
        ▼
CrackedIn FastAPI Backend
  ├── Existing SQLite auth and user identity
  ├── Extension-scoped auth/session routes
  ├── Extension sync ingestion routes
  ├── Validation and normalization services
  ├── Supabase Postgres writer
  └── Catalog worker
        │
        ▼
Supabase Postgres Extension Corpus
  ├── catalog.coding_problems
  ├── user_data.user_coding_profiles
  ├── user_data.user_coding_problems
  ├── user_data.user_coding_submissions
  ├── code_blobs.user_coding_submission_code
  └── system.sync_state
```

---

## 2. Stage 1 Scope

Stage 1 is the first-time sync system. It creates the raw coding corpus needed by later CrackedIn intelligence features.

### 2.1 Included in Stage 1

| Area | Included behavior |
|---|---|
| Extension packaging | Chrome/Chromium extension added under `extension/` |
| CrackedIn auth | Extension connects to existing CrackedIn account |
| LeetCode auth check | Extension verifies active LeetCode browser session |
| Historical metadata | Fetch all LeetCode submission metadata using `submissionList` |
| Code fetch | Fetch latest 100 submitted code blobs in foreground |
| Background backfill | Fetch all remaining historical code blobs after core sync completes |
| Retry queue | Persist and retry failed code/detail fetches |
| Backend ingestion | Validate, normalize, dedupe, and store extension data |
| Supabase schema | Use the 6-table extension sync schema adapted for current SQLite users |
| Catalog worker | Fill missing problem statements/metadata asynchronously |
| Progress UI | Popup displays sync phase, counts, status, pause/resume state |
| Resumability | Sync resumes after service worker shutdown, popup close, tab switch, or browser restart |
| Future readiness | Schema, routes, and fields prepared for Stage 2 and Stage 3 |

### 2.2 Not Implemented in Stage 1

These are intentionally not part of Stage 1 execution:

| Area | Stage 1 decision |
|---|---|
| Stage 2 live capture | Prepare route/schema compatibility only; do not implement full live tracker yet |
| Stage 3 delta repair | Prepare route/schema compatibility only; do not implement periodic repair yet |
| LLM code analysis | Defer until raw corpus is stable |
| Weakness intelligence tables | Defer until raw corpus is stable |
| Recommendation precomputation | Defer until raw corpus is stable |
| Code embeddings/similarity | Defer until intelligence design pass |
| S3 archival | Columns are present, but all Stage 1 code stays in Postgres |
| Full app DB migration | Existing app remains SQLite-first |

---

## 3. Existing CrackedIn System Context

The current CrackedIn app already has:

1. A FastAPI backend under `api/`.
2. A Next.js frontend under `web/`.
3. SQLite as the existing app database.
4. JWT authentication.
5. Existing LeetCode public profile sync.
6. Existing interview-data and recommendation features.

Stage 1 must extend this architecture without replacing it.

### 3.1 Existing SQLite Responsibilities

SQLite remains responsible for:

| SQLite area | Purpose |
|---|---|
| `users` | Primary CrackedIn account identity |
| `user_lc_profiles` | Existing public LeetCode profile snapshot |
| `user_lc_topics` | Existing topic-level public profile stats |
| `user_contest_history` | Existing contest profile stats |
| `user_solved_problems` | Existing solved-problem list used by current recommendations |
| `chat_sessions`, `chat_messages` | Existing chat history |
| `interview_posts`, `interview_rounds`, problem tables | Existing interview intelligence and recommendation data |

### 3.2 New Supabase Responsibilities

Supabase Postgres is responsible only for extension sync data:

| Supabase area | Purpose |
|---|---|
| Connected coding profiles | External coding platform accounts linked to CrackedIn users |
| User solve graph | Per-user-per-problem aggregate derived from submissions |
| Submission metadata | Every historical attempt, accepted or failed |
| Code blobs | Submitted source code stored separately from hot metadata |
| Sync state | Durable backend cursor and phase tracking |
| Shared coding catalog | Platform problem metadata for LeetCode now and other platforms later |

The extension corpus is not stored in SQLite.

---

## 4. Repository Design

The extension will live inside the existing repo as a separate client. Backend extension logic will live inside the existing FastAPI backend as a new module group.

```text
interview-prep-ai/
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
│   │       ├── catalog_worker.py
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
├── extension/
│   ├── manifest.json
│   ├── package.json
│   ├── tsconfig.json
│   ├── src/
│   │   ├── popup/
│   │   │   ├── Popup.tsx
│   │   │   ├── ProgressPanel.tsx
│   │   │   └── ConnectPanel.tsx
│   │   ├── service-worker/
│   │   │   ├── index.ts
│   │   │   ├── job-runner.ts
│   │   │   ├── sync-stage-1.ts
│   │   │   ├── code-backfill.ts
│   │   │   ├── retry-queue.ts
│   │   │   └── alarms.ts
│   │   ├── platform-adapters/
│   │   │   ├── types.ts
│   │   │   └── leetcode.ts
│   │   ├── api-client.ts
│   │   ├── auth.ts
│   │   ├── storage.ts
│   │   └── sync-state.ts
│   └── tests/
│
├── supabase/
│   ├── migrations/
│   │   └── 001_extension_sync_schema.sql
│   └── seed/
│       └── seed_leetcode_catalog.py
│
├── db/
│   ├── schema.sql
│   └── init_db.py
│
└── docs/
    └── crackedin_stage1_extension_design_report.md
```

---

## 5. Identity and Ownership Model

The existing CrackedIn user remains the root identity.

```text
SQLite users.id
  ↓
Extension access token subject
  ↓
Supabase app_user_id
  ↓
user_coding_profiles
  ↓
user_coding_submissions and user_coding_problems
```

### 5.1 User ID Adaptation

The existing app uses SQLite integer user IDs. Supabase cannot directly foreign-key into SQLite, so extension tables use:

```sql
app_user_id INTEGER NOT NULL
```

The backend must never trust an `app_user_id` from the extension body. It always derives the user from the authenticated extension token.

### 5.2 Connected Platform Identity

For LeetCode Stage 1:

```text
app_user_id = existing CrackedIn SQLite users.id
platform = 'leetcode'
external_handle = LeetCode username
external_user_id = LeetCode userId if available
```

For future platforms, the same shape remains:

```text
app_user_id + platform = one connected coding profile
```

---

## 6. Extension Authentication Design

Stage 1 uses extension-scoped CrackedIn authentication.

### 6.1 Extension Connection Flow

```text
1. User logs into CrackedIn web app.
2. User clicks Connect Extension.
3. Backend creates a short-lived one-time auth code.
4. Extension receives the auth code.
5. Extension exchanges the code for extension tokens.
6. Extension stores extension tokens in chrome.storage.local.
7. Extension calls GET /api/extension/me.
8. Backend validates token and returns current CrackedIn user summary.
9. Extension proceeds to LeetCode auth check.
```

### 6.2 Token Types

| Token | Storage | Lifetime | Purpose |
|---|---|---:|---|
| One-time auth code | Not persisted | 1-5 minutes | Connect extension session once |
| Extension access token | `chrome.storage.local` | 15-60 minutes | Authorize `/api/extension/*` requests |
| Extension refresh token | `chrome.storage.local` | 30-90 days or until revoked | Refresh extension access token |
| Main web JWT | Existing web app behavior | Existing setting | Normal web app usage |
| LeetCode cookie | Browser cookie jar only | Controlled by LeetCode | Attached automatically to LeetCode requests |

### 6.3 Extension Token Claims

Access token payload:

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
  "iat": 1710000000,
  "exp": 1710003600
}
```

### 6.4 SQLite Session Tables

Add these tables to SQLite because app identity still lives there.

```sql
CREATE TABLE IF NOT EXISTS extension_auth_codes (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    code_hash TEXT NOT NULL UNIQUE,
    expires_at TEXT NOT NULL,
    used_at TEXT,
    created_at TEXT DEFAULT (datetime('now'))
);

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

---

## 7. LeetCode Trust Model

The extension uses the user's existing browser session for LeetCode access.

The extension must not:

- ask the user to paste `LEETCODE_SESSION`,
- read the LeetCode cookie directly,
- upload the LeetCode cookie,
- upload CSRF tokens,
- upload raw browser headers,
- upload unrelated browsing data,
- upload unrelated page HTML.

The extension calls:

```ts
fetch("https://leetcode.com/graphql", {
  method: "POST",
  credentials: "include",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ query, variables })
});
```

Chrome attaches the user's existing LeetCode cookies automatically because the request goes to `leetcode.com`. The cookie value never enters CrackedIn's backend.

### 7.1 Required Extension Permissions

`manifest.json` Stage 1 permissions:

```json
{
  "manifest_version": 3,
  "name": "CrackedIn",
  "permissions": ["storage", "alarms"],
  "host_permissions": [
    "https://leetcode.com/*",
    "https://crackedin.app/*"
  ],
  "background": {
    "service_worker": "service-worker.js",
    "type": "module"
  },
  "action": {
    "default_popup": "popup.html"
  }
}
```

Stage 1 does not require `cookies` permission.

---

## 8. Stage 1 Extension Architecture

Chrome Manifest V3 gives the extension **one background service worker**. Stage 1 jobs run as queues/phases inside that single service worker.

```text
One Manifest V3 service worker
  ├── auth check job
  ├── Stage 1A metadata sync job
  ├── Stage 1B foreground code fetch job
  ├── Stage 1C background code backfill job
  ├── failed-code retry job
  ├── progress writer
  └── backend upload client
```

### 8.1 Extension Component Ownership

| Component | Stage 1 responsibility |
|---|---|
| Popup | Shows connection status, LeetCode auth status, sync button, progress, pause/resume |
| Service worker | Owns all Stage 1 sync jobs and backend upload |
| `chrome.storage.local` | Stores tokens, cursor, progress, queues, retry state |
| LeetCode adapter | Builds GraphQL queries and parses LeetCode responses |
| API client | Talks to CrackedIn backend only |

### 8.2 Service Worker Lifecycle Rule

The service worker can be stopped by Chrome at any time. Therefore:

1. No critical state lives only in memory.
2. Cursor is written after every successful batch.
3. Backend writes are idempotent.
4. A repeated batch must be safe.
5. On wake, the service worker reads state and resumes.

---

## 9. Stage 1 Sync Flow Overview

Stage 1 has three active sync phases:

```text
A_bulk  →  B_fg  →  B_bg  →  live_ready
```

| Phase | User impact | What happens |
|---|---|---|
| `A_bulk` | Required | Fetch all historical submission metadata |
| `B_fg` | Required for core sync complete | Fetch latest 100 code blobs |
| `B_bg` | Non-blocking | Backfill remaining historical code |
| `live_ready` | Stage 1 endpoint state | Historical sync is ready for future Stage 2/3 |

### 9.1 User-Facing Timeline

```text
T=0      User connects extension to CrackedIn.
T=5s     Extension verifies LeetCode login.
T=10s    User confirms detected LeetCode account.
T=10s    Stage 1A starts: metadata sync.
T=40s    Metadata sync completes for typical user.
T=40s    Dashboard can show full solve graph.
T=44s    Latest 100 code blobs fetched.
T=45s    Core sync complete.
T=45s+   Remaining code blobs backfill in background.
```

The exact timing depends on account size and LeetCode network behavior.

---

## 10. Stage 1A: Historical Metadata Sync

### 10.1 Goal

Fetch every historical LeetCode submission metadata row for the logged-in user.

Metadata includes:

- `submission_id`,
- problem slug,
- problem title,
- raw verdict/status,
- language,
- runtime display,
- memory display,
- timestamp.

Code is not fetched in Stage 1A.

### 10.2 LeetCode Source

Use:

```graphql
submissionList(offset: Int, limit: Int)
```

LeetCode limits page size to around 20, so the extension batches pages using GraphQL aliases.

### 10.3 Aliased Metadata Query Shape

```graphql
query bulkSubmissionPages {
  p0: submissionList(offset: 0, limit: 20) {
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
  p1: submissionList(offset: 20, limit: 20) {
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
}
```

Stage 1 uses 10 aliased pages per request:

```text
10 pages × 20 submissions = up to 200 metadata rows per request
```

### 10.4 Service Worker Algorithm

```ts
async function runMetadataSync() {
  while (true) {
    const state = await loadSyncState();
    const offset = state.cursor.offset ?? 0;

    const response = await leetcode.fetchSubmissionPages({
      startOffset: offset,
      pageSize: 20,
      pageCount: 10,
    });

    const rows = flattenSubmissionPages(response);
    const lastBatch = response.pages.some(p => !p.hasNext || p.submissions.length < 20);

    await backend.uploadSubmissionMetadata({
      platform: "leetcode",
      batch: sanitizeMetadataRows(rows),
      cursor_offset: offset,
      last_batch: lastBatch,
    });

    const allSubIdsByTs = updateLocalSubmissionIdIndex(rows);

    if (lastBatch) {
      const sorted = sortByTimestampDesc(allSubIdsByTs);
      await saveSyncState({
        phase: "B_fg",
        cursor: {
          foregroundCodeQueue: sorted.slice(0, 100),
          backgroundCodeQueue: sorted.slice(100),
          codeDone: 0,
          codeFailed: 0,
        },
      });
      break;
    }

    await saveSyncState({
      phase: "A_bulk",
      cursor: { offset: offset + 200 },
    });
  }
}
```

### 10.5 Backend Ingestion Behavior

Endpoint:

```http
POST /api/extension/platforms/leetcode/submissions
Authorization: Bearer <extension_access_token>
```

Backend steps:

1. Authenticate extension token.
2. Derive `app_user_id` from token subject.
3. Validate platform is `leetcode`.
4. Validate batch size and row shape.
5. Normalize verdict.
6. Parse timestamp.
7. Parse runtime and memory where possible.
8. Ensure connected profile exists.
9. Upsert stub catalog problem rows for unknown slugs.
10. Upsert rows into `user_data.user_coding_submissions` with `has_code=false`.
11. Recompute aggregates in `user_data.user_coding_problems` for affected slugs.
12. Publish catalog events for unknown/pending slugs.
13. Update `system.sync_state`.
14. Return success with counts.

### 10.6 Metadata Batch Payload

```json
{
  "platform": "leetcode",
  "batch": [
    {
      "submission_id": 1973994054,
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
  "cursor_offset": 0,
  "last_batch": false
}
```

### 10.7 Metadata Response

```json
{
  "ok": true,
  "ingested": 200,
  "updated_problem_aggregates": 74,
  "unknown_slugs_queued": 3,
  "next_phase": "A_bulk"
}
```

---

## 11. Stage 1B: Foreground Code Fetch

### 11.1 Goal

Fetch submitted source code for the latest 100 submissions immediately after metadata sync. This makes the user's recent code corpus available quickly and unlocks code-aware views for the most recent activity.

### 11.2 LeetCode Source

Use:

```graphql
submissionDetails(submissionId: Int!)
```

### 11.3 Aliased Code Query Shape

```graphql
query codeBatch {
  s0: submissionDetails(submissionId: 1973994054) {
    code
    runtime
    memory
    runtimePercentile
    memoryPercentile
    totalCorrect
    totalTestcases
    lastTestcase
    codeOutput
    expectedOutput
    runtimeError
    compileError
    lang { name verboseName }
    question { title titleSlug }
  }
  s1: submissionDetails(submissionId: 1973990919) {
    code
    runtime
    memory
    runtimePercentile
    memoryPercentile
    totalCorrect
    totalTestcases
    lastTestcase
    codeOutput
    expectedOutput
    runtimeError
    compileError
    lang { name verboseName }
    question { title titleSlug }
  }
}
```

Stage 1 fetches 10 code details per request.

### 11.4 Service Worker Algorithm

```ts
async function runForegroundCodeFetch() {
  let state = await loadSyncState();
  let queue = state.cursor.foregroundCodeQueue ?? [];

  while (queue.length > 0) {
    const batchIds = queue.slice(0, 10);

    try {
      const details = await leetcode.fetchSubmissionDetailsBatch(batchIds);

      await backend.uploadSubmissionCode({
        platform: "leetcode",
        capture_source: "historical_sync",
        code_capture_method: "server_only",
        batch: sanitizeCodeRows(details),
      });

      queue = queue.slice(10);
      await saveSyncState({
        phase: "B_fg",
        cursor: {
          ...state.cursor,
          foregroundCodeQueue: queue,
          codeDone: (state.cursor.codeDone ?? 0) + details.length,
        },
      });
    } catch (err) {
      await moveBatchToFailedCodeQueue(batchIds, err);
      queue = queue.slice(10);
    }
  }

  await saveSyncState({ phase: "B_bg" });
}
```

### 11.5 Backend Code Ingestion Behavior

Endpoint:

```http
POST /api/extension/platforms/leetcode/code
Authorization: Bearer <extension_access_token>
```

Backend steps:

1. Authenticate extension token.
2. Derive `app_user_id`.
3. Validate every submission ID belongs to this user/platform or exists in metadata table.
4. Normalize code language.
5. Compute SHA-256 code hash.
6. Upsert `code_blobs.user_coding_submission_code`.
7. Update `user_data.user_coding_submissions.has_code=true`.
8. Fill runtime/memory/testcase/error fields if detail payload has richer information.
9. Publish `submission.received` catalog event if problem statement is still missing.
10. Update `system.sync_state`.

### 11.6 Code Batch Payload

```json
{
  "platform": "leetcode",
  "capture_source": "historical_sync",
  "code_capture_method": "server_only",
  "batch": [
    {
      "submission_id": 1973994054,
      "slug": "two-sum",
      "code": "class Solution { ... }",
      "code_lang": "cpp",
      "lang_verbose": "C++",
      "runtime_ms": 4,
      "runtime_percentile": 91.2,
      "memory_kb": 12800,
      "memory_percentile": 72.5,
      "total_correct": 57,
      "total_testcases": 57,
      "last_testcase": null,
      "expected_output": null,
      "actual_output": null,
      "runtime_error": null,
      "compile_error": null,
      "raw_detail": {}
    }
  ]
}
```

---

## 12. Stage 1C: Background Code Backfill

### 12.1 Goal

Fetch all remaining historical code blobs after the latest 100 have completed. This runs without blocking the user.

### 12.2 Behavior

The service worker continues with `backgroundCodeQueue`:

1. Pull 10 submission IDs.
2. Fetch details through aliased `submissionDetails`.
3. Upload code batch.
4. Mark rows as `has_code=true`.
5. Save cursor.
6. Wait briefly between batches.
7. Retry failures using backoff.
8. Continue across service worker restarts.

### 12.3 Background Cadence

Recommended initial cadence:

```text
10 submissionDetails per request
1 request every 1-2 seconds during background backfill
exponential backoff on LeetCode/network errors
```

### 12.4 Background Completion

When `backgroundCodeQueue` is empty and `failedCodeQueue` is empty:

```json
{
  "phase": "live_ready",
  "cursor": {
    "backgroundCodeQueue": [],
    "failedCodeQueue": [],
    "codeDone": 2140,
    "codeFailed": 0
  }
}
```

Stage 1 stops here. `live_ready` means the data model is prepared for Stage 2 but Stage 2 behavior is not active yet.

---

## 13. Failed-Code Retry Queue

### 13.1 Purpose

Some code fetches can fail because of network issues, LeetCode throttling, malformed responses, auth expiration, or backend upload failures. These failures must not block Stage 1 metadata sync or core dashboard readiness.

### 13.2 Queue Shape

```ts
export type FailedCodeQueueItem = {
  platform: "leetcode";
  submission_id: number;
  slug?: string;
  reason: string;
  retry_count: number;
  next_retry_at: number;
  last_error?: string;
};
```

### 13.3 Retry Rules

| Retry count | Next action |
|---:|---|
| 0 | Retry after 10-30 seconds |
| 1 | Retry after 1-2 minutes |
| 2 | Retry after 5-10 minutes |
| 3+ | Keep persistent; retry on popup open, browser wake, or manual resume |

### 13.4 Backend Representation

The metadata row remains present:

```text
user_data.user_coding_submissions.has_code = false
```

This allows the UI to show complete attempt history even when code is temporarily missing.

---

## 14. Sync State Model

Sync state lives in two places:

| Location | Purpose |
|---|---|
| `chrome.storage.local` | Fast local resume and popup progress |
| `system.sync_state` | Backend visibility, dashboard progress, support/debugging |

### 14.1 Local Sync State

```ts
export type LocalSyncState = {
  appUserId: number;
  platform: "leetcode";
  phase:
    | "not_started"
    | "auth_checking"
    | "A_bulk"
    | "B_fg"
    | "B_bg"
    | "paused"
    | "auth_required"
    | "failed"
    | "live_ready";

  cursor: {
    offset?: number;
    foregroundCodeQueue?: number[];
    backgroundCodeQueue?: number[];
    failedCodeQueue?: FailedCodeQueueItem[];
    codeDone?: number;
    codeFailed?: number;
    metadataFetched?: number;
    totalSubmissions?: number;
  };

  progress: {
    metadataFetched: number;
    codeFetched: number;
    codeTotal: number;
    foregroundDone: boolean;
    backgroundDone: boolean;
  };

  pauseRequested: boolean;
  lastProgressAt: number;
  lastError?: string;
};
```

### 14.2 Backend Sync State

`system.sync_state` stores:

| Field | Meaning |
|---|---|
| `app_user_id` | CrackedIn SQLite user ID |
| `platform` | `leetcode` for Stage 1 |
| `phase` | Current sync phase |
| `cursor` | JSONB cursor summary |
| `last_progress_at` | Progress heartbeat |
| `last_full_sync_at` | Set when Stage 1 foreground sync completes |
| `retry_count` | Backend-tracked retry count |
| `last_error` | Last known error |

---

## 15. Supabase Postgres Schema

Stage 1 ships the six-table extension sync schema, adapted to `app_user_id INTEGER`.

### 15.1 Schemas

```sql
CREATE SCHEMA IF NOT EXISTS catalog;
CREATE SCHEMA IF NOT EXISTS user_data;
CREATE SCHEMA IF NOT EXISTS code_blobs;
CREATE SCHEMA IF NOT EXISTS system;
```

### 15.2 `catalog.coding_problems`

Shared problem catalog. Backend-owned. The extension never writes directly.

```sql
CREATE TABLE catalog.coding_problems (
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
```

### 15.3 `user_data.user_coding_profiles`

One connected coding account per CrackedIn user per platform.

```sql
CREATE TABLE user_data.user_coding_profiles (
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
```

### 15.4 `user_data.user_coding_problems`

Per-user-per-problem aggregate. This is the main dashboard read table for extension corpus state.

```sql
CREATE TABLE user_data.user_coding_problems (
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
```

### 15.5 `user_data.user_coding_submissions`

Every individual attempt. This is the atomic unit of the corpus.

```sql
CREATE TABLE user_data.user_coding_submissions (
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
```

### 15.6 `code_blobs.user_coding_submission_code`

Submitted source code only. It is separated from hot metadata to keep dashboard queries fast and make future archival easy.

```sql
CREATE TABLE code_blobs.user_coding_submission_code (
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
```

### 15.7 `system.sync_state`

Backend mirror of the extension sync cursor.

```sql
CREATE TABLE system.sync_state (
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

### 15.8 Required Indexes

```sql
CREATE INDEX idx_profiles_user ON user_data.user_coding_profiles(app_user_id);
CREATE INDEX idx_profiles_handle ON user_data.user_coding_profiles(platform, external_handle);

CREATE INDEX idx_user_problems_status ON user_data.user_coding_problems(app_user_id, status);
CREATE INDEX idx_user_problems_latest_attempt ON user_data.user_coding_problems(app_user_id, latest_attempt_at DESC);

CREATE INDEX idx_submissions_user_ts ON user_data.user_coding_submissions(app_user_id, ts DESC);
CREATE INDEX idx_submissions_user_slug ON user_data.user_coding_submissions(app_user_id, platform, slug, ts DESC);
CREATE INDEX idx_submissions_verdict ON user_data.user_coding_submissions(app_user_id, verdict, ts DESC);
CREATE INDEX idx_submissions_lang ON user_data.user_coding_submissions(app_user_id, lang);
CREATE INDEX idx_submissions_missing_code ON user_data.user_coding_submissions(app_user_id, platform, has_code) WHERE has_code = FALSE;

CREATE INDEX idx_code_storage_tier ON code_blobs.user_coding_submission_code(storage_tier) WHERE storage_tier = 'hot';
CREATE INDEX idx_coding_problems_sync_status ON catalog.coding_problems(sync_status) WHERE sync_status != 'complete';
```

---

## 16. Backend API Design

All Stage 1 extension endpoints live under:

```text
/api/extension
```

Platform routes use `{platform}` to remain future-ready.

### 16.1 Auth Endpoints

| Endpoint | Purpose |
|---|---|
| `POST /api/extension/auth/start` | Create one-time auth code from logged-in web session |
| `POST /api/extension/auth/exchange` | Exchange one-time code for extension tokens |
| `POST /api/extension/auth/refresh` | Refresh extension access token |
| `POST /api/extension/auth/revoke` | Revoke current extension session |
| `GET /api/extension/me` | Return current extension-authenticated user |

### 16.2 Stage 1 Sync Endpoints

| Endpoint | Purpose |
|---|---|
| `POST /api/extension/platforms/{platform}/connect` | Create/update connected coding profile |
| `POST /api/extension/platforms/{platform}/submissions` | Ingest historical metadata batch |
| `POST /api/extension/platforms/{platform}/code` | Ingest code/detail batch |
| `GET /api/extension/platforms/{platform}/sync-state` | Read backend sync state |
| `PATCH /api/extension/platforms/{platform}/sync-state` | Update backend sync state |
| `POST /api/extension/platforms/{platform}/pause` | Pause sync |
| `POST /api/extension/platforms/{platform}/resume` | Resume sync |

### 16.3 Future-Ready Endpoints Not Active in Stage 1

These routes may be added as stubs or reserved but are not Stage 1 active behavior:

| Endpoint | Future use |
|---|---|
| `POST /api/extension/platforms/{platform}/live` | Stage 2 live capture ingestion |
| `POST /api/extension/platforms/{platform}/delta` | Stage 3 delta repair ingestion |

---

## 17. Backend Service Design

### 17.1 `supabase_db.py`

Responsibilities:

- Create Supabase/Postgres connection.
- Provide transaction helper.
- Expose query execution helpers.
- Keep service-role secrets backend-only.

Environment variables:

```env
SUPABASE_DB_URL=
SUPABASE_PROJECT_URL=
SUPABASE_SERVICE_ROLE_KEY=
```

The extension must never receive Supabase service-role credentials.

### 17.2 `normalization.py`

Responsibilities:

- Normalize verdicts.
- Parse runtime.
- Parse memory.
- Normalize language names.
- Convert Unix timestamps to `TIMESTAMPTZ`.
- Compute code hashes.

Verdict normalization:

| Raw status | Normalized |
|---|---|
| `Accepted` | `AC` |
| `Wrong Answer` | `WA` |
| `Time Limit Exceeded` | `TLE` |
| `Memory Limit Exceeded` | `MLE` |
| `Runtime Error` | `RE` |
| `Compile Error` | `CE` |
| Any unknown status | `OTHER` |

### 17.3 `validators.py`

Responsibilities:

- Reject unsafe payload fields.
- Limit batch size.
- Validate required fields.
- Validate `platform`.
- Validate submission IDs.
- Validate timestamp shape.
- Ensure code payload matches known submission or creates safe metadata when needed.

Unsafe fields to reject or ignore:

- cookies,
- headers,
- CSRF tokens,
- browser storage dumps,
- unrelated HTML,
- unrelated network logs,
- passwords,
- raw extension debug dumps.

### 17.4 `profiles.py`

Responsibilities:

- Create/update `user_coding_profiles`.
- Detect account switch.
- Track connection state.
- Track `last_full_sync_at`, `last_sync_error`, `sync_enabled`.

### 17.5 `ingest_submissions.py`

Responsibilities:

- Upsert metadata rows.
- Stub-insert catalog problems.
- Recompute user solve graph.
- Publish catalog events.
- Update sync state.

### 17.6 `ingest_code.py`

Responsibilities:

- Upsert code blob rows.
- Compute code hash.
- Update `has_code`.
- Fill detail fields.
- Update sync state.

### 17.7 `sync_state.py`

Responsibilities:

- Read/write `system.sync_state`.
- Merge cursor updates safely.
- Mark paused/auth-required/failed/live-ready states.
- Store backend-visible progress.

### 17.8 `catalog_events.py` and `catalog_worker.py`

Responsibilities:

- Publish `submission.received` events.
- Consume catalog queue.
- Fetch missing LeetCode problem details.
- Convert problem HTML to markdown.
- Update `catalog.coding_problems`.

---

## 18. Catalog Worker Design

Stage 1 includes one backend worker: the catalog worker.

### 18.1 Event Shape

```json
{
  "type": "submission.received",
  "app_user_id": 123,
  "platform": "leetcode",
  "slug": "two-sum",
  "submission_id": 1973994054
}
```

### 18.2 Worker Behavior

```text
1. Receive submission.received event.
2. Check catalog.coding_problems for platform+slug.
3. If statement_md exists and sync_status='complete', do nothing.
4. If missing/pending, fetch LeetCode question(titleSlug).
5. Convert HTML content to markdown.
6. Extract constraints/examples/hints/topics where available.
7. Update catalog.coding_problems.
8. Mark sync_status='complete'.
9. On repeated failure, mark sync_status='failed' and store raw error.
```

### 18.3 Stage 1 Queue Choice

Use Supabase Postgres/pgmq for catalog events. This keeps Stage 1 infrastructure inside Supabase and avoids adding another queue service.

---

## 19. Extension Popup Design

The popup is not a sync runner. It is a UI controller and progress viewer.

### 19.1 Popup States

| State | UI behavior |
|---|---|
| Not connected | Show `Connect CrackedIn` |
| CrackedIn connected, LeetCode unknown | Check LeetCode login |
| LeetCode logged out | Show `Open LeetCode and log in` |
| LeetCode detected | Show username and `Start Sync` |
| `A_bulk` | Show metadata progress |
| `B_fg` | Show recent code fetch progress |
| `B_bg` | Show core sync complete and background progress |
| Paused | Show resume button |
| Failed | Show error and retry button |
| Live ready | Show sync complete |

### 19.2 Popup Copy

| Phase | Message |
|---|---|
| Auth checking | `Checking your CrackedIn and LeetCode connection...` |
| LeetCode required | `Open LeetCode and log in to continue.` |
| Ready | `Detected LeetCode account: {username}` |
| Metadata sync | `Finding your LeetCode submissions... {count} found` |
| Foreground code | `Fetching your latest submitted code... {done}/100` |
| Background code | `Core sync complete. Backfilling older code in the background...` |
| Complete | `Historical sync complete.` |
| Paused | `Sync paused. You can resume anytime.` |
| Failed | `Sync paused because of an error. Retry is available.` |

### 19.3 Progress Calculation

During metadata sync, total is unknown until the end. Use found-so-far count:

```text
Finding your submissions... 1,200 found
```

During code sync, total is known:

```text
Fetching submitted code... 40 / 100
```

During background backfill:

```text
Background code backfill: 420 / 2,140
```

---

## 20. LeetCode Platform Adapter

Stage 1 implements only the LeetCode adapter.

### 20.1 Extension Adapter Interface

```ts
export interface PlatformAdapter {
  platform: "leetcode" | "codeforces" | "gfg";

  checkAuth(): Promise<PlatformAuthStatus>;
  getExternalProfile(): Promise<ExternalProfile>;

  fetchMetadataBatch(cursor: SyncCursor): Promise<MetadataBatch>;
  fetchCodeBatch(submissionIds: number[]): Promise<CodeBatch>;
}
```

### 20.2 LeetCode Adapter Responsibilities

| Function | Responsibility |
|---|---|
| `checkAuth()` | Calls `userStatus` through LeetCode GraphQL |
| `getExternalProfile()` | Returns username/user ID/premium status |
| `fetchMetadataBatch()` | Calls aliased `submissionList` pages |
| `fetchCodeBatch()` | Calls aliased `submissionDetails` |
| `sanitizeMetadataRows()` | Keeps only required metadata fields |
| `sanitizeCodeRows()` | Keeps only code/detail fields needed by backend |

### 20.3 LeetCode Auth Query

```graphql
query userStatus {
  userStatus {
    isSignedIn
    username
    userId
    isPremium
  }
}
```

---

## 21. Backend Platform Adapter Design

The backend should also be platform-aware.

```python
class PlatformIngestAdapter:
    platform: str

    def normalize_submission(self, raw: dict) -> dict:
        ...

    def normalize_code(self, raw: dict) -> dict:
        ...

    def normalize_verdict(self, raw_status: str) -> str:
        ...
```

Stage 1 implements:

```text
api/services/extension_sync/platforms/leetcode.py
```

Future platforms plug into the same ingestion flow without changing the database shape.

---

## 22. Data Contracts

### 22.1 Connect Profile Request

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

### 22.2 Connect Profile Response

```json
{
  "ok": true,
  "profile": {
    "platform": "leetcode",
    "external_handle": "leetcode_user",
    "sync_enabled": true,
    "is_connected": true
  }
}
```

### 22.3 Metadata Ingest Request

```json
{
  "platform": "leetcode",
  "batch": [
    {
      "submission_id": 1973994054,
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
  "cursor_offset": 0,
  "last_batch": false
}
```

### 22.4 Metadata Ingest Response

```json
{
  "ok": true,
  "ingested": 200,
  "duplicates": 0,
  "affected_slugs": 74,
  "unknown_slugs_queued": 3
}
```

### 22.5 Code Ingest Request

```json
{
  "platform": "leetcode",
  "capture_source": "historical_sync",
  "code_capture_method": "server_only",
  "batch": [
    {
      "submission_id": 1973994054,
      "slug": "two-sum",
      "code": "class Solution { ... }",
      "code_lang": "cpp",
      "lang_verbose": "C++",
      "runtime_ms": 4,
      "runtime_percentile": 91.2,
      "memory_kb": 12800,
      "memory_percentile": 72.5,
      "total_correct": 57,
      "total_testcases": 57,
      "last_testcase": null,
      "expected_output": null,
      "actual_output": null,
      "runtime_error": null,
      "compile_error": null,
      "raw_detail": {}
    }
  ]
}
```

### 22.6 Code Ingest Response

```json
{
  "ok": true,
  "code_rows_upserted": 10,
  "submissions_marked_has_code": 10,
  "unknown_slugs_queued": 0
}
```

---

## 23. Aggregation Logic for `user_coding_problems`

After each metadata or code batch, backend updates affected problem aggregates.

For each `(app_user_id, platform, slug)`:

| Field | Computation |
|---|---|
| `status` | `ac` if any AC exists, else `tried` |
| `first_attempt_at` | Minimum submission timestamp |
| `latest_attempt_at` | Maximum submission timestamp |
| `first_ac_at` | Minimum AC timestamp |
| `latest_ac_at` | Maximum AC timestamp |
| `attempt_count` | Count all submissions |
| `ac_count` | Count verdict `AC` |
| `wa_count` | Count verdict `WA` |
| `tle_count` | Count verdict `TLE` |
| `mle_count` | Count verdict `MLE` |
| `re_count` | Count verdict `RE` |
| `ce_count` | Count verdict `CE` |
| `other_count` | Count verdict `OTHER` |
| `best_runtime_ms` | Best AC runtime where available |
| `best_memory_kb` | Best AC memory where available |
| `best_lang` | Language of selected best/latest AC |
| `primary_submission_id` | Latest AC submission by default; latest attempt if no AC |

This table is the dashboard's primary read table and avoids expensive group-by queries over the submission table.

---

## 24. Idempotency and Deduplication

Every ingestion endpoint must be safe to replay.

### 24.1 Submission Idempotency

Primary key:

```text
(platform, submission_id)
```

If the same submission appears again through replay, retry, background backfill, later live capture, or future delta repair, the backend updates the existing row instead of creating a duplicate.

### 24.2 Code Idempotency

Primary key:

```text
(platform, submission_id)
```

If the same code blob is uploaded again, backend updates or no-ops based on code hash.

### 24.3 Sync Cursor Replay Safety

If the service worker dies mid-batch:

1. It resumes from the last persisted cursor.
2. It may replay the last batch.
3. Backend upsert prevents duplication.
4. Aggregates are recomputed from canonical rows.

---

## 25. Error Handling

### 25.1 LeetCode Auth Expired

Behavior:

1. LeetCode request fails as unauthenticated.
2. Service worker sets phase `auth_required`.
3. Popup shows `Open LeetCode and log in to continue`.
4. After user logs in, sync resumes from cursor.

### 25.2 Backend Unavailable

Behavior:

1. Service worker stores pending batch locally.
2. Phase becomes `failed` or retrying state.
3. Retry with exponential backoff.
4. Popup shows retryable error.

### 25.3 LeetCode Rate Limiting

Behavior:

1. Slow down batch cadence.
2. Use exponential backoff.
3. Keep cursor unchanged for failed batch.
4. Resume later.

### 25.4 Malformed LeetCode Response

Behavior:

1. Record error on affected batch.
2. Move affected submission IDs to failed-code queue if metadata already exists.
3. Continue with remaining batches if safe.
4. Store `last_error` in sync state.

### 25.5 Unknown Problem Slug

Behavior:

1. Backend stub-inserts catalog row with `sync_status='pending'`.
2. Metadata ingest succeeds.
3. Catalog worker fetches statement later.

---

## 26. Security Requirements

### 26.1 Extension Must Never Upload

- LeetCode session cookie,
- LeetCode password,
- CrackedIn password,
- CSRF token,
- raw browser headers,
- unrelated browsing history,
- unrelated page content,
- local/session storage dumps,
- full raw network logs.

### 26.2 Backend Must Enforce

- Authenticate every `/api/extension/*` request.
- Derive `app_user_id` from token, not request body.
- Validate platform access.
- Reject unsafe fields.
- Limit batch sizes.
- Enforce ownership on profile and submissions.
- Use idempotent upserts.
- Keep Supabase service credentials server-side only.

### 26.3 Data Sensitivity

Submitted code is sensitive user data. Stage 1 stores code in Postgres `code_blobs` with future archival columns. User-facing controls must include pause, disconnect, and delete imported data.

---

## 27. Stage 2 and Stage 3 Readiness

Stage 1 does not implement Stage 2 or Stage 3, but the design must be ready for them.

### 27.1 Stage 2 Readiness: Live Capture

Stage 1 prepares for live capture through:

| Prepared item | How Stage 1 supports it |
|---|---|
| `capture_source` column | Can store `live_capture` later |
| `code_capture_method` column | Can store `client_then_server` later |
| Idempotent submission key | Live capture can upsert same submission safely |
| Platform adapter structure | Content-script events can route through same backend ingestion |
| `/live` route reservation | Endpoint can be added without changing URL strategy |
| `last_live_capture_at` field | Profile table already has field |

Stage 2 will later add content scripts that observe LeetCode submission events, extract submission IDs, and ask the service worker to fetch official `submissionDetails`.

### 27.2 Stage 3 Readiness: Delta Repair

Stage 1 prepares for delta repair through:

| Prepared item | How Stage 1 supports it |
|---|---|
| `capture_source` column | Can store `delta_repair` later |
| idempotent upserts | Missed submissions can be inserted or updated safely |
| `last_delta_sync_at` field | Profile table already has field |
| `sync_state` cursor | Delta state can be added to existing JSONB cursor |
| missing-code index | Delta can find rows with `has_code=false` |
| platform route shape | `/platforms/{platform}/delta` can be added later |

Stage 3 will later run periodic checks against recent accepted submissions and recent general submission pages to recover missed activity.

---

## 28. Implementation Plan

### 28.1 Backend Implementation Order

| Step | Task |
|---:|---|
| 1 | Add SQLite tables for extension auth codes and extension sessions |
| 2 | Add Supabase env vars and Postgres client |
| 3 | Add Supabase migration for the six extension tables |
| 4 | Add `api/routes/extension_sync.py` router |
| 5 | Add extension auth start/exchange/refresh/revoke endpoints |
| 6 | Add `/api/extension/me` endpoint |
| 7 | Add profile connect endpoint |
| 8 | Add metadata ingest endpoint |
| 9 | Add code ingest endpoint |
| 10 | Add sync-state read/update endpoints |
| 11 | Add normalization and validation services |
| 12 | Add user problem aggregate recomputation |
| 13 | Add catalog event publisher |
| 14 | Add catalog worker |
| 15 | Add pause/resume endpoints |
| 16 | Add structured logs and sync error tracking |

### 28.2 Extension Implementation Order

| Step | Task |
|---:|---|
| 1 | Add `extension/` project and Manifest V3 config |
| 2 | Add popup shell |
| 3 | Add extension auth storage and API client |
| 4 | Add CrackedIn connection flow |
| 5 | Add LeetCode `userStatus` auth check |
| 6 | Add LeetCode profile confirmation UI |
| 7 | Add Stage 1 local sync state model |
| 8 | Implement Stage 1A metadata sync with aliasing |
| 9 | Implement backend metadata upload |
| 10 | Implement progress UI for metadata sync |
| 11 | Implement Stage 1B latest-100 code fetch |
| 12 | Implement backend code upload |
| 13 | Implement Stage 1C background backfill |
| 14 | Implement failed-code retry queue |
| 15 | Implement pause/resume |
| 16 | Implement browser wake/alarm resume |
| 17 | Add extension tests and manual QA scripts |

---

## 29. Testing Plan

### 29.1 Backend Tests

| Test area | Required tests |
|---|---|
| Auth exchange | One-time code exchange succeeds once and expires |
| Token refresh | Refresh token returns new access token |
| Ownership | Request body cannot override `app_user_id` |
| Profile connect | Creates and updates LeetCode profile |
| Metadata ingest | Inserts new submissions and updates duplicates |
| Code ingest | Inserts code blobs and sets `has_code=true` |
| Aggregates | `user_coding_problems` counts are correct |
| Catalog events | Unknown slug queues event exactly once or idempotently |
| Validation | Unsafe fields rejected or ignored |
| Sync state | Cursor updates are persisted correctly |
| Pause/resume | Sync-enabled flag changes correctly |

### 29.2 Extension Tests

| Test area | Required tests |
|---|---|
| Auth storage | Tokens stored and cleared correctly |
| LeetCode auth | Logged-in and logged-out states detected |
| Metadata aliasing | Correct offsets and response flattening |
| Code aliasing | Correct submission ID mapping to response aliases |
| Cursor persistence | Cursor saved after every batch |
| Replay safety | Re-running same batch does not break progress |
| Popup progress | Popup reads state after reopening |
| Service worker wake | Resumes from saved state |
| Retry queue | Failed code IDs move to retry queue |
| Pause/resume | Stops between batches and resumes correctly |

### 29.3 Manual QA Scenarios

| Scenario | Expected result |
|---|---|
| User has zero submissions | Metadata sync completes, code phase skipped |
| User has less than 100 submissions | Foreground queue includes all submissions, background queue empty |
| User has thousands of submissions | Metadata paginates fully and backfill continues |
| Popup closes during sync | Sync continues or resumes; popup shows latest state when reopened |
| Browser closes during sync | Sync resumes after browser restarts |
| User logs out of LeetCode mid-sync | Phase becomes `auth_required` |
| Backend goes down | Pending work remains retryable |
| Duplicate batch replay | No duplicate rows |
| Unknown problem slug | Catalog stub row created and worker fills it later |

---

## 30. Observability and Logging

Stage 1 must log enough to debug sync failures without logging sensitive data.

### 30.1 Extension Logs

Local logs may include:

- phase,
- batch offset,
- count fetched,
- count uploaded,
- retry count,
- error category.

Local logs must not include:

- cookies,
- tokens,
- full code bodies,
- raw headers.

### 30.2 Backend Logs

Backend logs should include:

- `app_user_id`,
- platform,
- endpoint,
- batch size,
- ingested count,
- duplicate count,
- failed count,
- duration,
- error category.

Backend logs should not include full code bodies unless explicitly enabled in local-only debug mode.

### 30.3 Sync Metrics

Track:

| Metric | Purpose |
|---|---|
| metadata rows ingested | Sync progress |
| code blobs ingested | Code backfill progress |
| failed code fetches | Reliability |
| average batch latency | Performance |
| LeetCode auth failures | User support |
| backend validation failures | Bug/security detection |
| catalog worker failures | Catalog quality |

---

## 31. Deployment Requirements

### 31.1 Backend Environment

Add:

```env
SUPABASE_DB_URL=
SUPABASE_PROJECT_URL=
SUPABASE_SERVICE_ROLE_KEY=
EXTENSION_AUTH_CODE_TTL_SECONDS=300
EXTENSION_ACCESS_TOKEN_MINUTES=60
EXTENSION_REFRESH_TOKEN_DAYS=60
```

### 31.2 Supabase Setup

1. Create schemas.
2. Apply six-table migration.
3. Enable pgmq or equivalent Postgres queue setup.
4. Run LeetCode catalog seed script.
5. Verify indexes.
6. Configure backups.

### 31.3 Extension Build

1. Build extension assets.
2. Load unpacked extension in Chrome for dev.
3. Verify host permissions.
4. Verify LeetCode GraphQL calls work from service worker.
5. Verify backend auth and upload.
6. Package for Chrome Web Store later.

---

## 32. Stage 1 Acceptance Criteria

Stage 1 is complete when:

1. A user can connect the extension to their CrackedIn account.
2. The extension can detect LeetCode login state.
3. The user can confirm the detected LeetCode account.
4. The extension can fetch all historical submission metadata.
5. Backend stores all metadata in Supabase idempotently.
6. `user_coding_problems` aggregate rows are created correctly.
7. The extension fetches code for the latest 100 submissions in foreground.
8. Backend stores code in `code_blobs.user_coding_submission_code`.
9. Remaining code blobs backfill in the background.
10. Failed code fetches enter a persistent retry queue.
11. Sync can resume after popup close, service worker stop, or browser restart.
12. No LeetCode cookies or credentials are uploaded.
13. Unknown problem slugs create catalog events.
14. Catalog worker fills missing problem metadata.
15. Backend and extension state agree on sync phase and progress.
16. The data model is ready for Stage 2 live capture and Stage 3 delta repair.

---

## 33. Final Stage 1 Architecture Summary

CrackedIn Stage 1 extension sync will be implemented as follows:

```text
Existing CrackedIn app remains SQLite-first.

The Chrome extension is added as a separate client inside the current repo.

The extension authenticates through CrackedIn using extension-scoped tokens.

The extension uses the user's browser session to call LeetCode GraphQL.

The extension never reads or uploads the LeetCode session cookie.

One Manifest V3 service worker runs all Stage 1 jobs:
  - metadata sync,
  - latest-100 code fetch,
  - background code backfill,
  - retry queue,
  - progress persistence.

The extension sends sanitized batches to FastAPI.

FastAPI validates, normalizes, deduplicates, and writes to Supabase.

Supabase stores the six-table extension corpus schema using app_user_id from the existing SQLite user.

The catalog worker fills missing problem metadata asynchronously.

Stage 1 stops after historical sync and background code backfill readiness.

The schema and APIs remain ready for Stage 2 live capture and Stage 3 delta repair.
```

This design is the implementation baseline for Stage 1.
