# Stage 3 Implementation Report: Event Streaming, Live Submission Processing, and Notification Readiness

## 1. Purpose

This stage completes the hybrid recommendation system by adding event-based processing on top of the Stage 1 foundation and Stage 2 batch system.

Stage 3 handles live submissions, recommendation interaction events, projector cursor progress, real-time lifecycle updates, event-triggered recommendation refresh, and future notification escalation hooks.

The final system after Stage 3 is:

```text
Curated DSA Sheet
  -> Stage 1 candidate engine
  -> Stage 2 batch generation
  -> Active recommendations

Live submissions and recommendation interactions
  -> app.recommendation_events
  -> Event projector
  -> app.projector_cursors
  -> Lifecycle updates
  -> Event-triggered refresh
  -> Active recommendations
  -> Future notification escalation
```

Stage 3 must not replace batch processing. It adds live responsiveness while keeping batch refresh as the safety net.

---

## 2. Stage 3 Scope

### 2.1 In scope

Implement:

1. Live submission integration events.
2. Recommendation event projector loop.
3. `app.projector_cursors` read/update logic.
4. Event idempotency handling.
5. Live accepted submission processing.
6. Live failed submission processing.
7. Recommendation lifecycle updates from submissions.
8. Event-triggered refresh using Stage 2 refresh service.
9. Urgent strong-signal refresh behavior.
10. Interaction-driven refresh after skip/dismiss/feedback.
11. Event-stream observability and dead-letter style failure metadata.
12. Future notification escalation service boundary.
13. Future email/WhatsApp provider interface boundary without provider-specific implementation.
14. Tests for event processing, cursor progress, idempotency, and lifecycle updates.

### 2.2 Out of scope

Do not implement these unless a separate product decision enables them:

1. Actual email sending.
2. Actual WhatsApp API sending.
3. Push notification provider integration.
4. Separate notification delivery table.
5. LLM-controlled recommendation decision-making.
6. Replacing the deterministic scorer with a black-box model.
7. Deleting event history.

Stage 3 must prepare the system so email and WhatsApp delivery can be added later without changing recommendation selection or ranking logic.

---

## 3. Dependency on Stage 1 and Stage 2

Stage 3 must reuse:

| Dependency | Owned by | Usage in Stage 3 |
|---|---|---|
| Tables and schemas | Stage 1 | Event log, recommendation rows, projector cursors |
| Candidate engine | Stage 1 | Used indirectly through refresh service |
| Batch refresh service | Stage 2 | Called after live events when refresh is needed |
| Lifecycle service | Stage 2 | Marks completed, started, skipped, dismissed |
| Event writer | Stage 1/2 | Writes internal and user-visible events |
| Serving table | Stage 2 | Updated after event processing |

Stage 3 must not duplicate candidate generators, scoring formula, diversity logic, or persistence upsert behavior.

---

## 4. Event-Based Architecture

Stage 3 uses `app.recommendation_events` as the event log.

The event projector reads internal processing events from this table and advances through `app.projector_cursors`.

```text
Live submission stored
  -> Write live_submission_processed
  -> Write accepted_submission_processed or failed_submission_processed
  -> Event projector reads new events
  -> Projector updates recommendation lifecycle
  -> Projector calls refresh service if needed
  -> Projector advances cursor
```

For V1, polling `app.recommendation_events` is enough. A message broker can be added later behind the same projector interface, but it must not change event payload contracts.

---

## 5. Event Sources

Stage 3 consumes these event types:

| Event type | Visibility | Meaning |
|---|---|---|
| `live_submission_processed` | internal | A live LeetCode submission was stored |
| `accepted_submission_processed` | internal | Accepted submission signal is ready |
| `failed_submission_processed` | internal | Failed submission signal is ready |
| `recommendation_skipped` | user | User skipped a recommendation |
| `recommendation_dismissed` | user | User dismissed a recommendation |
| `recommendation_marked_not_relevant` | user | User gave negative relevance feedback |
| `recommendation_skipped_already_solved` | user | User says recommendation was already solved |
| `recommendation_completed` | user | Completion should influence future state |
| `refresh_requested` | internal | Optional event source for queued refresh behavior |

Stage 3 writes these output events:

| Event type | Visibility | When written |
|---|---|---|
| `event_recommendations_generated` | internal | Live/event processing generates recommendations |
| `recommendation_completed` | user | Live AC completes active recommendation |
| `recommendation_started` | user | Live non-AC marks active recommendation as started |
| `recommendation_replaced` | user | Overly difficult active recommendation replaced |
| `refresh_requested` | internal | Event requires refresh |
| `refresh_started` | internal | Refresh begins through Stage 2 service |
| `refresh_completed` | internal | Refresh succeeds |
| `refresh_failed` | internal | Refresh fails |
| `recommendation_generation_skipped` | internal | Event does not require generation |

---

## 6. Live Submission Integration

After `ingest_live_submission` stores a submission and recomputes `user_data.user_coding_problems`, it must write recommendation-processing events.

### 6.1 Required event writes

For every live submission:

```text
1. Write live_submission_processed.
2. If verdict = AC, write accepted_submission_processed.
3. If verdict != AC, write failed_submission_processed.
```

### 6.2 Idempotency keys

```text
live_submission:{user_id}:{platform}:{submission_id}
accepted_submission:{user_id}:{platform}:{submission_id}
failed_submission:{user_id}:{platform}:{submission_id}
```

### 6.3 Metadata shape

```json
{
  "visibility": "internal",
  "processing_mode": "event",
  "source": "live_submission_sync",
  "submission_id": "123456",
  "profile_id": "profile-id",
  "verdict": "WA",
  "raw_status": "Wrong Answer",
  "problem_slug": "two-sum",
  "difficulty": "easy",
  "capture_source": "live_capture",
  "has_code": true,
  "submitted_at": "2026-06-05T10:15:00Z",
  "aggregate_recomputed": true
}
```

Do not store submitted code or raw testcase details in recommendation events.

---

## 7. Projector Cursor Contract

Use `app.projector_cursors` to track event progress.

### 7.1 Cursor identity

Recommended cursor:

```text
projector_name = live_recommendation_projector_v1
event_source = app.recommendation_events
```

### 7.2 Cursor read

```sql
select *
from app.projector_cursors
where projector_name = $1
  and event_source = $2;
```

If no cursor exists, create one with:

```text
last_processed_event_id = 0
last_processed_at = null
```

### 7.3 Event polling query

```sql
select *
from app.recommendation_events
where id > $1
  and event_type in (
    'live_submission_processed',
    'accepted_submission_processed',
    'failed_submission_processed',
    'recommendation_skipped',
    'recommendation_dismissed',
    'recommendation_marked_not_relevant',
    'recommendation_skipped_already_solved',
    'recommendation_completed',
    'refresh_requested'
  )
order by id asc
limit $2;
```

### 7.4 Cursor update

Update the cursor only after the event is successfully processed or safely skipped.

```sql
update app.projector_cursors
set last_processed_event_id = $1,
    last_processed_at = now(),
    metadata = $2,
    updated_at = now()
where projector_name = $3
  and event_source = $4;
```

### 7.5 Cursor metadata

```json
{
  "last_batch_size": 100,
  "last_successful_event_id": 12345,
  "last_error_event_id": null,
  "last_error_message": null,
  "projector_version": "v1",
  "processing_mode": "polling"
}
```

---

## 8. Projector Processing Algorithm

### 8.1 Main loop

```text
1. Acquire projector lock.
2. Read cursor.
3. Fetch events with id greater than cursor.
4. Process events in ascending id order.
5. For each event, run handler inside a transaction.
6. Write output events and lifecycle updates.
7. Advance cursor after successful or safe-skip handling.
8. On retryable error, do not advance cursor beyond failed event.
9. On non-retryable poison event, write failure metadata and safe-skip with explicit reason.
```

### 8.2 Pseudocode

```python
async def run_live_recommendation_projector(batch_size: int = 100):
    async with advisory_lock("live_recommendation_projector_v1"):
        cursor = await cursor_repo.get_or_create(
            projector_name="live_recommendation_projector_v1",
            event_source="app.recommendation_events",
        )

        events = await event_repo.fetch_events_after(
            after_id=cursor.last_processed_event_id or 0,
            limit=batch_size,
        )

        for event in events:
            try:
                async with db.transaction():
                    result = await process_event(event)
                    await cursor_repo.advance(
                        projector_name="live_recommendation_projector_v1",
                        event_source="app.recommendation_events",
                        last_processed_event_id=event.id,
                        metadata={
                            "last_result": result.status,
                            "last_event_type": event.event_type,
                            "projector_version": "v1"
                        },
                    )
            except RetryableProjectorError:
                await cursor_repo.update_error_metadata(event)
                raise
            except NonRetryableProjectorError as exc:
                await event_writer.write_internal(
                    user_id=event.user_id,
                    event_type="recommendation_generation_skipped",
                    visibility="internal",
                    platform=event.platform,
                    metadata={
                        "reason": "non_retryable_projector_error",
                        "source_event_id": event.id,
                        "source_event_type": event.event_type,
                        "error_message": safe_error_message(exc)
                    },
                )
                await cursor_repo.advance(...)
```

---

## 9. Accepted Submission Event Handling

When processing `accepted_submission_processed`:

### 9.1 Required actions

```text
1. Read user_id, platform, problem_slug, submission_id from event metadata.
2. Mark matching active recommendation rows as completed.
3. Write recommendation_completed if a row was completed and no duplicate event exists.
4. Call refresh service if active count falls below 5 or next-in-topic/similar recommendation should be generated.
5. Write event_recommendations_generated if refresh creates rows.
```

### 9.2 Lifecycle update

```sql
update app.user_problem_recommendations
set status = 'completed',
    completed_at = coalesce(completed_at, now()),
    updated_at = now(),
    metadata = metadata || jsonb_build_object(
      'completion_source', 'live_submission',
      'submission_id', $4
    )
where user_id = $1
  and platform = $2
  and problem_slug = $3
  and status in ('pending', 'active', 'shown', 'clicked', 'started');
```

### 9.3 Refresh trigger rules

Call `refresh_user_recommendations` with:

```text
trigger_source = accepted_submission
processing_mode = event
```

when any of these is true:

1. Active count after completion is below 5.
2. The accepted problem was an active recommendation.
3. The accepted problem is in the curated sheet and has a next-in-topic candidate.
4. The accepted problem was solved confidently and similar reinforcement is available.
5. The user's difficulty readiness may have changed.

### 9.4 Idempotency

Refresh idempotency key:

```text
accepted_submission_refresh:{user_id}:{platform}:{submission_id}
```

---

## 10. Failed Submission Event Handling

When processing `failed_submission_processed`:

### 10.1 Required actions

```text
1. Read user_id, platform, problem_slug, submission_id, verdict from event metadata.
2. If the problem has an active recommendation, mark it started if not already started/clicked/completed.
3. Update evidence through later refresh, not by mutating old score history.
4. Decide whether to trigger event refresh.
5. Generate failed retry or weak topic recommendations when signal is strong.
```

### 10.2 Lifecycle update for active recommendation

If an active recommendation exists for the failed problem:

```text
pending/active/shown -> started
clicked -> started
started -> keep started
completed -> do not downgrade
skipped/dismissed/expired/replaced -> do not reopen automatically
```

Write:

```text
recommendation_started
```

with metadata:

```json
{
  "source": "live_failed_submission",
  "submission_id": "123456",
  "verdict": "WA"
}
```

### 10.3 Refresh trigger rules

Call `refresh_user_recommendations` with:

```text
trigger_source = failed_submission
processing_mode = event
```

when any of these is true:

1. Same problem has 2 or more failed attempts and no AC.
2. Same problem has 3 or more failed attempts within 7 days.
3. Topic failed attempts cross the weak-topic threshold.
4. Active recommendation count is below 5.
5. Active recommendation has failed attempts greater than 5 and near-solve score below 0.50, requiring easier replacement.

### 10.4 Failure guardrail

If the user repeatedly fails an active recommendation:

```text
If failed_attempt_count <= 5:
  keep recommendation active or started.

If failed_attempt_count > 5 and near_solve_score < 0.50:
  mark current row as replaced.
  write recommendation_replaced.
  trigger weak_topic_practice refresh.
```

---

## 11. Recommendation Interaction Event Handling

Stage 2 writes lifecycle events for user actions. Stage 3 uses those events to react.

### 11.1 Skip and dismiss handling

When processing `recommendation_skipped` or `recommendation_dismissed`:

1. Check active count.
2. If active count is below 5, trigger refresh with `trigger_source = recommendation_interaction`.
3. If the user dismissed 3 or more recommendations in one session, bias the next refresh toward safer and more diverse recommendations.
4. Store interaction summary in refresh metadata.

### 11.2 Not relevant handling

When processing `recommendation_marked_not_relevant`:

1. Treat the row as dismissed or replaced according to product behavior.
2. Apply cooldown.
3. Trigger refresh if active count falls below 5.
4. Include metadata that reduces similar patterns in the short term.

### 11.3 Already solved handling

When processing `recommendation_skipped_already_solved`:

1. Check `user_data.user_coding_problems` for AC.
2. If AC exists, mark recommendation as completed.
3. If AC does not exist, mark row replaced or dismissed depending on product behavior.
4. Avoid immediate duplicate recommendation.

---

## 12. Event-Triggered Refresh Rules

Stage 3 must call the Stage 2 refresh service instead of generating rows directly.

### 12.1 Accepted submission refresh

```python
await refresh_user_recommendations(
    user_id=user_id,
    platform=platform,
    trigger_source="accepted_submission",
    surface="dashboard",
    request_id=f"accepted:{submission_id}",
)
```

### 12.2 Failed submission refresh

```python
await refresh_user_recommendations(
    user_id=user_id,
    platform=platform,
    trigger_source="failed_submission",
    surface="dashboard",
    request_id=f"failed:{submission_id}",
)
```

### 12.3 Interaction refresh

```python
await refresh_user_recommendations(
    user_id=user_id,
    platform=platform,
    trigger_source="recommendation_interaction",
    surface="dashboard",
    request_id=f"interaction:{event.id}",
)
```

### 12.4 Refresh skip behavior

If active count is sufficient and no strong signal exists, write:

```text
recommendation_generation_skipped
```

Metadata:

```json
{
  "reason": "event_signal_not_strong_enough",
  "source_event_id": 123,
  "source_event_type": "failed_submission_processed",
  "active_count": 7
}
```

---

## 13. Event Idempotency and Ordering

### 13.1 Handler idempotency

Each event handler must be safe to run more than once.

Rules:

1. Lifecycle updates must use conditional status transitions.
2. Completion events must not duplicate if recommendation is already completed for same submission.
3. Refresh requests must use request-specific idempotency keys.
4. Cursor advancement must happen after successful processing.
5. Duplicate source events must be skipped through `idempotency_key`.

### 13.2 Ordering expectations

Events are processed in ascending `app.recommendation_events.id` order. This gives stable ordering inside the database event log.

Do not assume wall-clock timestamps are perfectly ordered across systems. Use `id` for projector ordering.

### 13.3 Race handling

If batch refresh and event refresh run at the same time for the same user:

1. Use Stage 2 user-platform refresh lock.
2. Let idempotency prevent duplicate refresh work.
3. Let unique key prevent duplicate recommendation rows.
4. Protect clicked/started rows from replacement.

---

## 14. Future Notification Escalation Boundary

The product requirement states that future versions may send email or WhatsApp notifications when a user has not seen recommendations for a while.

Stage 3 must prepare this boundary without implementing provider-specific sending.

### 14.1 Notification principle

Recommendation generation decides what to recommend. Notification delivery decides where and when to remind the user.

The notification system must not change:

```text
candidate generation
recommendation type
score
rank
guardrails
diversity
```

### 14.2 Notification eligibility signal

Add a service-level function:

```python
async def evaluate_notification_escalation(
    user_id: str,
    platform: str = "leetcode",
    surface: str = "notification",
    now: datetime | None = None,
) -> NotificationEscalationDecision:
    ...
```

This function reads active recommendations and events. It returns a decision only. It does not send messages in Stage 3.

### 14.3 Suggested decision model

```python
class NotificationEscalationDecision:
    user_id: str
    platform: str
    eligible: bool
    reason: str
    recommended_channel_order: list[str]
    recommendation_ids: list[int]
    quiet_hours_blocked: bool
    last_seen_at: datetime | None
    last_notification_event_at: datetime | None
    metadata: dict
```

### 14.4 Eligibility rules

A user is eligible for future notification escalation when:

1. There are active recommendations with `surface in ('dashboard', 'today', 'notification')`.
2. The recommendation has not been shown or clicked for the configured unseen window.
3. The row is not expired.
4. The row is not cooldown-blocked.
5. The user has not dismissed similar recommendations recently.
6. Quiet hours and user preferences allow notification.
7. The recommendation has safe deterministic `reason` text.

### 14.5 Unseen window

Use configuration, not hard-coded values:

```text
NOTIFICATION_UNSEEN_WINDOW_HOURS = 24
NOTIFICATION_MAX_PER_DAY = 1
NOTIFICATION_MIN_GAP_HOURS = 24
```

### 14.6 Provider interface for future email/WhatsApp

Define interfaces only:

```python
class NotificationProvider(Protocol):
    channel: str

    async def send(
        self,
        user_id: str,
        destination: str,
        message: NotificationMessage,
        idempotency_key: str,
    ) -> NotificationSendResult:
        ...
```

Future providers:

```text
EmailNotificationProvider
WhatsAppNotificationProvider
PushNotificationProvider
InAppNotificationProvider
```

Do not instantiate real provider clients in Stage 3 unless product explicitly enables them.

### 14.7 Notification message input

The message composer receives only safe recommendation fields:

```text
problem_title
problem_slug
topic_name
recommendation_type
reason
llm_reason if available
score evidence summary
surface
```

It must not receive:

```text
submitted code
expected output
actual output
last testcase
runtime error details
compile error details
private raw metadata
```

### 14.8 Future schema note

Current `app.recommendation_events.event_type` does not include notification-specific events. When actual email or WhatsApp sending is added, add a migration for event types such as:

```text
notification_escalation_evaluated
notification_send_requested
notification_sent
notification_failed
notification_suppressed
```

Until that migration exists, Stage 3 may log notification escalation decisions in application logs or existing internal event metadata only when it fits an allowed event type. Do not overload user-visible recommendation events with provider delivery status.

---

## 15. Modular Recommendation Model Changes

Future versions may change recommendation models. Stage 3 must preserve model modularity.

### 15.1 Versioned scorer

Continue storing:

```text
scoring_version
score_breakdown
evidence
```

Every recommendation must be explainable even if a future scoring model changes weights or adds signals.

### 15.2 Recommendation model interface

Keep the current deterministic model behind an interface:

```python
class RecommendationModel(Protocol):
    version: str

    async def generate_candidates(self, state: UserRecommendationState) -> list[RecommendationCandidate]:
        ...

    async def score_candidates(self, state: UserRecommendationState, candidates: list[RecommendationCandidate]) -> list[RecommendationCandidate]:
        ...

    async def select_final_set(self, state: UserRecommendationState, candidates: list[RecommendationCandidate]) -> list[RecommendationCandidate]:
        ...
```

Current implementation:

```text
DeterministicMvpRecommendationModel(version='mvp_v1')
```

Future model versions can be added without changing event processing or notification interfaces.

### 15.3 Feature flags

Use feature flags for model rollout:

```text
recommendation_model_version
live_event_refresh_enabled
failed_submission_retry_enabled
weak_topic_practice_enabled
notification_escalation_enabled
llm_reason_enabled
```

---

## 16. Production-Grade Streaming Rules

### 16.1 Locking

Use a projector-level lock so only one projector processes a cursor at a time:

```text
projector:live_recommendation_projector_v1
```

Use a user-platform refresh lock when calling the Stage 2 refresh service:

```text
recommendations:{user_id}:{platform}
```

### 16.2 Retry policy

Classify errors:

| Error type | Behavior |
|---|---|
| Database timeout | Retry, do not advance cursor |
| Temporary connection error | Retry, do not advance cursor |
| Missing optional metadata | Safe-skip or use fallback, advance cursor |
| Missing required user_id/platform | Non-retryable, write skip event, advance cursor |
| Invalid problem slug | Non-retryable if catalog absent, write skip event, advance cursor |
| Refresh service failure | Retry if transient, otherwise write `refresh_failed` |

### 16.3 Poison event handling

For non-retryable poison events:

1. Write `recommendation_generation_skipped` with `reason = poison_event`.
2. Include source event id and safe error message.
3. Advance cursor to avoid permanent blockage.
4. Emit alert if poison event count exceeds threshold.

### 16.4 Backpressure

Use configurable batch sizes:

```text
PROJECTOR_BATCH_SIZE = 100
PROJECTOR_MAX_RUNTIME_SECONDS = 30
PROJECTOR_IDLE_SLEEP_SECONDS = 5
```

If event volume grows, add queue-based infrastructure behind the same handler contracts.

### 16.5 Privacy

Do not include code or raw debug payloads in events, notification decisions, logs, or LLM prompts.

---

## 17. Event Processing Metrics

Track:

| Metric | Meaning |
|---|---|
| `projector_events_processed_total` | Count processed events |
| `projector_events_skipped_total` | Count safe-skipped events |
| `projector_failures_total` | Count projector failures |
| `projector_lag_events` | Latest event id minus cursor event id |
| `projector_lag_seconds` | Now minus latest unprocessed event time |
| `accepted_submission_refresh_total` | Refreshes triggered by accepted submissions |
| `failed_submission_refresh_total` | Refreshes triggered by failed submissions |
| `live_completion_total` | Active recommendations completed by live AC |
| `live_started_total` | Active recommendations started by live non-AC |
| `event_generation_skipped_total` | Event signals that did not require generation |
| `notification_escalation_eligible_total` | Future notification decisions marked eligible |

Log fields:

```text
projector_name
event_source
source_event_id
source_event_type
user_id hash/internal id
platform
problem_slug
handler_result
refresh_triggered
refresh_result
cursor_before
cursor_after
duration_ms
error_code
```

---

## 18. Stage 3 API and Worker Interfaces

### 18.1 Projector worker command

Create a worker entrypoint:

```text
recommendation-projector run --projector live_recommendation_projector_v1
```

Equivalent application function:

```python
async def run_recommendation_projector_once(batch_size: int = 100) -> ProjectorRunResult:
    ...
```

### 18.2 Live submission hook

Expose internal service function:

```python
async def on_live_submission_ingested(
    user_id: str,
    platform: str,
    submission_id: str,
    problem_slug: str,
    verdict: str,
    submitted_at: datetime,
    profile_id: str | None = None,
    metadata: dict | None = None,
) -> None:
    ...
```

This function writes events only. The projector processes them.

### 18.3 Notification escalation evaluation

Expose internal service function:

```python
async def evaluate_notification_escalation(...):
    ...
```

Do not send email or WhatsApp messages in this function during Stage 3.

---

## 19. Stage 3 Test Plan

### 19.1 Unit tests

| Module | Tests |
|---|---|
| `projector.py` | cursor read, event fetch, handler dispatch, cursor advance |
| `live_submission_handler.py` | AC handling, failed handling, metadata validation |
| `event_handlers.py` | skip, dismiss, not relevant, already solved, completed |
| `cursor_repository.py` | create cursor, advance cursor, update error metadata |
| `notification_escalation.py` | eligible, quiet hours blocked, recent dismiss blocked, no active recommendations |
| `model_registry.py` | selected model version, fallback behavior |

### 19.2 Scenario tests

#### Live accepted submission completes active recommendation

Input:

```text
active recommendation for two-sum
live AC submission for two-sum
```

Expected:

```text
recommendation status = completed
recommendation_completed event written
accepted_submission refresh triggered if eligible
cursor advanced
```

#### Live failed submission starts active recommendation

Input:

```text
shown recommendation for dp-problem
live WA submission for dp-problem
```

Expected:

```text
status moves to started
recommendation_started event written
failed_submission_processed event handled
cursor advanced
```

#### Failed retry strong signal

Input:

```text
3 failed submissions on unsolved curated problem in 7 days
```

Expected:

```text
failed_submission refresh triggered
failed_submission_retry candidate selected by Stage 1/2 flow
```

#### Too many failures replacement

Input:

```text
active recommendation
6 failed attempts
near_solve_score = 0.30
```

Expected:

```text
existing recommendation marked replaced
recommendation_replaced event written
weak_topic_practice refresh triggered
```

#### Duplicate live submission event

Input:

```text
same submission_id event processed twice
```

Expected:

```text
second processing skipped by idempotency
no duplicate lifecycle event
cursor advances safely
```

#### Projector retryable failure

Input:

```text
database timeout while processing event id 100
```

Expected:

```text
cursor does not advance past 99
retry can process event 100 later
```

#### Poison event

Input:

```text
event missing required user_id or platform
```

Expected:

```text
recommendation_generation_skipped written with poison_event reason
cursor advances
alert metric increments
```

#### Notification escalation decision

Input:

```text
active recommendation not shown for configured unseen window
no recent dismiss
quiet hours not active
```

Expected:

```text
NotificationEscalationDecision.eligible = true
recommended_channel_order contains future-safe channels
no email or WhatsApp message is sent
```

### 19.3 Integration tests

Run full hybrid flow:

```text
1. Apply Stage 1 migrations.
2. Seed curated sheet.
3. Run Stage 2 initial_backfill refresh.
4. Fetch dashboard recommendations.
5. Mark one recommendation shown.
6. Ingest live failed submission for the shown problem.
7. Write live events.
8. Run projector.
9. Assert recommendation is started.
10. Ingest live accepted submission for same problem.
11. Run projector.
12. Assert recommendation is completed.
13. Assert accepted refresh generated next-in-topic or similar recommendation.
14. Assert cursor advanced.
15. Assert no duplicate events after re-running projector.
```

---

## 20. Stage 3 Edge Cases

### 20.1 Live event arrives before aggregate recomputation

If aggregate state is not ready:

1. Write retryable failure metadata.
2. Do not advance cursor if recomputation is expected soon.
3. If aggregate is permanently missing, safe-skip with reason.

### 20.2 Accepted submission for problem with no active recommendation

Do not force completion update. Use it as a progress signal and trigger refresh only if next-in-topic, similar, or active-count rules pass.

### 20.3 Failed submission for non-curated problem

Store event, but do not create direct retry recommendation. It may still contribute to topic weakness only if mapped safely to curated topic metadata.

### 20.4 User completes recommendation outside UI

Live accepted submission should mark matching active recommendation completed even if the user did not click Attempt.

### 20.5 User skips then solves same problem

If sync confirms AC after skip:

1. Mark completed if product wants solved truth to override skip.
2. Preserve skip event history.
3. Store metadata showing completion source.

### 20.6 Event refresh creates no candidates

Write `recommendation_generation_skipped` with reason:

```text
no_valid_candidates_after_guardrails
```

### 20.7 Projector downtime

When projector resumes, it must process events after the stored cursor. Batch refresh remains the fallback if live processing is delayed.

### 20.8 Notification unseen window overlaps with user activity

If the user is active but has not seen the recommendation, notification escalation must respect product preferences and quiet hours. Do not send future email/WhatsApp based only on active recommendations; require unseen-window and preference checks.

### 20.9 Future model version changes

Old recommendations keep their original `scoring_version` and `score_breakdown`. New event refreshes may use the active configured model version. Do not rewrite old score explanations.

---

## 21. Stage 3 Acceptance Criteria

Stage 3 is complete when:

1. Live submission ingestion writes recommendation-processing events.
2. Accepted submissions write `accepted_submission_processed` with idempotency.
3. Failed submissions write `failed_submission_processed` with idempotency.
4. Projector reads events using `app.projector_cursors`.
5. Projector advances cursor only after successful or safe-skip processing.
6. Live accepted submissions complete matching active recommendations.
7. Live failed submissions mark matching active recommendations as started.
8. Strong failed signals can trigger failed-retry or weak-topic refresh through Stage 2.
9. Accepted signals can trigger next-in-topic, similar, or next-difficulty refresh through Stage 2.
10. Skip/dismiss/not-relevant events can trigger refill when active count falls below 5.
11. Duplicate events do not create duplicate lifecycle updates or recommendations.
12. Retryable errors do not advance cursor incorrectly.
13. Poison events are safely skipped with explicit internal metadata.
14. Event processing metrics and logs are available.
15. Notification escalation decision boundary exists but does not send email or WhatsApp messages.
16. Future provider interfaces exist without affecting recommendation ranking.
17. Tests pass for event handlers, cursor progress, idempotency, and live lifecycle updates.
18. Batch processing remains available as fallback.
19. Candidate generation, scoring, and diversity remain centralized in Stage 1 and Stage 2 services.

---

## 22. Final System After Stage 3

After Stage 3, the recommendation system is complete as a hybrid MVP:

```text
Stage 1:
  Tables, curated sheet, user state, type decision, candidate generation, scoring helpers, guardrails.

Stage 2:
  Historical and scheduled batch generation, active recommendation persistence, serving APIs, lifecycle APIs, batch events.

Stage 3:
  Live submission events, projector processing, real-time lifecycle updates, event-triggered refresh, notification-readiness boundary.
```

The system remains modular:

1. Recommendation model changes are isolated behind scoring and model-version interfaces.
2. Notification channels are isolated behind future provider interfaces.
3. LLM usage remains limited to explanation text and notification copy.
4. The backend recommendation processor remains the only authority for candidate selection, ranking, cooldown, and active recommendation state.
