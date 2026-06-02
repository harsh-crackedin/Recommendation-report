# MVP Recommendation System Design Report

## 1. Purpose of This Report

This report defines the MVP architecture for a recommendation system that uses Supabase Postgres as the database foundation. The system is designed around three core ideas:

1. Store important user behavior as append-only events.
2. Build derived progress tables from those events.
3. Serve recommendations from fast, query-friendly read models.

The report is intentionally focused on design rather than implementation. It does not prescribe exact scoring logic, exact recommendation formulas, or SQL migrations. Instead, it defines what each database layer owns, how the layers relate to each other, which existing tables should be reused, which missing tables should be added, and how the recommendation system should treat source-of-truth data versus rebuildable derived data.

The MVP should remain Supabase/Postgres-only. No external event broker, feature store, cache, or streaming platform is required for the current phase.

---

## 2. High-Level Architecture

The recommended MVP architecture is:

```text
LeetCode submissions / user learning activity
        |
        v
Append-only event log tables
        |
        v
Projector / progress builder
        |
        v
Derived progress and recommendation read models
        |
        v
Recommendation API / UI
```

The key architectural pattern is:

```text
Event log -> Projection -> Read model -> Recommendation serving
```

This gives the product the main benefits of event-based design without overengineering the MVP:

- User behavior history is preserved.
- Current progress can be recomputed if scoring logic changes.
- Recommendation reads stay fast because the API reads from precomputed tables.
- The architecture can evolve later without changing the core data ownership model.

---

## 3. MVP Design Principles

### 3.1 Supabase Postgres is the only required database layer

All MVP data should live in Supabase Postgres. The system should not require any external broker or external state system. Events can be stored as normal append-only Postgres rows, and read models can be normal Postgres tables.

### 3.2 Events represent facts that happened

Event tables should describe what happened, not what the current recommendation score is.

Examples:

- A user submitted a LeetCode problem.
- A user solved a problem.
- A user struggled with a topic.
- A user revised a topic.
- A recommendation was shown.
- A recommendation was accepted, skipped, or abandoned.

Events should be append-only. They should not be updated to represent current state.

### 3.3 Read models represent current state

Read-model tables should answer application questions quickly.

Examples:

- What is the user's current progress on this problem?
- Which topics is the user weak on?
- Which topics are due for revision?
- Which recommendations should be shown right now?

Read models are mutable and rebuildable. They are not the historical source of truth.

### 3.4 Static catalog data is separate from user behavior

The system should keep the coding curriculum separate from user activity.

Static catalog data includes:

- Topics such as Array, Dynamic Programming, BFS, DFS, Graphs, Trees, etc.
- Coding problems.
- Difficulty.
- Topic-to-problem mapping.
- Platform metadata such as LeetCode slug and platform identifier.

User behavior data includes:

- Submissions.
- Practice events.
- Revision events.
- Recommendation interactions.
- Derived progress.

### 3.5 Recommendation logic must be replaceable

The exact scoring method should not be embedded into source-of-truth tables. The scoring logic can change later, and derived tables can be rebuilt from events.

For MVP, the recommendation logic can be simple:

- Prefer topics where the user has low coverage.
- Prefer topics with repeated failed attempts.
- Prefer topics whose `next_revision_at` is due.
- Prefer unsolved or stale questions under weak topics.
- Avoid repeatedly showing recently skipped recommendations.

The architecture should support this without requiring the logic to be final.

---

## 4. Layered Data Model

The design is divided into five layers.

```text
Layer 1: Identity and platform profile layer
Layer 2: Static catalog layer
Layer 3: Event log layer
Layer 4: Derived progress layer
Layer 5: Recommendation serving layer
```

Each layer has a different responsibility and a different ownership model.

---

## 5. Layer 1: Identity and Platform Profile Layer

### Purpose

This layer connects application users to coding-platform activity.

It answers:

- Who is the user?
- Which LeetCode or coding profile belongs to the user?
- Which user owns the imported submissions?

### Existing tables to reuse

#### `public.users`

| Property | Design Decision |
|---|---|
| Layer | Identity |
| Table lifecycle | Mutable source table |
| Reuse status | Reuse as-is |
| Purpose | Core user identity table |
| Key fields from discovery | `id`, `email`, `tier` |
| Primary role in recommendation system | Anchor every user-specific event and read model to `user_id` |

Design notes:

- `public.users.id` should be the root foreign key for user-owned recommendation data.
- All user-specific event and read-model tables should reference this user identity.
- This table should not contain recommendation state directly.

#### `user_data.user_coding_profiles`

| Property | Design Decision |
|---|---|
| Layer | Identity / platform connection |
| Table lifecycle | Mutable source table |
| Reuse status | Reuse as-is |
| Purpose | Tracks coding-platform connection, such as LeetCode profile ownership |
| Primary role in recommendation system | Links a user to imported coding activity |

Design notes:

- This table should remain responsible for platform identity and sync metadata.
- It should not become a progress or recommendation table.
- It can be used by ingestion flows to decide which submissions belong to which application user.

---

## 6. Layer 2: Static Catalog Layer

### Purpose

The static catalog layer defines what can be recommended.

It answers:

- What topics exist?
- Which questions exist?
- Which topics does each question belong to?
- What difficulty or metadata does each question have?

This layer is the curriculum/catalog source of truth.

---

### 6.1 Existing table: `catalog.coding_problems`

| Property | Design Decision |
|---|---|
| Layer | Static catalog |
| Table lifecycle | Mutable source table |
| Reuse status | Reuse |
| Purpose | Stores LeetCode/coding problems |
| Key fields from discovery | `platform`, `slug`, `difficulty`, `topic_tags`, `ac_rate` |
| Current topic representation | `topic_tags` as `TEXT[]` |
| Recommendation role | Provides candidate problems for recommendations |

Design notes:

- This table should remain the core problem catalog.
- The composite identity of a problem should be treated as `(platform, slug)`.
- `topic_tags` is useful for MVP discovery, but it is not ideal as the only long-term topic relationship because the topic progress table uses `topic_id`.
- The recommendation system should prefer a normalized topic mapping once available.

---

### 6.2 Missing table: `catalog.topics`

The discovery report indicates that a dedicated Postgres `catalog.topics` table is missing, while `app.pt_user_topic_progress` references `topic_id`. The MVP should add a proper topic catalog so topic-level progress can be linked to a stable taxonomy.

| Property | Design Decision |
|---|---|
| Layer | Static catalog |
| Table lifecycle | Mutable source table |
| Reuse status | Create if missing |
| Purpose | Defines coding topics and optional topic hierarchy |
| Source of truth? | Yes, for topic taxonomy |
| Mutable? | Yes, but changes should be controlled |

Recommended design fields:

| Field | Purpose |
|---|---|
| `id` | Stable topic identifier used by progress and event tables |
| `slug` | Stable human-readable identifier such as `array`, `dynamic-programming`, `bfs` |
| `name` | Display name such as `Array`, `Dynamic Programming`, `BFS` |
| `parent_id` | Optional self-reference for grouping subtopics under broader topics |
| `description` | Optional explanation of the topic |
| `is_active` | Allows retiring topics without deleting historical links |
| `created_at` | Creation timestamp |
| `updated_at` | Last update timestamp |

Design notes:

- Topic IDs should be stable because user progress and learning events will reference them.
- Topic slugs should be unique.
- Topics should not be deleted if they have historical events. Deactivate instead.
- Parent-child topic hierarchy is optional for MVP but useful for grouping topics such as Graphs -> BFS/DFS.

---

### 6.3 Recommended missing table: `catalog.problem_topics`

Although `catalog.coding_problems.topic_tags` can work for a quick MVP, a normalized mapping table is recommended because topic-level recommendation depends on reliable joins between problems and topics.

| Property | Design Decision |
|---|---|
| Layer | Static catalog |
| Table lifecycle | Mutable source table |
| Reuse status | Create if no equivalent exists |
| Purpose | Maps coding problems to one or more topics |
| Source of truth? | Yes, for problem-topic relationships |
| Mutable? | Yes, controlled by catalog updates |

Recommended design fields:

| Field | Purpose |
|---|---|
| `platform` | Matches `catalog.coding_problems.platform` |
| `slug` | Matches `catalog.coding_problems.slug` |
| `topic_id` | References `catalog.topics.id` |
| `weight` | Optional relevance weight for cases where a problem belongs to multiple topics |
| `is_primary` | Optional flag for the most important topic of a problem |
| `created_at` | Mapping creation timestamp |

Recommended primary key:

```text
(platform, slug, topic_id)
```

Design notes:

- This table creates the bridge between problem-level events and topic-level progress.
- A single problem may belong to multiple topics.
- `weight` can help later when deciding how much solving a problem should affect each topic.
- For MVP, `weight` can default to `1.0`.

---

## 7. Layer 3: Event Log Layer

### Purpose

The event log layer stores user behavior as append-only facts.

It answers:

- What did the user do?
- When did the user do it?
- Which problem, topic, or recommendation was involved?
- What raw facts should be replayable later?

This layer is the behavioral source of truth.

---

### 7.1 Existing table: `user_data.user_coding_submissions`

| Property | Design Decision |
|---|---|
| Layer | Event log |
| Table lifecycle | Append-only source table |
| Reuse status | Reuse |
| Purpose | Stores LeetCode/coding submissions |
| Key fields from discovery | `platform`, `submission_id`, `user_id`, `slug`, `verdict`, `ts` |
| Recommendation role | Primary source of coding behavior |

Design notes:

- This table should be treated as the canonical log of imported coding submissions.
- It should not be overwritten to store aggregate progress.
- Duplicate submissions should be prevented using submission identity.
- This table powers problem-level and topic-level progress projections.

Important relationships:

```text
user_data.user_coding_submissions.user_id -> public.users.id
user_data.user_coding_submissions.(platform, slug) -> catalog.coding_problems.(platform, slug)
catalog.coding_problems.(platform, slug) -> catalog.problem_topics.(platform, slug)
catalog.problem_topics.topic_id -> catalog.topics.id
```

Event examples:

| Scenario | Event interpretation |
|---|---|
| User submits and gets AC | Problem solved event source |
| User submits and gets WA/TLE/etc. | Struggle/attempt event source |
| User solves old topic again | Revision/practice signal |
| Multiple failed attempts before AC | Difficulty/weakness signal |

---

### 7.2 Existing table: `app.pt_learning_events`

| Property | Design Decision |
|---|---|
| Layer | Event log |
| Table lifecycle | Append-only source table |
| Reuse status | Reuse |
| Purpose | Stores learning/revision interactions |
| Key fields from discovery | `user_id`, `event_type`, `topic_id`, `readiness_delta`, `created_at` |
| Recommendation role | Provides direct topic-level learning signals |

Design notes:

- This table should store non-LeetCode learning activity.
- It is especially useful for revision, topic coverage, checkpoint, skipped, struggled, and snoozed events.
- It should reference `catalog.topics.id` once that table exists.
- It should remain append-only.

Important relationships:

```text
app.pt_learning_events.user_id -> public.users.id
app.pt_learning_events.topic_id -> catalog.topics.id
```

Event examples:

| Event type | Meaning for recommendations |
|---|---|
| `covered` | User has been exposed to the topic |
| `checkpoint_passed` | User demonstrated sufficient understanding |
| `revised` | Topic was recently refreshed |
| `struggled` | Topic should become a stronger candidate |
| `skipped` | Topic may need to be delayed or deprioritized temporarily |
| `snoozed` | Topic should not be recommended until a future time |

---

### 7.3 Missing table: `app.recommendation_events`

This table should store user interactions with recommendations. It is separate from active recommendations because it records what happened historically.

| Property | Design Decision |
|---|---|
| Layer | Event log |
| Table lifecycle | Append-only source table |
| Reuse status | Create if missing |
| Purpose | Stores recommendation lifecycle and interaction events |
| Source of truth? | Yes, for recommendation interaction history |
| Mutable? | No |

Recommended event types:

| Event type | Meaning |
|---|---|
| `generated` | System generated a recommendation candidate |
| `shown` | Recommendation was displayed to user |
| `accepted` | User opened/started the recommendation |
| `completed` | User completed the recommended action |
| `skipped` | User skipped the recommendation |
| `dismissed` | User dismissed it more explicitly than skip |
| `expired` | Recommendation was no longer valid |
| `replaced` | Recommendation was superseded by another recommendation |

Recommended design fields:

| Field | Purpose |
|---|---|
| `id` | Event identifier |
| `user_id` | User receiving or interacting with recommendation |
| `event_type` | Type of recommendation event |
| `recommendation_id` | Optional stable ID tying together events for one recommendation instance |
| `recommended_platform` | Platform for recommended problem, such as LeetCode |
| `recommended_slug` | Problem slug if the recommendation is question-specific |
| `topic_id` | Topic being recommended or revised |
| `reason` | Human-readable/system reason such as `weak_topic`, `revision_due`, `low_coverage` |
| `source_event_id` | Optional link to event that triggered the recommendation |
| `metadata` | Optional JSON for non-critical context/debugging |
| `created_at` | Event timestamp |

Design notes:

- This table helps prevent repeatedly showing the same recommendation after a user skips it.
- It supports future audit/debugging: why did the user see this recommendation?
- It should not be used directly as the fast serving table.
- Recommendation interactions can feed future read-model updates.

---

## 8. Layer 4: Derived Progress Layer

### Purpose

The derived progress layer turns raw events into current user state.

It answers:

- Which problems has the user solved?
- Which problems has the user attempted but not solved?
- Which topics are strong, weak, stale, or due for revision?
- What is the current progress bar for each topic?

This layer is rebuildable from events.

---

### 8.1 Existing table: `user_data.user_coding_problems`

| Property | Design Decision |
|---|---|
| Layer | Derived progress read model |
| Table lifecycle | Mutable and rebuildable |
| Reuse status | Reuse |
| Purpose | Aggregated per-user, per-problem progress |
| Key fields from discovery | `user_id`, `slug`, `status`, `ac_count`, `latest_ac_at` |
| Recommendation role | Helps determine solved, attempted, stale, and unsolved problems |

Design notes:

- This table should summarize the current state of a user's relationship with each problem.
- It should be derived from `user_data.user_coding_submissions`.
- It can remain synchronously updated for MVP if the current code already does this reliably.
- Long term, it should be owned by a projector/progress builder to keep write ownership clear.

Example derived states:

| Status | Meaning |
|---|---|
| `not_started` | No known attempt |
| `attempted` | At least one submission but no accepted solution |
| `solved` | At least one accepted solution |
| `revised` | Solved/practiced again after some time |

Potential recommendation usage:

- Avoid recommending solved problems too frequently unless they are due for revision.
- Recommend unsolved problems under weak topics.
- Prefer problems with prior failed attempts if the topic is being reinforced.
- Use `latest_ac_at` to detect long-gap revision opportunities.

---

### 8.2 Existing table: `app.pt_user_topic_progress`

| Property | Design Decision |
|---|---|
| Layer | Derived topic-progress read model |
| Table lifecycle | Mutable and rebuildable |
| Reuse status | Reuse |
| Purpose | Aggregated per-user, per-topic progress |
| Key fields from discovery | `user_id`, `topic_id`, `status`, `coverage_score`, `next_revision_at` |
| Recommendation role | Main table for progress bar, weak topics, and due revision topics |

Design notes:

- This should become the central topic-level recommendation read model.
- It should be derived from `user_coding_submissions`, `user_coding_problems`, `pt_learning_events`, and `problem_topics`.
- `next_revision_at` is important because revision recommendations depend on time even when no new submission happens.
- `coverage_score` can represent how much of a topic the user has covered.
- Topic strength/weakness should be represented here or in a closely related derived read model.

Recommended conceptual fields if not already present:

| Field | Purpose |
|---|---|
| `user_id` | User whose progress is being summarized |
| `topic_id` | Topic being summarized |
| `status` | Current learning status |
| `coverage_score` | Progress/coverage from 0 to 1 or 0 to 100 |
| `readiness_score` | How ready the user is for this topic |
| `weakness_score` | How much the topic should be prioritized for practice |
| `last_practiced_at` | Last relevant practice event or submission under this topic |
| `last_solved_at` | Last accepted problem under this topic |
| `attempt_count` | Total attempts under this topic |
| `solve_count` | Accepted/solved count under this topic |
| `struggle_count` | Count or weighted count of failed attempts/struggles |
| `next_revision_at` | When this topic should be revised |
| `updated_at` | Last projection update |

This table should support two major recommendation categories:

1. Weakness repair.
2. Revision scheduling.

---

### 8.3 Recommended missing table: `app.recommendation_projection_state`

This table is not part of the user-facing product, but it is useful for the projector that turns events into read models.

| Property | Design Decision |
|---|---|
| Layer | Projection control |
| Table lifecycle | Mutable operational state |
| Reuse status | Create if no equivalent exists |
| Purpose | Tracks projector progress through event logs |
| Source of truth? | No, operational state only |
| Mutable? | Yes |

Recommended design fields:

| Field | Purpose |
|---|---|
| `projector_name` | Name of the projection job, such as `topic_progress_projector` |
| `source_table` | Event table being processed |
| `last_processed_id` | Last event ID processed, if event IDs are sequential |
| `last_processed_at` | Last event timestamp processed |
| `status` | Current status such as `idle`, `running`, `failed` |
| `last_error` | Optional error details |
| `updated_at` | Last update time |

Design notes:

- This avoids reprocessing all events every time.
- It also makes backfills and rebuilds easier to reason about.
- If the MVP updates read models synchronously, this table can be added later. However, it is recommended if a background projector is introduced.

---

## 9. Layer 5: Recommendation Serving Layer

### Purpose

This layer stores the recommendations that are ready to be shown to the user.

It answers:

- What should the UI show right now?
- Why is this recommendation being shown?
- When should this recommendation expire?
- What topic/problem does this recommendation target?

This is the low-latency serving layer for the app.

---

### 9.1 Missing table: `app.user_active_recommendations`

| Property | Design Decision |
|---|---|
| Layer | Recommendation serving read model |
| Table lifecycle | Mutable and rebuildable |
| Reuse status | Create if missing |
| Purpose | Stores currently active recommendation candidates for each user |
| Source of truth? | No, serving read model only |
| Mutable? | Yes |

Recommended design fields:

| Field | Purpose |
|---|---|
| `user_id` | User who should receive the recommendation |
| `recommendation_id` | Stable identifier for a recommendation instance |
| `recommendation_type` | Category such as `weak_topic`, `revision_due`, `next_problem`, `retry_unsolved` |
| `topic_id` | Topic being recommended |
| `recommended_platform` | Platform of the recommended problem |
| `recommended_slug` | Problem slug, if question-specific |
| `score` | Ranking score for ordering recommendations |
| `reason` | Short reason to display or debug |
| `reason_details` | Optional JSON with computed explanation details |
| `status` | Current serving state such as `active`, `snoozed`, `expired` |
| `generated_at` | When the recommendation was generated |
| `expires_at` | When the recommendation should no longer be shown |
| `updated_at` | Last update time |

Recommended primary identity:

```text
(user_id, recommendation_id)
```

Recommended uniqueness rule for MVP:

```text
Only one active recommendation per user/topic/type when appropriate.
```

Design notes:

- The recommendation API should read from this table rather than scanning raw events.
- This table can be cleared and rebuilt from the event logs and progress read models.
- Recommendation rows should explain why they exist using `reason` and `reason_details`.
- The UI can use this table to render due topics, weak topics, and next questions.

---

## 10. Data Ownership by Table

| Table | Layer | Lifecycle | Owner | Rebuildable? | MVP action |
|---|---|---|---|---|---|
| `public.users` | Identity | Mutable source | Auth/user system | No | Reuse |
| `user_data.user_coding_profiles` | Identity/platform | Mutable source | Platform sync | No | Reuse |
| `catalog.coding_problems` | Static catalog | Mutable source | Catalog ingestion/admin | No | Reuse |
| `catalog.topics` | Static catalog | Mutable source | Catalog ingestion/admin | No | Create if missing |
| `catalog.problem_topics` | Static catalog | Mutable source | Catalog ingestion/admin | No | Create if no equivalent exists |
| `user_data.user_coding_submissions` | Event log | Append-only source | LeetCode sync | No | Reuse |
| `app.pt_learning_events` | Event log | Append-only source | Learning/practice flows | No | Reuse |
| `app.recommendation_events` | Event log | Append-only source | Recommendation API/UI | No | Create if missing |
| `user_data.user_coding_problems` | Derived progress | Mutable read model | Progress projector | Yes | Reuse |
| `app.pt_user_topic_progress` | Derived progress | Mutable read model | Topic progress projector | Yes | Reuse |
| `app.user_active_recommendations` | Serving read model | Mutable read model | Recommendation projector | Yes | Create if missing |
| `app.recommendation_projection_state` | Projection control | Mutable operational | Projector process | Recreate/reseedable | Optional but recommended |

---

## 11. Interrelation Between Layers

### 11.1 Problem-level path

```text
public.users
  -> user_data.user_coding_profiles
  -> user_data.user_coding_submissions
  -> user_data.user_coding_problems
```

Meaning:

1. The user owns a coding profile.
2. The coding profile/import process creates submission events.
3. Submission events are summarized into per-problem progress.
4. Problem progress is used to avoid or prioritize specific questions.

---

### 11.2 Topic-level path

```text
catalog.coding_problems
  -> catalog.problem_topics
  -> catalog.topics
  -> app.pt_user_topic_progress
```

Meaning:

1. A problem belongs to one or more topics.
2. Topic mappings convert problem activity into topic progress.
3. Topic progress identifies weak and due topics.
4. Topic progress drives high-level recommendations.

---

### 11.3 Event-to-progress path

```text
user_data.user_coding_submissions
app.pt_learning_events
        |
        v
Progress projector
        |
        v
user_data.user_coding_problems
app.pt_user_topic_progress
```

Meaning:

1. Raw events capture behavior.
2. The projector reads raw events.
3. The projector updates progress read models.
4. Read models represent current state.

---

### 11.4 Progress-to-recommendation path

```text
user_data.user_coding_problems
app.pt_user_topic_progress
catalog.coding_problems
catalog.problem_topics
        |
        v
Recommendation projector
        |
        v
app.user_active_recommendations
```

Meaning:

1. The system identifies weak or due topics.
2. The system finds candidate questions under those topics.
3. The system avoids questions that should not be shown.
4. The system writes ready-to-serve recommendations.

---

### 11.5 Recommendation interaction feedback path

```text
app.user_active_recommendations
        |
        v
UI / API interaction
        |
        v
app.recommendation_events
        |
        v
Future projection updates
```

Meaning:

1. Recommendations are served from the active recommendation table.
2. User interactions are logged as recommendation events.
3. Future projections can use those events to avoid repetition or adjust priorities.

---

## 12. Recommended MVP Event Flow

### 12.1 Submission import flow

```text
1. LeetCode submission is imported.
2. Row is inserted into user_data.user_coding_submissions.
3. Problem-level progress is updated in user_data.user_coding_problems.
4. Topic-level progress is updated in app.pt_user_topic_progress using catalog.problem_topics.
5. Recommendation candidates are refreshed in app.user_active_recommendations.
```

Design note:

For MVP, steps 3-5 may happen synchronously or through a simple backend projector. The key design requirement is that the derived tables should be treated as rebuildable from events.

---

### 12.2 Learning/revision event flow

```text
1. User revises, skips, struggles with, or completes a topic.
2. Row is inserted into app.pt_learning_events.
3. Topic-level progress is updated in app.pt_user_topic_progress.
4. Active recommendations are refreshed.
```

Design note:

This flow allows recommendation behavior to react to in-app learning actions even if there is no new LeetCode submission.

---

### 12.3 Recommendation serving flow

```text
1. Client requests recommendations.
2. API reads app.user_active_recommendations for the user.
3. API joins catalog tables to get problem/topic display details.
4. API returns ranked recommendations.
5. API optionally writes a app.recommendation_events row with event_type = 'shown'.
```

Design note:

The recommendation API should not need to scan all submissions for every request. Its normal path should be a simple read from the active recommendation read model.

---

### 12.4 Recommendation interaction flow

```text
1. User accepts, skips, dismisses, or completes a recommendation.
2. API inserts a row into app.recommendation_events.
3. Active recommendation row is updated, expired, or replaced.
4. Topic/problem progress may be updated later based on the interaction.
```

Design note:

Recommendation interactions are important because they prevent poor user experience, such as showing the same skipped item again and again.

---

## 13. Source of Truth Rules

### 13.1 Canonical source tables

These tables represent source data and should not be rebuilt from other tables:

- `public.users`
- `user_data.user_coding_profiles`
- `catalog.coding_problems`
- `catalog.topics`
- `catalog.problem_topics`

### 13.2 Canonical event logs

These tables represent historical user behavior and should be append-only:

- `user_data.user_coding_submissions`
- `app.pt_learning_events`
- `app.recommendation_events`

### 13.3 Rebuildable read models

These tables represent current state and should be rebuildable:

- `user_data.user_coding_problems`
- `app.pt_user_topic_progress`
- `app.user_active_recommendations`

### 13.4 Operational state

This table helps process events but is not product data:

- `app.recommendation_projection_state`

---

## 14. Table-by-Table MVP Design Report

### 14.1 `public.users`

| Attribute | Design |
|---|---|
| Status | Existing, reuse |
| Layer | Identity |
| Type | Source table |
| Mutability | Mutable |
| Primary purpose | Identifies application users |
| Used by | All user-specific event and read-model tables |
| Key relation | `id` should be referenced by `user_id` fields |

Design responsibility:

`public.users` should only represent user identity and account metadata. Recommendation behavior should not be stored here.

---

### 14.2 `user_data.user_coding_profiles`

| Attribute | Design |
|---|---|
| Status | Existing, reuse |
| Layer | Identity/platform profile |
| Type | Source table |
| Mutability | Mutable |
| Primary purpose | Tracks platform connection such as LeetCode |
| Used by | Submission import/sync flow |

Design responsibility:

This table connects users to external coding profiles. It should be used to determine ownership of imported submissions, not to calculate recommendation state.

---

### 14.3 `catalog.coding_problems`

| Attribute | Design |
|---|---|
| Status | Existing, reuse |
| Layer | Static catalog |
| Type | Source table |
| Mutability | Mutable |
| Primary purpose | Stores coding problems |
| Important fields | `platform`, `slug`, `difficulty`, `topic_tags`, `ac_rate` |
| Used by | Recommendation candidate selection |

Design responsibility:

This table defines the universe of recommendable problems. It should be joined with topic mappings and user progress to select the next question.

Design caution:

`topic_tags` as a text array is useful but should not be the only topic relationship if the system relies on `topic_id` in progress tables.

---

### 14.4 `catalog.topics`

| Attribute | Design |
|---|---|
| Status | Missing, create if absent |
| Layer | Static catalog |
| Type | Source table |
| Mutability | Mutable, controlled |
| Primary purpose | Defines coding topic taxonomy |
| Used by | Topic progress, learning events, recommendation reasons |

Design responsibility:

This table should define stable topics such as Array, Dynamic Programming, BFS, DFS, Graphs, Trees, and other curriculum groupings.

Minimum required relationships:

```text
app.pt_learning_events.topic_id -> catalog.topics.id
app.pt_user_topic_progress.topic_id -> catalog.topics.id
catalog.problem_topics.topic_id -> catalog.topics.id
```

---

### 14.5 `catalog.problem_topics`

| Attribute | Design |
|---|---|
| Status | Create if no equivalent exists |
| Layer | Static catalog |
| Type | Source table |
| Mutability | Mutable, controlled |
| Primary purpose | Maps problems to topics |
| Used by | Topic progress projection and question recommendation |

Design responsibility:

This table translates problem activity into topic progress. Without this relationship, the system cannot reliably infer that solving a specific problem improves a specific topic.

---

### 14.6 `user_data.user_coding_submissions`

| Attribute | Design |
|---|---|
| Status | Existing, reuse |
| Layer | Event log |
| Type | Append-only source table |
| Mutability | Append-only |
| Primary purpose | Stores imported coding submissions |
| Important fields | `platform`, `submission_id`, `user_id`, `slug`, `verdict`, `ts` |
| Used by | Problem progress, topic progress, recommendation refresh |

Design responsibility:

This table is the behavioral source of truth for LeetCode/coding activity. It should preserve each imported submission event.

Projection responsibility:

- Accepted submissions increase solved counts.
- Failed submissions increase attempt/struggle signals.
- Submission timestamps feed recency and revision calculations.

---

### 14.7 `user_data.user_coding_problems`

| Attribute | Design |
|---|---|
| Status | Existing, reuse |
| Layer | Derived progress |
| Type | Rebuildable read model |
| Mutability | Mutable |
| Primary purpose | Summarizes per-user, per-problem progress |
| Important fields | `user_id`, `slug`, `status`, `ac_count`, `latest_ac_at` |
| Used by | Recommendation candidate filtering |

Design responsibility:

This table should make it fast to know which problems are solved, attempted, stale, or unsolved for a user.

It should not be treated as the permanent historical truth. That role belongs to `user_data.user_coding_submissions`.

---

### 14.8 `app.pt_learning_events`

| Attribute | Design |
|---|---|
| Status | Existing, reuse |
| Layer | Event log |
| Type | Append-only source table |
| Mutability | Append-only |
| Primary purpose | Stores learning, revision, and topic interaction events |
| Important fields | `user_id`, `event_type`, `topic_id`, `readiness_delta`, `created_at` |
| Used by | Topic progress and recommendation refresh |

Design responsibility:

This table captures in-app learning behavior that is not necessarily represented by LeetCode submissions.

Examples:

- User revised a topic.
- User struggled with a topic.
- User skipped a topic.
- User passed a checkpoint.

---

### 14.9 `app.pt_user_topic_progress`

| Attribute | Design |
|---|---|
| Status | Existing, reuse |
| Layer | Derived progress |
| Type | Rebuildable read model |
| Mutability | Mutable |
| Primary purpose | Summarizes per-user, per-topic progress |
| Important fields | `user_id`, `topic_id`, `status`, `coverage_score`, `next_revision_at` |
| Used by | Weak topic and due revision recommendations |

Design responsibility:

This should be the main read model for the user's progress bar and topic-level recommendation decisions.

Recommended interpretation:

- `coverage_score` shows how much of a topic the user has covered.
- `status` shows current learning state.
- `next_revision_at` identifies due revision topics.
- Additional scores can represent weakness, readiness, or confidence.

---

### 14.10 `app.recommendation_events`

| Attribute | Design |
|---|---|
| Status | Missing, create if absent |
| Layer | Event log |
| Type | Append-only source table |
| Mutability | Append-only |
| Primary purpose | Stores recommendation lifecycle and user interaction events |
| Used by | Recommendation feedback and future projections |

Design responsibility:

This table records what happened to recommendations after they were generated.

It should answer:

- Was the recommendation shown?
- Did the user accept it?
- Did the user skip it?
- Did it expire?
- Why was it generated?

---

### 14.11 `app.user_active_recommendations`

| Attribute | Design |
|---|---|
| Status | Missing, create if absent |
| Layer | Recommendation serving |
| Type | Rebuildable read model |
| Mutability | Mutable |
| Primary purpose | Stores recommendations ready to show to the user |
| Used by | Recommendation API/UI |

Design responsibility:

This table should be the normal read path for the recommendation API.

It should contain ready-to-serve rows with enough information to rank, explain, and expire recommendations.

---

### 14.12 `app.recommendation_projection_state`

| Attribute | Design |
|---|---|
| Status | Optional, recommended if projector is asynchronous |
| Layer | Projection control |
| Type | Operational state |
| Mutability | Mutable |
| Primary purpose | Tracks which events have been processed |
| Used by | Projector service/job |

Design responsibility:

This table prevents duplicate projection work and supports controlled rebuilds. It is not product data and should not be exposed to the UI.

---

## 15. Recommendation Categories Supported by the Design

### 15.1 Weak topic recommendation

Inputs:

- `app.pt_user_topic_progress.coverage_score`
- failed submission counts from `user_data.user_coding_submissions`
- problem progress from `user_data.user_coding_problems`
- topic mapping from `catalog.problem_topics`

Output:

- Active recommendation with `recommendation_type = 'weak_topic'`

Example:

```text
User has many failed attempts under Dynamic Programming and low topic coverage.
Recommend an easier unsolved Dynamic Programming problem.
```

---

### 15.2 Revision due recommendation

Inputs:

- `app.pt_user_topic_progress.next_revision_at`
- `last_practiced_at`
- `latest_ac_at`
- `app.pt_learning_events`

Output:

- Active recommendation with `recommendation_type = 'revision_due'`

Example:

```text
User solved Array problems 45 days ago but has not revised the topic recently.
Recommend an Array revision problem.
```

---

### 15.3 Retry unsolved recommendation

Inputs:

- `user_data.user_coding_problems.status`
- recent failed submissions
- topic weakness score

Output:

- Active recommendation with `recommendation_type = 'retry_unsolved'`

Example:

```text
User attempted a BFS problem multiple times but did not solve it.
Recommend retrying a similar or same BFS problem after practice.
```

---

### 15.4 Next topic progression recommendation

Inputs:

- topic hierarchy from `catalog.topics`
- coverage and readiness from `app.pt_user_topic_progress`
- solved problem counts from `user_data.user_coding_problems`

Output:

- Active recommendation with `recommendation_type = 'next_topic'`

Example:

```text
User has sufficient Array coverage and is ready to move into Two Pointers or Sliding Window.
```

---

## 16. Recommended Index Intent

This section describes index intent, not exact migration SQL.

### Event log read indexes

Needed for reading recent events and projecting by user:

```text
user_data.user_coding_submissions(user_id, ts)
user_data.user_coding_submissions(platform, slug)
app.pt_learning_events(user_id, created_at)
app.pt_learning_events(topic_id, created_at)
app.recommendation_events(user_id, created_at)
```

### Catalog join indexes

Needed for mapping problems to topics:

```text
catalog.coding_problems(platform, slug)
catalog.topics(slug)
catalog.problem_topics(topic_id)
catalog.problem_topics(platform, slug)
```

### Read-model serving indexes

Needed for fast recommendations:

```text
user_data.user_coding_problems(user_id, status)
user_data.user_coding_problems(user_id, latest_ac_at)
app.pt_user_topic_progress(user_id, next_revision_at)
app.pt_user_topic_progress(user_id, coverage_score)
app.user_active_recommendations(user_id, status, expires_at)
app.user_active_recommendations(user_id, score)
```

---

## 17. How the Design Handles Rebuilds

A rebuild means derived read models are recalculated from source events and catalog data.

### Rebuildable tables

- `user_data.user_coding_problems`
- `app.pt_user_topic_progress`
- `app.user_active_recommendations`

### Non-rebuildable source tables

- `public.users`
- `user_data.user_coding_profiles`
- `catalog.coding_problems`
- `catalog.topics`
- `catalog.problem_topics`
- `user_data.user_coding_submissions`
- `app.pt_learning_events`
- `app.recommendation_events`

### Why rebuilds matter

The MVP scoring logic may change. For example, the team may later decide:

- Failed attempts should count more heavily.
- Old solved topics should decay faster.
- Easy problems should affect readiness less than medium/hard problems.
- Skipped recommendations should suppress a topic for a limited time.

If progress and recommendation tables are derived from events, the system can rebuild them using updated rules without losing history.

---

## 18. Design Boundaries and Non-Goals for MVP

### In scope

- Supabase Postgres tables.
- Append-only event logs.
- Derived progress read models.
- Active recommendation serving table.
- Topic taxonomy.
- Problem-to-topic mapping.
- Rebuildable projections.
- Simple scoring and prioritization.

### Out of scope for MVP

- External event brokers.
- External feature stores.
- External streaming systems.
- Complex ML training pipelines.
- Real-time multi-consumer event fanout.
- Distributed event replay infrastructure.
- Per-request full-history recommendation computation.

---

## 19. Recommended MVP Schema Creation Plan

This is a design-order plan, not migration SQL.

### Step 1: Confirm and preserve existing reusable tables

Reuse:

- `public.users`
- `user_data.user_coding_profiles`
- `catalog.coding_problems`
- `user_data.user_coding_submissions`
- `user_data.user_coding_problems`
- `app.pt_learning_events`
- `app.pt_user_topic_progress`

### Step 2: Add missing static catalog structure

Create if missing:

- `catalog.topics`
- `catalog.problem_topics`

Purpose:

- Make topic taxonomy explicit.
- Make problem-topic mapping queryable.
- Allow topic progress to reference actual catalog topics.

### Step 3: Add recommendation event history

Create if missing:

- `app.recommendation_events`

Purpose:

- Record generated/shown/accepted/skipped/completed recommendation facts.
- Preserve feedback history.

### Step 4: Add recommendation serving state

Create if missing:

- `app.user_active_recommendations`

Purpose:

- Store ready-to-show recommendations.
- Keep recommendation API fast.
- Allow recommendations to expire or be replaced.

### Step 5: Add projection control only if needed

Create if needed:

- `app.recommendation_projection_state`

Purpose:

- Track projector cursor/state.
- Support incremental event processing.
- Support rebuilds/backfills.

---

## 20. Recommended MVP Read Path

The recommendation API should usually follow this path:

```text
1. Receive user_id.
2. Read active rows from app.user_active_recommendations.
3. Join catalog.topics for topic display metadata.
4. Join catalog.coding_problems for question display metadata.
5. Return ranked recommendations.
6. Log app.recommendation_events event_type = 'shown' if appropriate.
```

This keeps the read path simple and fast.

The API should not normally scan all submissions, all learning events, or all user history on every recommendation request.

---

## 21. Recommended MVP Write Path

### Submission write path

```text
1. Insert into user_data.user_coding_submissions.
2. Update/recompute user_data.user_coding_problems.
3. Update/recompute app.pt_user_topic_progress.
4. Refresh app.user_active_recommendations.
```

### Learning event write path

```text
1. Insert into app.pt_learning_events.
2. Update/recompute app.pt_user_topic_progress.
3. Refresh app.user_active_recommendations.
```

### Recommendation interaction write path

```text
1. Insert into app.recommendation_events.
2. Update/expire/snooze app.user_active_recommendations.
3. Optionally update topic progress if the interaction represents learning behavior.
```

---

## 22. Key Design Decisions

### Decision 1: Keep submissions as event source

`user_data.user_coding_submissions` should remain the source of truth for imported coding behavior.

Reason:

- It captures raw behavior.
- It supports replay.
- It allows logic to change later.

### Decision 2: Keep topic progress as derived state

`app.pt_user_topic_progress` should be the central progress-bar table, but it should be treated as derived.

Reason:

- Topic progress depends on interpretation of events.
- Interpretation will change as the product evolves.
- Rebuilding from events is safer than locking early scoring rules into permanent state.

### Decision 3: Add explicit topic taxonomy

`catalog.topics` should exist because topic IDs are already used conceptually by learning events and topic progress.

Reason:

- Topic progress needs stable topic identity.
- Recommendations need clear topic labels.
- Topic hierarchy may be useful later.

### Decision 4: Add active recommendations as a read model

`app.user_active_recommendations` should store what the UI should show now.

Reason:

- Fast reads.
- Better separation between recommendation computation and serving.
- Clear expiration/replacement behavior.

### Decision 5: Track recommendation interactions as events

`app.recommendation_events` should be append-only.

Reason:

- Helps avoid repeating skipped recommendations.
- Enables future analysis.
- Makes recommendation behavior auditable.

---

## 23. Potential Risks and Design Responses

### Risk 1: Topic taxonomy is missing or inconsistent

Problem:

`app.pt_user_topic_progress` uses `topic_id`, but a corresponding `catalog.topics` table may not exist.

Design response:

Create `catalog.topics` and link topic progress, learning events, and problem mappings to it.

---

### Risk 2: `topic_tags` text arrays diverge from normalized topics

Problem:

`catalog.coding_problems.topic_tags` may not match canonical topic IDs.

Design response:

Use `catalog.problem_topics` as the canonical mapping. `topic_tags` can remain as imported metadata or fallback display data.

---

### Risk 3: Synchronous aggregation creates unclear ownership

Problem:

Current code may update derived problem progress during ingestion.

Design response:

For MVP, synchronous updates can remain if reliable. However, ownership should be documented: derived progress is still rebuildable and should not be treated as the source of truth.

---

### Risk 4: Recommendation API becomes too heavy

Problem:

If the API recomputes recommendations from all events per request, latency and complexity will grow.

Design response:

Use `app.user_active_recommendations` as the serving read model.

---

### Risk 5: Recommendation state becomes permanent truth

Problem:

If active recommendation rows are treated as permanent truth, future scoring changes become hard.

Design response:

Treat `app.user_active_recommendations` as rebuildable serving state and preserve history in `app.recommendation_events`.

---

## 24. Final MVP Architecture Summary

The MVP should use a Supabase Postgres event-plus-read-model architecture.

### Source/catalog truth

- `public.users`
- `user_data.user_coding_profiles`
- `catalog.coding_problems`
- `catalog.topics`
- `catalog.problem_topics`

### Behavioral event truth

- `user_data.user_coding_submissions`
- `app.pt_learning_events`
- `app.recommendation_events`

### Derived/read-model truth for serving

- `user_data.user_coding_problems`
- `app.pt_user_topic_progress`
- `app.user_active_recommendations`

### Operational projection state

- `app.recommendation_projection_state`

The core design relationship is:

```text
Catalog defines what can be recommended.
Events define what the user did.
Progress tables define what the user currently needs.
Active recommendations define what the product should show now.
Recommendation events define how the user reacted.
```

This architecture is appropriate for the MVP because it is simple, explainable, Postgres-native, and flexible enough to support future improvements without requiring infrastructure changes.
