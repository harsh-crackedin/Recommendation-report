# Hybrid Recommendation Model Design Report

## 1. Purpose

We are building a hybrid recommendation model for coding practice. The model recommends LeetCode problems to a user based on their submission history, failed attempts, solved history, recommendation interactions, and a curated internal DSA sheet.

The selected recommendation model uses:

```text
Curated DSA Sheet + Historical Batch Processing + Live Event Processing + Active Recommendation Serving
```

The model has four primary goals:

1. Recommend useful coding problems from a controlled internal DSA pool.
2. Use accepted submissions to understand progress and topic strength.
3. Use failed submissions to detect weak topics, retry opportunities, and practice gaps.
4. Keep a small ready-to-serve recommendation set available for frontend surfaces.

The model is designed for an MVP, so we keep the system simple, explainable, deterministic, and easy to debug.

---

## 2. Selected Model Summary

We use a hybrid recommendation model with two processing modes:

| Processing mode | Purpose | Input data | Output |
|---|---|---|---|
| Historical batch processing | Build the first recommendation set after a user connects their LeetCode account or extension | Historical accepted submissions, failed submissions, attempted problems, curated DSA sheet | Initial problem recommendations |
| Live event processing | Update recommendation state after new activity | New accepted submissions, new failed submissions, recommendation events | Refreshed or updated recommendations |

The frontend does not calculate recommendations directly. The frontend reads recommendations from:

```text
app.user_problem_recommendations
```

Recommendation lifecycle and internal processing activity are stored in:

```text
app.recommendation_events
```

The curated recommendation pool is stored in:

```text
catalog.dsa_sheet_problems
```

Event projector progress is tracked in:

```text
app.projector_cursors
```

---

## 3. Core Design Principles

### 3.1 Controlled recommendation pool

We recommend only from the curated DSA sheet table:

```text
catalog.dsa_sheet_problems
```

This keeps recommendations predictable and aligned with our learning path. Each curated problem points to the shared LeetCode problem catalog using:

```text
platform + problem_slug
```

The curated sheet controls:

- Which problems are eligible for recommendation.
- Topic grouping.
- Topic order.
- Problem order inside a topic.
- Problem importance.
- Revision value.
- Source sheet or source list.
- Active or inactive status.

### 3.2 Every submission attempt is useful

We store and analyze all submission attempts, including:

- Accepted
- Wrong Answer
- Time Limit Exceeded
- Runtime Error
- Compilation Error
- Other non-accepted verdicts

Accepted submissions show confirmed progress. Failed submissions show struggle, weak topics, retry opportunities, and abandoned problems.

### 3.3 Recommendations are precomputed

We generate recommendations in the backend and store them in:

```text
app.user_problem_recommendations
```

The frontend reads from this table instead of computing recommendations during request time.

### 3.4 Recommendation events are append-only

We record recommendation lifecycle events and internal processing events in:

```text
app.recommendation_events
```

We append new events instead of overwriting old events. This gives us an auditable history of recommendation behavior.

### 3.5 LLM is used only for explanation text

The recommendation model decides which problem to recommend. The LLM can generate a friendly explanation or notification message, but it does not decide ranking, filtering, or candidate selection.

---

## 4. High-Level Architecture

```text
LeetCode Extension / Sync Source
        |
        v
Submission Storage
        |
        v
Historical Batch Processor  <---->  Live Event Processor
        |
        v
Curated Candidate Pool: catalog.dsa_sheet_problems
        |
        v
Candidate Generation
        |
        v
Scoring + Guardrails + Diversity Rules
        |
        v
Stored Recommendations: app.user_problem_recommendations
        |
        v
Frontend Surfaces / Notifications / Chat
        |
        v
Recommendation Events: app.recommendation_events
```

---

## 5. Main Tables in the Selected Design

| Table | Role |
|---|---|
| `catalog.dsa_sheet_problems` | Controlled pool of DSA problems eligible for recommendation |
| `app.user_problem_recommendations` | Stored recommendation rows for each user and problem |
| `app.recommendation_events` | Append-only event log for user-visible and internal recommendation activity |
| `app.projector_cursors` | Tracks event processing progress for safe event-based processing |

---

## 6. Data Sources Used by the Recommendation Model

### 6.1 Curated DSA sheet

The curated sheet contains the selected problem pool. Each row represents one problem in a sheet, topic, and order.

We use it to answer:

- Is this problem eligible for recommendation?
- Which topic does this problem belong to?
- What is the next problem in this topic?
- Is this a foundational problem?
- Is this a high-priority problem?
- Is this a good revision problem?
- Is this problem currently active?

### 6.2 Existing problem catalog

The shared problem catalog remains the source of truth for full problem details.

We use it for:

- Problem slug
- Problem title
- Difficulty
- Platform metadata
- Problem URL or external metadata

### 6.3 User submission history

We use submission history to calculate:

- Solved problems
- Attempted problems
- Failed problems
- Accepted count per problem
- Failed count per problem
- Total attempt count per problem
- Topic-level failed attempt patterns
- Recent activity
- Stale topics
- User difficulty level

### 6.4 Recommendation interaction history

We use recommendation events to calculate:

- Recently shown recommendations
- Clicked recommendations
- Started recommendations
- Skipped recommendations
- Dismissed recommendations
- Completed recommendations
- Recently expired recommendations
- Problems that should be cooled down

---

## 7. Recommendation Lifecycle

Each recommendation row moves through a lifecycle.

| Status | Meaning |
|---|---|
| `pending` | Recommendation has been generated but not shown yet |
| `active` | Recommendation is available for serving |
| `shown` | User has seen the recommendation |
| `clicked` | User clicked or opened the recommendation |
| `started` | User started attempting the problem |
| `skipped` | User skipped the recommendation |
| `dismissed` | User dismissed or rejected the recommendation |
| `completed` | User completed or solved the recommendation |
| `expired` | Recommendation is no longer fresh |
| `replaced` | Recommendation was replaced by a better or safer recommendation |

Recommended frontend serving statuses:

```text
pending, active, shown
```

The frontend should not serve rows that are completed, dismissed, expired, or replaced.

---

## 8. Recommendation Types

We support the following recommendation types in the selected model.

| Recommendation type | Purpose | Main signal |
|---|---|---|
| `standard_practice` | Recommend an important unsolved curated problem | Curated importance and interview value |
| `weak_topic_practice` | Strengthen a weak topic | Topic-level failed attempts and weak solve history |
| `failed_submission_retry` | Ask the user to retry a problem they failed earlier | Recent or repeated failed attempts |
| `revision` | Ask the user to revise a previously solved problem | Solved long ago or important for retention |
| `next_in_topic` | Continue progress inside a curated topic order | Previous problem solved, next problem unsolved |
| `similar_problem` | Reinforce a recently solved or failed pattern | Same topic, related pattern, or nearby concept |
| `next_difficulty` | Move the user to the next suitable difficulty level | Current solve strength and difficulty readiness |
| `daily_practice` | Build a small balanced daily practice set | Mix of weakness, revision, progression, and standard practice |

---

## 9. Historical Backfill Flow

Historical backfill runs when a user connects their extension or syncs older LeetCode activity for the first time.

```text
User connects extension
        |
        v
We import historical submissions
        |
        v
We store accepted and failed attempts
        |
        v
We analyze solved history, failed patterns, and topic coverage
        |
        v
We compare user history with catalog.dsa_sheet_problems
        |
        v
We generate initial user_problem_recommendations
        |
        v
We record internal recommendation_events
```

Historical backfill uses:

- Accepted submissions to identify solved topics and confirmed progress.
- Failed submissions to identify repeated struggles and weak areas.
- Curated sheet order to recommend next problems.
- Recommendation event history to avoid immediately repeating skipped or dismissed recommendations.

---

## 10. Live Submission Sync Flow

Live sync runs after a new LeetCode submission is detected.

```text
User submits code on LeetCode
        |
        v
Extension detects result
        |
        v
We store submission attempt
        |
        v
We append recommendation processing event
        |
        v
We update recommendation state if needed
        |
        v
We generate or replace recommendations when useful
```

Accepted submissions can trigger:

- Marking a recommended problem as completed.
- Generating the next problem in the same topic.
- Generating similar reinforcement problems.
- Updating user strength and difficulty readiness.

Failed submissions can trigger:

- Failed submission retry recommendation.
- Weak topic practice recommendation.
- Easier related problem recommendation.
- Replacement of overly difficult active recommendations.

---

## 11. Active Recommendation Count Rules

We keep the active recommendation set small.

| Rule | Value |
|---|---:|
| Maximum active recommendations per user | 10 |
| Minimum active recommendation threshold | 5 |
| Refresh trigger | Active count falls below 5 |
| Frontend read target | Top ranked active rows |

Active recommendation count includes rows with statuses:

```text
pending, active, shown
```

Rows with the following statuses do not count as active:

```text
completed, skipped, dismissed, expired, replaced
```

---

## 12. Candidate Generation

Candidate generation creates possible recommendation rows before scoring.

We generate candidates from the curated DSA sheet and user history.

### 12.1 Standard practice candidates

We select unsolved curated problems with high importance or interview frequency.

### 12.2 Weak topic candidates

We select problems from topics where the user has repeated failures, low solve rate, or stale progress.

### 12.3 Failed submission retry candidates

We select problems that the user attempted but did not solve, especially if failures were recent or repeated.

### 12.4 Revision candidates

We select previously solved problems that are important and have not been practiced recently.

### 12.5 Next in topic candidates

We select the next unsolved problem in the curated order after recently solved topic problems.

### 12.6 Similar problem candidates

We select related problems that reinforce a recently solved or recently failed concept.

### 12.7 Next difficulty candidates

We select problems one difficulty step above the user's current comfort zone when recent solve history supports readiness.

### 12.8 Daily practice candidates

We build a balanced set using a mix of standard practice, weak topic practice, revision, retry, and progression.

---

## 13. Scoring Model

We use a deterministic scoring model for MVP.

```text
final_score =
  0.16 * importance_score
+ 0.14 * interview_frequency_score
+ 0.18 * topic_weakness_score
+ 0.14 * difficulty_match_score
+ 0.16 * failed_attempt_score
+ 0.10 * revision_due_score
+ 0.08 * progression_score
+ 0.04 * freshness_score
- penalty_score
```

### 13.1 Score weight table

| Score component | Weight | Purpose |
|---|---:|---|
| `importance_score` | 0.16 | Prioritizes canonical and high-value curated problems |
| `interview_frequency_score` | 0.14 | Prioritizes commonly useful interview problems |
| `topic_weakness_score` | 0.18 | Prioritizes topics where the user is weak |
| `difficulty_match_score` | 0.14 | Keeps recommendations suitable for current user level |
| `failed_attempt_score` | 0.16 | Prioritizes retry and recovery from failed attempts |
| `revision_due_score` | 0.10 | Prioritizes useful spaced revision |
| `progression_score` | 0.08 | Prioritizes next-in-topic and path continuity |
| `freshness_score` | 0.04 | Avoids stale and repetitive recommendations |
| `penalty_score` | variable | Reduces unsafe, repetitive, or low-quality recommendations |

### 13.2 Score breakdown storage

Each recommendation stores the full scoring explanation in:

```text
app.user_problem_recommendations.score_breakdown
```

Example:

```json
{
  "scoring_version": "mvp_v1",
  "weights": {
    "importance_score": 0.16,
    "interview_frequency_score": 0.14,
    "topic_weakness_score": 0.18,
    "difficulty_match_score": 0.14,
    "failed_attempt_score": 0.16,
    "revision_due_score": 0.10,
    "progression_score": 0.08,
    "freshness_score": 0.04
  },
  "raw_scores": {
    "importance_score": 0.80,
    "interview_frequency_score": 0.70,
    "topic_weakness_score": 0.90,
    "difficulty_match_score": 0.80,
    "failed_attempt_score": 1.00,
    "revision_due_score": 0.00,
    "progression_score": 0.60,
    "freshness_score": 0.50
  },
  "penalties": {
    "recently_shown_penalty": 0.00,
    "recently_dismissed_penalty": 0.00,
    "same_topic_overexposure_penalty": 0.10,
    "too_hard_penalty": 0.00,
    "already_solved_penalty": 0.00
  },
  "final_score": 0.7540
}
```

---

## 14. Score Component Definitions

### 14.1 Importance score

We store this on the curated DSA sheet.

| Value | Meaning |
|---:|---|
| 1.00 | Must-do canonical problem |
| 0.80 | Strong interview problem |
| 0.60 | Useful practice problem |
| 0.40 | Nice-to-have problem |
| 0.20 | Low-priority problem |

### 14.2 Interview frequency score

We store this on the curated DSA sheet.

| Value | Meaning |
|---:|---|
| 1.00 | Appears across many curated lists or has high interview value |
| 0.75 | Common problem |
| 0.50 | Moderate value problem |
| 0.25 | Low observed frequency |
| 0.00 | Unknown value |

### 14.3 Topic weakness score

We calculate this from user submission history.

```text
topic_weakness_score =
  min(1.0,
    0.45 * topic_failed_ratio
  + 0.25 * repeated_failure_score
  + 0.20 * unsolved_attempted_topic_score
  + 0.10 * stale_topic_score
  )
```

| Signal | Meaning |
|---|---|
| `topic_failed_ratio` | Failed attempts in topic divided by total attempts in topic |
| `repeated_failure_score` | Higher when the user fails multiple problems in the same topic |
| `unsolved_attempted_topic_score` | Higher when the user attempts problems in a topic but does not solve them |
| `stale_topic_score` | Higher when the user has not solved a topic recently |

### 14.4 Difficulty match score

We calculate this from solved count, recent failures, and problem difficulty.

| User profile | Easy | Medium | Hard |
|---|---:|---:|---:|
| Solved fewer than 20 problems | 1.00 | 0.50 | 0.00 |
| Solved 20 to 80 problems | 0.50 | 1.00 | 0.25 |
| Solved more than 80 problems | 0.25 | 0.85 | 0.75 |

### 14.5 Failed attempt score

We calculate this from problem-level failed attempts.

```text
failed_attempt_score =
  min(1.0,
    0.20 * failed_attempt_count_score
  + 0.20 * recent_failure_boost
  + 0.20 * close_to_solve_boost
  + 0.20 * unsolved_boost
  + 0.20 * topic_relevance_boost
  )
```

| Failed attempt count | Score |
|---:|---:|
| 1 failed attempt | 0.25 |
| 2 failed attempts | 0.50 |
| 3 failed attempts | 0.75 |
| 4 or more failed attempts | 1.00 |

| Last failed attempt | Recent failure boost |
|---|---:|
| Last 1 day | 1.00 |
| Last 7 days | 0.75 |
| Last 30 days | 0.40 |
| Older | 0.10 |

### 14.6 Revision due score

We calculate this for solved problems.

| Last solved time | Score |
|---|---:|
| 90+ days ago | 1.00 |
| 45 to 90 days ago | 0.75 |
| 21 to 45 days ago | 0.50 |
| 7 to 21 days ago | 0.25 |
| Less than 7 days ago | 0.00 |

### 14.7 Progression score

We calculate this from curated order and recent progress.

| Condition | Score |
|---|---:|
| Exact next problem in curated topic order | 1.00 |
| Nearby next problem in same topic | 0.75 |
| Next useful difficulty step | 0.60 |
| Useful but not progression-related | 0.30 |
| No progression relationship | 0.00 |

### 14.8 Freshness score

We calculate this from recommendation history.

| Condition | Score |
|---|---:|
| Never shown before | 1.00 |
| Not shown in last 30 days | 0.75 |
| Not shown in last 14 days | 0.50 |
| Shown recently | 0.25 |
| Already active or shown very recently | 0.00 |

---

## 15. Recommendation-Type Boosts

After base scoring, we apply type-specific boosts and penalties.

| Recommendation type | Boost rule |
|---|---|
| `standard_practice` | Boost high importance and high interview frequency problems |
| `weak_topic_practice` | Boost topics with high weakness score and repeated failures |
| `failed_submission_retry` | Boost recent, repeated, unsolved failed attempts |
| `revision` | Boost important problems solved long ago |
| `next_in_topic` | Boost the next unsolved problem after a recently solved topic problem |
| `similar_problem` | Boost problems related to a recent solve or failure |
| `next_difficulty` | Boost one-step difficulty progression when readiness is clear |
| `daily_practice` | Boost recommendations that improve daily set balance |

### 15.1 Selected boost examples

| Type | Condition | Score change |
|---|---|---:|
| `standard_practice` | `importance_score >= 0.80` | +0.10 |
| `standard_practice` | `interview_frequency_score >= 0.70` | +0.05 |
| `weak_topic_practice` | `topic_weakness_score >= 0.75` | +0.15 |
| `weak_topic_practice` | User failed 3+ problems in the topic | +0.10 |
| `failed_submission_retry` | Failed attempts >= 3 | +0.20 |
| `failed_submission_retry` | Last failed attempt within 7 days | +0.10 |
| `revision` | Solved 45+ days ago | +0.15 |
| `revision` | High revision value score | +0.10 |
| `next_in_topic` | Previous problem order in same topic was solved | +0.15 |
| `similar_problem` | Similar to a problem solved in last 7 days | +0.15 |
| `next_difficulty` | User solved 3+ recent problems at current difficulty | +0.15 |
| `daily_practice` | Fills missing daily mix slot | +0.10 |

---

## 16. Hard Guardrails

We apply hard guardrails before final ranking.

| Guardrail | Rule |
|---|---|
| Active sheet only | We recommend only problems where `catalog.dsa_sheet_problems.is_active = true` |
| Curated pool only | We recommend only problems present in the curated DSA sheet |
| No duplicate active recommendation | We do not create another active row for the same user, problem, and recommendation type |
| Solved problem rule | We do not recommend solved problems unless type is `revision` |
| Recently dismissed rule | We do not recommend a dismissed problem before `cooldown_until` |
| Expired row rule | We do not serve rows after `expires_at` |
| Active count rule | We keep a maximum of 10 active recommendations per user |
| Refresh threshold rule | We refresh when active recommendations fall below 5 |
| Beginner safety rule | We avoid hard problems for users with very low solved count |
| Missing catalog rule | We do not recommend a curated problem if the referenced catalog problem is missing |

---

## 17. Diversity Rules

We apply diversity rules after scoring.

| Rule | Limit |
|---|---:|
| Same topic in top 5 | Maximum 2 |
| Same topic in total active set | Maximum 3 |
| Failed submission retry recommendations active at once | Maximum 2 |
| Revision recommendations active at once | Maximum 2 |
| Same difficulty in top 5 | Avoid all 5 being the same difficulty |
| Daily practice set | Prefer a mix of weakness, retry, revision, and progression |

Diversity keeps the recommendation set useful and prevents the user from seeing only one topic or one type of recommendation.

---

## 18. Cooldown and Expiry Rules

| Recommendation type | Expiry | Cooldown after dismissal |
|---|---|---|
| `standard_practice` | 7 days | 21 days |
| `weak_topic_practice` | 7 days | 14 days |
| `failed_submission_retry` | 3 days | 7 days |
| `revision` | 14 days | 30 days |
| `next_in_topic` | 7 days | 14 days |
| `similar_problem` | 3 days | 7 days |
| `next_difficulty` | 7 days | 14 days |
| `daily_practice` | 24 hours | 3 days |

---

## 19. Event Model

We use `app.recommendation_events` as the append-only log for both user-visible lifecycle events and internal processing events.

### 19.1 User-visible events

| Event | Meaning |
|---|---|
| `recommendation_created` | We created a recommendation row |
| `recommendation_shown` | The recommendation was shown to the user |
| `recommendation_clicked` | The user clicked the recommendation |
| `recommendation_started` | The user started attempting the recommendation |
| `recommendation_skipped` | The user skipped the recommendation |
| `recommendation_dismissed` | The user dismissed or rejected the recommendation |
| `recommendation_completed` | The user completed the recommended problem |
| `recommendation_expired` | The recommendation expired |
| `recommendation_replaced` | We replaced the recommendation |
| `recommendation_marked_not_relevant` | The user marked the recommendation as not relevant |
| `recommendation_skipped_already_solved` | The user indicated the problem was already solved |

### 19.2 Internal events

| Event | Meaning |
|---|---|
| `refresh_requested` | We requested a recommendation refresh |
| `refresh_started` | We started a recommendation refresh |
| `refresh_completed` | We completed a recommendation refresh |
| `refresh_failed` | Recommendation refresh failed |
| `historical_submission_processed` | We processed historical submission data |
| `live_submission_processed` | We processed a live submission |
| `accepted_submission_processed` | We processed an accepted submission signal |
| `failed_submission_processed` | We processed a failed submission signal |
| `batch_recommendations_generated` | We generated recommendations from batch processing |
| `event_recommendations_generated` | We generated recommendations from event processing |
| `recommendation_generation_skipped` | We skipped generation because guardrails blocked it |

---

## 20. Internal Event Examples

### 20.1 Refresh requested

```json
{
  "event_type": "refresh_requested",
  "visibility": "internal",
  "metadata": {
    "trigger_source": "below_active_threshold",
    "active_count": 4,
    "idempotency_key": "below_threshold:user_id:2026-06-04"
  }
}
```

### 20.2 Refresh completed

```json
{
  "event_type": "refresh_completed",
  "visibility": "internal",
  "metadata": {
    "processing_mode": "batch",
    "scoring_version": "mvp_v1",
    "candidate_count": 58,
    "selected_count": 8,
    "inserted_count": 6,
    "replaced_count": 2,
    "duration_ms": 420
  }
}
```

### 20.3 Recommendation generation skipped

```json
{
  "event_type": "recommendation_generation_skipped",
  "visibility": "internal",
  "metadata": {
    "reason": "active_count_above_threshold",
    "active_count": 7,
    "minimum_threshold": 5
  }
}
```

---

## 21. LLM Explanation Boundary

The recommendation engine stores a rule-based reason in:

```text
app.user_problem_recommendations.reason
```

The LLM may create a user-friendly version in:

```text
app.user_problem_recommendations.llm_reason
```

The LLM receives only selected recommendation evidence such as:

- Problem title
- Topic name
- Recommendation type
- Rule-based reason
- Score evidence
- Recent failed attempt context
- Revision due context

The LLM output is used only for explanation or notification text.

Example rule-based reason:

```text
You had repeated failed attempts in Dynamic Programming, and this problem is an easier curated practice problem in the same topic.
```

Example user-friendly LLM reason:

```text
Dynamic Programming has been difficult recently. Try this related problem next to rebuild confidence before moving to harder DP problems.
```

---

## 22. Frontend Surfaces

The same stored recommendation model can serve multiple surfaces.

| Surface | Purpose |
|---|---|
| `dashboard` | General recommendation widget |
| `today` | Daily practice plan |
| `weak_topics` | Topic-specific improvement area |
| `retry` | Failed problem retry area |
| `revision` | Revision practice area |
| `notification` | Push, email, or in-app notification |
| `chat` | Chat-based recommendation explanation |

Each recommendation row stores the intended surface in:

```text
app.user_problem_recommendations.surface
```

---

## 23. Ranking and Serving Rules

We serve recommendations using:

1. Status eligibility.
2. Expiry check.
3. Cooldown check.
4. Priority.
5. Rank.
6. Final score.
7. Diversity constraints.

Recommended serving logic:

```text
Use recommendations where:
- status is pending, active, or shown
- expires_at is null or in the future
- cooldown_until is null or in the past

Order by:
- priority descending
- rank ascending
- score descending

Limit:
- maximum 10 active recommendations
```

---

## 24. Observability and Debugging

Each recommendation stores enough evidence to explain why it exists.

We store:

- Recommendation type
- Reason code
- Human-readable reason
- Score
- Score breakdown
- Evidence JSON
- Trigger source
- Scoring version
- Status timestamps
- Related recommendation events

This lets us answer:

- Why did we recommend this problem?
- Which signal created it?
- Was it shown?
- Did the user click it?
- Did the user solve it?
- Was it dismissed?
- Was it generated from batch or live events?

---

## 25. Success Metrics

We measure recommendation quality through lifecycle events.

| Metric | Meaning |
|---|---|
| Impression rate | Recommendations shown to users |
| Click-through rate | Shown recommendations that are clicked |
| Start rate | Clicked recommendations that users attempt |
| Completion rate | Recommendations that lead to solved problems |
| Skip rate | Recommendations skipped by users |
| Dismiss rate | Recommendations dismissed by users |
| Retry success rate | Failed submission retry recommendations that later become solved |
| Revision completion rate | Revision recommendations completed by users |
| Active recommendation health | Whether users consistently have 5 to 10 active recommendations |
| Refresh success rate | Whether refresh events complete successfully |

---

## 26. Edge Cases and Handling Rules

### 26.1 New user with no submissions

Use:

- `standard_practice`
- `next_in_topic`
- `daily_practice`

Rules:

- Start from beginner-friendly curated order.
- Avoid `weak_topic_practice` because weakness is not known yet.
- Avoid `failed_submission_retry` because no failed attempts exist.
- Avoid `revision` because no solved problems exist.
- Avoid `next_difficulty` until enough solve history exists.

### 26.2 User has only failed submissions

Use:

- `failed_submission_retry`
- `weak_topic_practice`
- `standard_practice`

Rules:

- Prefer easier related problems.
- Avoid harder progression.
- Detect repeated failed topics.
- Recommend a confidence-building problem in the same topic.

### 26.3 User solved the recommended problem outside the recommendation flow

Handling:

- Mark recommendation as `completed`.
- Set `completed_at`.
- Append `recommendation_completed` event.
- Store `metadata.source = submission_sync`.

### 26.4 User dismisses many recommendations

Handling:

- If the user dismisses 3 or more recommendations in one session, record `refresh_requested`.
- Adjust new recommendations toward easier, more diverse, or more standard practice items.
- Apply dismissal cooldown to dismissed problems.

### 26.5 Recommendation is clicked but not solved

Handling:

- Move status to `clicked`.
- Set `clicked_at`.
- Do not immediately replace the recommendation.
- Give the user time to solve it.

### 26.6 User repeatedly fails a recommended problem

Handling:

- Keep the recommendation active while failed attempts are reasonable.
- If failed attempts become too high without progress, replace with an easier related problem.
- Append `recommendation_replaced` event.
- Store the replacement reason in metadata.

### 26.7 Same topic dominates recommendations

Handling:

- Maximum 2 problems from the same topic in top 5.
- Maximum 3 active recommendations from the same topic.
- Re-rank lower-scoring topics upward when needed for diversity.

### 26.8 Sync data is stale

Handling:

- Prefer `standard_practice` and `next_in_topic`.
- Reduce confidence in recent weakness or retry signals.
- Store sync freshness in evidence metadata.

### 26.9 Problem exists in curated sheet but not in problem catalog

Handling:

- Do not recommend the problem.
- Append `recommendation_generation_skipped` event.
- Store `metadata.reason = missing_catalog_problem`.

### 26.10 LLM explanation generation fails

Handling:

- Keep the recommendation.
- Use the rule-based `reason` field.
- Leave `llm_reason` null.
- Do not block recommendation serving.

### 26.11 User already has 10 active recommendations

Handling:

- Do not create more active recommendations.
- Append `recommendation_generation_skipped` if refresh was requested.
- Re-rank existing active recommendations if needed.

### 26.12 Active recommendations fall below 5

Handling:

- Append `refresh_requested` event.
- Generate new recommendations from recent accepted and failed signals.
- Refill up to the active recommendation target.

### 26.13 User marks a recommendation as not relevant

Handling:

- Mark status as `dismissed` or `replaced` based on product behavior.
- Append `recommendation_marked_not_relevant` event.
- Apply cooldown.
- Reduce similar recommendation patterns in the short term.

### 26.14 User skips a problem because it is already solved

Handling:

- Append `recommendation_skipped_already_solved` event.
- Mark recommendation as `completed` if submission sync confirms solve.
- Mark recommendation as `replaced` if solve is not confirmed.
- Avoid immediate duplicate recommendation.

---

## 27. Final Selected Architecture

We use four recommendation-specific tables:

```text
catalog.dsa_sheet_problems
app.user_problem_recommendations
app.recommendation_events
app.projector_cursors
```

We use a curated DSA sheet as the controlled candidate pool.

We use accepted submissions for progress and strength.

We use failed submissions for weakness, retry, and recovery recommendations.

We use batch processing for historical backfill.

We use event processing for live submissions and recommendation interactions.

We store recommendations before serving them to the frontend.

We use deterministic scoring with score breakdown and evidence.

We use event history to manage lifecycle, cooldowns, refreshes, and analytics.

We use LLM only for user-facing explanation text.
