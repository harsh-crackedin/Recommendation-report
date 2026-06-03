# CrackedIn — Recommendation System Design

**Last updated:** 2026-06-03
**Status:** V1 spec — focused on what ships with extension launch

---

## Product Model

One chat. One brain. The recommendation engine runs behind the chat.

**Triggers:**
1. **First sync (immediate)** — analyze last 60 submissions → quick popup recommendation
2. **Full sync complete (background)** — full profile analysis → enriched recommendations
3. **Daily email (cron)** — company-specific recommendation → hooks user back
4. **In-app notification** — same as email, in-app badge
5. **User asks** — "what should I solve next?" / "I'm done" / "I know 20 DP problems, what topic next?"

**Output format:** Problem card in chat with `[Solve →]` `[Done ✅]` `[Skip ⏭]` `[Already know 🙈]`

---

## Data Sources

### MVP (ships now):
- **User LC history** — from extension sync (`user_data.user_coding_submissions`)
- **Static curated sheets** — Striver / NeetCode / Blind 75 (200-400 problems, topic-grouped)
- **1p3a frequency data** — company question bank (already downloaded, 400+ companies)

### Phase 2 (after scraping pipeline):
- **Interview experience corpus** — 134k threads with round details
- **Company process guides** — 32 guides already scraped
- **Cross-source frequency** — GFG, Reddit, LC Discuss

---

## Recommendation Categories

### CORE — MVP (ships with extension)

#### 1. Next Problem Recommendation
*"What should I solve next?"*

| Signal | Source | Logic |
|---|---|---|
| User's solved set | Extension sync | Exclude already solved |
| Topic coverage gaps | Solved vs curated sheet | Prefer uncovered topics |
| Problem order in sheet | Striver/NeetCode ordering | Follow curated sequence |
| Company frequency | 1p3a question bank | Boost problems asked at target company |
| Difficulty band | User's current level | Don't jump beginner → hard |

**Scoring (V1 — rule-based):**
```
score = 0
if problem.topic is user's weakest:     score += 40
if problem is in target company top 20:  score += 30
if problem is next in curated sequence:  score += 20
if problem.difficulty matches user band: score += 10
if problem was skipped before:           score -= 50
if problem was dismissed:                score = -999 (never show)
```

---

#### 2. Coverage Gap Recommendation
*"You haven't touched Graph/BFS at all"*

| Signal | Source | Logic |
|---|---|---|
| Topics in curated sheet | Static sheet data | List all expected topics |
| User's solved topics | Extension sync | Mark covered vs uncovered |
| Company importance | 1p3a frequency | Rank gaps by interview relevance |

**Output:** "You've covered 8/15 topics. Missing: Graph/BFS (asked 47x), Sliding Window (asked 23x), Trie (asked 12x)"

---

#### 3. Weakness Detection
*"You struggle with DP"*

| Signal | Source | Logic |
|---|---|---|
| Submission count per problem | Extension sync | >5 submissions = struggle |
| Fail rate by topic | Aggregate submissions | Low AC% in a topic = weakness |
| Recent failures | Last 60 submissions | Immediate signal |

**Output:** "Your DP accuracy is 40% (vs 75% average). You took 8 attempts on Coin Change. Here's an easier DP to rebuild confidence."

---

#### 4. Company-Specific Gap
*"Meta asked these 12, you've done 2"*

| Signal | Source | Logic |
|---|---|---|
| Company hot questions | 1p3a frequency data | Top questions by frequency + recency |
| User's solved set | Extension sync | Cross-reference |
| Freshness | reported_to / last_asked | Prefer this quarter's data |

**Output:** Problem card with "Why: Meta asked this 11x this quarter. You haven't solved it."

---

#### 5. Readiness Score
*"How ready am I for Google?"*

| Signal | Source | Logic |
|---|---|---|
| Company top 20 questions | 1p3a frequency | The bar |
| User's solved overlap | Extension sync | What they've hit |
| Topic coverage | Sheet vs solved | Breadth check |
| Weakness areas | Submission patterns | Risk areas |

**Output:** "Google L5 readiness: 38%. You've solved 3/10 of their most-asked. Biggest gap: Graph/BFS."

---

#### 6. Behavioral Nudge
*"You're avoiding hard problems"*

| Signal | Source | Logic |
|---|---|---|
| Difficulty distribution | Extension sync | % easy/medium/hard |
| Topic distribution | Recent 30 solves | Topic concentration |
| Activity gaps | Submission timestamps | Days since last solve |
| Submission pattern | Attempts per problem | Getting worse or better |

**Output:** "Last 2 weeks: 80% Arrays, 0% Graphs. You're avoiding your weak areas."

---

### IMPORTANT — Phase 2 (after scraping corpus)

#### 7. Interview Experience Intelligence
*"What's the Google onsite actually like?"*

| Signal | Source | When |
|---|---|---|
| Company interview process | 1p3a guides (already have) | User asks or sets target |
| Round-by-round breakdown | Experience corpus | User asks |
| Recent changes | Trend reports | Proactive notification |

---

#### 8. Follow-up Prediction
*"They'll ask: what if the input is a stream?"*

| Signal | Source | When |
|---|---|---|
| Common follow-ups per problem | Experience corpus analysis | After user solves a problem |
| Company-specific variants | Thread data | When target company set |

---

#### 9. Dynamic Plan Adjustment
*"Interview moved up — here's your new plan"*

| Signal | Source | When |
|---|---|---|
| Interview date | User-provided | User updates timeline |
| Current progress vs plan | Solved set vs recommendations shown | Ongoing |
| New weakness detected | Recent failures | Real-time |

---

### OPTIONAL — Phase 3+ (needs user base / outcome data)

| # | Category | What it does | Needs |
|---|---|---|---|
| 10 | Success path mining | "People who passed Google solved these" | 100+ outcome reports |
| 11 | Code quality analysis | "Your solution is O(n²), there's O(n)" | Code content from extension |
| 12 | Communication coaching | "Practice explaining WHY before coding" | Mock interview feature |
| 13 | System design gaps | "You haven't practiced caching, load balancing" | SD content + tracking |
| 14 | Mock interview readiness | "You've solved 80+, time for mocks" | Progress thresholds |
| 15 | Confidence/risk assessment | "70% pass probability at your level" | Statistical model from outcomes |
| 16 | Resume analysis | "Your resume reads as L4, target is L5" | Resume upload feature |
| 17 | Revision (spaced repetition) | "You solved this 45 days ago, time to revisit" | Timestamp analysis |
| 18 | Social/cohort | "47 others prepping for same company" | User base growth |
| 19 | Career timing | "Apply now — your readiness is above pass threshold" | Outcome model |
| 20 | Pre-interview plan | "5 days to onsite: Day 1 do X, Day 2 do Y" | Interview date + gaps |

---

## Database Tables

### Event Ledger (append-only, never update/delete)

```sql
-- What recommendations we showed and what user did
CREATE TABLE app.recommendation_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES public.users(id),
    event_type      TEXT NOT NULL, -- 'shown', 'accepted', 'completed', 'skipped', 'dismissed'
    recommendation_id UUID, -- groups events for same recommendation instance
    problem_platform TEXT, -- 'leetcode'
    problem_slug    TEXT, -- 'number-of-islands'
    topic_id        UUID, -- if topic-level recommendation
    reason          TEXT, -- 'weak_topic', 'company_hot', 'coverage_gap', 'next_in_sequence'
    dismiss_reason  TEXT, -- 'already_know', 'not_relevant', 'too_easy', 'too_hard'
    source_sheet    TEXT, -- 'striver', 'neetcode', 'blind75', 'company_frequency'
    metadata        JSONB, -- any extra context (score, rank, company, etc.)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Indexes for the ledger
CREATE INDEX idx_rec_events_user_time ON app.recommendation_events(user_id, created_at);
CREATE INDEX idx_rec_events_user_type ON app.recommendation_events(user_id, event_type);
CREATE INDEX idx_rec_events_problem ON app.recommendation_events(user_id, problem_slug);
```

---

### Active Recommendations (serving table, mutable, rebuildable)

```sql
-- What to show user RIGHT NOW (precomputed, fast reads)
CREATE TABLE app.user_active_recommendations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES public.users(id),
    recommendation_id UUID NOT NULL UNIQUE, -- stable ID for tracking
    rec_type        TEXT NOT NULL, -- 'next_problem', 'coverage_gap', 'weakness', 'company_hot', 'behavioral'
    problem_platform TEXT,
    problem_slug    TEXT,
    topic_id        UUID,
    score           REAL NOT NULL DEFAULT 0, -- ranking score
    reason_short    TEXT NOT NULL, -- "Asked 47x at FAANG. You haven't done Graph/BFS."
    reason_detail   JSONB, -- full explanation data
    status          TEXT NOT NULL DEFAULT 'active', -- 'active', 'shown', 'snoozed', 'expired'
    source_sheet    TEXT, -- which sheet this came from
    company_slug    TEXT, -- if company-specific
    generated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ,
    
    CONSTRAINT unique_active_per_problem UNIQUE(user_id, problem_slug, status)
);

-- Fast reads for serving
CREATE INDEX idx_active_recs_user ON app.user_active_recommendations(user_id, status, score DESC);
```

---

### Curated Sheets (static catalog — Striver, NeetCode, Blind 75)

```sql
-- The problem sheets we recommend from
CREATE TABLE catalog.curated_sheets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sheet_name      TEXT NOT NULL, -- 'striver_sde', 'neetcode_150', 'blind_75'
    sheet_version   TEXT NOT NULL DEFAULT '1.0',
    topic_name      TEXT NOT NULL, -- 'Array', 'Graph/BFS', 'Dynamic Programming'
    topic_slug      TEXT NOT NULL, -- 'array', 'graph-bfs', 'dp'
    problem_title   TEXT NOT NULL, -- 'Number of Islands'
    problem_slug    TEXT NOT NULL, -- 'number-of-islands'
    problem_platform TEXT NOT NULL DEFAULT 'leetcode',
    problem_order   INT NOT NULL, -- sequence within topic
    difficulty      TEXT, -- 'easy', 'medium', 'hard'
    is_active       BOOLEAN DEFAULT true,
    created_at      TIMESTAMPTZ DEFAULT now(),
    
    CONSTRAINT unique_sheet_problem UNIQUE(sheet_name, problem_platform, problem_slug)
);

-- Find problems by topic
CREATE INDEX idx_sheets_topic ON catalog.curated_sheets(sheet_name, topic_slug, problem_order);
```

---

### Company Frequency (from 1p3a question bank)

```sql
-- What companies actually ask (scraped data)
CREATE TABLE catalog.company_questions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_slug    TEXT NOT NULL, -- 'google', 'meta', 'amazon'
    company_name    TEXT NOT NULL,
    question_title  TEXT NOT NULL, -- '1p3a title: Onsite Coding: LRU Cache'
    question_slug   TEXT, -- their slug
    category        TEXT, -- 'coding', 'system-design', 'behavioral'
    frequency       TEXT, -- 'very-high', 'high', 'medium', 'low'
    source_count    INT, -- how many reports confirm this
    reported_from   DATE, -- earliest report
    reported_to     DATE, -- latest report
    last_asked      DATE,
    lc_problem_slug TEXT, -- mapped LC problem if known (e.g., 'number-of-islands')
    roles           TEXT[], -- ['swe', 'mle']
    stages          TEXT[], -- ['onsite-coding', 'phone-screen']
    tags            TEXT[], -- ['graph', 'bfs', 'medium']
    source          TEXT DEFAULT '1p3a', -- where we got this data
    raw_data        JSONB, -- full original response preserved
    created_at      TIMESTAMPTZ DEFAULT now(),
    updated_at      TIMESTAMPTZ DEFAULT now(),
    
    CONSTRAINT unique_company_question UNIQUE(company_slug, question_slug, source)
);

-- Find hot questions for a company
CREATE INDEX idx_company_q_hot ON catalog.company_questions(company_slug, frequency, last_asked DESC);
-- Find questions by LC problem (for gap analysis)
CREATE INDEX idx_company_q_lc ON catalog.company_questions(lc_problem_slug) WHERE lc_problem_slug IS NOT NULL;
```

---

### User Recommendation Preferences (simple settings)

```sql
CREATE TABLE app.user_rec_preferences (
    user_id         UUID PRIMARY KEY REFERENCES public.users(id),
    target_companies TEXT[] DEFAULT '{}', -- ['google', 'meta']
    target_role     TEXT, -- 'swe', 'mle', 'frontend'
    target_level    TEXT, -- 'l4', 'l5', 'senior'
    goal            TEXT DEFAULT 'general', -- 'general', 'company_specific', 'revision'
    daily_email     BOOLEAN DEFAULT true,
    notifications   BOOLEAN DEFAULT true,
    disabled_types  TEXT[] DEFAULT '{}', -- ['behavioral'] if user opts out
    interview_date  DATE, -- if set, affects plan urgency
    updated_at      TIMESTAMPTZ DEFAULT now()
);
```

---

## How the Recommendation Engine Works (V1)

### On First Sync (last 60 submissions):

```python
def quick_recommendation(user_id, last_60_submissions):
    # 1. Identify solved problems
    solved_slugs = {s.slug for s in last_60_submissions if s.verdict == 'AC'}
    
    # 2. Identify topics covered
    covered_topics = get_topics_for_slugs(solved_slugs)
    
    # 3. Find biggest gap in curated sheet
    all_topics = get_sheet_topics('striver_sde')
    uncovered = [t for t in all_topics if t not in covered_topics]
    
    # 4. Pick first problem from biggest gap topic
    # Boost if target company asks it frequently
    target = get_user_target_companies(user_id)
    candidates = get_unsolved_from_sheet(user_id, uncovered[0])
    
    if target:
        candidates.sort(key=lambda p: company_frequency(p, target), reverse=True)
    
    return candidates[0]  # Show as problem card
```

### On "What should I solve next?" (full engine):

```python
def full_recommendation(user_id):
    prefs = get_preferences(user_id)
    solved = get_all_solved(user_id)
    dismissed = get_dismissed_slugs(user_id)
    recently_skipped = get_recent_skips(user_id, days=7)
    
    candidates = []
    
    # Source 1: Curated sheet gaps
    for problem in get_sheet_unsolved(user_id, 'striver_sde'):
        if problem.slug in dismissed: continue
        score = 20  # base: in curated sheet
        if problem.topic in user_weak_topics(user_id): score += 40
        if problem.slug in recently_skipped: score -= 30
        candidates.append((problem, score, 'coverage_gap'))
    
    # Source 2: Company hot questions
    if prefs.target_companies:
        for q in get_company_hot_questions(prefs.target_companies):
            if q.lc_problem_slug in solved: continue
            if q.lc_problem_slug in dismissed: continue
            score = 30 + (q.source_count * 2)  # more reports = higher score
            candidates.append((q, score, 'company_hot'))
    
    # Source 3: Weakness reinforcement
    for topic in user_weak_topics(user_id, top_n=3):
        easier_problem = get_easier_unsolved(user_id, topic)
        if easier_problem:
            candidates.append((easier_problem, 35, 'weakness'))
    
    # Sort by score, pick top
    candidates.sort(key=lambda x: x[1], reverse=True)
    
    # Store in active recommendations + log event
    winner = candidates[0]
    store_active_recommendation(user_id, winner)
    log_recommendation_event(user_id, winner, 'shown')
    
    return winner
```

### Daily Email Cron:

```python
def daily_email_job():
    for user in get_users_with_email_enabled():
        if not user.target_companies:
            # General recommendation
            rec = full_recommendation(user.id)
        else:
            # Company-specific: pick from their target's hot list
            hot = get_company_hot_unsolved(user.id, user.target_companies[0])
            rec = hot[0] if hot else full_recommendation(user.id)
        
        send_email(user.email, template='daily_recommendation', data=rec)
        log_recommendation_event(user.id, rec, 'shown', channel='email')
```

---

## What This Achieves

### V1 (ships with extension):
- User syncs LC → instant recommendation in 5 seconds
- "What should I solve next?" → personalized answer based on gaps + company data
- Daily email → retention hook
- Readiness score → "38% ready for Google"
- All from: Striver sheet + 1p3a frequency + user's solved set

### V2 (after scraping loaded):
- "What's the Google onsite like?" → real data from 138 reports
- "What follow-ups do they ask for LRU Cache?" → from experience corpus
- Process changes → "Meta added AI-native coding in April"
- Richer scoring using temporal frequency (this quarter vs last year)

### V3 (after user base grows):
- "People who passed solved these 30" → outcome-based recommendations
- Code quality feedback → "Your solution works but is O(n²)"
- Mock interview suggestions → "You're ready, time to simulate"
- Confidence scoring → "70% pass probability"

---

## Table Summary

| Table | Type | Purpose | MVP? |
|---|---|---|---|
| `app.recommendation_events` | Ledger (append-only) | Track what we showed + user response | ✅ |
| `app.user_active_recommendations` | Serving (mutable) | What to show right now | ✅ |
| `catalog.curated_sheets` | Static catalog | Striver/NeetCode/Blind75 problems | ✅ |
| `catalog.company_questions` | Scraped catalog | 1p3a frequency data | ✅ |
| `app.user_rec_preferences` | User settings | Target company, goal, opt-outs | ✅ |

These 5 tables + existing `user_coding_submissions` + existing `user_coding_problems` = complete V1 recommendation system.

---

## Open Questions

1. Should we pre-generate N recommendations per user (batch) or generate on-demand per request?
2. How often does the daily email run? Morning local time? Fixed UTC?
3. When user has multiple target companies, do we rotate daily or let them choose?
4. Should "Already know this" require any proof, or trust the user?
5. Rate limit: max how many recommendations per day in chat?
