# Recommendation Layer Design Report

## 1. Purpose

This report explains the updated recommendation-layer design for the coding practice and interview preparation product. It is written as a product and engineering handoff document for team members who need to understand what modules should be created, what each module owns, which existing tables should be reused, which new tables are actually needed, and how the system should remain production-ready and future-proof.

The goal is to build a recommendation layer that can answer:

**What should this user practice right now, and why?**

The recommendation layer should not generate generic problem lists. It should recommend practice actions using the user's actual coding history, solved problems, failed attempts, weak topics, target company, revision needs, and previous recommendation interactions.

The design avoids request-time heavy computation. Recommendations should be generated asynchronously, stored, and served quickly through the main backend.

---

## 2. Updated Direction

The recommendation layer should be a standalone intelligence layer, but it should reuse the already existing source-of-truth tables wherever possible.

The correct direction is:

| Area | Decision |
|---|---|
| User profile, submissions, solve graph, code blobs, sync state | Reuse existing tables |
| Recommendation features, problem intelligence, topic intelligence, recommendation outputs, jobs, events, scoring versions | Add focused intelligence tables |
| Main backend | Reads stored recommendations and records user actions |
| Recommendation layer | Generates, ranks, stores, refreshes, and explains recommendations |
| LLM usage | Optional, limited, and mostly used for explanation or plan packaging |

Creating more tables is not the default answer. New tables should only be created for derived intelligence, persisted recommendation outputs, async jobs, and analytics that do not already exist in the current schema.

---

## 3. Production Principles

The system should be designed for at least 5,000 daily active users.

The main production principles are:

| Principle | Meaning |
|---|---|
| Precompute recommendations | Do not calculate full recommendations during every page load |
| Store generated output | Recommendation APIs should mostly read stored rows |
| Use async jobs | Scheduled, post-sync, post-submission, and manual refreshes should run outside the request path |
| Keep LLM optional | Backend scoring should work even if LLMs are unavailable |
| Version everything | Scoring formulas, feature logic, and prompt logic should be versioned |
| Track all user actions | Click, dismiss, solve, skip, and feedback events should be recorded |
| Reuse source-of-truth tables | Do not duplicate raw user activity or problem catalog data |
| Fail gracefully | Old recommendations should remain available if new generation fails |

---

## 4. Existing Tables to Reuse

The current database already has several strong source-of-truth tables. These should be reused instead of duplicated.

### 4.1 User and Identity

| Existing table | How recommendation layer uses it |
|---|---|
| `public.users` | User identity, tier, account ownership |

The recommendation layer should not duplicate user identity fields.

---

### 4.2 Problem Catalog

| Existing table | How recommendation layer uses it |
|---|---|
| `catalog.coding_problems` | Problem title, platform, slug, display ID, difficulty, topic tags, statement, constraints, examples, hints, similar problems, raw metadata |

This table is the main factual problem catalog. Do not create another general problem table.

The recommendation layer can extend problem intelligence through a separate derived table, but factual catalog data should remain here.

---

### 4.3 User Coding Profiles

| Existing table | How recommendation layer uses it |
|---|---|
| `user_data.user_coding_profiles` | Connected coding account, platform, handle, sync state, profile metadata, connection status |

This table tells whether the user has a connected coding profile and what platform/account the recommendation layer should consider.

---

### 4.4 User Solve Graph

| Existing table | How recommendation layer uses it |
|---|---|
| `user_data.user_coding_problems` | Per-user-per-problem status, attempt count, AC count, WA/TLE/MLE/RE/CE counts, latest attempt, latest AC, best runtime/memory, primary submission |

This is one of the most important recommendation inputs.

It should power:

| Use case | Why this table is enough |
|---|---|
| Solved exclusion | Status and AC count show solved state |
| Failed retry | Attempt counts and failure counts show tried-but-unsolved problems |
| Revision | Latest AC date shows whether a solved problem is stale |
| Difficulty readiness | Solved distribution can be grouped by catalog difficulty |
| Recent practice | Latest attempt and latest AC timestamps show activity |
| Repetition control | Shows whether a problem was already handled by the user |

Do not create a duplicate `user_solved_problems` table if this table is already the current source of truth.

---

### 4.5 User Submissions

| Existing table | How recommendation layer uses it |
|---|---|
| `user_data.user_coding_submissions` | Atomic submissions, verdict, language, runtime, memory, testcase progress, errors, timestamp, source, code availability |

This table should power deeper signals, such as:

| Signal | Example |
|---|---|
| Close-to-solve failed problem | User failed but passed most testcases |
| Repeated WA pattern | Many wrong answers on one topic |
| TLE-heavy topic | User understands logic but struggles with complexity |
| Language preference | User solves mostly in Python/C++/Java |
| Recent struggle | Several failed attempts in a short window |

This table should be read by the User Feature Builder and Failed Retry Candidate Generator.

---

### 4.6 Code Blob Storage

| Existing table | How recommendation layer uses it |
|---|---|
| `code_blobs.user_coding_submission_code` | Future advanced code-analysis features |

Normal recommendation refresh should not read code blobs. Code blobs are heavier and should only be used for future advanced modules like code review, repeated mistake detection, or solution-quality analysis.

---

### 4.7 Sync State

| Existing table | How recommendation layer uses it |
|---|---|
| `system.sync_state` | Detect sync status, full sync completion, delta sync, retry state, errors |

The recommendation layer can use sync completion as a trigger, but it should not own sync state.

---

### 4.8 Chat, Context, and User Memory

| Existing table | How recommendation layer uses it |
|---|---|
| `app.chat_sessions` | Session context |
| `app.chat_messages` | Conversation history, if needed for later context-aware recommendations |
| `app.chat_feedback` | Quality signal for recommendation explanations and coaching responses |
| `app.user_context_events` | Extracted user goals, constraints, preferences, and context events |
| `app.user_memory` | Summarized durable user memory |
| `app.user_profile_summary` | Materialized user profile summary, target context, preferences, focus domain, recent signals |

The recommendation layer should reuse these for target company, target level, current focus, preferences, and recent user intent. Do not create another generic user preference table unless a recommendation-specific feature cache is needed.

---

### 4.9 Prep and Progress Tables

| Existing table | How recommendation layer uses it |
|---|---|
| `app.pt_user_targets` | Target company, level, role family, active/primary target |
| `app.pt_user_topic_progress` | Learning progress per topic, coverage, readiness, revision dates |
| `app.pt_learning_events` | Append-only learning events and user progress changes |
| `app.user_prep_plans` | Active/paused/completed prep plans |
| `app.user_prep_task_progress` | Daily task status, skipped/done/hinted, time spent, revision due date |
| `app.user_readiness_snapshots` | Readiness score history and component breakdown |

These tables should be reused for plan-aware recommendations, readiness-aware recommendations, revision loops, and target-specific prioritization.

---

## 5. New Tables That Are Actually Needed

The existing database covers raw data and progress well, but recommendation-specific output and derived intelligence are still needed.

Recommended new schema:

**`intelligence`**

Recommended new tables:

| New table | Required? | Purpose |
|---|---:|---|
| `intelligence.user_recommendation_features` | Yes | Cached user-level recommendation features |
| `intelligence.topic_intelligence` | Yes | Topic-level metadata for recommendation ranking |
| `intelligence.problem_intelligence` | Yes | Problem-level recommendation metadata |
| `intelligence.user_problem_recommendations` | Yes | Stored generated recommendations |
| `intelligence.recommendation_events` | Yes | User interaction events for recommendations |
| `intelligence.recommendation_jobs` | Yes | Async generation jobs and retries |
| `intelligence.recommendation_model_versions` | Yes | Scoring and prompt version tracking |
| `intelligence.recommendation_snapshots` | Optional | Daily/weekly analytics snapshots |

These tables should not duplicate existing raw data. They should store derived intelligence, outputs, and lifecycle events.

---

## 6. Module Overview

The recommendation layer should be split into the following modules:

1. Recommendation Orchestrator
2. User Feature Builder
3. Topic Intelligence Builder
4. Problem Intelligence Builder
5. Candidate Generator
6. Scoring Engine
7. Diversity and Cooldown Engine
8. Recommendation Persistence Manager
9. Recommendation Job Scheduler
10. Recommendation Event Tracker
11. LLM Explanation Generator
12. Notification Bridge
13. Metrics and Quality Monitor

Each module has a clear responsibility and dependency boundary.

---

## 7. Module 1: Recommendation Orchestrator

### Purpose

The Recommendation Orchestrator coordinates the full recommendation generation flow. It should not own all logic directly. It calls other modules in the correct order.

### Responsibilities

| Responsibility |
|---|
| Receive refresh triggers |
| Create or validate a recommendation job |
| Load cached or freshly computed user features |
| Ask candidate generators for possible recommendations |
| Send candidates to the scoring engine |
| Apply diversity and cooldown rules |
| Persist selected recommendations |
| Trigger optional LLM explanation generation |
| Trigger notification bridge if meaningful recommendations exist |
| Record success, failure, skip, and latency |

### Dependencies

Reads from:

| Source | Reason |
|---|---|
| `intelligence.recommendation_jobs` | Check duplicate or active jobs |
| `intelligence.user_recommendation_features` | Load user feature cache |
| `app.pt_user_targets` | Understand target company and level |
| `system.sync_state` | Know whether sync is complete or stale |

Writes to:

| Target | Reason |
|---|---|
| `intelligence.recommendation_jobs` | Track job lifecycle |
| `intelligence.user_problem_recommendations` | Store final recommendations |
| `intelligence.recommendation_events` | Record lifecycle events |

### Important fields handled

| Field | Meaning |
|---|---|
| `user_id` | User being processed |
| `job_type` | Daily, post-sync, post-submission, manual refresh, etc. |
| `trigger_source` | Scheduler, sync, submission, solve event, user action |
| `idempotency_key` | Prevents duplicate jobs |
| `model_version_id` | Scoring version used |
| `started_at` | Job start time |
| `finished_at` | Job finish time |
| `status` | Queued, running, succeeded, failed, skipped |

---

## 8. Module 2: User Feature Builder

### Purpose

The User Feature Builder converts raw user activity into a recommendation-ready feature profile.

This module prevents every recommendation job from repeatedly scanning raw submission and progress tables.

### Existing table dependencies

| Existing table | Used for |
|---|---|
| `user_data.user_coding_profiles` | Connected account, platform, profile state |
| `user_data.user_coding_problems` | Solved/tried/failed state per problem |
| `user_data.user_coding_submissions` | Recent verdicts, failure types, language, recency |
| `catalog.coding_problems` | Difficulty and topic mapping for solved/attempted problems |
| `app.user_profile_summary` | User preferences, current focus, durable profile context |
| `app.pt_user_targets` | Target company, level, role family |
| `app.pt_user_topic_progress` | Topic readiness and revision dates |
| `app.user_prep_plans` | Active plan context |
| `app.user_prep_task_progress` | Task completion and revision due dates |
| `app.user_readiness_snapshots` | Readiness trend and component breakdown |

### Output table

`intelligence.user_recommendation_features`

### Suggested fields

| Field | Meaning |
|---|---|
| `user_id` | User identifier |
| `platform` | Coding platform |
| `target_company` | Current primary target company |
| `target_level` | Target level |
| `role_family` | Backend, frontend, ML, etc. |
| `total_solved` | Total accepted problems |
| `total_attempted` | Total attempted problems |
| `easy_solved` | Easy accepted count |
| `medium_solved` | Medium accepted count |
| `hard_solved` | Hard accepted count |
| `failed_problem_count` | Attempted but unsolved count |
| `recent_failed_count` | Recent failed submissions count |
| `revision_due_count` | Important solved problems due for revision |
| `recent_activity_score` | Activity strength over recent window |
| `current_difficulty_band` | Recommended difficulty level for next practice |
| `weak_topics_json` | Ranked weak topics with evidence |
| `strong_topics_json` | Ranked strong topics |
| `stale_topics_json` | Topics not practiced recently |
| `recent_topics_json` | Recently practiced topics |
| `language_preference` | Main language used successfully |
| `readiness_summary_json` | Readiness components if available |
| `confidence_score` | How reliable the feature profile is |
| `feature_version` | Version of feature logic |
| `computed_at` | Last computed timestamp |

### LLM usage

No LLM should be used here.

---

## 9. Module 3: Topic Intelligence Builder

### Purpose

The Topic Intelligence Builder produces topic-level recommendation metadata. It is not user-specific unless used together with user features.

It answers:

| Question |
|---|
| Which topics are generally important? |
| Which topics are important for specific companies? |
| Which topics are foundational? |
| Which topics should be revised more often? |
| Which topics unlock other topics? |
| Which topics are relevant by level or role? |

### Existing table dependencies

| Existing table | Used for |
|---|---|
| `catalog.coding_problems` | Extract topic tags and topic-to-problem mapping |
| `app.pt_user_topic_progress` | Align with existing progress topics |
| `app.pt_learning_events` | Understand topic learning activity trends |
| Interview/company frequency data | Topic importance, if available |

### Output table

`intelligence.topic_intelligence`

### Suggested fields

| Field | Meaning |
|---|---|
| `topic_slug` | Canonical topic key |
| `display_name` | Human-readable topic name |
| `topic_group` | DSA, system design, behavioral, etc. |
| `interview_frequency_score` | Global importance |
| `company_frequency_json` | Company-specific importance |
| `level_relevance_json` | L4/L5/staff relevance |
| `role_relevance_json` | Backend/frontend/ML relevance |
| `foundation_score` | How foundational the topic is |
| `prerequisite_topics_json` | Topics to learn first |
| `next_topics_json` | Natural progression topics |
| `decay_days` | Suggested revision interval |
| `difficulty_curve_json` | How difficulty increases inside topic |
| `is_core_topic` | Whether topic belongs to core prep |
| `source_confidence` | Confidence in data |
| `computed_at` | Last computed timestamp |

### Should this be a new table?

Yes. Topic tags already exist in the catalog, but topic intelligence does not. This table stores derived ranking metadata, not raw facts.

### LLM usage

Optional and offline only. For example, LLMs may later help normalize topic aliases or describe prerequisites, but normal recommendation refresh should not call an LLM for topic intelligence.

---

## 10. Module 4: Problem Intelligence Builder

### Purpose

The Problem Intelligence Builder enriches catalog problems with recommendation-specific metadata.

The problem catalog should remain factual. Problem Intelligence should store ranking and recommendation metadata.

### Existing table dependencies

| Existing table | Used for |
|---|---|
| `catalog.coding_problems` | Problem title, slug, platform, difficulty, topics, statement, hints, similar problems |
| `intelligence.topic_intelligence` | Topic ranking and prerequisite context |
| User solve graph aggregate | Optional global difficulty/engagement insights |
| Interview/company frequency data | Problem importance and company relevance |

### Output table

`intelligence.problem_intelligence`

### Suggested fields

| Field | Meaning |
|---|---|
| `platform` | Coding platform |
| `slug` | Problem slug |
| `display_id` | Problem display number or ID |
| `is_standard_important` | Curated important problem flag |
| `interview_frequency_score` | Global interview importance |
| `company_frequency_json` | Company-specific frequency and relevance |
| `foundation_value_score` | How useful as a foundation problem |
| `revision_value_score` | How useful for future revision |
| `pattern_tags_json` | Patterns such as BFS, DFS, sliding window, DP |
| `primary_pattern` | Main pattern |
| `secondary_patterns_json` | Supporting patterns |
| `prerequisite_problem_refs_json` | Problems to solve first |
| `next_problem_refs_json` | Natural next problems |
| `similar_problem_refs_json` | Similar reinforcement problems |
| `estimated_minutes` | Estimated solve time |
| `common_failure_modes_json` | Common mistakes or failure reasons |
| `source_confidence` | Confidence in intelligence metadata |
| `llm_enriched` | Whether enrichment used LLM |
| `llm_enrichment_version` | Version of enrichment prompt/model |
| `computed_at` | Last computed timestamp |

### Should this be a new table?

Yes. Do not add all of these fields to `catalog.coding_problems`. The catalog should stay clean and factual. Problem intelligence is derived and versioned.

### LLM usage

Optional and offline only. LLM can help classify patterns, prerequisites, common mistakes, or explanations, but not during every recommendation refresh.

---

## 11. Module 5: Candidate Generator

### Purpose

The Candidate Generator creates possible recommendation candidates before scoring.

It should not decide final ranking. It only returns candidates from different sources.

### Dependencies

| Source | Used for |
|---|---|
| `intelligence.user_recommendation_features` | User-level weakness and readiness |
| `intelligence.topic_intelligence` | Topic importance |
| `intelligence.problem_intelligence` | Problem ranking metadata |
| `catalog.coding_problems` | Basic problem data |
| `user_data.user_coding_problems` | Solved, tried, failed, revision state |
| `user_data.user_coding_submissions` | Recent failed attempts |
| `app.pt_user_targets` | Company and level targeting |
| `app.user_prep_task_progress` | Existing plan progress and skipped tasks |
| `intelligence.user_problem_recommendations` | Avoid duplicates and recently shown problems |

### Candidate generator submodules

| Submodule | Generates |
|---|---|
| Standard Practice Generator | Important unsolved standard problems |
| Weak Topic Generator | Problems from weak user topics |
| Failed Retry Generator | Attempted but unsolved problems worth retrying |
| Revision Generator | Solved important problems due for revision |
| Similar Problem Generator | Reinforcement problems similar to recent solves |
| Next Difficulty Generator | Natural difficulty progression problems |
| Company Target Generator | Problems relevant to target company and level |
| Daily Plan Generator | Balanced daily practice bundle candidates |

### Should this create tables?

No. Candidate generation is a runtime process. Only final selected recommendations should be persisted.

---

## 12. Module 6: Scoring Engine

### Purpose

The Scoring Engine ranks candidates using a deterministic, versioned scoring model.

It should not call LLMs.

### Dependencies

| Source | Used for |
|---|---|
| Candidate objects | Candidate metadata |
| `intelligence.user_recommendation_features` | User weakness, readiness, activity |
| `intelligence.problem_intelligence` | Problem-level scores |
| `intelligence.topic_intelligence` | Topic-level scores |
| `intelligence.recommendation_model_versions` | Current scoring weights and penalties |

### Score components

| Component | Meaning |
|---|---|
| `importance_score` | Is this a core or standard problem? |
| `interview_frequency_score` | Is this commonly asked? |
| `company_relevance_score` | Is it relevant to target company? |
| `topic_weakness_score` | Does it address a weak topic? |
| `difficulty_match_score` | Is it the right difficulty now? |
| `failed_attempt_score` | Has the user tried and failed this problem? |
| `revision_due_score` | Is this solved problem due for revision? |
| `progression_score` | Is this the natural next step? |
| `recency_momentum_score` | Does it build on recent practice? |
| `freshness_score` | Is it still timely and useful? |
| `penalty_score` | Reduces score for dismissal, repetition, staleness, or bad fit |

### Output

The scoring engine outputs:

| Field | Meaning |
|---|---|
| `score` | Final numeric score |
| `score_breakdown_json` | Component-level score details |
| `reason_code` | Machine-readable explanation |
| `evidence_json` | Data that supports the recommendation |
| `model_version_id` | Scoring model used |

---

## 13. Module 7: Diversity and Cooldown Engine

### Purpose

This module prevents repetitive, annoying, or stale recommendations.

### Dependencies

| Source | Used for |
|---|---|
| Scored candidate list | Input list to filter and reorder |
| `intelligence.user_problem_recommendations` | Previous active, expired, dismissed, solved recommendations |
| `intelligence.recommendation_events` | User interactions and behavior history |
| `user_data.user_coding_problems` | Current solved/attempted state |

### Example rules

| Rule | Purpose |
|---|---|
| Max 2 same topic in top 5 | Avoid topic repetition |
| Max 2 same difficulty in top 5 | Keep daily practice balanced |
| Include revision if due | Build retention loop |
| Include failed retry if high value | Use personal struggle data |
| Suppress recently dismissed problems | Respect user intent |
| Suppress recently shown problems | Avoid spam |
| Allow solved problems only for revision | Avoid bad recommendations |
| Suppress expired plans | Keep outputs fresh |

### Should this create tables?

No separate table is needed. Cooldown, expiry, and status fields should live on `intelligence.user_problem_recommendations`.

---

## 14. Module 8: Recommendation Persistence Manager

### Purpose

This module stores final generated recommendations and makes them readable by the main backend.

### Output table

`intelligence.user_problem_recommendations`

### Suggested fields

| Field | Meaning |
|---|---|
| `id` | Recommendation ID |
| `user_id` | User identifier |
| `platform` | Coding platform |
| `slug` | Problem slug |
| `recommendation_type` | Standard practice, weak topic, retry, revision, company targeted, etc. |
| `status` | Pending, shown, clicked, solved, dismissed, expired |
| `priority` | High, medium, low |
| `rank` | Rank in generated bundle |
| `score` | Final recommendation score |
| `score_breakdown_json` | Component-level scoring details |
| `reason_code` | Machine-readable reason |
| `reason_text` | Rule-based user-facing explanation |
| `llm_reason_text` | Optional LLM-generated explanation |
| `evidence_json` | Supporting evidence |
| `source_job_id` | Job that created the recommendation |
| `model_version_id` | Scoring model version |
| `recommended_at` | Creation time |
| `shown_at` | First shown time |
| `clicked_at` | Click time |
| `solved_at` | Solve time |
| `dismissed_at` | Dismiss time |
| `expires_at` | Recommendation expiry |
| `cooldown_until` | Suppression until date |
| `metadata_json` | Extra data |

### Why this table is needed

The backend needs a fast read path. It should fetch stored active recommendations, not compute recommendations live.

---

## 15. Module 9: Recommendation Job Scheduler

### Purpose

Runs recommendation generation outside the request path.

### Output table

`intelligence.recommendation_jobs`

### Suggested fields

| Field | Meaning |
|---|---|
| `id` | Job ID |
| `user_id` | User being processed |
| `job_type` | Daily, after sync, after submission, after solve, manual refresh |
| `trigger_source` | Scheduler, sync service, submission capture, user request |
| `status` | Queued, running, succeeded, failed, skipped |
| `idempotency_key` | Prevents duplicate jobs |
| `attempt_count` | Retry count |
| `started_at` | Start time |
| `finished_at` | Finish time |
| `failed_at` | Failure time |
| `error_message` | Failure reason |
| `input_json` | Trigger context |
| `output_json` | Result summary |
| `created_at` | Job creation time |

### Required job types

| Job type | Purpose |
|---|---|
| `recommendation_refresh_daily` | Generate daily active recommendations |
| `recommendation_refresh_after_sync` | Refresh after profile sync |
| `recommendation_refresh_after_submission` | Refresh after new submission |
| `recommendation_refresh_after_failed_attempt` | Refresh retry candidates |
| `recommendation_refresh_after_solve` | Update solved/revision/progression state |
| `recommendation_refresh_manual` | User-requested refresh |
| `llm_reason_generation` | Generate explanations for selected items |
| `recommendation_cleanup` | Expire stale recommendations |

### Idempotency examples

| Trigger | Example idempotency key pattern |
|---|---|
| Daily refresh | User + date |
| After submission | User + submission ID |
| After sync | User + platform + sync timestamp |
| Manual refresh | User + hour window |

---

## 16. Module 10: Recommendation Event Tracker

### Purpose

Tracks user interactions with recommendations.

### Output table

`intelligence.recommendation_events`

### Suggested fields

| Field | Meaning |
|---|---|
| `id` | Event ID |
| `user_id` | User identifier |
| `recommendation_id` | Related recommendation |
| `platform` | Coding platform |
| `slug` | Problem slug |
| `event_type` | Created, shown, clicked, dismissed, solved, skipped, expired |
| `surface` | Today, weak topic, revision, retry, company, notification, chat |
| `metadata_json` | Extra event context |
| `created_at` | Event time |

### Events to track

| Event | Meaning |
|---|---|
| `recommendation_created` | Recommendation generated |
| `recommendation_shown` | User saw it |
| `recommendation_clicked` | User opened it |
| `recommendation_solved` | User solved it |
| `recommendation_dismissed` | User dismissed it |
| `recommendation_skipped_already_solved` | User says already solved |
| `recommendation_marked_not_relevant` | User says it is not useful |
| `recommendation_expired` | System expired it |
| `recommendation_replaced` | New recommendation replaced old one |

This table powers quality measurement and future model tuning.

---

## 17. Module 11: LLM Explanation Generator

### Purpose

Generates optional natural-language explanations and daily plan summaries.

The LLM should not decide the full recommendation list. The backend should rank and select candidates first.

### Dependencies

| Source | Used for |
|---|---|
| `intelligence.user_problem_recommendations` | Selected recommendations |
| `intelligence.user_recommendation_features` | User weakness/readiness context |
| `intelligence.problem_intelligence` | Problem explanation details |
| `catalog.coding_problems` | Problem title, difficulty, topics |
| `app.user_profile_summary` | User preference and target context |

### When to use LLM

| Case | LLM needed? |
|---|---:|
| Normal scheduled refresh | No |
| After profile sync refresh | Usually no |
| After new submission refresh | Usually no |
| After failed attempt refresh | Usually no |
| After solve event refresh | Usually no |
| User opens detailed daily plan | Optional |
| Top 3 recommendation explanations | Optional |
| User asks “why this?” | Yes |
| Company-specific plan summary | Optional |
| Weekly plan narrative | Optional |

### Output fields

| Field | Meaning |
|---|---|
| `llm_reason_text` | Human-friendly generated explanation |
| `llm_prompt_version` | Prompt version used |
| `llm_generated_at` | Generation timestamp |
| `llm_cost_metadata_json` | Optional cost/usage tracking |

### Rule

The system must work even if the LLM is unavailable. In that case, use `reason_text`, which should be rule-based.

---

## 18. Module 12: Notification Bridge

### Purpose

The Notification Bridge tells the notification system when meaningful recommendations exist.

It should not deliver notifications itself unless notification delivery is part of the same service. Its main job is to emit notification-worthy events.

### Dependencies

| Source | Used for |
|---|---|
| `intelligence.user_problem_recommendations` | Count and type of active recommendations |
| `intelligence.recommendation_events` | Avoid duplicate notification triggers |
| Notification tables/service | Store and deliver notification |

### Notification triggers

| Notification type | Trigger |
|---|---|
| `new_daily_plan` | Fresh daily bundle generated |
| `weak_topic_practice` | High-priority weak-topic recommendation exists |
| `retry_failed_problem` | Valuable failed retry recommendation exists |
| `revision_due` | Revision items are due |
| `company_targeted_practice` | Target-company recommendations are ready |
| `reorientation` | Inactive user needs restart recommendations |
| `expiring_recommendations` | Useful recommendations expire soon |

---

## 19. Module 13: Metrics and Quality Monitor

### Purpose

Measures whether recommendations are useful.

### Dependencies

| Source | Used for |
|---|---|
| `intelligence.recommendation_events` | Clicks, solves, dismissals, skips |
| `intelligence.user_problem_recommendations` | Score, type, reason, status |
| `app.chat_feedback` | Feedback on explanation quality |
| `app.user_prep_plan_feedback` | Plan quality feedback |
| `user_data.user_coding_problems` | Solve outcomes after recommendation |

### Optional output table

`intelligence.recommendation_snapshots`

### Suggested snapshot fields

| Field | Meaning |
|---|---|
| `id` | Snapshot ID |
| `snapshot_date` | Date |
| `model_version_id` | Scoring version |
| `recommendations_created` | Count created |
| `recommendations_shown` | Count shown |
| `recommendations_clicked` | Count clicked |
| `recommendations_solved` | Count solved |
| `recommendations_dismissed` | Count dismissed |
| `click_rate` | Click-through rate |
| `solve_rate` | Solve conversion rate |
| `dismiss_rate` | Dismiss percentage |
| `already_solved_skip_rate` | Bad recommendation signal |
| `avg_generation_latency_ms` | Generation performance |
| `llm_calls_count` | LLM usage count |
| `llm_cost_estimate` | Estimated LLM cost |
| `created_at` | Snapshot creation time |

This table is optional for the first version but useful for product reporting.

---

## 20. Recommendation Types

Every recommendation should have a clear type.

| Type | Meaning |
|---|---|
| `standard_practice` | Important unsolved standard problem |
| `weak_topic_practice` | Problem from a weak topic |
| `failed_submission_retry` | Problem user attempted but did not solve |
| `revision` | Previously solved important problem due for review |
| `similar_problem` | Similar to recently solved problem |
| `next_difficulty` | Next step in difficulty progression |
| `company_targeted` | Relevant to target company or role |
| `daily_plan` | Part of today’s recommended plan |
| `reorientation` | Restart path for inactive/returning user |

This allows the UI, notification system, and analytics to treat recommendation types differently.

---

## 21. Recommendation Refresh Triggers

The recommendation layer should refresh recommendations in these cases:

| Trigger | AI needed every time? | Expected behavior |
|---|---:|---|
| Scheduled daily refresh | No | Backend scoring generates/stores recommendations |
| After profile sync | No | Recompute user features and refresh recommendations |
| After new submission | No | Update features and generate progression/revision changes |
| After failed attempt | No | Generate retry/weak-topic candidates if useful |
| After solve event | No | Remove solved active recs, create next-step or revision path |
| Explicit user request | Only if needed | Refresh backend recommendations; optionally use LLM for explanation |

The default recommendation refresh should be deterministic backend processing. LLM calls should be reserved for explanations, summaries, and user-facing coaching.

---

## 22. Daily Recommendation Bundle

The daily recommendation surface should not simply show the top five highest-scoring problems. It should produce a balanced practice bundle.

Recommended daily bundle shape:

| Slot | Purpose |
|---|---|
| Warm-up | Easier or familiar topic problem |
| Main practice 1 | Highest ROI weak-topic or company-relevant problem |
| Main practice 2 | Important medium-level or progression problem |
| Revision | Previously solved important problem due for review |
| Optional stretch | Harder or advanced problem if user is ready |

The exact bundle can vary based on user state.

Examples:

| User state | Bundle behavior |
|---|---|
| New user with little data | Use standard important and interview-frequency problems |
| User with failed attempts | Include failed retry candidate |
| User with active target company | Include company-targeted candidate |
| User inactive for 30 days | Start with reorientation and easier restart problem |
| User with stale solved topics | Include revision item |

---

## 23. Scoring Model

The V2 scoring model should be weighted and explainable.

Recommended score components:

| Component | Meaning |
|---|---|
| `importance_score` | Curated or foundational problem importance |
| `interview_frequency_score` | Global interview relevance |
| `company_relevance_score` | Relevance to target company/role/level |
| `topic_weakness_score` | Whether it addresses a weak topic |
| `difficulty_match_score` | Fit with current user ability |
| `failed_attempt_score` | Value of retrying this problem |
| `revision_due_score` | Value of reviewing it now |
| `progression_score` | Whether it is the next natural step |
| `recency_momentum_score` | Whether it builds on recent work |
| `freshness_score` | Whether it is still timely |
| `penalty_score` | Suppression for repetition, dismissal, solved state, poor fit |

The score should always be stored with a breakdown so that the team can debug why a recommendation was shown.

---

## 24. Cooldown and Expiry Rules

Each recommendation should have expiry and cooldown behavior.

| Recommendation type | Suggested expiry |
|---|---:|
| Daily practice | 24 hours |
| Weak topic practice | 7 days |
| Failed retry | 3–7 days |
| Revision | 7–14 days |
| Company targeted | 7–14 days |
| Similar problem | 3 days |
| Reorientation | 3–7 days |

Suggested cooldowns:

| User action | Suggested cooldown |
|---|---:|
| Dismissed | 14–30 days |
| Shown but ignored | 3–7 days |
| Clicked but unsolved | 2–5 days |
| Marked not relevant | 30+ days |
| Solved | Do not recommend again except as future revision |

---

## 25. Backend API Behavior

The main backend should not compute recommendations live.

Recommendation read APIs should:

| Step | Behavior |
|---|---|
| Authenticate user | Ensure user owns request |
| Fetch active recommendations | Read from `intelligence.user_problem_recommendations` |
| Mark shown if needed | Create `recommendation_shown` event |
| Return data | Include problem info, reason, score, type, priority |
| Trigger refresh if stale | Queue async job, do not block response |

The backend should support surfaces such as:

| Surface | Purpose |
|---|---|
| Today | Daily balanced practice |
| Weak topics | Practice by weakness |
| Retry | Attempted-unsolved problems |
| Revision | Due review items |
| Company prep | Target-specific practice |
| Plan | Multi-day or weekly prep |
| Chat | Explanation and coaching layer |

---

## 26. LLM Usage Policy

LLM should be used carefully.

### Do not use LLM for

| Task |
|---|
| Checking solved status |
| Ranking thousands of candidates |
| Calculating topic weakness |
| Applying cooldowns |
| Filtering dismissed recommendations |
| Deduplicating recommendations |
| Normal refresh after every event |
| Notification trigger checks |

### Use LLM for

| Task |
|---|
| Explaining selected top recommendations |
| Packaging daily/weekly plan into readable text |
| Answering “why this recommendation?” |
| Coaching around tradeoffs, time constraints, or alternatives |
| Offline enrichment of problem patterns or common mistakes |

### Rule

Backend decides what to recommend. LLM explains why.

---

## 27. Future-Proofing

The design should allow these future features without major rewrites:

| Future feature | How design supports it |
|---|---|
| Multi-platform coding recommendations | Existing platform + slug model supports this |
| Company-specific plans | Target fields and company relevance scores support this |
| Revision loops | Revision due fields and task progress support this |
| Code-quality analysis | Code blobs can be used by future analysis modules |
| Mistake pattern detection | Submission verdicts and future code analysis can feed features |
| Readiness scoring | Readiness snapshots and user features can be combined |
| A/B testing scoring models | Model version table supports multiple rankers |
| Paid premium recommendations | User tier and recommendation types can drive access |
| Personalized daily OS | Daily bundle design already supports this |
| Notification personalization | Recommendation type and priority can drive notification rules |

---

## 28. Updated Design Summary

The updated recommendation layer should:

| Decision | Summary |
|---|---|
| Reuse current raw data tables | Do not duplicate user/problem/submission/progress data |
| Add only intelligence tables | Add derived features, problem/topic intelligence, recommendations, jobs, events, versions |
| Keep generation async | Use scheduled and event-based jobs |
| Store recommendations | Backend reads stored output quickly |
| Use deterministic scoring | Backend ranking should work without LLM |
| Keep LLM optional | Use for explanation and plan packaging only |
| Track lifecycle events | Measure click, solve, dismiss, skip, expiry |
| Support future features | Design allows readiness, company prep, revision, code analysis, and plan adaptation |

---

## 29. Daily Update Report

### What changed in this updated design

| Area | Update |
|---|---|
| Module clarity | Added detailed module breakdown for all recommendation-layer components |
| Existing table reuse | Clarified which existing tables should be reused instead of duplicated |
| New table scope | Reduced new tables to only intelligence outputs, jobs, events, and model versions |
| User Feature Builder | Added fields and dependencies for user-level recommendation features |
| Topic Intelligence Builder | Added topic-level intelligence fields and purpose |
| Problem Intelligence Builder | Added problem-level intelligence fields and separation from catalog |
| Candidate Generator | Split candidate generation into submodules by recommendation type |
| Scoring Engine | Added score components and output expectations |
| Cooldown/Diversity | Added rules for avoiding repetitive recommendations |
| LLM policy | Clarified that AI is not used on every refresh |
| Production behavior | Reinforced async generation, stored output, and fast read APIs |
| Future-proofing | Added future feature compatibility notes |

### Current recommendation-layer direction

The recommendation layer will be a standalone intelligence layer that reads source-of-truth tables, computes derived features, generates candidates, scores them deterministically, stores recommendations, optionally generates explanations, and tracks user interactions.

The main backend will remain responsible for user-facing APIs, authentication, and serving stored recommendations.

### Open decisions for next step

| Decision | Notes |
|---|---|
| Exact scoring weights | Needs product/engineering agreement |
| First recommendation surfaces | Suggested: Today, Weak Topics, Retry, Revision |
| First model version | Create `v2_default` scoring version |
| Notification integration | Confirm whether existing notification tables are present in active branch |
| Topic taxonomy | Confirm whether a global topic catalog exists or must be created |
| Company frequency data source | Confirm current table/source for company/problem frequencies |

---

## 30. Final Recommendation

The team should not build a large new duplicate recommendation database.

The team should build a focused intelligence layer on top of the existing schema.

Recommended first implementation scope:

| Priority | Build |
|---:|---|
| 1 | `intelligence.user_recommendation_features` |
| 2 | `intelligence.problem_intelligence` |
| 3 | `intelligence.topic_intelligence` |
| 4 | `intelligence.recommendation_model_versions` |
| 5 | `intelligence.recommendation_jobs` |
| 6 | `intelligence.user_problem_recommendations` |
| 7 | `intelligence.recommendation_events` |
| 8 | Recommendation Orchestrator |
| 9 | User Feature Builder |
| 10 | Candidate Generators and Scoring Engine |
| 11 | Diversity/Cooldown Engine |
| 12 | LLM Explanation Generator as optional layer |
| 13 | Metrics and Quality Monitor |

This design keeps the system production-ready, modular, explainable, and scalable while avoiding unnecessary duplication of tables that already exist.
