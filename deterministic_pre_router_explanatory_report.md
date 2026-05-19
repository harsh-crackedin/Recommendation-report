# Deterministic Pre-Router Proposal for Latency Reduction

## Purpose

The proposed deterministic pre-router is a latency optimization layer that sits before the model-based router. Its purpose is to avoid calling a routing model when the correct routing decision is already obvious from the user message, conversation history, or a fixed product rule.

This report focuses on **why** the change is useful, where it creates value, where it can be risky, and how to reason about safe adoption. It intentionally avoids implementation code and presents the idea in product and engineering terms.

---

## Core idea

Every user message currently has to be classified before the system can decide what kind of response path to use. That router decision is useful for ambiguous or complex prompts, but it is unnecessary for many simple prompts.

For example, messages such as “hi”, “thanks”, “make it shorter”, or “go deeper” usually do not need a language model to decide what should happen next. A deterministic rule can classify these cases faster, more cheaply, and more predictably.

The deterministic pre-router should therefore act as a **high-confidence filter**:

1. If the request is obvious, return a ready-to-use plan immediately.
2. If the request is ambiguous, do nothing and let the model router decide.
3. If the request requires complex reasoning, retrieval judgment, user-state interpretation, or tool choice, do not shortcut it.

The goal is not to replace the router. The goal is to remove router calls that are clearly avoidable.

---

## Why this improves latency

The router call is an extra model call before the real answer can begin. Even if the router uses a fast model, it still adds network latency, model processing time, JSON parsing, fallback handling, and orchestration overhead.

For simple messages, that extra step gives almost no quality improvement. The user does not benefit from a model deciding that “thanks” is a shallow direct reply or that “make it shorter” is a rewrite request. In these cases, the deterministic layer allows the system to move directly toward the response path.

| Source of delay | Why it matters | How the deterministic layer helps |
|---|---|---|
| Router model request | Adds a full model round trip before the answer path | Avoids the model call when classification is obvious |
| Router response parsing | Router output must be parsed and validated before continuing | Deterministic plans are already structured and predictable |
| Fallback handling | Router failures or malformed outputs require fallback logic | Rule-based outcomes avoid model-output uncertainty |
| Time to first byte | The answer stream cannot start until routing is complete | Direct plans allow the response pipeline to begin sooner |
| Cost overhead | Each router call consumes model/API budget | Skipping unnecessary calls reduces repeated small costs |

The largest user-visible improvement is expected on short, common, low-complexity prompts because the router cost can be a large percentage of the total request time.

---

## Proposed behavior

The deterministic pre-router should classify only a limited set of high-confidence request categories. Its behavior should be conservative: it should prefer missing an optimization over choosing the wrong path.

| Request type | Example user message | Deterministic decision | Why this is safe |
|---|---|---|---|
| Greeting | “hi”, “hello”, “hey” | Shallow direct response | No retrieval, planning, or model routing decision is needed |
| Acknowledgement | “ok”, “got it”, “cool” | Shallow continuation response | These messages usually confirm or continue the prior state |
| Thanks | “thanks”, “thank you” | Shallow direct response | The response is predictable and low-risk |
| Short rewrite follow-up | “make it shorter”, “rephrase that” | Shallow answer using prior context | The intent is explicit and does not require tool selection |
| Explanation-depth follow-up | “go deeper”, “explain more” | Deeper answer only when prior context exists | The phrase is meaningful only when a previous answer is available |
| Clear product/static question | “what can you do?”, “daily limit?” | Cached/template response where possible | These answers should be stable and fast |
| Clear company-interview lookup | “what did Google ask recently?” | Tool-backed plan only if the company and intent are unambiguous | Saves router time while preserving retrieval/tool usage |
| Clear concept explanation | “how does Raft leader election work?” | Knowledge-context plan when the concept request is explicit | The need for knowledge retrieval is obvious |

The important distinction is that the pre-router does not need to answer everything. It only needs to confidently handle the repetitive cases that appear often enough to matter.

---

## Why the pre-router should be conservative

A deterministic shortcut has two possible failure modes:

| Failure type | What happens | Impact |
|---|---|---|
| False negative | The pre-router does not match a prompt that it could have handled | Minor impact: the normal router still handles it |
| False positive | The pre-router matches a prompt incorrectly and bypasses the router | Higher impact: the user may get the wrong tool, wrong depth, or wrong response style |

Because false positives are more harmful than false negatives, the pre-router should only handle prompts where the reason is obvious and explainable. For unclear prompts, it should fall through to the model router.

This is the central safety principle: **a missed optimization is acceptable; a wrong deterministic route is not.**

---

## Categories that should bypass the router first

### 1. Very short conversational turns

These are the safest starting point because they are frequent and low risk.

Examples include greetings, thanks, acknowledgements, and basic continuation prompts. These requests generally do not need retrieval, tool execution, deep reasoning, or a Pro model. They should be served using a shallow, fast response path.

Why this matters:

- They appear frequently in real conversations.
- The router decision is almost always the same.
- The latency cost feels especially noticeable because the expected answer is short.
- The user’s perceived experience improves when simple replies are immediate.

### 2. Rewrite and formatting follow-ups

Requests like “make it shorter”, “simplify this”, “rephrase that”, or “give one example” are usually direct transformations of previous context. They do not require the router to decide between retrieval, interview-question lookup, deep reasoning, or product flows.

Why this matters:

- The user is asking to transform existing content, not start a new task.
- The correct response style is usually shallow or medium.
- Tool calls are usually unnecessary.
- Router latency delays a task that should feel instant.

The main guardrail is that these should only be treated as follow-ups when there is useful prior conversation context.

### 3. History-backed depth changes

Messages such as “go deeper”, “explain more”, or “walk me through it” can be routed deterministically only when the conversation history contains a substantial previous answer. Without that prior answer, the prompt is vague.

Why this matters:

- With history, the intent is clear: expand the previous explanation.
- Without history, the message lacks a subject.
- This avoids wasting a router call on common follow-up behavior while still protecting new or empty conversations.

### 4. Static product and account-style questions

Some prompts have stable answers or can be served from known state without model routing. Examples include “what can you do?”, “daily limit?”, “show my plan”, or “how many problems solved?”

Why this matters:

- These answers can often be generated from a fixed template or a direct database read.
- A router model is unnecessary if the product intent is exact.
- Users expect these answers to be fast and factual.

This category should be handled carefully because account-specific questions may need authenticated state. The deterministic pre-router can decide the path, but the actual data should still come from the correct source of truth.

### 5. High-confidence tool-backed intents

Some requests clearly require a specific tool. For example, a user asking “what did Google ask recently?” likely needs interview-question retrieval. A user asking “how does Raft leader election work?” likely needs knowledge-context retrieval.

Why this matters:

- The router call can be skipped while still preserving the tool call.
- The system saves only the routing latency, not the retrieval latency.
- This is still valuable because routing happens before retrieval and answer generation.

This category should be introduced after the simpler categories are stable, because wrong tool routing is more damaging than a wrong shallow/direct response.

---

## Edge cases and expected handling

| Edge case | Example | Recommended behavior | Reason |
|---|---|---|---|
| Vague prompt | “help” | Fall through to model router | The user intent is underspecified |
| Ambiguous company word | “google this for me” | Fall through | “Google” is used as a verb, not a company target |
| Ambiguous product word | “meta question” | Fall through | Could refer to Meta company or a meta-level question |
| Contextless follow-up | “go deeper” in a new chat | Fall through | There is no subject to expand |
| Missing artifact | “review this” without attached content | Fall through or ask for content | The task cannot be completed deterministically |
| Multi-intent request | “summarize this and make a study plan” | Fall through | Requires planning and possibly multiple response modes |
| Safety-sensitive query | Medical, legal, financial, or policy-sensitive request | Fall through | Needs current, careful, or policy-aware reasoning |
| Tool ambiguity | “show recent questions” | Fall through | The target company, topic, or source may be unclear |
| User-specific state | “how many problems did I solve?” | Deterministic path only if state lookup is exact | The path is obvious, but the data must come from the database |
| Long complex task | “design a full interview prep roadmap” | Fall through | Requires nuanced depth and planning |

These edge cases are important because they define the boundary of the optimization. The deterministic layer should be narrow enough that engineers can explain every bypass decision in plain language.

---

## Decision policy

The deterministic pre-router should follow a simple confidence policy.

| Confidence level | Action | Explanation |
|---|---|---|
| Certain | Return a deterministic plan | The message matches a strict, known pattern with no ambiguity |
| Likely but not certain | Fall through | Avoid guessing; the router exists for this case |
| Ambiguous | Fall through | The system needs model judgment |
| Requires current/contextual reasoning | Fall through | The router or downstream tools should decide |
| Requires side effects | Fall through | Deterministic shortcuts should not trigger risky actions |

This keeps the design easy to reason about. The pre-router is not a second router; it is a small collection of safe shortcuts.

---

## Why this should live inside router planning

The deterministic layer should be part of routing rather than a separate route-level shortcut. The reason is architectural consistency.

If the shortcut is placed only in one API route, other callers may still pay the router cost. If it is placed inside the routing step itself, every caller that uses routing benefits from the same optimization. This also keeps the contract clean: the deterministic layer returns the same kind of plan that the router would have returned.

| Placement option | Benefit | Risk |
|---|---|---|
| API route shortcut | Easy to add for one endpoint | Can duplicate logic and miss other callers |
| Executor shortcut | Too late in the flow | Router latency has already been paid |
| Answerer shortcut | Too late and mixes responsibilities | Answering should not decide routing |
| Router-internal pre-router | Best fit | Keeps routing decisions in one place and avoids downstream changes |

Router-internal placement gives the latency benefit without forcing changes to the executor, answerer, streaming logic, or response contract.

---

## Why templates are useful

Templates are valuable because many product and conversational responses do not need creative generation. A template can be faster, more consistent, and less error-prone than asking a model to produce a fresh answer every time.

Good template candidates:

- Greetings and acknowledgements.
- Product capability explanations.
- Pricing or plan summaries, when sourced from current product data.
- Daily limit explanations.
- Simple account-stat summaries.
- “I need more information” clarification prompts.

Poor template candidates:

- Complex technical explanations.
- Personalized study strategy.
- Debugging user-submitted code.
- Open-ended interview preparation.
- Any request where nuance or reasoning quality matters.

Templates should not make the product feel robotic. They should be short, useful, and adapted only where the required data is known.

---

## Expected latency impact by request type

| Request type | Router latency saved | Retrieval/tool latency saved | Overall expected improvement |
|---|---:|---:|---|
| Greeting / thanks / acknowledgement | Yes | Yes | High |
| Short rewrite follow-up | Yes | Usually yes | High |
| Static product answer | Yes | Often yes | High |
| Account/database read | Yes | No, database read still needed | Medium |
| Company interview lookup | Yes | No, retrieval still needed | Medium |
| Concept explanation with retrieval | Yes | No, retrieval still needed | Medium |
| Complex open-ended prompt | No | No | No change, by design |
| Ambiguous prompt | No | No | No change, by design |

The improvement will not be uniform across all traffic. The biggest win comes from the percentage of traffic that falls into simple or repeated categories. Even if the deterministic layer handles only a modest share of requests, it can still improve average and p95 latency because it removes a full model round trip from those requests.

---

## Quality and reliability benefits

The proposal is mainly about latency, but it also improves reliability in several ways.

| Benefit | Explanation |
|---|---|
| Lower variance | Rule-based decisions are more predictable than model-router outputs |
| Fewer malformed router outputs | Deterministic plans do not require parsing model-generated JSON |
| Easier debugging | Each bypass can have a clear reason, such as “greeting” or “rewrite follow-up” |
| Reduced model dependency | Simple turns remain fast even if router model latency fluctuates |
| Better user perception | Short prompts receive short, fast responses instead of waiting behind orchestration |

These benefits matter most in production, where small delays and occasional routing inconsistencies accumulate across many requests.

---

## Main risks

| Risk | Why it matters | Mitigation |
|---|---|---|
| Overmatching vague prompts | The system may skip the router when it should not | Use strict patterns and allowlists |
| Wrong tool selection | A deterministic tool plan may retrieve the wrong data | Start with direct-answer categories before tool-backed categories |
| Poor handling of contextless follow-ups | “go deeper” may lack a subject | Require substantial prior assistant context |
| Stale template answers | Product/pricing/account information can change | Source templates from current config or database-backed values |
| Hard-to-maintain rule sprawl | Too many regexes can become fragile | Keep categories grouped and reviewed |
| Hidden quality regression | Users may receive faster but less appropriate responses | Compare bypassed vs routed samples during rollout |

The most important mitigation is discipline: the deterministic layer should stay small, auditable, and high precision.

---

## Recommended rollout strategy

| Phase | Scope | Why this phase is safe |
|---|---|---|
| Phase 1 | Greetings, thanks, acknowledgements | Lowest risk and easiest to validate |
| Phase 2 | Rewrite and formatting follow-ups | Clear intent, usually no tools needed |
| Phase 3 | History-backed depth changes | Useful but needs prior-context guardrails |
| Phase 4 | Static product/account paths | Strong latency win if backed by reliable data |
| Phase 5 | Tool-backed deterministic plans | Higher impact but requires stricter validation |

This staged approach lets the team capture early latency wins without immediately taking on the highest-risk categories.

---

## Success criteria

The deterministic pre-router should be considered successful if it reduces router calls without reducing answer quality.

Recommended success indicators:

- A meaningful share of requests bypass the router.
- Time to first byte improves for bypassed categories.
- No increase in routing-related fallbacks or user-visible errors.
- No noticeable increase in thumbs-down or correction messages.
- Manual review shows near-zero false positives.
- Engineering can clearly explain every deterministic category.

A good target is not maximum coverage. A good target is **safe coverage**.

---

## Final recommendation

Implement the deterministic pre-router as a conservative routing optimization focused on obvious, repeatable requests. It should not try to understand every message. It should only catch the cases where the correct route is already clear.

The strongest starting point is shallow direct-answer traffic: greetings, thanks, acknowledgements, and simple rewrite follow-ups. These requests produce the highest confidence and the cleanest latency win. After that, the system can expand into history-aware follow-ups, product templates, and eventually high-confidence tool-backed intents.

The expected result is a faster and more predictable user experience, especially for short turns, while preserving the model router for the ambiguous and complex prompts where it is actually valuable.
