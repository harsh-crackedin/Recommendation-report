# Recommendation Model Table Schema Report

## 1. Purpose

This report defines the selected recommendation model table schema. We use these tables to store the curated recommendation pool, generated user problem recommendations, recommendation lifecycle events, and event processing cursors.

The selected recommendation-specific tables are:

```text
catalog.dsa_sheet_problems
app.user_problem_recommendations
app.recommendation_events
app.projector_cursors
```

The existing shared tables remain external dependencies:

```text
public.users
catalog.coding_problems
```

---

## 2. Schema Overview

| Table | Purpose |
|---|---|
| `catalog.dsa_sheet_problems` | Stores curated DSA problems eligible for recommendations |
| `app.user_problem_recommendations` | Stores generated recommendations for each user and problem |
| `app.recommendation_events` | Stores append-only user-visible and internal recommendation events |
| `app.projector_cursors` | Stores event processing progress for recommendation projectors |

---

## 3. Table: `catalog.dsa_sheet_problems`

### 3.1 Purpose

We use `catalog.dsa_sheet_problems` as the controlled recommendation pool. Each row represents one curated DSA problem in a specific sheet, version, topic, and order.

We do not store full problem metadata here. We reference the shared problem catalog through:

```text
platform + problem_slug
```

### 3.2 SQL schema

```sql
create table catalog.dsa_sheet_problems (
  id bigserial primary key,

  platform text not null default 'leetcode',
  problem_slug text not null,

  sheet_name text not null default 'default_dsa_sheet',
  sheet_version text not null default 'v1',

  topic_slug text not null,
  topic_name text not null,

  problem_order integer not null,
  problem_title text not null,

  difficulty text
    check (difficulty in ('easy', 'medium', 'hard')),

  importance_score numeric(5,4) not null default 0.5000,
  interview_frequency_score numeric(5,4) not null default 0.5000,
  foundation_score numeric(5,4) not null default 0.5000,
  revision_value_score numeric(5,4) not null default 0.5000,

  estimated_minutes integer,

  source_name text,
  source_url text,

  metadata jsonb not null default '{}'::jsonb,

  is_active boolean not null default true,

  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),

  unique (sheet_name, sheet_version, topic_slug, problem_order),
  unique (sheet_name, sheet_version, platform, problem_slug),

  foreign key (platform, problem_slug)
    references catalog.coding_problems(platform, slug)
);
```

### 3.3 Column usage

| Column | Type | Required | Usage |
|---|---|---:|---|
| `id` | `bigserial` | Yes | Internal primary key for the curated sheet row |
| `platform` | `text` | Yes | Identifies the coding platform, defaulting to `leetcode` |
| `problem_slug` | `text` | Yes | References the problem slug in `catalog.coding_problems` |
| `sheet_name` | `text` | Yes | Groups problems under a named curated sheet |
| `sheet_version` | `text` | Yes | Allows versioned curated sheets |
| `topic_slug` | `text` | Yes | Stable machine-readable topic key, such as `dynamic-programming` |
| `topic_name` | `text` | Yes | Human-readable topic name, such as `Dynamic Programming` |
| `problem_order` | `integer` | Yes | Defines the curated order inside a topic |
| `problem_title` | `text` | Yes | Stores display title for quick recommendation rendering |
| `difficulty` | `text` | No | Stores easy, medium, or hard for simple scoring and filtering |
| `importance_score` | `numeric(5,4)` | Yes | Scores how important the problem is in our curated path |
| `interview_frequency_score` | `numeric(5,4)` | Yes | Scores how useful or common the problem is for interview preparation |
| `foundation_score` | `numeric(5,4)` | Yes | Scores how foundational the problem is for learning the topic |
| `revision_value_score` | `numeric(5,4)` | Yes | Scores how valuable the problem is for future revision |
| `estimated_minutes` | `integer` | No | Estimates how long the problem may take for planning and daily practice |
| `source_name` | `text` | No | Stores source list name, such as Striver, NeetCode, Blind 75, or internal sheet |
| `source_url` | `text` | No | Stores source URL when available |
| `metadata` | `jsonb` | Yes | Stores flexible extra information, tags, patterns, prerequisites, or notes |
| `is_active` | `boolean` | Yes | Controls whether the problem is eligible for recommendation |
| `created_at` | `timestamptz` | Yes | Tracks when the row was created |
| `updated_at` | `timestamptz` | Yes | Tracks when the row was last updated |

### 3.4 Constraints and indexes

| Constraint | Purpose |
|---|---|
| `primary key (id)` | Gives each curated row a stable internal identifier |
| `unique (sheet_name, sheet_version, topic_slug, problem_order)` | Prevents duplicate order positions inside a topic |
| `unique (sheet_name, sheet_version, platform, problem_slug)` | Prevents duplicate problem entries inside the same sheet version |
| `foreign key (platform, problem_slug)` | Ensures every curated problem exists in the shared problem catalog |
| `difficulty check` | Restricts difficulty to `easy`, `medium`, or `hard` |

### 3.5 Recommended supporting indexes

```sql
create index idx_dsa_sheet_active_topic_order
on catalog.dsa_sheet_problems(sheet_name, sheet_version, topic_slug, is_active, problem_order);

create index idx_dsa_sheet_problem_lookup
on catalog.dsa_sheet_problems(platform, problem_slug);

create index idx_dsa_sheet_active_score
on catalog.dsa_sheet_problems(is_active, importance_score, interview_frequency_score);
```

---

## 4. Table: `app.user_problem_recommendations`

### 4.1 Purpose

We use `app.user_problem_recommendations` to store generated recommendations for each user. This table is the serving table for frontend recommendation surfaces.

Each row represents one recommended problem for one user and one recommendation type.

### 4.2 SQL schema

```sql
create table app.user_problem_recommendations (
  id bigserial primary key,

  user_id uuid not null references public.users(id) on delete cascade,

  platform text not null default 'leetcode',
  problem_slug text not null,

  sheet_name text not null default 'default_dsa_sheet',
  sheet_version text not null default 'v1',

  topic_slug text not null,
  topic_name text not null,

  recommendation_type text not null
    check (
      recommendation_type in (
        'standard_practice',
        'weak_topic_practice',
        'failed_submission_retry',
        'revision',
        'next_in_topic',
        'similar_problem',
        'next_difficulty',
        'daily_practice'
      )
    ),

  status text not null default 'pending'
    check (
      status in (
        'pending',
        'active',
        'shown',
        'clicked',
        'started',
        'skipped',
        'dismissed',
        'completed',
        'expired',
        'replaced'
      )
    ),

  surface text not null default 'dashboard'
    check (
      surface in (
        'dashboard',
        'today',
        'weak_topics',
        'retry',
        'revision',
        'notification',
        'chat'
      )
    ),

  priority integer not null default 0,
  rank integer not null,
  score numeric(8,4) not null default 0,

  reason_code text not null,
  reason text not null,
  llm_reason text,

  score_breakdown jsonb not null default '{}'::jsonb,
  evidence jsonb not null default '{}'::jsonb,

  scoring_version text not null default 'mvp_v1',

  trigger_source text not null default 'batch'
    check (
      trigger_source in (
        'initial_backfill',
        'below_active_threshold',
        'live_submission',
        'failed_submission',
        'accepted_submission',
        'manual_refresh',
        'daily_refresh',
        'recommendation_interaction',
        'batch'
      )
    ),

  generated_at timestamptz not null default now(),
  shown_at timestamptz,
  clicked_at timestamptz,
  started_at timestamptz,
  completed_at timestamptz,
  skipped_at timestamptz,
  dismissed_at timestamptz,
  expires_at timestamptz,
  cooldown_until timestamptz,

  metadata jsonb not null default '{}'::jsonb,

  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),

  unique (user_id, platform, problem_slug, recommendation_type),

  foreign key (platform, problem_slug)
    references catalog.coding_problems(platform, slug)
);
```

### 4.3 Column usage

| Column | Type | Required | Usage |
|---|---|---:|---|
| `id` | `bigserial` | Yes | Primary key for the recommendation row |
| `user_id` | `uuid` | Yes | Identifies the user receiving the recommendation |
| `platform` | `text` | Yes | Identifies the coding platform, defaulting to `leetcode` |
| `problem_slug` | `text` | Yes | Identifies the recommended problem |
| `sheet_name` | `text` | Yes | Identifies the curated sheet used for the recommendation |
| `sheet_version` | `text` | Yes | Identifies the curated sheet version |
| `topic_slug` | `text` | Yes | Stores machine-readable topic key for filtering and ranking |
| `topic_name` | `text` | Yes | Stores display topic name |
| `recommendation_type` | `text` | Yes | Stores why this recommendation exists at a category level |
| `status` | `text` | Yes | Tracks lifecycle state of the recommendation |
| `surface` | `text` | Yes | Identifies where we intend to show the recommendation |
| `priority` | `integer` | Yes | Allows urgent or product-specific ordering before rank |
| `rank` | `integer` | Yes | Stores final order after scoring and diversity rules |
| `score` | `numeric(8,4)` | Yes | Stores final recommendation score |
| `reason_code` | `text` | Yes | Stores machine-readable reason, such as `repeated_dp_failures` |
| `reason` | `text` | Yes | Stores rule-based user-facing explanation fallback |
| `llm_reason` | `text` | No | Stores optional LLM-generated explanation text |
| `score_breakdown` | `jsonb` | Yes | Stores component scores, weights, penalties, and final score details |
| `evidence` | `jsonb` | Yes | Stores supporting facts used to create the recommendation |
| `scoring_version` | `text` | Yes | Stores the scoring formula or rule version used to generate the row |
| `trigger_source` | `text` | Yes | Stores what caused this recommendation to be generated |
| `generated_at` | `timestamptz` | Yes | Tracks when the recommendation was generated |
| `shown_at` | `timestamptz` | No | Tracks when the recommendation was first shown |
| `clicked_at` | `timestamptz` | No | Tracks when the recommendation was clicked |
| `started_at` | `timestamptz` | No | Tracks when the user started the recommended problem |
| `completed_at` | `timestamptz` | No | Tracks when the recommended problem was completed |
| `skipped_at` | `timestamptz` | No | Tracks when the recommendation was skipped |
| `dismissed_at` | `timestamptz` | No | Tracks when the recommendation was dismissed |
| `expires_at` | `timestamptz` | No | Controls when the recommendation becomes stale |
| `cooldown_until` | `timestamptz` | No | Controls when the same problem can be recommended again |
| `metadata` | `jsonb` | Yes | Stores flexible extra data for serving, debugging, or product rules |
| `created_at` | `timestamptz` | Yes | Tracks row creation time |
| `updated_at` | `timestamptz` | Yes | Tracks row update time |

### 4.4 Recommendation type values

| Value | Usage |
|---|---|
| `standard_practice` | Important unsolved curated problem |
| `weak_topic_practice` | Practice problem for weak topic improvement |
| `failed_submission_retry` | Retry problem based on prior failed attempts |
| `revision` | Previously solved problem due for revision |
| `next_in_topic` | Next unsolved problem in curated topic order |
| `similar_problem` | Similar or related problem for reinforcement |
| `next_difficulty` | Suitable step up in difficulty |
| `daily_practice` | Recommendation used in a daily practice set |

### 4.5 Status values

| Value | Usage |
|---|---|
| `pending` | Generated but not yet shown |
| `active` | Available for frontend serving |
| `shown` | Already shown to the user |
| `clicked` | User clicked the recommendation |
| `started` | User started attempting the problem |
| `skipped` | User skipped the recommendation |
| `dismissed` | User dismissed or rejected the recommendation |
| `completed` | User completed the recommended problem |
| `expired` | Recommendation is stale |
| `replaced` | Recommendation was replaced |

### 4.6 Surface values

| Value | Usage |
|---|---|
| `dashboard` | General dashboard recommendation area |
| `today` | Daily practice surface |
| `weak_topics` | Weak-topic recommendation area |
| `retry` | Failed retry recommendation area |
| `revision` | Revision surface |
| `notification` | In-app, push, or email notification |
| `chat` | Chat-based recommendation response |

### 4.7 Trigger source values

| Value | Usage |
|---|---|
| `initial_backfill` | Historical sync created the recommendation |
| `below_active_threshold` | Active count dropped below the minimum threshold |
| `live_submission` | New submission triggered generation |
| `failed_submission` | Failed submission triggered generation |
| `accepted_submission` | Accepted submission triggered generation |
| `manual_refresh` | User or admin triggered refresh |
| `daily_refresh` | Daily refresh generated the recommendation |
| `recommendation_interaction` | User interaction triggered update or replacement |
| `batch` | Generic batch processing trigger |

### 4.8 Example `score_breakdown`

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

### 4.9 Example `evidence`

```json
{
  "topic": "Dynamic Programming",
  "recent_failed_attempts_in_topic": 5,
  "problem_failed_attempts": 3,
  "problem_accepted_count": 0,
  "last_failed_at": "2026-06-04T10:15:00Z",
  "difficulty_match": "medium_is_current_best_fit",
  "curated_source": "Internal DSA Sheet",
  "active_count_at_generation": 4
}
```

### 4.10 Constraints and indexes

| Constraint | Purpose |
|---|---|
| `primary key (id)` | Gives each recommendation row a stable identifier |
| `unique (user_id, platform, problem_slug, recommendation_type)` | Prevents duplicate recommendation rows for the same problem and type |
| `foreign key user_id` | Removes recommendations when a user is deleted |
| `foreign key platform, problem_slug` | Ensures recommended problem exists in the shared catalog |
| `recommendation_type check` | Restricts recommendation types to selected values |
| `status check` | Restricts lifecycle state to selected values |
| `surface check` | Restricts frontend surface values |
| `trigger_source check` | Restricts allowed generation triggers |

### 4.11 Recommended supporting indexes

```sql
create index idx_user_problem_recommendations_serving
on app.user_problem_recommendations(user_id, status, expires_at, priority desc, rank asc, score desc);

create index idx_user_problem_recommendations_problem
on app.user_problem_recommendations(user_id, platform, problem_slug);

create index idx_user_problem_recommendations_type
on app.user_problem_recommendations(user_id, recommendation_type, status);

create index idx_user_problem_recommendations_cooldown
on app.user_problem_recommendations(user_id, cooldown_until);

create index idx_user_problem_recommendations_topic
on app.user_problem_recommendations(user_id, topic_slug, status);
```

---

## 5. Table: `app.recommendation_events`

### 5.1 Purpose

We use `app.recommendation_events` as the append-only event log for recommendation activity.

This table stores:

- User-visible lifecycle events.
- Internal processing events.
- Submission-driven recommendation processing events.
- Refresh request, refresh start, refresh completion, and refresh failure events.
- Event metadata needed for debugging and analytics.

### 5.2 SQL schema

```sql
create table app.recommendation_events (
  id bigserial primary key,

  user_id uuid not null references public.users(id) on delete cascade,

  recommendation_id bigint
    references app.user_problem_recommendations(id)
    on delete set null,

  event_type text not null
    check (
      event_type in (
        'recommendation_created',
        'recommendation_shown',
        'recommendation_clicked',
        'recommendation_started',
        'recommendation_skipped',
        'recommendation_dismissed',
        'recommendation_completed',
        'recommendation_expired',
        'recommendation_replaced',
        'recommendation_marked_not_relevant',
        'recommendation_skipped_already_solved',

        'refresh_requested',
        'refresh_started',
        'refresh_completed',
        'refresh_failed',
        'historical_submission_processed',
        'live_submission_processed',
        'accepted_submission_processed',
        'failed_submission_processed',
        'batch_recommendations_generated',
        'event_recommendations_generated',
        'recommendation_generation_skipped'
      )
    ),

  visibility text not null default 'user'
    check (visibility in ('user', 'internal')),

  platform text not null default 'leetcode',
  problem_slug text,

  sheet_name text default 'default_dsa_sheet',
  sheet_version text default 'v1',

  topic_slug text,
  topic_name text,

  recommendation_type text,
  surface text default 'dashboard',

  reason text,
  score numeric(8,4),

  idempotency_key text,
  request_id text,
  session_id text,

  metadata jsonb not null default '{}'::jsonb,

  created_at timestamptz not null default now(),

  foreign key (platform, problem_slug)
    references catalog.coding_problems(platform, slug)
);
```

### 5.3 Column usage

| Column | Type | Required | Usage |
|---|---|---:|---|
| `id` | `bigserial` | Yes | Primary key for the event row |
| `user_id` | `uuid` | Yes | Identifies the user connected to the event |
| `recommendation_id` | `bigint` | No | Links the event to a recommendation row when applicable |
| `event_type` | `text` | Yes | Stores the event name |
| `visibility` | `text` | Yes | Marks event as `user` or `internal` |
| `platform` | `text` | Yes | Identifies coding platform, defaulting to `leetcode` |
| `problem_slug` | `text` | No | Identifies related problem when applicable |
| `sheet_name` | `text` | No | Identifies related curated sheet when applicable |
| `sheet_version` | `text` | No | Identifies related curated sheet version when applicable |
| `topic_slug` | `text` | No | Stores related topic key when applicable |
| `topic_name` | `text` | No | Stores related topic display name when applicable |
| `recommendation_type` | `text` | No | Stores recommendation type involved in the event |
| `surface` | `text` | No | Stores frontend or notification surface involved in the event |
| `reason` | `text` | No | Stores short explanation for the event |
| `score` | `numeric(8,4)` | No | Stores score at event time when useful |
| `idempotency_key` | `text` | No | Prevents duplicate processing events for the same trigger |
| `request_id` | `text` | No | Groups events from the same backend request |
| `session_id` | `text` | No | Groups events from the same user session |
| `metadata` | `jsonb` | Yes | Stores event-specific payload, debug data, counts, errors, and processing details |
| `created_at` | `timestamptz` | Yes | Stores event creation time |

### 5.4 User-visible event values

| Event type | Usage |
|---|---|
| `recommendation_created` | We created a recommendation row |
| `recommendation_shown` | The recommendation was shown to the user |
| `recommendation_clicked` | The user clicked the recommendation |
| `recommendation_started` | The user started attempting the recommendation |
| `recommendation_skipped` | The user skipped the recommendation |
| `recommendation_dismissed` | The user dismissed the recommendation |
| `recommendation_completed` | The user completed the recommendation |
| `recommendation_expired` | The recommendation expired |
| `recommendation_replaced` | The recommendation was replaced |
| `recommendation_marked_not_relevant` | The user marked the recommendation as not relevant |
| `recommendation_skipped_already_solved` | The user indicated the problem was already solved |

### 5.5 Internal event values

| Event type | Usage |
|---|---|
| `refresh_requested` | We requested recommendation generation or refresh |
| `refresh_started` | We started recommendation generation or refresh |
| `refresh_completed` | We completed recommendation generation or refresh |
| `refresh_failed` | Recommendation generation or refresh failed |
| `historical_submission_processed` | Historical submission data was processed |
| `live_submission_processed` | A live submission was processed |
| `accepted_submission_processed` | An accepted submission signal was processed |
| `failed_submission_processed` | A failed submission signal was processed |
| `batch_recommendations_generated` | Batch processing generated recommendations |
| `event_recommendations_generated` | Event processing generated recommendations |
| `recommendation_generation_skipped` | Generation was skipped due to guardrails or missing data |

### 5.6 Example internal event metadata

```json
{
  "visibility": "internal",
  "trigger_source": "below_active_threshold",
  "processing_mode": "batch",
  "scoring_version": "mvp_v1",
  "active_count_before": 4,
  "candidate_count": 58,
  "selected_count": 8,
  "inserted_count": 6,
  "replaced_count": 2,
  "duration_ms": 420
}
```

### 5.7 Example failure metadata

```json
{
  "visibility": "internal",
  "trigger_source": "live_submission",
  "error_code": "SCORING_FAILED",
  "error_message": "difficulty missing for candidate",
  "retryable": true
}
```

### 5.8 Constraints and indexes

| Constraint | Purpose |
|---|---|
| `primary key (id)` | Gives each event a stable identifier |
| `foreign key user_id` | Removes user events when the user is deleted |
| `foreign key recommendation_id` | Links an event to a recommendation when applicable |
| `foreign key platform, problem_slug` | Ensures problem-linked events reference known catalog problems |
| `event_type check` | Restricts event names to selected values |
| `visibility check` | Restricts visibility to `user` or `internal` |

### 5.9 Recommended supporting indexes

```sql
create index idx_recommendation_events_user_time
on app.recommendation_events(user_id, created_at desc);

create index idx_recommendation_events_recommendation
on app.recommendation_events(recommendation_id, created_at desc);

create index idx_recommendation_events_type_time
on app.recommendation_events(event_type, created_at desc);

create index idx_recommendation_events_visibility
on app.recommendation_events(visibility, created_at desc);

create unique index idx_recommendation_events_idempotency
on app.recommendation_events(user_id, idempotency_key)
where idempotency_key is not null;
```

---

## 6. Table: `app.projector_cursors`

### 6.1 Purpose

We use `app.projector_cursors` to track how far a processor has read from an event source. This prevents repeated processing of the same events and keeps live recommendation processing safer.

### 6.2 SQL schema

```sql
create table app.projector_cursors (
  id bigserial primary key,

  projector_name text not null,
  event_source text not null,

  last_processed_event_id bigint,
  last_processed_at timestamptz,

  metadata jsonb not null default '{}'::jsonb,

  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),

  unique (projector_name, event_source)
);
```

### 6.3 Column usage

| Column | Type | Required | Usage |
|---|---|---:|---|
| `id` | `bigserial` | Yes | Primary key for the cursor row |
| `projector_name` | `text` | Yes | Identifies the processor or projector using the cursor |
| `event_source` | `text` | Yes | Identifies the event stream or source being processed |
| `last_processed_event_id` | `bigint` | No | Stores the latest processed event ID |
| `last_processed_at` | `timestamptz` | No | Stores when processing last advanced |
| `metadata` | `jsonb` | Yes | Stores extra processing state or debug data |
| `created_at` | `timestamptz` | Yes | Tracks cursor creation time |
| `updated_at` | `timestamptz` | Yes | Tracks cursor update time |

### 6.4 Constraints and indexes

| Constraint | Purpose |
|---|---|
| `primary key (id)` | Gives each cursor row a stable identifier |
| `unique (projector_name, event_source)` | Ensures one cursor per processor and source pair |

### 6.5 Recommended supporting index

```sql
create index idx_projector_cursors_lookup
on app.projector_cursors(projector_name, event_source);
```

---

## 7. Scoring Fields Used Across Tables

### 7.1 Curated problem scoring fields

| Field | Stored in | Usage |
|---|---|---|
| `importance_score` | `catalog.dsa_sheet_problems` | Scores how important the problem is |
| `interview_frequency_score` | `catalog.dsa_sheet_problems` | Scores interview usefulness |
| `foundation_score` | `catalog.dsa_sheet_problems` | Scores learning foundation value |
| `revision_value_score` | `catalog.dsa_sheet_problems` | Scores revision value |

### 7.2 Generated recommendation scoring fields

| Field | Stored in | Usage |
|---|---|---|
| `score` | `app.user_problem_recommendations` | Final ranking score |
| `rank` | `app.user_problem_recommendations` | Final serving order |
| `priority` | `app.user_problem_recommendations` | Product-level ordering before rank |
| `score_breakdown` | `app.user_problem_recommendations` | Full score explanation |
| `evidence` | `app.user_problem_recommendations` | Supporting facts behind the recommendation |
| `scoring_version` | `app.user_problem_recommendations` | Rule version used to generate the score |

---

## 8. Selected Scoring Formula

We use this deterministic MVP formula:

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

The final value is stored in:

```text
app.user_problem_recommendations.score
```

The full calculation is stored in:

```text
app.user_problem_recommendations.score_breakdown
```

---

## 9. Serving Query Shape

Recommended frontend serving query shape:

```sql
select *
from app.user_problem_recommendations
where user_id = :user_id
  and status in ('pending', 'active', 'shown')
  and (expires_at is null or expires_at > now())
  and (cooldown_until is null or cooldown_until <= now())
order by priority desc, rank asc, score desc
limit 10;
```

---

## 10. Data Retention and Lifecycle Notes

### 10.1 Recommendation rows

We keep recommendation rows after completion, dismissal, expiry, or replacement. The row status changes instead of deleting the row.

### 10.2 Event rows

We keep event rows append-only. We do not update old events to represent new behavior.

### 10.3 Curated sheet rows

We deactivate curated problems using:

```text
is_active = false
```

We do not need to delete old curated rows when a problem should stop being recommended.

### 10.4 Projector cursors

We update cursor rows as processing progresses. Each projector and event source pair has one row.

---

## 11. Selected Guardrails Represented in Schema

| Guardrail | Schema support |
|---|---|
| Only curated problems are recommended | `catalog.dsa_sheet_problems` controls eligible pool |
| Problem must exist in shared catalog | Foreign key on `platform, problem_slug` |
| Avoid duplicate same-type recommendations | Unique key on `user_id, platform, problem_slug, recommendation_type` |
| Track lifecycle cleanly | `status` and lifecycle timestamp columns |
| Track why a recommendation exists | `reason_code`, `reason`, `score_breakdown`, `evidence` |
| Track frontend surface | `surface` |
| Track generation source | `trigger_source` |
| Track cooldown | `cooldown_until` |
| Track expiry | `expires_at` |
| Track internal events without another table | `recommendation_events.visibility = internal` |
| Prevent duplicate processing events | `idempotency_key` unique partial index |
| Track event processing progress | `projector_cursors` |

---

## 12. Final Table Relationship Summary

```text
public.users
    |
    | 1-to-many
    v
app.user_problem_recommendations
    |
    | 1-to-many
    v
app.recommendation_events

catalog.coding_problems
    ^
    |
    | referenced by platform + problem_slug
    |
catalog.dsa_sheet_problems
    |
    | used to generate
    v
app.user_problem_recommendations

app.recommendation_events
    |
    | processed by projector
    v
app.projector_cursors
```

---

## 13. Final Schema Decision

We use four recommendation-specific tables:

```text
catalog.dsa_sheet_problems
app.user_problem_recommendations
app.recommendation_events
app.projector_cursors
```

We store recommendation decisions in `app.user_problem_recommendations`.

We store lifecycle and processing history in `app.recommendation_events`.

We store curated recommendation eligibility and scoring hints in `catalog.dsa_sheet_problems`.

We store event processing progress in `app.projector_cursors`.
