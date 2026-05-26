# Extension Sync — Stage 1 (First-Time Sync)

**Status:** Locked after design discussion 2026-05-26.
**Scope:** First-time sync flow only. Stage 2 (live capture) and Stage 3 (delta refresh) are separate docs.
**Supersedes:** `EXTENSION-SYNC-ARCHITECTURE.md` §6 Phase B and §10 (those described 3-bucket fetch + 100s sync; this doc replaces with last-100 + background backfill + event queue).

---

## 1. Goal

A brand-new user installs the Crackedin extension, connects their LeetCode account, and within ~70 seconds:
- Full solve graph is in our DB (every problem ever attempted, every verdict)
- Last 100 codes are loaded (covers recent ACs + recent failures + active struggles)
- Dashboard is fully usable
- Background backfill continues to load remaining codes asynchronously

Total perceived sync time: **~70 seconds.** Background completion: ~5–10 minutes for power users (non-blocking).

---

## 2. Core decisions (locked)

| Decision | Outcome |
|---|---|
| Code fetch policy | **Last 100 chronological** in foreground; **rest in background**. Drop the 3-bucket logic from the original architecture doc. |
| `coding_problems` ownership | **Backend only.** Extension never writes catalog data — only user-specific tables. |
| Catalog freshness | **Two layers: bulk seed (one-time) + lazy fetch on submit.** No cron. No page-open triggers. |
| Event queue | **pgmq on Supabase.** One event type: `submission.received`. One consumer in Stage 1: catalog worker (lazy statement fetch). Code analysis / intelligence / recommendation consumers are deferred until those features are designed. |
| Hosting | **Single Postgres on Supabase Pro ($25/mo).** Four schemas (catalog, user_data, code_blobs, system). |
| Cache | **In-process FastAPI TTL cache.** No Redis until 5k+ DAU. |
| Storage tier | **All code in Postgres for v1.** Add `storage_tier`/`s3_key` columns now, defer S3 archival to v2. |
| Intelligence layer | **Out of scope for Stages 1–3.** Code analysis, weakness aggregates, recommendations all need separate design. Raw corpus must exist first. |

---

## 3. Actors

| Actor | Where | Role in Stage 1 |
|---|---|---|
| **Web app** | Vercel / crackedin.app | Onboarding UI, dashboard polling |
| **Extension popup** | Chrome popup | Sync trigger, progress display |
| **Service worker** | Chrome background | LC GraphQL calls, batched ingest to backend |
| **Backend (FastAPI)** | Server | Receives ingest, writes user tables, publishes events |
| **Postgres (Supabase)** | Managed | 4-schema DB |
| **pgmq queue** | Same Postgres | Single `events` queue |
| **Catalog worker** | Same FastAPI service, async task | Lazy-fetch unknown problem statements |
| **LeetCode GraphQL** | leetcode.com/graphql | Source of truth |

Catalog worker is the only async task function for Stages 1–3. Code analyzer, intelligence worker, and recommend worker are deferred — they will be designed and added once raw corpus is stable.

---

## 4. Stage 1 timeline

```
T=0      User signs up on Crackedin (web)
         → backend creates user row, issues JWT

T=10s    User clicks "Connect LeetCode" on onboarding page
         → web shows "Install our extension"
         → link to Chrome Web Store

T=30s    User installs extension
         → Chrome shows host_permissions consent (leetcode.com)
         → user approves
         → extension popup auto-opens

T=32s    Popup checks Crackedin auth
         → reads our app cookie OR shows "Open Crackedin and log in"

T=33s    Popup checks LeetCode auth via userStatus query
         (credentials: 'include')
         → if !isSignedIn: "Open LC and log in" CTA
         → if signed in: shows username + Premium status

T=35s    User clicks "Sync my LeetCode"
         → popup → service worker: {action: 'START_SYNC'}
         → worker writes sync_state to chrome.storage.local
           {phase: 'A_bulk', cursor: {offset: 0}}
         → enters PHASE A

T=35-65s PHASE A — Bulk metadata walk (~30s)
         (see §5)

T=65s    Phase A done. Dashboard already shows full solve graph.
         (No code yet, but 80% of analytics already work)
         → enters PHASE B foreground

T=65-69s PHASE B-foreground — Last 100 codes (~4s)
         (see §6)

T=69s    Foreground sync complete.
         → popup: "✅ All synced. Background sync continuing."
         → dashboard chat unlocks code-aware tools
         → enters PHASE B-background

T=69s+   PHASE B-background — Remaining codes
         (Power users: ~5-10 minutes; casual users: seconds)
         → non-blocking; user can use the app
         → progress visible in popup if reopened

End      Worker enters 'live' phase (Stage 2 territory)
```

---

## 5. Phase A — Bulk metadata walk

**Goal:** enumerate user's complete submission history with metadata (no code).

### Loop

```
worker_phase_A():
  loop:
    offset = read sync_state.cursor.offset
    aliased_query = build 10 pages: p0..p9
                    each: submissionList(offset=N+i*20, limit=20)
    response = POST leetcode.com/graphql {query: aliased_query}
              with credentials: 'include'
    rows = flatten response.p0..p9
    
    POST /api/sync/leetcode/submissions
      Authorization: Bearer <crackedin JWT>
      body: {batch: rows, cursor_offset: offset, last_batch: bool}
    
    backend:
      - upsert coding_problems (slug, title only — stub if unknown)
      - upsert user_coding_profiles (if first batch)
      - upsert user_coding_problems aggregate counts
      - upsert user_coding_submissions (one per attempt, has_code=false)
      - publish submission.received events for unknown slugs ONLY
        (so catalog worker fetches their statements)
    
    if any page returned hasNext: false:
      cursor.last_phase_A_sub_ids = top 100 sub IDs by ts DESC
      cursor.background_queue = remaining sub IDs by ts DESC
      sync_state.phase = 'B_fg'
      break
    else:
      cursor.offset += 200
      write sync_state to storage
      post progress to popup
```

### Performance

- 10-aliased call returns 200 rows in ~3s (validated)
- 2,000 submissions = 10 round trips = ~30s
- 5,000 submissions = 25 round trips = ~75s (rare)

### Resumability

- `sync_state` written to `chrome.storage.local` after every batch
- If worker dies mid-batch: next wake re-reads cursor, re-runs aliased query at same offset
- Backend upserts are idempotent on `(platform, submission_id)` PK — replay safe

### Backend ingest endpoint

```
POST /api/sync/leetcode/submissions
{
  user_id: extracted from JWT,
  batch: [
    {sub_id, slug, title, status_display, lang, runtime, memory, ts},
    ...
  ],
  cursor_offset: 200,
  last_batch: false
}

response: {ok: true, unknown_slugs_queued: 3}
```

Backend logic:
1. Upsert `coding_problems(platform, slug)` — INSERT IF NOT EXISTS, fields: slug, title, sync_status='pending' (full statement filled later)
2. Upsert `user_coding_profiles` if this is first batch (extension passes a flag)
3. Upsert `user_coding_submissions` — bulk INSERT ... ON CONFLICT DO UPDATE
4. Recompute `user_coding_problems` aggregates for affected slugs (status, attempt counts, verdict mix)
5. For each slug not in `coding_problems` already, publish event `submission.received` for catalog worker (catalog worker handles dedup if multiple events for same slug)
6. Return ok

---

## 6. Phase B-foreground — Last 100 codes

**Goal:** fetch code for the 100 most recent submissions. Blocks the "sync done" UX state.

### Loop

```
worker_phase_B_fg():
  queue = sync_state.cursor.last_phase_A_sub_ids  // top 100 by ts DESC
  while queue not empty:
    batch = queue.shift(10)
    aliased = build s0..s9: submissionDetails(submissionId=batch[i])
    response = POST leetcode.com/graphql {query: aliased}
    codes = flatten response.s0..s9
    
    POST /api/sync/leetcode/code
      body: {batch: codes}
    
    backend:
      - upsert user_coding_submission_code rows
      - update user_coding_submissions.has_code=true
      - publish submission.received event ONLY for slugs whose
        coding_problems.statement_md is still NULL
        (so catalog worker fetches those statements)
    
    sync_state.cursor.code_done += 10
    write to storage
    post progress
  
  sync_state.phase = 'B_bg'
```

### Performance

- 10-aliased `submissionDetails` returns 10 codes in ~350ms (validated)
- 100 codes = 10 round trips = ~4s

### What gets published to the event queue

- One `submission.received` event per unique slug whose statement isn't yet in catalog
- Catalog worker (idempotent) fetches the missing statement and fills `coding_problems.statement_md`
- No code analysis runs in Stage 1 — that worker doesn't exist yet (deferred until intelligence layer is designed)

---

## 7. Phase B-background — Remaining codes

**Goal:** load remaining codes without blocking UX.

### Behavior

- Service worker continues processing `cursor.background_queue` after foreground sync completes
- Same 10-aliased fetch loop as B-foreground
- Smaller batch cadence (e.g. 1 batch every 2s) to avoid LC rate limits during steady-state usage
- Resumable across worker restarts via cursor
- User can use the app fully while this runs
- Popup shows "Background sync: 234/2000" if user reopens

### Why background instead of "skip entirely"

Future intelligence features will need the full corpus:
- Weakness detection (patterns across all failures)
- Code style critique (recurring patterns across many ACs)
- Cross-problem similarity ("you've solved 5 sliding window problems")

Stage 1 doesn't ship those features — but storing the raw code now means we don't have to re-sync later when we do build them. Without backfill, those features would only see the recent 100 codes — too narrow.

### Cost discipline

Background-loaded codes do NOT trigger any analysis events. Code is stored as raw corpus only. When the intelligence layer is designed (post-Stage 3), it can read this corpus retroactively — no need to re-sync. Hot-running an LLM over every backfilled submission would cost ~$50–200 per power user, which we explicitly avoid in Stage 1.

---

## 8. Event queue contract

### Single producer

```
{type: 'submission.received',
 user_id: ...,
 platform: 'leetcode',
 slug: 'two-sum',
 sub_id: 1973994054}
```

### Single consumer (Stage 1)

```
[Catalog Worker]
  Idempotent: SELECT FROM coding_problems WHERE platform=$1 AND slug=$2
  If statement_md IS NULL or sync_status='pending':
    fetch question(titleSlug) via LC GraphQL
    convert HTML -> markdown
    update coding_problems SET statement_md, constraints_md, examples_json,
                                  sync_status='complete'
```

**Deferred consumers** (will be added in a separate intelligence-layer design pass after Stages 1–3 ship):
- Code analyzer — per-submission LLM analysis (complexity, style, correctness)
- Intelligence worker — per-user weakness/strength aggregates
- Recommend worker — precomputed next-problem recommendations

The single producer + single consumer in Stage 1 is intentional: we don't add event types or consumers until both ends actually exist and the data shape is locked.

### Why pgmq

- Zero new infrastructure (already on Supabase)
- Transactional with DB writes — no "submission saved but event lost" bugs
- Free
- Migrate to Redis/SQS later only if throughput becomes a bottleneck (~10k events/sec — far away)

---

## 9. Catalog freshness — two layers

### Layer 1 — Bulk seed (one-time, before public launch)

Admin script seeds `coding_problems` with full metadata:
- LC: paginated `problemsetQuestionList` → ~3,500 rows
- CF: `problemset.problems` → ~10,000 rows in one API call
- For each: aliased `question(titleSlug) {content, examples, hints, topics}`
- Convert HTML → markdown, store in `coding_problems`
- Total: ~30 minutes, ~70 MB

### Layer 2 — Lazy fetch on `submission.received`

Catalog worker handles the gap between bulk seed and new problems added by LC after launch:
- New problem added Monday, user solves it Tuesday
- Extension submits → backend ingests submission
- Backend publishes `submission.received`
- Catalog worker sees `coding_problems` has stub row (or no row)
- Fetches statement async, fills in
- ~1-2s latency, doesn't block ingest

### What we explicitly do NOT do

- ❌ No cron job to fetch new problems weekly
- ❌ No fetch-on-page-open trigger (refresh-spam unsafe)
- ❌ No daily-challenge pre-fetch
- ❌ No catalog write from extension (backend-only)
- ❌ No embeddings yet (defer to recommendation work)

---

## 10. Storage schema (relevant tables for Stage 1)

Full column definitions with types and indexes live in `EXTENSION-SYNC-SCHEMA.md`. This section is the human-readable summary so you can see at a glance what each table holds and why it exists.

### Schema: `catalog` — shared problem catalog (backend-owned)

**Table: `catalog.coding_problems`** — one row per (platform, slug). Shared across all users.

| Column | Purpose |
|---|---|
| `platform` | `'leetcode'` \| `'codeforces'` \| `'gfg'` — part of PK |
| `slug` | URL slug, e.g. `'two-sum'` — part of PK |
| `display_id` | Human-visible problem number (`'1'` for LC, `'1234A'` for CF) |
| `title` | Problem title |
| `difficulty` | `'easy'` \| `'medium'` \| `'hard'` |
| `topic_tags` | Array of tags, e.g. `['array', 'hash-table']` |
| `statement_md` | Problem body, HTML→markdown converted |
| `constraints_md` | Constraints section |
| `examples_json` | `[{input, output, explanation}]` |
| `sync_status` | `'pending'` (stub) \| `'complete'` \| `'failed'` |
| `raw_meta` | Platform-specific extras (JSONB) |

### Schema: `user_data` — per-user data (high-write during sync)

**Table: `user_data.user_coding_profiles`** — links Crackedin user to one external coding account.

| Column | Purpose |
|---|---|
| `id` | Surrogate PK |
| `user_id` | FK → `public.users.id` |
| `platform` | `'leetcode'` \| `'codeforces'` \| `'gfg'` |
| `external_handle` | Visible username on the platform |
| `is_premium` | LC Premium flag |
| `is_connected` | False if user disconnected |
| `sync_enabled` | False if user paused sync |
| `connected_at` | First connection time |
| `last_full_sync_at` | Last completed Stage 1 sync |
| `last_delta_sync_at` | Last Stage 3 delta refresh |
| `last_live_capture_at` | Last Stage 2 live capture |

**Table: `user_data.user_coding_problems`** — per-user-per-problem aggregate. Dashboard's primary read table.

| Column | Purpose |
|---|---|
| `user_id, platform, slug` | Composite PK |
| `status` | `'ac'` \| `'tried'` \| `'untouched'` |
| `first_ac_at`, `latest_ac_at` | First and most recent accepted submission |
| `latest_attempt_at` | Most recent attempt of any verdict |
| `attempt_count` | Total attempts |
| `ac_count`, `wa_count`, `tle_count`, `mle_count`, `re_count`, `ce_count`, `other_count` | Per-verdict counts |
| `best_runtime_ms`, `best_memory_kb`, `best_lang` | Best AC stats |
| `primary_submission_id` | FK → `user_coding_submissions`, latest AC by default |

**Table: `user_data.user_coding_submissions`** — every individual attempt. Idempotent on `(platform, submission_id)`.

| Column | Purpose |
|---|---|
| `platform, submission_id` | Composite PK |
| `user_id` | Denormalized for fast queries |
| `profile_id` | FK → `user_coding_profiles.id` |
| `slug` | FK → `coding_problems.slug` |
| `verdict` | Normalized: `AC`\|`WA`\|`TLE`\|`MLE`\|`RE`\|`CE`\|`OTHER` |
| `raw_status` | Original status string from LC (`'Accepted'`, `'Wrong Answer'`, ...) |
| `lang`, `lang_verbose` | `'cpp'` / `'C++'` |
| `runtime_ms`, `runtime_percentile` | Speed metrics |
| `memory_kb`, `memory_percentile` | Memory metrics |
| `total_correct`, `total_testcases` | Testcase pass count |
| `last_testcase`, `expected_output`, `actual_output` | Non-AC failure details |
| `runtime_error`, `compile_error` | Error messages for RE/CE |
| `ts` | Platform-reported submission time |
| `captured_at` | When we ingested it |
| `capture_source` | `'historical_sync'` \| `'live_capture'` \| `'delta_repair'` \| `'manual_resync'` |
| `has_code` | True if code blob exists in `user_coding_submission_code` |

### Schema: `code_blobs` — code text isolated for archival

**Table: `code_blobs.user_coding_submission_code`** — separate table because blobs are large and S3-archivable independently.

| Column | Purpose |
|---|---|
| `platform, submission_id` | Composite PK; FK → `user_coding_submissions` |
| `code` | Raw source code text |
| `code_lang` | Language slug |
| `code_hash` | SHA256 for dedup |
| `storage_tier` | `'hot'` (in Postgres) \| `'archived'` (moved to S3) — v2 |
| `s3_key` | Populated when `storage_tier='archived'` — v2 |
| `fetched_at` | When the code was retrieved |

### Schema: `system` — sync orchestration

**Table: `system.sync_state`** — per-user-per-platform sync cursor. Mirrored to extension's `chrome.storage.local`.

| Column | Purpose |
|---|---|
| `user_id, platform` | Composite PK |
| `phase` | `'A_bulk'` \| `'B_fg'` \| `'B_bg'` \| `'live'` \| `'paused'` \| `'done'` |
| `cursor` | JSONB: `{offset, last_phase_A_sub_ids, background_queue, code_done}` |
| `last_progress_at` | Heartbeat for stuck-sync detection |
| `last_full_sync_at` | Last successful end-to-end Stage 1 |
| `retry_count`, `last_error` | For retry/backoff logic |

---

## 11. Failure modes

| Failure | Recovery |
|---|---|
| Worker dies mid-Phase A | Next wake reads cursor, resumes at last offset. Idempotent upsert. |
| LC rate-limits us | Worker exponential backoff (10s → 30s → 60s); popup shows "LC throttling, retrying…" |
| User logs out of LC mid-sync | Worker gets 401; sync paused; popup prompts re-login |
| User closes Chrome | Sync pauses; resumes via `chrome.alarms` heartbeat on relaunch |
| Backend down | Worker exponential backoff, queues batches in chrome.storage |
| User uninstalls mid-sync | Backend has partial data; if reinstalled, sync_state row resumes from cursor |
| LC GraphQL schema breaks | Worker logs to backend; popup shows "Sync issue, we're on it" |
| Catalog worker fails on a slug | Event re-delivered (pgmq retry); after N failures → DLQ + alert |

---

## 12. Cost at 1k DAU

| Component | Monthly cost |
|---|---|
| Supabase Pro (Postgres + Auth + Pooler + pgmq) | $25 |
| Vercel Pro (Next.js) | $20 |
| OpenAI tokens (chat + code analysis) | $50–150 (variable) |
| LC catalog embeddings (deferred) | $0 |
| S3 backups | $1 |
| **Total** | **~$100/mo** |

The expensive line is OpenAI tokens, not infra. Optimization focus belongs there, not on the queue or DB.

---

## 13. Open questions

1. **Extension ↔ backend auth:** how does the extension get a Crackedin JWT?
   - Option A: scoped cookie permission on `crackedin.app`, read app cookie
   - Option B: token paste in popup (worse UX, more secure)
   - **Tentative pick:** A with B as fallback for users blocking third-party cookies
2. **What if user has 10k+ submissions?** Phase A scales linearly. 10k = ~150s. Show ETA in popup.
3. **What if user has zero submissions?** Skip Phase B; show empty state with "Solve a problem and we'll sync it" CTA.
4. **Encryption at rest for code blobs?** Probably yes. Supabase supports it natively.
5. **User uninstall purge:** delete `user_coding_*` rows immediately. Catalog stays (shared, non-personal).

---

## 14. What's NOT in this doc

- **Stage 2:** live capture during active LC sessions (separate doc)
- **Stage 3:** periodic delta refresh (separate doc)
- **Multi-platform:** Codeforces and GFG sync (separate doc; schema is platform-agnostic)
- **Code analyzer internals:** which model, prompt template, output schema (separate doc)
- **Intelligence/Recommendation worker logic:** what gets computed, how (separate docs)
- **Onboarding UI mockups:** out of scope here
