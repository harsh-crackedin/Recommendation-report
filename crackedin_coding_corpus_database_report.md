# CrackedIn Coding Corpus Database Report

## 1. Purpose

This report defines the MVP database design for storing coding-platform activity inside **CrackedIn**.

The selected approach is **Option A**, which adds only two new tables:

1. `user_coding_profiles`
2. `user_coding_submissions`

These two tables will connect external coding-platform data to the existing CrackedIn user system, starting with LeetCode and later supporting other platforms such as GFG, Codeforces, coding courses, and internal practice environments.

The focus of this report is database structure, integration with the current CrackedIn schema, and how the stored data will become usable for AI-driven user analysis and recommendations.

---

## 2. Existing CrackedIn Database Context

CrackedIn already has a user-centered schema. The existing user table stores the main product identity: email, password hash, display name, LeetCode username, target companies, target role, account tier, authentication provider, and creation metadata.

CrackedIn also already has problem-level and LeetCode-summary tables. These are useful for dashboards and existing product features, but they do not store full attempt-level history.

The current schema can represent things like:

- the user account,
- target companies,
- target role,
- known LeetCode username,
- shared problem metadata,
- solved-problem relationships,
- LeetCode profile summaries,
- topic summaries,
- contest summaries.

However, the current schema does not fully represent:

- every submitted attempt,
- failed submissions,
- multiple attempts on the same problem,
- historical code snapshots,
- live future code snapshots,
- runtime and memory per attempt,
- testcase details per attempt,
- compile/runtime error details,
- source of capture, such as historical sync, live capture, or delta repair.

Therefore, the new extension data should **extend** the current CrackedIn user model instead of replacing it.

The correct integration model is:

| Existing CrackedIn area | Current role | New extension data role |
|---|---|---|
| `users` | Main CrackedIn user identity | Parent user record |
| `problems` | Shared LeetCode problem metadata | Optional join target for LeetCode submissions |
| `user_solved_problems` | Basic solved-problem status | Can be updated from submission history |
| `user_lc_profiles` | Aggregate LeetCode profile snapshot | Can be refreshed from richer sync data |
| `user_lc_topics` | Topic-level summary | Can remain a fast summary table |
| `user_contest_history` | Contest-level history | Remains separate from submission corpus |
| New `user_coding_profiles` | External platform account connection | Links user to LeetCode/GFG/etc. |
| New `user_coding_submissions` | Full attempt-level corpus | Stores code, verdicts, metadata, and analysis inputs |

---

## 3. Why Option A Is Best for MVP

Option A uses two tables instead of a more normalized three- or four-table design.

This is better for MVP because the first goal is not perfect long-term normalization. The first goal is to reliably capture, store, and use the user's complete coding corpus.

A larger design would separate platform profiles, problems, submissions, code snapshots, sync jobs, and analytics summaries into separate tables. That may become useful later, but it adds complexity before the capture pipeline is fully stabilized.

For MVP, the two-table design is better because:

| Reason | Why it matters |
|---|---|
| Faster implementation | Fewer migrations, fewer joins, simpler ingestion |
| Easier debugging | One table for platform account, one table for submissions |
| Supports full historical code storage | Code can be stored directly with each submission |
| Works with existing CrackedIn users | No new auth/user model required |
| Supports multiple platforms | Platform-specific details can live in flexible metadata fields |
| Good enough for AI analysis | All code, verdict, topic, language, and timestamp data are queryable |
| Easy to normalize later | Can split problems/code/sync-runs once usage grows |

The larger normalized schema should remain a future scale-up path, not the MVP starting point.

---

## 4. Chosen MVP Schema Overview

The two new tables are:

1. `user_coding_profiles`
2. `user_coding_submissions`

### 4.1 `user_coding_profiles`

This table represents an external coding account connected to a CrackedIn user.

Examples:

| CrackedIn user | Platform | External username |
|---|---|---|
| User 1 | LeetCode | harshsrivastava05 |
| User 1 | GFG | harsh05 |
| User 1 | Codeforces | harsh_cf |

A user can connect one or more external coding profiles over time.

This table should answer:

- Which coding accounts has this CrackedIn user connected?
- Which platform does each account belong to?
- What is the external username or user ID?
- Is sync currently enabled?
- What phase is sync in?
- When was the last successful sync?
- Where should sync resume if interrupted?
- What permissions/settings did the user choose?

### 4.2 `user_coding_submissions`

This table stores the actual coding corpus.

Every row represents one submitted attempt from LeetCode or another platform.

This table should answer:

- What problem did the user attempt?
- What code did they submit?
- Which language did they use?
- Was the result accepted or failed?
- What kind of failure happened?
- What was the runtime and memory usage?
- When was it submitted?
- Was it captured historically, live, or through delta repair?
- What raw platform metadata is available for future analysis?

This table is the core AI analysis dataset.

---

# 5. Table 1: `user_coding_profiles`

## 5.1 Purpose

`user_coding_profiles` links a CrackedIn user to one external coding-platform identity.

This table is needed because the existing `users` table should not be expanded every time a new coding platform is added.

The existing `users.leetcode_username` field can remain useful for backward compatibility and display, but the new platform-agnostic table should become the primary connection model for extension-based sync.

## 5.2 Recommended Fields

| Field | Purpose |
|---|---|
| `id` | Internal primary key for this connected coding profile |
| `user_id` | References the existing CrackedIn user |
| `platform` | External platform name, such as `leetcode`, `gfg`, `codeforces`, `code_course` |
| `platform_user_id` | External platform's user ID if available |
| `platform_username` | External platform username or handle |
| `platform_display_name` | External profile display name if different from username |
| `platform_profile_url` | Public or user-facing profile URL |
| `is_connected` | Whether this account is currently connected |
| `sync_enabled` | Whether automatic sync is enabled |
| `sync_mode` | Sync mode; MVP default should be `full_archive` |
| `sync_status` | Current status: idle, syncing, paused, failed, complete |
| `last_full_sync_at` | Last completed historical sync timestamp |
| `last_delta_sync_at` | Last periodic repair sync timestamp |
| `last_live_capture_at` | Last successful live-captured submission timestamp |
| `sync_cursor` | Resumable sync state, stored as structured JSON/text |
| `sync_error` | Last sync error message, if any |
| `settings` | User/platform-specific sync settings |
| `raw_profile` | Sanitized raw profile metadata from the platform |
| `created_at` | Row creation timestamp |
| `updated_at` | Last update timestamp |

---

## 5.3 Field Behavior

### `user_id`

This is the main integration point with CrackedIn.

Every coding profile must belong to one existing CrackedIn user.

Relationship:

| Parent | Child |
|---|---|
| `users.id` | `user_coding_profiles.user_id` |

This ensures there is no duplicate user identity system.

### `platform`

This makes the table future-proof.

Initial value:

| Platform | Meaning |
|---|---|
| `leetcode` | LeetCode synced through browser extension |

Future values:

| Platform | Meaning |
|---|---|
| `gfg` | GeeksforGeeks |
| `codeforces` | Codeforces |
| `code_course` | Internal or external coding course platform |
| `crackedin_practice` | Native CrackedIn coding environment |

The schema should not assume every platform behaves like LeetCode.

### `platform_user_id`

For LeetCode, this comes from the authenticated `userStatus` response.

This is useful because usernames can sometimes change, while numeric or internal platform IDs are often more stable.

### `platform_username`

For LeetCode, this is the username shown in the user's profile.

This is useful for display, logs, support, and manual debugging.

### `sync_mode`

For MVP, the selected product behavior is:

| Mode | Meaning |
|---|---|
| `full_archive` | Store code for every historical submission |

Even though MVP will default to full archive, keeping this field is useful because future modes may include:

| Future mode | Meaning |
|---|---|
| `metadata_only` | Store attempts without code |
| `smart_code` | Store selected high-signal code only |
| `full_archive` | Store all historical and future code |

### `sync_status`

This field helps the product and extension show the user what is happening.

Recommended statuses:

| Status | Meaning |
|---|---|
| `not_started` | User connected but has not started sync |
| `metadata_syncing` | Historical metadata is being fetched |
| `code_syncing` | Historical code snapshots are being fetched |
| `live_enabled` | Historical sync is done and live capture is active |
| `delta_syncing` | Periodic repair is running |
| `paused` | User or system paused sync |
| `failed` | Last sync failed |
| `complete` | Full historical sync completed |

### `sync_cursor`

This is important because browser extension sync can be interrupted.

The cursor should store resumable state such as:

| Cursor item | Purpose |
|---|---|
| Current historical offset | Resume `submissionList` pagination |
| Current code-fetch position | Resume full code archive |
| Remaining submission IDs | Continue after service worker shutdown |
| Phase name | Know whether sync is in metadata, code, live, or delta phase |
| Last processed submission ID | Debugging and recovery |
| Retry count | Avoid infinite loops |

This allows the extension and backend to continue from the last safe point.

---

# 6. Table 2: `user_coding_submissions`

## 6.1 Purpose

`user_coding_submissions` stores every coding attempt.

For LeetCode, every row represents one LeetCode submission ID.

This includes:

- historical submissions,
- future live submissions,
- accepted attempts,
- failed attempts,
- compile errors,
- runtime errors,
- time limit exceeded attempts,
- multiple attempts on the same problem.

This is the most important table for AI analysis.

---

## 6.2 Recommended Fields

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
| `code_hash` | Hash of canonical code for dedup/change detection |
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
| `capture_source` | Historical, live, or delta source |
| `is_historical_imported` | Whether this came from historical sync |
| `is_live_captured` | Whether this came from live session capture |
| `raw_submission_meta` | Sanitized raw metadata from submission-list response |
| `raw_detail_response` | Sanitized raw detail response from platform |
| `created_at` | Row creation timestamp |
| `updated_at` | Last update timestamp |

---

## 6.3 Required Unique Constraint

The table must prevent duplicate submissions.

The natural uniqueness rule is:

| Constraint | Purpose |
|---|---|
| `platform + platform_submission_id` | Ensures one row per external submission |

This matters because the same submission can be discovered multiple ways:

- historical sync,
- live capture,
- periodic delta repair,
- manual re-sync,
- retry after failed upload.

All ingestion should use upsert behavior, not blind insert behavior.

---

## 6.4 Why Store Both `user_id` and `coding_profile_id`

Technically, `coding_profile_id` can lead back to the user through `user_coding_profiles`.

However, storing `user_id` directly in `user_coding_submissions` is still recommended.

Reason:

| Benefit | Explanation |
|---|---|
| Faster AI queries | Most analysis starts with “all submissions for this user” |
| Simpler application logic | Backend does not need to join for every common query |
| Easier authorization | API can verify `submission.user_id` directly |
| Better indexing | User-level submission queries become straightforward |

The ingestion layer should ensure the `user_id` matches the owner of the `coding_profile_id`.

---

# 7. Problem Metadata Strategy

CrackedIn already has a problem table for LeetCode-style problem metadata.

For LeetCode submissions, `problem_slug` in `user_coding_submissions` can be used to connect to the existing problem metadata.

However, for MVP, this connection should be optional rather than mandatory.

Reason:

- LeetCode data can be matched by slug.
- Other platforms may not share the same problem model.
- GFG and coding courses may have different IDs, URLs, and topic structures.
- Some submitted code may come from non-LeetCode environments.
- Requiring a problem-row match during ingestion can make sync fragile.

Recommended MVP approach:

| Field | Purpose |
|---|---|
| `problem_slug` | Primary platform-level problem key |
| `problem_title` | Display and AI context |
| `problem_external_id` | Platform-specific problem ID |
| `problem_frontend_id` | Display number, such as LeetCode number |
| `problem_url` | Direct link |
| `difficulty` | Useful for analysis |
| `topic_tags` | Useful for recommendations |

Later, a separate `coding_problems` table can be added if multi-platform problem normalization becomes important.

---

# 8. Code Storage Strategy

Since the product decision is to store all historical code, `user_coding_submissions` should include code fields directly in MVP.

The table should distinguish three code-related concepts:

| Field | Meaning |
|---|---|
| `client_captured_code` | Code captured during live submit/editor flow |
| `server_confirmed_code` | Code returned by official platform details endpoint |
| `code` | Canonical code used by CrackedIn |

For historical sync, the canonical code will usually be the server-confirmed code from LeetCode submission details.

For live capture, the extension may first store client-captured code, then update the row after server confirmation arrives.

Recommended canonical rule:

| Situation | Canonical `code` value |
|---|---|
| Historical sync | Use server-confirmed code |
| Live capture before judge result | Temporarily use client-captured code |
| Live capture after details fetch | Replace or confirm with server-confirmed code |
| Client/server mismatch | Prefer server-confirmed code, keep client code for debugging |
| Detail fetch fails | Keep client code and mark record for delta repair |

This gives CrackedIn both speed and correctness.

---

# 9. Verdict and Status Strategy

The database should store both raw platform status and normalized verdict.

## 9.1 Raw Status

`raw_status` preserves exactly what the platform returned.

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

This is useful for debugging and future platform-specific handling.

## 9.2 Normalized Verdict

`verdict` should use a smaller internal set.

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
| `OTHER` | Unknown or unsupported status |

This makes analytics easier across platforms.

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

This lets CrackedIn understand data provenance.

A row may have multiple boolean flags too:

| Field | Meaning |
|---|---|
| `is_historical_imported` | Seen during historical sync |
| `is_live_captured` | Seen during live browser capture |

A submission can technically be both live-captured and later seen in delta repair, so `capture_source` should represent original source while booleans or metadata can reflect later reconciliation.

---

# 11. Sync Integration With CrackedIn

## 11.1 Account Connection

When the user connects LeetCode through the extension:

1. The extension checks the logged-in LeetCode user.
2. CrackedIn maps that external account to the current logged-in CrackedIn user.
3. A row is created or updated in `user_coding_profiles`.
4. The profile's `platform` is set to `leetcode`.
5. The profile's sync mode is set to `full_archive`.
6. Historical sync begins.

The current CrackedIn `users` table remains the root identity.

## 11.2 Historical Sync Integration

During historical sync:

1. Extension collects all LeetCode submission IDs and metadata.
2. CrackedIn creates or updates rows in `user_coding_submissions`.
3. Each row is linked to the existing user and the connected coding profile.
4. The extension fetches full details and code for every submission.
5. Each submission row is updated with canonical code and judge details.
6. `user_coding_profiles` stores progress and sync status.

The existing solved-problem relationship can optionally be updated from accepted submissions after sync, but it should not be the primary storage for the new corpus because it cannot represent full attempt history.

## 11.3 Live Capture Integration

During live capture:

1. User submits code on LeetCode.
2. Extension captures client-side code and submission context.
3. Extension detects or receives the platform submission ID.
4. CrackedIn creates or updates a `user_coding_submissions` row.
5. The row initially may contain client-captured code.
6. Extension fetches official submission details.
7. CrackedIn updates the same row with server-confirmed code, verdict, runtime, memory, and testcase details.

This requires idempotent upsert behavior because the same submission may be seen more than once.

## 11.4 Delta Repair Integration

Periodic delta repair should:

1. Check recent submissions from the platform.
2. Compare platform submission IDs against `user_coding_submissions`.
3. Insert missing rows.
4. Fetch missing code/details.
5. Update `user_coding_profiles.last_delta_sync_at`.

Delta repair protects the system from missed live submissions.

---

# 12. Relationship to Existing CrackedIn LeetCode Tables

The existing CrackedIn schema already includes LeetCode-oriented summary data. The new two-table design should coexist with it.

## 12.1 `users.leetcode_username`

This can remain as a convenience/display field, but the canonical external platform mapping should move to `user_coding_profiles`.

Reason:

- `users.leetcode_username` supports only one platform-specific field.
- `user_coding_profiles` supports many platforms.
- A user may later connect multiple external coding identities.

## 12.2 `user_lc_profiles`

This table stores LeetCode profile summary data such as total solved, easy/medium/hard solved, submissions, contest rating, streak, active years, languages, beats stats, failed counts, ranking, and reputation.

It should remain useful for profile-level dashboards.

The new `user_coding_submissions` table is more granular and can eventually be used to refresh or validate this profile summary.

## 12.3 `user_lc_topics`

This table stores per-topic solve counts.

It can remain as a fast dashboard/summary table.

The new submission corpus will provide deeper inputs, including failed attempts and code quality by topic.

## 12.4 `user_contest_history`

This table stores contest-level performance.

It does not overlap much with submission-level code storage, so it should remain separate.

## 12.5 `problems`

The existing `problems` table can be used for LeetCode joins by slug or LeetCode number.

However, the new submission table should not require every record to match this table because future platforms may not use the same problem universe.

---

# 13. AI Data Use

The AI recommendation system will use `user_coding_submissions` as the user's real coding-history corpus.

This table gives the AI structured access to:

| Data type | Examples |
|---|---|
| Code history | Every submitted code snapshot |
| Attempt history | Multiple attempts per problem |
| Failure patterns | WA, TLE, RE, CE, MLE |
| Topic patterns | Arrays, DP, graph, linked list, etc. |
| Difficulty patterns | Easy, Medium, Hard |
| Language usage | C++, Java, Python, etc. |
| Performance data | Runtime, memory, percentiles |
| Timeline data | Submission timestamps |
| Improvement data | Failed attempt before accepted solution |
| Stuck problems | Tried but never accepted |
| Fresh activity | Live-captured recent submissions |

The AI layer should not need to call LeetCode directly. It should consume the normalized CrackedIn database.

The database should give the AI enough information to answer questions such as:

- What has this user solved?
- What has this user failed repeatedly?
- Which topics cause the most wrong answers?
- Which problems required many attempts?
- How did the user's code change before acceptance?
- Which language does the user perform best in?
- What recently changed in the user's activity?
- Which submissions should be reviewed next?

This report does not define the recommendation algorithm. It defines the storage structure that makes those recommendations possible.

---

# 14. Recommended Indexing Strategy

For MVP, indexes should support the most common user-level and submission-level reads.

Recommended indexed fields:

| Field or combination | Why |
|---|---|
| `user_coding_profiles.user_id` | Get all connected platforms for user |
| `user_coding_profiles.platform + platform_user_id` | Find external account |
| `user_coding_submissions.user_id` | Get all user submissions |
| `user_coding_submissions.coding_profile_id` | Get submissions for one external profile |
| `user_coding_submissions.platform + platform_submission_id` | Prevent duplicates and upsert safely |
| `user_coding_submissions.problem_slug` | Show problem history |
| `user_coding_submissions.verdict` | Analyze accepted/failed patterns |
| `user_coding_submissions.language` | Analyze language usage |
| `user_coding_submissions.submitted_at` | Timeline and recency |
| `user_coding_submissions.capture_source` | Debug sync source |

The most important unique index is:

| Unique key | Purpose |
|---|---|
| `platform + platform_submission_id` | One canonical row per external submission |

---

# 15. Data Retention and Privacy Notes

Because the selected MVP stores all historical code, privacy controls matter.

The schema should support:

| Requirement | Schema support |
|---|---|
| Disconnect platform | `user_coding_profiles.is_connected` |
| Pause sync | `user_coding_profiles.sync_enabled` |
| Delete imported data | Delete rows by `user_id` and/or `coding_profile_id` |
| Export data | Query submissions by `user_id` |
| Audit source | `capture_source`, timestamps, raw sanitized metadata |
| Avoid credential storage | Do not store cookies, passwords, or raw request headers |

Code is sensitive user data. It should be treated as part of the user's private CrackedIn corpus.

---

# 16. Migration Plan

The migration should be additive.

No current table needs to be removed.

## Step 1: Add `user_coding_profiles`

Create the external coding-account connection layer.

## Step 2: Add `user_coding_submissions`

Create the submission corpus layer.

## Step 3: Map Existing LeetCode Username

For users with an existing `users.leetcode_username`, CrackedIn can optionally create a `user_coding_profiles` row for LeetCode.

## Step 4: Update Extension Ingestion

The extension should write to the new tables through backend APIs.

## Step 5: Keep Existing Summary Tables

Existing summary tables should continue to support dashboards and legacy flows.

## Step 6: Gradually Move Analytics to the New Corpus

New AI features should query `user_coding_submissions`.

Existing solved-problem/profile summary features can remain as they are until replaced.

---

# 17. Future Normalization Path

The two-table model is enough for MVP, but it should be designed so future normalization is easy.

Possible future tables:

| Future table | When to add |
|---|---|
| `coding_problems` | When multi-platform problem normalization becomes important |
| `user_coding_submission_code` | When code blobs become too large for main submission table |
| `coding_sync_runs` | When detailed sync observability is needed |
| `coding_analysis_cache` | When AI summaries need caching |
| `coding_platform_events` | When event-level live capture debugging is needed |

The MVP should not start with these unless a clear performance or product need appears.

---

# 18. Final Recommendation

CrackedIn should add two tables for MVP:

1. `user_coding_profiles`
2. `user_coding_submissions`

This design is the right balance between speed, flexibility, and future scalability.

It integrates cleanly with the existing CrackedIn `users` table, complements the existing LeetCode profile and solved-problem tables, and gives the AI layer access to the full code-submission corpus.

The core design principle is:

> Keep CrackedIn's current user system as the source of truth, attach external coding profiles to it, and store every coding attempt as a normalized submission record.

This gives the product enough structure to support full LeetCode historical sync, live future capture, periodic repair, and later expansion into GFG, Codeforces, code courses, and internal CrackedIn practice environments.
