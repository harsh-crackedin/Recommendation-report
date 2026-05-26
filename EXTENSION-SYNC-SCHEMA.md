# Extension Sync — Database Schema

**Status:** Locked 2026-05-26.
**Scope:** Schema for Stages 1–3 (sync, live capture, delta refresh). Multi-platform from day one.
**Companion docs:** `EXTENSION-SYNC-STAGE-1.md` (flow), `EXTENSION-SYNC-ARCHITECTURE.md` (extension internals).

**Intelligence layer is intentionally out of scope for this doc.** Code analysis, weakness aggregates, and recommendations need separate design work — they are not straightforward, and rushing them into the schema before we've shipped Stages 1–3 risks locking in the wrong shape. Defer until raw corpus is stable.

---

## 1. Final schema — 6 tables for Stages 1–3

### Schema layout (Postgres on Supabase)

```
crackedin_db
├── catalog            -- shared, low-write
│   └── coding_problems
│
├── user_data          -- per-user, high-write during sync
│   ├── user_coding_profiles
│   ├── user_coding_problems
│   └── user_coding_submissions
│
├── code_blobs         -- the heavy table; isolated for archival later
│   └── user_coding_submission_code
│
└── system
    └── sync_state
```

### Deferred (separate design pass after Stages 1–3 ship)

- `intelligence.user_coding_code_analysis` — needs decision on which model, prompt template, when to run, output schema
- `intelligence.user_coding_intelligence` — needs decision on which signals matter, refresh cadence, scoring approach
- `intelligence.user_coding_recommendations` — needs decision on ranking algorithm, candidate set, freshness
- `catalog.coding_problem_similarity` — depends on recommendation approach (embeddings vs. structural)

These tables will be designed once raw corpus exists and we can study real usage patterns.

---

## 2. Table definitions

### 2.1 `catalog.coding_problems`

Shared problem catalog. One row per (platform, slug). Backend-owned — extension never writes.

```sql
CREATE TABLE catalog.coding_problems (
  platform        TEXT NOT NULL,            -- 'leetcode' | 'codeforces' | 'gfg'
  slug            TEXT NOT NULL,            -- 'two-sum'
  display_id      TEXT,                     -- '1' for LC, '1234A' for CF
  title           TEXT NOT NULL,
  difficulty      TEXT,                     -- 'easy' | 'medium' | 'hard'
  topic_tags      TEXT[],                   -- ['array', 'hash-table']
  ac_rate         NUMERIC(5,2),
  
  statement_md    TEXT,                     -- problem body, markdown-converted
  constraints_md  TEXT,
  examples_json   JSONB,                    -- [{input, output, explanation}]
  hints           TEXT[],
  similar_slugs   TEXT[],                   -- LC's similarQuestions field
  
  sync_status     TEXT DEFAULT 'pending',   -- 'pending' | 'complete' | 'failed'
  raw_meta        JSONB,                    -- platform-specific extras
  
  created_at      TIMESTAMPTZ DEFAULT now(),
  updated_at      TIMESTAMPTZ DEFAULT now(),
  
  PRIMARY KEY (platform, slug)
);

CREATE INDEX idx_coding_problems_difficulty ON catalog.coding_problems(platform, difficulty);
CREATE INDEX idx_coding_problems_topics ON catalog.coding_problems USING GIN(topic_tags);
CREATE INDEX idx_coding_problems_sync_status ON catalog.coding_problems(sync_status) WHERE sync_status != 'complete';
```

### 2.2 `user_data.user_coding_profiles`

Links a Crackedin user to one external coding account. One row per (user, platform) pair.

```sql
CREATE TABLE user_data.user_coding_profiles (
  id                      BIGSERIAL PRIMARY KEY,
  user_id                 UUID NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
  platform                TEXT NOT NULL,
  
  external_user_id        TEXT,                 -- platform's internal user ID (stable)
  external_handle         TEXT NOT NULL,        -- visible username
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
  
  UNIQUE (user_id, platform)
);

CREATE INDEX idx_profiles_user ON user_data.user_coding_profiles(user_id);
CREATE INDEX idx_profiles_handle ON user_data.user_coding_profiles(platform, external_handle);
```

### 2.3 `user_data.user_coding_problems`

Per-user-per-problem aggregate. The "solve graph." UI's primary read table.

```sql
CREATE TABLE user_data.user_coding_problems (
  user_id                 UUID NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
  platform                TEXT NOT NULL,
  slug                    TEXT NOT NULL,
  
  status                  TEXT NOT NULL,       -- 'ac' | 'tried' | 'untouched'
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
  
  primary_submission_id   BIGINT,              -- FK → user_coding_submissions, latest AC by default
  
  created_at              TIMESTAMPTZ DEFAULT now(),
  updated_at              TIMESTAMPTZ DEFAULT now(),
  
  PRIMARY KEY (user_id, platform, slug),
  FOREIGN KEY (platform, slug) REFERENCES catalog.coding_problems(platform, slug) ON DELETE RESTRICT
);

CREATE INDEX idx_user_problems_status ON user_data.user_coding_problems(user_id, status);
CREATE INDEX idx_user_problems_latest_attempt ON user_data.user_coding_problems(user_id, latest_attempt_at DESC);
```

### 2.4 `user_data.user_coding_submissions`

Every individual attempt. The atomic unit. Idempotent on `(platform, submission_id)`.

```sql
CREATE TABLE user_data.user_coding_submissions (
  platform                TEXT NOT NULL,
  submission_id           BIGINT NOT NULL,
  
  user_id                 UUID NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
  profile_id              BIGINT NOT NULL REFERENCES user_data.user_coding_profiles(id) ON DELETE CASCADE,
  slug                    TEXT NOT NULL,
  
  verdict                 TEXT NOT NULL,       -- 'AC'|'WA'|'TLE'|'MLE'|'RE'|'CE'|'OTHER'
  raw_status              TEXT,                -- 'Accepted' | 'Wrong Answer' | etc.
  status_code             INTEGER,
  
  lang                    TEXT NOT NULL,       -- 'cpp' | 'python3' | 'java'
  lang_verbose            TEXT,                -- 'C++' | 'Python3'
  
  runtime_ms              INTEGER,
  runtime_percentile      NUMERIC(5,2),
  memory_kb               INTEGER,
  memory_percentile       NUMERIC(5,2),
  
  total_correct           INTEGER,
  total_testcases         INTEGER,
  
  -- only populated for non-AC verdicts
  last_testcase           TEXT,
  expected_output         TEXT,
  actual_output           TEXT,
  runtime_error           TEXT,
  compile_error           TEXT,
  
  ts                      TIMESTAMPTZ NOT NULL,    -- platform-reported submission time
  captured_at             TIMESTAMPTZ DEFAULT now(),
  capture_source          TEXT NOT NULL,           -- 'historical_sync'|'live_capture'|'delta_repair'|'manual_resync'
  
  has_code                BOOLEAN DEFAULT FALSE,
  
  raw_meta                JSONB,
  
  created_at              TIMESTAMPTZ DEFAULT now(),
  updated_at              TIMESTAMPTZ DEFAULT now(),
  
  PRIMARY KEY (platform, submission_id),
  FOREIGN KEY (platform, slug) REFERENCES catalog.coding_problems(platform, slug) ON DELETE RESTRICT
);

CREATE INDEX idx_submissions_user_ts ON user_data.user_coding_submissions(user_id, ts DESC);
CREATE INDEX idx_submissions_user_slug ON user_data.user_coding_submissions(user_id, platform, slug, ts DESC);
CREATE INDEX idx_submissions_verdict ON user_data.user_coding_submissions(user_id, verdict, ts DESC);
CREATE INDEX idx_submissions_lang ON user_data.user_coding_submissions(user_id, lang);
```

### 2.5 `code_blobs.user_coding_submission_code`

Code text only. Separate table because blobs are large and bench-archivable independently of metadata.

```sql
CREATE TABLE code_blobs.user_coding_submission_code (
  platform                TEXT NOT NULL,
  submission_id           BIGINT NOT NULL,
  
  code                    TEXT NOT NULL,
  code_lang               TEXT NOT NULL,
  code_hash               TEXT,                    -- SHA256 for dedup
  
  storage_tier            TEXT DEFAULT 'hot',      -- 'hot' | 'archived'
  s3_key                  TEXT,                    -- populated when storage_tier='archived'
  
  fetched_at              TIMESTAMPTZ DEFAULT now(),
  
  PRIMARY KEY (platform, submission_id),
  FOREIGN KEY (platform, submission_id) REFERENCES user_data.user_coding_submissions(platform, submission_id) ON DELETE CASCADE
);

CREATE INDEX idx_code_storage_tier ON code_blobs.user_coding_submission_code(storage_tier) WHERE storage_tier = 'hot';
```

### 2.6 `system.sync_state`

Per-user-per-platform sync cursor. Mirrored into extension's `chrome.storage.local`.

```sql
CREATE TABLE system.sync_state (
  user_id                 UUID NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
  platform                TEXT NOT NULL,
  
  phase                   TEXT,                    -- 'A_bulk'|'B_fg'|'B_bg'|'live'|'paused'|'done'
  cursor                  JSONB,
  
  last_progress_at        TIMESTAMPTZ,
  last_full_sync_at       TIMESTAMPTZ,
  
  retry_count             INTEGER DEFAULT 0,
  last_error              TEXT,
  
  created_at              TIMESTAMPTZ DEFAULT now(),
  updated_at              TIMESTAMPTZ DEFAULT now(),
  
  PRIMARY KEY (user_id, platform)
);
```

---

## 3. Tables required for Stages 1–3

All 6 tables ship in one migration. Stages 1–3 use them as follows:

| Table | Stage 1 (sync) | Stage 2 (live) | Stage 3 (delta) |
|---|---|---|---|
| `catalog.coding_problems` | Backend stub-inserts on unknown slug | Same | Same |
| `user_data.user_coding_profiles` | Created on connect | `last_live_capture_at` updated | `last_delta_sync_at` updated |
| `user_data.user_coding_problems` | Aggregates computed from Phase A | Refreshed per new submission | Refreshed per recovered submission |
| `user_data.user_coding_submissions` | Bulk insert from Phase A + B | Single insert per live capture | Single insert per missed sub |
| `code_blobs.user_coding_submission_code` | Last-100 + background backfill | Single insert per live capture | Single insert per missed sub |
| `system.sync_state` | Phase cursor written each batch | `phase = 'live'` | `last_delta_sync_at` heartbeat |

Intelligence layer is intentionally deferred — see §1 note.

---

## 4. Comparison — Harsh's 2-table proposal vs. our 6-table design

### Harsh's MVP proposal (`crackedin_coding_corpus_database_report.md`)

```
user_coding_profiles      -- account connection + sync state
user_coding_submissions   -- 40+ fields including code, raw_detail_response
                          -- problem metadata denormalized (slug, title, difficulty,
                          --  topic_tags, problem_url) on every row
```

His doc explicitly defers these to "Future Expansion" (§18):
- `coding_problems` (problem catalog normalization)
- `user_coding_submission_code` (code blob isolation)
- `coding_sync_runs`
- `coding_analysis_cache`

### What Harsh got right

| Decision | Why it's correct |
|---|---|
| User-coding-profile as account-connection layer | ✅ Adopted as-is |
| `(platform, submission_id)` as canonical key | ✅ Adopted |
| Capture-source field + capture-method tracking | ✅ Adopted |
| Verdict normalization (AC/WA/TLE/...) + raw status preservation | ✅ Adopted |
| User-id stored directly on submissions for fast queries | ✅ Adopted |
| Sync cursor on profile row for resumability | ✅ Adopted (we put it in a separate `sync_state` table for cleanliness) |
| Platform-agnostic from day one | ✅ Adopted |
| Idempotent upserts as ingestion contract | ✅ Adopted |

The shape of his thinking is right. The issues are about **how big "submissions" is allowed to get** and **what's missing for AI features.**

### What's wrong with the 2-table approach for our 6-month roadmap

| Problem | 2-table impact | 6-table fix |
|---|---|---|
| **Code blobs inflate the hot table** | `user_coding_submissions` rows are ~3KB each (code + raw_detail_response). 50k users × 1k subs = 1.5M rows × 3KB = 4.5GB on the most-read table. Every metadata-only query (dashboard, verdict counts, recent activity) drags blobs through buffer cache. | Code lives in `code_blobs.user_coding_submission_code`. Hot table rows shrink to ~300 bytes. Indexes 10x smaller. Buffer cache holds 10x more rows. |
| **Problem metadata duplicated 1.5M times** | `slug`, `title`, `difficulty`, `topic_tags`, `problem_url`, `problem_external_id`, `problem_frontend_id` repeated on every submission row. ~500 bytes × 1.5M = 750MB of pure duplication. Updating problem metadata = 1.5M-row UPDATE. | `catalog.coding_problems` has one row per problem (~20k total). Submissions reference (platform, slug) as FK. Update once, applies everywhere. |
| **No solve-graph table** | Computing "740 solved problems, 18 topics" for the dashboard requires `GROUP BY slug` over 1.5M submissions every page load. ~500ms even with indexes. | `user_coding_problems` has one row per (user, problem) with pre-computed verdict mix, attempt counts, best runtime. Dashboard load = ~5ms. |
| **Problem statement nowhere to go** | If we want to feed the LLM problem context for code review ("what was the problem you're reviewing?"), there's no `statement_md` field. Adding it to submissions = duplicate it 1.5M times. Adding it to a new `coding_problems` table = the 6-table design. | Statement on `coding_problems`, fetched once, used by every user. |
| **Deletes get expensive** | User uninstalls → DELETE 1.5M+ rows on a table actively being read. Lock contention, index thrashing. | User data spread across smaller tables; FK CASCADE handles it cleanly. |
| **Can't archive old code without rewriting submissions** | To move cold codes to S3, you'd need to NULL out the `code` column on submissions. Mixed table state, harder reasoning. | `user_coding_submission_code` row independent. Set `storage_tier='archived'`, NULL code, write s3_key. Submissions table untouched. |
| **No catalog ownership boundary** | Extension writes problem metadata as part of submissions. 5k users hitting "Sync" race-write the same denormalized fields with possibly stale values. | Backend owns `coding_problems`. Extension only writes user_data. Clear authorial boundary. |

### Side-by-side feature support

| Feature | 2-table (Harsh) | 6-table (ours) |
|---|---|---|
| Store every submission with code | ✅ | ✅ |
| Multi-platform (LC, CF, GFG) | ✅ — both designs platform-agnostic | ✅ |
| Idempotent ingest | ✅ | ✅ |
| Dashboard "740 solved, verdict mix" | ⚠️ recompute on every load | ✅ pre-computed in user_coding_problems |
| Lazy code archival to S3 | ❌ requires submission-row rewrites | ✅ separate code blob table with storage_tier |
| LLM gets problem statement for code review | ❌ no statement field | ✅ coding_problems.statement_md |
| Cron-free catalog freshness | ⚠️ extension would need to write catalog | ✅ backend lazy-fetch on submission.received |
| Per-user weakness analysis | ❌ no aggregate layer | ⚠️ intelligence layer deferred (designed later) |
| Recommendation precomputation | ❌ nowhere to store | ⚠️ intelligence layer deferred (designed later) |

### Same goals, different scales

Both designs achieve the same user-visible MVP:
- Sync history ✅
- Store code ✅
- Multi-platform ready ✅
- Idempotent ✅

The difference is **what they enable next**:

| Want to ship... | 2-table | 6-table |
|---|---|---|
| ...sync + raw history view | Both work | Both work |
| ...dashboard at 1k users | Slow GROUP BY on every load | Fast — `user_coding_problems` precomputed |
| ...code review at 5k users | Buffer cache thrashing from inline code blobs | Fine — code lives in separate table |
| ...code archival to S3 later | Rewrites submission rows | Independent code-blob row, no rewrite |
| ...weakness / recommendations | Needs new tables — start of intelligence layer redesign | Same — new intelligence tables, but raw corpus already correctly shaped |

**Harsh's design is correct for a 1-month MVP.** Our 6-table design is correct for a 6-month roadmap. Same user-visible features at MVP, but the raw corpus is shaped so we can add intelligence tables later without rewriting any of it.

---

## 5. Are we missing anything Harsh's design covers?

Audit pass. Things in Harsh's doc we should double-check we haven't dropped:

| Harsh field/concern | In our schema? |
|---|---|
| `client_captured_code` vs `server_confirmed_code` distinction | ⚠️ **NOT YET.** Our `code_blobs.user_coding_submission_code.code` only stores final canonical. For Stage 1 (historical sync) only server-confirmed exists, so this is fine. **Add `code_capture_method` field for Stage 2 live capture** when we want to track the client/server reconciliation. |
| `total_correct`, `total_testcases` | ✅ on `user_coding_submissions` |
| `runtime_percentile`, `memory_percentile` | ✅ on `user_coding_submissions` |
| `runtime_error`, `compile_error` text | ✅ on `user_coding_submissions` |
| `runtime_display`, `memory_display` (formatted strings) | ❌ **dropped intentionally.** Compute from runtime_ms / memory_kb at render time. Storing both = duplication, drift. |
| `code_hash` for dedup | ✅ on `user_coding_submission_code` |
| `raw_submission_meta`, `raw_detail_response` JSONB | ✅ as `raw_meta` on submissions. We don't dump full raw_detail_response on every row — too big. |
| `is_historical_imported`, `is_live_captured` flags | ⚠️ replaced by single `capture_source` enum. Two booleans are redundant; `capture_source = 'live_capture'` is cleaner. |
| `language` + `language_verbose` | ✅ as `lang` + `lang_verbose` |
| Sync cursor as JSON | ✅ in `system.sync_state.cursor` |
| Disconnect / pause flags | ✅ `is_connected`, `sync_enabled` on profiles |
| Audit by capture_source | ✅ |

**Net additions to add to our schema before intern starts:**

```sql
-- on user_coding_submissions
ALTER TABLE user_data.user_coding_submissions
  ADD COLUMN code_capture_method TEXT;   -- 'historical' | 'client_then_server' | 'server_only'
  -- only meaningful in Stage 2; defaults NULL in Stage 1
```

That's the only gap. Everything else in Harsh's doc maps cleanly onto our schema.

---

## 6. Migration strategy for the intern

**Single migration, all 6 tables, in this order:**

```
Step 1: CREATE SCHEMA catalog, user_data, code_blobs, system;

Step 2: CREATE TABLE catalog.coding_problems
        (no FK dependencies)

Step 3: CREATE TABLE user_data.user_coding_profiles
        (FK to public.users only)

Step 4: CREATE TABLE user_data.user_coding_problems
        (FK to public.users + catalog.coding_problems)

Step 5: CREATE TABLE user_data.user_coding_submissions
        (FK to public.users, profiles, coding_problems)

Step 6: CREATE TABLE code_blobs.user_coding_submission_code
        (FK to user_coding_submissions)

Step 7: CREATE TABLE system.sync_state
        (FK to public.users)

Step 8: All indexes (per table definitions above)

Step 9: Seed catalog.coding_problems with bulk LC + CF problems
        (separate one-time admin script — see Stage 1 doc §9)
```

---

## 7. Decisions to flag back to Harsh

When sharing the schema with Harsh, summary should be:

1. **You were right about:** profile-as-connection layer, capture_source tracking, idempotent upsert on submission_id, multi-platform from day one, normalized verdict + raw status, user_id denormalized onto submissions for query speed.
2. **We split your `user_coding_submissions` into 4 tables:**
   - Code blob → `code_blobs.user_coding_submission_code` (archival-ready)
   - Problem metadata → `catalog.coding_problems` (shared catalog)
   - Per-user solve-graph aggregate → `user_data.user_coding_problems` (dashboard performance)
   - Submission core stays in `user_data.user_coding_submissions` (slim, hot-path)
3. **We added one field you didn't have:** `coding_problems.statement_md` for LLM-aware code review.
4. **We dropped:** `runtime_display`/`memory_display` (compute at render), `is_historical_imported`/`is_live_captured` booleans (subsumed by `capture_source`).
5. **Non-changes:** sync cursor stays as JSONB, raw platform metadata stays preserved.
6. **Deferred:** code analysis, weakness intelligence, and recommendation tables. Designed separately after Stages 1–3 ship — they need real corpus before we lock the schema.
