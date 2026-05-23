# Feasibility Report: Extracting and Storing Full LeetCode Submission History Through a Browser Extension

## Executive Summary

The claim is technically feasible: a browser extension can extract a logged-in user's LeetCode submission history, collect submission metadata, fetch code snapshots for selected or all submissions, and store the cleaned result in the product backend.

The strongest feasible path is not profile-page scraping and not manual cookie upload. The correct architecture is a Chrome or Chromium extension that makes authenticated GraphQL requests to LeetCode from the user's browser using the existing logged-in session. The extension does not need to read, store, or send the user's LeetCode session token to the backend. It only sends cleaned submission data.

The profile page request alone is insufficient because it exposes only recent accepted submissions. In the tested profile data, the recent accepted list contained only a small recent subset, while the profile aggregate showed 437 attempted questions and 791 total submissions. Therefore, the full sync must use the authenticated GraphQL submission history API rather than the public profile recent-activity request.

The submitted code snapshot is available through the submission detail GraphQL response. The uploaded submission detail response confirmed that the real user-submitted code is present under submissionDetails.code. This is separate from LeetCode's starter templates, which appear under question.codeSnippets and should not be treated as user code.

The implementation is not completely risk-free, but there is no fundamental blocker in the current architecture. The main risks are rate limiting, GraphQL schema changes, Manifest V3 service worker interruptions, and Chrome Web Store review expectations. These can be handled with batching, retry logic, resumable sync state, conservative request pacing, and a transparent privacy model.

## Final Feasibility Verdict

The extraction and storage plan is feasible for an authenticated user who installs the extension and is logged into LeetCode in the same browser profile.

A more precise version of the claim is:

"We can reliably sync a user's LeetCode submission corpus through a browser extension by using the user's own authenticated browser session, walking all submissions through paginated GraphQL, fetching code snapshots through submissionDetails, and storing normalized metadata and code in our backend with resumable sync."

The phrase "without any problem" should be avoided in engineering documentation. A better phrasing is:

"There is no known technical blocker, but the implementation must include pagination, retry handling, rate-limit safety, session checks, resumable state, and privacy controls."

## What Has Been Verified

### Authenticated browser access works

The extension can call LeetCode GraphQL from the browser with credentials included. Chrome attaches the user's existing LeetCode cookies automatically when the request targets leetcode.com and the extension has the proper host permission. This allows authenticated access without reading the cookie value.

This is the correct trust model because the extension does not handle the LeetCode password, does not send LEETCODE_SESSION to our server, and does not require the user to paste cookies into a backend or config file.

### Profile data is not enough

The user profile GraphQL response includes useful aggregate data such as accepted question count, failed question count, total submission count, and recent accepted submissions. However, recentAcSubmissionList is only a recent accepted-submission feed. It does not contain the full historical submission list.

The uploaded profile response showed:

| Metric | Value |
|---|---:|
| Accepted questions | 434 |
| Accepted submissions | 617 |
| Total attempted questions | 437 |
| Total submissions | 791 |
| Failed or not-accepted attempted questions | 3 |

This proves that a recent accepted-submission list cannot be used as the primary full-history source. It is useful only for periodic delta refresh.

### Full submission history can be walked through submissionList

The correct full-history source is the authenticated GraphQL field submissionList(offset, limit). It returns submission IDs and metadata. The limit is effectively capped at 20 by LeetCode, so increasing limit above 20 is not reliable. The correct implementation is to walk offsets until hasNext is false.

The metadata returned by submissionList is enough to build the user's solve graph before fetching code. It provides the submission ID, problem slug, verdict, language, runtime, memory, and timestamp. This means the product can start producing useful analytics even before every code snapshot is fetched.

### GraphQL aliasing makes full sync practical

Because LeetCode caps each submissionList page at 20, a naive implementation would require many sequential requests. GraphQL aliasing solves this by requesting multiple pages in one POST. For example, 10 aliases can fetch offsets 0, 20, 40, and so on up to 180 in one request, returning up to 200 submissions.

This makes the historical metadata sync practical even for users with thousands of submissions.

### Code snapshot is available through submissionDetails

The uploaded submission detail response confirms that submissionDetails contains the user's actual submitted code under the code field. It also includes language, status code, timestamp, runtime, memory, testcase counts, runtime error, compile error, output fields, and the problem's titleSlug.

This confirms that the extension does not need to scrape the editor DOM to recover old code. It can collect submission IDs through submissionList and then fetch code snapshots through submissionDetails.

### The /check/ endpoint is only for live capture

The URL /submissions/detail/<submissionId>/check/ is not the historical code source. When opened manually for an old submission, it may return PENDING and should not be interpreted as a failed historical sync.

Its value is live capture. During a fresh LeetCode submit, the frontend polls this endpoint while judging. The submission ID appears in the URL. The extension can detect this request, extract the submission ID, then call submissionDetails after a short delay or after success.

Therefore:

| Source | Correct use |
|---|---|
| recentAcSubmissionList | Recent accepted delta only |
| submissionList(offset, limit) | Full historical metadata sync |
| submissionDetails(submissionId) | Code snapshot and detailed result data |
| /submissions/detail/<id>/check/ | Live submission ID detection only |

## Recommended Product Architecture

### Component 1: Browser extension

The browser extension is responsible for communicating with LeetCode. It runs in the user's browser, where the LeetCode session already exists.

Responsibilities:

| Responsibility | Description |
|---|---|
| Auth check | Call userStatus to verify the user is logged in |
| Full sync | Walk submissionList until all metadata is collected |
| Code fetch | Call submissionDetails for selected or all submission IDs |
| Live capture | Detect new submission IDs from verdict polling requests |
| Retry and resume | Store progress locally so sync can continue after service worker shutdown |
| Data cleaning | Remove cookies, headers, and unrelated browser data before backend upload |

### Component 2: Backend ingestion API

The backend should receive cleaned normalized payloads from the extension. It should never receive LeetCode cookies or raw browser headers.

Responsibilities:

| Responsibility | Description |
|---|---|
| User mapping | Link LeetCode username/userId to the app user |
| Upserts | Idempotently store problems, user problems, submissions, and code |
| Deduplication | Use platform plus submission ID for submissions, and platform plus titleSlug for problems |
| Encryption | Prefer encrypting code snapshots at rest |
| Deletion | Provide immediate purge when the user disconnects or deletes data |
| Analytics preparation | Build aggregate tables or materialized views for AI recommendations |

### Component 3: AI analysis layer

The AI layer should consume the stored solve graph and code snapshots. It does not need to directly call LeetCode.

Useful AI features include weakness detection, failed-to-accepted comparison, problem-pattern inference, time-to-solve estimation, code-style critique, language-specific feedback, spaced repetition scheduling, and next-problem recommendation.

## Recommended Sync Flow

### Phase A: Auth check

The extension first calls userStatus.

If the user is not signed in, the extension should ask them to open LeetCode and log in normally. If the user is signed in, show the LeetCode username and start sync.

### Phase B: Full metadata sync

The extension walks submissionList from offset 0 until the end. Each returned submission is upserted into user_submissions. Problems are deduplicated by titleSlug.

This phase gives the backend every historical attempt's high-level metadata. It is the source of truth for the user's complete solve graph.

### Phase C: Code snapshot sync

After metadata is collected, the extension builds a code-fetch queue.

For MVP, fetch code for:

| Priority | Submission type | Reason |
|---|---|---|
| 1 | Latest accepted submission per problem/language | Main solution review |
| 2 | Last failed submission before first accepted | Best learning signal for improvement analysis |
| 3 | Latest submission for unsolved/tried problems | Best signal for stuck-problem coaching |
| 4 | Older attempts | Fetch lazily or during full archive mode |

For a complete archive product mode, fetch submissionDetails for every submission ID.

### Phase D: Live capture

A content script runs on LeetCode problem pages. It observes verdict polling requests that include /submissions/detail/<id>/check/. When a new submission ID is seen, it sends the ID to the service worker.

The service worker waits briefly and then calls submissionDetails. If the result is not ready, it retries with backoff. Once details are available, it sends the cleaned payload to the backend.

### Phase E: Periodic delta refresh

Live capture only works when the user submits in a tab where the extension is active. It can miss mobile submissions, submissions from another browser, or submissions made while the extension is disabled.

Therefore, add a periodic delta refresh. The simplest approach is to periodically check recentAcSubmissionList and the first few pages of submissionList. recentAcSubmissionList catches recent accepted submissions, while submissionList catches recent non-accepted attempts too.

## Storage Feasibility

The storage model is straightforward and should be platform-agnostic from day one.

Recommended logical tables:

| Table | Purpose |
|---|---|
| problems | Shared problem metadata keyed by platform and titleSlug |
| user_problems | Per-user problem-level solve status and aggregates |
| user_submissions | One row per submission attempt |
| user_submission_code | Code blobs separated from metadata |
| sync_state | Progress tracking for resumable sync |

The most important keys are:

| Entity | Recommended key |
|---|---|
| Problem | platform + titleSlug |
| Submission | platform + submissionId |
| User-problem relation | userId + platform + titleSlug |
| Code snapshot | platform + submissionId |

Separate code from submission metadata. This keeps common analytics fast and allows selective or lazy code fetching.

## Data That Can Be Stored

The following fields are safe and useful to store after user consent:

| Category | Fields |
|---|---|
| Identity mapping | app user ID, LeetCode user ID, LeetCode username |
| Problem metadata | titleSlug, title, difficulty, topic tags, frontend ID |
| Submission metadata | submission ID, timestamp, verdict, language, runtime, memory |
| Code snapshot | submitted code, code language, fetched timestamp |
| Result details | status code, totalCorrect, totalTestcases, runtimeError, compileError |
| Sync state | phase, cursor, last progress timestamp, last full sync timestamp |

Do not store or transmit LeetCode session cookies, CSRF tokens, raw request headers, passwords, or unrelated page/network data.

## Why This Is Better Than Profile Scraping

Profile scraping only shows a partial user-facing view. It is not designed to export a full submission corpus.

The full sync approach is better because:

| Requirement | Profile recent list | submissionList + submissionDetails |
|---|---|---|
| All historical submissions | No | Yes |
| Failed attempts | No | Yes |
| TLE/WA/RE/CE verdicts | No | Yes |
| Code snapshots | No | Yes |
| Multiple attempts per problem | No | Yes |
| Reliable database keys | Partial | Yes |
| Good for AI analysis | Limited | Strong |

## Implementation Risks and Mitigations

### Risk: LeetCode GraphQL schema changes

LeetCode can change field names or response shapes. This would break brittle parsers.

Mitigation: keep the parser defensive, version the sync worker, log schema errors, and maintain smoke tests against a test account.

### Risk: Rate limits or abuse detection

Aggressive requests could trigger throttling.

Mitigation: batch with aliasing, cap aliases at around 10 fields per request, add random jitter, retry with backoff, and fallback from 10 aliases to 5 or 2 if failures increase.

### Risk: Manifest V3 service worker shutdown

Chrome can stop idle service workers.

Mitigation: persist sync_state after every batch. The sync must be idempotent so an interrupted batch can be safely retried.

### Risk: User privacy concern over code upload

Code is personal work and may contain notes or identifiable style.

Mitigation: show exactly what is collected, do not collect cookies, encrypt code at rest, provide export, pause, delete, and disconnect controls.

### Risk: Live capture endpoint behavior changes

The /check/ endpoint might change.

Mitigation: treat /check/ only as an optimization for detecting new submission IDs. Periodic submissionList delta refresh should still catch missed submissions.

### Risk: Free-tier behavior differs from premium

The initial architecture validation included premium testing. The uploaded user profile response also shows a non-premium signed-in user, which is encouraging, but full free-tier load testing should still be performed.

Mitigation: test the full flow on at least one non-premium account with real submissions before launch.

## Privacy and Consent Requirements

The product should be explicit that it is importing submission history and code snapshots. The consent screen should clearly state:

- We read LeetCode submission history from your logged-in browser session.
- We do not read, store, or upload your LeetCode password.
- We do not upload your LeetCode session cookie.
- We upload submission metadata and, depending on selected mode, submitted code snapshots.
- You can pause sync, delete imported data, or disconnect LeetCode.

This privacy posture is a major product advantage over server-side scraping or tools that ask users to paste session cookies.

## Recommended MVP Scope

For MVP, implement:

1. Auth check using userStatus.
2. Full metadata backfill using submissionList offset pagination.
3. Aliased submissionList batching after the simple version is verified.
4. Selective code fetching using submissionDetails.
5. Backend upserts for problems, user_problems, user_submissions, and user_submission_code.
6. Resumable sync_state in extension storage and backend.
7. Live capture using /submissions/detail/<id>/check/ only as a submission ID detector.
8. Periodic delta refresh using recentAcSubmissionList plus recent submissionList pages.
9. User-facing pause, resume, delete, and re-sync controls.

Avoid for MVP:

- Full DOM scraping as the main source.
- Asking users to paste cookies.
- Server-side LeetCode scraping with uploaded cookies.
- Fetching every historical code snapshot before the user can use the product.
- Cross-platform matching before LeetCode sync is stable.

## Engineering Conclusion

The plan is feasible and implementation-ready.

The critical finding is that there are two separate extraction paths:

1. Historical extraction: use authenticated GraphQL submissionList and submissionDetails.
2. Live extraction: use /check/ only to detect the new submission ID, then call submissionDetails.

The strongest implementation path is to make the extension the LeetCode-facing client, make the backend the storage and analytics layer, and keep all credentials inside the browser. The resulting corpus can support the AI product vision because it contains every attempted problem, every submission's metadata, and the selected or complete code history.

The claim should not be presented as "there will be no problems." It should be presented as "there is no known blocker, and the remaining risks are normal engineering risks with clear mitigations."

## Implementation Snippets

The following snippets are implementation-oriented and intentionally placed at the end of the report.

### Auth check

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

### Full submission metadata page

```graphql
query submissions($offset: Int!, $limit: Int!) {
  submissionList(offset: $offset, limit: $limit) {
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

### Aliased metadata batch

```graphql
query bulkPages {
  p0: submissionList(offset: 0, limit: 20) {
    hasNext
    submissions { id titleSlug statusDisplay langName runtime memory timestamp }
  }
  p1: submissionList(offset: 20, limit: 20) {
    hasNext
    submissions { id titleSlug statusDisplay langName runtime memory timestamp }
  }
  p2: submissionList(offset: 40, limit: 20) {
    hasNext
    submissions { id titleSlug statusDisplay langName runtime memory timestamp }
  }
  p3: submissionList(offset: 60, limit: 20) {
    hasNext
    submissions { id titleSlug statusDisplay langName runtime memory timestamp }
  }
  p4: submissionList(offset: 80, limit: 20) {
    hasNext
    submissions { id titleSlug statusDisplay langName runtime memory timestamp }
  }
}
```

### Submission details with code snapshot

```graphql
query submissionDetail($submissionId: Int!) {
  submissionDetails(submissionId: $submissionId) {
    code
    timestamp
    statusCode
    runtime
    runtimeDisplay
    runtimePercentile
    memory
    memoryDisplay
    memoryPercentile
    totalCorrect
    totalTestcases
    runtimeError
    compileError
    lastTestcase
    codeOutput
    expectedOutput
    fullCodeOutput
    stdOutput
    lang {
      name
      verboseName
    }
    question {
      questionId
      titleSlug
    }
  }
}
```

### Extension GraphQL fetch helper

```ts
async function leetcodeGraphQL<T>(query: string, variables?: Record<string, unknown>): Promise<T> {
  const res = await fetch("https://leetcode.com/graphql", {
    method: "POST",
    credentials: "include",
    headers: {
      "content-type": "application/json"
    },
    body: JSON.stringify({ query, variables })
  });

  if (!res.ok) {
    throw new Error(`LeetCode GraphQL failed: ${res.status}`);
  }

  const json = await res.json();

  if (json.errors?.length) {
    throw new Error(JSON.stringify(json.errors));
  }

  return json.data;
}
```

### Slow full-history walk for verification

```ts
async function fetchAllSubmissionsSlow() {
  const all: any[] = [];
  let offset = 0;
  const limit = 20;

  while (true) {
    const data = await leetcodeGraphQL<{
      submissionList: {
        hasNext: boolean;
        submissions: any[];
      };
    }>(SUBMISSION_LIST_QUERY, { offset, limit });

    const page = data.submissionList.submissions;
    all.push(...page);

    if (!data.submissionList.hasNext || page.length < limit) {
      break;
    }

    offset += limit;
    await new Promise(resolve => setTimeout(resolve, 300));
  }

  return all;
}
```

### Live capture ID detector

```js
const seenSubmissionIds = new Set();

function maybeCaptureSubmissionId(url) {
  const match = String(url).match(/\/submissions\/detail\/(\d+)\/check\//);
  if (!match) return;

  const submissionId = match[1];
  if (seenSubmissionIds.has(submissionId)) return;
  seenSubmissionIds.add(submissionId);

  chrome.runtime.sendMessage({
    type: "NEW_LEETCODE_SUBMISSION",
    submissionId
  });
}
```

### Minimal storage schema

```sql
create table problems (
  platform text not null,
  slug text not null,
  display_id text,
  title text,
  difficulty text,
  topic_tags jsonb,
  raw_meta jsonb,
  primary key (platform, slug)
);

create table user_problems (
  user_id uuid not null,
  platform text not null,
  slug text not null,
  status text not null,
  first_ac_at timestamptz,
  latest_ac_at timestamptz,
  attempt_count integer not null default 0,
  ac_count integer not null default 0,
  wa_count integer not null default 0,
  tle_count integer not null default 0,
  mle_count integer not null default 0,
  re_count integer not null default 0,
  ce_count integer not null default 0,
  best_runtime_ms integer,
  best_memory_kb integer,
  best_lang text,
  primary_submission_id text,
  primary key (user_id, platform, slug)
);

create table user_submissions (
  platform text not null,
  submission_id text not null,
  user_id uuid not null,
  slug text not null,
  verdict text not null,
  raw_status text,
  lang text,
  runtime_ms integer,
  memory_kb integer,
  submitted_at timestamptz,
  has_code boolean not null default false,
  raw_meta jsonb,
  primary key (platform, submission_id)
);

create table user_submission_code (
  platform text not null,
  submission_id text not null,
  code text not null,
  code_lang text,
  fetched_at timestamptz not null default now(),
  primary key (platform, submission_id)
);

create table sync_state (
  user_id uuid not null,
  platform text not null,
  phase text not null,
  cursor jsonb,
  last_progress_at timestamptz,
  last_full_sync_at timestamptz,
  primary key (user_id, platform)
);
```
