# Stage 2 Implementation Report: Batch Processing and Active Recommendation Generation

## 1. Purpose

This stage turns the Stage 1 foundation into a working batch recommendation system.

Stage 2 is responsible for generating, scoring, ranking, persisting, and serving recommendations through batch-oriented flows. It uses the tables and candidate-selection engine created in Stage 1.

Stage 2 creates the first production version of the active recommendation system:

```text
Historical submissions
  -> Batch trigger
  -> Stage 1 candidate engine
  -> Scoring
  -> Guardrails
  -> Diversity selection
  -> app.user_problem_recommendations
  -> app.recommendation_events
  -> Frontend recommendation surfaces
```

Stage 2 must support initial recommendations after historical sync, manual refresh, active-count refill, and daily refresh. It must not implement live event streaming yet; Stage 3 will add live submission and event projector processing.

---

## 2. Stage 2 Scope

### 2.1 In scope

Implement:

1. Batch refresh service.
2. Historical backfill recommendation generation.
3. Below-active-threshold refresh.
4. Manual refresh endpoint.
5. Daily refresh support for `today` and `dashboard` surfaces.
6. Final scoring and score breakdown persistence.
7. Diversity selector and active-set construction.
8. Recommendation upsert logic.
9. Recommendation expiry and cooldown defaults.
10. Internal event writing for batch lifecycle.
11. User-visible event writing for recommendation creation and frontend interactions.
12. Frontend fetch endpoint for active recommendations.
13. Frontend lifecycle endpoints for shown, clicked, started, skipped, dismissed.
14. Completion update from historical or batch-detected accepted submissions.
15. Batch observability, metrics, and tests.

### 2.2 Out of scope

Do not implement these in Stage 2:

1. Live submission event projector loop.
2. Continuous event stream polling.
3. `app.projector_cursors` usage beyond schema existence.
4. Email or WhatsApp sending.
5. LLM deciding recommendation ranking.
6. Separate recommendation job table.
7. Real-time replacement after every live submission.

---

## 3. Dependency on Stage 1

Stage 2 must call the Stage 1 candidate service. It must not duplicate candidate-generation logic.

Required Stage 1 function:

```python
async def preview_recommendation_candidates(
    user_id: str,
    platform: str,
    sheet_name: str,
    sheet_version: str,
    surface: str,
    trigger_source: str,
    limit: int,
    now: datetime | None = None,
) -> CandidatePreviewResult:
    ...
```

Stage 2 adds:

```text
candidate preview
  -> final selection
  -> persistence
  -> event writing
  -> serving APIs
```

---

## 4. Batch Processing Triggers

Stage 2 supports these triggers:

| Trigger source | Processing mode | When used |
|---|---|---|
| `initial_backfill` | Batch | After historical LeetCode sync completes |
| `below_active_threshold` | Batch | Active recommendation count falls below 5 |
| `manual_refresh` | Batch | User/admin explicitly requests refresh |
| `daily_refresh` | Batch | System prepares dashboard/today recommendations |
| `batch` | Batch | Generic internal batch refresh |

Stage 3 will add stronger processing for:

```text
live_submission
failed_submission
accepted_submission
recommendation_interaction
```

Stage 2 may write or accept those values only when they are passed by an upstream caller, but live stream processing must remain Stage 3 responsibility.

---

## 5. Active Recommendation Count Rules

Use these constants:

```text
MAX_ACTIVE_RECOMMENDATIONS = 10
MIN_ACTIVE_RECOMMENDATION_THRESHOLD = 5
DEFAULT_BATCH_TARGET_COUNT = 10
DEFAULT_FRONTEND_LIMIT = 10
```

Active count includes rows with statuses:

```text
pending, active, shown
```

Rows that are `clicked` or `started` must not be casually replaced, but they should not be counted as refill capacity unless product wants them shown in the same active widget. Treat them as protected in-progress rows.

Rows with these statuses do not count as active:

```text
completed, skipped, dismissed, expired, replaced
```

---

## 6. Batch Service Layout

Extend the recommendation package:

```text
api/services/recommendations/
  refresh_service.py
  persistence.py
  lifecycle_service.py
  batch_triggers.py
  serving_service.py
  metrics.py
```

### 6.1 File responsibilities

| File | Responsibility |
|---|---|
| `refresh_service.py` | Orchestrates batch refresh execution |
| `persistence.py` | Upserts recommendations and updates lifecycle timestamps |
| `lifecycle_service.py` | Handles shown/clicked/started/skip/dismiss/completed transitions |
| `batch_triggers.py` | Trigger-specific wrappers for initial backfill, below threshold, daily refresh |
| `serving_service.py` | Reads active recommendations for frontend surfaces |
| `metrics.py` | Emits refresh metrics and event-derived counters |

---

## 7. Batch Refresh Function Contract

Implement this function:

```python
async def refresh_user_recommendations(
    user_id: str,
    platform: str = "leetcode",
    trigger_source: str = "manual_refresh",
    surface: str = "dashboard",
    sheet_name: str = "default_dsa_sheet",
    sheet_version: str = "v1",
    request_id: str | None = None,
    session_id: str | None = None,
    force: bool = False,
) -> RefreshResult:
    ...
```

### 7.1 `RefreshResult`

```python
class RefreshResult:
    user_id: str
    platform: str
    trigger_source: str
    surface: str
    skipped: bool
    skip_reason: str | None
    active_count_before: int
    generated_count: int
    filtered_count: int
    selected_count: int
    inserted_count: int
    updated_count: int
    protected_count: int
    event_ids: list[int]
    duration_ms: int
```

---

## 8. Idempotency Rules

Every batch refresh must use idempotency keys through `app.recommendation_events.idempotency_key`.

### 8.1 Idempotency key patterns

```text
initial_backfill:{user_id}:{platform}:{last_full_sync_at}
below_threshold:{user_id}:{platform}:{yyyy-mm-dd-hh}
manual_refresh:{user_id}:{platform}:{request_id}
daily_refresh:{user_id}:{platform}:{yyyy-mm-dd}:{surface}
batch:{user_id}:{platform}:{request_id_or_bucket}
```

### 8.2 Duplicate behavior

If an idempotency key already exists:

1. Do not run the refresh again.
2. Write `recommendation_generation_skipped` only if a distinct skip event is useful and the idempotency key for the skip event will not collide.
3. Return `RefreshResult(skipped=True, skip_reason='duplicate_idempotency_key')`.

---

## 9. Batch Refresh Algorithm

### 9.1 Main sequence

```text
1. Build idempotency key.
2. Check duplicate idempotency key.
3. Write `refresh_requested` internal event.
4. Write `refresh_started` internal event.
5. Load candidate preview from Stage 1.
6. Check active count and trigger rules.
7. Select final recommendation set.
8. Assign ranks, expiry, priority, and metadata.
9. Upsert selected recommendations.
10. Write `recommendation_created` events for new rows.
11. Write `batch_recommendations_generated` internal event.
12. Write `refresh_completed` internal event.
13. Return refresh result.
```

### 9.2 Pseudocode

```python
async def refresh_user_recommendations(...):
    started_at = monotonic_time()
    idempotency_key = build_batch_idempotency_key(...)

    if await event_writer.exists_idempotency_key(user_id, idempotency_key):
        return RefreshResult(skipped=True, skip_reason="duplicate_idempotency_key")

    await event_writer.write_internal(
        user_id=user_id,
        event_type="refresh_requested",
        visibility="internal",
        platform=platform,
        surface=surface,
        idempotency_key=idempotency_key,
        request_id=request_id,
        session_id=session_id,
        metadata={
            "trigger_source": trigger_source,
            "processing_mode": "batch"
        },
    )

    await event_writer.write_internal(
        user_id=user_id,
        event_type="refresh_started",
        visibility="internal",
        platform=platform,
        surface=surface,
        request_id=request_id,
        session_id=session_id,
        metadata={
            "trigger_source": trigger_source,
            "processing_mode": "batch",
            "scoring_version": "mvp_v1"
        },
    )

    try:
        preview = await candidate_service.preview_recommendation_candidates(
            user_id=user_id,
            platform=platform,
            sheet_name=sheet_name,
            sheet_version=sheet_version,
            surface=surface,
            trigger_source=trigger_source,
            limit=50,
        )

        if trigger_source == "below_active_threshold" and preview.active_count >= 5 and not force:
            await event_writer.write_internal(
                user_id=user_id,
                event_type="recommendation_generation_skipped",
                visibility="internal",
                platform=platform,
                surface=surface,
                metadata={
                    "reason": "active_count_sufficient",
                    "active_count": preview.active_count,
                    "minimum_threshold": 5
                },
            )
            return RefreshResult(skipped=True, skip_reason="active_count_sufficient")

        selected = diversity.select_final_set(
            state=preview.state,
            candidates=preview.candidates,
            max_count=10,
            surface=surface,
        )

        selected = assign_rank_priority_expiry_and_metadata(selected, trigger_source, surface)

        persisted = await persistence.upsert_recommendations(selected)
        await event_writer.write_created_events(persisted, trigger_source=trigger_source)

        await event_writer.write_internal(
            user_id=user_id,
            event_type="batch_recommendations_generated",
            visibility="internal",
            platform=platform,
            surface=surface,
            metadata={
                "processing_mode": "batch",
                "trigger_source": trigger_source,
                "candidate_count": preview.generated_count,
                "filtered_count": preview.filtered_count,
                "selected_count": len(selected),
                "inserted_count": persisted.inserted_count,
                "updated_count": persisted.updated_count,
                "scoring_version": "mvp_v1"
            },
        )

        await event_writer.write_internal(
            user_id=user_id,
            event_type="refresh_completed",
            visibility="internal",
            platform=platform,
            surface=surface,
            metadata={
                "processing_mode": "batch",
                "trigger_source": trigger_source,
                "duration_ms": elapsed_ms(started_at),
                "selected_count": len(selected)
            },
        )

        return RefreshResult(skipped=False, selected_count=len(selected))

    except Exception as exc:
        await event_writer.write_internal(
            user_id=user_id,
            event_type="refresh_failed",
            visibility="internal",
            platform=platform,
            surface=surface,
            metadata={
                "processing_mode": "batch",
                "trigger_source": trigger_source,
                "error_message": safe_error_message(exc),
                "retryable": is_retryable(exc)
            },
        )
        raise
```

---

## 10. Final Selection and Diversity

Stage 2 must take scored candidates from Stage 1 and produce the final active set.

### 10.1 Sort order

Sort candidates by:

```text
priority desc
score desc
recommendation_type_priority asc
freshness_score desc
problem_order asc
```

### 10.2 Selection caps

| Cap | Limit |
|---|---:|
| Total active recommendations | 10 |
| Same topic in top 5 | 2 |
| Same topic in full active set | 3 |
| `failed_submission_retry` active | 2 |
| `weak_topic_practice` active | 3 |
| `revision` active | 2 |
| `next_in_topic` active | 3 |
| `similar_problem` active | 2 |
| `next_difficulty` active | 1 |

### 10.3 Daily set packaging

For `surface = 'today'`, select up to 5 rows using this preferred slot structure:

| Slot | Preferred type |
|---:|---|
| 1 | `failed_submission_retry` or `weak_topic_practice` |
| 2 | `next_in_topic` |
| 3 | `standard_practice` |
| 4 | `revision` |
| 5 | `similar_problem` or `next_difficulty` |

If a slot type has no valid candidate, fill with `standard_practice` or the next highest-scoring safe candidate.

---

## 11. Persistence Rules

### 11.1 Upsert key

Upsert recommendations by:

```text
user_id + platform + problem_slug + recommendation_type
```

### 11.2 Insert behavior

For new selected rows:

```text
status = pending
generated_at = now()
created_at = now()
updated_at = now()
```

Write a `recommendation_created` event.

### 11.3 Update behavior

If an existing row has status:

```text
completed, skipped, dismissed, expired, replaced
```

and cooldown has passed, update:

```text
status
surface
priority
rank
score
reason_code
reason
llm_reason = existing llm_reason or null
score_breakdown
evidence
scoring_version
trigger_source
generated_at
expires_at
cooldown_until
metadata
updated_at
```

### 11.4 Protected rows

Do not update or replace rows where:

```text
status in ('clicked', 'started')
and (expires_at is null or expires_at > now())
```

These rows indicate that the user may be working on the recommendation.

### 11.5 Duplicate problem conflict

If the same problem qualifies for multiple recommendation types:

1. Keep the highest-priority valid type.
2. Persist only that type unless there is a deliberate product need for separate lifecycle reasons.
3. Store alternate type evidence in `metadata.alternate_candidate_types`.

---

## 12. Expiry and Cooldown Defaults

### 12.1 Expiry defaults

| Recommendation type | Expiry |
|---|---|
| `standard_practice` | 7 days |
| `weak_topic_practice` | 7 days |
| `failed_submission_retry` | 3 days |
| `revision` | 14 days |
| `next_in_topic` | 7 days |
| `similar_problem` | 3 days |
| `next_difficulty` | 7 days |
| `daily_practice` | 24 hours |

### 12.2 Cooldown after skip or dismiss

| Recommendation type | Cooldown |
|---|---|
| `standard_practice` | 21 days |
| `weak_topic_practice` | 14 days |
| `failed_submission_retry` | 7 days |
| `revision` | 30 days |
| `next_in_topic` | 14 days |
| `similar_problem` | 7 days |
| `next_difficulty` | 14 days |
| `daily_practice` | 3 days |

---

## 13. Batch Event Writing

Stage 2 must write internal and user-visible events into `app.recommendation_events`.

### 13.1 Internal events

| Event | When to write |
|---|---|
| `refresh_requested` | Before refresh starts |
| `refresh_started` | Refresh execution begins |
| `refresh_completed` | Refresh succeeds |
| `refresh_failed` | Refresh fails |
| `historical_submission_processed` | Historical sync data is ready for recommendation refresh |
| `batch_recommendations_generated` | Batch generation creates or updates recommendation rows |
| `recommendation_generation_skipped` | Refresh skipped because active count/idempotency/guardrails block it |

### 13.2 User-visible events

| Event | When to write |
|---|---|
| `recommendation_created` | New recommendation row is inserted |
| `recommendation_shown` | Frontend confirms display |
| `recommendation_clicked` | User clicks/open attempt |
| `recommendation_started` | User begins attempting the problem from UI |
| `recommendation_skipped` | User skips recommendation |
| `recommendation_dismissed` | User dismisses recommendation |
| `recommendation_completed` | Batch or sync detects accepted solution for active recommendation |
| `recommendation_expired` | Expiry cleanup marks row expired |
| `recommendation_replaced` | Batch replaces a stale or unsafe recommendation |

### 13.3 Internal event metadata

Use this structure for refresh completion:

```json
{
  "visibility": "internal",
  "trigger_source": "below_active_threshold",
  "processing_mode": "batch",
  "scoring_version": "mvp_v1",
  "active_count_before": 4,
  "candidate_count": 58,
  "filtered_count": 12,
  "selected_count": 8,
  "inserted_count": 6,
  "updated_count": 2,
  "protected_count": 1,
  "duration_ms": 420
}
```

---

## 14. Historical Backfill Integration

After historical sync completes and `user_data.user_coding_problems` is recomputed:

```text
1. Write `historical_submission_processed` internal event.
2. Call `refresh_user_recommendations` with trigger_source = initial_backfill.
3. Generate up to 10 active recommendations.
4. Write `batch_recommendations_generated`.
5. Write `refresh_completed`.
```

### 14.1 Historical backfill idempotency

Use:

```text
initial_backfill:{user_id}:{platform}:{last_full_sync_at}
```

If `last_full_sync_at` is null, use a stable sync completion marker from `system.sync_state`.

### 14.2 Historical processing rules

1. Use accepted submissions to identify solved history and progress.
2. Use failed submissions to identify weak topics and failed retry opportunities.
3. Prefer a balanced initial set of 10 recommendations.
4. Do not require live capture data.
5. If sync is incomplete or stale, prefer `standard_practice` and safe `next_in_topic`.

---

## 15. Below-Threshold Batch Refresh

When active recommendations fall below 5:

```text
1. Write refresh_requested with trigger_source = below_active_threshold.
2. Check active count again inside the refresh transaction.
3. If active count is still below 5, generate recommendations.
4. Fill up to max 10 active recommendations.
5. Write batch events.
```

### 15.1 Trigger condition

```sql
select count(*)
from app.user_problem_recommendations
where user_id = $1
  and platform = $2
  and status in ('pending', 'active', 'shown')
  and (expires_at is null or expires_at > now())
  and (cooldown_until is null or cooldown_until <= now());
```

If count is less than 5, refresh is eligible.

### 15.2 Race-condition handling

Use one of these locking strategies:

1. Transaction-level advisory lock by `user_id + platform`.
2. Serializable transaction around active count and upsert.
3. Existing application worker single-flight lock.

Do not rely only on application memory for production concurrency control.

---

## 16. Daily Refresh

Daily refresh prepares recommendations for the `today` surface.

Rules:

1. Use `trigger_source = daily_refresh`.
2. Use `surface = today`.
3. Prefer 5 rows.
4. Use daily slot packaging.
5. Expire daily recommendations after 24 hours.
6. Do not generate daily rows if the user already has fresh `today` rows.

Idempotency key:

```text
daily_refresh:{user_id}:{platform}:{yyyy-mm-dd}:today
```

---

## 17. Frontend Serving API

### 17.1 Fetch recommendations

```text
GET /api/recommendations
```

Query params:

```text
surface=dashboard|today|weak_topics|retry|revision|notification|chat
limit=10
```

SQL shape:

```sql
select upr.*, cp.title, cp.difficulty, cp.raw_meta
from app.user_problem_recommendations upr
join catalog.coding_problems cp
  on cp.platform = upr.platform
 and cp.slug = upr.problem_slug
where upr.user_id = $1
  and upr.platform = $2
  and upr.surface = coalesce($3, upr.surface)
  and upr.status in ('pending', 'active', 'shown', 'clicked', 'started')
  and (upr.expires_at is null or upr.expires_at > now())
  and (upr.cooldown_until is null or upr.cooldown_until <= now())
order by upr.priority desc, upr.rank asc, upr.score desc
limit $4;
```

Rules:

1. Fetching recommendations must not automatically mark rows as shown.
2. Mark shown only when frontend confirms actual display.
3. Do not return rows with expired or cooldown-blocked state.
4. Do not return raw code or submission debug payload.

---

## 18. Frontend Lifecycle APIs

### 18.1 Mark shown

```text
POST /api/recommendations/{id}/shown
```

Update:

```text
status = shown
shown_at = coalesce(shown_at, now())
updated_at = now()
```

Write:

```text
recommendation_shown
```

### 18.2 Mark clicked

```text
POST /api/recommendations/{id}/clicked
```

Update:

```text
status = clicked
clicked_at = coalesce(clicked_at, now())
updated_at = now()
```

Write:

```text
recommendation_clicked
```

### 18.3 Mark started

```text
POST /api/recommendations/{id}/started
```

Update:

```text
status = started
started_at = coalesce(started_at, now())
updated_at = now()
```

Write:

```text
recommendation_started
```

### 18.4 Skip

```text
POST /api/recommendations/{id}/skip
```

Update:

```text
status = skipped
skipped_at = now()
cooldown_until = now() + type cooldown
updated_at = now()
```

Write:

```text
recommendation_skipped
```

After skip, check active count. If below 5, trigger below-threshold batch refresh.

### 18.5 Dismiss

```text
POST /api/recommendations/{id}/dismiss
```

Update:

```text
status = dismissed
dismissed_at = now()
cooldown_until = now() + type cooldown
updated_at = now()
```

Write:

```text
recommendation_dismissed
```

After dismiss, check active count. If below 5, trigger below-threshold batch refresh.

---

## 19. Completion from Batch-Detected Accepted Submissions

Stage 2 may mark recommendations completed when batch or historical sync detects an accepted solution for an active recommendation.

Update:

```sql
update app.user_problem_recommendations
set status = 'completed',
    completed_at = coalesce(completed_at, now()),
    updated_at = now(),
    metadata = metadata || jsonb_build_object('completion_source', 'batch_or_historical_sync')
where user_id = $1
  and platform = $2
  and problem_slug = $3
  and status in ('pending', 'active', 'shown', 'clicked', 'started');
```

Write:

```text
recommendation_completed
```

Stage 3 will perform this in near real time for live submissions.

---

## 20. Expiry Cleanup

Implement a batch cleanup routine that marks expired rows.

Rules:

1. Select rows where `expires_at <= now()` and status is pending/active/shown.
2. Update status to `expired`.
3. Set `updated_at = now()`.
4. Write `recommendation_expired` event.
5. After expiry, check active count and trigger below-threshold refresh if needed.

Do not delete expired recommendation rows.

---

## 21. LLM Boundary in Stage 2

Stage 2 must generate deterministic `reason` without LLM dependency.

LLM usage is optional and limited to:

```text
llm_reason
notification text
friendly explanation
```

Do not let the LLM decide:

```text
candidate selection
recommendation type
score
rank
hard filters
diversity
cooldown
expiry
```

If LLM generation is not available:

1. Keep the recommendation.
2. Use deterministic `reason`.
3. Leave `llm_reason` null.

For Stage 2, LLM integration can be skipped entirely.

---

## 22. Future Notification Modularity

Stage 2 must prepare data for future notification delivery without sending email or WhatsApp messages.

Rules:

1. Keep `surface = 'notification'` supported in `app.user_problem_recommendations`.
2. Store notification-safe evidence in `evidence` and `metadata`.
3. Store deterministic fallback text in `reason`.
4. Do not add provider-specific fields such as WhatsApp IDs or email message IDs to recommendation core logic.
5. Future notification delivery must consume recommendations as output, not influence ranking.

Suggested future-safe metadata:

```json
{
  "notification": {
    "eligible": true,
    "priority": "normal",
    "quiet_hours_respected": true,
    "requires_user_unseen_window": true,
    "copy_source": "reason"
  }
}
```

Stage 3 will define the event-driven notification escalation hook.

---

## 23. Production-Grade Batch Rules

### 23.1 Transactions

Use transactions for:

1. Active count check and selection persistence.
2. Recommendation upsert and `recommendation_created` events.
3. Lifecycle update and lifecycle event write.

If a recommendation row update succeeds but event writing fails, the system loses auditability. Keep row update and event write in one transaction where possible.

### 23.2 Concurrency

Use one user-platform refresh lock per refresh execution.

Recommended lock key:

```text
recommendations:{user_id}:{platform}
```

### 23.3 Error handling

1. Write `refresh_failed` with safe error metadata.
2. Mark errors retryable only when retry is safe.
3. Do not expose stack traces to frontend.
4. Do not partially replace active recommendations if selection fails.

### 23.4 Data minimization

Do not store raw code, expected output, actual output, compile error, runtime error, or last testcase inside recommendation rows or user-visible events.

### 23.5 Feature flags

Use feature flags for:

```text
recommendations_enabled
batch_refresh_enabled
daily_refresh_enabled
manual_refresh_enabled
llm_reason_enabled
notification_surface_enabled
```

---

## 24. Metrics and Observability

Track these metrics from events and logs:

| Metric | Source |
|---|---|
| Generation success rate | `refresh_completed / refresh_started` |
| Generation failure rate | `refresh_failed / refresh_started` |
| Average candidate count | `refresh_completed.metadata.candidate_count` |
| Average selected count | `refresh_completed.metadata.selected_count` |
| Active recommendation health | Count active rows per user |
| Impression count | `recommendation_shown` |
| Click-through rate | `recommendation_clicked / recommendation_shown` |
| Start rate | `recommendation_started / recommendation_shown` |
| Completion rate | `recommendation_completed / recommendation_started` |
| Skip rate | `recommendation_skipped / recommendation_shown` |
| Dismiss rate | `recommendation_dismissed / recommendation_shown` |
| Retry success rate | Completed `failed_submission_retry` rows / started retry rows |
| Revision completion rate | Completed `revision` rows / shown revision rows |

Refresh logs must include:

```text
request_id
user_id hash/internal id
platform
trigger_source
surface
candidate_count
selected_count
inserted_count
updated_count
protected_count
duration_ms
skip_reason
error_code
```

Do not log raw submitted code.

---

## 25. Stage 2 Test Plan

### 25.1 Unit tests

| Module | Required tests |
|---|---|
| `refresh_service.py` | successful refresh, skipped duplicate, skipped active sufficient, refresh failed |
| `persistence.py` | insert, update expired row, protect clicked/started row, cooldown passed update |
| `lifecycle_service.py` | shown, clicked, started, skip, dismiss, complete transitions |
| `diversity.py` | topic caps, type caps, rank assignment, daily slots |
| `event_writer.py` | internal events, user events, idempotency keys |
| `serving_service.py` | status filtering, expiry filtering, cooldown filtering, surface filtering |

### 25.2 Scenario tests

#### New user initial refresh

Input:

```text
0 submissions
seeded beginner curated sheet
trigger_source = initial_backfill
```

Expected:

```text
standard_practice and first next_in_topic rows
no failed_submission_retry
no revision
no next_difficulty
```

#### Weak topic batch refresh

Input:

```text
8 DP attempts
1 solved
7 failed attempts
active count = 3
```

Expected:

```text
weak_topic_practice rows generated
active count refilled toward 10
topic diversity cap enforced
```

#### Failed retry batch refresh

Input:

```text
3 WA on curated problem
0 AC
latest attempt yesterday
near_solve_score = 0.80
```

Expected:

```text
failed_submission_retry created
reason_code = almost_solved_retry
high failed_attempt_score
```

#### Revision due batch refresh

Input:

```text
Solved important curated problem 70 days ago
```

Expected:

```text
revision row generated
revision_due_score uses 0.75 time due score
```

#### Active count sufficient

Input:

```text
active count = 7
trigger_source = below_active_threshold
```

Expected:

```text
refresh skipped
recommendation_generation_skipped written
no new recommendation rows
```

#### Protected started row

Input:

```text
existing row status = started
same problem selected again
```

Expected:

```text
existing row not replaced
protected_count increments
```

### 25.3 Integration tests

Run full flow:

```text
1. Apply Stage 1 migrations.
2. Seed curated sheet.
3. Insert catalog problems.
4. Insert user submissions.
5. Recompute user aggregates.
6. Run initial_backfill refresh.
7. Assert recommendation rows exist.
8. Assert refresh events exist.
9. Fetch recommendations from frontend endpoint.
10. Mark shown, clicked, skipped.
11. Assert lifecycle events exist.
12. Trigger below-threshold refresh.
13. Assert cooldown and duplicate guardrails hold.
```

---

## 26. Stage 2 Edge Cases

### 26.1 User has no submissions

Use beginner-friendly `standard_practice` and initial `next_in_topic`. Do not use failure-based or revision-based recommendations.

### 26.2 User has only failed submissions

Use `failed_submission_retry`, `weak_topic_practice`, and `standard_practice`. Avoid `next_difficulty`.

### 26.3 User has only accepted submissions

Use `next_in_topic`, `revision`, `similar_problem`, `next_difficulty`, and `standard_practice` when eligible.

### 26.4 Curated sheet has too few candidates

Generate from available valid rows. Do not force a type if no valid candidate exists. Fill remaining slots with `standard_practice` if possible.

### 26.5 Active recommendation count is already 10

Do not create additional active rows. Allow re-ranking only if product requires it and protected rows are respected.

### 26.6 User dismisses many recommendations

Apply cooldown, increase diversity, and prefer safer recommendations in the next refresh.

### 26.7 LLM reason generation fails

Do not block persistence. Use deterministic `reason` and leave `llm_reason` null.

---

## 27. Stage 2 Acceptance Criteria

Stage 2 is complete when:

1. Batch refresh service exists and calls Stage 1 candidate engine.
2. Initial historical backfill refresh generates active recommendations.
3. Below-threshold refresh refills active recommendations when count falls below 5.
4. Manual refresh endpoint works with idempotency.
5. Daily refresh works for the `today` surface.
6. Recommendations are persisted in `app.user_problem_recommendations`.
7. Internal refresh events are written to `app.recommendation_events`.
8. User-visible lifecycle events are written.
9. Frontend fetch endpoint returns eligible active rows.
10. Shown, clicked, started, skip, and dismiss endpoints update lifecycle state.
11. Completion can be marked from batch/historical accepted data.
12. Expiry cleanup marks stale rows expired.
13. Duplicate, solved, too-hard, inactive, and cooldown-blocked recommendations are prevented.
14. Diversity caps are enforced.
15. Refresh operations are idempotent.
16. Tests pass for batch generation, persistence, serving, lifecycle, and metrics.
17. No separate recommendation jobs table is introduced.
18. No live event projector is required for Stage 2.
19. Email and WhatsApp delivery are not implemented but notification surface data remains future-ready.

---

## 28. Handoff to Stage 3

Stage 3 must consume these Stage 2 contracts:

```python
async def refresh_user_recommendations(...):
    ...

async def mark_recommendation_completed_from_submission(
    user_id: str,
    platform: str,
    problem_slug: str,
    submission_id: str,
    source: str,
) -> LifecycleUpdateResult:
    ...

async def get_active_recommendation_count(
    user_id: str,
    platform: str,
) -> int:
    ...
```

Stage 3 must add live event processing without rewriting candidate generation or batch persistence.
