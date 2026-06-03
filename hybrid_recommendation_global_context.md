# Global Context: Hybrid Recommendation Model

## 1. Purpose

We are building the first version of a hybrid recommendation system for coding practice.

The system will recommend LeetCode questions to users based on:

- Their historical LeetCode submissions
- Their newly accepted LeetCode submissions
- Their interaction with previous recommendations
- A curated internal DSA sheet created by us

The goal is to create an MVP recommendation model that uses both batch processing and event-based processing.

---

## 2. Curated DSA Sheet

We will create a curated DSA sheet table named:

```text
catalog.dsa_sheet_problems
```

This table will contain the selected DSA questions that are eligible for recommendations.

The sheet will be divided by topics such as:

- Arrays
- Strings
- Binary Search
- Two Pointers
- Dynamic Programming
- Graphs
- BFS
- DFS
- Trees
- Greedy

Each row in this table will represent one curated question under a specific topic.

The curated sheet will not store full LeetCode problem metadata. Instead, each row will reference the existing shared LeetCode problem catalog using:

```text
platform + problem_slug
```

The existing problem catalog will remain the source of truth for full problem information.

The curated DSA sheet will mainly store:

- Problem slug
- Problem title
- Topic name
- Topic slug
- Problem order inside the topic
- Sheet name
- Sheet version
- Source name
- Source URL
- Active status
- Extra metadata

The source fields can store where the problem was taken from, such as:

- Striver Sheet
- NeetCode Sheet
- Blind 75
- Internal curated sheet

This curated sheet acts as the recommendation pool.

---

## 3. Existing User Submission Data

We already have an extension that extracts the user's LeetCode submissions.

The extension can extract:

- Historical LeetCode submissions
- New submissions made after the extension is installed
- Submission status, such as Accepted or Failed
- Submitted code
- Problem metadata
- Problem title and slug
- Difficulty and other available metadata

For example, if a user has 500 past LeetCode submissions, the extension will import those submissions and store them in the backend database.

This gives the backend enough data to understand the user's solved problems, attempted problems, recent activity, and coding history.

---

## 4. Hybrid Processing Model

The recommendation system will use a hybrid model:

```text
Batch processing + event-based processing
```

Batch processing will be used for historical or grouped analysis.

Event-based processing will be used for new activity and recommendation interactions.

---

## 5. Historical Backfill Flow

When a user connects the extension for the first time, the system will import the user's historical LeetCode submissions.

After the historical import is complete, a first processing task will run.

This first processing task will analyze approximately:

```text
50-60 past accepted submissions
```

The purpose of this processing task is to understand the user's existing strengths, solved topics, and practice history.

After processing the historical accepted submissions, the system will generate the first set of active recommendations.

Flow:

```text
Extension imports historical LeetCode submissions
        ↓
Backend stores submissions and code
        ↓
Batch processor analyzes 50-60 accepted submissions
        ↓
Processor compares user history with curated DSA sheet
        ↓
Processor creates active recommendations
        ↓
Recommendations are stored for frontend use
```

---

## 6. Live Submission Flow

After the initial backfill, the extension will continue tracking new LeetCode submissions.

When the user clicks submit on LeetCode and receives the server response, the extension will detect the result.

If the submission is accepted, the extension will send the submission data to the backend.

The backend will store:

- Submitted code
- Problem slug
- Problem title
- Submission status
- Difficulty
- Timestamp
- Any other available metadata

At the same time, a recommendation-related event will be stored in the recommendation event log.

Then a backend processor will analyze the new accepted submission and generate or refresh recommendations from the curated DSA sheet.

Flow:

```text
User submits code on LeetCode
        ↓
Extension detects accepted response
        ↓
Backend stores submission and code
        ↓
Backend stores recommendation event
        ↓
Event processor analyzes the new submission
        ↓
Processor finds related questions from curated DSA sheet
        ↓
Processor refreshes active recommendations
```

---

## 7. Recommendation Event Log

The table:

```text
app.recommendation_events
```

will act as the event log for the recommendation system.

It stores recommendation-related events such as:

- Recommendation shown
- Recommendation clicked
- Recommendation accepted
- Recommendation skipped
- Recommendation dismissed
- Recommendation completed
- Recommendation expired

This table is used to track the recommendation lifecycle and user interactions with recommendations.

For this MVP, this event log is important because it helps the system understand:

- Which recommendations were shown
- Which recommendations were skipped
- Which recommendations were attempted
- Which recommendations were completed
- Which recommendations should not be shown again immediately

The recommendation event log is append-only. Old events should not be overwritten. New user actions should create new events.

---

## 8. Active Recommendation Table

The table:

```text
app.user_active_recommendations
```

will store the recommendations that are currently ready to be shown to the user.

This table is the serving table for recommendations.

The frontend should read recommendations from this table instead of calculating recommendations directly.

Each user will have a controlled number of active recommendations.

The planned threshold is:

```text
Maximum active recommendations: 10
Minimum active recommendation threshold: 5
```

If the number of active recommendations falls below 5, the backend will run another batch recommendation process.

That batch process will analyze approximately:

```text
10-20 recent accepted submissions
```

and generate more active recommendations for the user.

Flow:

```text
Frontend reads active recommendations
        ↓
User attempts or skips recommendations
        ↓
Active recommendation count decreases
        ↓
If active count falls below 5
        ↓
Batch process analyzes recent accepted submissions
        ↓
New recommendations are inserted
```

---

## 9. Frontend Interaction Flow

The frontend will show recommendations to the user.

Each recommendation will have two main actions:

```text
Attempt
Skip
```

### Attempt

When the user clicks Attempt, the system will record that the user attempted the recommendation.

This interaction should be stored as a recommendation event.

The active recommendation row can also be updated to show that the recommendation was attempted or shown.

### Skip

When the user clicks Skip, the system will record that the user skipped the recommendation.

This interaction should also be stored as a recommendation event.

The active recommendation can then be dismissed, expired, or removed from the active recommendation list.

---

## 10. Notification Generation

The active recommendation table will contain enough information to generate user-facing recommendation notifications.

This data can include:

- Problem title
- Problem slug
- Topic name
- Topic slug
- Sheet name
- Sheet version
- Source name
- Reason
- Score
- Rank
- Metadata

This data can be sent to an LLM to create a user-friendly notification.

The LLM should only generate the notification text.

The LLM should not decide which problem to recommend.

The recommendation decision should come from the backend recommendation processor.

Example notification:

```text
You recently solved several Array problems. Try this Binary Search problem next to strengthen your search pattern recognition.
```

---

## 11. Processing Cursor

The optional table:

```text
app.projector_cursors
```

can be used to track how far the processor has read from an event source.

This is useful for batch/event processing because it helps avoid processing the same events repeatedly.

It can track:

- Projector name
- Event source
- Last processed event ID
- Last processed timestamp
- Extra processing metadata

This table is optional for the MVP but useful for reliability.

---

## 12. Main Tables In Scope

The first step of the hybrid recommendation model uses these new tables:

```text
catalog.dsa_sheet_problems
app.recommendation_events
app.user_active_recommendations
app.projector_cursors
```

The table schemas have already been prepared separately.

There may be small schema changes later, but the current focus is the architecture and flow of the recommendation system.

---

## 13. Final Architecture Summary

The planned architecture is:

```text
Curated DSA Sheet
        ↓
Existing LeetCode Problem Catalog
        ↓
Historical LeetCode Submission Import
        ↓
Batch Recommendation Processor
        ↓
Active Recommendations
        ↓
Frontend Recommendation UI

Live LeetCode Submission
        ↓
Recommendation Event Log
        ↓
Event Processor
        ↓
Active Recommendations
        ↓
Frontend Recommendation UI
```

The system uses:

```text
Historical backfill for first-time recommendations
Live event processing for new accepted submissions
Batch refresh when active recommendations fall below threshold
Recommendation events to track lifecycle and interactions
Active recommendations as the serving layer for the frontend
LLM only for notification text generation
```

The main idea is:

```text
Store user activity and recommendation interactions as events.
Process historical data in batches.
Process new accepted submissions as events.
Keep only a small active recommendation set ready for the frontend.
Use the curated DSA sheet as the controlled recommendation pool.
```
