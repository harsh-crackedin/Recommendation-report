# Hybrid Recommendation Model Implementation Plan

## 1. Purpose

This implementation plan explains how to build the selected hybrid recommendation model for coding practice.

The recommendation model recommends LeetCode problems from a curated internal DSA sheet by using:

- Historical LeetCode submissions.
- Live LeetCode submissions.
- Accepted and failed submission patterns.
- User problem aggregates.
- Problem catalog metadata.
- Curated sheet metadata.
- Recommendation interaction events.

The plan is designed to be executable by the engineering team. It defines the implementation phases, data reads, user-state classification, recommendation-type selection, scoring, ranking, event handling, frontend APIs, guardrails, test cases, and edge cases.

This plan keeps the selected MVP architecture simple:

```text
Existing user coding data
  -> Recommendation trigger
  -> User state classifier
  -> Recommendation type decision layer
  -> Candidate generation
  -> Scoring
  -> Guardrails and diversity rules
  -> app.user_problem_recommendations
  -> Frontend surfaces
  -> app.recommendation_events
```

The model does not add a separate recommendation jobs table in V1. Internal processing events are stored in `app.recommendation_events`.

---

## 2. Existing Schema Alignment

The recommendation model reads from the existing production schema and writes into the recommendation-specific tables.

### 2.1 Existing catalog table

We use:

```text
catalog.coding_problems
```

Important fields used by the recommendation model:

| Field | Use |
|---|---|
| `platform` | Platform key, currently `leetcode` |
| `slug` | Stable problem identifier |
| `title` | Display title |
| `difficulty` | Used for difficulty matching and next-difficulty decisions |
| `topic_tags` | Used for topic matching, similar-problem fallback, and weak-topic mapping |
| `ac_rate` | Optional difficulty/usefulness signal |
| `similar_slugs` | Used for `similar_problem` recommendations |
| `sync_status` | Used to avoid recommending incomplete catalog rows when required |
| `raw_meta` | Optional metadata for future explanation or source mapping |

### 2.2 Existing user profile table

We use:

```text
user_data.user_coding_profiles
```

Important fields used by the recommendation model:

| Field | Use |
|---|---|
| `user_id` | User identity |
| `platform` | Platform key |
| `external_handle` | External coding profile handle |
| `is_connected` | User is connected to a coding platform |
| `sync_enabled` | Sync is enabled for the platform |
| `last_full_sync_at` | Used to detect stale historical sync |
| `last_delta_sync_at` | Used to detect stale delta sync |
| `last_live_capture_at` | Used to detect recent live activity |
| `last_sync_error` | Used to downgrade personalization if sync is failing |

### 2.3 Existing user problem aggregate table

We use:

```text
user_data.user_coding_problems
```

Important fields used by the recommendation model:

| Field | Use |
|---|---|
| `user_id` | User identity |
| `platform` | Platform key |
| `slug` | Problem identifier |
| `status` | Current aggregate status, such as `ac` or `tried` |
| `first_attempt_at` | Used to detect old attempts |
| `first_ac_at` | Used for revision due calculation |
| `latest_ac_at` | Used for revision due and recency |
| `latest_attempt_at` | Used for failed retry and recent activity |
| `attempt_count` | Used for effort and difficulty signals |
| `ac_count` | Used to determine solved status |
| `wa_count` | Used for failed-attempt signal |
| `tle_count` | Used for failed-attempt signal |
| `mle_count` | Used for failed-attempt signal |
| `re_count` | Used for failed-attempt signal |
| `ce_count` | Used for failed-attempt signal |
| `other_count` | Used for failed-attempt signal |
| `best_runtime_ms` | Optional future confidence signal |
| `best_memory_kb` | Optional future confidence signal |
| `best_lang` | Optional explanation and language preference signal |
| `primary_submission_id` | Used to connect aggregate state to a representative submission |

### 2.4 Existing user submission table

We use:

```text
user_data.user_coding_submissions
```

Important fields used by the recommendation model:

| Field | Use |
|---|---|
| `platform` | Platform key |
| `submission_id` | Submission identifier |
| `user_id` | User identity |
| `profile_id` | Coding profile reference |
| `slug` | Problem identifier |
| `verdict` | Normalized verdict: `AC`, `WA`, `TLE`, `MLE`, `RE`, `CE`, `OTHER` |
| `raw_status` | Original platform status |
| `status_code` | Platform status code |
| `lang` | Language preference signal |
| `runtime_ms` | Optional confidence/performance signal |
| `memory_kb` | Optional confidence/performance signal |
| `total_correct` | Used to detect near-solve failed attempts |
| `total_testcases` | Used to detect near-solve failed attempts |
| `last_testcase` | Debug/evidence only |
| `expected_output` | Debug/evidence only |
| `actual_output` | Debug/evidence only |
| `runtime_error` | Debug/evidence only |
| `compile_error` | Debug/evidence only |
| `ts` | Submission timestamp |
| `captured_at` | Backend capture timestamp |
| `capture_source` | `historical_sync` or `live_capture` |
| `has_code` | Whether code is available |
| `code_capture_method` | Historical or live server capture method |
| `raw_meta` | Flexible metadata |

### 2.5 Existing sync state table

We use:

```text
system.sync_state
```

Important fields used by the recommendation model:

| Field | Use |
|---|---|
| `phase` | Detect whether sync is idle, running, done, live, failed, paused, or auth-required |
| `cursor` | Debug and sync progress evidence |
| `last_progress_at` | Detect stale sync |
| `last_full_sync_at` | Detect whether historical import is complete |
| `retry_count` | Detect unstable sync |
| `last_error` | Downgrade personalization when sync has errors |

---

## 3. Recommendation-Specific Tables Used by This Plan

The implementation uses these selected recommendation tables:

```text
catalog.dsa_sheet_problems
app.user_problem_recommendations
app.recommendation_events
app.projector_cursors
```

### 3.1 `catalog.dsa_sheet_problems`

Purpose:

```text
Controlled recommendation pool.
```

This table decides which problems are eligible for recommendations.

### 3.2 `app.user_problem_recommendations`

Purpose:

```text
Serving table for generated recommendations.
```

The frontend reads from this table.

### 3.3 `app.recommendation_events`

Purpose:

```text
Append-only lifecycle and internal processing log.
```

This table stores user-visible events and internal processing events.

### 3.4 `app.projector_cursors`

Purpose:

```text
Tracks event processing progress.
```

This prevents duplicate processing when reading from event streams.

---

## 4. Implementation Goals

The implementation must satisfy these goals:

1. Generate recommendations only from `catalog.dsa_sheet_problems`.
2. Use accepted and failed submissions.
3. Select recommendation types deterministically.
4. Store every generated recommendation in `app.user_problem_recommendations`.
5. Store lifecycle and processing events in `app.recommendation_events`.
6. Keep active recommendations limited and diverse.
7. Make each recommendation explainable using `reason_code`, `reason`, `score_breakdown`, and `evidence`.
8. Keep LLM usage out of ranking and candidate selection.
9. Support historical batch refresh and live event refresh.
10. Keep the MVP implementation debuggable without adding extra operational tables.

---

## 5. Recommended Service Layout

Create a recommendation service package:

```text
api/services/recommendations/
  __init__.py
  models.py
  repository.py
  user_state.py
  type_decider.py
  candidate_generator.py
  scorer.py
  guardrails.py
  diversity.py
  service.py
  routes.py
  event_writer.py
```

### 5.1 File responsibilities

| File | Responsibility |
|---|---|
| `models.py` | Internal Pydantic/data classes for user state, candidates, scores, and recommendation commands |
| `repository.py` | SQL reads/writes for recommendation inputs and output tables |
| `user_state.py` | Builds the user state snapshot from submissions, aggregates, sync state, and events |
| `type_decider.py` | Decides which recommendation types are eligible for a user |
| `candidate_generator.py` | Generates candidates for each eligible recommendation type |
| `scorer.py` | Computes component scores, penalties, and final score |
| `guardrails.py` | Applies hard filters and safety rules |
| `diversity.py` | Applies caps and ranking diversity rules |
| `service.py` | Orchestrates refresh generation and lifecycle transitions |
| `routes.py` | Exposes frontend and internal endpoints |
| `event_writer.py` | Writes user-visible and internal events into `app.recommendation_events` |

---

## 6. Implementation Phase 1: Database Migration

### 6.1 Create or update `catalog.dsa_sheet_problems`

Use this as the curated sheet table:

```sql
create table if not exists catalog.dsa_sheet_problems (
  id bigserial primary key,

  platform text not null default 'leetcode',
  problem_slug text not null,

  sheet_name text not null default 'default_dsa_sheet',
  sheet_version text not null default 'v1',

  topic_slug text not null,
  topic_name text not null,

  problem_order integer not null,
  problem_title text not null,

  difficulty text check (difficulty in ('easy', 'medium', 'hard')),

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

### 6.2 Create `app.user_problem_recommendations`

```sql
create table if not exists app.user_problem_recommendations (
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
        'recommendation_interaction'
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

### 6.3 Create `app.recommendation_events`

```sql
create table if not exists app.recommendation_events (
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

### 6.4 Create or keep `app.projector_cursors`

```sql
create table if not exists app.projector_cursors (
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

### 6.5 Add indexes

```sql
create index if not exists idx_dsa_sheet_active_topic_order
on catalog.dsa_sheet_problems(sheet_name, sheet_version, topic_slug, is_active, problem_order);

create index if not exists idx_dsa_sheet_problem_lookup
on catalog.dsa_sheet_problems(platform, problem_slug);

create index if not exists idx_dsa_sheet_active_score
on catalog.dsa_sheet_problems(is_active, importance_score, interview_frequency_score);

create index if not exists idx_user_problem_recommendations_serving
on app.user_problem_recommendations(user_id, status, expires_at, priority desc, rank asc, score desc);

create index if not exists idx_user_problem_recommendations_problem
on app.user_problem_recommendations(user_id, platform, problem_slug);

create index if not exists idx_user_problem_recommendations_type
on app.user_problem_recommendations(user_id, recommendation_type, status);

create index if not exists idx_user_problem_recommendations_cooldown
on app.user_problem_recommendations(user_id, cooldown_until);

create index if not exists idx_user_problem_recommendations_topic
on app.user_problem_recommendations(user_id, topic_slug, status);

create index if not exists idx_recommendation_events_user_time
on app.recommendation_events(user_id, created_at desc);

create index if not exists idx_recommendation_events_recommendation
on app.recommendation_events(recommendation_id, created_at desc);

create index if not exists idx_recommendation_events_type_time
on app.recommendation_events(event_type, created_at desc);

create index if not exists idx_recommendation_events_visibility
on app.recommendation_events(visibility, created_at desc);

create unique index if not exists idx_recommendation_events_idempotency
on app.recommendation_events(user_id, idempotency_key)
where idempotency_key is not null;
```

---

## 7. Implementation Phase 2: Seed the Curated DSA Sheet

### 7.1 Seed requirements

Before generation can work, the curated sheet must contain active problems.

Each seeded row must include:

```text
platform
problem_slug
sheet_name
sheet_version
topic_slug
topic_name
problem_order
problem_title
difficulty
importance_score
interview_frequency_score
foundation_score
revision_value_score
estimated_minutes
source_name
metadata
is_active
```

### 7.2 Metadata format

Use `metadata` for pattern-level and prerequisite information:

```json
{
  "patterns": ["hash-map", "prefix-sum"],
  "prerequisites": ["arrays", "hashing"],
  "learning_stage": "foundation",
  "is_beginner_friendly": true,
  "is_revision_friendly": true,
  "similar_concepts": ["two-sum", "lookup-table"],
  "notes": "Canonical hash map problem for array lookups."
}
```

### 7.3 Seed validation rules

Before enabling recommendations, validate:

1. Every `problem_slug` exists in `catalog.coding_problems`.
2. Every active row has `difficulty`.
3. Every topic has ordered `problem_order` values.
4. No duplicate problem exists in the same sheet version.
5. `importance_score`, `interview_frequency_score`, `foundation_score`, and `revision_value_score` are between `0` and `1`.
6. Beginner topics have at least 3 active beginner-friendly problems.
7. High-value topics have at least one revision-friendly problem.

---

## 8. Implementation Phase 3: Build the User State Snapshot

Every recommendation refresh starts by building a user state snapshot.

### 8.1 Snapshot inputs

Read:

```text
user_data.user_coding_profiles
system.sync_state
user_data.user_coding_problems
user_data.user_coding_submissions
catalog.coding_problems
catalog.dsa_sheet_problems
app.user_problem_recommendations
app.recommendation_events
```

### 8.2 Snapshot output

Create an in-memory object:

```text
UserRecommendationState
```

Suggested fields:

```python
class UserRecommendationState:
    user_id: str
    platform: str
    profile_connected: bool
    sync_enabled: bool
    sync_phase: str | None
    sync_stale: bool
    total_attempted: int
    total_solved: int
    total_failed_attempts: int
    recent_attempt_count_7d: int
    recent_attempt_count_30d: int
    recent_ac_count_7d: int
    recent_failed_count_7d: int
    solved_slugs: set[str]
    attempted_slugs: set[str]
    failed_unsolved_slugs: set[str]
    recently_shown_slugs: set[str]
    recently_dismissed_slugs: set[str]
    active_recommendation_slugs: set[str]
    topic_stats: dict[str, TopicStats]
    difficulty_stats: dict[str, DifficultyStats]
    user_maturity: str
    active_count: int
```

### 8.3 Problem-level derived signals

For each problem in `user_data.user_coding_problems`, derive:

| Signal | Formula |
|---|---|
| `is_solved` | `ac_count > 0` or `status = 'ac'` |
| `is_attempted` | `attempt_count > 0` |
| `failed_attempt_count` | `wa_count + tle_count + mle_count + re_count + ce_count + other_count` |
| `failure_ratio` | `failed_attempt_count / greatest(attempt_count, 1)` |
| `days_since_latest_attempt` | `now - latest_attempt_at` |
| `days_since_latest_ac` | `now - latest_ac_at` |
| `is_failed_unsolved` | `attempt_count > 0 and ac_count = 0` |
| `is_revision_due` | `ac_count > 0 and latest_ac_at older than threshold` |

### 8.4 Submission-level derived signals

For each recent submission in `user_data.user_coding_submissions`, derive:

| Signal | Formula |
|---|---|
| `is_accepted` | `verdict = 'AC'` |
| `is_failed` | `verdict in ('WA', 'TLE', 'MLE', 'RE', 'CE', 'OTHER')` |
| `near_solve_score` | `total_correct / total_testcases` when both values exist |
| `recent_failure` | `is_failed and ts >= now - 7 days` |
| `recent_success` | `is_accepted and ts >= now - 7 days` |

### 8.5 Topic-level derived signals

Use `catalog.dsa_sheet_problems.topic_slug` as the recommendation topic source.

Join user problem aggregates to curated sheet rows by:

```text
platform + slug/problem_slug
```

For each topic, compute:

| Signal | Formula |
|---|---|
| `topic_attempt_count` | Sum of attempts across topic problems |
| `topic_solved_count` | Count of solved topic problems |
| `topic_failed_attempt_count` | Sum of failed attempts across topic problems |
| `topic_failed_problem_count` | Count of problems with failed attempts |
| `topic_unsolved_attempted_count` | Count of attempted but unsolved topic problems |
| `topic_total_curated_count` | Count of active curated problems in topic |
| `topic_solve_rate` | `topic_solved_count / topic_total_curated_count` |
| `topic_failure_ratio` | `topic_failed_attempt_count / topic_attempt_count` |
| `last_topic_ac_at` | Latest accepted timestamp inside topic |
| `last_topic_attempt_at` | Latest attempt timestamp inside topic |
| `topic_stale` | No accepted problem in topic for 30+ days |

---

## 9. Implementation Phase 4: Classify the User Type

The model must identify what kind of user is being served. This classification controls which recommendation types become eligible.

A user can have multiple states at the same time.

### 9.1 Primary maturity type

Use solved count as the primary maturity signal.

| User maturity | Condition | Meaning |
|---|---|---|
| `new_user` | `total_solved = 0` and `total_attempted = 0` | No useful coding history yet |
| `starter` | `total_solved < 5` | Very limited accepted history |
| `beginner` | `total_solved >= 5 and total_solved < 25` | Enough history for simple personalization |
| `regular` | `total_solved >= 25 and total_solved < 100` | Full MVP personalization is useful |
| `advanced` | `total_solved >= 100` | Use revision, next difficulty, and targeted weakness strongly |

### 9.2 Activity state

| Activity state | Condition | Meaning |
|---|---|---|
| `inactive_user` | No attempt in last 14 days | User needs safer re-entry recommendations |
| `recently_active_user` | At least 1 attempt in last 7 days | Live signals can be used strongly |
| `high_activity_user` | 10+ attempts in last 7 days | More frequent refresh is useful |
| `sync_stale_user` | Sync has old progress or recent error | Avoid over-personalizing from stale data |

### 9.3 Learning state

| Learning state | Condition | Meaning |
|---|---|---|
| `recent_failed_problem` | Any unsolved problem with failed attempts in last 7 days | Eligible for `failed_submission_retry` |
| `weak_topic_detected` | Any topic weakness score >= 0.65 | Eligible for `weak_topic_practice` |
| `revision_due` | Any solved curated problem older than revision threshold | Eligible for `revision` |
| `topic_progression_ready` | User solved or attempted a topic problem and next ordered problem exists | Eligible for `next_in_topic` |
| `similar_reinforcement_ready` | User recently solved a problem confidently | Eligible for `similar_problem` |
| `difficulty_progression_ready` | User has recent success at current difficulty with low failure rate | Eligible for `next_difficulty` |
| `low_active_recommendations` | Active count below 5 | Generate more recommendations |

---

## 10. Implementation Phase 5: Decide Which Recommendation Types to Use

The recommendation type decision layer runs before candidate generation.

It answers:

```text
Which recommendation types are eligible for this user right now?
```

### 10.1 Type priority

Use this priority order:

```text
1. failed_submission_retry
2. weak_topic_practice
3. revision
4. next_in_topic
5. similar_problem
6. next_difficulty
7. standard_practice
8. daily_practice
```

Priority decides which candidate groups are filled first. Diversity rules still prevent one type from dominating the final list.

### 10.2 Type caps

| Recommendation type | Max active rows per user | Purpose |
|---|---:|---|
| `failed_submission_retry` | 2 | Avoid overwhelming the user with failure reminders |
| `weak_topic_practice` | 3 | Keep weakness practice focused |
| `revision` | 2 | Avoid too much old material |
| `next_in_topic` | 3 | Support structured progress |
| `similar_problem` | 2 | Reinforce without repetition |
| `next_difficulty` | 1 | Keep difficulty jumps controlled |
| `standard_practice` | Fill remaining slots | Safe fallback and balanced practice |
| `daily_practice` | Surface wrapper only | Used for today/dashboard packaging |

### 10.3 Decision table

| User state | Eligible recommendation type | Eligibility rule |
|---|---|---|
| `new_user` | `standard_practice` | Always eligible when curated beginner problems exist |
| `new_user` | `next_in_topic` | Eligible from the first problem in a beginner topic |
| `starter` | `standard_practice` | Always eligible |
| `starter` | `next_in_topic` | Eligible after first attempt or first solved curated problem |
| `starter` | `failed_submission_retry` | Eligible if failed unsolved problem has 2+ attempts |
| `beginner` | `weak_topic_practice` | Eligible when topic weakness score >= 0.65 |
| `beginner` | `failed_submission_retry` | Eligible for recent unsolved failed problem |
| `beginner` | `next_in_topic` | Eligible when next ordered problem is suitable |
| `regular` | All types | Apply all eligibility rules |
| `advanced` | All types | Apply all rules, increase revision and next-difficulty quality bar |
| `inactive_user` | `revision` | Eligible for old solved curated problems |
| `inactive_user` | `standard_practice` | Eligible for safe re-entry |
| `sync_stale_user` | `standard_practice` | Use safe curated problems |
| `recent_failed_problem` | `failed_submission_retry` | Eligible when failed problem is not solved |
| `weak_topic_detected` | `weak_topic_practice` | Eligible when topic weakness score >= 0.65 |
| `revision_due` | `revision` | Eligible when solved curated problem is stale |
| `topic_progression_ready` | `next_in_topic` | Eligible when next ordered curated problem exists |
| `similar_reinforcement_ready` | `similar_problem` | Eligible after confident recent solve |
| `difficulty_progression_ready` | `next_difficulty` | Eligible after current difficulty mastery |

---

## 11. Recommendation Type Rules

### 11.1 `standard_practice`

Use when:

```text
No stronger signal exists
OR user is new/starter
OR active recommendations are below threshold
OR sync data is stale
```

Candidate source:

```text
catalog.dsa_sheet_problems
```

Candidate rules:

1. `is_active = true`.
2. Problem is not solved.
3. Problem is not active for the user.
4. Problem is not recently dismissed.
5. Prefer high `importance_score` and high `foundation_score`.
6. For new users, prefer `difficulty = 'easy'` and `metadata.is_beginner_friendly = true`.

Reason codes:

```text
important_unsolved_problem
curated_foundation_problem
beginner_friendly_start
safe_reentry_practice
```

### 11.2 `weak_topic_practice`

Use when:

```text
topic_weakness_score >= 0.65
OR topic_failed_attempt_count >= 5
OR topic_failed_problem_count >= 2
OR topic_solve_rate <= 0.30 and topic_attempt_count >= 3
```

Candidate source:

```text
Unsolved active curated problems from the weak topic.
```

Candidate rules:

1. Same `topic_slug` as weak topic.
2. Problem is unsolved.
3. Difficulty is not above the user's allowed difficulty band.
4. If the user repeatedly failed hard problems, choose easier or same-level problems first.
5. Prefer `foundation_score` when failure ratio is high.

Reason codes:

```text
repeated_topic_failures
low_topic_solve_rate
topic_needs_strengthening
easier_topic_rebuild
```

### 11.3 `failed_submission_retry`

Use when:

```text
Problem has failed_attempt_count >= 2
AND ac_count = 0
AND latest_attempt_at is recent enough
```

Strong signal:

```text
failed_attempt_count >= 3
AND latest_attempt_at within 7 days
```

Candidate source:

```text
The same failed problem, only if that problem exists in the active curated DSA sheet.
```

Candidate rules:

1. Problem is attempted.
2. Problem is unsolved.
3. Problem exists in `catalog.dsa_sheet_problems`.
4. Problem was not recently dismissed.
5. If failed attempts are more than 5 and no progress exists, replace with `weak_topic_practice` instead of retry.
6. If `total_correct / total_testcases >= 0.70`, strongly prefer retry.

Reason codes:

```text
recent_failed_problem
multiple_failed_attempts
almost_solved_retry
retry_unsolved_problem
```

### 11.4 `revision`

Use when:

```text
Problem is solved
AND problem exists in active curated sheet
AND latest_ac_at is old enough
```

Revision thresholds:

| Time since latest AC | Revision due score |
|---|---:|
| 0-7 days | 0.00 |
| 8-21 days | 0.25 |
| 22-45 days | 0.50 |
| 46-90 days | 0.75 |
| 90+ days | 1.00 |

Candidate source:

```text
Solved curated problems.
```

Candidate rules:

1. Problem is solved.
2. Problem has high `revision_value_score` or high `importance_score`.
3. Avoid recommending too many revisions at once.
4. Prefer topics not practiced recently.

Reason codes:

```text
revision_due
important_problem_stale
topic_not_practiced_recently
high_value_revision
```

### 11.5 `next_in_topic`

Use when:

```text
User has solved or attempted a problem in a topic
AND the next ordered unsolved curated problem exists
AND user is not currently struggling too much in that topic
```

Candidate source:

```text
catalog.dsa_sheet_problems ordered by topic_slug + problem_order.
```

Candidate rules:

1. Find highest solved or attempted `problem_order` in the topic.
2. Select the next unsolved active problem in the same topic.
3. If the user failed the previous topic problem many times, choose a foundation problem instead.
4. Do not jump too far ahead in the curated sequence.

Reason codes:

```text
next_problem_in_curated_order
continue_topic_progression
complete_topic_sequence
```

### 11.6 `similar_problem`

Use when:

```text
User recently solved a problem confidently
AND an unsolved similar active curated problem exists
```

Confident solve condition:

```text
verdict = 'AC'
AND failed attempts before AC <= 2
AND latest_ac_at within 14 days
```

Candidate source priority:

1. `catalog.coding_problems.similar_slugs`.
2. Curated problems with matching `metadata.patterns`.
3. Curated problems with same `topic_slug` and nearby difficulty.

Candidate rules:

1. Candidate must be unsolved.
2. Candidate must be active in the curated sheet.
3. Candidate must not duplicate an active recommendation.
4. Avoid too many same-topic recommendations.

Reason codes:

```text
reinforce_recent_solve
same_pattern_practice
similar_problem_after_success
```

### 11.7 `next_difficulty`

Use when:

```text
User has shown mastery at current difficulty
AND next difficulty problem exists
AND recent failure rate is acceptable
```

Readiness rules:

| Current maturity | Readiness condition |
|---|---|
| Beginner | 5+ easy solves and recent failure ratio < 0.40 |
| Regular | 3+ recent solves at current difficulty and recent failure ratio < 0.50 |
| Advanced | 3+ recent medium solves or strong topic solve rate |

Candidate source:

```text
Active curated problems one difficulty level above current comfort band.
```

Candidate rules:

1. Only move `easy -> medium` or `medium -> hard`.
2. Do not move `easy -> hard`.
3. Do not recommend next difficulty if the user is inactive or has many recent failures.
4. Prefer same topic where the user has demonstrated success.

Reason codes:

```text
ready_for_medium
ready_for_hard
difficulty_progression
stretch_problem_ready
```

### 11.8 `daily_practice`

Use as a packaging surface, not as the main generation reason.

The daily set is selected from the final ranked recommendations and may contain:

```text
failed_submission_retry
weak_topic_practice
revision
next_in_topic
similar_problem
next_difficulty
standard_practice
```

Reason codes:

```text
balanced_daily_set
mixed_practice_plan
```

---

## 12. Implementation Phase 6: Generate Candidates

### 12.1 Candidate object

Use an internal candidate model:

```python
class RecommendationCandidate:
    user_id: str
    platform: str
    problem_slug: str
    problem_title: str
    sheet_name: str
    sheet_version: str
    topic_slug: str
    topic_name: str
    difficulty: str | None
    recommendation_type: str
    reason_code: str
    evidence: dict
    raw_scores: dict
```

### 12.2 Candidate generation order

Generate candidates in this order:

```text
1. failed_submission_retry candidates
2. weak_topic_practice candidates
3. revision candidates
4. next_in_topic candidates
5. similar_problem candidates
6. next_difficulty candidates
7. standard_practice candidates
```

`daily_practice` is created by selecting and packaging the final top recommendations.

### 12.3 Candidate generation SQL patterns

#### Active curated pool

```sql
select
  dsp.*,
  cp.title as catalog_title,
  cp.difficulty as catalog_difficulty,
  cp.topic_tags,
  cp.similar_slugs,
  cp.ac_rate
from catalog.dsa_sheet_problems dsp
join catalog.coding_problems cp
  on cp.platform = dsp.platform
 and cp.slug = dsp.problem_slug
where dsp.sheet_name = $1
  and dsp.sheet_version = $2
  and dsp.is_active = true;
```

#### Existing active recommendations

```sql
select platform, problem_slug, recommendation_type, status, cooldown_until
from app.user_problem_recommendations
where user_id = $1
  and status in ('pending', 'active', 'shown', 'clicked', 'started')
  and (expires_at is null or expires_at > now());
```

#### Recently dismissed recommendations

```sql
select platform, problem_slug, recommendation_type, cooldown_until
from app.user_problem_recommendations
where user_id = $1
  and status in ('skipped', 'dismissed', 'replaced')
  and cooldown_until is not null
  and cooldown_until > now();
```

#### User problem aggregates

```sql
select
  ucp.*,
  cp.difficulty,
  cp.topic_tags,
  cp.similar_slugs
from user_data.user_coding_problems ucp
join catalog.coding_problems cp
  on cp.platform = ucp.platform
 and cp.slug = ucp.slug
where ucp.user_id = $1
  and ucp.platform = $2;
```

#### Recent submissions

```sql
select *
from user_data.user_coding_submissions
where user_id = $1
  and platform = $2
  and ts >= now() - interval '30 days'
order by ts desc;
```

---

## 13. Implementation Phase 7: Score Candidates

### 13.1 Final score formula

Use this deterministic formula:

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

### 13.2 Score components

| Component | Range | Source |
|---|---:|---|
| `importance_score` | 0.00-1.00 | `catalog.dsa_sheet_problems.importance_score` |
| `interview_frequency_score` | 0.00-1.00 | `catalog.dsa_sheet_problems.interview_frequency_score` |
| `topic_weakness_score` | 0.00-1.00 | Derived topic stats |
| `difficulty_match_score` | 0.00-1.00 | User maturity + problem difficulty |
| `failed_attempt_score` | 0.00-1.00 | User problem aggregate + recent submissions |
| `revision_due_score` | 0.00-1.00 | Solved timestamp + revision value |
| `progression_score` | 0.00-1.00 | Curated topic order / difficulty step |
| `freshness_score` | 0.00-1.00 | Recommendation history |
| `penalty_score` | 0.00+ | Guardrail penalties |

### 13.3 Topic weakness score

```text
topic_weakness_score = min(1.0,
  0.45 * topic_failure_ratio
+ 0.25 * repeated_failure_score
+ 0.20 * unsolved_attempted_topic_score
+ 0.10 * stale_topic_score
)
```

Where:

```text
topic_failure_ratio = topic_failed_attempt_count / greatest(topic_attempt_count, 1)
repeated_failure_score = min(1.0, topic_failed_problem_count / 3)
unsolved_attempted_topic_score = topic_unsolved_attempted_count / greatest(topic_attempted_problem_count, 1)
stale_topic_score = 1.0 if no accepted topic problem in last 30 days else 0.0
```

### 13.4 Difficulty match score

| User maturity | Easy | Medium | Hard |
|---|---:|---:|---:|
| `new_user` | 1.00 | 0.20 | 0.00 |
| `starter` | 1.00 | 0.40 | 0.00 |
| `beginner` | 0.75 | 1.00 | 0.20 |
| `regular` | 0.40 | 1.00 | 0.60 |
| `advanced` | 0.20 | 0.85 | 0.90 |

Apply a hard block when the score is `0.00`.

### 13.5 Failed attempt score

```text
failed_attempt_score = min(1.0,
  0.35 * failed_attempt_count_score
+ 0.25 * recent_failure_score
+ 0.25 * near_solve_score
+ 0.15 * unsolved_boost
)
```

| Failed attempts | `failed_attempt_count_score` |
|---:|---:|
| 0 | 0.00 |
| 1 | 0.25 |
| 2 | 0.50 |
| 3 | 0.75 |
| 4+ | 1.00 |

| Last failed attempt | `recent_failure_score` |
|---|---:|
| Within 1 day | 1.00 |
| Within 7 days | 0.75 |
| Within 30 days | 0.40 |
| Older | 0.10 |
| No failure | 0.00 |

`near_solve_score`:

```text
near_solve_score = total_correct / total_testcases
```

If `total_correct` or `total_testcases` is missing, use `0.00`.

### 13.6 Revision due score

| Time since latest AC | Score |
|---|---:|
| 0-7 days | 0.00 |
| 8-21 days | 0.25 |
| 22-45 days | 0.50 |
| 46-90 days | 0.75 |
| 90+ days | 1.00 |

Final revision component:

```text
revision_due_score = time_due_score * max(revision_value_score, importance_score)
```

### 13.7 Progression score

| Condition | Score |
|---|---:|
| Exact next unsolved problem in same topic | 1.00 |
| Nearby next problem in same topic | 0.75 |
| One difficulty level higher after mastery | 0.70 |
| Same topic but not ordered progression | 0.40 |
| No progression relation | 0.00 |

### 13.8 Freshness score

| Event history | Score |
|---|---:|
| Never recommended | 1.00 |
| Not shown in 30+ days | 0.75 |
| Not shown in 14+ days | 0.50 |
| Shown within 14 days | 0.20 |
| Currently active | 0.00 |

### 13.9 Type-specific boosts

Apply boosts after base scoring:

| Recommendation type | Boost condition | Boost |
|---|---|---:|
| `failed_submission_retry` | Failed attempts >= 3 | +0.10 |
| `failed_submission_retry` | Near solve score >= 0.70 | +0.10 |
| `weak_topic_practice` | Topic weakness score >= 0.75 | +0.10 |
| `revision` | Latest AC 45+ days ago | +0.08 |
| `next_in_topic` | Exact next ordered problem | +0.08 |
| `similar_problem` | Recent confident solve within 7 days | +0.06 |
| `next_difficulty` | Current-level mastery detected | +0.08 |
| `standard_practice` | Importance score >= 0.80 | +0.05 |

### 13.10 Penalties

| Penalty | Condition | Value |
|---|---|---:|
| `recently_shown_penalty` | Shown within 7 days | 0.20 |
| `recently_dismissed_penalty` | Dismissed and cooldown active | Hard filter |
| `same_topic_overexposure_penalty` | Topic already has 2 active recommendations | 0.15 |
| `too_hard_penalty` | Difficulty is above allowed band | Hard filter or 0.40 |
| `already_solved_penalty` | Solved and type is not `revision` | Hard filter |
| `stale_sync_penalty` | Sync is stale and type relies on recent data | 0.10 |
| `too_many_failures_penalty` | Failed attempts > 5 and near solve score < 0.50 | 0.25 |

### 13.11 Stored score breakdown

Store the full calculation in `score_breakdown`:

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
    "importance_score": 0.8,
    "interview_frequency_score": 0.7,
    "topic_weakness_score": 0.9,
    "difficulty_match_score": 0.8,
    "failed_attempt_score": 1.0,
    "revision_due_score": 0.0,
    "progression_score": 0.6,
    "freshness_score": 0.5
  },
  "boosts": {
    "type_boost": 0.1
  },
  "penalties": {
    "same_topic_overexposure_penalty": 0.1
  },
  "final_score": 0.754
}
```

---

## 14. Implementation Phase 8: Apply Guardrails

Guardrails are hard filters applied before final ranking.

### 14.1 Hard filters

Remove a candidate if:

1. The curated sheet row is inactive.
2. The problem is missing from `catalog.coding_problems`.
3. The problem is already active for the user.
4. The problem is solved and recommendation type is not `revision`.
5. The problem was dismissed and cooldown is still active.
6. The problem is outside the curated sheet.
7. The problem has inappropriate difficulty for the user.
8. The candidate has missing required fields.
9. The recommendation type has already reached its active cap.
10. The same problem appears multiple times in the candidate pool and a higher-priority type already selected it.

### 14.2 Difficulty guardrails

| User maturity | Allowed difficulty |
|---|---|
| `new_user` | Easy only |
| `starter` | Easy, selected medium only |
| `beginner` | Easy and medium |
| `regular` | Easy, medium, selected hard |
| `advanced` | Medium and hard, selected easy for revision/foundation only |

### 14.3 Failure guardrails

For `failed_submission_retry`:

```text
If failed_attempt_count > 5 and near_solve_score < 0.50:
  do not recommend the same failed problem.
  use weak_topic_practice with easier/foundation candidate instead.
```

### 14.4 Sync guardrails

If sync is stale or failing:

1. Do not overuse recent failed-submission signals.
2. Prefer `standard_practice`, `revision`, and safe `next_in_topic`.
3. Store `metadata.sync_freshness = 'stale'` in generated recommendations.

---

## 15. Implementation Phase 9: Apply Diversity Rules

After scoring, sort candidates by:

```text
priority desc, score desc, recommendation_type priority, freshness desc
```

Then build the final set using caps.

### 15.1 Active set size

```text
maximum active recommendations = 10
minimum active recommendation threshold = 5
```

Refresh should run when active count is below 5 or when a strong live signal appears.

### 15.2 Topic caps

```text
Top 5: max 2 same-topic problems
Full active set: max 3 same-topic problems
```

### 15.3 Type caps

Use the type caps from Section 10.2.

### 15.4 Difficulty mix

For a normal daily set of 5:

```text
Beginner: 3 easy, 2 medium
Regular: 1 easy, 3 medium, 1 hard/stretch
Advanced: 3 medium, 2 hard
```

Use this as a soft target, not a hard rule.

### 15.5 Daily set packaging

For the `today` surface:

| Slot | Preferred type |
|---:|---|
| 1 | `failed_submission_retry` or `weak_topic_practice` |
| 2 | `next_in_topic` |
| 3 | `standard_practice` |
| 4 | `revision` |
| 5 | `similar_problem` or `next_difficulty` |

If a preferred type has no valid candidate, fill with `standard_practice`.

---

## 16. Implementation Phase 10: Persist Recommendations

### 16.1 Insert or update behavior

For each selected recommendation:

1. Upsert into `app.user_problem_recommendations` using unique key:

```text
user_id + platform + problem_slug + recommendation_type
```

2. If the existing row is completed, dismissed, skipped, or expired and cooldown has passed, create a refreshed row by updating:

```text
status
score
rank
reason_code
reason
score_breakdown
evidence
trigger_source
generated_at
expires_at
metadata
updated_at
```

3. Do not update a row if:

```text
status in ('clicked', 'started') and expires_at > now()
```

This avoids replacing a problem while the user is working on it.

### 16.2 Status on creation

Use:

```text
status = 'pending'
```

When the frontend fetches and displays it, update to:

```text
status = 'shown'
shown_at = now()
```

### 16.3 Expiry defaults

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

### 16.4 Cooldown defaults after dismiss/skip

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

## 17. Implementation Phase 11: Write Recommendation Events

Use `app.recommendation_events` for user-visible and internal events.

### 17.1 Internal events

| Event | When to write |
|---|---|
| `refresh_requested` | Before refresh starts |
| `refresh_started` | When refresh execution begins |
| `refresh_completed` | When refresh succeeds |
| `refresh_failed` | When refresh fails |
| `historical_submission_processed` | After historical sync batch is processed |
| `live_submission_processed` | After live captured submission is processed |
| `accepted_submission_processed` | After accepted submission is used as signal |
| `failed_submission_processed` | After failed submission is used as signal |
| `batch_recommendations_generated` | After historical/batch generation |
| `event_recommendations_generated` | After live event generation |
| `recommendation_generation_skipped` | When refresh is skipped due to active count or idempotency |

### 17.2 User-visible events

| Event | When to write |
|---|---|
| `recommendation_created` | A new recommendation row is created |
| `recommendation_shown` | Frontend displays it |
| `recommendation_clicked` | User clicks attempt/open |
| `recommendation_started` | User actually starts/attempts the problem |
| `recommendation_skipped` | User skips it |
| `recommendation_dismissed` | User dismisses it |
| `recommendation_completed` | User solves it |
| `recommendation_expired` | Recommendation expires |
| `recommendation_replaced` | Recommendation is replaced by another |
| `recommendation_marked_not_relevant` | User gives negative relevance feedback |
| `recommendation_skipped_already_solved` | User says they already solved it |

### 17.3 Idempotency keys

Use `idempotency_key` to prevent duplicate internal processing events.

Examples:

```text
initial_backfill:{user_id}:{platform}:{last_full_sync_at}
live_submission:{user_id}:{platform}:{submission_id}
accepted_submission:{user_id}:{platform}:{submission_id}
failed_submission:{user_id}:{platform}:{submission_id}
below_threshold:{user_id}:{platform}:{yyyy-mm-dd-hh}
manual_refresh:{user_id}:{platform}:{request_id}
daily_refresh:{user_id}:{platform}:{yyyy-mm-dd}
```

### 17.4 Example internal event metadata

```json
{
  "visibility": "internal",
  "trigger_source": "live_submission",
  "processing_mode": "event",
  "idempotency_key": "live_submission:user-id:leetcode:123456",
  "scoring_version": "mvp_v1",
  "candidate_count": 42,
  "selected_count": 8,
  "inserted_count": 6,
  "updated_count": 2,
  "skipped_count": 4,
  "duration_ms": 430
}
```

---

## 18. Implementation Phase 12: Integrate With Submission Sync

### 18.1 Historical sync integration

After historical metadata/code sync completes and `user_data.user_coding_problems` is recomputed:

1. Write `historical_submission_processed` event.
2. Trigger recommendation refresh with:

```text
trigger_source = initial_backfill
processing_mode = batch
```

3. Generate up to 10 active recommendations.
4. Write `batch_recommendations_generated`.

### 18.2 Live submission integration

After `ingest_live_submission` upserts the submission, code blob, and recomputes the aggregate:

1. Read the normalized verdict.
2. Write `live_submission_processed`.
3. If verdict is `AC`, write `accepted_submission_processed`.
4. If verdict is not `AC`, write `failed_submission_processed`.
5. Trigger recommendation refresh with:

```text
trigger_source = accepted_submission
```

or:

```text
trigger_source = failed_submission
```

6. If the submitted problem has an active recommendation, update its lifecycle:

```text
AC      -> completed
Non-AC  -> started
```

### 18.3 Active recommendation completion by sync

When a live or historical submission shows that a recommended problem is accepted:

```sql
update app.user_problem_recommendations
set status = 'completed',
    completed_at = now(),
    updated_at = now()
where user_id = $1
  and platform = $2
  and problem_slug = $3
  and status in ('pending', 'active', 'shown', 'clicked', 'started');
```

Then write:

```text
recommendation_completed
```

---

## 19. Implementation Phase 13: API Endpoints

### 19.1 Fetch recommendations

```text
GET /api/recommendations
```

Query params:

```text
surface=dashboard|today|weak_topics|retry|revision|notification|chat
limit=10
```

Behavior:

1. Read from `app.user_problem_recommendations`.
2. Return non-expired rows with status `pending`, `active`, `shown`, `clicked`, or `started`.
3. Join with `catalog.coding_problems` for title, difficulty, and display metadata.
4. Mark rows as `shown` only when the frontend confirms they were actually displayed.

### 19.2 Mark shown

```text
POST /api/recommendations/{id}/shown
```

Updates:

```text
status = shown
shown_at = now()
```

Writes event:

```text
recommendation_shown
```

### 19.3 Mark clicked

```text
POST /api/recommendations/{id}/clicked
```

Updates:

```text
status = clicked
clicked_at = now()
```

Writes event:

```text
recommendation_clicked
```

### 19.4 Mark started

```text
POST /api/recommendations/{id}/started
```

Updates:

```text
status = started
started_at = now()
```

Writes event:

```text
recommendation_started
```

### 19.5 Skip/dismiss

```text
POST /api/recommendations/{id}/skip
POST /api/recommendations/{id}/dismiss
```

Updates:

```text
status = skipped or dismissed
skipped_at/dismissed_at = now()
cooldown_until = now() + recommendation_type cooldown
```

Writes event:

```text
recommendation_skipped
recommendation_dismissed
```

### 19.6 Manual refresh

```text
POST /api/recommendations/refresh
```

Behavior:

1. Write `refresh_requested`.
2. Run refresh synchronously or enqueue internally using the existing application worker pattern.
3. Write `refresh_started`.
4. Generate recommendations.
5. Write `refresh_completed` or `refresh_failed`.

No separate job table is required.

---

## 20. Implementation Phase 14: Main Refresh Algorithm

### 20.1 Refresh function signature

```python
async def refresh_user_recommendations(
    user_id: str,
    platform: str = "leetcode",
    trigger_source: str = "manual_refresh",
    surface: str = "dashboard",
    request_id: str | None = None,
) -> RefreshResult:
    ...
```

### 20.2 Refresh pseudocode

```python
async def refresh_user_recommendations(user_id, platform, trigger_source, surface, request_id=None):
    idempotency_key = build_idempotency_key(user_id, platform, trigger_source, request_id)

    if await events.exists_idempotency_key(user_id, idempotency_key):
        await events.write_internal("recommendation_generation_skipped", reason="duplicate_idempotency_key")
        return RefreshResult(skipped=True)

    await events.write_internal("refresh_requested", idempotency_key=idempotency_key)
    await events.write_internal("refresh_started", idempotency_key=idempotency_key)

    try:
        state = await user_state_builder.build(user_id, platform)

        if state.active_count >= 5 and trigger_source == "below_active_threshold":
            await events.write_internal("recommendation_generation_skipped", reason="active_count_sufficient")
            return RefreshResult(skipped=True)

        eligible_types = type_decider.decide(state)

        candidates = []
        for recommendation_type in eligible_types:
            candidates.extend(candidate_generator.generate(state, recommendation_type))

        candidates = guardrails.apply_hard_filters(state, candidates)
        scored_candidates = scorer.score_all(state, candidates)
        selected = diversity.select_final_set(state, scored_candidates, max_count=10)

        persisted = await repository.upsert_recommendations(selected)
        await events.write_created_events(persisted)

        await events.write_internal(
            "refresh_completed",
            metadata={
                "candidate_count": len(candidates),
                "selected_count": len(selected),
                "inserted_or_updated_count": len(persisted),
                "eligible_types": eligible_types,
            },
        )

        return RefreshResult(skipped=False, selected_count=len(selected))

    except Exception as exc:
        await events.write_internal(
            "refresh_failed",
            metadata={"error_message": str(exc), "retryable": True},
        )
        raise
```

---

## 21. Implementation Phase 15: Reason Text and LLM Boundary

### 21.1 Rule-based reason

Every recommendation must have a deterministic `reason` before any LLM call.

Examples:

| Type | Reason |
|---|---|
| `failed_submission_retry` | `You recently attempted this problem multiple times but have not solved it yet.` |
| `weak_topic_practice` | `You have repeated failed attempts in Dynamic Programming, so this problem strengthens that topic.` |
| `revision` | `You solved this important problem earlier, and it is due for revision.` |
| `next_in_topic` | `This is the next unsolved problem in your current topic path.` |
| `similar_problem` | `You recently solved a similar pattern, so this problem helps reinforce it.` |
| `next_difficulty` | `You have been solving your current difficulty level consistently, so this is a controlled step up.` |
| `standard_practice` | `This is a high-value curated problem for your current practice level.` |

### 21.2 LLM usage

The LLM may only generate:

```text
llm_reason
notification text
friendly explanation
```

The LLM must not decide:

```text
candidate generation
recommendation type
score
rank
filters
cooldown
expiry
```

### 21.3 LLM fallback

If LLM generation fails:

```text
Keep the recommendation.
Use the deterministic reason.
Leave llm_reason as null.
```

---

## 22. Implementation Phase 16: Test Plan

### 22.1 Unit tests

Create tests for:

| Module | Tests |
|---|---|
| `user_state.py` | Maturity classification, active counts, sync stale detection |
| `type_decider.py` | Eligible type decisions for new, beginner, regular, advanced, inactive users |
| `candidate_generator.py` | Candidate generation per type |
| `scorer.py` | Component scores, boosts, penalties, final score |
| `guardrails.py` | Solved filters, cooldown filters, difficulty filters |
| `diversity.py` | Type caps, topic caps, active set size |
| `event_writer.py` | Internal/user event creation and idempotency key behavior |

### 22.2 Scenario tests

#### New user

Input:

```text
0 submissions
0 solved
0 failed
```

Expected:

```text
standard_practice
next_in_topic from beginner-friendly curated problems
```

#### Recent failed problem

Input:

```text
3 WA submissions on same curated problem
0 AC
latest attempt yesterday
```

Expected:

```text
failed_submission_retry
reason_code = multiple_failed_attempts
high failed_attempt_score
```

#### Weak topic

Input:

```text
8 attempts in Dynamic Programming
1 solved
7 failed attempts
```

Expected:

```text
weak_topic_practice
topic_weakness_score >= 0.65
```

#### Revision due

Input:

```text
Solved important curated problem 70 days ago
```

Expected:

```text
revision
revision_due_score = 0.75 * revision_value_score/importance factor
```

#### Next in topic

Input:

```text
Solved Arrays order 1 and 2
Arrays order 3 unsolved
```

Expected:

```text
next_in_topic for Arrays order 3
```

#### Similar problem

Input:

```text
Recently solved Two Sum confidently
Unsolved similar slug exists in curated sheet
```

Expected:

```text
similar_problem
```

#### Next difficulty

Input:

```text
5 recent Easy solves
low failure ratio
medium problem exists in same topic
```

Expected:

```text
next_difficulty
```

### 22.3 Integration tests

Test full flow:

1. Seed curated sheet.
2. Insert catalog problems.
3. Insert historical submissions.
4. Recompute `user_data.user_coding_problems`.
5. Run recommendation refresh.
6. Assert rows in `app.user_problem_recommendations`.
7. Assert events in `app.recommendation_events`.
8. Simulate click/skip/completion.
9. Run refresh again.
10. Assert cooldown and duplicate guardrails.

### 22.4 Data quality tests

Validate:

1. No active curated problem references a missing catalog problem.
2. No active curated row has missing topic or order.
3. No generated recommendation is outside curated sheet.
4. No active recommendation violates difficulty guardrails.
5. No duplicate active problem exists for the same user.
6. Every generated recommendation has `score_breakdown` and `evidence`.

---

## 23. Implementation Phase 17: Rollout Plan

### 23.1 Local development

1. Apply migrations.
2. Seed 50-100 curated problems.
3. Create fixture users:
   - new user
   - beginner
   - weak-topic user
   - failed-retry user
   - advanced user
4. Run refresh manually.
5. Inspect recommendation rows and event logs.

### 23.2 Internal staging

1. Enable refresh only through manual endpoint.
2. Add dashboard API read path.
3. Log refresh events.
4. Review recommendation quality manually.
5. Tune thresholds and scores.

### 23.3 Controlled production rollout

1. Enable for internal users first.
2. Enable for users with completed historical sync.
3. Enable live-submission refresh.
4. Monitor event volume and duplicate rows.
5. Monitor click, skip, dismiss, and completion rates.

---

## 24. Observability and Metrics

Track these metrics from `app.recommendation_events`:

| Metric | Calculation |
|---|---|
| Recommendation generation success rate | `refresh_completed / refresh_started` |
| Generation failure rate | `refresh_failed / refresh_started` |
| Average candidate count | From `refresh_completed.metadata.candidate_count` |
| Average selected count | From `refresh_completed.metadata.selected_count` |
| Impression count | Count `recommendation_shown` |
| Click-through rate | `recommendation_clicked / recommendation_shown` |
| Start rate | `recommendation_started / recommendation_shown` |
| Completion rate | `recommendation_completed / recommendation_started` |
| Skip rate | `recommendation_skipped / recommendation_shown` |
| Dismiss rate | `recommendation_dismissed / recommendation_shown` |
| Retry success rate | Completed `failed_submission_retry` / started `failed_submission_retry` |
| Revision completion rate | Completed `revision` / shown `revision` |
| Weak topic engagement | Clicked `weak_topic_practice` / shown `weak_topic_practice` |

---

## 25. Edge Cases

### 25.1 New user has no submissions

Behavior:

```text
Use standard_practice and next_in_topic.
Use beginner-friendly easy problems.
Do not use failed_submission_retry, revision, similar_problem, or next_difficulty.
```

### 25.2 User has only failed submissions

Behavior:

```text
Use failed_submission_retry if the failed problem is appropriate.
Use weak_topic_practice if repeated topic failure exists.
Use standard_practice to add safe foundation problems.
Avoid next_difficulty.
```

### 25.3 User has only accepted submissions

Behavior:

```text
Use next_in_topic, revision, similar_problem, next_difficulty, and standard_practice.
Weak topic practice should only appear if there are enough failed attempts or low topic coverage.
```

### 25.4 User solved the recommended problem outside the recommendation UI

Behavior:

```text
When sync detects AC for the problem, mark active recommendation as completed.
Write recommendation_completed with metadata.source = submission_sync.
```

### 25.5 User repeatedly fails the recommended problem

Behavior:

```text
If failed attempts are <= 5, keep it active or started.
If failed attempts > 5 and near_solve_score < 0.50, replace with easier weak_topic_practice.
Write recommendation_replaced.
```

### 25.6 User dismisses many recommendations

Behavior:

```text
If 3+ dismissals occur in one session, reduce similar recommendation types.
Increase freshness and diversity requirements.
Prefer standard_practice or easier weak_topic_practice.
```

### 25.7 Sync is stale or failing

Behavior:

```text
Prefer stable recommendations from curated sheet.
Avoid strong recent-failure logic.
Set metadata.sync_freshness = stale.
```

### 25.8 Catalog metadata is incomplete

Behavior:

```text
If the problem is missing from catalog, do not recommend.
If difficulty is missing, use curated sheet difficulty.
If both are missing, exclude from difficulty-sensitive types.
Write recommendation_generation_skipped with reason = missing_catalog_or_difficulty.
```

### 25.9 Curated sheet has too few problems in a topic

Behavior:

```text
Generate from available active problems.
Do not force weak_topic_practice if no valid topic candidate exists.
Fill remaining slots with standard_practice.
```

### 25.10 Same topic dominates the candidate list

Behavior:

```text
Apply topic diversity caps.
Max 2 same-topic recommendations in top 5.
Max 3 same-topic recommendations in active set.
```

### 25.11 Same problem qualifies for multiple types

Behavior:

Use type priority:

```text
failed_submission_retry
weak_topic_practice
revision
next_in_topic
similar_problem
next_difficulty
standard_practice
```

Persist only the highest-priority valid type unless a separate lifecycle reason is needed.

### 25.12 User returns after inactivity

Behavior:

```text
Prefer revision, standard_practice, and easy/medium weak_topic_practice.
Avoid immediate hard next_difficulty recommendation.
```

### 25.13 Recommendation expires while user is viewing it

Behavior:

```text
Allow frontend action if the row is still present.
If clicked after expiry, update to clicked only if not replaced.
Next refresh may replace it.
```

### 25.14 LLM reason generation fails

Behavior:

```text
Do not block recommendation creation.
Use deterministic reason.
Keep llm_reason null.
```

### 25.15 Duplicate refresh trigger

Behavior:

```text
Use recommendation_events.idempotency_key.
If idempotency key exists, write recommendation_generation_skipped or return existing result.
```

### 25.16 User has active count >= 5

Behavior:

```text
Skip below-threshold batch refresh.
Still allow urgent live refresh for accepted/failed submissions if it updates existing recommendations.
```

---

## 26. Acceptance Criteria

The recommendation model implementation is complete when:

1. Migrations for the four recommendation tables are applied.
2. The curated DSA sheet is seeded and validated.
3. The refresh service can classify user maturity and learning states.
4. The type decider selects eligible recommendation types deterministically.
5. Candidate generators exist for all selected types.
6. Scoring produces `score`, `score_breakdown`, `reason_code`, `reason`, and `evidence`.
7. Guardrails prevent invalid, duplicate, too-hard, solved, inactive, and cooldown-blocked recommendations.
8. Diversity rules cap topic and type repetition.
9. Recommendations are persisted in `app.user_problem_recommendations`.
10. User-visible and internal events are written to `app.recommendation_events`.
11. Frontend can fetch and update lifecycle state.
12. Live accepted submissions complete matching active recommendations.
13. Live failed submissions can trigger retry or weak-topic updates.
14. Tests pass for all core user states and recommendation types.
15. Manual review confirms recommendations are explainable and aligned with the curated sheet.

---

## 27. Final Implementation Sequence

Use this order:

```text
1. Apply recommendation migrations.
2. Add indexes.
3. Seed curated DSA sheet.
4. Add recommendation service package.
5. Implement repository reads.
6. Implement user state builder.
7. Implement user maturity and learning-state classification.
8. Implement recommendation type decider.
9. Implement candidate generators.
10. Implement scoring and score breakdown.
11. Implement guardrails.
12. Implement diversity selector.
13. Implement persistence into app.user_problem_recommendations.
14. Implement recommendation event writer.
15. Implement manual refresh endpoint.
16. Implement frontend fetch and lifecycle endpoints.
17. Integrate historical sync refresh.
18. Integrate live submission refresh.
19. Add tests.
20. Roll out to internal users.
21. Tune thresholds and weights based on events.
```

This sequence keeps the implementation controlled, testable, and aligned with the selected MVP architecture.
