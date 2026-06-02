# Simple Design Report: MVP Coding Recommendation System

## 1. What We Are Building

We are building a simple MVP recommendation system that suggests coding questions to users based on:

- Their LeetCode submission history
- Their weak topics
- Topics they have not practiced recently
- A curated DSA question sheet prepared by us

The goal is not to build a complex recommendation engine in the first version. The goal is to create a clean architecture where user activity is stored, processed, and converted into useful recommendations.

The system will recommend questions from a precompiled DSA sheet that contains around 200-400 selected questions. These questions will be grouped under topics such as Arrays, Strings, Binary Search, Two Pointers, Dynamic Programming, Graphs, BFS, DFS, Trees, and Greedy.

Each question in the sheet will point to an existing problem in the shared LeetCode problem catalog.

---

## 2. What Data We Are Using

The recommendation system will use four main types of data.

### 2.1 Shared Problem Catalog

This already exists in the database.

It contains the full LeetCode problem data such as platform, problem slug, title, difficulty, topic tags, acceptance rate, and problem metadata.

We will not duplicate this data.

The curated sheet will only store a reference to the problem catalog using the problem slug and platform.

### 2.2 Curated DSA Sheet Data

We will create one simple curated sheet table.

This table will contain only the questions we want to use for recommendations.

It will store:

- Sheet name
- Sheet version
- Topic name
- Topic slug
- Problem title
- Problem slug
- Problem order inside the topic
- Source name or source URL, if available
- Active/inactive status

This table does not replace the full problem catalog. It only defines which problems are part of our recommendation sheet and how they are grouped.

Example:

```text
Topic: Arrays
1. Two Sum
2. Best Time to Buy and Sell Stock
3. Product of Array Except Self

Topic: Dynamic Programming
1. Climbing Stairs
2. House Robber
3. Coin Change
```

The curated sheet acts as the recommendation pool.

### 2.3 User Submission Data

The system will use the user's LeetCode submission history.

This data tells us:

- Which problems the user attempted
- Which problems the user solved
- Which problems the user failed
- When the user last attempted or solved a problem
- How many times the user attempted a problem

This is the main behavior signal used to understand user progress.

### 2.4 User Topic Progress Data

The system will use topic-level progress data to understand the user's strength or weakness in each topic.

This data helps answer:

- Is the user weak in Dynamic Programming?
- Has the user practiced Binary Search recently?
- Is the user due for revision in Graphs?
- Which topic should be recommended next?

This table is not manually filled. It is updated from user activity and submission history.

---

## 3. How The System Works

The system follows a simple event-to-recommendation flow.

```text
User activity
    ↓
Event is stored
    ↓
Progress is updated
    ↓
Recommendations are generated
    ↓
User sees next suggested questions
```

---

## 4. Event-Based Flow

Whenever a meaningful action happens, we store it as an event.

Examples of events:

- User solved a problem
- User failed a problem
- User revised a topic
- Recommendation was shown
- Recommendation was skipped
- Recommendation was completed

The event log keeps a history of what happened. This is useful because the recommendation logic can evolve later without losing historical user behavior.

The event data is not directly used for serving recommendations on every request. Instead, a background process reads these events and updates faster read-model tables.

---

## 5. Derived Progress Flow

The system will maintain derived progress data.

This means we do not calculate the user's full progress from scratch every time they open the app.

Instead:

1. A user submits or solves a problem.
2. The event is stored.
3. A worker processes that event.
4. The user's problem-level progress is updated.
5. The user's topic-level progress is updated.
6. The active recommendation list is refreshed.

This keeps the recommendation API fast.

---

## 6. Recommendation Flow

When the system needs to recommend a question, it follows this process:

1. Check which topics the user is weak in.
2. Check which topics are due for revision.
3. Look at the curated DSA sheet.
4. Find questions under those topics.
5. Remove questions the user has already solved confidently.
6. Prefer questions that come earlier in the curated topic order.
7. Save the selected questions as active recommendations.
8. The frontend reads and displays those recommendations.

Example:

```text
User is weak in Dynamic Programming
    ↓
Find Dynamic Programming questions from curated sheet
    ↓
Remove already solved questions
    ↓
Pick the next suitable question
    ↓
Recommend Coin Change
```

---

## 7. Architecture Overview

The architecture is divided into four simple layers.

### Layer 1: Catalog Layer

This contains the shared problem catalog and the curated DSA sheet.

The shared problem catalog contains all available LeetCode problem metadata.

The curated sheet contains only the selected 200-400 problems that we want to recommend.

Purpose:

- Define the recommendation pool
- Organize questions by topic
- Connect curated questions to real LeetCode problems

### Layer 2: Event Layer

This stores user and recommendation activity as events.

Purpose:

- Keep history of user behavior
- Track submissions, revisions, skips, and recommendation interactions
- Allow future rebuilding of progress logic if needed

Events are append-only. We do not update old events. If something new happens, we add a new event.

### Layer 3: Progress Layer

This stores the current interpreted progress of the user.

Purpose:

- Know which problems the user has attempted or solved
- Know which topics the user is weak in
- Know which topics are due for revision
- Avoid scanning raw events on every request

This layer is derived from the event layer.

### Layer 4: Recommendation Serving Layer

This stores the current active recommendations for each user.

Purpose:

- Make recommendation reads fast
- Avoid recalculating recommendations during every frontend request
- Allow the frontend to simply ask: "What should this user solve next?"

The frontend should read from this layer instead of calculating recommendations itself.

---

## 8. How This Affects The Existing Architecture

The existing architecture will not be replaced. We are adding a recommendation layer on top of the current database.

The existing problem catalog continues to be the source of truth for LeetCode problem metadata.

The new curated sheet table only defines which problems are part of our MVP recommendation sheet.

The existing submission data continues to capture user LeetCode activity.

The recommendation system will use that activity to update progress and generate recommendations.

This keeps the architecture simple:

```text
Existing LeetCode catalog
        ↓
Curated DSA sheet
        ↓
User submission history
        ↓
User topic progress
        ↓
Active recommendations
```

---

## 9. Main Design Rules

The MVP should follow these rules:

1. Do not duplicate full LeetCode problem data in the curated sheet.
2. Store only selected sheet-specific data in the curated sheet.
3. Use the existing problem catalog as the source of truth for problem details.
4. Store important user actions as events.
5. Keep event logs append-only.
6. Use background processing to update progress.
7. Do not calculate recommendations from raw history during every request.
8. Serve recommendations from a precomputed active recommendation table.
9. Keep the recommendation logic simple for the MVP.
10. Use Supabase Postgres only.

---

## 10. Curated Sheet Table Purpose

The curated sheet table is the main new table needed for the catalog side of the MVP.

Its job is to answer:

```text
Which selected questions belong to which topic?
```

It should store:

```text
sheet_name
sheet_version
topic_name
topic_slug
problem_title
problem_slug
problem_order
source_name
source_url
is_active
```

It should reference the existing problem catalog using:

```text
platform + problem_slug
```

This gives us a clean way to recommend questions without duplicating the complete problem metadata.

---

## 11. Final Outcome

The final MVP design is a simple recommendation system built on top of the existing Supabase Postgres database.

It uses:

- The existing LeetCode problem catalog
- A new curated DSA sheet table
- Existing user submission history
- Derived user progress data
- Active recommendation data

The system will recommend questions from the curated sheet based on the user's weak topics, revision needs, and previous LeetCode activity.

This design keeps the MVP simple, avoids unnecessary infrastructure, and leaves room to improve the recommendation logic later.
