# Stage 1 Implementation Report: Database Foundation and Candidate Selection

## 1. Purpose

This stage creates the production foundation for the hybrid recommendation system and implements the candidate-selection engine.

The output of Stage 1 is not the full recommendation system. It creates the tables, indexes, service boundaries, repository reads, user-state snapshot builder, recommendation type decision layer, candidate generators, scoring helpers, hard guardrails, and deterministic candidate preview flow that Stage 2 and Stage 3 will use.

Stage 1 must be implemented first because every later processor depends on these contracts:

```text
catalog.dsa_sheet_problems
app.user_problem_recommendations
app.recommendation_events
app.projector_cursors

Existing user coding data
  -> User state snapshot
  -> Type decision
  -> Candidate generation
  -> Scoring-ready candidates
  -> Guardrail-filtered candidate list
```

Stage 1 must keep the design modular because future versions may change recommendation scoring, recommendation type rules, candidate-source logic, notification surfaces, or model versions without forcing table rewrites or endpoint rewrites.

---

## 2. Stage 1 Scope

### 2.1 In scope

Implement the following:

1. Database migrations for all recommendation-specific tables.
2. Recommended indexes and constraints.
3. Curated DSA sheet seed and validation framework.
4. Recommendation service package structure.
5. Repository layer for reads from existing user data and writes needed for validation or preview metadata.
6. User state snapshot builder.
7. User maturity and learning-state classifier.
8. Recommendation type decision layer.
9. Candidate generators for all selected recommendation types.
10. Scoring component helpers and score-breakdown object structure.
11. Hard guardrails for candidate eligibility.
12. Diversity pre-checks needed during candidate selection.
13. Candidate preview or dry-run endpoint for internal validation.
14. Unit tests and data-quality tests for the foundation.

### 2.2 Out of scope

Do not implement these in Stage 1:

1. Historical backfill refresh execution.
2. Active recommendation refill jobs.
3. Batch refresh orchestration.
4. Live submission event processing.
5. Projector loops.
6. User-facing notification delivery.
7. Email or WhatsApp API integration.
8. LLM-generated notification text.
9. Any new recommendation jobs table.

Stage 2 will persist recommendations through batch processing. Stage 3 will process live events and prepare future notification-channel integration.

---

## 3. System Contract Established by Stage 1

Stage 1 establishes this stable internal contract:

```text
Input:
  user_id
  platform
  sheet_name
  sheet_version
  surface
  trigger_source

Process:
  read existing user coding data
  read curated DSA sheet
  read existing active recommendations
  read recent recommendation events
  build user state
  decide eligible recommendation types
  generate candidate list
  apply hard candidate filters
  compute score inputs
  return candidate preview

Output:
  RecommendationCandidate[]
  CandidateSelectionDebug
  DataQualityValidationResult
```

The candidate-selection engine must be pure and reusable. It must not depend on whether the caller is a batch processor, live event processor, manual endpoint, or future notification workflow.

---

## 4. Selected Tables

Create and use exactly these recommendation-specific tables:

| Table | Stage 1 responsibility | Later-stage usage |
|---|---|---|
| `catalog.dsa_sheet_problems` | Create, seed, validate, query as controlled candidate pool | Used by all candidate generators |
| `app.user_problem_recommendations` | Create schema and serving indexes | Stage 2 persists batch recommendations; Stage 3 updates lifecycle from events |
| `app.recommendation_events` | Create append-only event table and idempotency index | Stage 2 writes batch events; Stage 3 consumes and writes live events |
| `app.projector_cursors` | Create cursor table | Stage 3 tracks streaming/projector progress |

Do not introduce another recommendation job table in V1. Internal processing status must be represented through `app.recommendation_events`.

---

## 5. Database Migration Requirements

### 5.1 Migration rules

Implement migrations as idempotent, backward-compatible database migrations.

Rules:

1. Use `create table if not exists` only when the migration system supports safe re-runs.
2. Use `create index if not exists` for supporting indexes.
3. Keep check constraints aligned with the selected enum values.
4. Use `jsonb not null default '{}'::jsonb` for extensibility fields.
5. Do not delete or overwrite existing recommendation history.
6. Add future enum values through explicit migrations, not ad-hoc application code.
7. Apply migrations in dependency order:
   - `catalog.dsa_sheet_problems`
   - `app.user_problem_recommendations`
   - `app.recommendation_events`
   - `app.projector_cursors`
   - indexes

### 5.2 Required schema: `catalog.dsa_sheet_problems`

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

### 5.3 Required schema: `app.user_problem_recommendations`

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

### 5.4 Required schema: `app.recommendation_events`

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

### 5.5 Required schema: `app.projector_cursors`

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

### 5.6 Required indexes

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

create index if not exists idx_projector_cursors_lookup
on app.projector_cursors(projector_name, event_source);
```

---

## 6. Curated DSA Sheet Seed Contract

### 6.1 Required seed fields

Every active seeded row must include:

| Field | Required | Notes |
|---|---:|---|
| `platform` | Yes | Default `leetcode` |
| `problem_slug` | Yes | Must exist in `catalog.coding_problems.slug` |
| `sheet_name` | Yes | Default `default_dsa_sheet` |
| `sheet_version` | Yes | Default `v1` |
| `topic_slug` | Yes | Stable key, e.g. `dynamic-programming` |
| `topic_name` | Yes | Display name |
| `problem_order` | Yes | Integer order within topic |
| `problem_title` | Yes | Display fallback |
| `difficulty` | Yes for active rows | `easy`, `medium`, or `hard` |
| `importance_score` | Yes | 0.0000 to 1.0000 |
| `interview_frequency_score` | Yes | 0.0000 to 1.0000 |
| `foundation_score` | Yes | 0.0000 to 1.0000 |
| `revision_value_score` | Yes | 0.0000 to 1.0000 |
| `estimated_minutes` | Recommended | Used later for daily plans |
| `source_name` | Recommended | Internal sheet, Striver, NeetCode, Blind 75, etc. |
| `metadata` | Yes | Defaults to empty JSON |
| `is_active` | Yes | Only active rows can be recommended |

### 6.2 Metadata shape

Use `metadata` for future-proof candidate selection. Do not add columns for every new signal unless it becomes a stable high-volume query requirement.

```json
{
  "patterns": ["hash-map", "prefix-sum"],
  "prerequisites": ["arrays", "hashing"],
  "learning_stage": "foundation",
  "is_beginner_friendly": true,
  "is_revision_friendly": true,
  "similar_concepts": ["lookup-table", "frequency-map"],
  "notification_copy_tags": ["confidence_building", "short_practice"],
  "notes": "Canonical hash map problem for array lookups."
}
```

### 6.3 Seed validation queries

Implement a validator that returns structured errors. Fail deployment or disable recommendation generation when blocking errors exist.

Required validations:

| Validation | Severity | Rule |
|---|---|---|
| Missing catalog row | Blocker | Every active `platform + problem_slug` must exist in `catalog.coding_problems` |
| Missing difficulty | Blocker | Active row must have difficulty or catalog fallback difficulty |
| Duplicate problem in sheet version | Blocker | Enforced by unique constraint |
| Duplicate topic order | Blocker | Enforced by unique constraint |
| Score outside range | Blocker | All score fields must be between 0 and 1 |
| Empty topic | Blocker | `topic_slug` and `topic_name` required |
| Too few beginner rows | Warning | Beginner topics should have at least 3 active beginner-friendly rows |
| No revision-friendly row in high-value topic | Warning | At least one high-value row should be revision-friendly |
| Missing estimated minutes | Warning | Useful for daily practice packaging |

---

## 7. Service Package Layout

Create the recommendation service package:

```text
api/services/recommendations/
  __init__.py
  constants.py
  models.py
  repository.py
  sheet_validator.py
  user_state.py
  classifiers.py
  type_decider.py
  candidate_generator.py
  scoring_components.py
  scorer.py
  guardrails.py
  diversity.py
  candidate_service.py
  event_writer.py
  routes.py
```

### 7.1 File responsibilities

| File | Responsibility |
|---|---|
| `constants.py` | Enum-like constants for statuses, types, surfaces, scoring versions, expiry defaults, cooldown defaults |
| `models.py` | Pydantic/data classes for state, topic stats, candidates, scores, validation results |
| `repository.py` | SQL reads for existing user data, curated sheet, active recommendations, events |
| `sheet_validator.py` | DSA sheet seed validation |
| `user_state.py` | Builds `UserRecommendationState` |
| `classifiers.py` | Maturity, activity, sync, and learning-state classification |
| `type_decider.py` | Selects eligible recommendation types |
| `candidate_generator.py` | Generates typed candidates |
| `scoring_components.py` | Pure helpers for component scores |
| `scorer.py` | Builds `score_breakdown` and final score for candidates |
| `guardrails.py` | Hard filters and candidate-level exclusion reasons |
| `diversity.py` | Topic/type/difficulty caps used before final selection |
| `candidate_service.py` | Orchestrates candidate preview flow |
| `event_writer.py` | Shared writer used later by Stage 2 and Stage 3 |
| `routes.py` | Internal validation and candidate preview routes |

Keep the service package independent from web framework concerns except `routes.py`.

---

## 8. Data Models

### 8.1 `UserRecommendationState`

```python
class UserRecommendationState:
    user_id: str
    platform: str
    sheet_name: str
    sheet_version: str

    profile_connected: bool
    sync_enabled: bool
    sync_phase: str | None
    sync_stale: bool
    sync_error: str | None

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

    active_recommendation_slugs: set[str]
    recently_shown_slugs: set[str]
    recently_dismissed_slugs: set[str]
    cooldown_blocked_slugs: set[str]

    active_count: int

    topic_stats: dict[str, TopicStats]
    difficulty_stats: dict[str, DifficultyStats]
    problem_stats: dict[str, ProblemStats]

    maturity: str
    activity_states: set[str]
    learning_states: set[str]

    now: datetime
```

### 8.2 `TopicStats`

```python
class TopicStats:
    topic_slug: str
    topic_name: str
    curated_count: int
    attempted_problem_count: int
    solved_count: int
    failed_problem_count: int
    failed_attempt_count: int
    unsolved_attempted_count: int
    attempt_count: int
    solve_rate: float
    failure_ratio: float
    last_ac_at: datetime | None
    last_attempt_at: datetime | None
    topic_weakness_score: float
    topic_stale: bool
```

### 8.3 `ProblemStats`

```python
class ProblemStats:
    platform: str
    slug: str
    status: str | None
    attempt_count: int
    ac_count: int
    failed_attempt_count: int
    failure_ratio: float
    first_attempt_at: datetime | None
    latest_attempt_at: datetime | None
    first_ac_at: datetime | None
    latest_ac_at: datetime | None
    is_solved: bool
    is_attempted: bool
    is_failed_unsolved: bool
    days_since_latest_attempt: int | None
    days_since_latest_ac: int | None
    near_solve_score: float
```

### 8.4 `RecommendationCandidate`

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
    surface: str
    reason_code: str
    reason: str

    trigger_source: str
    scoring_version: str

    raw_scores: dict
    score_breakdown: dict
    evidence: dict
    metadata: dict

    priority: int
    score: float
    rank: int | None
    expires_at: datetime | None
    cooldown_until: datetime | None
```

### 8.5 `CandidatePreviewResult`

```python
class CandidatePreviewResult:
    user_id: str
    platform: str
    sheet_name: str
    sheet_version: str
    trigger_source: str
    surface: str
    active_count: int
    maturity: str
    activity_states: list[str]
    learning_states: list[str]
    eligible_types: list[str]
    generated_count: int
    filtered_count: int
    scored_count: int
    candidates: list[RecommendationCandidate]
    guardrail_exclusions: list[dict]
    warnings: list[str]
```

---

## 9. Repository Read Contracts

Implement repository methods that return normalized domain objects. Keep raw SQL contained in `repository.py`.

### 9.1 Required reads

| Method | Purpose |
|---|---|
| `get_user_profile(user_id, platform)` | Reads `user_data.user_coding_profiles` |
| `get_sync_state(user_id, platform)` | Reads `system.sync_state` |
| `get_user_problem_aggregates(user_id, platform)` | Reads `user_data.user_coding_problems` joined with `catalog.coding_problems` |
| `get_recent_submissions(user_id, platform, days=30)` | Reads `user_data.user_coding_submissions` |
| `get_curated_pool(sheet_name, sheet_version)` | Reads active `catalog.dsa_sheet_problems` joined with `catalog.coding_problems` |
| `get_active_recommendations(user_id, platform)` | Reads pending/active/shown/clicked/started rows |
| `get_recent_recommendation_events(user_id, platform, days=90)` | Reads event history used for freshness/cooldown |
| `get_recently_dismissed_recommendations(user_id, platform)` | Reads cooldown-blocked dismissed/skipped/replaced rows |

### 9.2 Required SQL patterns

Active curated pool:

```sql
select
  dsp.*,
  cp.title as catalog_title,
  cp.difficulty as catalog_difficulty,
  cp.topic_tags,
  cp.similar_slugs,
  cp.ac_rate,
  cp.sync_status,
  cp.raw_meta
from catalog.dsa_sheet_problems dsp
join catalog.coding_problems cp
  on cp.platform = dsp.platform
 and cp.slug = dsp.problem_slug
where dsp.sheet_name = $1
  and dsp.sheet_version = $2
  and dsp.is_active = true;
```

Active recommendations:

```sql
select *
from app.user_problem_recommendations
where user_id = $1
  and platform = $2
  and status in ('pending', 'active', 'shown', 'clicked', 'started')
  and (expires_at is null or expires_at > now());
```

Cooldown-blocked recommendations:

```sql
select *
from app.user_problem_recommendations
where user_id = $1
  and platform = $2
  and status in ('skipped', 'dismissed', 'replaced')
  and cooldown_until is not null
  and cooldown_until > now();
```

User aggregates:

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

Recent submissions:

```sql
select *
from user_data.user_coding_submissions
where user_id = $1
  and platform = $2
  and ts >= now() - interval '30 days'
order by ts desc;
```

---

## 10. User State Builder

### 10.1 Build sequence

The state builder must run in this order:

```text
1. Read user profile.
2. Read sync state.
3. Read curated pool.
4. Read user aggregate problems.
5. Read recent submissions.
6. Read active recommendations.
7. Read recent events and cooldown rows.
8. Derive problem-level stats.
9. Derive topic-level stats.
10. Derive difficulty stats.
11. Classify user maturity.
12. Classify activity states.
13. Classify learning states.
14. Return immutable state snapshot.
```

### 10.2 Problem-level formulas

| Signal | Formula |
|---|---|
| `is_solved` | `ac_count > 0 or status = 'ac'` |
| `is_attempted` | `attempt_count > 0` |
| `failed_attempt_count` | `wa_count + tle_count + mle_count + re_count + ce_count + other_count` |
| `failure_ratio` | `failed_attempt_count / max(attempt_count, 1)` |
| `is_failed_unsolved` | `attempt_count > 0 and ac_count = 0` |
| `days_since_latest_attempt` | `now - latest_attempt_at` |
| `days_since_latest_ac` | `now - latest_ac_at` |
| `is_revision_due` | `ac_count > 0 and latest_ac_at older than revision threshold` |

### 10.3 Submission-level formulas

| Signal | Formula |
|---|---|
| `is_accepted` | `verdict = 'AC'` |
| `is_failed` | `verdict in ('WA', 'TLE', 'MLE', 'RE', 'CE', 'OTHER')` |
| `near_solve_score` | `total_correct / total_testcases` if both exist, else `0.0` |
| `recent_failure` | Failed submission within 7 days |
| `recent_success` | Accepted submission within 7 days |

### 10.4 Topic-level formulas

Join aggregates to curated sheet rows by:

```text
platform + slug/problem_slug
```

For each topic:

| Signal | Formula |
|---|---|
| `curated_count` | Count active curated problems in topic |
| `attempted_problem_count` | Count curated topic problems with attempts |
| `solved_count` | Count curated topic problems with AC |
| `failed_problem_count` | Count curated topic problems with failed attempts |
| `failed_attempt_count` | Sum failed attempts in topic |
| `unsolved_attempted_count` | Count attempted but unsolved problems in topic |
| `attempt_count` | Sum total attempts in topic |
| `solve_rate` | `solved_count / max(curated_count, 1)` |
| `failure_ratio` | `failed_attempt_count / max(attempt_count, 1)` |
| `topic_stale` | `last_ac_at is null or last_ac_at < now - 30 days` |

### 10.5 Topic weakness score

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
repeated_failure_score = min(1.0, failed_problem_count / 3)
unsolved_attempted_topic_score = unsolved_attempted_count / max(attempted_problem_count, 1)
stale_topic_score = 1.0 if topic_stale else 0.0
```

---

## 11. User Classification Rules

### 11.1 Maturity classification

| Maturity | Condition |
|---|---|
| `new_user` | `total_solved = 0 and total_attempted = 0` |
| `starter` | `total_solved < 5` |
| `beginner` | `total_solved >= 5 and total_solved < 25` |
| `regular` | `total_solved >= 25 and total_solved < 100` |
| `advanced` | `total_solved >= 100` |

### 11.2 Activity state classification

| State | Condition |
|---|---|
| `inactive_user` | No attempt in last 14 days |
| `recently_active_user` | At least 1 attempt in last 7 days |
| `high_activity_user` | 10 or more attempts in last 7 days |
| `sync_stale_user` | Sync is old, failed, paused, auth-required, or not progressing |

### 11.3 Learning state classification

| State | Condition |
|---|---|
| `recent_failed_problem` | Any unsolved problem failed in last 7 days |
| `weak_topic_detected` | Any topic weakness score >= 0.65 |
| `revision_due` | Any solved curated problem is older than revision threshold |
| `topic_progression_ready` | A next ordered unsolved curated problem exists after solved/attempted topic progress |
| `similar_reinforcement_ready` | Recent confident solve exists |
| `difficulty_progression_ready` | Recent success at current difficulty with acceptable failure rate |
| `low_active_recommendations` | Active count < 5 |

---

## 12. Recommendation Type Decision Layer

### 12.1 Type priority

Use this fixed priority order:

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

The priority order resolves conflicts when the same problem qualifies for multiple types.

### 12.2 Type caps

| Type | Max active rows |
|---|---:|
| `failed_submission_retry` | 2 |
| `weak_topic_practice` | 3 |
| `revision` | 2 |
| `next_in_topic` | 3 |
| `similar_problem` | 2 |
| `next_difficulty` | 1 |
| `standard_practice` | Fill remaining slots |
| `daily_practice` | Packaging surface only |

### 12.3 Eligibility rules

| User state | Eligible type | Rule |
|---|---|---|
| `new_user` | `standard_practice` | Beginner curated problems exist |
| `new_user` | `next_in_topic` | First problem in beginner topic exists |
| `starter` | `standard_practice` | Always eligible |
| `starter` | `next_in_topic` | User attempted or solved at least one curated problem |
| `starter` | `failed_submission_retry` | Failed unsolved problem has 2 or more attempts |
| `beginner` | `weak_topic_practice` | Topic weakness score >= 0.65 |
| `beginner` | `failed_submission_retry` | Recent unsolved failed problem exists |
| `regular` | all selected types | Apply individual rules |
| `advanced` | all selected types | Apply individual rules with stricter difficulty readiness |
| `inactive_user` | `revision` | Old solved curated problems exist |
| `inactive_user` | `standard_practice` | Safe re-entry fallback |
| `sync_stale_user` | `standard_practice` | Avoid strong recent-signal personalization |
| `recent_failed_problem` | `failed_submission_retry` | Failed problem is unsolved and curated |
| `weak_topic_detected` | `weak_topic_practice` | Weak topic candidates exist |
| `revision_due` | `revision` | Due solved curated problem exists |
| `topic_progression_ready` | `next_in_topic` | Next ordered curated problem exists |
| `similar_reinforcement_ready` | `similar_problem` | Confident recent solve and similar curated candidate exists |
| `difficulty_progression_ready` | `next_difficulty` | Readiness conditions pass |

---

## 13. Candidate Generators

Each generator must return candidates only. It must not persist database rows. Persistence starts in Stage 2.

### 13.1 Shared candidate exclusions

Every generator must avoid candidates that are:

1. Outside `catalog.dsa_sheet_problems`.
2. Inactive in the curated sheet.
3. Missing from `catalog.coding_problems`.
4. Already active for the same user.
5. Cooldown-blocked from skip, dismissal, or replacement.
6. Missing required fields.

### 13.2 `standard_practice`

Use when no stronger signal exists, user is new/starter, sync is stale, or active recommendations are below threshold.

Rules:

1. Candidate source is active curated sheet.
2. Problem must be unsolved.
3. Problem must not be active or cooldown-blocked.
4. Prefer high `importance_score` and `foundation_score`.
5. For new users, prefer `difficulty = 'easy'` and `metadata.is_beginner_friendly = true`.

Reason codes:

```text
important_unsolved_problem
curated_foundation_problem
beginner_friendly_start
safe_reentry_practice
```

### 13.3 `weak_topic_practice`

Use when topic weakness exists.

Rules:

1. Select unsolved active curated problems from weak topics.
2. Difficulty must be allowed for user maturity.
3. If user failed hard problems repeatedly, choose easier or foundation problems first.
4. Prefer high `foundation_score` when failure ratio is high.
5. Use `topic_weakness_score` in evidence.

Reason codes:

```text
repeated_topic_failures
low_topic_solve_rate
topic_needs_strengthening
easier_topic_rebuild
```

### 13.4 `failed_submission_retry`

Use when the user has an unsolved failed problem that is in the active curated sheet.

Rules:

1. Problem has `failed_attempt_count >= 2`.
2. Problem has `ac_count = 0`.
3. Latest failed attempt should be recent enough.
4. If failed attempts are more than 5 and near-solve score is below 0.50, do not recommend retry. Let `weak_topic_practice` choose an easier related problem.
5. If near-solve score is 0.70 or higher, boost retry strongly.

Reason codes:

```text
recent_failed_problem
multiple_failed_attempts
almost_solved_retry
retry_unsolved_problem
```

### 13.5 `revision`

Use for solved curated problems that are old enough to revise.

Rules:

1. Problem must be solved.
2. Problem must exist in active curated sheet.
3. Latest AC must be older than revision threshold.
4. Prefer high `revision_value_score` and high `importance_score`.
5. Avoid over-serving revision candidates.

Reason codes:

```text
revision_due
important_problem_stale
topic_not_practiced_recently
high_value_revision
```

### 13.6 `next_in_topic`

Use when the user has progress in a curated topic and the next unsolved ordered problem exists.

Rules:

1. For each topic, find highest solved or attempted `problem_order`.
2. Select the next unsolved active problem.
3. Do not jump over large gaps in the curated path.
4. If the previous problem had excessive failures, choose a foundation problem instead.

Reason codes:

```text
next_problem_in_curated_order
continue_topic_progression
complete_topic_sequence
```

### 13.7 `similar_problem`

Use after a confident recent solve or useful failed pattern.

Candidate source priority:

1. `catalog.coding_problems.similar_slugs`.
2. Matching `metadata.patterns` from curated sheet.
3. Same `topic_slug` and nearby difficulty.

Rules:

1. Candidate must be unsolved.
2. Candidate must be active in curated sheet.
3. Avoid duplicate active recommendations.
4. Avoid too many same-topic recommendations.

Reason codes:

```text
reinforce_recent_solve
same_pattern_practice
similar_problem_after_success
```

### 13.8 `next_difficulty`

Use when the user is ready to move one difficulty step up.

Readiness rules:

| Maturity | Readiness condition |
|---|---|
| `beginner` | 5 or more easy solves and recent failure ratio < 0.40 |
| `regular` | 3 or more recent solves at current difficulty and recent failure ratio < 0.50 |
| `advanced` | 3 or more recent medium solves or strong topic solve rate |

Rules:

1. Only allow `easy -> medium` or `medium -> hard`.
2. Never allow `easy -> hard`.
3. Do not use for inactive users or users with many recent failures.
4. Prefer same topic where success was demonstrated.

Reason codes:

```text
ready_for_medium
ready_for_hard
difficulty_progression
stretch_problem_ready
```

### 13.9 `daily_practice`

Do not generate primary candidates as `daily_practice` in Stage 1.

Treat `daily_practice` as a packaging surface that Stage 2 can use when selecting a balanced set for the `today` surface.

---

## 14. Score Component Helpers

Stage 1 must implement scoring helpers even though Stage 2 will perform the first production persistence. This keeps candidate selection deterministic and testable.

### 14.1 Formula

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

### 14.2 Difficulty match score

| Maturity | Easy | Medium | Hard |
|---|---:|---:|---:|
| `new_user` | 1.00 | 0.20 | 0.00 |
| `starter` | 1.00 | 0.40 | 0.00 |
| `beginner` | 0.75 | 1.00 | 0.20 |
| `regular` | 0.40 | 1.00 | 0.60 |
| `advanced` | 0.20 | 0.85 | 0.90 |

A score of 0.00 is a hard block.

### 14.3 Failed attempt score

```text
failed_attempt_score = min(1.0,
  0.35 * failed_attempt_count_score
+ 0.25 * recent_failure_score
+ 0.25 * near_solve_score
+ 0.15 * unsolved_boost
)
```

### 14.4 Revision due score

```text
revision_due_score = time_due_score * max(revision_value_score, importance_score)
```

| Time since latest AC | Time due score |
|---|---:|
| 0-7 days | 0.00 |
| 8-21 days | 0.25 |
| 22-45 days | 0.50 |
| 46-90 days | 0.75 |
| 90+ days | 1.00 |

### 14.5 Stored score breakdown shape

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
  "raw_scores": {},
  "boosts": {},
  "penalties": {},
  "final_score": 0.0
}
```

---

## 15. Hard Guardrails

Apply these after candidate generation and before final selection:

| Guardrail | Rule |
|---|---|
| Active sheet only | Exclude if `is_active = false` |
| Curated pool only | Exclude if not present in `catalog.dsa_sheet_problems` |
| Missing catalog | Exclude if missing from `catalog.coding_problems` |
| Duplicate active row | Exclude if user already has active row for same problem |
| Solved problem | Exclude solved problems unless type is `revision` |
| Cooldown | Exclude if skip/dismiss/replaced cooldown is active |
| Difficulty | Exclude if difficulty is blocked for user maturity |
| Type cap | Exclude or down-rank if active cap reached |
| Missing required fields | Exclude if slug, topic, type, reason, or score source is missing |
| Same problem multi-type conflict | Keep only highest-priority valid type |

### 15.1 Difficulty guardrails

| Maturity | Allowed difficulty |
|---|---|
| `new_user` | Easy only |
| `starter` | Easy, selected medium only |
| `beginner` | Easy and medium |
| `regular` | Easy, medium, selected hard |
| `advanced` | Medium and hard, selected easy only for revision/foundation |

### 15.2 Sync guardrail

If sync is stale or failing:

1. Reduce confidence in live/recent failure signals.
2. Prefer `standard_practice`, `revision`, and safe `next_in_topic`.
3. Store `metadata.sync_freshness = 'stale'` on candidates.

---

## 16. Diversity Pre-Selection Rules

Implement reusable diversity helpers in Stage 1. Stage 2 will use these helpers for final persistence.

| Rule | Limit |
|---|---:|
| Same topic in top 5 | Max 2 |
| Same topic in full active set | Max 3 |
| `failed_submission_retry` active at once | Max 2 |
| `revision` active at once | Max 2 |
| `next_difficulty` active at once | Max 1 |
| Same difficulty in top 5 | Avoid all 5 being same difficulty |

---

## 17. Candidate Preview Route

Create an internal route for testing candidate selection.

```text
POST /api/internal/recommendations/candidates/preview
```

Request:

```json
{
  "user_id": "uuid",
  "platform": "leetcode",
  "sheet_name": "default_dsa_sheet",
  "sheet_version": "v1",
  "surface": "dashboard",
  "trigger_source": "manual_refresh",
  "limit": 20
}
```

Response:

```json
{
  "user_id": "uuid",
  "platform": "leetcode",
  "maturity": "beginner",
  "activity_states": ["recently_active_user"],
  "learning_states": ["weak_topic_detected"],
  "active_count": 4,
  "eligible_types": ["weak_topic_practice", "next_in_topic", "standard_practice"],
  "generated_count": 42,
  "filtered_count": 11,
  "scored_count": 31,
  "candidates": [],
  "guardrail_exclusions": []
}
```

Rules:

1. Route must not persist recommendations.
2. Route must not write lifecycle events.
3. Route may be protected behind admin/internal auth.
4. Route must not expose submitted code, raw testcases, expected output, actual output, runtime errors, or compile errors.

---

## 18. Production-Grade Rules for Stage 1

### 18.1 Reliability

1. All repository methods must use parameterized SQL.
2. Candidate generation must be deterministic for the same input snapshot.
3. Date calculations must use database time or a supplied `now` value consistently.
4. Missing optional metadata must not crash candidate generation.
5. Blocking validation errors must disable generation rather than generating unsafe recommendations.

### 18.2 Security and privacy

1. Do not expose submitted code through candidate preview.
2. Do not include expected output, actual output, last testcase, compile error, or runtime error in user-facing reasons.
3. Evidence may include counts and timestamps, not sensitive raw execution payloads.
4. Keep user-specific state scoped to the authenticated user or internal/admin context.

### 18.3 Modularity

1. Keep scoring versioned through `scoring_version`.
2. Keep recommendation type rules in `type_decider.py`, not scattered through routes.
3. Keep candidate generators isolated by type.
4. Keep channel-specific notification logic out of candidate selection.
5. Keep LLM logic out of ranking and candidate generation.
6. Design candidate evidence so future LLM notification generation can consume it safely.

### 18.4 Observability

Stage 1 preview logs should include:

```text
user_id hash or internal id
platform
sheet_name
sheet_version
active_count
maturity
eligible_types
generated_count
filtered_count
scored_count
guardrail exclusion counts
runtime_ms
```

Do not log submitted code or raw error payloads.

---

## 19. Stage 1 Test Plan

### 19.1 Migration tests

Verify:

1. All four tables exist.
2. All check constraints are present.
3. All unique constraints are present.
4. All recommended indexes exist.
5. Foreign keys reject invalid curated or recommended problems.

### 19.2 Sheet validation tests

Test:

1. Missing catalog problem returns blocker.
2. Missing difficulty returns blocker.
3. Duplicate order fails.
4. Duplicate problem in sheet fails.
5. Out-of-range score returns blocker.
6. Valid seed returns pass.

### 19.3 User state tests

Test:

1. New user classification.
2. Starter classification.
3. Beginner classification.
4. Regular classification.
5. Advanced classification.
6. Sync stale classification.
7. Topic weakness calculation.
8. Failed unsolved slug detection.
9. Active count calculation.
10. Cooldown-blocked slug detection.

### 19.4 Type decision tests

Test:

1. New user returns `standard_practice` and initial `next_in_topic`.
2. Only failed submissions returns `failed_submission_retry`, `weak_topic_practice`, and `standard_practice`.
3. Revision due returns `revision`.
4. Confident recent solve returns `similar_problem`.
5. Difficulty readiness returns `next_difficulty`.
6. Stale sync downshifts toward `standard_practice`.

### 19.5 Candidate generator tests

Test every recommendation type with positive and negative cases.

### 19.6 Guardrail tests

Test:

1. Solved non-revision is blocked.
2. Active duplicate is blocked.
3. Cooldown active is blocked.
4. Hard problem for new user is blocked.
5. Missing catalog row is blocked.
6. Same problem multi-type conflict keeps highest-priority type.

---

## 20. Stage 1 Acceptance Criteria

Stage 1 is complete when:

1. Migrations create the four recommendation tables and indexes.
2. Curated sheet seed validation exists and blocks unsafe generation.
3. Recommendation service package exists with clear file responsibilities.
4. Repository reads are implemented for all required sources.
5. `UserRecommendationState` is built correctly.
6. User maturity, activity, and learning states are classified correctly.
7. Eligible recommendation types are selected deterministically.
8. Candidate generators exist for all selected recommendation types.
9. Guardrails filter invalid candidates.
10. Score components and score-breakdown shape are available.
11. Candidate preview route returns deterministic debug output.
12. Tests pass for migrations, validation, state, type decision, candidate generation, and guardrails.
13. No candidate-generation code depends on batch or live processing execution.
14. No LLM code affects candidate selection.
15. No email or WhatsApp channel code is added to candidate selection.

---

## 21. Handoff to Stage 2

Stage 2 must consume Stage 1 through this function-level contract:

```python
async def preview_recommendation_candidates(
    user_id: str,
    platform: str = "leetcode",
    sheet_name: str = "default_dsa_sheet",
    sheet_version: str = "v1",
    surface: str = "dashboard",
    trigger_source: str = "manual_refresh",
    limit: int = 20,
    now: datetime | None = None,
) -> CandidatePreviewResult:
    ...
```

Stage 2 must not duplicate candidate logic. It must call the Stage 1 candidate service, select final recommendations, persist them to `app.user_problem_recommendations`, and write internal/user-visible events to `app.recommendation_events`.
