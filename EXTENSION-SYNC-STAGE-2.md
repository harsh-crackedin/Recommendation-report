# Extension Sync — Stage 2 Live Capture Design Report

**Project:** CrackedIn / LeetCode Access  
**Stage:** Stage 2 — Live LeetCode Submission Capture  
**Status:** Final design report  
**Depends on:** Stage 1 first-time historical sync implementation  
**Primary goal:** Capture new LeetCode submissions made after the user has connected and synced their account.

---

## 1. Purpose of Stage 2

Stage 1 imports the user's historical LeetCode submission corpus. Stage 2 keeps that corpus updated in near real time when the user submits new code on LeetCode.

Stage 2 does not replace Stage 1. It extends the existing system by adding a live capture path:

```text
User submits code on LeetCode
  ↓
Content script detects the LeetCode submission ID
  ↓
Service worker fetches official submission details from LeetCode GraphQL
  ↓
Service worker uploads sanitized live-submission data to CrackedIn backend
  ↓
Backend validates, normalizes, and upserts into existing Stage 1 tables
  ↓
User's coding corpus and solve graph are updated
```

The final design keeps the same core trust model:

- The extension uses the user's existing browser LeetCode session.
- The extension does not upload LeetCode cookies.
- The extension does not store LeetCode passwords.
- The backend derives user identity from the CrackedIn HttpOnly extension cookie.
- The backend does not trust user IDs sent from the client.

---

## 2. Stage 2 Final Strategy

The chosen Stage 2 strategy is:

> Detect the new submission ID from the LeetCode page using the content script, then fetch canonical submission code and metadata through the service worker using `submissionDetails(submissionId)`.

The content script is only a detector. It should not be the canonical source of submitted code.

The service worker is the official fetcher and uploader.

The backend remains the canonical validator, normalizer, deduplicator, and database writer.

---

## 3. Existing Stage 1 Implementation Context

Stage 1 already provides most of the required foundation.

### 3.1 Extension files already present

| Purpose | Existing file |
|---|---|
| Manifest | `extension/manifest.json` |
| Service worker | `extension/src/service-worker/index.ts` |
| Popup | `extension/src/popup/popup.html` |
| Stage 2 placeholder content script | `extension/src/content-scripts/leetcodeLivePlaceholder.ts` |
| Shared constants | `extension/src/shared/constants.ts` |
| Shared types | `extension/src/shared/types.ts` |
| Shared storage helpers | `extension/src/shared/storage.ts` |
| LeetCode API helpers | `extension/src/platforms/leetcode.ts` |
| CrackedIn backend API helpers | `extension/src/api/crackedinApi.ts` |

### 3.2 Existing extension registration

The content script already exists and is registered for:

```text
https://leetcode.com/problems/*
```

Stage 2 should add logic to the existing placeholder file:

```text
extension/src/content-scripts/leetcodeLivePlaceholder.ts
```

Do not create a separate competing content script unless the manifest structure later requires splitting files.

### 3.3 Existing LeetCode GraphQL helper

Stage 1 already has:

```ts
leetcodeGraphQL<T>(query: string)
```

Location:

```text
extension/src/platforms/leetcode.ts
```

It already uses:

```ts
credentials: "include"
```

Stage 2 must reuse this helper instead of creating a second LeetCode request system.

### 3.4 Existing submission details helper

Stage 1 already has:

```ts
fetchSubmissionDetails(submissionIds: number[])
```

Location:

```text
extension/src/platforms/leetcode.ts
```

Stage 2 should call it with one submission ID:

```ts
await fetchSubmissionDetails([submissionId])
```

This keeps Stage 2 aligned with the existing GraphQL aliasing/batching implementation.

### 3.5 Existing backend endpoints

Stage 1 currently uses:

```text
POST  /api/extension/platforms/leetcode/connect
POST  /api/extension/platforms/leetcode/submissions
POST  /api/extension/platforms/leetcode/code
PATCH /api/extension/platforms/leetcode/sync-state
```

Stage 2 will add one dedicated route:

```text
POST /api/extension/platforms/leetcode/live-submission
```

Internally, this route must reuse or wrap the same ingestion logic used by Stage 1.

### 3.6 Existing backend files

| Purpose | Existing file/function |
|---|---|
| Extension routes | `api/routes/extension_sync.py` |
| Extension user dependency | `get_current_extension_pg_user_id` from `cookie_auth.py` |
| Metadata ingestion | `ingest_metadata_batch` in `api/services/extension_sync/repository.py` |
| Code ingestion | `ingest_code_batch` in `api/services/extension_sync/repository.py` |
| Aggregate recomputation | `recompute_aggregates(...)` |
| Catalog queue publisher | `_publish_catalog_event(...)` |

Stage 2 should add a new repository function:

```py
ingest_live_submission(...)
```

or extract shared lower-level helpers from Stage 1 ingestion and call them from the new live ingestion wrapper.

---

## 4. Stage 2 Architecture

## 4.1 High-level system diagram

```text
LeetCode Problem Page
  ↓
Content Script
  extension/src/content-scripts/leetcodeLivePlaceholder.ts
  ↓
Chrome Runtime Message
  LEETCODE_SUBMISSION_DETECTED
  ↓
Service Worker
  extension/src/service-worker/index.ts
  ↓
LeetCode GraphQL
  submissionDetails(submissionId)
  ↓
CrackedIn API Client
  extension/src/api/crackedinApi.ts
  ↓
FastAPI Backend
  POST /api/extension/platforms/leetcode/live-submission
  ↓
Repository Layer
  ingest_live_submission(...)
  ↓
Supabase Postgres
  user_data.user_coding_submissions
  code_blobs.user_coding_submission_code
  user_data.user_coding_problems
  user_data.user_coding_profiles
  system.sync_state
  catalog.coding_problems
  extension_events queue
```

---

## 5. Component Responsibilities

## 5.1 Content script responsibilities

File:

```text
extension/src/content-scripts/leetcodeLivePlaceholder.ts
```

The content script is responsible for:

- running on LeetCode problem pages,
- detecting LeetCode submission-check network requests,
- extracting the `submissionId`,
- sending the submission ID to the service worker,
- avoiding disruption to the LeetCode page.

The content script must not:

- upload data directly to the backend,
- fetch LeetCode GraphQL details,
- read CrackedIn auth cookies,
- read LeetCode cookies,
- write to Supabase,
- store long-term sync state,
- decide canonical code.

## 5.2 Service worker responsibilities

File:

```text
extension/src/service-worker/index.ts
```

The service worker is responsible for:

- receiving `LEETCODE_SUBMISSION_DETECTED`,
- deduplicating repeated live events,
- checking whether sync/live capture is allowed,
- fetching official LeetCode details with `fetchSubmissionDetails([submissionId])`,
- sanitizing the live payload,
- uploading to the backend through `crackedinApi.ts`,
- storing retry state if fetch/upload fails,
- updating sync state where needed.

The service worker must remain the only extension component that performs live submission upload.

## 5.3 Backend responsibilities

Files:

```text
api/routes/extension_sync.py
api/services/extension_sync/repository.py
```

The backend is responsible for:

- authenticating the current user from the HttpOnly extension cookie,
- validating the connected LeetCode profile,
- validating the live-submission payload,
- normalizing verdict/runtime/memory/language fields,
- upserting `user_data.user_coding_submissions`,
- upserting `code_blobs.user_coding_submission_code`,
- setting `has_code = true`,
- setting `capture_source = 'live_capture'`,
- setting `code_capture_method = 'server_only'`,
- recomputing the affected `user_data.user_coding_problems` aggregate,
- updating `user_data.user_coding_profiles.last_live_capture_at`,
- publishing a `submission.received` event to `extension_events` if catalog data is missing.

The backend must not trust `user_id` from the request body.

## 5.4 Database responsibilities

No new Stage 2 tables are required.

Stage 2 uses the existing Stage 1 schema:

| Table | Stage 2 use |
|---|---|
| `user_data.user_coding_profiles` | update `last_live_capture_at` |
| `user_data.user_coding_submissions` | insert/update live submission metadata |
| `code_blobs.user_coding_submission_code` | store official submitted code |
| `user_data.user_coding_problems` | update per-problem aggregate counts |
| `system.sync_state` | track live/retry state if needed |
| `catalog.coding_problems` | stub/fill problem metadata if needed |
| `extension_events` PGMQ queue | queue missing catalog metadata |

---

## 6. How the Submission ID Is Captured

When a user submits code on LeetCode, LeetCode starts polling a check endpoint shaped like:

```text
/submissions/detail/<submissionId>/check/
```

Example:

```text
https://leetcode.com/submissions/detail/1973994054/check/
```

The content script extracts:

```text
1973994054
```

That value is the LeetCode `submissionId`.

The content script sends only this ID to the service worker:

```ts
chrome.runtime.sendMessage({
  type: "LEETCODE_SUBMISSION_DETECTED",
  submissionId: 1973994054,
  source: "leetcode_check_poll",
  detectedAt: Date.now(),
});
```

The service worker then fetches official details.

---

## 7. Content Script Design

## 7.1 Detection method

The content script should intercept or observe LeetCode network activity and look for URLs matching:

```regex
\/submissions\/detail\/(\d+)\/check\/
```

The first implementation should patch `window.fetch` and also support `XMLHttpRequest` if LeetCode uses XHR in the current frontend path.

## 7.2 Fetch interception behavior

The content script must:

1. call the original fetch first,
2. clone the response,
3. inspect the URL,
4. extract `submissionId`,
5. optionally inspect response JSON to check for final state,
6. send a runtime message to the service worker,
7. return the original response unchanged.

The content script must never block or alter LeetCode behavior.

## 7.3 Message payload

Recommended message:

```ts
type LeetCodeSubmissionDetectedMessage = {
  type: "LEETCODE_SUBMISSION_DETECTED";
  submissionId: number;
  source: "leetcode_check_poll";
  detectedAt: number;
};
```

The content script may also include the current problem slug if easily available from `window.location.pathname`, but the service worker should still trust `submissionDetails` as the canonical source.

Optional field:

```ts
problemSlug?: string;
```

## 7.4 Content script safety rules

The content script should wrap all custom logic in `try/catch`.

Any failure should be logged at most as a warning and should not break LeetCode.

Recommended behavior:

```ts
try {
  // capture logic
} catch (error) {
  console.warn("CrackedIn live capture detection failed", error);
}
```

---

## 8. Service Worker Design

## 8.1 Add new message type

File:

```text
extension/src/shared/types.ts
```

Add:

```ts
export type LiveSubmissionDetectedMessage = {
  type: "LEETCODE_SUBMISSION_DETECTED";
  submissionId: number;
  source: "leetcode_check_poll";
  detectedAt: number;
  problemSlug?: string;
};
```

Add it to the existing `WorkerMessage` union.

## 8.2 Add live-capable phase

Current phases are:

```ts
"idle" | "auth_checking" | "A_bulk" | "B_fg" | "B_bg" | "paused" | "auth_required" | "failed" | "done"
```

Add:

```ts
"live"
```

Recommended final phase type:

```ts
export type SyncPhase =
  | "idle"
  | "auth_checking"
  | "A_bulk"
  | "B_fg"
  | "B_bg"
  | "live"
  | "paused"
  | "auth_required"
  | "failed"
  | "done";
```

Stage 2 should treat both `B_bg` and `live` as live-capable. This allows live capture while background backfill is still running.

Recommended rule:

```text
Live capture is allowed when phase is B_bg, live, or done,
and sync_enabled/is_connected are true.
```

## 8.3 Add service worker handler

File:

```text
extension/src/service-worker/index.ts
```

Add handling inside the existing `chrome.runtime.onMessage.addListener`:

```ts
if (message.type === "LEETCODE_SUBMISSION_DETECTED") {
  void handleLiveSubmissionDetected(message.submissionId, message);
  sendResponse({ ok: true });
  return true;
}
```

## 8.4 Live handler function

Recommended file:

```text
extension/src/service-worker/liveCapture.ts
```

If the service worker is still small, this can initially live inside:

```text
extension/src/service-worker/index.ts
```

Recommended function:

```ts
export async function handleLiveSubmissionDetected(
  submissionId: number,
  context: LiveSubmissionDetectedMessage
): Promise<void> {
  // 1. Dedupe repeated check polling
  // 2. Load local sync state
  // 3. Verify live capture is allowed
  // 4. Fetch official submission details
  // 5. Build sanitized live payload
  // 6. Upload to CrackedIn backend
  // 7. Mark ID as captured
  // 8. Store retry state if needed
}
```

## 8.5 Reuse Stage 1 detail fetch

Use:

```ts
const details = await fetchSubmissionDetails([submissionId]);
```

Expected behavior:

- one submission ID in,
- one official detail result out,
- code and metadata come from LeetCode GraphQL,
- no separate GraphQL implementation is needed.

## 8.6 Live upload API helper

File:

```text
extension/src/api/crackedinApi.ts
```

Add:

```ts
export async function uploadLiveSubmission(payload: LiveSubmissionPayload) {
  return fetch(`${API_BASE}/api/extension/platforms/leetcode/live-submission`, {
    method: "POST",
    credentials: "include",
    headers: {
      "Content-Type": "application/json",
    },
    body: JSON.stringify(payload),
  });
}
```

The request must use:

```ts
credentials: "include"
```

The extension should not store or attach JWT tokens manually.

---

## 9. Local Dedupe and Retry Design

## 9.1 Why dedupe is required

LeetCode may poll the check endpoint multiple times for the same submission ID.

Without dedupe, the service worker may repeatedly fetch and upload the same submission.

The backend upsert protects the database, but local dedupe avoids unnecessary work.

## 9.2 Local dedupe storage

Store recent live submission IDs in `chrome.storage.local`.

Recommended structure inside the existing sync state cursor or a related local key:

```json
{
  "recentLiveSubmissionIds": {
    "leetcode:1973994054": 1717050000
  }
}
```

Retention:

```text
24 hours
```

If the same submission ID is detected again within the retention window and it was successfully uploaded, ignore it.

## 9.3 Retry queue

If official detail fetch or backend upload fails, store a retry entry.

Recommended cursor field:

```json
{
  "liveRetryQueue": [
    {
      "submission_id": 1973994054,
      "reason": "backend_upload_failed",
      "retry_count": 1,
      "next_retry_at": 1717050300,
      "last_error": "HTTP 503"
    }
  ]
}
```

Retry rules:

- retry on temporary LeetCode failures,
- retry on backend 5xx/network failures,
- do not retry forever without backoff,
- keep the submission ID even if details must be refetched later,
- if backend upload fails after details are fetched, either keep sanitized details if small or refetch by ID during retry.

Recommended backoff:

```text
10 seconds → 30 seconds → 60 seconds → 5 minutes
```

---

## 10. Backend API Design

## 10.1 New route

File:

```text
api/routes/extension_sync.py
```

Add:

```text
POST /api/extension/platforms/leetcode/live-submission
```

The route should:

1. resolve the authenticated user using `get_current_extension_pg_user_id`,
2. validate `platform == "leetcode"`,
3. validate request payload shape,
4. call `ingest_live_submission(...)`,
5. return a simple success response.

Recommended response:

```json
{
  "ok": true,
  "submission_id": 1973994054,
  "capture_source": "live_capture"
}
```

## 10.2 Request payload

Recommended payload:

```json
{
  "platform": "leetcode",
  "submission": {
    "submission_id": 1973994054,
    "slug": "two-sum",
    "title": "Two Sum",
    "verdict": "AC",
    "raw_status": "Accepted",
    "status_code": null,
    "lang": "cpp",
    "lang_verbose": "C++",
    "runtime_ms": 4,
    "runtime_percentile": 92.41,
    "memory_kb": 12300,
    "memory_percentile": 81.22,
    "total_correct": 57,
    "total_testcases": 57,
    "last_testcase": null,
    "expected_output": null,
    "actual_output": null,
    "runtime_error": null,
    "compile_error": null,
    "ts": "2026-05-30T10:00:00Z",
    "raw_meta": {}
  },
  "code": {
    "code": "class Solution { ... }",
    "code_lang": "cpp",
    "code_hash": "sha256-value"
  },
  "capture": {
    "capture_source": "live_capture",
    "code_capture_method": "server_only",
    "detected_at": "2026-05-30T10:00:05Z"
  }
}
```

The backend should derive `user_id` from the cookie, not from this payload.

## 10.3 Backend service function

File:

```text
api/services/extension_sync/repository.py
```

Add:

```py
async def ingest_live_submission(conn, user_id, platform, payload):
    async with conn.transaction():
        # 1. Resolve or verify connected coding profile
        # 2. Upsert catalog.coding_problems stub if needed
        # 3. Upsert user_data.user_coding_submissions
        # 4. Upsert code_blobs.user_coding_submission_code
        # 5. Set has_code = true
        # 6. Set capture_source = 'live_capture'
        # 7. Set code_capture_method = 'server_only'
        # 8. Recompute aggregate for affected slug
        # 9. Update last_live_capture_at
        # 10. Publish catalog event if statement is missing
```

This function should reuse or extract logic from `ingest_metadata_batch` and `ingest_code_batch`. Avoid duplicating large SQL blocks where possible.

## 10.4 Transaction rule

Live ingestion should be transaction-wrapped.

At minimum, these operations should be atomic:

- submission metadata upsert,
- code blob upsert,
- `has_code` update,
- aggregate recompute,
- `last_live_capture_at` update.

Queue publishing can happen inside the transaction if using PGMQ through Postgres, preserving the current Stage 1 pattern.

---

## 11. Database Write Behavior

## 11.1 `user_data.user_coding_submissions`

For live capture:

```text
capture_source = 'live_capture'
code_capture_method = 'server_only'
has_code = true
```

Conflict key:

```text
(platform, submission_id)
```

On conflict, update missing or newer fields safely.

Recommended conflict behavior:

- keep `platform` and `submission_id` fixed,
- update `verdict`, `raw_status`, runtime/memory/testcase fields,
- update `has_code = true`,
- update `code_capture_method`,
- preserve or update `capture_source` depending on completeness,
- update `updated_at = now()`.

## 11.2 `code_blobs.user_coding_submission_code`

For live capture:

```text
platform = 'leetcode'
submission_id = LeetCode submission ID
code = official submissionDetails.code
code_lang = official language
code_hash = SHA256(code)
storage_tier = 'hot'
fetched_at = now()
```

On conflict, update code fields if the new payload is server-confirmed.

## 11.3 `user_data.user_coding_problems`

After a live submission is inserted or updated, recompute aggregate for the affected slug.

Use existing:

```py
recompute_aggregates(conn, user_id, platform, slugs)
```

Only recompute the affected slug.

## 11.4 `user_data.user_coding_profiles`

Update:

```sql
last_live_capture_at = now()
```

for the authenticated user and platform.

## 11.5 `system.sync_state`

If adding `live` phase, update backend sync state to:

```text
phase = 'live'
```

after Stage 1 foreground sync completes or when Stage 2 first successfully captures a live submission.

Do not erase background queues or retry queues when setting live capability.

## 11.6 `catalog.coding_problems` and `extension_events`

If the live submission references a slug that does not have complete catalog metadata, publish:

```json
{
  "type": "submission.received",
  "user_id": "<authenticated-user-uuid>",
  "platform": "leetcode",
  "slug": "two-sum",
  "submission_id": 1973994054
}
```

The existing catalog worker should fill missing problem metadata.

---

## 12. Sync Phase Behavior

## 12.1 New phase

Add:

```text
live
```

## 12.2 Live-capable states

Stage 2 should run when the local state is one of:

```text
B_bg
live
done
```

Reason:

- `B_bg` means Stage 1 foreground sync is complete and background backfill is continuing.
- `live` means the extension is actively maintaining future submissions.
- `done` may exist from Stage 1 and should remain compatible.

## 12.3 Non-live-capable states

Stage 2 should not upload live submissions when local state is:

```text
idle
auth_checking
A_bulk
B_fg
paused
auth_required
failed
```

If a submission is detected during a non-live-capable state, the service worker may store it in a retry queue, but it should not upload until auth/sync state is valid.

---

## 13. Popup and User-Facing State

The popup does not own Stage 2 logic. It only displays state.

Add simple UI states:

| State | Suggested copy |
|---|---|
| Live active | `Live capture is active` |
| Submission captured | `Captured your latest LeetCode submission` |
| Auth required | `Log in to LeetCode to continue live capture` |
| CrackedIn disconnected | `Reconnect CrackedIn to continue syncing` |
| Retry pending | `Submission detected. Sync will retry soon` |
| Paused | `Sync is paused` |

Popup should read state the same way Stage 1 currently reads state: through Chrome messaging.

No dashboard polling is currently implemented. Stage 2 does not require dashboard polling for the core capture pipeline, but the backend timestamp `last_live_capture_at` should be available for future dashboard display.

---

## 14. Security and Privacy Rules

Stage 2 must follow the existing Stage 1 security posture.

Do not upload or store:

- LeetCode session cookie,
- LeetCode CSRF token,
- raw browser request headers,
- CrackedIn extension cookie value,
- raw network logs,
- unrelated page HTML,
- unrelated browsing history,
- local/session storage dumps.

The content script should only send:

```text
submissionId
source
detectedAt
optional problemSlug
```

The service worker should only upload sanitized submission/code payloads from official LeetCode detail responses.

The backend should derive `user_id` only from the HttpOnly cookie dependency.

---

## 15. Chrome Permissions Note

Current implementation reportedly includes:

```json
"permissions": ["storage", "alarms", "cookies"]
```

and LeetCode GraphQL helper uses `chrome.cookies.get` for CSRF handling.

Stage 2 should not add any new permissions.

There is a known test mismatch where an existing test asserts `chrome.cookies` is not used even though Stage 1 currently uses it. Stage 2 implementation should either:

1. update the outdated test to match the current Stage 1 implementation, or
2. separately refactor CSRF handling to avoid `chrome.cookies` if the project wants to remove the cookies permission later.

This is not part of the Stage 2 live-capture design itself, but it may affect test results.

---

## 16. Failure Handling

| Failure | Required behavior |
|---|---|
| LeetCode check endpoint fires multiple times | Local dedupe ignores repeated successful captures |
| Service worker receives duplicate event | Ignore if already captured or already queued |
| `submissionDetails` fails | Add submission ID to live retry queue |
| Backend upload fails | Add to retry queue with backoff |
| User logged out of LeetCode | Mark auth-required or retry later after auth recovery |
| CrackedIn cookie expired | Do not upload; require reconnect/login |
| User paused sync | Queue or ignore according to pause policy; do not upload immediately |
| Content script fails | Do not break LeetCode page; Stage 3 delta repair will catch later |
| User submits from mobile/another browser | Stage 2 will not see it; Stage 3 delta repair catches later |
| Backend receives duplicate submission | Upsert by `(platform, submission_id)` |

---

## 17. Implementation Steps

## 17.1 Extension implementation

1. Update `extension/src/shared/types.ts`.
   - Add `LEETCODE_SUBMISSION_DETECTED` message type.
   - Add `live` to sync phase type.
   - Add optional live retry queue types if needed.

2. Update `extension/src/content-scripts/leetcodeLivePlaceholder.ts`.
   - Patch/observe fetch and XHR.
   - Detect `/submissions/detail/<id>/check/`.
   - Extract `submissionId`.
   - Send runtime message to service worker.
   - Use defensive `try/catch`.

3. Update `extension/src/service-worker/index.ts`.
   - Add message handler for `LEETCODE_SUBMISSION_DETECTED`.
   - Call `handleLiveSubmissionDetected(...)`.

4. Add or update live handler.
   - Suggested file: `extension/src/service-worker/liveCapture.ts`.
   - Implement dedupe, state check, official detail fetch, upload, retry.

5. Reuse `fetchSubmissionDetails([submissionId])` from `extension/src/platforms/leetcode.ts`.

6. Update `extension/src/api/crackedinApi.ts`.
   - Add `uploadLiveSubmission(...)`.
   - Use `credentials: "include"`.

7. Update popup state display.
   - Show live-active status.
   - Show last capture/retry/auth-required states where local state supports them.

## 17.2 Backend implementation

1. Update `api/routes/extension_sync.py`.
   - Add `POST /api/extension/platforms/leetcode/live-submission`.
   - Use `get_current_extension_pg_user_id`.

2. Update `api/services/extension_sync/repository.py`.
   - Add `ingest_live_submission(...)`.
   - Reuse or extract shared metadata/code upsert logic.
   - Reuse `recompute_aggregates(...)`.
   - Reuse `_publish_catalog_event(...)`.

3. Ensure transaction-wrapped write behavior.

4. Update tests in `tests/test_extension_sync.py`.
   - Add route tests for live submission ingest.
   - Update outdated `chrome.cookies` assertion if tests are currently failing.

5. Add backend tests for:
   - duplicate live submission upsert,
   - missing catalog event publish,
   - aggregate recompute,
   - `last_live_capture_at` update,
   - cookie-auth-only user identity.

---

## 18. Acceptance Criteria

Stage 2 is complete when all of the following are true:

1. The existing LeetCode content script runs on `https://leetcode.com/problems/*`.
2. A LeetCode submit/check request is detected.
3. The content script extracts `submissionId` from `/submissions/detail/<id>/check/`.
4. The content script sends `LEETCODE_SUBMISSION_DETECTED` to the service worker.
5. The service worker receives the message without breaking existing Stage 1 messages.
6. The service worker deduplicates repeated check events.
7. The service worker calls `fetchSubmissionDetails([submissionId])`.
8. The service worker uploads a sanitized live payload to the backend.
9. Backend route authenticates user from the HttpOnly extension cookie.
10. Backend does not accept or trust user ID from the request body.
11. Backend inserts or updates `user_data.user_coding_submissions`.
12. Backend inserts or updates `code_blobs.user_coding_submission_code`.
13. Submission row has `capture_source = 'live_capture'`.
14. Submission row has `code_capture_method = 'server_only'`.
15. Submission row has `has_code = true`.
16. Backend recomputes the affected `user_data.user_coding_problems` row.
17. Backend updates `user_data.user_coding_profiles.last_live_capture_at`.
18. Backend publishes `submission.received` if catalog metadata is missing.
19. Duplicate events do not create duplicate rows.
20. Temporary fetch/upload failures enter a retry queue.
21. Popup can show live capture as active after Stage 1 foreground sync.
22. Stage 1 historical sync remains unaffected.

---

## 19. Final Stage 2 Design Summary

Stage 2 adds live submission capture to the already implemented Stage 1 historical sync system.

The final selected design is:

```text
Content script detects submissionId
  ↓
Service worker fetches official submissionDetails
  ↓
Service worker uploads sanitized live payload
  ↓
Backend validates and upserts existing tables
  ↓
Aggregates and profile timestamps are updated
```

No new database tables are required.

No new Chrome permissions are required.

No client-side user identity is trusted.

No LeetCode credentials are uploaded.

No AI/code-analysis worker is added in Stage 2.

The only new major pieces are:

- live logic inside `leetcodeLivePlaceholder.ts`,
- `LEETCODE_SUBMISSION_DETECTED` message type,
- service worker live handler,
- `uploadLiveSubmission(...)` API helper,
- backend `/live-submission` route,
- backend `ingest_live_submission(...)` function,
- local dedupe/retry support,
- simple popup live status.

This keeps Stage 2 small, reliable, and aligned with the existing Stage 1 architecture.
