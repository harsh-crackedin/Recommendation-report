# CrackedIn / InterviewPrep.ai LeetCode Access — Global Context Summary

**Project context name:** LeetCode Access  
**Product name:** CrackedIn / InterviewPrep.ai  
**Primary artifact purpose:** Global context document for future AI/chat sessions  
**Current platform target:** Chrome/Chromium extension first  
**Primary coding platform target:** LeetCode first  
**Long-term platform direction:** LeetCode → Codeforces → GFG → code courses → CrackedIn native practice  
**Current strategic decision:** Build a browser extension that imports and continuously updates a user's LeetCode coding-submission corpus into CrackedIn, connected to the user's existing CrackedIn account.

---

## 1. What We Are Building

CrackedIn is building a browser extension linked to the existing InterviewPrep.ai / CrackedIn backend. The extension imports a user's real LeetCode submission history and keeps it updated over time.

The product is not just a tracker. It is a corpus-building system for AI-powered interview preparation.

The extension will collect:

- historical LeetCode submissions,
- accepted submissions,
- failed submissions,
- language used,
- verdict/status,
- submitted code snapshots,
- runtime and memory details,
- testcase/error details when available,
- problem slug/title/difficulty/topics when available,
- future submissions made after onboarding.

This imported dataset becomes the user's **coding corpus** inside CrackedIn.

The AI layer can later use this corpus for:

- old solution review,
- weakness detection,
- topic-pattern analysis,
- code-style critique,
- repeated mistake detection,
- spaced repetition suggestions,
- before/after improvement analysis,
- identifying stuck problems,
- personalized interview prep paths,
- comparing failed attempts to final accepted solutions.

The main goal is to give CrackedIn access to the user's real coding history rather than relying only on manually entered progress.

---

## 2. Final Product Positioning

The selected product wedge is:

> Capture the user's complete LeetCode solve graph and submitted code corpus, then build AI features on top of that corpus.

The project is positioned around three concepts:

| Concept | Meaning |
|---|---|
| Capture | Collect historical and future coding submissions |
| Corpus | Store submitted code, attempts, verdicts, and metadata in CrackedIn |
| AI | Use the stored corpus to generate personalized analysis and guidance |

The key product difference is that CrackedIn is not only helping on the current LeetCode page. It is building a persistent, user-owned dataset of all past and future coding attempts.

---

## 3. Core Architecture Summary

The final architecture is a hybrid browser-extension and backend ingestion system.

```text
User installs extension
  ↓
User connects existing CrackedIn account
  ↓
Extension verifies LeetCode login in same browser profile
  ↓
User confirms LeetCode account
  ↓
Extension performs historical backfill
  ↓
Extension uploads sanitized submission batches
  ↓
Backend validates, normalizes, deduplicates, and stores
  ↓
Live capture keeps future submissions updated
  ↓
Periodic delta repair catches missed submissions
  ↓
CrackedIn AI uses stored corpus
```

The major system components are:

| Component | Main Responsibility |
|---|---|
| Extension popup | User actions, status display, progress UI |
| Extension service worker | Historical sync, backend upload, sync state, retries, delta repair |
| Content script | Live LeetCode page observation and submit detection |
| `chrome.storage.local` | Local resumable sync state |
| CrackedIn backend | Authentication, validation, normalization, canonical storage |
| CrackedIn database | Persistent user coding corpus |
| CrackedIn AI layer | Analysis over stored corpus, not direct LeetCode calls |

---

## 4. Identity Model

There are two separate identities in this project:

1. **CrackedIn / InterviewPrep.ai identity**
2. **LeetCode identity**

They must not be confused.

### 4.1 CrackedIn Identity

CrackedIn identity is the primary product identity.

It is represented by the existing CrackedIn backend user, usually `users.id`.

This identity owns:

- account settings,
- dashboard,
- subscription/billing if applicable,
- imported coding corpus,
- AI analysis,
- connected coding profiles.

### 4.2 LeetCode Identity

LeetCode identity is an external coding profile connected to the CrackedIn user.

It is represented by:

- LeetCode username,
- LeetCode user ID if available,
- LeetCode profile metadata.

LeetCode should never become the primary CrackedIn user identity.

### 4.3 Final Identity Mapping

The selected mapping is:

```text
CrackedIn users.id
  ↓
user_coding_profiles
  ↓
platform = leetcode
  ↓
LeetCode username / LeetCode user ID
  ↓
user_coding_submissions
```

This allows one CrackedIn user to later connect multiple coding platforms.

---

## 5. Chosen Extension Authentication Flow

The selected authentication flow is:

> Continue with CrackedIn

The extension should authenticate against the existing CrackedIn / InterviewPrep.ai backend account. It should not create a separate extension-only account.

### 5.1 User Flow

The chosen onboarding flow is:

1. User installs the extension.
2. User opens the extension popup.
3. Popup shows **Continue with CrackedIn**.
4. Extension opens the CrackedIn web app auth/connection page.
5. If user is already logged into CrackedIn, show a confirmation screen.
6. If not logged in, user logs in or signs up through the normal CrackedIn flow.
7. Backend issues a short-lived one-time authorization code.
8. Extension exchanges that code for extension-scoped tokens.
9. Extension stores CrackedIn extension tokens locally.
10. Extension calls backend `/me` endpoint to confirm the user.
11. Extension checks LeetCode login state in the browser.
12. Extension shows the detected LeetCode account.
13. User confirms and starts sync.

Recommended UX sequence:

```text
Connect CrackedIn
↓
Connect LeetCode
↓
Start Sync
```

### 5.2 Token Model

The extension should store only CrackedIn extension-auth tokens.

Recommended token model:

| Token | Stored In | Purpose |
|---|---|---|
| Extension access token | `chrome.storage.local` | Authorize short-lived backend API calls |
| Extension refresh token | `chrome.storage.local` | Refresh extension access token |
| One-time auth code | Backend flow only | Exchange during connection |
| LeetCode session cookie | Browser cookie jar only | Used automatically by browser for LeetCode requests |

The extension must not upload the LeetCode session cookie to CrackedIn.

### 5.3 Extension Token Scope

Extension tokens should be scoped and limited. They should not act like a full web-app session.

Recommended extension permissions/scopes:

- read current CrackedIn user,
- connect coding profile,
- update coding profile sync state,
- upload/upsert own coding submissions,
- read own sync status,
- disconnect own coding profile,
- delete/export own imported coding data.

The extension should not receive permissions for:

- billing changes,
- password changes,
- account deletion,
- admin operations,
- unrelated user data.

---

## 6. LeetCode Authentication and Trust Model

LeetCode authentication uses the user's existing logged-in browser session.

The user logs into LeetCode normally. The extension then makes requests to `https://leetcode.com/graphql` from the browser/extension context using browser credentials.

The browser attaches the existing LeetCode cookies automatically because the request goes to LeetCode.

The extension should not:

- ask the user to paste `LEETCODE_SESSION`,
- read the LeetCode cookie directly,
- send the LeetCode cookie to CrackedIn,
- store the LeetCode password,
- store raw browser request headers.

This is the core trust model:

```text
CrackedIn token → sent only to CrackedIn backend
LeetCode browser session → used only for LeetCode requests
LeetCode cookie → never sent to CrackedIn
```

Recommended permission posture:

| Permission | Reason |
|---|---|
| `host_permissions: ["https://leetcode.com/*"]` | Allow extension to call LeetCode APIs and observe LeetCode pages |
| `storage` | Store extension auth and sync cursor |
| `alarms` | Periodic delta repair |
| `scripting` if needed | Inject content/live-capture scripts |

Avoid the `cookies` permission unless absolutely necessary. The trust story is stronger if the extension never reads cookies directly.

---

## 7. Confirmed LeetCode Data Sources

The project has identified the correct LeetCode sources and their roles.

| Source | Correct Use | Limitation |
|---|---|---|
| `userStatus` | Check whether user is signed in and get identity | No submission data |
| `submissionList(offset, limit)` | Full historical metadata and submission IDs | Page size appears capped around 20 |
| `submissionDetails(submissionId)` | Canonical submitted code and judge details | Requires submission ID |
| `recentAcSubmissionList(username, limit)` | Recent accepted delta checks | Accepted only; not full history |
| `submissionList(questionSlug)` | Attempts for known problem slug | Requires knowing slug first |
| `/submissions/detail/<id>/check/` | Live submission ID detection during judging | Not a historical details endpoint |
| Editor/submit payload | Early live code capture | Not final judge result |
| `question.codeSnippets` | Starter templates | Not user-submitted code |

Important conclusions:

- Full history must come from `submissionList(offset, limit)`.
- Actual submitted code must come from `submissionDetails(submissionId)`.
- `question.codeSnippets` must never be treated as user-submitted code.
- `/check/` is useful for live ID detection, not historical sync.
- Profile/recent activity is incomplete and cannot be used as the main source.

---

## 8. Historical Sync Strategy

The selected sync mode is:

> Full historical archive

This means CrackedIn should import every historical LeetCode submission and fetch code for every submission, not only accepted solutions.

### 8.1 Historical Sync Goals

Historical sync must import:

- all accepted submissions,
- all wrong answers,
- all time limit exceeded attempts,
- all memory limit exceeded attempts,
- all runtime errors,
- all compile errors,
- all output limit exceeded attempts,
- all internal/other statuses,
- multiple attempts on the same problem,
- tried-but-unsolved problems,
- submitted code for every historical submission.

### 8.2 Historical Sync Flow

The selected flow is:

1. Confirm CrackedIn extension auth.
2. Check LeetCode `userStatus`.
3. If user is not signed into LeetCode, show auth-required state.
4. If signed in, create/update `user_coding_profiles`.
5. Walk `submissionList(offset, limit)` until the end.
6. Store/queue every submission ID and metadata record.
7. Fetch `submissionDetails(submissionId)` for every discovered submission.
8. Send sanitized submission batches to CrackedIn backend.
9. Backend validates and upserts canonical records.
10. Mark historical sync complete.
11. Enable live capture and periodic repair.

### 8.3 Metadata First, Code Second

Historical sync should be staged:

| Phase | What Happens | Why |
|---|---|---|
| Metadata discovery | Fetch submission IDs, slugs, statuses, timestamps | Builds queue and enables progress visibility |
| Code/details sync | Fetch `submissionDetails` for every ID | Gets canonical code and judge data |
| Finalization | Backend marks profile as synced | Enables live capture/dashboard |

This staged design improves resumability and user feedback.

---

## 9. GraphQL Aliasing Strategy

GraphQL aliasing should be used to batch multiple LeetCode GraphQL operations into fewer HTTP requests.

Aliasing is important because LeetCode caps submission pages around 20 items, and a user may have hundreds or thousands of submissions.

Use aliasing for:

- multiple `submissionList` pages in one request,
- multiple `submissionDetails(submissionId)` calls in one request.

Conceptually:

```text
Without aliasing:
  many sequential requests

With aliasing:
  multiple independent fields in one GraphQL request
```

Recommended batching idea:

| Task | Batch Shape |
|---|---|
| Metadata discovery | 10 aliased `submissionList` pages per request |
| Code/details fetch | 10 aliased `submissionDetails` calls per request |

Aliasing is a performance optimization but should be used with backoff, pacing, and retry handling to avoid rate-limit issues.

---

## 10. Data Cleanup and Ingestion Strategy

The selected ingestion model is:

> Extension performs minimum safe sanitization. Backend performs final validation, normalization, deduplication, and storage.

The extension should not dump raw browser JSON directly into the database. The backend should not blindly trust the extension.

### 10.1 Extension Responsibilities

The extension should:

- collect LeetCode API responses,
- select only required fields,
- remove sensitive or unrelated fields,
- remove cookies, request headers, CSRF tokens, and browser metadata,
- package records into clean ingestion batches,
- include capture source,
- preserve sanitized raw fragments where useful.

The extension should send structured payloads containing:

- platform,
- LeetCode username/user ID,
- submission ID,
- problem slug/title,
- language,
- raw status,
- timestamp,
- runtime/memory display,
- testcase counts,
- server-confirmed code,
- client-captured code where applicable,
- sanitized raw metadata/detail fragments.

The extension should never send:

- LeetCode cookies,
- CrackedIn password,
- LeetCode password,
- request headers,
- CSRF tokens,
- full browser request objects,
- unrelated network events,
- unrelated page HTML,
- local/session storage dumps,
- debug logs containing secrets.

### 10.2 Backend Responsibilities

The backend should:

- authenticate the CrackedIn user,
- validate request shape,
- verify connected coding profile,
- enforce ownership,
- normalize verdicts,
- parse runtime and memory values,
- compute code hash,
- upsert profile and submission records,
- deduplicate by platform submission ID,
- reject unsafe payloads,
- store sanitized raw fragments where useful,
- expose delete/export controls.

The backend must treat extension payloads as untrusted client input.

### 10.3 Final Ingestion Principle

```text
Extension:
  collect → sanitize → package → batch upload

Backend:
  authenticate → validate → normalize → dedupe → store
```

---

## 11. Database Strategy

The selected MVP schema adds two new tables:

1. `user_coding_profiles`
2. `user_coding_submissions`

This is the chosen Option A database design.

It is intentionally simple for MVP and can be normalized later.

### 11.1 Existing CrackedIn Tables

The existing CrackedIn system already has user-centered tables, likely including:

- `users`,
- `problems`,
- `user_solved_problems`,
- `user_lc_profiles`,
- `user_lc_topics`,
- `user_contest_history`,
- `chat_sessions`,
- `chat_messages`.

The new coding tables should extend the existing system, not replace it.

### 11.2 `user_coding_profiles`

Purpose:

> Link one CrackedIn user to one external coding account.

Example rows:

| user_id | platform | platform_username |
|---|---|---|
| user 1 | leetcode | leetcode_handle |
| user 1 | codeforces | cf_handle |
| user 1 | gfg | gfg_handle |

Recommended conceptual fields:

| Field | Purpose |
|---|---|
| `id` | Internal primary key |
| `user_id` | Existing CrackedIn user ID |
| `platform` | `leetcode`, later `gfg`, `codeforces`, etc. |
| `platform_user_id` | External platform user ID if available |
| `platform_username` | External username/handle |
| `platform_display_name` | External display name if available |
| `platform_profile_url` | External profile URL |
| `is_connected` | Whether account is connected |
| `sync_enabled` | Whether automatic sync is enabled |
| `sync_mode` | MVP default: `full_archive` |
| `sync_status` | Current sync state |
| `sync_cursor` | Resumable sync state |
| `last_full_sync_at` | Last completed historical sync |
| `last_delta_sync_at` | Last repair sync |
| `last_live_capture_at` | Last live captured event |
| `sync_error` | Last sync error if any |
| `settings` | Platform/user-specific settings |
| `raw_profile` | Sanitized raw profile metadata |
| `created_at` | Creation timestamp |
| `updated_at` | Update timestamp |

Recommended sync status values:

| Status | Meaning |
|---|---|
| `not_started` | Connected but not synced |
| `metadata_syncing` | Historical metadata is being fetched |
| `code_syncing` | Historical code/details are being fetched |
| `uploading` | Data is being sent to backend |
| `finalizing` | Sync is being completed |
| `live_enabled` | Historical sync done; live capture active |
| `delta_syncing` | Periodic repair running |
| `paused` | User/system paused sync |
| `auth_required` | LeetCode login needed |
| `failed` | Last sync failed |
| `complete` | Full historical sync completed |

### 11.3 `user_coding_submissions`

Purpose:

> Store every submitted attempt from connected coding platforms.

For LeetCode, one row equals one LeetCode submission ID.

Recommended conceptual fields:

| Field | Purpose |
|---|---|
| `id` | Internal primary key |
| `user_id` | Existing CrackedIn user ID |
| `coding_profile_id` | Connected external profile ID |
| `platform` | Example: `leetcode` |
| `platform_submission_id` | External submission ID |
| `problem_slug` | Canonical problem slug |
| `problem_title` | Human-readable problem title |
| `problem_external_id` | Platform problem ID if available |
| `problem_frontend_id` | LeetCode display number if available |
| `problem_url` | Direct problem URL |
| `difficulty` | Problem difficulty |
| `topic_tags` | Platform/CrackedIn topic tags |
| `language` | Language slug/name |
| `language_verbose` | Human-readable language |
| `code` | Canonical submitted code |
| `client_captured_code` | Code captured from editor/submit flow |
| `server_confirmed_code` | Code from `submissionDetails` |
| `code_hash` | Hash of canonical code |
| `code_capture_method` | Source/reconciliation method |
| `verdict` | Normalized verdict |
| `raw_status` | Raw platform status |
| `status_code` | Platform status code if available |
| `runtime_ms` | Runtime in milliseconds if parsed |
| `runtime_display` | Original runtime display |
| `runtime_percentile` | Runtime percentile if available |
| `memory_bytes` | Memory in bytes if parsed |
| `memory_display` | Original memory display |
| `memory_percentile` | Memory percentile if available |
| `total_correct` | Passed testcase count |
| `total_testcases` | Total testcase count |
| `last_testcase` | Last/failing testcase if available |
| `code_output` | User output if available |
| `expected_output` | Expected output if available |
| `runtime_error` | Runtime error message |
| `compile_error` | Compile error message |
| `submitted_at` | Platform submission timestamp |
| `captured_at` | CrackedIn capture timestamp |
| `capture_source` | Historical/live/delta/manual |
| `is_historical_imported` | Seen during historical sync |
| `is_live_captured` | Seen during live capture |
| `raw_submission_meta` | Sanitized metadata response |
| `raw_detail_response` | Sanitized detail response |
| `created_at` | Creation timestamp |
| `updated_at` | Update timestamp |

### 11.4 Uniqueness Rule

The backend must enforce uniqueness on:

```text
platform + platform_submission_id
```

This prevents duplicates when a submission is found through multiple capture paths.

### 11.5 Why Store Both `user_id` and `coding_profile_id`

Store both because:

| Field | Why It Matters |
|---|---|
| `coding_profile_id` | Preserves relationship to the external account |
| `user_id` | Makes AI queries, authorization, and user-level retrieval simpler |

The ingestion layer must ensure they match.

---

## 12. Normalization Rules

### 12.1 Platform Names

Use consistent platform names:

- `leetcode`
- `codeforces`
- `gfg`
- `code_course`
- `crackedin_practice`

### 12.2 Problem Identity

For LeetCode, use:

```text
titleSlug
```

as the canonical problem key.

Do not use only `questionFrontendId` as the primary key. It is useful for display but less stable for joins.

For future platforms, use:

```text
platform + problem_slug
```

### 12.3 Submission Identity

Use:

```text
platform + platform_submission_id
```

as the external uniqueness key.

### 12.4 Verdict Normalization

Store both raw and normalized status.

Recommended normalized verdicts:

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
| `OTHER` | Unknown/platform-specific |

### 12.5 Code Fields

The canonical code should usually be:

```text
server_confirmed_code from submissionDetails.code
```

Client-captured code can be stored separately during live capture.

If server-confirmed code and client-captured code differ, prefer server-confirmed code as canonical and keep the difference for debugging if useful.

---

## 13. Service Worker Architecture

The extension's Manifest V3 service worker is the background controller for sync.

The service worker should own anything that must survive popup closing or tab switching.

### 13.1 Service Worker Responsibilities

The service worker should handle:

- CrackedIn extension token usage,
- LeetCode auth check,
- historical metadata sync,
- historical code/details sync,
- backend upload,
- local sync progress,
- sync cursor persistence,
- pause/resume,
- retry handling,
- backend progress mirroring,
- periodic delta repair,
- receiving live capture events from content scripts.

### 13.2 What the Service Worker Should Not Do

The service worker cannot directly access:

- LeetCode page DOM,
- Monaco editor content,
- page-level JavaScript variables.

Those are content-script responsibilities.

### 13.3 Correct Component Ownership

| Task | Owner |
|---|---|
| Historical sync | Service worker |
| Backend upload | Service worker |
| Sync progress updates | Service worker |
| Live submit/page observation | Content script |
| User-facing progress UI | Popup |
| Canonical data storage | Backend |
| AI analysis | Backend/AI layer |

### 13.4 Important Lifecycle Rule

Manifest V3 service workers are not always running. Chrome may stop them when idle.

Therefore:

- do not depend on in-memory state,
- save progress after every batch,
- read saved state when waking,
- design all backend writes as idempotent.

---

## 14. Historical Sync and Tab Behavior

Historical sync should be tab-independent.

It should run from the extension service worker and not depend on the visible LeetCode page.

Expected behavior:

| User Action | Historical Sync | Live Capture |
|---|---|---|
| User stays on LeetCode tab | Continues | Works |
| User switches tab | Continues/resumes | Usually works if LeetCode page remains open |
| LeetCode tab in background | Continues/resumes | Usually works |
| User closes LeetCode tab | Can continue | Stops until page opens again |
| User closes popup | Continues/resumes | Not affected |
| User closes browser | Pauses | Stops |
| Browser reopens | Resumes from cursor | Works again when LeetCode page is open |
| User logs out of LeetCode | Pauses with auth-required state | Stops |

User-facing wording should be:

```text
You can close this popup or switch tabs. Sync will continue in the background while the browser remains open. If the browser closes, sync pauses and resumes later.
```

Do not promise that sync continues while Chrome itself is closed.

---

## 15. Resumable Sync State

Because service workers can be stopped, historical sync must be resumable.

State should be saved in:

- `chrome.storage.local` for local extension recovery and popup progress,
- backend `user_coding_profiles.sync_status` and `sync_cursor` for dashboard/recovery/support.

### 15.1 Local State Should Track

| State | Purpose |
|---|---|
| `phase` | Current sync phase |
| `metadataOffset` | Where metadata pagination resumes |
| `metadataFetched` | Count found so far |
| `totalSubmissions` | Known after metadata discovery |
| `codeQueue` | Submission IDs remaining |
| `codeFetched` | Code/details count completed |
| `codeFailed` | Failed code/detail fetches |
| `uploadedTotal` | Uploaded records count |
| `retryQueue` | Failed records pending retry |
| `pauseRequested` | User/system pause flag |
| `lastProgressAt` | Stale/progress detection |
| `error` | Last known error |

### 15.2 Backend State Should Track

Backend state should track:

- profile sync status,
- phase,
- cursor summary,
- last full sync time,
- last delta sync time,
- last live capture time,
- last error.

Local state can update every batch. Backend state can update less frequently, such as on phase transitions or every several batches.

---

## 16. Progress Tracker Strategy

The selected progress tracker method is:

> State-driven progress bar controlled by service-worker progress state.

The popup should not run sync. It should only display sync state.

### 16.1 Progress Ownership

| Layer | Responsibility |
|---|---|
| Service worker | Calculates and writes progress |
| `chrome.storage.local` | Stores latest progress |
| Popup | Reads and renders progress |
| Backend | Stores durable progress/cursor |

### 16.2 Progress Phases

The popup should support these phases:

| Phase | UI Meaning |
|---|---|
| `auth_checking` | Checking CrackedIn and LeetCode connections |
| `metadata_syncing` | Finding historical submissions |
| `code_syncing` | Fetching submitted code/details |
| `uploading` | Uploading data to CrackedIn |
| `finalizing` | Completing sync |
| `complete` | Historical corpus imported |
| `paused` | Sync paused |
| `auth_required` | LeetCode login required |
| `failed` | Sync failed but may be retryable |

### 16.3 Indeterminate Then Determinate Progress

At the beginning, the exact total submission count may not be known.

So metadata discovery should be displayed as:

```text
Finding your submissions...
1,200 submissions found so far
```

Once metadata discovery finishes, the total is known:

```text
Found 2,143 submissions
Fetching submitted code...
820 / 2,143 submissions processed
```

### 16.4 Suggested Weighted Progress

Simple MVP weighting:

| Phase | Weight |
|---|---:|
| Metadata sync | 35% |
| Code/details sync | 60% |
| Finalization | 5% |

Before total is known, avoid fake exact percentages. Use an indeterminate progress indicator or a capped estimate.

### 16.5 Popup UX Copy

Recommended popup messages:

| State | Message |
|---|---|
| Connecting | Connecting to CrackedIn and LeetCode |
| Finding submissions | Finding your LeetCode submissions |
| Found submissions | Found X submissions |
| Fetching code | Fetching submitted code |
| Uploading | Uploading to CrackedIn |
| Complete | Sync complete. Live capture is now enabled |
| Paused | Sync paused. You can resume anytime |
| Auth required | LeetCode login expired. Log in to continue |
| Failed | Sync paused due to an error. Retry available |

The popup should include:

```text
You can close this popup. Sync will continue in the background.
```

---

## 17. Live Capture Strategy

Live capture handles future submissions after historical sync.

Unlike historical sync, live capture depends on the LeetCode problem page being open.

### 17.1 Live Capture Flow

1. User opens LeetCode problem page.
2. Content script / injected observer is active.
3. User writes and submits code.
4. Extension captures client-side code or submit context.
5. Extension detects final submission ID from judging/check flow.
6. Content script sends event to service worker.
7. Service worker calls `submissionDetails(submissionId)`.
8. Service worker uploads sanitized data to backend.
9. Backend upserts same canonical submission table.

### 17.2 Canonical Code Rule

The canonical code should be:

```text
server-confirmed code from submissionDetails
```

Client-captured code is useful for immediacy and debugging but should not override server-confirmed code unless the official detail fetch is unavailable.

### 17.3 Live Capture Limitations

Live capture can miss submissions if:

- user submits from mobile,
- user submits from another browser,
- extension is disabled,
- LeetCode page is not open,
- content script fails,
- browser is closed,
- network upload fails,
- LeetCode frontend changes.

This is why periodic delta repair is required.

---

## 18. Periodic Delta Repair

Periodic delta repair is the consistency layer.

It catches submissions missed by live capture.

### 18.1 Delta Repair Triggers

Delta repair can run:

- on a timer using `chrome.alarms`,
- when browser starts,
- when popup opens,
- after auth recovery,
- when user manually refreshes/resyncs.

### 18.2 Delta Repair Sources

Use both:

| Source | Why |
|---|---|
| `recentAcSubmissionList` | Catches recent accepted submissions |
| First few pages of `submissionList` | Catches recent failed submissions too |

Accepted-only repair is insufficient because it misses wrong answers, compile errors, TLEs, and other failed attempts.

### 18.3 Delta Repair Flow

1. Service worker wakes.
2. Calls recent accepted/general submission sources.
3. Compares platform submission IDs against known backend/local IDs.
4. Queues missing IDs.
5. Fetches `submissionDetails` for missing IDs.
6. Uploads sanitized records.
7. Backend upserts by `platform + platform_submission_id`.
8. Updates `last_delta_sync_at`.

---

## 19. Progress, Pause, Resume, and Retry

### 19.1 Pause

Pause should be user-controlled and checked between batches.

When user pauses:

- save current cursor,
- stop after the current safe batch,
- mark local and backend status as paused.

### 19.2 Resume

Resume should:

- read saved local cursor,
- verify CrackedIn auth,
- verify LeetCode auth,
- continue from the saved phase and cursor.

### 19.3 Retry

Retry should handle:

- network failures,
- backend upload failures,
- temporary LeetCode errors,
- service worker interruptions.

Retries should use:

- idempotent backend upserts,
- retry queue,
- backoff/jitter for repeated failures,
- safe resumption from last persisted cursor.

---

## 20. Backend API Surface

Minimum backend API concepts:

| Endpoint Concept | Purpose |
|---|---|
| Extension auth exchange | Exchange one-time code for extension tokens |
| Extension token refresh | Refresh access token |
| Extension current user | Confirm connected CrackedIn user |
| Connect coding profile | Create/update LeetCode profile |
| Read coding profiles | Show connected platforms |
| Update sync state | Mirror phase/cursor/status |
| Batch upsert submissions | Upload sanitized historical/live/delta records |
| Disconnect profile | Stop syncing a platform |
| Delete imported data | Remove user's coding corpus |
| Export imported data | Let user download imported corpus |

MVP can combine some endpoints, but the logical responsibilities should remain clear.

---

## 21. User Controls

The product should expose these controls:

| Control | Purpose |
|---|---|
| Continue with CrackedIn | Authenticate extension with existing account |
| Connect LeetCode | Link detected LeetCode account |
| Start sync | Explicitly begin historical import |
| Pause sync | Stop between batches |
| Resume sync | Continue from cursor |
| Manual resync | Run repair/check again |
| Disconnect account | Stop future sync |
| Delete imported data | Remove stored corpus |
| Export imported data | Give user access to stored data |

The extension should not automatically import historical code at install time. The user should explicitly start sync.

---

## 22. Privacy, Consent, and Store Review Posture

Because submitted code is sensitive, the product must be clear and minimal.

### 22.1 Consent Should Say CrackedIn Collects

- LeetCode problem metadata,
- submission IDs,
- submission timestamps,
- verdict/status,
- language,
- runtime/memory,
- testcase/error details when available,
- submitted code.

### 22.2 Consent Should Say CrackedIn Does Not Collect

- LeetCode password,
- LeetCode session cookie,
- raw browser cookies,
- unrelated browsing history,
- unrelated page content,
- unrelated network traffic.

### 22.3 Data Controls

Users should be able to:

- pause syncing,
- disconnect LeetCode,
- delete imported data,
- export imported data.

### 22.4 Chrome Web Store Friendly Posture

The privacy story should be:

> The extension uses the user's existing LeetCode browser session to retrieve user-authorized submission data. It does not collect or transmit the user's LeetCode password or session cookie. Only coding-submission data needed for CrackedIn features is uploaded.

---

## 23. Edge Cases and Required Behavior

| Edge Case | Required Behavior |
|---|---|
| User switches tabs during historical sync | Continue or resume sync through service worker |
| User closes popup | Continue/resume; popup reads state when reopened |
| User closes LeetCode tab | Historical sync can continue; live capture stops |
| User closes browser | Sync pauses; resumes when browser opens |
| Service worker stops | Resume from saved cursor |
| User logs out of LeetCode | Pause with `auth_required` |
| User logs into different LeetCode account | Ask confirmation before linking/syncing |
| Network failure | Save progress and retry later |
| Backend upload failure | Preserve cursor/retry queue |
| Duplicate submission found | Backend upserts by platform + submission ID |
| Extension storage cleared | Use backend state where possible |
| User pauses sync | Stop between batches and preserve cursor |
| User resumes sync | Continue from saved state |
| User disconnects LeetCode | Stop sync and apply disconnect/delete policy |
| Client code differs from server code | Prefer server-confirmed code |
| Detail fetch temporarily unavailable | Retry and keep in repair queue |
| LeetCode schema changes | Defensive parsing and smoke tests |
| Large accounts | Batch, show progress, resume safely |
| Rate limiting | Cap alias batch size, backoff, jitter |
| User submits from mobile | Delta repair should catch later |
| User submits from another browser | Delta repair should catch later |
| Extension disabled during submit | Delta repair should catch later |

---

## 24. Anti-Goals

Future chats and agents should not propose these unless project direction changes:

- Do not ask users to paste LeetCode cookies.
- Do not send LeetCode cookies to CrackedIn backend.
- Do not store LeetCode passwords.
- Do not use server-side scraping with uploaded cookies.
- Do not treat profile recent activity as full history.
- Do not treat `/check/` as a historical details endpoint.
- Do not store LeetCode starter templates as submitted code.
- Do not use LeetCode username as primary CrackedIn identity.
- Do not create a separate extension-only account system.
- Do not let the popup own historical sync.
- Do not store final sync state only in memory.
- Do not upload raw browser dumps directly into the database.
- Do not make extension-side normalization the only validation layer.
- Do not build multi-platform sync before LeetCode is stable.
- Do not start MVP with an unnecessarily large normalized schema.

---

## 25. Multi-Platform Direction

LeetCode ships first.

The database should still be platform-agnostic from day one.

Future platforms:

| Platform | Future Strategy |
|---|---|
| Codeforces | Public REST for graph; authenticated code access if needed |
| GFG | Likely authenticated scraping/API-like extraction |
| Code courses | Platform-specific events/API |
| CrackedIn practice | Native event capture |

Cross-platform problem matching should not happen during sync. It is a later AI/data-normalization problem.

Use flexible fields and `platform + problem_slug` identity so future integrations can be added without redesigning the entire corpus model.

---

## 26. MVP Build Order

Recommended implementation order:

1. Add `user_coding_profiles`.
2. Add `user_coding_submissions`.
3. Add backend extension-auth exchange.
4. Add backend current-user endpoint for extension.
5. Add coding-profile connect/upsert endpoint.
6. Add submission batch-upsert endpoint.
7. Build extension popup with Continue with CrackedIn.
8. Implement extension token storage/refresh.
9. Implement LeetCode `userStatus` check.
10. Create/update LeetCode coding profile.
11. Implement historical `submissionList` pagination.
12. Add GraphQL aliasing for metadata batches.
13. Store local sync cursor after every batch.
14. Upload sanitized metadata batches.
15. Implement `submissionDetails` code/detail fetching.
16. Add GraphQL aliasing for detail batches.
17. Upload sanitized code/detail batches.
18. Add progress tracker in popup.
19. Mirror sync state to backend.
20. Add pause/resume.
21. Add auth-required recovery.
22. Add live capture on LeetCode problem pages.
23. Add server-confirmed detail fetch after live capture.
24. Add periodic delta repair.
25. Add disconnect/delete/export controls.
26. Add monitoring and logs.

---

## 27. Testing Checklist

### 27.1 Authentication Tests

- User can connect extension with CrackedIn account.
- Extension receives scoped token.
- Extension can call backend `/me`.
- Token refresh works.
- Logout/disconnect clears token.

### 27.2 LeetCode Auth Tests

- User logged into LeetCode returns signed-in status.
- User logged out shows auth-required state.
- Switching LeetCode accounts triggers confirmation.
- LeetCode cookies are not read or uploaded.

### 27.3 Historical Metadata Tests

- First page of `submissionList` works.
- Pagination reaches end.
- Total fetched roughly matches profile stats.
- Aliased metadata requests return multiple pages.
- Cursor resumes after interruption.

### 27.4 Code Details Tests

- `submissionDetails(submissionId)` returns code.
- Aliased detail requests return multiple submissions.
- Code is stored as server-confirmed code.
- Failed detail fetches enter retry queue.

### 27.5 Backend Ingestion Tests

- Payload validation rejects unsafe fields.
- Duplicate submission does not create duplicate row.
- Upsert by `platform + platform_submission_id` works.
- `user_id` and `coding_profile_id` ownership is enforced.
- Runtime/memory/verdict normalization works.
- Raw fragments are sanitized.

### 27.6 Service Worker Tests

- Sync continues if popup closes.
- Sync continues/resumes if user switches tabs.
- Sync pauses if browser closes.
- Sync resumes on browser restart/popup open/alarm.
- State persists after every batch.

### 27.7 Progress UI Tests

- Popup reads progress on open.
- Popup updates live while open.
- Metadata phase shows found-so-far progress.
- Code phase shows determinate progress.
- Complete state appears only after backend finalization.
- Paused/auth-required/failed states show correct UI.

### 27.8 Live Capture Tests

- LeetCode problem page content script loads.
- Submit event is detected.
- Submission ID is extracted from judging/check flow.
- Service worker fetches official detail.
- Backend upserts same row.

### 27.9 Delta Repair Tests

- Missed accepted submission is found.
- Missed failed submission is found.
- Missing IDs are fetched and uploaded.
- Existing IDs are ignored/deduped.

---

## 28. Engineering Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| LeetCode GraphQL schema changes | Sync breaks | Minimal fields, defensive parsing, smoke tests |
| Rate limiting | Slow or failed sync | Batch caps, jitter, backoff |
| MV3 service worker shutdown | Partial sync | Persist cursor after every batch |
| Popup closes | UI disappears | Popup only displays saved state |
| Live capture fragility | Missed future submissions | Periodic delta repair |
| Browser closes | Sync pauses | Resume later from cursor |
| User privacy concern | Trust issue | No cookie upload, clear consent, delete/export |
| Large user history | Long onboarding | Aliasing, progress UI, resumable sync |
| Backend duplicate rows | Polluted corpus | Unique constraint and idempotent upserts |
| Client tampering | Bad data/security issue | Backend validation and ownership checks |
| Code storage sensitivity | Privacy/security concern | Treat code as sensitive; consider encryption at rest |

---

## 29. Open Questions

These remain implementation decisions:

1. Should submitted code be encrypted at rest immediately?
2. What exact backend endpoint names should be used?
3. How much sanitized raw detail response should be stored?
4. What is the safe alias batch size under real-world rate limits?
5. Should extension upload metadata and code in separate endpoints or one batch endpoint?
6. Should there be a local-only debug mode?
7. What exact Chrome Web Store privacy wording should be used?
8. What happens to imported data if user uninstalls extension but does not delete CrackedIn account?
9. How long should extension refresh tokens live?
10. How frequently should delta repair run in production?

---

## 30. Future AI Behavior Instructions

When this document is loaded as project context, future AI assistants should assume:

- The project is CrackedIn / InterviewPrep.ai LeetCode Access.
- The goal is a Chrome extension that builds a user coding corpus from LeetCode.
- The chosen auth flow is **Continue with CrackedIn**.
- CrackedIn identity is primary; LeetCode is a connected external profile.
- LeetCode access uses the browser's existing session.
- LeetCode cookies must never be uploaded to CrackedIn.
- Historical sync uses `submissionList` for metadata.
- Historical code uses `submissionDetails`.
- Sync mode is full historical archive.
- The extension sanitizes data before upload.
- Backend performs final validation and normalization.
- MVP database uses `user_coding_profiles` and `user_coding_submissions`.
- Historical sync runs from the extension service worker.
- Popup only displays progress and user controls.
- Content script handles live LeetCode page observation.
- Sync must be resumable through `chrome.storage.local` and backend cursor state.
- Progress tracker is phase-based and state-driven.
- Live capture is useful but not sufficient.
- Periodic delta repair is required.
- AI analysis should use CrackedIn database, not call LeetCode directly.

Do not re-litigate alternatives unless explicitly asked. Use the chosen strategy as the baseline.

---

## 31. Final Principle

The complete system should follow this principle:

> Connect the extension to the user's existing CrackedIn account, use the user's own browser session to collect authorized LeetCode submission data, sanitize before upload, let the backend create canonical records, keep sync resumable through service-worker state, show progress from persisted sync status, and maintain the corpus through live capture plus periodic repair.

This document is the global context that future LeetCode Access project chats should use unless the project direction is explicitly changed.
