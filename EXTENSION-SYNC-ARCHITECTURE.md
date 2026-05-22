# LeetCode Sync Extension — Final Architecture

**Status:** Locked after probe-driven validation against a real authenticated LC account.
**Audience:** Engineering team building the Chrome extension + backend sync.
**Date:** 2026-05-22

---

## 1. The product wedge

We are building a Chrome extension that captures the user's full LeetCode solve graph (every submission ever made, including code) into our backend, and sells AI features on top of that corpus: review old solutions, weakness detection, pattern analysis, spaced repetition, cross-problem similarity, code-style critique.

Nobody else is doing this end-to-end (see §9). The market splits into:
- **Capture-only extensions** (LeetHub family, ~5k stars combined) — code → user's GitHub, no AI
- **In-page AI helpers** (LeetBuddy, LeetCode Assistant, etc.) — chat overlay on current page, no history, no auth, no corpus
- **MCP servers** (jinzcdev, doggybee, etc.) — stateless data shims for LLMs, no persistence
- **SR trackers** — Anki-style, metadata-only

Our position: **capture + corpus + AI features that require the corpus.** First-to-corpus wins; LeetBuddy is the team to watch.

---

## 2. Auth model — the trust moat

**The extension never reads, stores, or transmits the user's LC session token.**

Mechanism: the extension makes `fetch('https://leetcode.com/graphql', { credentials: 'include', ... })` calls from the service worker. Chrome's cookie jar attaches the existing `LEETCODE_SESSION` cookie automatically because the request is to `leetcode.com` and the user is logged in there. Our extension code is never on the read side of that cookie.

This is strictly better than every alternative:

| Approach | Token handling | Trust posture |
|---|---|---|
| MCP server | User pastes `LEETCODE_SESSION` into config file on disk | Worst — plaintext credential |
| Server-side scraper | User uploads cookie to our backend | Worst — we hold the credential |
| **Chrome extension (us)** | Cookie never leaves browser; we ride existing session | Best — same trust boundary as LC website |
| In-page DOM helpers | No LC auth at all | Trivial trust ask, trivial product (no history access) |

The user logs into LC normally. We do not see their password, their cookie, or any credential. We only see what the LC API returns for queries made on their behalf by their own browser.

`host_permissions: ["https://leetcode.com/*"]` in `manifest.json` is what unlocks `credentials: 'include'`. This is also what the user sees and approves at install time on the Chrome Web Store.

---

## 3. Verified API capabilities

All of the following were validated against `bhaskarbhakat` (real account, Premium, ~740 AC problems) on 2026-05-22:

| Capability | Endpoint | Verified result |
|---|---|---|
| Auth check | `userStatus { isSignedIn username userId isPremium }` | ✅ 600ms, returns auth state |
| Bulk submission walk | `submissionList(offset, limit)` | ✅ paginated; **`limit` is hardcoded server-side at 20**, larger values silently clamped (tested up to 9999) |
| Per-slug attempts | `submissionList(questionSlug)` | ✅ returns all attempts for one problem in one call, no pagination needed |
| Single-call code fetch | `submissionDetails(submissionId)` | ✅ returns code + lang + runtime/memory + percentiles + testcase counts |
| AC problem universe | `problemsetQuestionList(filters:{status:"AC"}, limit:50, skip:N)` | ✅ paginated full solve graph |
| TRIED-not-AC universe | `problemsetQuestionList(filters:{status:"TRIED"})` | ✅ same shape |
| Difficulty/topic counts | `userProfileUserQuestionProgressV2(userSlug)` | ✅ aggregate counts |
| Recent AC delta | `recentAcSubmissionList(username, limit)` | ✅ for periodic delta refresh |
| **GraphQL aliasing** | Multi-field same-endpoint queries | ✅ **defeats the limit:20 cap** — see §4 |

### What we cannot do

- ❌ Increase `limit` beyond 20 — server clamps silently
- ❌ Get code without an authenticated session
- ❌ Cross-user queries (only the logged-in user's submissions are visible)

---

## 4. The aliasing trick — why our sync is 5–10x faster than competitors

Standard GraphQL feature, defined since 2015. Lets us call the same field N times in one HTTP request, with different arguments, by giving each call a unique label.

**Without aliasing** (what every other LC tool does — sequential calls):

```
walk 2,000 submissions → 100 round trips × 600ms = 60s
fetch 1,000 codes      → 1,000 round trips × 400ms = 6.5min
Total: ~7.5 minutes
```

**With aliasing** (10 fields per request):

```
walk 2,000 submissions → 10 round trips × 3s = 30s
fetch 1,000 codes      → 100 round trips × 400ms = 40s
Total: ~70 seconds
```

This is the single biggest performance unlock. We measured 354ms for a 10-aliased `submissionDetails` call returning 10 full code blobs.

### Example — bulk submissionList aliased

```graphql
query bulkPages {
  p0: submissionList(offset: 0,   limit: 20) { hasNext submissions { id titleSlug statusDisplay langName runtime memory timestamp } }
  p1: submissionList(offset: 20,  limit: 20) { hasNext submissions { id titleSlug statusDisplay langName runtime memory timestamp } }
  p2: submissionList(offset: 40,  limit: 20) { hasNext submissions { id titleSlug statusDisplay langName runtime memory timestamp } }
  p3: submissionList(offset: 60,  limit: 20) { hasNext submissions { id titleSlug statusDisplay langName runtime memory timestamp } }
  p4: submissionList(offset: 80,  limit: 20) { hasNext submissions { id titleSlug statusDisplay langName runtime memory timestamp } }
  p5: submissionList(offset: 100, limit: 20) { hasNext submissions { id titleSlug statusDisplay langName runtime memory timestamp } }
  p6: submissionList(offset: 120, limit: 20) { hasNext submissions { id titleSlug statusDisplay langName runtime memory timestamp } }
  p7: submissionList(offset: 140, limit: 20) { hasNext submissions { id titleSlug statusDisplay langName runtime memory timestamp } }
  p8: submissionList(offset: 160, limit: 20) { hasNext submissions { id titleSlug statusDisplay langName runtime memory timestamp } }
  p9: submissionList(offset: 180, limit: 20) { hasNext submissions { id titleSlug statusDisplay langName runtime memory timestamp } }
}
```

One HTTP POST → 200 submissions back. Stop when any `hasNext: false` appears. Otherwise advance offset by 200, repeat.

### Example — code fetch aliased

```graphql
query codeBatch {
  s0: submissionDetails(submissionId: 1973994054) { code lang { name } runtime memory runtimePercentile question { titleSlug } }
  s1: submissionDetails(submissionId: 1973990919) { code lang { name } runtime memory runtimePercentile question { titleSlug } }
  s2: submissionDetails(submissionId: 1973038164) { code lang { name } runtime memory runtimePercentile question { titleSlug } }
  ...
}
```

The `s0:`, `s1:` prefixes are arbitrary labels we make up — they don't correspond to slugs or IDs. They're just keys in the response JSON so we can tell which `submissionDetails` call produced which result.

Response shape:

```json
{
  "data": {
    "s0": { "code": "class Solution { ... }", "lang": { "name": "cpp" }, "question": { "titleSlug": "find-and-replace-in-string" } },
    "s1": { "code": "...", "lang": { "name": "cpp" }, ... },
    ...
  }
}
```

---

## 5. Identity model — pick `titleSlug` as the canonical key

LeetCode exposes four IDs per problem. Use `titleSlug`.

| Identifier | What it is | Use? |
|---|---|---|
| `id` (submission id) | Per-submission, monotonic int | ✅ key for `user_submissions` rows |
| `questionId` | Internal LC db id; can mismatch frontend display | ❌ confusing |
| `questionFrontendId` | What user sees ("833. Find And Replace") | Display only |
| `titleSlug` | URL-safe slug ("find-and-replace-in-string") | ✅ **canonical join key** |
| `title` | Human-readable | Display only |

`titleSlug` is what every LC API uses as filter, what the URL uses, and what cross-user dedup works against. Make it the PK on `problems` and the FK from `user_submissions`.

For multi-platform (Codeforces, GFG), use `(platform, slug)` as the universal key. CF slug = `f"{contestId}-{problemIndex}"`. GFG slug = path segment from URL.

---

## 6. Sync flow — three phases, ~100 seconds total

### Phase A — Bulk truth pass (~30–45s for 2,000 subs)

Walk global `submissionList(offset, limit:20)` paginated to the end, batched 10 pages per HTTP request via aliasing.

```
offset = 0
loop:
  send aliased query with 10 pages (offset, offset+20, ..., offset+180)
  upsert all submissions returned into user_submissions
  if any page returned hasNext: false (or fewer than 20 results) → done
  else offset += 200
```

This phase enumerates the user's entire problem universe naturally — no need for `problemsetQuestionList` first. Every problem the user has touched (AC or failed) shows up because we walk every submission.

After Phase A we know:
- Every problem ever attempted (deduplicated by slug)
- Every attempt's verdict, language, runtime, memory, timestamp
- Verdict mix per problem, per topic, per month
- Attempt count and time-to-solve per problem

UI is fully usable from this data alone — **80% of analytics work without code.**

### Phase B — Selective code fetch (~30–45s for ~1,100 codes)

Decide which submissions need code. Three categories per problem:

1. **AC problems:** fetch latest AC code per `(slug, lang)` — always
2. **AC problems with prior failures:** fetch the last failed attempt before the first AC — gives the "why brute force failed → optimization" diff (~30% of AC problems have this)
3. **TRIED-not-AC problems:** fetch latest attempt — these are problems the user is still stuck on, the highest-signal data for SR and weakness AI

For a 740-AC user: ~750 (latest AC) + ~250 (last-fail-before-AC) + ~100 (still-stuck) ≈ 1,100 codes.

Fetch via aliased `submissionDetails(submissionId)` batched 10 per HTTP request. ~110 round trips × ~400ms = ~45s.

### Phase C — Lazy on-demand code (post-onboarding)

For attempts beyond the three categories above, store metadata only in Phase A. If the user clicks "show all 12 attempts on Two Sum," fetch the rest on demand (still aliased, batched). Most users never trigger this.

### Live capture (continuous, post-sync)

Content script on `leetcode.com/problems/*` listens for the verdict-poll XHR (`/submissions/detail/<id>/check/`). When the verdict comes back, grab the submission ID from the URL and post it to the service worker. Worker calls one `submissionDetails(id)` and inserts into DB.

**Caveat:** requires the LC tab to be open at submit time. If the user submits via mobile or with the tab closed, live capture misses it — but the next periodic delta refresh (below) will catch it.

### Periodic delta refresh (every ~60 minutes)

`chrome.alarms.create({ periodInMinutes: 60 })` wakes the service worker. It calls `recentAcSubmissionList(limit: 20)`. Any IDs not already in our DB get fetched via batched `submissionDetails`. Cheap — ~1 round trip per check unless new submissions exist.

---

## 7. Service worker lifecycle (Manifest V3 reality)

Chrome kills idle service workers after ~30 seconds. Don't fight this — make the sync resumable.

State persisted to `chrome.storage.local` after every batch:

```ts
type SyncState = {
  user_id: string,
  platform: 'leetcode' | 'codeforces' | 'gfg',
  phase: 'A_bulk' | 'B_code' | 'live' | 'paused' | 'done',
  cursor: {
    offset?: number,                      // for Phase A
    code_queue?: number[],                // submission IDs remaining for Phase B
    code_done?: number,
  },
  last_progress_at: number,
  last_full_sync_at: number,
}
```

On worker wake (alarm or message), read state and continue from cursor. Idempotent: if a batch was in-flight when the worker died, refetch — `submissionDetails` is read-only and cheap.

Pause = user-controlled boolean in storage. Worker checks between batches.

**Tab open vs closed:**
- LC tab open, foreground/background: ✅ all sync runs
- LC tab closed, Chrome running: ✅ background sync continues (worker calls `leetcode.com/graphql` directly, doesn't need the tab) — only **live capture** stops
- Chrome closed: ❌ everything pauses; resumes on relaunch

---

## 8. Storage schema (platform-agnostic)

```sql
problems
  PK (platform, slug)              -- ('leetcode', 'two-sum')
  display_id, title, difficulty, topic_tags[], ac_rate, raw_meta jsonb
  -- shared across users; sync cost amortizes

user_problems                      -- the SOLVE GRAPH; UI's primary table
  PK (user_id, platform, slug)
  status: 'ac' | 'tried' | 'untouched'
  first_ac_at, latest_ac_at
  attempt_count, ac_count, wa_count, tle_count, mle_count, re_count, ce_count
  best_runtime_ms, best_memory_kb, best_lang
  primary_submission_id            -- FK into user_submissions, picked for code display

user_submissions                   -- attempt-level
  PK (platform, submission_id)
  user_id, slug (FK to problems.slug), verdict (normalized 7-bucket), lang
  runtime_ms, memory_kb, ts
  has_code: bool

user_submission_code               -- separate table; large blobs, fetched lazily
  PK (platform, submission_id)
  code text, code_lang, fetched_at

sync_state                         -- per (user, platform); also mirrored to extension storage
  user_id, platform
  phase, cursor jsonb
  last_progress_at, last_full_sync_at
```

Verdict normalization (7 buckets): `AC | WA | TLE | MLE | RE | CE | OTHER`. Keep raw `statusDisplay` in `user_submissions.raw_status` for debugging.

---

## 9. Multi-platform plan

Schema is platform-agnostic from day one. Sync workers are per-platform.

| Platform | Tier | Auth | Solve graph | Code fetch | Notes |
|---|---|---|---|---|---|
| **LeetCode** | 1 (ship first) | Cookie passthrough via `credentials: 'include'` | `submissionList` aliased | `submissionDetails` aliased | Full sync ~100s |
| **Codeforces** | 2 (ship +1 week) | Public REST API for graph; cookie for code | `user.status?handle=X&from=1&count=10000` — one call returns everything | HTML scrape of `codeforces.com/contest/{id}/submission/{sid}` | Generous rate limits (~1 req/2s recommended). Identity: `(contestId, problemIndex)` |
| **GeeksforGeeks** | 3 (only if demand) | Cookie passthrough; pure HTML scraping | Scrape user profile page | Scrape submission detail page (authed) | Fragile to UI changes. Indian-market specific |

Cross-platform problem linking (e.g. "Two Sum" on LC ≈ "Subarray with Given Sum" on GFG) is a **post-sync ML problem**, not a sync-time concern. Do it later via title/topic embedding similarity.

---

## 10. Onboarding UX

```
T=0       User installs extension. Popup detects leetcode.com cookie presence.
T=0       "Open LeetCode and log in" if no cookie detected.
T=2s      User clicks "Sync my LeetCode."
T=3s      Auth check (userStatus). Show username + Premium status.
T=3s      Phase A starts. Progress bar: "Loading your submissions… 200/2000"
T=35s     Phase A done. UI now shows: "740 problems solved, 18 topics, 2,143 submissions"
T=35s     Phase B starts. Progress bar: "Fetching your code… 100/1100"
T=80s     Phase B done. Full corpus loaded. AI features unlock.
```

Hide the first round-trip latency spike (we observed one 27s cold-cache hit) behind a "Setting up your sync…" indeterminate spinner — only show the deterministic progress bar from call 2 onwards.

Resumable: if the worker dies mid-sync, the next wake continues from `cursor`. UI shows "Resuming sync…"

---

## 11. Live capture mechanism

Content script registered for `https://leetcode.com/problems/*`:

```js
// Listen for the verdict-poll XHR
const origFetch = window.fetch;
window.fetch = async (url, opts) => {
  const res = await origFetch(url, opts);
  if (typeof url === 'string' && url.match(/\/submissions\/detail\/(\d+)\/check\//)) {
    const sid = url.match(/\/(\d+)\//)[1];
    const data = await res.clone().json();
    if (data.state === 'SUCCESS') {
      chrome.runtime.sendMessage({ type: 'NEW_SUBMISSION', submission_id: sid });
    }
  }
  return res;
};
```

Service worker on `NEW_SUBMISSION`: fetch one `submissionDetails(submissionId)`, upsert into DB, optionally show a toast ("Captured your AC on Two Sum").

---

## 12. What we still need to validate

| Item | Risk | How to verify |
|---|---|---|
| Free-tier behavior identical to Premium | Low — same GraphQL endpoints | Re-run probe on experiment account (`Lww7uHAhAB`) |
| Sustained 10-aliased batches don't trigger rate limits | Medium | Load test 100 sequential aliased requests with random sleeps |
| `chrome.alarms` interval in production (30 min minimum for non-Chrome-OS) | Low | Documented Chrome behavior |
| LC HTML changes breaking live-capture XHR hook | Medium | Monitor XHR signature; fallback to DOM observation |
| Chrome Web Store review approves cookie-passthrough use | Medium | LeetHub family precedent (4k+ users, in store for years) suggests yes |

---

## 13. Anti-goals (what we're explicitly NOT doing)

- ❌ Server-side scraping with user-uploaded cookies — destroys the trust moat
- ❌ MCP server distribution as primary channel — developer-toy reach (~100 stars max)
- ❌ Storing the user's password or session cookie value
- ❌ DOM scraping as primary capture path — fragile, LeetHub broke 3 times
- ❌ Premium-gated sync features at MVP — we tested on Premium but design for non-Premium parity
- ❌ Cross-user analytics that surface another user's submissions — no API supports it anyway

---

## 14. Competitive landscape (one-line summary per category)

- **LeetHub family** (4.3k+515+260 stars) — proves capture demand; ships code to user's GitHub; **no AI, no corpus**
- **LeetBuddy** (Chrome Web Store, 2k installs, 4.5★, weekly updates) — proves AI-on-LC distribution; **no auth, no history, no corpus**
- **jinzcdev MCP** (114 stars) — broadest read API surface; stateless; developer-only reach
- **interactive-leetcode-mcp** (10 stars) — closest hybrid (auth + AI); MCP-only; no persistence
- **Spaced-rep trackers** (LeetCodeAnki et al., <10 stars each) — metadata only; no code; no AI

**Our gap:** capture + corpus + corpus-dependent AI features. **Nobody has assembled this combination.** Speed-to-market matters because LeetBuddy could pivot in 3–6 months.

---

## 15. Open questions for the team

1. Where does the corpus live — managed Postgres, or our existing infra?
2. Do we encrypt code at rest? (Probably yes — user expectations.)
3. What's the deletion story when a user uninstalls? (Probably immediate purge.)
4. Are we OK with the Chrome Web Store review timeline (1–2 weeks first submission)?
5. Do we ship a Firefox port at v1 or v2?

---

## Appendix: probe artifacts

All numbers in this doc were measured against real LC accounts on 2026-05-22. Probe scripts were temporary (deleted post-validation). Re-run by:

1. Authenticate with `LEETCODE_SESSION` + `csrftoken` cookies
2. POST to `https://leetcode.com/graphql` with `credentials: 'include'`
3. Aliased queries as shown in §4

Key validated numbers:
- Auth check: ~600ms
- `submissionDetails` single call: ~300–400ms
- 10-aliased `submissionDetails`: ~350ms
- 10-aliased `submissionList` pages: ~3s for 200 submissions
- Per-slug `submissionList`: ~400ms, returns all attempts in one call (`hasNext: false`)
- `problemsetQuestionList(filter:AC)` for 740 problems: ~6s total across 15 paginated calls
