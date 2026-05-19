# Deterministic Pre-Router Proposal: Skipping Avoidable Router Calls

## Goal

Add a deterministic decision layer before the model-based router so common, obvious turns do not pay the extra router-model latency. The layer should return the same `ExecutionPlan` object the router already returns today, so executor, answerer, SSE streaming, logging, and downstream contracts stay unchanged.

The intended result is simple:

```text
user message
  -> deterministic pre-router
      -> if confident: return ExecutionPlan immediately
      -> if not confident: fall through to model router
  -> executor
  -> answerer
```

This is not a replacement for the router. It is a high-precision bypass for cases where a model call adds latency but no decision quality.

---

## Proposed code changes

### 1. Add `api/services/chat/pre_router.py`

Create a small, pure-Python module responsible for deterministic plan generation.

Recommended public function:

```python
# api/services/chat/pre_router.py

from __future__ import annotations

import re
from dataclasses import dataclass
from typing import Iterable

from api.services.chat.types import (
    AnswerStyle,
    Depth,
    ExecutionPlan,
    Step,
    ToolCall,
    empty_plan,
)

FLASH = "gemini-2.5-flash"
PRO = "gemini-2.5-pro"


@dataclass(frozen=True)
class PreRouterHit:
    plan: ExecutionPlan
    reason: str


def plan_if_confident(
    user_msg: str,
    history: list[dict] | None = None,
    *,
    model_by_depth: dict[Depth, str] | None = None,
) -> PreRouterHit | None:
    """Return a deterministic plan only for high-confidence cases.

    Return None for anything ambiguous so the model router remains the
    source of truth for hard routing decisions.
    """
    text = _norm(user_msg)
    hist = history or []

    if not text:
        return _direct(Depth.SHALLOW, 300, "empty")

    if _is_bare_greeting(text):
        return _direct(Depth.SHALLOW, 300, "greeting")

    if _is_thanks_or_ack(text):
        return _direct(Depth.SHALLOW, 300, "ack_or_thanks")

    if _is_short_rewrite_request(text):
        return _direct(Depth.SHALLOW, 600, "rewrite_followup")

    if _is_deep_followup(text) and _has_substantial_prior_answer(hist):
        return _direct(Depth.DEEP, 1700, "deep_followup")

    company = _extract_company(text)
    if company and _asks_company_interview_questions(text):
        return _tool_plan(
            tool_name="get_interview_questions",
            args={"q": user_msg},
            depth=Depth.MEDIUM,
            max_tokens=1100,
            cite_sources=False,
            reason="company_interview_questions",
        )

    if _asks_concept_mechanism(text):
        return _tool_plan(
            tool_name="get_knowledge_context",
            args={"q": user_msg},
            depth=Depth.DEEP,
            max_tokens=1700,
            cite_sources=True,
            reason="concept_mechanism",
        )

    return None
```

Supporting helpers should be intentionally conservative:

```python
def _norm(s: str) -> str:
    return re.sub(r"\s+", " ", (s or "").strip().lower())


def _direct(depth: Depth, max_tokens: int, reason: str) -> PreRouterHit:
    model = FLASH if depth == Depth.SHALLOW else PRO
    return PreRouterHit(
        plan=empty_plan(
            model=model,
            style=AnswerStyle(
                depth=depth,
                cite_sources=False,
                max_output_tokens=max_tokens,
                temperature=0.3,
            ),
            confidence=1.0,
            rationale=f"deterministic:{reason}",
        ),
        reason=reason,
    )


def _tool_plan(
    *,
    tool_name: str,
    args: dict,
    depth: Depth,
    max_tokens: int,
    cite_sources: bool,
    reason: str,
) -> PreRouterHit:
    model = FLASH if depth == Depth.SHALLOW else PRO
    return PreRouterHit(
        plan=ExecutionPlan(
            steps=(Step(calls=(ToolCall(name=tool_name, args=args),), label="prefetch"),),
            answerer_model=model,
            answer_style=AnswerStyle(
                depth=depth,
                cite_sources=cite_sources,
                max_output_tokens=max_tokens,
                temperature=0.3,
            ),
            confidence=1.0,
            rationale=f"deterministic:{reason}",
        ),
        reason=reason,
    )
```

---

### 2. Call `pre_router.plan_if_confident()` at the top of `router.plan()`

Patch `api/services/chat/router.py` so the deterministic layer runs after the empty-message guard and before `_call_router_model()`.

Proposed patch shape:

```python
from api.services.chat import pre_router


def plan(...):
    if not user_msg or not user_msg.strip():
        return empty_plan(model=_fallback_model(model_by_depth))

    pre = pre_router.plan_if_confident(
        user_msg,
        history=history or [],
        model_by_depth=model_by_depth,
    )
    if pre is not None:
        log.info("router: deterministic_pre_router reason=%s", pre.reason)
        return pre.plan

    raw = _call_router_model(
        user_msg=user_msg,
        history=_trim_history(history or []),
        registry=registry,
        model_chain=model_chain,
    )
    ...
```

Why this insertion point is best:

- It preserves the existing `ExecutionPlan` contract.
- It keeps executor and answerer untouched.
- It works for any caller of `router.plan()`, not only one FastAPI route.
- It keeps the model router as the fallback for ambiguous cases.

---

### 3. Add a deterministic category registry

Avoid scattering regexes across the function body. Keep rules grouped by outcome.

Recommended categories:

```python
BARE_GREETINGS = {
    "hi", "hello", "hey", "yo", "sup", "good morning", "good evening",
}

ACKS_AND_THANKS = {
    "ok", "okay", "k", "cool", "got it", "sounds good", "thanks",
    "thank you", "thx", "ty", "yes", "yep", "sure", "continue",
}

SHORT_REWRITE_PATTERNS = (
    "make it shorter",
    "summarize that",
    "say that again",
    "rephrase that",
    "give me one example",
    "explain simply",
)

DEEP_FOLLOWUP_PATTERNS = (
    "go deeper",
    "explain more",
    "walk me through",
    "why does that happen",
    "show the tradeoffs",
    "give the mechanism",
)

COMPANY_INTERVIEW_PATTERNS = (
    "what did .* ask",
    "questions asked at .*",
    ".* interview questions",
    ".* recently asked",
)

CONCEPT_MECHANISM_PATTERNS = (
    "how does .* work",
    "explain how .* works",
    "walk me through .*",
    ".* tradeoffs",
    ".* vs .*",
)
```

Keep this list high precision. The pre-router should prefer false negatives over false positives. A false negative only costs one router call. A false positive can pick the wrong tool or wrong answer depth.

---

### 4. Add a company allowlist for deterministic interview-question routing

Do not infer that every capitalized word is a company. Use an allowlist or known-company table.

Minimum static allowlist:

```python
KNOWN_COMPANIES = {
    "google", "meta", "facebook", "amazon", "apple", "microsoft",
    "netflix", "uber", "airbnb", "stripe", "doordash", "linkedin",
    "salesforce", "oracle", "adobe", "nvidia", "tesla", "databricks",
}
```

Better option: load known company names from the same source used by interview-question filtering, cache it in memory, and refresh periodically. That keeps deterministic routing aligned with the actual database.

Rule:

```text
Only deterministic-route company interview questions when:
1. the message matches an interview-question intent, and
2. exactly one known company is present, and
3. the request does not require user-specific context or clarification.
```

Ambiguous examples that should fall through:

```text
"google this for me"
"apple vs orange"
"meta question"
"what should I prepare?"
```

---

### 5. Add a history-aware deep-followup guard

The phrase `go deeper` should not always force a deep plan. It should only bypass the router when there is a meaningful previous assistant answer.

Recommended helper:

```python
def _has_substantial_prior_answer(history: list[dict]) -> bool:
    for msg in reversed(history):
        if msg.get("role") == "assistant":
            content = str(msg.get("content") or "")
            return len(content.split()) >= 80
    return False
```

This prevents a new empty chat containing only `go deeper` from getting an expensive Pro answer without context.

---

### 6. Add feature flags

Ship behind explicit environment flags.

```bash
CHAT_PRE_ROUTER_ENABLED=1
CHAT_PRE_ROUTER_TOOLS_ENABLED=0
CHAT_PRE_ROUTER_LOG_ONLY=0
```

Recommended rollout:

1. `CHAT_PRE_ROUTER_ENABLED=1`, `CHAT_PRE_ROUTER_TOOLS_ENABLED=0`
   - Only direct-answer and follow-up bypasses.
2. Enable tool plans only after logs show good precision.
3. Use `CHAT_PRE_ROUTER_LOG_ONLY=1` in staging to record what would have skipped without changing behavior.

Pseudo-code:

```python
if os.getenv("CHAT_PRE_ROUTER_ENABLED", "0") == "1":
    pre = pre_router.plan_if_confident(...)
    if pre is not None:
        if os.getenv("CHAT_PRE_ROUTER_LOG_ONLY", "0") == "1":
            log.info("pre_router_would_hit reason=%s", pre.reason)
        else:
            return pre.plan
```

---

### 7. Extend telemetry

Add fields that make router savings measurable.

In plan logs, include:

```json
{
  "route_source": "deterministic_pre_router",
  "pre_router_reason": "ack_or_thanks",
  "router_skipped": true,
  "router_ms_saved_estimate": 700,
  "model_router_called": false
}
```

For normal model-router turns:

```json
{
  "route_source": "model_router",
  "router_skipped": false,
  "model_router_called": true
}
```

In the existing turn-budget log, add:

```text
route_source=%s pre_router_reason=%s
```

This lets us answer three important questions from logs:

1. What percentage of turns skip the router?
2. Which deterministic categories produce the most savings?
3. Did skipped turns have lower thumbs-up or higher retry/edit rates?

---

### 8. Add tests

Add a new test file:

```text
tests/test_chat_pre_router.py
```

Core unit tests:

```python
def test_greeting_returns_shallow_flash_plan(): ...
def test_ack_returns_shallow_flash_plan(): ...
def test_short_rewrite_returns_shallow_flash_plan(): ...
def test_deep_followup_requires_prior_assistant_answer(): ...
def test_deep_followup_with_prior_answer_returns_deep_plan(): ...
def test_company_interview_question_returns_interview_tool_plan(): ...
def test_google_this_falls_through(): ...
def test_concept_mechanism_returns_knowledge_tool_plan(): ...
def test_ambiguous_prompt_returns_none(): ...
```

Router integration tests:

```python
def test_router_plan_skips_model_when_pre_router_hits(reg):
    with patch.object(router, "_call_router_model") as spy:
        plan = router.plan("thanks!", registry=reg)
    assert not spy.called
    assert plan.rationale.startswith("deterministic:")


def test_router_plan_falls_through_when_pre_router_misses(reg):
    with patch.object(router, "_call_router_model", return_value=router_json) as spy:
        plan = router.plan("teach me raft leader election", registry=reg)
    assert spy.called
```

Pipeline test:

```python
def test_pipeline_logs_pre_router_source_for_skipped_turn(...): ...
```

---

### 9. Add a benchmark script

Create:

```text
scripts/bench_pre_router.py
```

Run a fixed prompt set through `router.plan()` with and without the pre-router enabled.

Prompt buckets:

```text
- greetings / thanks / acknowledgements
- short rewrites
- deep follow-ups
- company interview questions
- concept mechanism questions
- ambiguous prompts
- full mock interview prompts
```

Metrics:

```text
- pre-router hit rate
- false-positive count from manually labeled prompts
- router_ms median / p95
- TTFB median / p95
- model-router calls avoided
- estimated Gemini router cost avoided
```

Target acceptance criteria:

```text
- >= 20% router-call reduction on real traffic sample
- 0 known false positives in manually labeled test set
- no increase in fallback/error rate
- no degradation in thumbs-up/down ratio after rollout
```

---

## Matching policy

The pre-router should be strict.

Use this rule:

```text
If the deterministic layer cannot explain exactly why a plan is safe,
return None and let the model router decide.
```

Safe to bypass:

```text
"hi"
"thanks"
"ok continue"
"make it shorter"
"go deeper" after a substantial assistant answer
"what did Google ask recently?"
"how does Raft leader election work?"
```

Must fall through:

```text
"help"
"prep me"
"what should I do next?"
"google this"
"is this right?"
"review this" with no user answer attached
"design Twitter"
"compare these options" with unclear referents
```

---

## Latency impact

The deterministic pre-router saves the entire model-router call for hit cases. Because it returns an `ExecutionPlan` immediately, the request can move directly to executor/answerer. For direct-answer cases with no tools, this should reduce user-perceived TTFB by roughly the router latency amount.

Expected impact by category:

| Category | Router skipped? | Tool skipped? | Answerer model | Expected win |
|---|---:|---:|---|---|
| Greeting / thanks / ack | Yes | Yes | Flash | High |
| Short rewrite follow-up | Yes | Yes | Flash | High |
| Deep follow-up with history | Yes | Usually yes | Pro | Medium |
| Company interview lookup | Yes | No | Pro | Medium |
| Concept mechanism lookup | Yes | No | Pro | Medium |
| Ambiguous request | No | No | Router decides | None, safe fallback |

For tool-backed deterministic plans, the system still pays retrieval/tool latency, but it avoids the router-model decision latency. For direct-answer plans, it avoids both router latency and unnecessary executor span overhead.

---

## Rollout plan

### Phase 1: Direct-answer bypass only

Enable:

```bash
CHAT_PRE_ROUTER_ENABLED=1
CHAT_PRE_ROUTER_TOOLS_ENABLED=0
```

Allowed categories:

```text
- greeting
- thanks / acknowledgement
- short rewrite
- history-backed deep follow-up
```

This is the safest phase because it does not choose tools.

### Phase 2: Log-only tool plans

Enable:

```bash
CHAT_PRE_ROUTER_LOG_ONLY=1
CHAT_PRE_ROUTER_TOOLS_ENABLED=1
```

Record proposed tool plans but continue using the model router. Compare deterministic decisions against actual router decisions.

Promote a category only when deterministic decisions match or improve model-router behavior on real traffic.

### Phase 3: Deterministic tool routing for high-confidence patterns

Enable tool bypasses for:

```text
- known company + interview-question intent -> get_interview_questions
- explicit mechanism question -> get_knowledge_context
```

Keep all other tool decisions with the model router.

### Phase 4: Tune from logs

Every week, review:

```text
- top pre-router hit categories
- pre-router miss examples that could be safely added
- false-positive reports
- thumbs-down/retry rate by route_source
- p50/p95 TTFB change by route_source
```

Only add new deterministic rules when they are obvious and testable.

---

## Safety guardrails

1. Never return a tool plan with missing required args.
2. Never route side-effect tools from the pre-router.
3. Never force Pro for contextless short prompts.
4. Never deterministic-route company names unless exactly one known company is detected.
5. Never deterministic-route vague prompts like `help`, `prep me`, or `review this`.
6. Always keep model-router fallback available.
7. Always log `rationale="deterministic:<reason>"` so every bypass is auditable.

---

## Implementation checklist

```text
[ ] Add api/services/chat/pre_router.py
[ ] Add conservative deterministic categories
[ ] Add company allowlist or DB-backed company-name cache
[ ] Add feature flags
[ ] Patch router.plan() to call pre_router before _call_router_model()
[ ] Add route_source / pre_router_reason telemetry
[ ] Add unit tests for pre_router
[ ] Add router integration tests proving _call_router_model is skipped
[ ] Add pipeline/logging test
[ ] Add benchmark script
[ ] Roll out direct-answer bypass first
[ ] Run tool-routing in log-only mode
[ ] Promote high-confidence tool rules after traffic validation
```

---

## Final recommendation

Implement this as a router-internal optimization, not as a route-level shortcut. The deterministic layer should return the existing `ExecutionPlan` shape and live directly inside `router.plan()`. That gives us the latency win while preserving the current architecture, contracts, tests, and fallback behavior.

Start with shallow direct-answer and history-backed follow-up rules, then add deterministic tool plans only after log-only validation. This approach reduces router latency immediately while keeping the risk of wrong routing very low.
