# Event-Sourced Recommendation Architecture Reference

## Document Purpose

This document merges two design concerns into one implementation reference:

1. The recommendation strategy described in the provided recommendation reference design.
2. The event-sourcing, CQRS, projection, and materialized-view architecture that should be used to implement that strategy in a durable, inspectable, and extensible way.

This report is intended for a codebase-aware LLM or engineering agent that already has access to the project repository, database models, historical user submissions, question bank, and current service boundaries.

The goal is **not** to prescribe exact database schemas or force a specific table layout. The goal is to describe the architecture, responsibilities, event flow, and implementation constraints so the codebase-aware LLM can design the final implementation using existing project data structures wherever possible.

---

## Primary Instruction for the Codebase-Aware LLM

Before creating any new table, worker, queue, service, endpoint, or model, inspect the existing project for reusable concepts.

The implementation agent must cross-check:

- Existing submission records.
- Existing question bank or problem catalog.
- Existing topic, tag, concept, or difficulty metadata.
- Existing recommendation routes or services.
- Existing user progress or mastery records.
- Existing notification, alert, or activity-feed structures.
- Existing job queue, worker, scheduler, or background-task infrastructure.
- Existing event, audit, logging, telemetry, or sync tables.
- Existing Supabase/Postgres/SQLite models and migrations.
- Existing user preference, device-token, or notification settings.
- Existing read models that can be extended rather than duplicated.

If existing structures already provide the required information, extend or adapt them instead of creating parallel tables with overlapping responsibility.

This document intentionally describes conceptual components such as "event log", "projection", "recommendation read model", and "notification read model". The implementation agent should map those concepts to the existing project architecture.

---

## Why Use Event Sourcing, CQRS, and Projections for Recommendations?

The recommendation system depends on historical user behavior. A user's current recommendation state is not only a static profile; it is the accumulated result of many events:

- User attempted a question.
- User solved a question.
- User failed a question.
- User took too long on a question.
- User repeatedly failed a concept.
- User improved on a topic.
- User dismissed a recommendation.
- User accepted a recommendation.
- User ignored a recommendation.
- User changed target company or learning path.
- User synced external coding profile data.
- System generated a recommendation.
- System delivered a notification.

A standard CRUD-only model stores the latest state, but it often loses the reason why that state exists. For recommendations, the "why" is important. We need to know not just that a user is weak in graphs, but how that weakness emerged and how it changed over time.

Event sourcing solves this by storing important domain facts as immutable events. Projection tables then convert those events into fast queryable state.

The architecture should follow this principle:

```text
Events are the source of truth.
Projection tables are derived views.
Recommendation and notification services read projections and append new events.
```

---

## Architecture Summary

The recommended architecture is:

```text
User action or backend process
        ↓
Command handler / API service
        ↓
Append domain event to event log
        ↓
Queue or event dispatcher wakes workers
        ↓
Projection workers update query-optimized read models
        ↓
Recommendation worker reads projection state
        ↓
Recommendation worker appends RecommendationCreated event
        ↓
Recommendation projection updates recommendations view
        ↓
Notification worker creates NotificationCreated event
        ↓
Notification projection updates notification view
        ↓
Frontend reads notification and recommendation read models
```

This is a selective event-sourcing design. It does not require event sourcing the entire product. Apply it first to recommendation-relevant user learning events, recommendation decisions, and notification lifecycle events.

---

## Conceptual Components

### 1. Event Log

The event log is the append-only record of domain facts. It should capture the business meaning of what happened.

Examples:

```text
QuestionSubmitted
QuestionSolved
QuestionFailed
QuestionSkipped
QuestionRecommended
RecommendationCreated
RecommendationAccepted
RecommendationDismissed
NotificationCreated
NotificationRead
NotificationDismissed
ProfileSynced
TargetCompanyChanged
DailyRecommendationTick
```

Events should represent intent and domain facts, not low-level database diffs.

Prefer:

```text
QuestionSolved
```

over:

```text
user_topic_state.updated
```

because `QuestionSolved` explains what actually happened.

The event log should be immutable. If something needs to be corrected, append a compensating event such as:

```text
QuestionSubmissionCorrected
RecommendationInvalidated
NotificationCancelled
```

### 2. Command Side

The command side handles writes and business actions.

Examples:

- Submit answer.
- Mark recommendation accepted.
- Dismiss recommendation.
- Mark notification read.
- Sync external profile.
- Change learning target.

The command side should validate the action and append the correct event. It should not directly update every derived recommendation table.

### 3. Query Side

The query side reads from projection tables or materialized views.

Examples:

- Current mastery by topic.
- Current weak topics.
- Current candidate recommendation state.
- Current active recommendations.
- Current unread notifications.
- Recent recommendation history.

The query side should avoid replaying raw events on every user-facing request. Event replay should be reserved for rebuilding projections, debugging, testing, and migration.

### 4. Projection Workers

Projection workers read events and update read models.

Examples:

- `user_learning_state_projector`
- `topic_mastery_projector`
- `concept_gap_projector`
- `recommendation_history_projector`
- `notification_projector`
- `analytics_projector`

Each projector should have a checkpoint or equivalent mechanism to remember which events it has processed.

Projection workers must be idempotent. If the same event is processed twice, the derived state should remain correct.

### 5. Recommendation Worker

The recommendation worker reads user learning projections and the existing question bank. It applies the recommendation strategy from the reference design.

It should append a `RecommendationCreated` event when it selects a new question.

It should not directly become the source of truth for recommendation history without writing an event first.

### 6. Notification Worker

The notification worker responds to recommendation or schedule events and creates notification events.

Example:

```text
RecommendationCreated
        ↓
NotificationCreated
```

The notification projection then updates the user-facing notification view.

---

## How the Recommendation Reference Design Maps to Event Sourcing

The uploaded reference design defines an adaptive recommendation strategy based on user submissions, weak topics, strong topics, recent behavior, accuracy trends, difficulty suitability, previous mistakes, concept gaps, and repetition control.

In the event-sourced architecture, those signals come from the event log and its projections.

| Recommendation concept | Event-sourced implementation mapping |
|---|---|
| Previous submissions | Derived from `QuestionSubmitted`, `QuestionSolved`, `QuestionFailed`, and imported profile events |
| Weak topics | Projection over submission events into topic mastery state |
| Strong topics | Projection over long-term and recent successful events |
| Recent behavior | Projection that weights recent events more heavily |
| Accuracy trends | Projection over correct/incorrect events by topic and time window |
| Difficulty suitability | Projection over outcomes grouped by difficulty |
| Previously wrong questions | Projection over failed attempts and retry timing |
| Concept-level gaps | Projection over concept/tag-specific failure patterns |
| Repetition avoidance | Projection over recommendation history and recent attempts |
| Recommendation explanation | Stored as payload/metadata on `RecommendationCreated` |
| Evaluation metrics | Analytics projections over recommendation and outcome events |

---

## Recommendation Strategy Preserved from the Reference Design

The system should remain a performance-driven adaptive recommender.

At a high level:

```text
Read projected user learning state
        ↓
Estimate topic and concept mastery
        ↓
Identify weak, improving, and strong topics
        ↓
Select a practice topic using priority and controlled randomness
        ↓
Select suitable difficulty
        ↓
Collect candidate questions from the existing question bank
        ↓
Score candidates
        ↓
Emit RecommendationCreated event
        ↓
Project recommendation into user-facing read model
        ↓
Optionally create notification
```

The recommender should use the existing question bank as the candidate pool. It should not invent new questions unless the product already supports generated questions.

The codebase-aware LLM must inspect the current question model and adapt the candidate selection logic to available metadata such as topic, difficulty, tags, concept labels, activity status, source metadata, platform metadata, acceptance level, or problem type.

---

## User Understanding Model as a Projection

The reference design says the system should maintain or derive a user understanding profile from past submissions. In this architecture, that profile should be treated as a projection.

The projection may include:

- Accuracy by topic.
- Recent accuracy by topic.
- Number of attempts by topic.
- Correct and incorrect attempts.
- Difficulty performance.
- Time taken compared with expected time.
- Recently failed topics.
- Repeated concept mistakes.
- Last practiced time.
- Improvement or decline trend.
- Recently recommended questions.
- Recently dismissed recommendations.

This profile should be derived from events, not manually treated as the source of truth.

If the projection is corrupted or the scoring logic changes, the system should be able to rebuild the profile from the event log.

---

## Mastery Score as a Derived Value

The reference design recommends mastery between `0` and `1`.

A basic formula is:

```text
mastery = correct_attempts / total_attempts
```

A stronger version combines multiple signals:

```text
mastery =
  0.60 * recent_accuracy
+ 0.25 * overall_accuracy
+ 0.10 * speed_score
+ 0.05 * difficulty_performance_score
```

In the event-sourced architecture, these values should be computed by projection logic.

The implementation should not assume a single permanent formula. The projection should be versioned or designed so that mastery logic can be recalculated later.

Recommended approach:

- Store raw events permanently.
- Store current projected mastery for fast reads.
- Store projection version or calculation metadata where useful.
- Allow projection rebuild when scoring logic changes.

---

## Topic Classification and Priority as Projections

The reference design classifies topics as:

```text
0.00 to 0.40 = weak
0.41 to 0.70 = improving
0.71 to 1.00 = strong
```

The system may use these thresholds initially, but the codebase-aware LLM should tune them based on actual available data and product behavior.

Topic priority can use:

```text
topic_priority =
  0.50 * weakness_score
+ 0.30 * recent_error_rate
+ 0.10 * recency_gap_score
+ 0.10 * topic_importance_score
```

MVP formula:

```text
topic_priority =
  0.70 * (1 - mastery_score)
+ 0.30 * recent_error_rate
```

In the architecture, topic priority can be calculated:

1. During projection updates, if the value is needed frequently.
2. On recommendation generation, if the calculation is cheap.
3. As a hybrid: project base metrics, compute final score at recommendation time.

Prefer projecting stable facts and computing volatile strategy scores inside the recommendation service unless the project already has a suitable read model.

---

## Candidate Question Selection

The reference design says the recommender should gather candidate questions from the existing question pool after selecting a topic and target difficulty.

Candidate filters include:

- Same selected topic.
- Suitable difficulty.
- Active and valid question.
- Not recommended too recently.
- Not already mastered.
- Related to weak concepts.
- Previously wrong but due for retry.

In this architecture, the candidate selector reads:

```text
Existing question bank
+ user topic projection
+ user concept projection
+ previous attempt projection
+ recommendation history projection
```

The output is not immediately inserted as final state. The recommendation decision should first become an event:

```text
RecommendationCreated
```

Then the recommendation projection updates the user-facing recommendation read model.

---

## Question Scoring Logic

The reference design recommends:

```text
question_score =
  0.35 * difficulty_match_score
+ 0.25 * new_or_due_score
+ 0.20 * previous_mistake_score
+ 0.15 * concept_gap_score
+ 0.05 * freshness_score
```

MVP formula:

```text
question_score =
  0.40 * difficulty_match_score
+ 0.30 * unattempted_or_due_score
+ 0.20 * concept_gap_score
+ 0.10 * freshness_score
```

The score and reason should be stored in the recommendation event payload or metadata so later analysis can explain why the recommendation was made.

Example conceptual payload:

```json
{
  "problem_id": "existing-problem-id",
  "problem_slug": "course-schedule",
  "selected_topic": "graph",
  "score": 0.86,
  "scoring_version": "mvp-v1",
  "reason_codes": [
    "weak_topic",
    "difficulty_match",
    "concept_gap",
    "not_recently_recommended"
  ],
  "explanation": "Graph is a weak topic and this question matches the user's current difficulty range."
}
```

Do not hardcode this payload shape blindly. Reuse existing identifiers and metadata from the project.

---

## Handling Previous Submissions

After every submission, the system should append an event and let projections update the user's learning state.

Reference design behavior:

```text
Correct easy question   -> small mastery increase
Correct medium question -> moderate mastery increase
Correct hard question   -> larger mastery increase
Wrong easy question     -> larger mastery decrease
Wrong medium question   -> moderate mastery decrease
Wrong hard question     -> small mastery decrease
```

Example deltas:

```text
Correct easy   -> +0.03
Correct medium -> +0.05
Correct hard   -> +0.08
Wrong easy     -> -0.08
Wrong medium   -> -0.05
Wrong hard     -> -0.03
```

In the event-sourced architecture, do not overwrite history when mastery changes. Store the submission outcome event, then update the mastery projection.

If the mastery formula changes later, replay historical submission events into a new or rebuilt projection.

---

## Full Example: User Solves a Question and Gets a New Recommendation

### Scenario

A user submits and solves a question:

```text
Question: Number of Islands
Difficulty: Medium
Topics: graph, dfs, bfs
Verdict: correct
```

### Step 1: Submission Command

The application receives a submission command.

Conceptually:

```text
SubmitQuestionAnswer(user, question, verdict, time_taken)
```

The command handler validates that:

- The user exists.
- The question exists in the existing question bank.
- Topic/difficulty metadata can be resolved from existing data.
- The submission is not a duplicate.
- Any existing submission storage is updated only if the project already requires it.

Then it appends a domain event:

```text
QuestionSolved
```

### Step 2: Event Is Stored

The event log receives an immutable event describing the business fact.

Conceptual event content:

```json
{
  "event_type": "QuestionSolved",
  "subject_user_id": "user-id",
  "payload": {
    "question_id": "existing-question-id",
    "question_slug": "number-of-islands",
    "difficulty": "medium",
    "topics": ["graph", "dfs", "bfs"],
    "time_taken_seconds": 1200,
    "result": "correct"
  }
}
```

This event is now the durable source of truth.

### Step 3: Projection Workers Update Learning State

The topic mastery projector consumes `QuestionSolved`.

It updates the user's projected state:

```text
graph solved count increases
dfs solved count increases
bfs solved count increases
recent accuracy for these topics updates
last practiced timestamp updates
mastery score is recalculated
weak/improving/strong classification may change
```

The concept projection may also update concept-specific state if concept metadata exists.

The recommendation history projection may record that the user has recently interacted with this question.

### Step 4: Recommendation Worker Runs

The recommendation worker is triggered because `QuestionSolved` is an allowed recommendation trigger.

It reads:

```text
User topic mastery projection
User concept gap projection
Recommendation history projection
Existing question bank
Existing attempt/submission projection
```

It identifies that graph-related topics are still weak or improving. It uses the reference design's priority and scoring logic to select a next question.

Example selected question:

```text
Course Schedule
Topics: graph, dfs, topological-sort
Difficulty: medium
```

The recommendation worker creates a `RecommendationCreated` event.

### Step 5: Recommendation Event Is Stored

Conceptual event:

```json
{
  "event_type": "RecommendationCreated",
  "subject_user_id": "user-id",
  "payload": {
    "recommended_question_id": "existing-question-id",
    "recommended_question_slug": "course-schedule",
    "selected_topic": "graph",
    "score": 0.86,
    "based_on_event_type": "QuestionSolved",
    "reason": "The user solved a graph question but graph mastery is still below target. This medium graph question is an appropriate next step.",
    "scoring_version": "mvp-v1"
  }
}
```

The event explains why the recommendation exists.

### Step 6: Recommendation Projection Updates User-Facing State

The recommendation projection consumes `RecommendationCreated`.

It updates or creates the user-facing recommendation view.

The frontend or API can now read active recommendations without replaying the full event history.

### Step 7: Notification Worker Creates a Notification Event

If the product wants to notify the user, the notification worker consumes `RecommendationCreated`.

It may check:

- User notification preferences.
- Quiet hours.
- Whether similar notification was sent recently.
- Whether the recommendation should appear immediately or later.
- Whether the user is currently active in the app.
- Whether in-app notification, push notification, email, or all are enabled.

Then it appends:

```text
NotificationCreated
```

Conceptual content:

```json
{
  "event_type": "NotificationCreated",
  "subject_user_id": "user-id",
  "payload": {
    "kind": "recommendation",
    "title": "Try Course Schedule next",
    "body": "You solved Number of Islands. Course Schedule is a good next graph problem.",
    "target": "/practice/course-schedule",
    "scheduled_at": "now"
  }
}
```

### Step 8: Notification Projection Updates Notification View

The notification projection consumes `NotificationCreated` and updates the user-facing notification read model.

The frontend notification panel reads this projection.

### Step 9: User Interacts with Notification

If the user reads or dismisses the notification, do not simply mutate state without history. Append:

```text
NotificationRead
```

or

```text
NotificationDismissed
```

Then update the notification projection.

### Step 10: Recommendation Feedback Loop

If the user accepts the recommendation and attempts the question, append:

```text
RecommendationAccepted
QuestionSubmitted
QuestionSolved or QuestionFailed
```

These events feed back into learning projections and improve future recommendations.

---

## Preventing Infinite Loops

The architecture creates events in response to events. This is powerful but dangerous if uncontrolled.

The recommender must only be triggered by explicit allowed events.

Recommended allowed triggers:

```text
QuestionSolved
QuestionFailed
QuestionSubmitted
ProfileSynced
TargetCompanyChanged
DailyRecommendationTick
ManualRecommendationRequest
```

Events that should not trigger recommendation generation:

```text
RecommendationCreated
NotificationCreated
NotificationRead
NotificationDismissed
RecommendationShown
RecommendationImpressionLogged
```

If `RecommendationCreated` triggers the recommender, the system can loop forever.

The implementation agent should define trigger allowlists per worker.

---

## Scheduled Recommendations and Notifications

The system may support time-based events.

Examples:

```text
DailyRecommendationTick
SpacedReviewDue
ContestReminderDue
InactiveUserNudgeDue
```

There are two recommended approaches:

### MVP Approach: Scheduled Rows or Jobs

Use a scheduled timestamp in an existing job, queue, or notification structure.

A scheduler worker periodically finds due work and emits the appropriate event.

Example:

```text
scheduled_at <= now
        ↓
emit DailyRecommendationTick or NotificationDue
```

### Managed Scheduler Approach

If the infrastructure already uses a managed scheduler, use it to enqueue a job or call an endpoint at a specific time.

The codebase-aware LLM should inspect existing infrastructure before adding a new scheduler.

---

## Queue and Worker Guidance

The architecture needs asynchronous workers so the user request does not perform every projection, recommendation, and notification update synchronously.

Recommended responsibilities:

```text
event_dispatch_worker
topic_projection_worker
concept_projection_worker
recommendation_worker
recommendation_projection_worker
notification_worker
notification_projection_worker
analytics_projection_worker
```

These can be separate services or separate job types in one worker process. Start simple; split later when operational load requires it.

If the project already has a queue or worker system, reuse it.

If not, suitable options include:

- Postgres-backed job queue.
- Supabase Queues or pgmq.
- Graphile Worker.
- pg-boss.
- Redis/BullMQ.
- AWS SQS plus EventBridge Scheduler.

For a Postgres-first project, prefer a Postgres-backed queue for the MVP unless there is already a different queue in production.

---

## Consistency Model

This architecture is eventually consistent.

Example:

```text
User solves question at 10:00:00
QuestionSolved event stored at 10:00:01
Projection updates at 10:00:02
Recommendation created at 10:00:04
Notification appears at 10:00:05
```

This is acceptable for recommendations and notifications.

Do not use eventual projections for flows that require immediate strict consistency unless the command handler also performs the required synchronous update.

---

## Idempotency Requirements

Every event handler and projection must be idempotent.

The same event may be delivered twice because queues and workers commonly use at-least-once processing.

Use one or more of:

- Event IDs.
- Idempotency keys.
- Projection checkpoints.
- Unique constraints on derived records.
- Processed-event tracking per projection.
- Upsert logic.
- Source event ID references.

If an event is replayed, it should not duplicate recommendations, notifications, or counters.

---

## Event Versioning and Schema Evolution

Events live longer than application code. The implementation should plan for event versioning from the start.

Guidelines:

- Include event type and event version.
- Prefer additive changes to event payloads.
- Make consumers tolerant of unknown fields.
- Use default values for missing fields.
- Consider upcasters for older event versions.
- Avoid storing unnecessary personal data in immutable events.
- Store references to user or question records instead of large sensitive payloads where possible.

---

## Privacy and Data Retention

Because event logs are append-only, avoid storing unnecessary personal data directly inside events.

Prefer:

```text
user_id
question_id
submission_id
recommendation_id
```

over copying full user profiles, emails, names, or large raw payloads.

If privacy deletion is required, design a strategy such as:

- Store personal data outside the event log and reference it by ID.
- Delete/anonymize external personal data while keeping event structure.
- Use crypto-shredding only if the project already has key-management maturity.
- Use retention policies for nonessential analytics events if allowed by product requirements.

---

## Materialized Views and Rebuild Strategy

Projection tables are disposable read models.

The implementation should support rebuilding projections from the event log.

Recommended rebuild abilities:

```text
rebuild user learning state
rebuild topic mastery
rebuild concept gaps
rebuild recommendation history
rebuild notification state
rebuild analytics metrics
```

Rebuilds are useful when:

- A projection bug is fixed.
- A scoring formula changes.
- A new projection is introduced.
- Historical analytics needs to be recalculated.
- Data migration is required.

The implementation agent should avoid making projection tables the only place where important historical facts exist.

---

## API Guidance

### Write APIs

Write APIs should append events.

Examples:

```text
POST /submissions
POST /recommendations/{id}/accept
POST /recommendations/{id}/dismiss
POST /notifications/{id}/read
POST /profile/sync
```

These should produce domain events.

### Read APIs

Read APIs should query projections.

Examples:

```text
GET /recommendations/current
GET /notifications
GET /notifications/unread-count
GET /users/me/learning-state
GET /users/me/topic-mastery
```

These should not replay raw events on every request.

---

## Observability and Debugging

Every recommendation should be explainable.

Recommendation events should include enough metadata to answer:

- Which user state triggered the recommendation?
- Which topic was selected?
- Which candidate questions were considered?
- Which scoring version was used?
- Why was this question selected?
- Was the recommendation shown, accepted, dismissed, ignored, or solved?
- Did the recommendation improve future performance?

The architecture should support internal debugging views such as:

```text
User event timeline
Projection state snapshot
Recommendation decision trace
Notification delivery trace
```

These can be built later as read models over the event log.

---

## Evaluation Metrics

The recommendation reference design suggests tracking:

- Accuracy improvement over time.
- Mastery improvement by topic.
- Reduction in repeated mistakes.
- Completion rate of recommended questions.
- Skip or abandonment rate.
- Average time spent per question.
- Percentage of recommendations answered correctly.
- User retention after recommendations.
- Difficulty progression over time.
- Diversity of topics recommended.

In the event-sourced architecture, these should be analytics projections over:

```text
QuestionSolved
QuestionFailed
RecommendationCreated
RecommendationShown
RecommendationAccepted
RecommendationDismissed
NotificationCreated
NotificationRead
```

The product should not optimize only for immediate correctness. If recommendations are always too easy, correctness may be high but learning may be low.

---

## MVP Implementation Plan

### Phase 1: Audit Existing Structures

The codebase-aware LLM should inspect and document:

```text
existing submission storage
existing question bank
existing topic and concept metadata
existing user progress records
existing recommendation code
existing notification code
existing queue/worker/scheduler infrastructure
existing database technology and migration system
```

Only after that audit should it propose additions.

### Phase 2: Introduce Event Logging Around Recommendation-Relevant Actions

Start by emitting events for:

```text
QuestionSubmitted
QuestionSolved
QuestionFailed
RecommendationCreated
RecommendationAccepted
RecommendationDismissed
NotificationCreated
NotificationRead
NotificationDismissed
```

### Phase 3: Build Core Projections

Build projections for:

```text
user topic mastery
recent topic performance
recommendation history
active recommendations
notifications
```

### Phase 4: Implement MVP Recommendation Worker

Use the reference design's MVP logic:

```text
topic_priority =
  0.70 * (1 - mastery_score)
+ 0.30 * recent_error_rate
```

```text
question_score =
  0.40 * difficulty_match_score
+ 0.30 * unattempted_or_due_score
+ 0.20 * concept_gap_score
+ 0.10 * freshness_score
```

The worker reads projections and question bank, then emits `RecommendationCreated`.

### Phase 5: Implement Notification Flow

Consume `RecommendationCreated` and create notification events/read models.

Support in-app notifications first. Add push/email only after in-app flow is stable.

### Phase 6: Add Rebuild and Testing

Add tests for:

```text
given events
when command happens
then new events are emitted
```

Add projection tests for:

```text
given events
when replayed
then projection state matches expected state
```

Add idempotency tests for duplicate event delivery.

---

## Implementation Boundaries

Recommended module boundaries:

```text
event writer
event reader / dispatcher
projection checkpoint manager
submission event mapper
user performance projection
mastery calculation
topic prioritization
difficulty selection
candidate filtering
question scoring
recommendation explanation
recommendation event producer
notification event producer
notification projection
analytics projection
projection rebuild runner
```

These modules can initially live in the same backend service. The design should allow them to be split later.

---

## Anti-Patterns to Avoid

Avoid:

1. Creating new tables without checking existing project data first.
2. Treating projection tables as the source of truth.
3. Replaying all events on every API read.
4. Emitting vague events like `TableUpdated`.
5. Storing sensitive user data directly in immutable events.
6. Letting `RecommendationCreated` trigger another recommendation.
7. Creating notifications directly without a durable event or record.
8. Ignoring idempotency.
9. Ignoring event versioning.
10. Building a complex ML system before the rule-based adaptive loop works.
11. Recommending questions not present in the existing question bank.
12. Losing the explanation for why a recommendation was made.

---

## Final Guidance for the Implementation LLM

The implementation LLM should treat this document as a reference architecture and decision guide.

It should not blindly create the conceptual structures named here. Instead, it should:

1. Inspect the codebase.
2. Identify current equivalents for event logs, submissions, question bank, recommendations, notifications, and queues.
3. Reuse existing structures when possible.
4. Add missing components only where necessary.
5. Preserve the adaptive recommendation logic from the reference design.
6. Use event sourcing selectively for recommendation-relevant workflows.
7. Use CQRS so writes append events and reads query projections.
8. Ensure projections can be rebuilt from events.
9. Ensure recommendation and notification events are explainable.
10. Keep the MVP rule-based, testable, and modular.

The final implementation should allow the product to answer:

```text
Why was this question recommended?
Which events caused this recommendation?
What user weakness or pattern did it target?
Was the recommendation useful?
Did the user act on it?
Did the user's mastery improve afterward?
```

That is the main reason to use this architecture.

---

# Appendix A: Recommendation Reference Design Content

The following content was provided as the recommendation strategy reference and should remain the basis for the recommendation algorithm. The event-sourcing architecture above describes how to store, derive, trigger, and observe the data needed to implement this strategy.


# Recommendation Reference Design

## Purpose

This document defines a reference design for a question recommendation system. It is intended to guide another LLM or engineering agent that has access to the actual codebase, data models, and historical user submissions.

The goal is not to prescribe exact database structures or force a specific implementation. Instead, this document describes the recommendation strategy, the reasoning model, the scoring approach, and the decision-making logic that can be adapted to the existing codebase.

The recommendation system should answer one core question:

> Based on the user's previous submissions, what question should be recommended next so that the user improves efficiently, especially in weaker topics?

---

## Core Recommendation Objective

The system should recommend the next question by considering:

1. The user's weak topics.
2. The user's strong topics.
3. Recent submission behavior.
4. Accuracy trends.
5. Difficulty suitability.
6. Previously attempted and incorrectly answered questions.
7. Topic and concept-level gaps.
8. Avoidance of excessive repetition.

The recommendation should not simply choose a random question from a weak topic. It should select a question that is useful, appropriately difficult, and aligned with the user's current learning state.

---

## High-Level Strategy

The recommended strategy is a performance-driven adaptive recommender.

At a high level:

1. Read the user's previous submissions.
2. Estimate the user's strength or mastery for each topic.
3. Identify weak and improving areas.
4. Select the topic that deserves the highest practice priority.
5. Select the difficulty level that best matches the user's current ability.
6. Score candidate questions from the selected topic.
7. Recommend the highest-value question.
8. After the user submits the answer, update the user's topic-level and concept-level understanding.

This approach can be implemented without requiring a complex machine learning model at the start. It can later be extended into a more advanced adaptive learning or ML-based recommendation system.


---

## Existing Question Bank Assumption

This design assumes that the product already has a pre-made question bank available in the database or codebase. The recommendation system should use that existing question source rather than generating new questions by default.

The question source may already organize questions by topic. For example, a topic such as `array` may contain multiple associated questions, and each question may include metadata such as difficulty level, tags, title, source information, platform-style attributes, and other LeetCode-like properties.

The recommendation system should treat this question bank as the candidate pool. Its job is to decide which existing question should be selected next for a user based on the user's previous submissions, weak topics, recent mistakes, and required practice areas.

The codebase-aware LLM should inspect the existing question data structure and adapt the recommendation logic to it. It should not assume that the question bank has a specific schema. It should only require that each candidate question can be interpreted using available metadata such as:

- Topic or topic grouping.
- Difficulty level.
- Concept tags or sub-topic labels, if available.
- Previous user attempt history, if available.
- Whether the question is active or eligible for recommendation.
- Any available source-style metadata, such as LeetCode-style tags, constraints, acceptance level, or problem type.

The recommender should therefore work as a selection layer over the existing question bank. It should not define the question bank itself.

---

## How the Recommender Uses the Pre-Made Question Bank

The recommendation flow should be:

```text
Read user's previous submissions
        ↓
Detect weak and strong topics
        ↓
Select the topic that needs practice
        ↓
Look inside the existing question bank for questions from that topic
        ↓
Filter by suitable difficulty and eligibility
        ↓
Score candidate questions using user-specific signals
        ↓
Recommend the best-fit existing question
```

Example behavior:

```text
User is weak in arrays
        ↓
System selects array as the target topic
        ↓
System reads existing array questions from the question bank
        ↓
System prefers easy or medium array questions depending on current mastery
        ↓
System prioritizes questions with tags similar to the user's recent mistakes
        ↓
System recommends one existing question from the array topic
```

If the question bank contains multiple questions under the same topic, the recommender should not select randomly. It should score those questions based on fit for the user.

Important selection signals include:

- Whether the question belongs to the selected weak topic.
- Whether the difficulty level matches the user's current mastery.
- Whether the user has attempted the question before.
- Whether the user previously answered the question or similar tagged questions incorrectly.
- Whether the question targets a concept that the user needs to practice.
- Whether the question has been recommended recently.
- Whether the question is suitable for the user's current progression level.

The recommender should output a question that already exists in the product's question data. It should not invent a new question unless the product explicitly supports generated questions.
---

## User Understanding Model

The recommendation system should maintain or derive a user understanding profile from past submissions.

The profile should represent how well the user is performing across topics and, if available, concepts within topics.

Useful signals include:

- Accuracy by topic.
- Recent accuracy by topic.
- Number of attempts by topic.
- Number of correct and incorrect attempts.
- Difficulty level of attempted questions.
- Time taken compared with expected time.
- Recently failed topics.
- Repeated mistakes in the same concept.
- Last practiced time for each topic.
- Whether the user has improved or declined recently.

The system should prioritize recent behavior more strongly than old behavior because recent submissions better represent the user's current state.

---

## Mastery Score

Each user-topic pair should have a mastery estimate between `0` and `1`.

A value near `0` means the user is weak in that topic.
A value near `1` means the user is strong in that topic.

A basic mastery score can be calculated from accuracy:

```text
mastery = correct_attempts / total_attempts
```

However, a stronger design should combine multiple signals:

```text
mastery =
  recent_accuracy_weight * recent_accuracy
+ overall_accuracy_weight * overall_accuracy
+ speed_weight * speed_score
+ difficulty_weight * difficulty_performance_score
```

Suggested weighting for an initial version:

```text
mastery =
  0.60 * recent_accuracy
+ 0.25 * overall_accuracy
+ 0.10 * speed_score
+ 0.05 * difficulty_performance_score
```

The exact weights can be adjusted based on available data and product goals.

---

## Why Recent Accuracy Should Matter More

A user may have performed poorly in the past but improved recently. Similarly, a user may have been strong earlier but started making mistakes recently.

For this reason, the system should give more importance to recent attempts.

Possible recent windows:

```text
last 5 attempts in the topic
last 10 attempts in the topic
last 7 days of topic activity
last 30 days of topic activity
```

The best window depends on the available submission volume. If users attempt many questions daily, a smaller attempt-based window may work well. If usage is sparse, a longer time-based window may be better.

---

## Topic Classification

Topics can be classified based on mastery score.

Suggested classification:

```text
0.00 to 0.40 = weak
0.41 to 0.70 = improving
0.71 to 1.00 = strong
```

These thresholds are not fixed. They should be tuned based on actual user behavior and content difficulty.

The recommender should mostly focus on weak and improving topics, while occasionally revisiting strong topics for retention and confidence.

A practical distribution:

```text
70% recommendations from weak topics
20% recommendations from improving topics
10% recommendations from strong topics for revision
```

This prevents the system from becoming too frustrating by only showing weak areas, while still prioritizing improvement.

---

## Topic Priority Logic

After calculating mastery for each topic, the system should compute a priority score for each topic.

The priority score decides which topic should be practiced next.

Recommended factors:

1. Low mastery.
2. High recent error rate.
3. Topic not practiced recently.
4. Topic importance, if the product has curriculum priority.
5. Repeated mistakes in the topic.
6. User's current learning path or assigned module, if applicable.

Suggested formula:

```text
topic_priority =
  0.50 * weakness_score
+ 0.30 * recent_error_rate
+ 0.10 * recency_gap_score
+ 0.10 * topic_importance_score
```

Where:

```text
weakness_score = 1 - mastery_score
```

`recent_error_rate` should represent how often the user recently answered questions incorrectly in that topic.

`recency_gap_score` should increase when the user has not practiced the topic for a while.

`topic_importance_score` should represent product or curriculum importance, if available.

If curriculum importance is unavailable, its weight can be redistributed to weakness and recent error rate.

Alternative simple formula for MVP:

```text
topic_priority =
  0.70 * (1 - mastery_score)
+ 0.30 * recent_error_rate
```

This is suitable for an initial implementation.

---

## Topic Selection Strategy

The system should generally select the topic with the highest topic priority score.

However, always selecting the highest-priority topic may create a repetitive experience. To avoid this, the algorithm can use controlled randomness.

Recommended approach:

1. Rank topics by priority score.
2. Select from the top few topics instead of always selecting only the top one.
3. Use weighted random selection, where higher-priority topics have a higher chance of being selected.

Example:

```text
Topic A priority = 0.80
Topic B priority = 0.68
Topic C priority = 0.55
```

Instead of always choosing Topic A, the system can usually choose Topic A but sometimes choose Topic B or C.

This makes recommendations feel less repetitive while still remaining targeted.

---

## Difficulty Selection Logic

After selecting a topic, the system should choose a suitable difficulty level.

The difficulty should not be too easy or too hard. It should be slightly challenging but solvable.

Suggested mapping:

```text
mastery < 0.35       -> easy
0.35 <= mastery < 0.60 -> easy to medium
0.60 <= mastery < 0.80 -> medium to hard
mastery >= 0.80      -> hard or revision challenge
```

For weak topics, the system should avoid jumping directly to hard questions unless the user has shown evidence of handling harder questions in that topic.

A useful learning principle is:

```text
Weak topic + low mastery -> rebuild foundation
Improving topic -> gradually increase challenge
Strong topic -> use harder questions or spaced revision
```

---

## Candidate Question Selection

Once a topic and target difficulty are selected, the system should gather candidate questions from the available question pool.

Candidate questions should preferably satisfy:

1. Same selected topic.
2. Suitable difficulty level.
3. Active and valid question.
4. Not recently recommended to the same user.
5. Not already mastered by the user.
6. Related to concepts where the user has struggled.

The system should not automatically exclude previously attempted questions. Recommending a previously wrong question can be valuable, especially after some time has passed.

However, it should avoid recommending the same question too soon after the user already attempted it.

---

## Question Scoring Logic

Each candidate question should receive a recommendation score.

Suggested scoring formula:

```text
question_score =
  0.35 * difficulty_match_score
+ 0.25 * new_or_due_score
+ 0.20 * previous_mistake_score
+ 0.15 * concept_gap_score
+ 0.05 * freshness_score
```

The candidate with the highest score should usually be recommended.

---

## Meaning of Question Scoring Factors

### Difficulty Match Score

This measures whether the question difficulty matches the user's current ability in the selected topic.

A good question should be close to the user's current level, with a slight challenge.

Example logic:

```text
perfect difficulty match -> high score
slightly easier or harder -> medium score
much too easy or too hard -> low score
```

---

### New or Due Score

This gives value to questions the user has not attempted before, or questions that are due for review.

Recommended behavior:

```text
never attempted -> high score
attempted long ago and relevant again -> medium to high score
attempted recently -> low score
```

This prevents repetition while still allowing useful review.

---

### Previous Mistake Score

If the user previously answered a question incorrectly, or repeatedly failed similar questions, the system should treat that as a useful learning opportunity.

A previously wrong question should not be repeated immediately. It should become eligible again after a suitable delay or after the user has practiced related easier concepts.

Recommended behavior:

```text
wrong before and enough time has passed -> high score
wrong very recently -> low or medium score
correct before with confidence -> low score
```

---

### Concept Gap Score

If questions have concept tags or sub-topic labels, the system should identify concepts where the user is weak.

For example, within a broad topic, the user may be weak in a specific concept even if the topic-level mastery is moderate.

The recommender should prefer questions that target these concept gaps.

Example:

```text
Topic: Geometry
Weak concept: triangles
Preferred recommendation: Geometry question tagged with triangles
```

Concept-level scoring is especially useful when topics are broad.

---

### Freshness Score

Freshness prevents repeated or stale recommendations.

A question should receive a lower freshness score if:

- It was recommended recently.
- It was attempted recently.
- The user has already solved it correctly multiple times.

A question should receive a higher freshness score if:

- It has never been recommended.
- It has not appeared for a long time.
- It fits the current weakness profile.

---

## Handling Previous Submissions

After every submission, the system should update the user's learning state.

The update should consider:

1. Whether the answer was correct.
2. The difficulty of the question.
3. Time taken.
4. Whether the user has made similar mistakes before.
5. Whether this was improvement compared with previous attempts.

Suggested mastery update behavior:

```text
Correct easy question   -> small mastery increase
Correct medium question -> moderate mastery increase
Correct hard question   -> larger mastery increase
Wrong easy question     -> larger mastery decrease
Wrong medium question   -> moderate mastery decrease
Wrong hard question     -> small mastery decrease
```

Reasoning:

- Solving a hard question is strong evidence of mastery.
- Failing an easy question is strong evidence of weakness.
- Failing a hard question does not necessarily mean the user is weak.
- Solving an easy question does not necessarily mean the user is strong.

Example update scale:

```text
Correct easy   -> +0.03
Correct medium -> +0.05
Correct hard   -> +0.08
Wrong easy     -> -0.08
Wrong medium   -> -0.05
Wrong hard     -> -0.03
```

The final mastery score should always remain between `0` and `1`.

---

## Time-Based Performance Signal

Time taken should be used carefully.

A correct answer with very high time may still indicate partial weakness.
An incorrect answer with very short time may indicate guessing or lack of effort.
A correct answer within expected time is a stronger mastery signal.

Suggested speed score:

```text
completed within expected time -> high speed score
slightly over expected time -> medium speed score
far over expected time -> low speed score
```

Time should not dominate correctness. It should be a secondary signal.

---

## Avoiding Bad Recommendations

The recommender should avoid:

1. Recommending questions that are much too hard for weak users.
2. Recommending only weak-topic questions continuously.
3. Recommending the same question repeatedly.
4. Recommending questions already solved correctly many times.
5. Ignoring recent incorrect submissions.
6. Ignoring difficulty level.
7. Treating old performance as equally important as recent performance.
8. Selecting random questions without user-specific reasoning.

The recommendation should always be explainable from the user's performance history.

---

## Cold Start Behavior

If the user has little or no submission history, the system should not assume strengths and weaknesses.

Possible cold start strategies:

1. Recommend diagnostic questions from multiple topics.
2. Start with easy or medium questions.
3. Use curriculum order, if available.
4. Use popular or representative questions.
5. Gradually build the user's topic mastery profile.

For new users, the system should collect enough signal before strongly targeting weak topics.

Suggested initial flow:

```text
first few recommendations -> diagnostic spread across topics
then -> estimate topic mastery
then -> move into adaptive weak-topic recommendations
```

---

## Balancing Exploration and Exploitation

The system should balance two goals:

### Exploitation

Recommend questions in known weak areas to improve the user efficiently.

### Exploration

Occasionally recommend questions from less-tested topics to discover hidden weaknesses or strengths.

Suggested balance:

```text
80% exploit known weak or improving areas
20% explore less-tested or scheduled revision areas
```

This prevents the system from becoming too narrow.

---

## Recommended MVP Logic

For an initial implementation, the following logic is sufficient:

1. Calculate mastery per topic from recent and overall submissions.
2. Calculate recent error rate per topic.
3. Compute topic priority.
4. Select the highest-priority weak or improving topic.
5. Choose difficulty based on mastery.
6. Score available questions in that topic and difficulty range.
7. Prefer questions that are unattempted, previously wrong after a delay, or mapped to weak concepts.
8. Recommend the highest-scoring question.
9. After submission, update the user's mastery state.

MVP topic priority:

```text
topic_priority =
  0.70 * (1 - mastery_score)
+ 0.30 * recent_error_rate
```

MVP question score:

```text
question_score =
  0.40 * difficulty_match_score
+ 0.30 * unattempted_or_due_score
+ 0.20 * concept_gap_score
+ 0.10 * freshness_score
```

This is simple, explainable, and practical.

---

## Advanced Recommendation Improvements

After the MVP is stable, the system can be improved using more advanced approaches.

Possible future improvements:

1. Concept-level mastery instead of only topic-level mastery.
2. Spaced repetition for previously wrong questions.
3. Adaptive difficulty progression.
4. Bayesian Knowledge Tracing.
5. Item Response Theory.
6. Learning path sequencing.
7. Multi-armed bandit selection.
8. Embedding-based similarity between questions.
9. Personalized revision scheduling.
10. Reinforcement learning based on long-term learning outcomes.

These should be considered only after the basic recommendation loop produces reliable results.

---

## Explanation Requirement

Every recommendation should be explainable internally.

The system should be able to produce a reason such as:

```text
This question was recommended because the user recently struggled with this topic, the question difficulty matches the user's current mastery level, and the concept has appeared in recent incorrect submissions.
```

The explanation may or may not be shown to the end user, but it should be available for debugging and evaluation.

---

## Evaluation Metrics

The effectiveness of the recommender should be measured using product and learning metrics.

Useful metrics:

1. Accuracy improvement over time.
2. Mastery improvement by topic.
3. Reduction in repeated mistakes.
4. Completion rate of recommended questions.
5. Skip rate or abandonment rate.
6. Average time spent per question.
7. Percentage of recommendations answered correctly.
8. User retention after recommendations.
9. Difficulty progression over time.
10. Diversity of topics recommended.

The system should not optimize only for immediate correctness. If questions are always too easy, correctness may be high but learning may be low.

A good recommender should improve learning outcomes while keeping the user engaged.

---

## Recommended Decision Flow

```text
User submits answer
        ↓
Read result, topic, difficulty, time, and concept metadata
        ↓
Update user-topic and user-concept mastery estimates
        ↓
Calculate topic priority scores
        ↓
Select target topic using priority and controlled randomness
        ↓
Select suitable difficulty based on mastery
        ↓
Collect candidate questions from selected topic and difficulty range
        ↓
Score candidate questions
        ↓
Apply repetition and freshness filters
        ↓
Recommend best-fit question
        ↓
Store recommendation event for future analysis
```

---

## Reference Pseudocode

```python
def recommend_next_question(user, available_questions, submission_history):
    user_profile = build_or_update_user_profile(user, submission_history)

    topic_scores = []

    for topic in user_profile.topics:
        mastery = topic.mastery_score
        recent_error_rate = topic.recent_error_rate
        recency_gap = topic.recency_gap_score
        importance = topic.importance_score

        priority = (
            0.50 * (1 - mastery)
            + 0.30 * recent_error_rate
            + 0.10 * recency_gap
            + 0.10 * importance
        )

        topic_scores.append({
            "topic": topic,
            "priority": priority,
        })

    selected_topic = select_topic(topic_scores)

    target_difficulty_range = choose_difficulty_range(
        selected_topic.mastery_score
    )

    candidates = filter_questions(
        available_questions=available_questions,
        topic=selected_topic,
        difficulty_range=target_difficulty_range,
        user_profile=user_profile,
    )

    scored_candidates = []

    for question in candidates:
        score = (
            0.35 * difficulty_match_score(question, selected_topic.mastery_score)
            + 0.25 * new_or_due_score(question, user_profile)
            + 0.20 * previous_mistake_score(question, user_profile)
            + 0.15 * concept_gap_score(question, user_profile)
            + 0.05 * freshness_score(question, user_profile)
        )

        scored_candidates.append({
            "question": question,
            "score": score,
        })

    selected_question = select_highest_scoring_question(scored_candidates)

    return selected_question
```

---

## Reference Mastery Update Pseudocode

```python
def update_mastery(current_mastery, is_correct, difficulty, time_signal=None):
    if is_correct:
        if difficulty == "easy":
            delta = 0.03
        elif difficulty == "medium":
            delta = 0.05
        else:
            delta = 0.08
    else:
        if difficulty == "easy":
            delta = -0.08
        elif difficulty == "medium":
            delta = -0.05
        else:
            delta = -0.03

    if time_signal == "very_slow_correct":
        delta *= 0.75
    elif time_signal == "fast_correct":
        delta *= 1.10

    updated_mastery = current_mastery + delta

    return max(0, min(1, updated_mastery))
```

---

## Implementation Notes for the Codebase-Aware LLM

The codebase-aware LLM should adapt this design to the existing code and data structures.

It should inspect the existing models, services, submission records, question metadata, and current recommendation flow before implementing changes.

It should avoid introducing unnecessary new data structures if the required information already exists.

It should prefer an explainable recommendation layer first, then optimize after observing real usage.

The implementation should be modular so that topic scoring, difficulty selection, question scoring, and mastery updates can be tested independently.

Recommended module boundaries:

```text
user performance extraction
mastery calculation
topic prioritization
difficulty selection
candidate filtering
question scoring
recommendation explanation
post-submission update
```

---

## Final Recommendation

Use a rule-based adaptive recommendation system as the first version.

The system should prioritize weak topics, recent mistakes, suitable difficulty, and concept-level gaps. It should avoid repetitive recommendations and gradually adapt after every submission.

This strategy is simple enough to implement quickly, transparent enough to debug, and flexible enough to evolve into a more advanced recommendation engine later.
