# MVP Recommendation System Tables

## 1. `catalog.dsa_sheet_problems`

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

  source_name text,
  source_url text,

  metadata jsonb not null default '{}'::jsonb,

  is_active boolean not null default true,

  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),

  unique (sheet_name, sheet_version, topic_slug, problem_order),

  foreign key (platform, problem_slug)
    references catalog.coding_problems(platform, slug)
);
```

---

## 2. `app.recommendation_events`

```sql
create table app.recommendation_events (
  id bigserial primary key,

  user_id uuid not null references public.users(id) on delete cascade,

  event_type text not null
    check (
      event_type in (
        'shown',
        'clicked',
        'accepted',
        'skipped',
        'dismissed',
        'completed',
        'expired'
      )
    ),

  platform text not null default 'leetcode',
  problem_slug text,

  sheet_name text default 'default_dsa_sheet',
  sheet_version text default 'v1',

  topic_slug text,
  topic_name text,

  reason text,
  score numeric(8,4),

  metadata jsonb not null default '{}'::jsonb,

  created_at timestamptz not null default now(),

  foreign key (platform, problem_slug)
    references catalog.coding_problems(platform, slug)
);
```

---

## 3. `app.user_active_recommendations`

```sql
create table app.user_active_recommendations (
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
        'weak_topic',
        'revision_due',
        'next_in_topic',
        'daily_practice'
      )
    ),

  reason text not null,

  score numeric(8,4) not null default 0,
  rank integer not null,

  status text not null default 'active'
    check (
      status in (
        'active',
        'shown',
        'dismissed',
        'completed',
        'expired'
      )
    ),

  generated_at timestamptz not null default now(),
  shown_at timestamptz,
  expires_at timestamptz,

  metadata jsonb not null default '{}'::jsonb,

  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),

  unique (user_id, platform, problem_slug, recommendation_type),

  foreign key (platform, problem_slug)
    references catalog.coding_problems(platform, slug)
);
```

---

## 4. `app.projector_cursors` Optional

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
