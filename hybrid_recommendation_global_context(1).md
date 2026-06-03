# Global Context: Hybrid Recommendation Model

## 1. Purpose

We are building the first version of a hybrid recommendation system for coding practice.

The system will recommend LeetCode questions to users based on:

- Their historical LeetCode submissions
- Their newly synced LeetCode submissions
- Their accepted and failed submission patterns
- The number of attempts made for a problem or topic
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

This curated sheet acts as the controlled recommendation pool.

---

## 3. Existing User Submission Data

We already have an extension that extracts the user's LeetCode submissions.

The extension can extract:

- Historical LeetCode submissions
- New submissions made after the extension is installed
- Accepted submissions
- Failed submissions
- Time limit errors
- Runtime errors
- Wrong answers
- Other non-accepted verdicts
- Submitted code
- Problem metadata
- Problem title and slug
- Difficulty and other available metadata

The system will not import only accepted code. It will import every available submission attempt, including accepted and failed submissions.

For example, if a user has 500 past LeetCode submissions, the extension will import those submissions and store them in the backend database.

This gives the backend enough data to understand:

- Which problems the user solved
- Which problems the user attempted
- Which problems the user failed
- How many times the user attempted each problem
- How many times the user got accepted for each problem
- How many times the user failed for each problem
- Which topics have repeated failed attempts
- Which topics have strong accepted history
- What code was submitted for each attempt
- When the submissions happened

This submission history is a major input signal for the recommendation system.

---

## 4. Hybrid Processing Model

The recommendation system will use a hybrid model:

```text
Batch processing + event-based processing
```

Batch processing will be used for historical or grouped analysis.

Event-based processing will be used for new activity and recommendation interactions.

The system will use both accepted and failed submissions as recommendation signals.

Accepted submissions help identify solved areas and user strengths.

Failed submissions help identify weak problems, weak topics, and areas where the user needs more practice.

---

## 5. Historical Backfill Flow

When a user connects the extension for the first time, the system will import the user's historical LeetCode submissions.

This historical import includes all available submission attempts, not just accepted submissions.

After the historical import is complete, a first processing task will run.

This first processing task will analyze approximately:

```text
50-60 past accepted submissions
```

The accepted submissions are used to understand the user's confirmed solved history and topic exposure.

The system can also use failed submission counts from the historical backfill to identify weak topics.

For example:

```text
User attempted multiple Dynamic Programming problems.
Many of those attempts failed before acceptance.
The system can treat Dynamic Programming as a weaker topic.
```

The purpose of the first processing task is to understand the user's existing strengths, solved topics, failed patterns, and practice history.

After processing the historical submissions, the system will generate the first set of active recommendations.

Flow:

```text
Extension imports historical LeetCode submissions
        ↓
Backend stores all submissions and code
        ↓
Batch processor analyzes accepted submissions and failed-attempt patterns
        ↓
Processor compares user history with curated DSA sheet
        ↓
Processor creates active recommendations
        ↓
Recommendations are stored for frontend use
```

---

## 6. Live Submission Sync Flow

After the initial backfill, the extension will continue tracking new LeetCode submissions.

When the user clicks submit on LeetCode and receives the server response, the extension will detect the result.

The system will sync every submitted code attempt, including:

- Accepted
- Wrong Answer
- Time Limit Exceeded
- Runtime Error
- Compilation Error
- Other failed verdicts

The backend will store:

- Submitted code
- Problem slug
- Problem title
- Submission status/verdict
- Difficulty
- Timestamp
- Attempt metadata
- Any other available metadata

The live sync flow is not limited to accepted submissions.

Every new submission attempt becomes useful behavioral data.

Accepted submissions show progress.

Failed submissions show struggle, repeated attempts, and possible weak topics.

Flow:

```text
User submits code on LeetCode
        ↓
Extension detects server response
        ↓
Extension sends accepted or failed submission to backend
        ↓
Backend stores submission and code
        ↓
Backend stores recommendation-processing event
        ↓
Processor uses the event to update recommendation state
```

---

## 7. Failed Submission Signals

Failed submissions are important for recommendation quality.

The system can use failed attempts to understand:

- Which problems are difficult for the user
- Which topics cause repeated failures
- Whether the user eventually solved a problem after many attempts
- Whether the user abandoned a problem after failed attempts
- Which topic should be recommended for focused practice

Example:

```text
User submits 7 times on a Dynamic Programming problem.
Only 1 submission is accepted, or no accepted submission exists.
The system can mark Dynamic Programming as an area that needs more practice.
```

The recommendation processor can use signals such as:

- Attempt count
- Accepted count
- Failed count
- Failure-to-accepted ratio
- Recent failed submissions
- Repeated failures within the same topic
- Whether the user eventually solved the problem

These signals can help generate recommendations from the curated DSA sheet.

---

## 8. Recommendation Event Log

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

In addition to visible recommendation lifecycle events, this table will also store internal recommendation-processing events.

These internal events can represent submission-driven processing, such as:

- Historical submission processed
- Live submission processed
- Accepted submission processed
- Failed submission processed
- Batch recommendation generated
- Event-based recommendation generated

Some events in this table are user-visible lifecycle events.

Other events are internal backend processing events and should not be shown to the user.

The recommendation event log helps the system understand:

- Which recommendations were shown
- Which recommendations were skipped
- Which recommendations were attempted
- Which recommendations were completed
- Which submissions triggered recommendation updates
- Whether a recommendation came from historical batch processing or live event processing
- Which recommendations should not be shown again immediately

The recommendation event log is append-only. Old events should not be overwritten. New user actions or backend processing steps should create new events.

---

## 9. Event Visibility

Not every recommendation event is meant to be shown to the user.

Some events are only kept for backend processing, debugging, and recommendation history.

For example:

```text
live_submission_processed
historical_submission_processed
batch_recommendation_generated
event_recommendation_generated
```

These events are useful for the system but should not appear in the user interface.

The system should maintain a way to distinguish:

```text
user-visible recommendation events
internal recommendation-processing events
```

This can be handled through the event type or through metadata.

Example metadata:

```json
{
  "visibility": "internal",
  "source": "live_submission_sync",
  "submission_verdict": "Wrong Answer",
  "processing_mode": "event"
}
```

---

## 10. Active Recommendation Table

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

The processor can also use recent failed submission patterns during this refresh.

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
Batch process analyzes recent accepted submissions and failed-attempt signals
        ↓
New recommendations are inserted
```

---

## 11. Recommendation Generation Logic

The recommendation processor uses several signals to generate recommendations:

- Curated DSA sheet topic grouping
- Existing LeetCode problem catalog
- Historical accepted submissions
- Historical failed submissions
- Live accepted submissions
- Live failed submissions
- Number of attempts per problem
- Number of accepted submissions per problem
- Number of failed submissions per problem
- Topic-level failure patterns
- User's previous recommendation skips
- User's previous recommendation attempts
- Existing active recommendation count

The processor should avoid recommending problems that:

- The user has already solved confidently
- The user recently skipped
- Are already active recommendations
- Are inactive in the curated DSA sheet

The processor can prefer problems that:

- Belong to weak topics
- Come after recently solved problems in the curated sheet
- Match topics with repeated failed submissions
- Are from the curated DSA sheet
- Have not been shown recently

---

## 12. Frontend Interaction Flow

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

## 13. Notification Generation

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
You had multiple recent failed attempts in Dynamic Programming. Try this related problem next to strengthen that topic.
```

---

## 14. Processing Cursor

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

## 15. Main Tables In Scope

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

## 16. Final Architecture Summary

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
Submission Storage
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
Live syncing for every submitted code attempt
Batch processing for historical accepted submissions and failed-attempt patterns
Event processing for new live submissions
Failed submission counts as weakness signals
Active recommendation thresholding
Recommendation events to track both user-visible events and internal processing events
Active recommendations as the serving layer for the frontend
LLM only for notification text generation
```

The main idea is:

```text
Store every user submission attempt.
Use accepted submissions to identify progress.
Use failed submissions to identify weak areas.
Store recommendation lifecycle and processing events.
Process historical data in batches.
Process new submissions as events.
Keep only a small active recommendation set ready for the frontend.
Use the curated DSA sheet as the controlled recommendation pool.
```
