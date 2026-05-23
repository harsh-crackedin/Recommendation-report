# CrackedIn Coding Corpus Database Report

**Document purpose:** This report defines the MVP database structure for storing coding-platform activity inside CrackedIn.

**Scope:** Database model, integration with the existing CrackedIn user system, submission storage, sync state storage, and AI-readiness of the captured corpus.

**Current product direction:** Add two coding-corpus tables that attach external coding-platform data to existing CrackedIn users:

1. `user_coding_profiles`
2. `user_coding_submissions`

The system starts with LeetCode and remains flexible enough to support GFG, Codeforces, code courses, and native CrackedIn practice environments later.

---

## 1. Executive Summary

CrackedIn needs a database layer that can store a user's real coding history, not just solved-problem counts or profile summaries.

The browser extension will import historical LeetCode submissions and continue capturing future submissions. The backend must store that data in a way that is:

- connected to the existing CrackedIn user account,
- simple enough for MVP implementation,
- rich enough for AI analysis,
- safe against duplicate ingestion,
- flexible enough for future coding platforms,
- compatible with full historical code archival.

The selected database structure uses two new tables:

| Table | Purpose |
|---|---|
| `user_coding_profiles` | Links a CrackedIn user to an external coding-platform account |
| `user_coding_submissions` | Stores every submitted attempt, including code, verdict, language, runtime, memory, and metadata |

This design keeps CrackedIn's current `users` table as the source of truth and adds a platform-agnostic coding corpus around it.

---

## 2. Existing CrackedIn Database Context

CrackedIn already has a user-centered database. The current user system should remain the root identity system.

The existing schema already contains concepts such as:

- users,
- problems,
- solved-problem status,
- LeetCode profile summaries,
- topic summaries,
- contest history,
- chat sessions,
- chat messages.

The new coding-corpus tables should not replace these existing tables. They should extend the system by adding full submission-level history and code snapshots.

The important distinction is:

| Existing data | What it represents |
|---|---|
| User profile data | Who the CrackedIn user is |
| Problem data | Shared coding-question metadata |
| Solved-problem data | Whether a user solved a problem |
| LeetCode profile summary | Aggregated stats from a profile |
| New coding-submission data | Every actual coding attempt and code snapshot |

The current system can tell that a user solved or attempted problems. The new system should explain **how** they attempted them, **what code** they wrote, **what failed**, and **how their submissions changed over time**.

---

## 3. Database Design Principle

The central database principle is:

> Keep CrackedIn users as the source of truth, attach external coding identities to those users, and store every coding attempt as a submission record.

This means:

- do not create a separate user system for coding platforms,
- do not put every platform-specific field into the `users` table,
- do not store only summary data,
- do not rely only on solved-problem status,
- do not treat LeetCode as the only long-term platform,
- do not duplicate the same external submission multiple times.

The model should support this relationship:

```text
CrackedIn user
  -> connected coding profile
      -> coding submissions
```

A single CrackedIn user may later connect several external coding accounts.

Example:

| CrackedIn user | Connected platform | External username |
|---|---|---|
| User 1 | LeetCode | harshsrivastava05 |
| User 1 | GFG | harsh05 |
| User 1 | Codeforces | harsh_cf |
| User 1 | CrackedIn practice | internal |

---

## 4. Final Table Set

The MVP coding-corpus database layer contains two new tables.

## 4.1 `user_coding_profiles`

This table stores the connection between a CrackedIn user and one external coding-platform profile.

It answers questions such as:

- Which LeetCode account belongs to this CrackedIn user?
- Is the external account connected?
- Is sync enabled?
- What platform is this profile from?
- What is the external username?
- When was the last full sync?
- When was the last delta sync?
- Where should sync resume if interrupted?
- What was the last sync error?

This table is the account-connection and sync-state table.

## 4.2 `user_coding_submissions`

This table stores the actual submission corpus.

It answers questions such as:

- What problem did the user submit?
- What code did the user write?
- What language did they use?
- Was the result accepted or failed?
- What kind of failure happened?
- What was the runtime?
- What was the memory usage?
- How many testcases passed?
- Was this imported historically or captured live?
- What raw platform metadata was available?

This table is the main AI analysis table.

---

## 5. Why This Two-Table Structure Is Appropriate for MVP

The two-table structure gives CrackedIn the fastest path to a useful product while preserving future scalability.

It is appropriate for MVP because:

| Reason | Explanation |
|---|---|
| Minimal schema change | Only two tables need to be added to the existing CrackedIn database |
| Clean user integration | The current CrackedIn `users` table remains the root identity |
| Easier ingestion | Extension data has a clear destination: profile data or submission data |
| Easier debugging | Sync issues can be traced through one profile row and related submission rows |
| Full code archival support | Every historical and future submission can store code directly |
| Platform flexibility | LeetCode-specific and future platform-specific fields can coexist |
| AI readiness | Attempt-level code, verdicts, language, topics, and timestamps are available in one corpus |
| Future normalization path | Larger tables can be split out later without changing the product model |

For MVP, the immediate value comes from reliably importing and using the corpus. Additional separation of problems, code blobs, sync runs, analysis caches, and event logs can be introduced later when scale or product requirements justify it.

---

# 6. Table: `user_coding_profiles`

## 6.1 Purpose

`user_coding_profiles` represents one external coding account connected to one CrackedIn user.

A CrackedIn user may have zero, one, or many connected coding profiles.

Examples:

| user_id | platform | platform_username |
|---|---|---|
| 1 | leetcode | harshsrivastava05 |
| 1 | gfg | harsh05 |
| 1 | codeforces | harsh_cf |

This table prevents the existing `users` table from becoming cluttered with platform-specific fields.

The existing `users.leetcode_username` field can remain for backward compatibility or display, but the long-term source of truth for external coding accounts should be `user_coding_profiles`.

---

## 6.2 Recommended Fields

| Field | Purpose |
|---|---|
| `id` | Internal primary key for the connected coding profile |
| `user_id` | References the existing CrackedIn user |
| `platform` | External platform name, such as `leetcode`, `gfg`, `codeforces`, `code_course`, or `crackedin_practice` |
| `platform_user_id` | External platform's user ID if available |
| `platform_username` | External platform username or handle |
| `platform_display_name` | External display name if available |
| `platform_profile_url` | Public or user-facing profile URL |
| `is_connected` | Whether this account is currently connected |
| `sync_enabled` | Whether automatic sync is enabled |
| `sync_mode` | Sync behavior for this profile; MVP default is full archive |
| `sync_status` | Current sync state |
| `last_full_sync_at` | Last completed historical sync timestamp |
| `last_delta_sync_at` | Last periodic repair sync timestamp |
| `last_live_capture_at` | Last successful live-captured submission timestamp |
| `sync_cursor` | Resumable sync state |
| `sync_error` | Last sync error message, if any |
| `settings` | User/platform-specific sync settings |
| `raw_profile` | Sanitized raw profile metadata from the platform |
| `created_at` | Row creation timestamp |
| `updated_at` | Last update timestamp |

---

## 6.3 Field Details

### `user_id`

This field connects the external platform profile to the existing CrackedIn user.

Relationship:

```text
users.id -> user_coding_profiles.user_id
```

This keeps CrackedIn's existing authentication and account system intact.

---

### `platform`

This field identifies where the profile came from.

Initial value:

| Value | Meaning |
|---|---|
| `leetcode` | LeetCode account synced through the browser extension |

Future values may include:

| Value | Meaning |
|---|---|
| `gfg` | GeeksforGeeks |
| `codeforces` | Codeforces |
| `code_course` | Coding-course platform |
| `crackedin_practice` | Native CrackedIn coding environment |

The platform field is required because different platforms have different user IDs, problem IDs, submission formats, verdict names, and code-fetch mechanisms.

---

### `platform_user_id`

This stores the external platform's internal user ID if available.

For LeetCode, this comes from the authenticated user-status response.

This field is useful because usernames can change, while platform user IDs are often more stable.

---

### `platform_username`

This stores the visible username or handle.

For LeetCode, this is the profile username.

It is useful for:

- display,
- support,
- debugging,
- account reconnection,
- manual verification.

---

### `sync_mode`

For the current implementation, the default sync behavior is:

```text
full_archive
```

This means CrackedIn imports and stores code for every historical submission, not just accepted submissions or selected attempts.

Keeping `sync_mode` as a field gives the product flexibility later, even though MVP uses full archive by default.

---

### `sync_status`

This field records where the sync currently is.

Recommended values:

| Status | Meaning |
|---|---|
| `not_started` | Profile connected but sync has not started |
| `metadata_syncing` | Historical metadata is being fetched |
| `code_syncing` | Historical code snapshots are being fetched |
| `live_enabled` | Historical sync is done and live capture is active |
| `delta_syncing` | Periodic repair is running |
| `paused` | User or system paused sync |
| `failed` | Last sync failed |
| `complete` | Full historical sync completed |

This gives the frontend and extension enough state to display progress and resume safely.

---

### `sync_cursor`

This field stores resumable sync state.

It is important because browser extension workers can stop during long-running jobs. Full historical sync may also be interrupted by browser shutdown, network failure, rate limiting, backend errors, or user pause.

The cursor should be able to track:

| Cursor data | Purpose |
|---|---|
| Current sync phase | Metadata sync, code sync, delta repair, etc. |
| Current historical offset | Resume paginated submission-list fetching |
| Submission IDs discovered | Continue after metadata discovery |
| Code-fetch progress | Resume all historical code archival |
| Last processed submission ID | Debugging and retry recovery |
| Retry count | Avoid repeated failing loops |
| Total discovered submissions | Show progress |
| Total processed submissions | Show progress |

This field can be structured JSON/text depending on the database engine.

---

### `raw_profile`

This stores sanitized platform profile metadata.

For LeetCode, this may include:

- username,
- user ID,
- avatar URL,
- real/display name if available,
- premium flag if available,
- profile ranking,
- public summary stats.

Do not store credentials, cookies, raw headers, or unrelated browser data.

---

# 7. Table: `user_coding_submissions`

## 7.1 Purpose

`user_coding_submissions` stores every submitted coding attempt.

For LeetCode, one row corresponds to one LeetCode submission ID.

This table stores:

- historical submissions,
- live future submissions,
- accepted submissions,
- failed submissions,
- multiple attempts on the same problem,
- code snapshots,
- judge details,
- platform metadata.

This table is the core user coding corpus.

---

## 7.2 Recommended Fields

| Field | Purpose |
|---|---|
| `id` | Internal primary key |
| `user_id` | References existing CrackedIn user |
| `coding_profile_id` | References connected external coding profile |
| `platform` | Platform name, such as `leetcode` |
| `platform_submission_id` | Submission ID from external platform |
| `problem_slug` | Platform problem slug, such as LeetCode `two-sum` |
| `problem_title` | Human-readable problem title |
| `problem_external_id` | Platform internal problem ID if available |
| `problem_frontend_id` | Display problem number if available |
| `problem_url` | URL to problem page |
| `difficulty` | Problem difficulty |
| `topic_tags` | Topic tags from platform or CrackedIn mapping |
| `language` | Language slug, such as `cpp`, `python3`, `java` |
| `language_verbose` | Display name, such as `C++`, `Python3`, `Java` |
| `code` | Final canonical submitted code snapshot |
| `client_captured_code` | Code captured from editor/submit flow, if available |
| `server_confirmed_code` | Code returned by official submission-details API |
| `code_hash` | Hash of canonical code for deduplication and change detection |
| `code_capture_method` | How code was captured or reconciled |
| `verdict` | Normalized verdict |
| `raw_status` | Raw platform status text |
| `status_code` | Platform status code if available |
| `runtime_ms` | Runtime in milliseconds |
| `runtime_display` | Runtime as displayed by platform |
| `runtime_percentile` | Runtime percentile if available |
| `memory_bytes` | Memory usage in bytes |
| `memory_display` | Memory as displayed by platform |
| `memory_percentile` | Memory percentile if available |
| `total_correct` | Number of passed testcases |
| `total_testcases` | Total testcase count |
| `last_testcase` | Last testcase or failing testcase if available |
| `code_output` | User code output if available |
| `expected_output` | Expected output if available |
| `runtime_error` | Runtime error message if available |
| `compile_error` | Compile error message if available |
| `submitted_at` | Timestamp when user submitted |
| `captured_at` | Timestamp when CrackedIn captured it |
| `capture_source` | Historical sync, live capture, delta repair, or manual resync |
| `is_historical_imported` | Whether this came from historical sync |
| `is_live_captured` | Whether this came from live session capture |
| `raw_submission_meta` | Sanitized raw metadata from submission-list response |
| `raw_detail_response` | Sanitized raw detail response from platform |
| `created_at` | Row creation timestamp |
| `updated_at` | Last update timestamp |

---

## 7.3 Required Uniqueness Rule

Every platform submission must have one canonical row.

The uniqueness rule should be:

```text
platform + platform_submission_id
```

This prevents duplicates when the same submission is discovered through multiple paths.

The same submission can appear during:

- historical sync,
- live capture,
- periodic delta repair,
- manual re-sync,
- retry after failed upload.

All writes should behave as upserts.

---

## 7.4 Why Store Both `user_id` and `coding_profile_id`

`coding_profile_id` already points to a connected profile, and that profile points to a user. However, submissions should also store `user_id` directly.

This is recommended because:

| Benefit | Explanation |
|---|---|
| Faster AI queries | Most AI analysis starts with all submissions for one user |
| Simpler authorization | The backend can verify submission ownership directly |
| Easier dashboards | User-level submission queries do not always need joins |
| Better indexing | Queries by user, time, verdict, topic, and language become simpler |

The ingestion layer must ensure that `user_id` matches the owner of the `coding_profile_id`.

---

# 8. Code Storage Strategy

The current product decision is to store all historical code.

Therefore, the submission table should include code fields directly for MVP.

There are three code concepts:

| Field | Meaning |
|---|---|
| `client_captured_code` | Code captured from the editor or submit payload during live capture |
| `server_confirmed_code` | Code returned by the official submission-details source |
| `code` | Canonical code used by CrackedIn |

## 8.1 Historical Sync

For historical sync:

- `server_confirmed_code` is fetched from the platform's official submission-details response.
- `code` should usually equal `server_confirmed_code`.
- `client_captured_code` is usually null because the submission happened before extension installation.

## 8.2 Live Capture

For live capture:

- the extension first captures `client_captured_code`,
- then it detects the official submission ID,
- then it fetches server-confirmed details,
- then it updates `server_confirmed_code`,
- then it sets or confirms canonical `code`.

## 8.3 Conflict Handling

If client-captured and server-confirmed code differ:

- keep both values,
- set `code` to the server-confirmed version,
- record the mismatch through `code_capture_method`,
- optionally mark the record for inspection or debugging.

Recommended canonical rule:

| Situation | Canonical `code` |
|---|---|
| Historical sync | Server-confirmed code |
| Live capture before server details | Client-captured code temporarily |
| Live capture after server details | Server-confirmed code |
| Client/server mismatch | Server-confirmed code |
| Detail fetch failed | Client-captured code until delta repair succeeds |

---

# 9. Verdict and Status Strategy

The database should store both raw platform status and normalized CrackedIn verdict.

## 9.1 Raw Status

`raw_status` preserves the exact status text returned by the platform.

Examples:

| Raw status |
|---|
| Accepted |
| Wrong Answer |
| Time Limit Exceeded |
| Memory Limit Exceeded |
| Runtime Error |
| Compile Error |
| Output Limit Exceeded |
| Internal Error |
| Timeout |
| Restrictions Failed |

This is useful for debugging, future platform support, and status mapping updates.

## 9.2 Normalized Verdict

`verdict` should use a small internal vocabulary.

Recommended values:

| Verdict | Meaning |
|---|---|
| `AC` | Accepted |
| `WA` | Wrong Answer |
| `TLE` | Time Limit Exceeded |
| `MLE` | Memory Limit Exceeded |
| `RE` | Runtime Error |
| `CE` | Compile Error |
| `OLE` | Output Limit Exceeded |
| `IE` | Internal Error |
| `OTHER` | Unknown or platform-specific status |

This makes AI analysis and dashboards easier across platforms.

---

# 10. Capture Source Strategy

Every submission should record how it entered CrackedIn.

Recommended `capture_source` values:

| Value | Meaning |
|---|---|
| `historical_sync` | Imported during full historical backfill |
| `live_capture` | Captured while user submitted code in browser |
| `delta_repair` | Found by periodic repair after being missed |
| `manual_resync` | Imported during user-triggered re-sync |

This tells CrackedIn how each record was discovered.

A submission may be touched multiple times. For example, a live-captured row may later be seen again by delta repair. The backend should update the existing row rather than insert a duplicate.

---

# 11. Problem Metadata Strategy

CrackedIn already has a shared problem table for LeetCode-style problems. The submission table should still store problem metadata directly because future platforms may not fit the same problem model.

Recommended submission-level fields:

| Field | Purpose |
|---|---|
| `problem_slug` | Platform-level problem key |
| `problem_title` | Display and AI context |
| `problem_external_id` | Platform-specific problem ID |
| `problem_frontend_id` | Display problem number |
| `problem_url` | Direct link to problem |
| `difficulty` | Easy, Medium, Hard, or platform-specific equivalent |
| `topic_tags` | Topic tags for analysis |

For LeetCode, `problem_slug` can be matched against CrackedIn's existing problem slug data.

For future platforms, `platform + problem_slug` should be treated as the platform-specific problem identity.

Do not require a submission to match the existing `problems` table during ingestion. Missing matches should not block sync.

---

# 12. Sync Integration With CrackedIn

## 12.1 Account Connection

When a user connects LeetCode:

1. The user is already authenticated in CrackedIn.
2. The extension checks the logged-in LeetCode identity.
3. CrackedIn creates or updates a `user_coding_profiles` row.
4. The row links the CrackedIn user to the LeetCode account.
5. Sync status is initialized.
6. Historical sync starts.

The existing CrackedIn user remains the parent account.

---

## 12.2 Historical Sync Integration

Historical sync should use the profile row as the sync anchor.

Flow:

1. `user_coding_profiles` identifies the connected platform account.
2. Extension walks historical submission metadata.
3. Backend upserts each submission into `user_coding_submissions`.
4. Extension fetches official details and code for every submission.
5. Backend updates the same submission rows with code and judge details.
6. Profile sync state is updated throughout.
7. When complete, the profile row enters a completed or live-enabled state.

---

## 12.3 Live Capture Integration

Live capture uses the same submission table.

Flow:

1. User submits code on LeetCode.
2. Extension captures client-side code and context.
3. Extension detects the official submission ID.
4. Backend upserts a `user_coding_submissions` row.
5. Extension fetches official details.
6. Backend updates the same row with server-confirmed code and judge result.
7. Profile `last_live_capture_at` is updated.

This keeps historical and live submissions in one unified corpus.

---

## 12.4 Delta Repair Integration

Delta repair also writes to the same submission table.

Flow:

1. Extension or backend checks recent platform activity.
2. Missing submission IDs are identified.
3. Missing rows are inserted or updated.
4. Official details are fetched.
5. Profile `last_delta_sync_at` is updated.

This ensures missed live events are eventually captured.

---

# 13. Relationship With Existing CrackedIn Tables

## 13.1 `users`

`users` remains the source of truth for CrackedIn identity.

The new tables attach to `users.id`.

No separate coding-user identity should be created.

---

## 13.2 `users.leetcode_username`

This field can remain for compatibility, display, or migration.

The primary external-account mapping should be `user_coding_profiles`, because it supports multiple platforms and richer sync metadata.

---

## 13.3 `problems`

The existing problem table can support LeetCode joins by slug or LeetCode number.

The submission table should not depend on this join during ingestion. It should store enough problem metadata on the submission row itself.

---

## 13.4 `user_solved_problems`

This table can remain as a solved-status table.

It can optionally be updated from accepted submissions after full sync.

It should not become the main corpus because it cannot store:

- failed attempts,
- multiple attempts,
- submitted code,
- runtime,
- memory,
- language,
- testcase details,
- compile/runtime errors.

---

## 13.5 LeetCode Summary Tables

Existing LeetCode profile, topic, and contest summary tables can remain useful for dashboard views.

The new submission corpus can eventually refresh, validate, or enrich those summaries.

---

# 14. AI Data Usage

The AI layer should use the CrackedIn database instead of calling external platforms directly.

The main AI input table will be `user_coding_submissions`.

It provides:

| Data type | Examples |
|---|---|
| Code history | Every submitted code snapshot |
| Attempt history | Multiple attempts per problem |
| Failure patterns | WA, TLE, RE, CE, MLE |
| Topic patterns | Arrays, DP, graph, linked list |
| Difficulty patterns | Easy, Medium, Hard |
| Language usage | C++, Java, Python, JavaScript |
| Performance data | Runtime, memory, percentiles |
| Timeline data | Submission timestamps |
| Improvement data | Failed attempts before accepted attempts |
| Stuck problems | Tried but never accepted |
| Recent activity | Live-captured submissions |

This report does not define the recommendation algorithm. It defines the data structure required to support future recommendations.

The AI should be able to ask structured questions such as:

- What has this user submitted recently?
- Which topics have the most failed attempts?
- Which problems took many attempts before acceptance?
- Which language does the user use most?
- Which accepted submissions should be reviewed?
- Which failed submissions are still unresolved?
- What code did the user write for a specific problem?
- How did the user's code evolve across attempts?

---

# 15. Recommended Indexing Strategy

The following indexes are recommended for MVP.

## 15.1 `user_coding_profiles`

| Index | Purpose |
|---|---|
| `user_id` | Fetch all connected coding profiles for a user |
| `platform + platform_user_id` | Find an external account |
| `platform + platform_username` | Find or reconnect a platform profile |
| `sync_status` | Find profiles needing sync/retry |

## 15.2 `user_coding_submissions`

| Index | Purpose |
|---|---|
| `user_id` | Fetch all submissions for a user |
| `coding_profile_id` | Fetch all submissions for one connected profile |
| `platform + platform_submission_id` | Enforce uniqueness and safe upsert |
| `problem_slug` | Fetch all attempts for one problem |
| `verdict` | Analyze accepted vs failed attempts |
| `language` | Analyze language usage |
| `submitted_at` | Timeline and recent-activity queries |
| `capture_source` | Debug historical/live/delta ingestion |

The most important uniqueness rule is:

```text
platform + platform_submission_id
```

---

# 16. Data Retention, Privacy, and Controls

Because CrackedIn stores code, the data should be treated as sensitive user data.

The schema should support:

| User control | Required database behavior |
|---|---|
| Disconnect platform | Update `user_coding_profiles.is_connected` |
| Pause sync | Update `user_coding_profiles.sync_enabled` |
| Resume sync | Continue from `sync_cursor` |
| Delete imported data | Delete rows by `user_id` and/or `coding_profile_id` |
| Export data | Query profile and submissions by `user_id` |
| Audit source | Use `capture_source`, timestamps, and sanitized raw metadata |

The system should not store:

- LeetCode passwords,
- LeetCode cookies,
- raw browser request headers,
- unrelated network traffic,
- unrelated page HTML.

Only user-authorized coding data should be stored.

---

# 17. Migration Plan

The migration should be additive and low risk.

## Step 1: Add `user_coding_profiles`

Add the connected coding-profile table.

## Step 2: Add `user_coding_submissions`

Add the full submission corpus table.

## Step 3: Backfill Existing LeetCode Username Where Available

For users who already have a LeetCode username stored on their CrackedIn profile, optionally create a LeetCode coding-profile row when they connect or sync.

## Step 4: Add Backend Ingestion Endpoints

The extension should send cleaned data to backend endpoints that create/update profile and submission records.

## Step 5: Keep Existing Summary Tables

Existing solved-problem and profile-summary flows should keep working.

## Step 6: Gradually Move AI Features to the New Corpus

New AI features should read from `user_coding_submissions`.

Existing dashboards can be migrated gradually.

---

# 18. Future Expansion

The two-table structure is the starting model for the coding corpus.

When needed, CrackedIn can add specialized tables such as:

| Future table | Purpose |
|---|---|
| `coding_problems` | Normalize problem metadata across platforms |
| `user_coding_submission_code` | Move large code blobs out of the main submission table |
| `coding_sync_runs` | Track every sync job and batch |
| `coding_analysis_cache` | Cache expensive AI summaries |
| `coding_platform_events` | Store low-level live-capture/debug events |

These should be introduced only when they solve a real product, performance, or observability problem.

---

# 19. Final Recommendation

CrackedIn should implement the coding-corpus database layer with:

1. `user_coding_profiles`
2. `user_coding_submissions`

This gives the product a practical, extensible foundation for:

- full LeetCode historical sync,
- all historical code storage,
- future live submission capture,
- periodic repair of missed submissions,
- platform expansion,
- AI-ready user coding analysis.

The final implementation principle is:

> CrackedIn's existing user remains the parent identity. Each external coding account becomes a connected profile. Every submitted attempt becomes a submission record. The AI layer reads from the stored CrackedIn corpus, not directly from external coding platforms.
