# Deterministic Pre-Router and Model Policy Tier Report

## 1. Objective

The goal of this proposal is to reduce chat latency by avoiding unnecessary routing and by reserving Gemini 2.5 Pro only for tasks that truly need higher reasoning quality. The current discussion shows that many user queries are predictable enough to route without an LLM router call. Examples include greetings, simple definitions, beginner explanations, lookup-style answers, and routine teaching questions.

The recommended direction is to introduce a deterministic pre-router before the existing LLM router and to separate answer depth from model policy. This gives the system a faster path for obvious requests while preserving the existing router for ambiguous or complex requests.

The main expected benefits are:

- Faster time-to-first-token for simple and common requests.
- Lower total response latency for normal teaching and explanation tasks.
- Reduced Gemini 2.5 Pro usage.
- Lower serving cost.
- More predictable model selection.
- Better control over token limits and thinking budgets.

This proposal should be understood as a routing and policy improvement, not simply as an enum expansion.

---

## 2. Core Problem

The current routing design relies heavily on a broad depth classification. The system receives a query, sends it to the router, receives a depth value, and then uses that depth to influence the response path.

The issue is that a single depth value is too coarse. It does not fully capture what the system needs to know in order to choose the right model and latency budget.

For example, these two requests may both look “deep” in a broad routing system, but they should not necessarily use the same model:

| Query | Desired Behavior | Reason |
|---|---|---|
| “Explain Kafka sendfile with tradeoffs” | Use Gemini 2.5 Flash first | It is technical, but Flash should be strong enough for a fast, useful explanation. |
| “Compare Raft and Paxos deeply at staff-engineer level” | Use Gemini 2.5 Pro | The user explicitly asks for deep, high-quality reasoning. |

Similarly, these two requests may both be “simple,” but they still need different handling:

| Query | Desired Behavior | Reason |
|---|---|---|
| “hi” | Instant or Flash-Lite path | No meaningful routing is needed. |
| “what is a bloom filter?” | Fast teaching path on Flash | It needs explanation, but not Pro-level reasoning. |

The main problem is not only which model is selected. The larger latency problem is that many obvious requests may still pay the cost of the router call before the answerer can begin.

---

## 3. Why Adding More Depth Values Alone Is Not Enough

A natural first idea is to divide the existing depth key into more categories. This is directionally useful, but it is not the cleanest design.

Depth should describe the answer shape: shallow, medium, deep, or mock-style. It should not also be responsible for model choice, token limits, thinking budget, tool behavior, and routing confidence.

When one field controls too many things, the system becomes hard to tune. A small change meant to improve quality can accidentally increase latency or cost. A small change meant to reduce latency can accidentally reduce answer depth.

The better design is to separate the concerns:

| Concern | What It Should Decide | Example |
|---|---|---|
| Answer depth | How detailed the answer should be | Shallow, medium, deep, mock |
| Policy tier | Which model, token budget, and thinking budget to use | instant, teach_fast, deep_pro |
| Tool plan | Whether the system should call tools | No tool, retrieval, interview question lookup |
| Routing source | Who made the route decision | Deterministic pre-router or LLM router |
| Confidence | Whether the route is safe to execute directly | High-confidence match or fallback |

This separation allows the system to improve latency without making the meaning of depth confusing.

---

## 4. Proposed Architecture

The recommended architecture is a two-layer routing system.

| Stage | Role | When It Should Handle the Query |
|---|---|---|
| Deterministic pre-router | Fast rule-based classification | Only when the query is obvious and high-confidence |
| Existing LLM router | Flexible semantic routing | When the query is ambiguous, multi-step, tool-heavy, or context-dependent |
| Executor/tools | Performs required tool calls | When the selected plan needs external or internal data |
| Answerer | Produces final response | Uses the selected policy tier and answer depth |

The deterministic pre-router should sit before the existing LLM router. It should not replace the LLM router. Its job is to catch the easy cases and skip unnecessary work.

A safe mental model is:

| If the pre-router is confident | If the pre-router is unsure |
|---|---|
| Return a complete route immediately | Send the query to the existing LLM router |
| Skip router latency | Preserve current flexible behavior |
| Use a known policy tier | Let the router decide |

This design keeps the latency improvement bounded and safe. The system only skips the router when the query type is clear.

---

## 5. Proposed Policy Tiers

Instead of using depth alone, introduce model policy tiers. These tiers describe the latency, cost, token, and reasoning profile for a response.

| Policy Tier | Model | Token Budget | Thinking Budget | Intended Use |
|---|---:|---:|---:|---|
| instant | Gemini 2.5 Flash-Lite | 300 | 0 | Greetings, acknowledgements, very small replies |
| shallow | Gemini 2.5 Flash | 600 | 0 | Simple product/help answers or short factual responses |
| teach_fast | Gemini 2.5 Flash | 1200 | 0 | Definitions, beginner teaching, normal explanations |
| lookup_format | Gemini 2.5 Flash | 1000 | 0 | Lookup-style or formatting-heavy responses |
| deep_fast | Gemini 2.5 Flash | 1700 | 0 | Technical explanations with tradeoffs, but not explicitly Pro-level |
| deep_pro | Gemini 2.5 Pro | 2000 | 128 | Explicitly deep, rigorous, staff-level, or research-level reasoning |
| mock_eval | Gemini 2.5 Pro | 2800 | 512 | Mock interviews, grading, answer review, evaluation |

The important change is that Flash becomes the default for most normal tasks, while Pro becomes the escalation path for tasks that clearly require it.

---

## 6. Why These Tiers Are Useful

The proposed tiers solve multiple problems at once.

### 6.1 They reduce overuse of Pro

A lot of interview-prep and learning queries do not need Pro. For example, “what is a bloom filter?” or “teach me consistent hashing” should usually be answered quickly by Flash. Using Pro for these cases increases latency and cost without a proportional quality gain.

### 6.2 They make latency predictable

Each tier has a known model, token budget, and thinking budget. This makes performance easier to reason about. If a query is routed to `teach_fast`, the system knows it should produce a helpful explanation without paying Pro latency.

### 6.3 They preserve quality for high-stakes tasks

Mock interviews, grading, and detailed answer review should stay on Pro because they require judgment, evaluation, and nuanced feedback. The proposal does not remove Pro. It reserves Pro for the moments where it matters.

### 6.4 They make future tuning easier

If `deep_fast` answers are too short, the token budget can be increased without changing depth logic. If `teach_fast` is too slow, its budget can be reduced. If Flash-Lite is not reliable enough for a category, that tier can be moved to Flash without changing the router architecture.

---

## 7. Recommended Routing Behavior

The deterministic pre-router should focus on high-confidence patterns. It should avoid complex semantic judgment.

| Query Type | Example | Recommended Policy Tier | Router Skipped? | Why |
|---|---|---|---|---|
| Greeting | “hi”, “hello” | instant | Yes | No LLM routing is needed. |
| Acknowledgement | “thanks”, “ok”, “got it” | instant | Yes | Response can be very short. |
| Simple definition | “what is a bloom filter?” | teach_fast | Yes | Common teaching pattern, low ambiguity. |
| Beginner learning | “I’m a novice, teach me coding” | teach_fast | Usually yes | Flash should be sufficient for a structured intro. |
| Simple explanation | “explain consistent hashing simply” | teach_fast | Usually yes | The user wants education, not deep evaluation. |
| Lookup/format request | “show my plan”, “how many problems solved?” | lookup_format or shallow | Yes if data path is deterministic | These can often be DB read plus template. |
| Technical tradeoff explanation | “explain Kafka sendfile with tradeoffs” | deep_fast | Sometimes | Flash first is appropriate unless user asks for staff-level depth. |
| Explicit deep comparison | “compare Raft and Paxos deeply” | deep_pro | Yes or router-confirmed | The word “deeply” signals stronger reasoning need. |
| Mock interview | “mock interview me on Dropbox system design” | mock_eval | Yes | Mock evaluation should use Pro. |
| Answer grading | “review my answer and grade it” | mock_eval | Yes | Quality of judgment matters. |
| Ambiguous follow-up | “what about that?” | Existing router | No | Needs conversation context. |
| Tool-heavy request | “find questions asked by Google and explain patterns” | Existing router or exact deterministic tool route | Usually no | Tool needs must be planned carefully. |

The highest-value early wins are greetings, acknowledgements, simple definitions, and beginner teaching requests. These are common, easy to identify, and low risk.

---

## 8. Why Flash Should Be the Default for Normal Tasks

Gemini 2.5 Flash should handle most normal product, teaching, and explanation queries. The goal is not to reduce quality; the goal is to stop spending Pro-level latency on tasks that do not need Pro-level reasoning.

| Task Category | Flash Suitability | Reason |
|---|---|---|
| Definitions | High | Usually requires clarity, not heavy reasoning. |
| Beginner explanations | High | Needs structure and examples, not deep inference. |
| Standard CS concepts | High | Common knowledge patterns are well suited to Flash. |
| Formatting responses | High | Mostly presentation and organization. |
| Product/help answers | High | Usually short and predictable. |
| Tradeoff explanations | Medium to high | Flash can handle many practical tradeoffs quickly. |
| Deep theoretical comparisons | Medium to low | Pro may be better when rigor is requested. |
| Mock interviews | Low | Requires adaptive questioning and judgment. |
| Grading user answers | Low | Requires nuanced evaluation and feedback. |

The recommended strategy is “Flash first for normal tasks, Pro when explicitly justified.”

---

## 9. When Pro Should Be Used

Pro should not be treated as the default deep-answer model. It should be used when the request signals that quality and reasoning depth are more important than speed.

Strong Pro triggers include:

- The user asks for a mock interview.
- The user asks the system to grade, score, review, or evaluate an answer.
- The user asks for staff-level, principal-level, research-level, or rigorous analysis.
- The user asks for a deeply comparative answer where subtle tradeoffs matter.
- The request is long, multi-part, and reasoning-heavy.
- The cost of a bad answer is higher than the cost of latency.

Weak Pro triggers should not automatically escalate:

- The word “explain” by itself.
- A normal CS concept question.
- A simple “compare X and Y” unless depth is explicitly requested.
- A beginner teaching request.
- A formatting or summarization request.

This keeps Pro usage intentional rather than accidental.

---

## 10. Edge Cases and How to Handle Them

The deterministic pre-router must be conservative. Most failures will come from routing a query too confidently when it needed context or deeper planning.

| Edge Case | Example | Risk | Recommended Handling |
|---|---|---|---|
| Ambiguous follow-up | “explain that more” | The pre-router may not know what “that” refers to. | Fall back to LLM router when the query depends on prior context. |
| User asks for depth indirectly | “give me the real tradeoffs” | Could be deeper than a normal explanation. | Route to deep_fast first; allow Pro escalation if wording suggests rigor. |
| Short but complex query | “Raft vs Paxos?” | Short text can still be complex. | Use router unless strong deterministic pattern exists. |
| Long but simple query | User pastes a paragraph and asks for formatting | Length may look complex but task is simple. | Route based on intent, not only length. |
| Multiple intents | “Teach me Kafka and then quiz me” | Teaching and evaluation need different policies. | Fall back to router. |
| Tool requirement hidden in wording | “show my plan” | Needs DB or user state, not just answer generation. | Use deterministic route only if the data path is known. |
| User asks for “best” or “latest” | “best model for this today” | May require current information or nuanced judgment. | Use router and/or retrieval, not pre-router. |
| Safety-sensitive request | Legal, medical, financial, security-sensitive topics | Wrong route could produce unsafe oversimplification. | Fall back to router and safer policy. |
| Very high user expectation | “be extremely precise” | Flash may answer too generally. | Escalate to deep_pro when precision/rigor is explicit. |
| Mock-like request without keyword | “pretend you are my interviewer” | Could be missed by simple keyword rules. | Add tested phrases gradually; otherwise router fallback. |

The main design rule is: false negatives are acceptable, false positives are expensive. It is better for the pre-router to miss some opportunities and fall back to the existing router than to incorrectly skip the router for a complex request.

---

## 11. Expected Latency Impact

This change improves latency through two mechanisms.

First, deterministic pre-router hits skip the LLM router. This directly reduces time before the answerer can start. The improvement should be especially visible in time-to-first-token.

Second, model policy tiers reduce unnecessary Pro calls. Flash and Flash-Lite should generally respond faster and cheaper than Pro, especially for short or normal-complexity requests.

| Area | Expected Impact | Explanation |
|---|---|---|
| Greetings and acknowledgements | Very high improvement | Router and heavy answer generation can be skipped. |
| Definitions and simple teaching | High improvement | Router can often be skipped and Flash can answer directly. |
| Normal technical explanations | Medium improvement | Flash can handle many of these without Pro. |
| Deep technical reasoning | Low to medium improvement | Some requests still need Pro. |
| Mock interviews | Low improvement | These should remain on Pro for quality. |
| Grading and answer review | Low improvement | These should remain on Pro for evaluation quality. |
| Overall cost | Meaningful reduction | Fewer Pro calls and fewer router calls. |
| TTFB | Meaningful reduction | The answerer starts earlier for pre-router hits. |

The biggest gains will come from the frequency of simple and normal teaching queries. If these make up a large percentage of traffic, the overall latency improvement should be noticeable.

---

## 12. Quality Risks

The main risk is quality degradation from moving too many requests to Flash or from skipping the router too aggressively.

| Risk | Why It Matters | Mitigation |
|---|---|---|
| Overrouting to Flash | Some complex tasks may receive weaker answers. | Only route obvious cases deterministically; use Pro triggers for explicit depth. |
| Shallow answers for advanced users | Experienced users may expect more detail. | Detect “deep,” “staff-level,” “rigorous,” and similar phrases. |
| Incorrect handling of follow-ups | Follow-ups often depend on prior context. | Fall back to LLM router for pronouns and context-dependent messages. |
| Rule growth over time | Deterministic rules can become messy. | Keep the pre-router small, tested, and high-precision. |
| Hidden tool needs | Some short queries require DB or retrieval. | Only skip router when the required data path is known. |
| User dissatisfaction | Faster answers are not useful if quality drops. | Monitor feedback and “go deeper” follow-up rates. |

The proposal is safe if rollout starts narrow and expands only after telemetry confirms quality.

---

## 13. Telemetry Needed to Prove the Change Works

The system should log enough information to compare behavior before and after the change.

Recommended fields:

| Field | Purpose |
|---|---|
| pre_router_hit | Whether the deterministic layer handled the query. |
| pre_router_reason | Why the deterministic layer matched. |
| router_skipped | Whether the LLM router was bypassed. |
| policy_tier | Which model policy was selected. |
| depth | What answer shape was selected. |
| model | Which Gemini model answered. |
| tool_count | Whether tools were used. |
| ttfb_ms | Time-to-first-token measurement. |
| total_ms | End-to-end response latency. |
| answer_tokens | Token usage. |
| user_feedback | Thumbs-up/down or similar signal. |
| followup_depth_request | Whether the user immediately asked for more depth. |

The most important metrics are:

- Pre-router hit rate.
- Router skip rate.
- Gemini 2.5 Pro call rate.
- Gemini 2.5 Flash and Flash-Lite call rate.
- p50 and p95 time-to-first-token.
- p50 and p95 total latency.
- User negative feedback rate.
- “Go deeper” follow-up rate.
- Fallback rate to the LLM router.

Success should not be measured by latency alone. The change is successful only if latency improves without a meaningful quality regression.

---

## 14. Testing Strategy

Testing should focus on whether the right categories are routed deterministically and whether ambiguous cases still fall back to the existing router.

| Test Query | Expected Policy | Expected Model | Router Behavior |
|---|---|---|---|
| “hi” | instant | Flash-Lite | Skipped |
| “hello” | instant | Flash-Lite | Skipped |
| “thanks” | instant | Flash-Lite | Skipped |
| “what is a bloom filter?” | teach_fast | Flash | Skipped |
| “define consistent hashing” | teach_fast | Flash | Skipped |
| “explain consistent hashing simply” | teach_fast | Flash | Skipped |
| “I’m a novice, teach me coding” | teach_fast | Flash | Skipped or high-confidence route |
| “explain Kafka sendfile with tradeoffs” | deep_fast | Flash | Skipped only if rule is confident |
| “compare Raft and Paxos deeply” | deep_pro | Pro | Skipped or router-confirmed |
| “mock interview me on Dropbox system design” | mock_eval | Pro | Skipped |
| “review my answer and grade it” | mock_eval | Pro | Skipped |
| “what about that?” | Router-selected | Depends | Not skipped |
| “can you do the same for the other one?” | Router-selected | Depends | Not skipped |
| “teach me Kafka and then quiz me” | Router-selected | Depends | Not skipped |

Regression tests should also ensure that bad router output, missing fields, unknown policy tiers, or malformed plans fall back safely.

---

## 15. Rollout Plan

A phased rollout is recommended. The goal is to reduce risk while still getting measurable latency wins early.

### Phase 1: Separate Policy from Depth

Introduce policy tiers and central model policy while keeping existing behavior mostly unchanged. This makes the system easier to reason about before changing routing behavior.

Why this comes first:

- It reduces coupling.
- It makes later changes easier to test.
- It creates a single place to tune model, token, and thinking settings.

### Phase 2: Add Pre-Router for Instant Queries

Start with greetings and acknowledgements such as “hi,” “hello,” “thanks,” “ok,” and “got it.”

Why this is safe:

- These requests are easy to classify.
- They rarely need tools or deep context.
- The user expects a short response.
- The latency win is immediate.

### Phase 3: Add Definitions and Beginner Teaching

Add common teaching patterns such as “what is X,” “define X,” “explain X simply,” and “I am a beginner, teach me X.”

Why this is valuable:

- These are common in an interview-prep product.
- Flash is well suited for this style of response.
- Pro is usually unnecessary.

### Phase 4: Add Deep Fast vs Deep Pro Split

Route normal technical explanations with tradeoffs to `deep_fast` on Flash. Escalate to `deep_pro` only when the user explicitly asks for deep, rigorous, staff-level, or research-level analysis.

Why this matters:

- Many technical questions benefit from detail but do not require Pro.
- The system can give a fast strong answer first.
- Users can still ask for more depth if needed.

### Phase 5: Keep Evaluation and Mock Interviews on Pro

Mock interviews, grading, and answer review should remain on Pro.

Why this matters:

- These tasks require judgment.
- Quality is more important than speed.
- A poor evaluation can reduce user trust.

---

## 16. Recommended Decision Rules

The following decision rules summarize the intended behavior.

| Rule | Recommendation |
|---|---|
| If the query is a greeting or acknowledgement | Use instant path and skip router. |
| If the query is a simple definition | Use teach_fast on Flash. |
| If the query is beginner teaching | Use teach_fast on Flash. |
| If the query asks for formatting or simple account/product state | Use lookup_format or shallow if the data path is known. |
| If the query asks for technical tradeoffs | Use deep_fast unless the user explicitly asks for high rigor. |
| If the query asks for deep/staff-level/research-level analysis | Use deep_pro. |
| If the query asks for a mock interview | Use mock_eval. |
| If the query asks for grading or answer review | Use mock_eval. |
| If the query is ambiguous or context-dependent | Use existing LLM router. |
| If the query has multiple intents | Use existing LLM router. |
| If the query may require tools and the path is not deterministic | Use existing LLM router. |

These rules keep the pre-router simple and reduce the chance of misrouting.

---

## 17. Final Recommendation

This proposal is worth implementing.

The best framing is not “divide depth into more values.” The better framing is:

- Add a deterministic pre-router for high-confidence cases.
- Keep the existing LLM router for ambiguous and complex requests.
- Separate answer depth from model policy.
- Make Gemini 2.5 Flash the default for normal teaching and explanation tasks.
- Reserve Gemini 2.5 Pro for mock interviews, grading, explicit deep reasoning, and high-judgment tasks.
- Measure the change using latency, Pro usage, feedback, and follow-up-depth metrics.

This approach should reduce latency and cost while preserving quality. It avoids unnecessary router calls for obvious requests and prevents Pro from being used when Flash is sufficient. The safest implementation is phased, conservative, and telemetry-driven.
