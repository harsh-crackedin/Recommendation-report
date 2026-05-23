# Hybrid LeetCode Submission Capture Report

## 1. Purpose of This Report

This report defines the chosen hybrid approach for capturing a user's LeetCode activity inside the InterviewPrepAI ecosystem.

The goal is to build a browser extension that can import a user's complete historical LeetCode submission corpus and continue capturing future submissions while the user solves problems. This data will later power AI-based recommendations, code review, weakness detection, topic analysis, spaced repetition, and personalized interview preparation.

The selected approach has three major parts:

1. Historical backfill for all past LeetCode submissions.
2. Live capture for future submissions during active LeetCode sessions.
3. Periodic delta repair to catch anything missed by live capture.

Backend ingestion is part of the system, but this report intentionally focuses more on the capture and synchronization strategy than on backend implementation details.

---

## 2. High-Level Conclusion

The hybrid approach is technically feasible and has already been validated through browser testing.

The important validated facts are:

- A logged-in browser session can call LeetCode GraphQL using the user's existing browser authentication.
- `submissionList` can retrieve historical submission metadata.
- `submissionDetails` can retrieve the actual submitted code snapshot and judge details.
- GraphQL aliasing can batch multiple historical pages or multiple submission-detail fetches into fewer requests.
- The `/submissions/detail/<submissionId>/check/` endpoint can be used during live submission flow to detect new submission IDs.
- The `/check/` endpoint is not the source of truth for historical data; it is only a live detection signal.
- `submissionDetails(submissionId)` should remain the canonical source of final code and judge result data.

The final architecture should therefore combine:

```text
Historical GraphQL sync
+ Live session capture
+ Periodic delta repair
+ Idempotent backend storage
```

This is stronger than a one-time import because the user's coding corpus remains fresh over time.

---

## 3. Why a Hybrid Approach Is Needed

A pure historical sync is not enough. It captures everything the user did before onboarding, but it does not automatically keep the database updated when the user continues solving problems.

A pure live-capture system is also not enough. It only sees submissions made while the extension is active in the user's browser. It can miss submissions made from mobile, another browser, another computer, or while Chrome is closed.

The hybrid model solves both problems:

| Layer | Purpose | Reliability Role |
|---|---|---|
| Historical backfill | Imports the user's complete past submission corpus | Creates the initial full dataset |
| Live capture | Captures new submissions as the user solves | Keeps the dataset fresh in real time |
| Periodic delta repair | Re-checks recent activity | Repairs missed live events |

This design is the most reliable practical approach for a browser-extension-based LeetCode corpus product.

---

## 4. Authentication Model

The extension should not ask users to paste their LeetCode cookies, session tokens, or passwords.

Instead, the extension should use the browser's existing authenticated LeetCode session. When the extension sends requests to LeetCode GraphQL with browser credentials included, the browser attaches the user's LeetCode cookies automatically.

This means:

- The user logs into LeetCode normally.
- The extension makes LeetCode requests from the user's browser.
- The extension does not need to read the `LEETCODE_SESSION` cookie.
- The backend does not receive LeetCode credentials.
- The backend only receives cleaned submission data.

This is the preferred trust model because the extension uses the same authenticated browser boundary as the LeetCode website itself.

---

## 5. Historical Backfill

Historical backfill is the first major phase. It runs when the user connects LeetCode and starts sync.

Its purpose is to import all previous submissions, including:

- Accepted submissions.
- Wrong answers.
- Time limit exceeded submissions.
- Memory limit exceeded submissions.
- Runtime errors.
- Compile errors.
- Multiple attempts on the same problem.
- Tried-but-unsolved problems.
- Code snapshots for every historical submission.

The selected product decision is:

> **All historical code will be stored.**

That means the sync should not stop at metadata or only selected accepted solutions. It should eventually fetch `submissionDetails` for every historical submission ID collected from `submissionList`.

---

## 6. Why the Profile Page Is Not Enough

The LeetCode profile page exposes only partial activity. In testing, the profile GraphQL data included recent accepted submissions, but the account's aggregate statistics showed many more total submissions and attempted problems.

This means the profile request is useful for summary counts and recent display, but it is not a full-history export.

The historical sync must not rely on:

- visible profile activity,
- recent accepted submissions,
- DOM-rendered profile data,
- profile-page scraping.

Instead, it should use the authenticated GraphQL submission history path.

The important distinction is:

| Source | Use |
|---|---|
| `recentAcSubmissionList` | Recent accepted activity only |
| `submissionList(offset, limit)` | Full historical submission metadata |
| `submissionDetails(submissionId)` | Code snapshot and judge details |

---

## 7. Historical Metadata Extraction

The historical sync begins by collecting submission metadata.

The system should walk the user's submission history page by page. Each page returns a set of submissions containing information such as:

- Submission ID.
- Problem title.
- Problem slug.
- Status or verdict.
- Programming language.
- Runtime.
- Memory.
- Timestamp.

The most important value collected here is the **submission ID**, because that ID is later used to fetch the full code snapshot and detailed judge result.

LeetCode appears to cap the submission-list page size at around 20, so the sync must paginate rather than assuming a large limit will work.

---

## 8. GraphQL Aliasing for Faster Backfill

GraphQL aliasing should be used to make historical sync practical and fast.

Without aliasing, the extension would need one request for every page of submission history. With aliasing, multiple pages can be fetched in a single GraphQL request by giving each requested page a different alias.

For example, instead of making ten separate page requests, the extension can request ten different offsets in one request.

Aliasing is useful for both:

- historical submission-list pagination,
- batched submission-details/code fetches.

This is important because the selected mode stores all historical code. For users with hundreds or thousands of submissions, batching is necessary for a smooth onboarding experience.

---

## 9. Historical Code Snapshot Extraction

After all submission IDs are collected, the extension should fetch full submission details for every submission.

The canonical source is:

```text
submissionDetails(submissionId)
```

This returns the user's actual submitted code and execution data.

The system should treat this as the official server-confirmed version of the submission. It should be preferred over any client-side captured code if there is a mismatch.

The important distinction is:

| Field | Meaning |
|---|---|
| `submissionDetails.code` | Actual user-submitted code |
| `question.codeSnippets` | LeetCode starter templates |

Only `submissionDetails.code` should be treated as the user's submitted code.

---

## 10. Full Historical Sync Flow

The full historical flow should look like this:

1. Check whether the user is logged into LeetCode.
2. Identify the LeetCode username and user ID.
3. Start paginated historical submission metadata sync.
4. Collect all submission IDs.
5. Store or queue all metadata.
6. Fetch submission details for every collected submission ID.
7. Store the actual code snapshot and judge metadata.
8. Mark the historical sync complete.
9. Keep sync state resumable in case the browser extension service worker is stopped.

The system should not assume the sync will finish in one uninterrupted session. Chrome Manifest V3 service workers can be stopped while idle, so sync state should be saved after each batch.

---

## 11. Live Capture

Live capture is the second major part of the hybrid system.

It runs while the user is actively solving problems on LeetCode.

The selected product decision is:

> **Capture code directly from the active LeetCode session, then call `submissionDetails` to get the judge result and any missing official details.**

This means the live capture layer has two sources:

1. Client-side capture from the submit/editor flow.
2. Server-confirmed capture from `submissionDetails(submissionId)`.

This is a strong approach because it captures the code immediately while still reconciling with LeetCode's official stored submission after judging finishes.

---

## 12. Why Live Capture Should Not Rely Only on the Submit Request

The submit request may contain the code, language, and problem slug. However, it does not necessarily contain the final judge result.

At the moment of submission, LeetCode may not yet know:

- whether the result is accepted,
- whether it failed,
- which testcase failed,
- final runtime,
- final memory,
- final status code,
- runtime error details,
- compile error details,
- final testcase count.

Therefore, the submit request is useful for early code capture, but it should not be treated as the final record.

The final record should be completed by calling `submissionDetails(submissionId)` after the submission ID is known.

---

## 13. Live Submission ID Detection

During a normal LeetCode submission flow, the frontend polls a URL shaped like:

```text
/submissions/detail/<submissionId>/check/
```

This endpoint may return `PENDING` while the judge is still processing the submission. That is expected.

The important part is not the response body. The important part is that the URL contains the new submission ID.

The extension can detect this URL, extract the submission ID, and then trigger a details fetch after a short delay or retry cycle.

---

## 14. Correct Interpretation of the `/check/` Endpoint

The `/check/` endpoint should not be used as a historical data endpoint.

When opened manually or reused for an old submission, it may return only:

```text
PENDING
```

This is not a failure of the architecture. It simply means `/check/` is a live judge-polling mechanism, not the final details API.

The correct interpretation is:

| Endpoint / Source | Correct Role |
|---|---|
| Submit request | Early code/session capture |
| `/submissions/detail/<id>/check/` | Live submission ID detector |
| `submissionDetails(submissionId)` | Final source of code and judge data |

This distinction is critical for the extension design.

---

## 15. Recommended Live Capture Flow

The live capture flow should work as follows:

1. User opens a LeetCode problem.
2. Extension content script is active on the problem page.
3. User writes code.
4. User clicks Submit.
5. Extension captures the submitted code from the client-side session or submit payload.
6. LeetCode begins judge polling.
7. Extension detects the `/submissions/detail/<id>/check/` URL.
8. Extension extracts the new submission ID.
9. Extension waits briefly for judging to complete.
10. Extension calls `submissionDetails(submissionId)`.
11. Extension merges the client-captured code with the server-confirmed judge details.
12. Extension sends cleaned final data to the backend.
13. If final details are not ready, the extension retries.
14. If retries fail, the submission is marked for periodic delta repair.

This approach gives both immediacy and correctness.

---

## 16. Client-Captured Code vs Server-Confirmed Code

Because live capture will collect code directly from the user session and then fetch server-confirmed details, the system should distinguish between two concepts:

| Type | Meaning | Trust Level |
|---|---|---|
| Client-captured code | Code seen in editor or submit payload | Useful early snapshot |
| Server-confirmed code | Code returned by `submissionDetails` | Canonical final snapshot |

Usually these should match. If they do not, the server-confirmed version should be treated as canonical because it is what LeetCode stored against the submission ID.

The client-captured code is still valuable because:

- it can be available immediately,
- it can help debug failed detail fetches,
- it may preserve context if LeetCode delays the details response,
- it confirms what the user attempted to submit.

But final analytics should prefer the server-confirmed submission record.

---

## 17. Live Capture Reliability Requirements

Live capture should be designed defensively.

The extension should not assume only one network pattern. It may need to observe:

- fetch requests,
- XMLHttpRequest requests,
- submit payloads,
- GraphQL responses,
- judge polling URLs.

LeetCode's frontend may change over time. The most reliable design is to treat live capture as a best-effort real-time layer and rely on periodic delta repair as the consistency layer.

This is important because live capture can miss submissions if:

- the user submits from another browser,
- the user submits from mobile,
- the browser is closed,
- the extension is disabled,
- the LeetCode tab is not open,
- the content script fails to inject,
- LeetCode changes the frontend request shape.

---

## 18. Periodic Delta Repair

Periodic delta repair is the third major layer.

Its job is to find submissions that were missed by live capture.

This is necessary because live capture is limited to the user's active browser session. It cannot guarantee that every future LeetCode submission will be observed.

Periodic delta repair should run on a schedule, such as:

- when the browser starts,
- when the extension wakes,
- every fixed interval,
- when the user opens the extension popup,
- when the user manually clicks refresh.

The repair process should compare recent LeetCode submissions with the backend's known submission IDs.

Any missing submission IDs should be fetched through `submissionDetails`.

---

## 19. What Delta Repair Should Check

Delta repair should not only check recent accepted submissions.

It should check both:

1. Recent accepted submissions.
2. Recent general submissions.

The reason is that recent accepted feeds may miss:

- wrong answers,
- TLE,
- compile errors,
- runtime errors,
- still-unsolved attempts.

The recommended delta approach is:

```text
recentAcSubmissionList(username, limit)
+ first few pages of submissionList(offset, limit)
```

This combination lets the system catch both accepted and failed recent submissions.

This is especially important for AI coaching because failed attempts are often more valuable than accepted attempts. They reveal where the user struggled.

---

## 20. Final Hybrid Sync Lifecycle

The complete lifecycle should be:

### Stage 1: User Connects LeetCode

The extension detects whether the user is signed in to LeetCode and confirms the external LeetCode identity.

### Stage 2: Historical Metadata Sync

The extension fetches every historical submission entry using paginated GraphQL.

### Stage 3: Historical Code Sync

The extension fetches `submissionDetails` for every historical submission ID and stores all historical code.

### Stage 4: Live Capture Activation

The extension begins monitoring active LeetCode problem sessions.

### Stage 5: Live Submit Capture

When the user submits code, the extension captures the code/session data and detects the new submission ID.

### Stage 6: Server Confirmation

The extension calls `submissionDetails` to complete the record with final judge information.

### Stage 7: Periodic Repair

The extension periodically checks recent activity and fills in any missed submissions.

### Stage 8: Continuous AI Readiness

The backend corpus remains current and can be used by the AI layer to generate recommendations, analysis, and review.

---

## 21. Chosen Schema Direction: Option A

The selected schema direction is **Option A**, which keeps the database extension minimal.

The current InterviewPrepAI user system should remain the source of truth for users. The LeetCode and future coding-platform data should attach to the existing user rather than creating a separate user model.

The two new conceptual tables are:

1. `user_coding_profiles`
2. `user_coding_submissions`

This supports LeetCode now and other platforms later.

The purpose of these tables is:

| Table | Purpose |
|---|---|
| `user_coding_profiles` | Links an InterviewPrepAI user to an external coding account |
| `user_coding_submissions` | Stores each submitted attempt, code snapshot, verdict, and metadata |

This keeps the system modular without over-engineering the first version.

---

## 22. Minimal Role of Backend Ingestion in This Report

Backend ingestion should be simple and idempotent.

The backend should receive cleaned records from the extension and upsert them by platform and submission ID.

The key backend rule is:

> Historical sync, live capture, delta repair, and retries may all see the same submission. Duplicate records must not be created.

For now, the implementation focus should remain on accurate capture and reliable sync.

Detailed backend schema, API contracts, migrations, and AI-query optimization can be handled in a later report.

---

## 23. Recommended MVP Scope

The MVP should implement the hybrid approach in this order:

1. LeetCode auth check through the logged-in browser session.
2. Historical `submissionList` pagination.
3. Historical `submissionDetails` fetch for every submission.
4. Local resumable sync state.
5. Live capture on LeetCode problem pages.
6. Client-side code capture during submit.
7. Submission ID detection through judge polling.
8. Server confirmation through `submissionDetails`.
9. Periodic delta repair.
10. Basic backend upsert into Option A tables.

Do not start with multiple platforms in the first implementation. The model should be modular enough for future platforms, but LeetCode should be implemented and stabilized first.

---

## 24. Main Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| LeetCode GraphQL changes | Historical sync may break | Keep queries minimal, add error monitoring |
| Rate limiting | Sync slows or fails | Use batching, delays, retries, fallback smaller batches |
| Extension worker stops | Partial sync | Save sync cursor after every batch |
| Live capture misses events | Missing future submissions | Periodic delta repair |
| `/check/` response is incomplete | Bad final record | Use it only to detect submission ID |
| Submit payload differs from final stored code | Data inconsistency | Prefer `submissionDetails.code` as canonical |
| User privacy concern | Trust issue | Do not store cookies; explain code sync clearly |
| Large accounts | Long sync time | Use aliasing, progress UI, resumable sync |

---

## 25. Final Recommendation

The chosen hybrid approach is the correct architecture for this product.

Historical GraphQL sync gives the initial full corpus. Live capture keeps the corpus updated when the user actively solves problems. Periodic delta repair makes the system reliable even when live capture misses submissions. Server-confirmed `submissionDetails` should remain the canonical source for final code and judge metadata.

The final implementation principle should be:

```text
Capture early, confirm officially, repair periodically.
```

This gives InterviewPrepAI a strong data foundation for personalized AI coaching while keeping the user-authentication model safe and the database design modular for future coding platforms.
