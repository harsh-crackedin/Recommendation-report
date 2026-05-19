# Deterministic Pre-Router Layer: Reducing Chat Latency by Skipping Avoidable Router Calls

## Executive summary

The current router-first chat stack is already well-structured for this optimization. The code separates the chat turn into three stages:

```text
messages -> router.plan() -> ExecutionPlan
plan -> executor.run() -> dict[tool_name, ToolResult]
plan + context -> answerer.stream() -> Iterator[str]
```

That means we do **not** need to rewrite the executor, answerer, SSE route, or tool system. The lowest-risk change is to add a deterministic layer **inside `api/services/chat/router.py`, before `_call_router_model()`**. This layer should return a normal `ExecutionPlan` for high-confidence cases and return `None` for everything else. When it returns `None`, the existing Gemini router path runs unchanged.

The expected latency win is direct: every deterministic hit skips the non-streaming router model call, the router prompt build, network round trip, JSON parse, and route-generation observability span. The pipeline already records `router_ms`, `executor_ms`, TTFB, selected model, depth, tools, and chunk count, so rollout can be measured without building a new metrics system.

## What the current code is doing

### 1. The router-first pipeline is opt-in at the route layer

In `api/routes/chat.py`, the streaming endpoint selects the new router-first pipeline only when:

```python
os.getenv("CHAT_PIPELINE", "legacy") == "router-first"
```

Otherwise it uses the legacy `llm_stream.stream_chat`. The same route already has a deterministic `try_shortcut(req.content)` path for bare greetings and model-identity questions. That shortcut skips the LLM entirely and returns a prewritten response over SSE.

This means the product already accepts deterministic bypasses for safe cases. The proposed change extends that idea, but at the **planning layer** rather than the route layer.

### 2. `pipeline.stream()` calls the router first on every non-empty router-first turn

In `api/services/chat/pipeline.py`, the pipeline splits the latest user message from history, starts a timer, and calls:

```python
plan = router.plan(
    user_msg=user_msg,
    history=history,
    user_id=user_id,
    registry=registry,
)
router_ms = int((time.monotonic() - t0) * 1000)
```

Then it runs the executor, streams the answerer, and logs:

```text
total_ms, ttfb_ms, router_ms, executor_ms, answerer_ms, stream_ms,
output_chars, model, depth, tools, chunks
```

Important implication: if we skip the router model call inside `router.plan()`, the existing `router_ms` field will immediately show the impact. TTFB should improve because router + executor + answerer pre-token work all happen before the first streamed answer token.

### 3. `router.plan()` currently short-circuits only empty input

In `api/services/chat/router.py`, `plan()` currently does this:

```python
if not user_msg or not user_msg.strip():
    return empty_plan(model=_fallback_model(model_by_depth))

raw = _call_router_model(...)
if raw is None:
    return _fallback_plan(...)

parsed = _safe_parse(raw)
if parsed is None:
    return _fallback_plan(...)

return _to_plan(parsed, registry=registry, model_by_depth=model_by_depth)
```

So for normal user text, even simple messages like `thanks`, `ok`, `make it shorter`, or `go deeper` currently pay the router model-call cost when using the router-first pipeline.

### 4. The router model call is intentionally a classification step

The router file explicitly says routing is classification and argument filling, not final-answer generation. The router model chain is:

```python
_ROUTER_MODEL_CHAIN = (
    "gemini-2.5-flash",
    "gemini-2.5-flash-lite",
)
```

The router uses `thinkingBudget: 0`, low temperature, JSON mode, and an 8-second timeout. The code comments say warm Flash-Lite was around 700ms, but cold start and network can exceed 4s. This is exactly the kind of fixed pre-answer latency we should avoid for obvious turns.

### 5. The `ExecutionPlan` contract is already the correct interface

`api/services/chat/types.py` defines `ExecutionPlan` with:

- `steps`: prefetch tool calls, or empty for direct answer
- `answerer_model`: e.g. `gemini-2.5-flash` or `gemini-2.5-pro`
- `answer_style`: depth, token cap, temperature, citation flag
- `confidence`
- `rationale`

It also defines:

```python
def empty_plan(model: str, style: AnswerStyle | None = None) -> ExecutionPlan:
    return ExecutionPlan(
        steps=(),
        answerer_model=model,
        answer_style=style or AnswerStyle(),
    )
```

Therefore the deterministic layer should return the same `ExecutionPlan` shape. No new pipeline contract is needed.

## Proposed change

Add a new module:

```text
api/services/chat/pre_router.py
```

Its public function should be:

```python
def plan_if_confident(
    user_msg: str,
    history: list[dict] | None = None,
    *,
    registry=None,
    model_by_depth: dict[Depth, str] | None = None,
) -> ExecutionPlan | None:
    ...
```

It should have one rule: **return an `ExecutionPlan` only when deterministic confidence is very high; otherwise return `None`.**

Then modify `router.plan()`:

```python
if not user_msg or not user_msg.strip():
    return empty_plan(model=_fallback_model(model_by_depth))

from api.services.chat import pre_router

pre = pre_router.plan_if_confident(
    user_msg,
    history=history or [],
    registry=registry,
    model_by_depth=model_by_depth,
)
if pre is not None:
    return pre

raw = _call_router_model(...)
```

This preserves the current fallback behavior: if deterministic rules do not fire, Gemini routing continues exactly as today.

## Deterministic categories to support first

### Category A: exact casual direct replies

These should return a shallow direct plan using Flash:

```python
empty_plan(
    model="gemini-2.5-flash",
    style=AnswerStyle(
        depth=Depth.SHALLOW,
        max_output_tokens=300,
        cite_sources=False,
    ),
)
```

Examples:

```text
hi
hello
hey
thanks
thank you
ok
okay
got it
cool
sounds good
```

This category is safe because it needs no tool, no Pro model, and no citations. It duplicates the spirit of the existing route-level `try_shortcut`, but returns an `ExecutionPlan` instead of bypassing the pipeline.

### Category B: simple rewrite / formatting follow-ups

Examples:

```text
make it shorter
summarize that
rephrase it
make it simpler
explain in one paragraph
give me bullets
```

These should usually use Flash with `Depth.SHALLOW` or `Depth.MEDIUM`, no tools, and no citations. They need history, but not retrieval.

Recommended first version: only fire when there is at least one assistant message in recent history and the user message is short, e.g. fewer than 12 words.

### Category C: obvious depth follow-ups without new retrieval

Examples:

```text
go deeper
explain more
why?
walk me through it
what are the tradeoffs?
```

These should return a no-tool plan with `Depth.DEEP` when recent history already contains a substantive assistant answer. This avoids asking the router to infer that `go deeper` refers to the previous turn.

Recommended guardrails:

- require at least one recent assistant message
- require the latest assistant message to be non-trivial, e.g. more than 400 characters
- do not fire if the user mentions a new company, date, technology, or asks for “recent/latest/current”

### Category D: high-precision tool plans

This is optional for phase 1. It is more powerful but riskier than direct-answer templates.

Only add deterministic tool plans for patterns already hard-coded in the router prompt:

1. Company interview question lookups should call `get_interview_questions`.
2. Concept mechanism questions should call `get_knowledge_context`.

Example deterministic plan for company interview questions:

```python
ExecutionPlan(
    steps=(Step(calls=(ToolCall(
        name="get_interview_questions",
        args={"q": user_msg},
    ),), label="prefetch"),),
    answerer_model="gemini-2.5-pro",
    answer_style=AnswerStyle(
        depth=Depth.MEDIUM,
        max_output_tokens=1100,
        cite_sources=False,
    ),
    confidence=1.0,
    rationale="deterministic:company_interview_lookup",
)
```

Phase 1 can skip this category and still get meaningful latency gains from casual/follow-up traffic.

## Guardrails

The deterministic layer must be conservative. It should **not** try to replace the router. It should skip the router only for exact or near-exact patterns where the correct plan is obvious.

Recommended guardrails:

1. **No fuzzy company detection in phase 1.** Avoid parsing “google this” as company Google.
2. **No deterministic retrieval for latest/current/recent unless the tool call is obvious.** These are easy to misroute.
3. **No deterministic mock classification unless explicit.** Full system-design and evaluation prompts should keep using the router initially.
4. **Never emit unknown tool names.** Use the registry to verify a tool exists before returning a tool plan.
5. **Mark deterministic plans clearly.** Use rationale values like `deterministic:casual_ack`, `deterministic:rewrite_followup`, `deterministic:deep_followup`.
6. **Keep confidence at `1.0` only for rules that are truly exact.** For weaker rules, do not return a plan; fall through to Gemini.

## Implementation plan

### Step 1: Add `pre_router.py`

Create helper functions:

```python
def _norm(text: str) -> str:
    return " ".join(text.lower().strip().split())


def _has_recent_assistant(history: list[dict], min_chars: int = 1) -> bool:
    return any(
        m.get("role") == "assistant" and len(str(m.get("content", ""))) >= min_chars
        for m in reversed(history[-6:])
    )
```

Then implement exact sets:

```python
GREETINGS = {"hi", "hello", "hey", "yo"}
ACKS = {"ok", "okay", "k", "got it", "cool", "sounds good", "yes", "yep"}
THANKS = {"thanks", "thank you", "thx", "ty"}
REWRITE_FOLLOWUPS = {
    "make it shorter", "summarize that", "rephrase it",
    "make it simpler", "give me bullets",
}
DEEP_FOLLOWUPS = {
    "go deeper", "explain more", "walk me through it",
    "what are the tradeoffs", "why",
}
```

### Step 2: Call it from `router.plan()`

Place it immediately after the existing empty-input short-circuit and before `_call_router_model()`.

This is the best insertion point because:

- `router.plan()` already owns planning.
- The pipeline remains unchanged.
- Tests can patch `_call_router_model` and assert it was not called.
- Every router-first caller benefits.

### Step 3: Add tests

Add unit tests in `tests/test_chat_router.py`:

```python
def test_pre_router_exact_ack_skips_model(reg):
    with patch.object(router, "_call_router_model") as spy:
        plan = router.plan("thanks", registry=reg)
    assert not spy.called
    assert plan.is_direct_answer
    assert plan.answerer_model == "gemini-2.5-flash"
    assert plan.answer_style.depth == Depth.SHALLOW
    assert plan.rationale.startswith("deterministic:")


def test_pre_router_rewrite_followup_requires_history(reg):
    with patch.object(router, "_call_router_model", return_value='{"tool_calls":[],"depth":"shallow","cite_sources":false,"rationale":"fallback"}') as spy:
        router.plan("make it shorter", history=[], registry=reg)
    assert spy.called


def test_pre_router_deep_followup_skips_model_with_history(reg):
    history = [{"role": "assistant", "content": "x" * 500}]
    with patch.object(router, "_call_router_model") as spy:
        plan = router.plan("go deeper", history=history, registry=reg)
    assert not spy.called
    assert plan.answer_style.depth == Depth.DEEP
```

Add one integration test in `tests/test_chat_pipeline.py`:

```python
def test_pipeline_pre_router_ack_skips_router_model_and_streams(reg):
    with patch.object(router, "_call_router_model") as spy:
        with patch.object(answerer, "_consume_stream", side_effect=_fake_consume):
            chunks = list(pipeline.stream([
                {"role": "user", "content": "thanks"}
            ], registry=reg))
    assert not spy.called
    assert chunks
```

### Step 4: Add observability

The existing pipeline log already includes `router_ms` and `rationale`. Add a deterministic-specific prefix:

```text
deterministic:casual_ack
deterministic:rewrite_followup
deterministic:deep_followup
deterministic:company_interview_lookup
```

Then analyze:

- deterministic hit rate
- p50/p95 `router_ms` for deterministic vs model-routed turns
- p50/p95 TTFB for deterministic vs model-routed turns
- feedback / thumbs-down rate split by deterministic rationale
- fallback percentage from deterministic categories that users immediately rephrase

### Step 5: Rollout safely

Suggested rollout:

1. Add `PRE_ROUTER_ENABLED=false` env flag.
2. Enable in development.
3. Enable for internal/admin traffic.
4. Enable for 5-10% of authenticated router-first traffic.
5. Expand only if quality metrics stay flat and TTFB improves.

## Expected impact

The direct savings per deterministic hit are:

- no Gemini router API call
- no router model-chain fallback attempt
- no router JSON parsing and mapping
- no router prompt/catalog construction
- lower time to first streamed token
- lower cost from avoided input/output tokens

The largest user-visible improvement should appear on casual turns and short follow-ups because they currently pay a fixed pre-answer router cost despite needing no tool decision.

## Recommended first PR scope

Keep the first PR intentionally small:

1. Add `api/services/chat/pre_router.py`.
2. Wire it into `router.plan()` behind `PRE_ROUTER_ENABLED`.
3. Support only:
   - exact greetings / thanks / acknowledgements
   - exact rewrite follow-ups with history
   - exact deep follow-ups with substantial assistant history
4. Add router and pipeline tests proving `_call_router_model()` is not called.
5. Ship telemetry using existing `rationale` and `router_ms` fields.

Do **not** add broad semantic classification, company extraction, or complex tool routing in the first PR. Those are phase 2 optimizations after we validate that deterministic direct-answer planning improves latency without hurting quality.

## Final recommendation

Implement the deterministic layer **inside `router.plan()`**, not in `pipeline.stream()` and not only in `api/routes/chat.py`.

Reason: this keeps planning responsibility in one place, preserves the `ExecutionPlan` contract, uses existing executor/answerer behavior, and gives us immediate latency measurement through the current pipeline logs. The design is low-risk because unknown or ambiguous turns still fall through to the existing Gemini router path unchanged.
