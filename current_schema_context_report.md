problems
- id: internal problem row id
- leetcode_number: unique LeetCode problem number
- name: problem title
- slug: LeetCode slug
- difficulty: Easy | Medium | Hard
- topics: topic/tag text
- acceptance_rate: acceptance percentage
- description: short problem statement excerpt
- hints: hints text
- url: problem URL
- synced_at: last sync timestamp
- created_at: row creation timestamp

interview_posts
- id: internal post row id
- topic_id: source discussion/post id
- source_url: unique source URL
- source_platform: source platform name
- ingestion_source: ingestion source name
- title: post title
- post_date: original post date
- year: derived post year
- upvote_count: source upvote count
- comment_count: parsed comment count
- company: raw company name
- company_normalized: normalized company name
- company_tier: company tier
- industry: company industry
- role: raw role title
- role_family: normalized role family
- role_level: normalized role level
- role_level_canonical: canonical role level
- role_specialization: role specialization
- location: raw location
- location_country: normalized country
- work_arrangement: onsite | remote | hybrid value
- visa_status_needed: visa/sponsorship signal
- experience_years: candidate years of experience
- yoe_bucket: normalized experience bucket
- education_level: education level signal
- previous_company_tier: previous company tier
- process_channel: referral | recruiter | direct | other channel
- outcome: selected | rejected | offer | unknown outcome
- total_rounds: number of interview rounds
- reapplied: reapplication flag
- reapply_cooldown_months: reapply cooldown duration
- had_competing_offers: competing offers flag
- competing_offers_count: competing offers count
- negotiated: negotiation flag
- time_to_decision_days: time from process start to decision
- post_type: experience | question | compensation
- stage: stub | hydrated | classified | extracted | rejected
- is_experience: AI classification boolean
- quality: AI quality label
- stage_error: pipeline error text
- stage_retries: pipeline retry count
- content_hash: content hash for deduplication
- dedup_status: unique | duplicate status
- hydrated_at: hydration timestamp
- classified_at: classification timestamp
- extracted_at: extraction timestamp
- created_at: row creation timestamp

interview_posts_raw
- post_id: interview_posts id
- raw_content: full scraped post content
- raw_summary: summarized raw content
- archived: archive flag
- archive_url: archived copy URL

interview_posts_comments
- id: internal comment row id
- post_id: interview_posts id
- author: comment author
- content: comment text
- votes: comment vote count
- comment_order: order within post
- created_at: comment timestamp

interview_posts_analysis
- post_id: interview_posts id
- ai_classification: AI classification JSON
- extracted_json: structured extraction JSON
- analysis_md: markdown analysis
- model_used: model name
- tokens_used: token count
- generated_at: analysis timestamp

interview_rounds
- id: internal round row id
- interview_post_id: interview_posts id
- round_order: round sequence number
- round_label: round label/title
- round_type: normalized round type
- design_problem: system/design prompt text
- notes: round notes
- duration_minutes: round duration
- programming_language: language used
- coding_environment: coding environment
- platform: interview platform
- interviewer_role: interviewer role
- interviewer_helpfulness: helpfulness score
- gave_hints: hints-given flag
- round_outcome: pass/fail/unknown round outcome

round_problems
- id: internal row id
- round_id: interview_rounds id
- problem_id: problems id
- problem_description: extracted problem description
- topics: problem topics/tags

round_questions
- id: internal row id
- round_id: interview_rounds id
- question_type: lc_problem | design | behavioral | unknown
- problem_id: problems id
- title: question title
- description: question description
- topics: topic/tag text
- difficulty: Easy | Medium | Hard | unknown
- expected_approach: expected solving approach
- candidate_result: candidate result/outcome

compensations
- id: internal compensation row id
- interview_post_id: interview_posts id
- company_normalized: normalized company
- role_family: role family
- role_level: role level
- location: compensation location
- country: compensation country
- currency: compensation currency
- base_salary: base salary amount
- equity_total: total equity amount
- equity_per_year: annualized equity amount
- signing_bonus: signing bonus amount
- signing_vesting_months: signing bonus vesting months
- yearly_bonus_pct: yearly bonus percentage
- relocation: relocation amount
- total_comp_yearly: yearly total compensation
- first_year_total: first-year total compensation
- stock_vesting_schedule: vesting schedule text
- refresh_expected: refresh grant expected flag
- negotiation_delta: compensation delta from negotiation
- yoe: years of experience
- negotiated: negotiated flag
- post_date: source post date
- year: derived year

post_quotes
- id: internal quote row id
- post_id: interview_posts id
- quote_text: extracted quote
- category: quote category
- sentiment: quote sentiment
- source: quote source
- source_author: quote/comment author
- source_votes: source vote count
- relevance_tags: relevance tags text

post_insights
- id: internal insight row id
- post_id: interview_posts id
- insight_type: insight category
- content: insight text
- metadata: insight metadata JSON/text
- source: insight source
- source_author: source author
- source_votes: source vote count

prep_profiles
- id: internal prep profile row id
- post_id: interview_posts id
- months_preparing: months spent preparing
- hours_per_day_avg: average prep hours per day
- dsa_problems_solved: DSA problem count
- contest_rating_before: contest rating before interview
- mocks_done_count: mock interview count
- mocks_platform: mock interview platform
- used_premium_resources: premium resource usage flag
- had_study_group: study group flag

post_timeline_events
- id: internal timeline event row id
- post_id: interview_posts id
- event_type: application/interview timeline event type
- event_date: event date
- days_from_application: days from application
- notes: timeline notes

technologies
- id: internal technology row id
- name: unique technology name
- category: technology category
- aliases: aliases text/JSON
- mention_count: total mention count

post_technologies
- post_id: interview_posts id
- technology_id: technologies id
- context: mention context
- sentiment: mention sentiment

resources
- id: internal resource row id
- name: unique resource name
- resource_type: resource type
- category: resource category
- url: resource URL
- mention_count: total mention count
- positive_mentions: positive mention count
- negative_mentions: negative mention count

post_resources
- post_id: interview_posts id
- resource_id: resources id
- sentiment: resource mention sentiment
- context: resource mention context

hiring_signals
- id: internal signal row id
- company_normalized: normalized company
- signal_type: signal category
- strength: signal strength
- quote: evidence quote
- post_id: interview_posts id
- signal_date: signal date
- year: signal year

company_profiles
- id: internal company profile row id
- company_normalized: unique normalized company
- display_name: company display name
- tier: company tier
- industry: company industry
- hq_country: headquarters country
- typical_process_length_days: typical process duration
- typical_rounds: typical round count
- sponsor_h1b: H1B sponsorship flag
- remote_friendly_score: remote friendliness score
- offer_rate: offer rate
- total_posts: total related post count
- last_updated: profile update timestamp

interview_posts_fts
- rowid: interview_posts id
- title: searchable post title
- body: searchable post body

users
- id: internal user row id
- email: unique email
- password_hash: password hash
- display_name: display name
- leetcode_username: LeetCode username
- target_companies: target companies text
- target_role: target role
- target_date: target date
- profile_synced_at: profile sync timestamp
- tier: free | premium
- auth_provider: auth provider name
- last_ip: last seen IP address
- country: detected country
- created_at: user creation timestamp

user_solved_problems
- id: internal solved row id
- user_id: users id
- problem_id: problems id
- status: solved status
- scraped_at: scrape timestamp

sync_log
- id: internal sync log row id
- sync_type: sync job type
- status: sync job status
- records_processed: processed count
- records_skipped: skipped count
- error_message: error text
- started_at: sync start timestamp
- completed_at: sync completion timestamp

user_read_posts
- id: internal read row id
- user_id: users id
- post_id: interview_posts id
- read_at: read timestamp

chat_sessions
- id: internal chat session row id
- user_id: users id
- title: chat title
- attached_plan_id: user_prep_plans id
- pt_tracking_enabled: progress tracking flag
- uc_pause_until: user-context extraction pause timestamp
- created_at: session creation timestamp
- updated_at: session update timestamp

chat_messages
- id: internal chat message row id
- session_id: chat_sessions id
- role: user | assistant
- content: message text
- tool_calls: tool call JSON
- tokens_used: token count
- trace_id: observability trace id
- created_at: message timestamp

user_lc_profiles
- id: internal LeetCode profile row id
- user_id: users id
- lc_username: LeetCode username
- total_solved: total solved count
- easy_solved: easy solved count
- medium_solved: medium solved count
- hard_solved: hard solved count
- total_submissions: submission count
- contest_rating: contest rating
- contests_attended: attended contest count
- global_ranking: global contest ranking
- streak: current streak
- active_years: active years JSON
- submission_calendar: submission heatmap JSON
- languages: solved-by-language JSON
- beats_easy: easy percentile
- beats_medium: medium percentile
- beats_hard: hard percentile
- failed_easy: failed easy count
- failed_medium: failed medium count
- failed_hard: failed hard count
- lc_ranking: LeetCode profile ranking
- lc_reputation: LeetCode reputation
- synced_at: sync timestamp
- created_at: row creation timestamp

user_lc_topics
- id: internal topic row id
- user_id: users id
- tag_name: LeetCode tag name
- tag_slug: LeetCode tag slug
- tag_slug_canonical: canonical tag slug
- problems_solved: solved count for tag
- synced_at: sync timestamp

user_contest_history
- id: internal contest history row id
- user_id: users id
- contest_title: contest title
- ranking: contest ranking
- rating: contest rating after contest
- problems_solved: solved count in contest
- contest_date: contest date
- synced_at: sync timestamp

api_logs
- id: internal API log row id
- event_type: event type
- event_data: event payload JSON/text
- user_id: users id or 0
- duration_ms: request duration
- ip_address: request IP address
- created_at: log timestamp

prep_plan_templates
- id: internal template row id
- company: target company
- level_canonical: target level
- role_family: target role family
- duration_days: plan duration
- version: template version
- profile_json: generation profile JSON
- plan_json: full plan JSON
- generator_model: generator model name
- data_snapshot_hash: data snapshot hash
- generated_at: generation timestamp

user_prep_plans
- id: internal user prep plan row id
- user_id: users id
- template_id: prep_plan_templates id
- started_at: plan start timestamp
- target_date: target completion/interview date
- status: active | paused | completed | abandoned
- template_version_seen: template version seen by user
- hours_per_day_override: user hours-per-day override
- paused_at: pause timestamp

user_prep_task_progress
- id: internal task progress row id
- user_prep_plan_id: user_prep_plans id
- day_number: plan day number
- task_ref: task reference id
- status: pending | done | hinted | skipped
- time_spent_minutes: time spent
- note: user note
- revision_due_date: spaced revision due date
- completed_at: completion timestamp

user_readiness_snapshots
- id: internal readiness snapshot row id
- user_prep_plan_id: user_prep_plans id
- computed_at: computation timestamp
- readiness_score: readiness score
- component_breakdown: component breakdown JSON/text
- days_elapsed: elapsed days
- tasks_done: completed task count
- tasks_total: total task count

prep_unknown_round_types
- round_type: unknown round type value
- count: occurrence count
- first_seen: first seen timestamp
- last_seen: last seen timestamp
- sample_company: sample company
- notes: notes

user_prep_plan_feedback
- id: internal feedback row id
- user_prep_plan_id: user_prep_plans id
- rating: -1 | 1
- text: feedback text
- created_at: feedback timestamp

pt_areas
- id: internal area row id
- slug: unique area slug
- name: area display name
- display_order: display order

pt_topics
- id: internal topic row id
- area_id: pt_areas id
- parent_topic_id: parent pt_topics id
- slug: unique topic slug
- name: topic display name
- description: topic description
- topic_type: module | concept | subtopic | applied_problem | checkpoint | lc_problem
- difficulty: beginner | intermediate | senior | staff
- importance_score: topic importance score
- is_dashboard_visible: dashboard visibility flag
- display_order: display order
- verification_type: none | quick_check | mini_quiz | interview_checkpoint | mock_design
- verification_required: verification required flag
- is_provisional: provisional topic flag
- provisional_first_seen_at: first provisional detection timestamp
- provisional_user_count: provisional user count
- provisional_avg_confidence: provisional confidence average
- promoted_at: promotion timestamp
- promoted_reason: promotion reason
- created_at: creation timestamp
- updated_at: update timestamp

pt_topic_aliases
- id: internal alias row id
- topic_id: pt_topics id
- alias: alias text
- source: manual | generated | user_query
- created_at: alias creation timestamp

pt_topic_dependencies
- id: internal dependency row id
- topic_id: pt_topics id
- depends_on_topic_id: dependency pt_topics id
- dependency_type: prerequisite | related | unlocks
- strength: dependency strength score

pt_topic_level_expectations
- id: internal expectation row id
- topic_id: pt_topics id
- level: l4 | l5 | staff
- expectations_json: expectations JSON
- required_depth: aware | explain | defend | design
- importance_weight: importance weight
- created_by: generated | reviewed | expert_verified
- generated_at: generation timestamp
- reviewed_at: review timestamp

pt_sources
- id: internal source row id
- source_key: unique source key
- title: source title
- source_type: book | blog | paper | docs | video | interview_post | user_generated
- author: source author
- url: source URL
- reputation_score: reputation score
- license_policy: license policy
- notes: source notes
- created_at: source creation timestamp

pt_topic_sources
- id: internal topic source row id
- topic_id: pt_topics id
- source_id: pt_sources id
- usage_type: canonical | supporting | interview_evidence | reference_only
- summary: source summary
- tradeoffs: source tradeoffs
- confidence: linkage confidence

pt_user_targets
- id: internal target row id
- user_id: users id
- company: target company
- level: l4 | l5 | staff
- role_family: role family
- is_active: active target flag
- is_primary: primary target flag
- priority: target priority
- created_at: target creation timestamp
- activated_at: activation timestamp

pt_user_topic_progress
- id: internal progress row id
- user_id: users id
- topic_id: pt_topics id
- status: not_started | covered | attempted | verified
- coverage_score: coverage score
- readiness_score: readiness score
- covered_depth: l4 | l5 | staff
- verified_depth: l4 | l5 | staff
- confidence_score: confidence score
- quiz_best_score: best quiz score
- attempt_count: attempt count
- first_seen_at: first seen timestamp
- last_seen_at: last seen timestamp
- last_practiced_at: last practice timestamp
- last_revised_at: last revision timestamp
- next_revision_at: next revision timestamp
- snoozed_until: snooze expiration timestamp
- last_nudged_at: last nudge timestamp
- nudge_count: nudge count
- incomplete_checkpoints_json: incomplete checkpoints JSON
- covered_subtopics_json: covered subtopics JSON
- missing_subtopics_json: missing subtopics JSON
- created_at: creation timestamp
- updated_at: update timestamp

pt_learning_events
- id: internal learning event row id
- user_id: users id
- session_id: chat session id/text
- message_id: chat message id/text
- event_type: taught | covered | checkpoint_started | checkpoint_passed | checkpoint_failed | manual_mark_covered | self_reported_known | revised | struggled | skipped | snoozed | plan_task_completed | target_changed | calibration_self_report
- area_slug: pt_areas slug
- topic_id: pt_topics id
- subtopic_ids_json: subtopic ids JSON
- depth_inferred: aware | explain | defend | design
- coverage_delta: coverage score delta
- readiness_delta: readiness score delta
- confidence: extraction confidence
- quality_weight: quality weight
- response_quality: success | partial | failed
- source: extraction | manual | checkpoint | plan_task | calibration | import
- metadata_json: event metadata JSON
- idempotency_key: unique idempotency key
- shadow: shadow-mode flag
- invalidated_at: invalidation timestamp
- invalidated_reason: invalidation reason
- invalidated_by_event_id: invalidating pt_learning_events id
- created_at: event timestamp

pt_checkpoint_attempts
- id: internal checkpoint attempt row id
- user_id: users id
- topic_id: pt_topics id
- checkpoint_topic_id: checkpoint pt_topics id
- level: l4 | l5 | staff
- score: checkpoint score
- passed: pass flag
- questions_json: questions JSON
- answers_json: answers JSON
- weak_subtopic_ids_json: weak subtopics JSON
- evaluator_feedback: evaluator feedback
- duration_ms: attempt duration
- created_at: attempt timestamp

pt_extraction_jobs
- id: internal extraction job row id
- user_id: users id
- session_id: chat session id/text
- message_id: unique chat message id/text
- status: pending | running | done | failed | skipped
- attempts: attempt count
- max_attempts: max attempt count
- retry_after: retry timestamp
- last_error: last error text
- trace_id: observability trace id
- queued_at: queue timestamp
- started_at: start timestamp
- completed_at: completion timestamp

pt_plan_task_topic_map
- id: internal plan-topic map row id
- template_id: prep_plan_templates id
- day_number: plan day number
- task_ref: plan task reference
- topic_id: pt_topics id
- contribution_weight: topic contribution weight

pt_decisions
- id: internal decision row id
- user_id: users id
- decided_at: decision timestamp
- primary_topic_id: pt_topics id
- primary_action_type: learn | revise | continue | checkpoint | plan_task | none
- reason_code: P1_REVISION | P2_INCOMPLETE | P3_PLAN | P4_GRAPH_UNLOCKED | P5_REDIRECT | EMPTY
- reason_text: decision reason
- alternates_json: alternate decisions JSON
- inputs_snapshot_json: input snapshot JSON
- presented: presented flag
- clicked: clicked flag

pt_extraction_blocklist
- id: internal blocklist row id
- user_id: users id
- topic_id: pt_topics id
- expires_at: expiration timestamp
- reason: blocklist reason
- created_at: creation timestamp

pt_topic_search
- rowid: internal FTS row id
- slug: topic slug
- name: searchable topic name
- aliases: searchable aliases
- description: searchable description
- keywords: searchable keywords

user_context_events
- id: internal context event row id
- user_id: users id
- session_id: chat session id/text
- message_id: chat message id/text
- event_type: context event type
- subject: self | not_self
- confidence: extraction confidence
- source: extraction | manual | chat_feedback | leetcode_sync | import
- extracted_json: extracted context JSON
- schema_version: schema version
- response_quality: success | partial | failed | off_topic | too_short | truncated
- invalidated_at: invalidation timestamp
- invalidated_reason: invalidation reason
- invalidated_by_event_id: invalidating user_context_events id
- invalidated_by_user_action: user invalidation action
- idempotency_key: unique idempotency key
- importance_score: event importance score
- ttl_days: time-to-live days
- shadow: shadow-mode flag
- created_at: event timestamp

user_profile_summary
- user_id: users id
- current_company: current company
- years_experience: years of experience
- current_role: current role
- primary_target_id: pt_user_targets id
- strongest_domains_json: strongest domains JSON
- weakest_domains_json: weakest domains JSON
- preferred_learning_style: learning style preference
- preferred_depth: depth preference
- preferred_response_length: response length preference
- current_focus_domain: current focus domain
- current_focus_topic_slug: current focus topic
- last_active_at: last active timestamp
- emotional_state: emotional state label
- emotional_state_expires_at: emotional state TTL timestamp
- recent_interview_events_json: recent interview events JSON
- known_company_signals_json: known company signals JSON
- provenance_json: field provenance JSON
- rebuilt_at: rebuild timestamp
- rebuild_reason: rebuild reason
- schema_version: schema version

uc_forget_audit
- id: internal forget audit row id
- user_id: users id
- command_text: forget command text
- resolved_intent: resolved forget intent
- invalidated_event_ids: invalidated event ids JSON/text
- created_at: audit timestamp

user_memory
- id: internal memory row id
- user_id: users id
- session_id: chat session id
- message_id: chat message id
- raw_text: original memory text
- summary: memory summary
- tags_json: memory tags JSON
- importance_score: memory importance score
- ttl_days: time-to-live days
- invalidated_at: invalidation timestamp
- invalidated_reason: invalidation reason
- source: extraction | manual | import
- shadow: shadow-mode flag
- created_at: memory creation timestamp

user_memory_embeddings
- memory_id: user_memory id
- embedding: 384-dimensional vector embedding

knowledge_sources
- id: internal knowledge source row id
- area: pt_areas slug
- source_type: paper | eng_blog | book | docs | article | video | personal_blog
- source_format: source format label
- publisher: publisher name
- authors_json: authors JSON
- title: source title
- url: unique source URL
- published_at: publication date
- last_checked_at: freshness check timestamp
- is_deprecated: deprecated flag
- reputation: reputation score
- is_citable: citability flag
- content_hash: source content hash
- raw_object_key: raw object storage key
- metadata_json: source metadata JSON
- ingested_at: ingestion timestamp

knowledge_chunks
- id: internal knowledge chunk row id
- source_id: knowledge_sources id
- area: pt_areas slug
- chunk_type: chunk type label
- topic: chunk topic
- content: normalized chunk content
- raw_chunk_text: original chunk text
- keywords_json: keywords JSON
- section_title: section title
- heading_path: heading path
- page_start: starting page
- page_end: ending page
- chunk_index: order within source
- prev_chunk_id: previous knowledge_chunks id
- next_chunk_id: next knowledge_chunks id
- has_table: table-present flag
- has_code_block: code-block-present flag
- has_figure: figure-present flag
- content_hash: chunk content hash
- extractor: extractor name
- extractor_version: extractor version
- prompt_version: prompt version
- prompt_hash: prompt hash
- extracted_at: extraction timestamp
- extraction_status: success | failed | needs_review | skipped
- extraction_confidence: extraction confidence
- verification_status: pass | warn | fail
- verification_warnings: verification warnings JSON
- metadata_json: extraction metadata JSON
- metadata_normalized_json: normalized extraction metadata JSON
- metadata_schema_version: metadata schema version
- created_at: chunk creation timestamp

knowledge_chunks_vec
- chunk_id: knowledge_chunks id
- embedding: 1024-dimensional vector embedding

knowledge_chunks_fts
- rowid: knowledge_chunks id
- content: searchable chunk content
- topic: searchable topic
- keywords_json: searchable keywords JSON

knowledge_extraction_audit
- id: internal audit row id
- chunk_id: knowledge_chunks id
- raw_chunk_text: audited raw chunk text
- llm_response: audited LLM response
- prompt_used: exact prompt text
- prompt_hash: prompt hash
- extractor: extractor name
- extractor_version: extractor version
- metadata_schema_version: metadata schema version
- embedding_model: embedding model name
- reranker_model: reranker model name
- warnings_json: warnings JSON
- extraction_status: extraction status
- created_at: audit timestamp
