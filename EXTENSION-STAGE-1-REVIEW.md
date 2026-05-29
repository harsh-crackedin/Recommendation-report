# Extension Stage 1 — Code Review & Issues

**Branch:** `extension-stage-1-implemented` (PR #6)
**Reviewer:** Bhaskar + Claude
**Date:** 2026-05-29
**Scope:** Full critical review targeting 5-10k users over 6 months

---

## 0. Implementation Approach (Step-by-Step)

This section documents what Harsh built end-to-end so we're both aligned.

### Architecture Overview

```
extension/              ← Chrome extension (Vite + TypeScript, builds to dist/)
├── manifest.json       ← MV3, permissions: storage, alarms, host: leetcode.com + crackedin.io
├── src/
│   ├── service-worker/ ← Background sync orchestration
│   ├── platforms/      ← LeetCode GraphQL client (aliased queries)
│   ├── api/            ← CrackedIn backend HTTP client
│   ├── popup/          ← Progress UI
│   ├── content-scripts/← Placeholder (Stage 2 live capture)
│   └── shared/         ← Types, storage helpers, constants

api/
├── routes/extension_sync.py         ← FastAPI router (8 endpoints)
├── services/extension_sync/
│   ├── cookie_auth.py               ← JWT-in-cookie auth for extension
│   ├── models.py                    ← Pydantic request models
│   ├── repository.py                ← DB operations (ingest, aggregates, sync state)
│   ├── normalization.py             ← Verdict, runtime, memory parsing
│   └── catalog_worker.py            ← Background pgmq consumer (fills problem statements)

web/app/(dashboard)/extension/connect/page.tsx  ← "Enable extension sync" web page

migrations/prod_full_schema.sql                 ← All 6 extension tables + indexes
```

### Auth Flow (How Extension Gets Authenticated)

```
1. User logs into crackedin.io web app (existing JWT session)
2. User visits /extension/connect page
3. Page calls POST /api/extension/auth/connect (requires web JWT)
4. Backend verifies user exists in SQLite → ensures Postgres user → issues extension cookie
5. Cookie: httpOnly, Secure, SameSite=None, domain=.crackedin.io, path=/api/extension
6. Extension popup calls GET /api/extension/me with credentials:'include'
7. Cookie is automatically sent → backend decodes JWT → returns user_id
```

### Sync Flow (Phase A → B-fg → B-bg)

```
Phase A (Bulk Metadata Walk):
  - Service worker loops: 10-aliased submissionList(offset, limit:20) = 200 rows/request
  - Sends batch to POST /api/extension/platforms/leetcode/submissions
  - Backend: upsert submissions + stub catalog problems + publish pgmq events + recompute aggregates
  - Saves cursor to chrome.storage.local after each batch
  - Stops when hasNext=false; builds foreground queue (top 100 IDs) + background queue (rest)

Phase B-fg (Foreground Code Fetch):
  - 10-aliased submissionDetails for top 100 submission IDs
  - Sends to POST /api/extension/platforms/leetcode/code
  - Backend: insert code_blobs + update has_code flag + enrich submission metadata
  - ~4 seconds for 100 codes

Phase B-bg (Background Code Fetch):
  - Same as B-fg but with 2s delay between batches
  - User can use the app; runs until all codes fetched
  - On completion: phase → 'done'

Catalog Worker (Background Thread):
  - Polls pgmq every 30s
  - Reads submission.received events for unknown slugs
  - Fetches problem statement from LC public GraphQL (no auth needed)
  - Fills catalog.coding_problems.statement_md
```

### Key Design Decisions in Implementation

| Decision | What Harsh Did |
|----------|----------------|
| Extension location | Inside monorepo at `extension/`, builds independently via Vite |
| Auth mechanism | Cookie-based JWT (not bearer token) — cookie set by web app, sent with `credentials: 'include'` |
| Sync orchestration | Service worker with chrome.alarms heartbeat every 5 min |
| Data flow | Extension → sanitize → batch POST → backend validates/normalizes → Postgres |
| Progress tracking | Local state in chrome.storage.local, mirrored to system.sync_state via PATCH |
| Catalog ownership | Backend-only writes via pgmq consumer |
| Idempotency | ON CONFLICT (platform, submission_id) DO UPDATE |
| Aggregate computation | Full recompute of user_coding_problems after every batch |

---

## 1. CRITICAL Issues (Must Fix Before Production)

### C1. IDOR: User A Can Overwrite User B's Submissions

**File:** `api/services/extension_sync/repository.py:174-199`

The `user_coding_submissions` PK is `(platform, submission_id)`. The upsert does:
```sql
ON CONFLICT (platform, submission_id) DO UPDATE SET
    user_id = EXCLUDED.user_id, ...
```

LeetCode submission IDs are sequential and guessable. If User B has submission 12345 and User A sends the same ID, User A **overwrites `user_id`**, stealing the row.

**Fix:** Add `WHERE user_data.user_coding_submissions.user_id = $1::uuid` to the ON CONFLICT clause. If the row belongs to someone else, skip silently.

---

### C2. Full-Table Aggregate Recomputation on Every Batch

**File:** `api/services/extension_sync/repository.py:335-389`

`recompute_aggregates` runs:
```sql
SELECT ... FROM user_coding_submissions
WHERE user_id = $1 AND platform = $2
GROUP BY user_id, platform, slug
```

This scans ALL submissions for the user and upserts ALL problem aggregates on every 200-item batch. For a 2000-sub user with 10 batches during Phase A: 10 × full scan × upsert of ~500 rows = massive write amplification.

At 5k users syncing concurrently: thousands of full-table GROUP BY queries + upserts competing for locks on `user_coding_problems`.

**Fix:** Pass the affected slugs from the current batch:
```sql
WHERE user_id = $1 AND platform = $2 AND slug = ANY($3::text[])
GROUP BY user_id, platform, slug
```

---

### C3. Long-Running Transaction Holds Connection for Entire Batch

**File:** `api/services/extension_sync/repository.py:159-237`

`ingest_metadata_batch` opens ONE transaction for up to 500 items, each with:
- nested savepoints (catalog stub)
- nested savepoints (submission upsert)
- pgmq publish per unknown slug
- full aggregates recompute at the end

This can hold a Postgres connection for 5-10+ seconds.

Pool has `max_size=10`. With 10 concurrent syncing users, the pool is **exhausted**. All other endpoints (dashboard, chat, /me) block.

**Fix:** Remove the outer transaction. Each submission upsert is already idempotent. Process items individually without wrapping everything in one giant transaction. Alternatively, reduce batch size to ~50 and use bulk INSERT with VALUES lists.

---

### C4. Extension Sends Entire Queue State to Backend on Every Batch

**File:** `extension/src/api/crackedinApi.ts:65-72`

`patchBackendSyncState(state)` sends the FULL `LocalSyncState` including:
- `backgroundCodeQueue` — up to 5000+ integer IDs
- `failedCodeQueue` — array of objects
- `retryCountMap` — unbounded object
- `lastSeenSubmissionIds` — accumulated during Phase A

For a 5k-submission user, this is **100-200KB per PATCH request**, sent dozens of times during sync. At 5-10k users: 5-10 MB/s of pure sync-state traffic.

**Fix:** Only send summary to backend: `{phase, offset, metadataFetched, codeDone, codeFailed, lastError}`. The full queue is an extension-local concern.

---

### C5. `lastSeenSubmissionIds` Accumulates ALL IDs in Memory

**File:** `extension/src/service-worker/jobs.ts:129-138`

```typescript
const allSeen = [...state.cursor.lastSeenSubmissionIds, ...sanitized.map((row) => row.submission_id)];
```

For a 5000-submission user: 5000 integers × 8 bytes each + JSON overhead = ~80KB growing in chrome.storage.local, read/written on every single batch. Combined with `backgroundCodeQueue`, `failedCodeQueue`, and `retryCountMap`, this approaches the point where chrome.storage.local serialization becomes a perf bottleneck.

**Fix:** Don't accumulate. Build code queues only at the end of Phase A. During Phase A, only store the current offset (which is sufficient for resumability — re-fetched IDs on resume are deduped by the backend's ON CONFLICT).

---

## 2. HIGH Issues (Fix Before 1k Users)

### H1. No CSRF Token on LeetCode GraphQL Calls

**File:** `extension/src/platforms/leetcode.ts:55-66`

LeetCode's GraphQL endpoint checks for `X-CSRFToken` header matching the `csrftoken` cookie. Without it, requests may randomly fail or get rate-limited more aggressively.

**Fix:** Add `"cookies"` permission to manifest. Read `csrftoken` via `chrome.cookies.get({url: 'https://leetcode.com', name: 'csrftoken'})`. Attach as `X-CSRFToken` header.

**Note:** This changes the CWS review posture — cookie READ permission is a more sensitive permission. Consider testing first whether LC actually enforces this for GraphQL from service workers before adding.

---

### H2. Service Worker Lifetime vs Background Sync Duration

**File:** `extension/src/service-worker/jobs.ts:112-156` (Phase A loop), `175-243` (Phase B loops)

MV3 service workers are killed after ~5 minutes of continuous execution. Phase B-background for a 5k-submission user takes ~10 minutes in a `while(true)` loop with 2s sleeps. Chrome WILL kill this mid-execution.

The 5-minute alarm restarts it, but there's a gap. The `syncRunning` flag resets to `false` on restart, so `continueSyncFromState` picks it up — this works. But the gap means ~5 min of dead time per service worker kill for background sync.

**Fix:** For Phase B-bg, switch to an alarm-driven model: process one batch per alarm tick (set 5s alarm), rather than a continuous loop. For Phase A and B-fg (which are <70s), the continuous loop is fine — Chrome keeps workers alive while `fetch()` is in-flight.

---

### H3. `storage_tier` Value Mismatch

**File:** `api/services/extension_sync/repository.py:273`

Code inserts `storage_tier = 'postgres'`. Schema defines values as `'hot' | 'archived'`, default `'hot'`. The partial index is: `WHERE storage_tier = 'hot'`.

**Impact:** All rows are invisible to the index. Future queries like "find hot code blobs for archival" won't find any rows.

**Fix:** Change `'postgres'` to `'hot'`.

---

### H4. `status` Value Mismatch in Aggregates

**File:** `api/services/extension_sync/repository.py:349`

```sql
CASE WHEN COUNT(*) FILTER (WHERE verdict = 'AC') > 0 THEN 'solved' ELSE 'attempted' END
```

Schema specifies: `status TEXT NOT NULL -- 'ac' | 'tried' | 'untouched'`

Any dashboard query filtering by `status = 'ac'` will return zero results.

**Fix:** Use `'ac'` and `'tried'`.

---

### H5. Catalog Worker Infinite Retry Loop

**File:** `api/services/extension_sync/catalog_worker.py:109-118`

When `_fetch_problem` fails, the message is NOT deleted from pgmq. It becomes visible again after the 30-second visibility timeout and retries forever.

A single deleted/broken LC problem creates 2 retries/minute × forever = infinite traffic to LC. With 100 broken slugs: 200 useless requests/minute.

**Fix:** Use pgmq's `read_ct` to track retries. After 5 failures, `pgmq.archive()` the message. Add a periodic sweep for `sync_status = 'failed'` problems as a repair mechanism.

---

### H6. `catalog.coding_problems.title` is NOT NULL But Extension May Send NULL

**File:** `api/services/extension_sync/repository.py:134-149`

`_ensure_catalog_stub` inserts with `title = $3` where `$3` comes from `item.title` which can be `None`. The `catalog.coding_problems.title` column is `TEXT NOT NULL` in the migration. This causes:
1. Catalog stub INSERT fails with NOT NULL violation
2. Exception caught, `needs_catalog = True`
3. Submission INSERT fails on FK constraint because `(platform, slug)` doesn't exist in catalog

Result: submissions with `title=None` for new slugs are permanently skipped.

**Fix:** Use `COALESCE($3, $2)` (fall back to slug as title) in the stub insert, or make the column nullable.

---

### H7. No Rate-Limit Detection (429) for LeetCode Calls

**File:** `extension/src/platforms/leetcode.ts:61-65`

Only 401/403 trigger `LeetCodeAuthError`. A 429 (rate limit) falls through to generic error handling with inadequate backoff (1s, 2s, 4s via `withRetry`).

At 5k+ users, even a few concurrent syncs can trigger LC rate limiting. The fast retry makes it worse.

**Fix:** Detect 429, respect `Retry-After` header, use 10s+ delays, and surface "Rate limited — retrying" in sync state.

---

### H8. No Request Body Size Limit on Code Uploads

**File:** `api/services/extension_sync/models.py:67` — `code: str` has no max_length

A single batch of 500 items × large code strings = potentially hundreds of MB. Entire payload held in memory during the transaction.

**Fix:** Add `Field(max_length=500_000)` to `SubmissionCode.code`. Add ASGI body-size middleware (e.g., `LimitUploadSize` or nginx upstream limit).

---

## 3. MEDIUM Issues (Fix Before 5k Users)

### M1. `sync_status = 'stub'` Is Not a Valid Value

**File:** `api/services/extension_sync/repository.py:141`

Schema defines: `sync_status TEXT NOT NULL DEFAULT 'pending'` with convention `'pending' | 'complete' | 'failed'`. Code uses `'stub'`. This won't break functionally (partial index still includes it), but confuses consumers and breaks any future CHECK constraint.

**Fix:** Use `'pending'`.

---

### M2. `syncRunning` Is In-Memory and Non-Atomic

**File:** `extension/src/service-worker/jobs.ts:19`

`syncRunning` resets to `false` every time the service worker restarts. The guard prevents double-entry within a single worker lifecycle, but after a restart, `continueSyncFromState()` can re-enter while a previous batch might be in-flight (if the worker was killed mid-fetch, the fetch is abandoned, so this is actually safe in practice). However, there's no protection against the popup sending `START_SYNC` while background sync is running on a fresh worker instance.

**Fix:** Write a `sync_lock_at` timestamp to chrome.storage.local. On entry, check if lock is < 60s old. Refresh on each batch.

---

### M3. Foreground Code Queue is NOT Sorted by Timestamp

**File:** `extension/src/service-worker/jobs.ts:137`

```typescript
foregroundCodeQueue: uniqueIds.slice(0, 100),
```

The spec says: "top 100 sub IDs by ts DESC" (most recent 100). But `uniqueIds` is accumulated in walk order (oldest first for `submissionList` pagination). This means the foreground queue gets the OLDEST 100, not the newest.

**Fix:** Sort by timestamp descending before slicing. Either pass `ts` along with IDs during accumulation, or reverse the array (since `submissionList` returns newest-first by default — verify this assumption against actual LC API behavior).

---

### M4. Popup Button Logic Relies on Displayed Text Content

**File:** `extension/src/popup/popup.ts:164-176`

```typescript
if (primaryBtn.textContent === "Resume") { ... }
```

If you change labels, localize, or add icons, click handlers break silently.

**Fix:** Track state in a variable, branch on state not on rendered text.

---

### M5. No `updated_at = now()` in ON CONFLICT Clauses

**File:** `api/services/extension_sync/repository.py` — all upsert queries

The `updated_at` column has `DEFAULT now()` which only applies on INSERT. ON CONFLICT DO UPDATE does not set it, so `updated_at` is stale forever after the first write.

**Fix:** Add `updated_at = now()` to all ON CONFLICT SET lists.

---

### M6. `_publish_catalog_event` Silently Swallows Failures

**File:** `api/services/extension_sync/repository.py:114-131`

If pgmq is not installed or the queue doesn't exist, events are permanently lost. No retry, no dead-letter, no periodic sweep.

**Impact:** Some problems will never get `statement_md` filled. This degrades AI code-review features that need problem context.

**Fix:** Add a periodic sweep: `SELECT platform, slug FROM catalog.coding_problems WHERE sync_status != 'complete' AND created_at < now() - interval '1 hour'` → re-publish events.

---

### M7. `broadcastSyncState` Fires on Every State Update (Popup Usually Closed)

**File:** `extension/src/shared/storage.ts:58-64`

`chrome.runtime.sendMessage` throws "Could not establish connection" when no receiver exists (popup closed). This fires dozens of times per sync, generating console noise.

**Fix:** Use `chrome.runtime.getContexts?.({contextTypes: ['POPUP']})` to check if popup is open first.

---

### M8. No Per-User Rate Limiting on Sync Endpoints

**File:** `api/routes/extension_sync.py`

A compromised extension or replay attack could flood the backend. With 5-10k users, even 1% misbehaving = 50-100 clients saturating the connection pool.

**Fix:** Add per-user rate limiting: max 10 metadata batch requests/minute, max 10 code batch requests/minute.

---

### M9. Catalog Worker Creates Its Own Separate Connection Pool

**File:** `api/services/extension_sync/catalog_worker.py:130-143`

Separate `asyncpg` pool (min=1, max=2) in a background thread. This uses 2 additional connections invisible to the main pool. Supabase Pro pooler typically limits to 15-20 per role.

**Fix:** Share the main pool (pass it from startup), or at minimum account for these 2 connections in capacity planning.

---

### M10. `command_timeout=120` Is Too Long

**File:** `api/postgres.py:48`

A stuck query holds a connection for 2 minutes. With pool size 10: 10 stuck queries = full outage for 2 minutes.

**Fix:** Reduce to 30s. No sync operation should legitimately take 120s.

---

## 4. LOW Issues (Nice to Have)

### L1. Content Script Injected But Does Nothing

**File:** `extension/manifest.json:32-37`, `extension/src/content-scripts/leetcodeLivePlaceholder.ts`

The content script is `export {};`. It still runs on every `/problems/*` page, increasing CWS review surface.

**Fix:** Remove content_scripts from manifest until Stage 2.

---

### L2. Missing `minimum_chrome_version` in Manifest

**File:** `extension/manifest.json`

Uses MV3 features requiring Chrome 109+. Without this field, old Chrome installs get cryptic errors.

**Fix:** Add `"minimum_chrome_version": "109"`.

---

### L3. Vite May Code-Split Service Worker

**File:** `extension/vite.config.ts:19`

`chunkFileNames: "chunks/[name].js"` allows code-splitting. Dynamic imports in service workers can fail on restart.

**Fix:** For service-worker entry, set `inlineDynamicImports: true` or `manualChunks: undefined`.

---

### L4. No Integration Tests for Actual SQL Logic

**File:** `tests/test_extension_sync.py`

Tests only cover auth cookie handling (good coverage there). No tests for idempotency, FK handling, aggregate correctness, or the IDOR vulnerability.

**Fix:** Add at least 3 integration tests: (1) double-upsert same submission, (2) reject user_id overwrite, (3) aggregate correctness after batch.

---

### L5. `retryCountMap` Grows Unbounded

**File:** `extension/src/service-worker/jobs.ts:207-209`

Never cleaned up after success or permanent failure. Bloats storage over time.

**Fix:** Remove entries when items succeed or are permanently abandoned (retry_count >= 4).

---

### L6. `INITIAL_SYNC_STATE.lastProgressAt` Captures Module Load Time

**File:** `extension/src/shared/storage.ts:20`

`Date.now()` is evaluated once at module load. If used as a fallback, it shows stale time.

**Fix:** Use `0` as initial value.

---

## 5. Architecture Question: Extension in Monorepo

**Current state:** Extension lives at `extension/` inside the `interview-prep-ai` monorepo.

**Is this correct?** YES — this is a standard approach.

- Extension has its own `package.json`, `vite.config.ts`, and builds independently
- Zero runtime dependency on the API code
- Same repo = coordinated changes in single PRs, shared CI
- For CWS: `cd extension && npm run build` → zip `dist/` → upload
- Extraction to separate repo is trivial later if needed

No action needed here — this is fine for your scale.

---

## 6. Summary Table

| Severity | Count | Theme |
|----------|-------|-------|
| Critical | 5 | IDOR, aggregate perf, connection exhaustion, storage bloat |
| High | 8 | CSRF, worker lifetime, value mismatches, infinite retry, body limits |
| Medium | 10 | Schema inconsistencies, rate limits, silent failures |
| Low | 6 | Cleanup, tests, minor correctness |

### Priority Order for Harsh

1. **C1 (IDOR)** — security vulnerability, one-line fix
2. **C2 + C3** — performance/scalability, needed before any real traffic
3. **H3 + H4** — value mismatches that break queries silently
4. **H5** — infinite retry loop wastes resources
5. **C4 + C5** — bandwidth + storage, fix before 1k users
6. **H6** — FK failures silently dropping data
7. Everything else in priority order

---

## 7. What's NOT in This Review (Out of Scope)

- Web app changes (sidebar, chat, settings) — unrelated to extension
- Stage 2 (live capture) — not implemented yet
- Stage 3 (delta refresh) — not implemented yet
- Intelligence layer — explicitly deferred
- CWS listing copy, screenshots, privacy policy — product concerns

---

## 8. Testing Note

The extension cannot be tested locally by Bhaskar (Chrome extension upload is restricted). Harsh should:
1. Fix critical issues
2. Test with Chrome developer mode (Load unpacked → `extension/dist/`)
3. Test against crackedin-dev Supabase instance
4. Verify full sync for an account with 500+ submissions
5. Verify pool behavior with 3+ concurrent syncs (can simulate with multiple test users)
