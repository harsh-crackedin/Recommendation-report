# Contextual Hybrid RAG + Gated Rerank Report

## Objective

The current retrieval pipeline uses a local BGE reranker, which is CPU-intensive and memory-heavy. The goal is to determine whether the project can safely skip the local BGE reranker for some or all requests without degrading answer quality, and to propose an implementation plan for reducing latency and resource usage.

## Executive Summary

The BGE reranker should not be removed immediately. Instead, the project should implement **Contextual Hybrid RAG + gated rerank**.

The recommended approach is:

1. Keep the existing first-stage hybrid retrieval pipeline.
2. Use Voyage embeddings, BM25, and Reciprocal Rank Fusion as the default retrieval foundation.
3. Add a deterministic confidence gate after RRF.
4. Skip the local BGE reranker when RRF confidence is high.
5. Run BGE only when retrieval signals are uncertain.
6. Add separate scoring and score-floor logic for non-BGE paths.
7. Validate quality through shadow evaluation and staged rollout.

This should reduce CPU load, memory pressure, and retrieval latency while preserving quality for ambiguous or semantically complex queries.

## Problem

The local BGE reranker creates three major issues:

### 1. CPU latency

Cross-encoder reranking is expensive because every candidate passage must be scored together with the query. If the system reranks 20 candidates, it performs 20 query-passage inference operations.

This creates avoidable latency when the first-stage hybrid retrieval already has a confident result.

### 2. Memory pressure

The local BGE reranker requires hundreds of megabytes of memory. On small CPU instances, this contributes to high RAM usage and increases the risk of out-of-memory failures under load.

### 3. Startup and warmup cost

If the reranker is warmed at application startup, memory is consumed even before any request actually needs reranking. This is inefficient for simple queries where hybrid retrieval alone is sufficient.

## Key Insight

The project already has a strong retrieval foundation before BGE runs:

```text
Voyage vector retrieval + BM25 lexical retrieval + RRF merge
```

This means many queries can likely be answered using RRF-ranked candidates without invoking the local BGE model.

BGE is most useful when the retrieval signals disagree or when the query requires deeper semantic matching. It is less necessary when vector search and BM25 agree on the same top candidates.

## Recommendation

Implement **gated rerank** instead of fully removing BGE.

The reranker should become conditional:

```text
If hybrid retrieval confidence is high:
    skip BGE
    use normalized RRF proxy scores
else:
    run BGE reranker
    use BGE scores
```

This keeps quality protection for difficult queries while avoiding unnecessary local model inference for easy queries.

## Proposed Architecture

```text
User query
  -> Voyage query embedding
  -> Vector search
  -> BM25 search
  -> RRF merge
  -> Confidence gate
      -> High confidence: skip BGE, use RRF proxy scoring
      -> Low confidence: run BGE reranker
  -> Score floor filtering
  -> Reputation / diversity / neighbor expansion
  -> Final context
```

## Rerank Gate Design

Add a deterministic decision layer after RRF and before reranking.

The gate should inspect cheap retrieval features, including:

- Whether the top result appears in both vector and BM25 results.
- The RRF score gap between top-1 and top-2 candidates.
- Agreement between vector and BM25 results in the top 5.
- Number of fused candidates.
- Presence of rare exact query tokens.
- Whether the query is simple or complex.
- Whether candidate scores are near the refusal or confidence floor.

## Low-Risk Skip Conditions

BGE can usually be skipped when:

- The top result appears in both vector and BM25 retrieval.
- Top-1 has a clear RRF margin over top-2.
- Multiple top candidates are agreed upon by both retrieval systems.
- The query contains exact technical terms that BM25 matches strongly.
- The query is a simple lookup, definition, or factual retrieval request.
- The fused candidate list is small and consistent.

Examples:

```text
"What is quorum N=3 R=2 W=2?"
"Explain master-master replication"
"What is the daily limit?"
"Show my plan"
```

## High-Risk Rerank Conditions

BGE should still run when:

- Vector and BM25 disagree on top candidates.
- Top-1 and top-2 RRF scores are very close.
- Query wording is semantic or paraphrased.
- The query asks for comparison, reasoning, tradeoffs, or caveats.
- The candidate pool is large and diverse.
- The system is close to the answer refusal threshold.
- Multiple sources contain overlapping but conflicting information.

Examples:

```text
"Why is active-active replication hard?"
"Compare quorum consistency with leader-based replication"
"What are the tradeoffs of skipping rerank?"
"Explain why this architecture may fail under load"
```

## Testing Plan

Add tests for the following cases.

### 1. High-confidence RRF skips BGE

Scenario:

- Top chunk appears in both vector and BM25 results.
- RRF gap is large.
- Agreement exists in top 5.

Expected result:

- BGE reranker is not called.
- Final chunks are returned using RRF proxy scores.

### 2. Retrieval disagreement calls BGE

Scenario:

- Vector top result and BM25 top result are different.
- RRF gap is small.

Expected result:

- BGE reranker is called.

### 3. `RERANK_MODE=off` never loads BGE

Scenario:

- Rerank mode is disabled.
- Model import or model load is monkeypatched to fail.

Expected result:

- Retrieval still succeeds.
- BGE is not imported or initialized.

### 4. Separate score floors work correctly

Scenario:

- One request uses BGE scores.
- Another request uses RRF proxy scores.

Expected result:

- BGE path uses `SCORE_FLOOR`.
- RRF path uses `PROXY_SCORE_FLOOR`.

### 5. Observability includes gate decision

Scenario:

- Run both skip and rerank paths.

Expected result:

- Logs include `score_source`, `rerank_used`, `reason`, and gate features.

## Evaluation Plan

Before enabling gated rerank in production, run an offline evaluation script:

```bash
scripts/eval_rerank_gate.py
```

The script should replay a representative query set through three modes:

```text
always: current BGE behavior
gated: proposed confidence-gated behavior
off: pure RRF proxy behavior
```

Measure:

- Top-1 agreement with current BGE behavior.
- Top-5 overlap.
- Citation correctness.
- False refusal rate.
- False positive answer rate.
- Retrieval p50 latency.
- Retrieval p95 latency.
- Rerank p95 latency.
- Process RSS memory.
- Number and percentage of requests that skip BGE.

Suggested acceptance criteria:

```text
Top-1 agreement: >= 95%
Top-5 overlap: >= 98%
False citation increase: 0 meaningful increase
False refusal increase: 0 meaningful increase
p95 retrieval latency improvement: >= 150-300 ms on CPU
Memory improvement: BGE not loaded for skipped/off paths
```

## Rollout Plan

### Phase 1: Ship safely with no behavior change

Add the new code paths but keep:

```bash
RERANK_MODE=always
RERANK_WARMUP=1
```

This ensures production behavior remains unchanged.

### Phase 2: Run offline evaluation

Use real and curated queries to compare:

```text
BGE always vs gated rerank vs RRF-only
```

Tune the gate thresholds based on the results.

### Phase 3: Enable gated mode in staging

Use:

```bash
RERANK_MODE=gated
RERANK_WARMUP=0
```

Monitor:

- Skipped rerank percentage.
- Latency improvements.
- Answer quality.
- Refusal rate.
- Memory usage.

### Phase 4: Limited production rollout

Enable gated rerank for a small production slice.

Keep `RERANK_MODE=always` available as an instant rollback.

### Phase 5: Make gated rerank default

If quality metrics hold, switch default production settings to:

```bash
RERANK_MODE=gated
RERANK_WARMUP=0
```

## Expected Benefits

### Latency reduction

Skipping BGE removes local cross-encoder inference from high-confidence requests. This should reduce p50 and p95 retrieval latency, especially on CPU-only instances.

### Lower memory usage

Lazy loading or skipping BGE prevents the local reranker from occupying memory on simple requests.

### Better stability

Reducing local model memory pressure lowers the risk of OOM errors on small machines.

### Preserved answer quality

Keeping BGE for uncertain cases protects quality on harder queries where neural reranking is still valuable.

## Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---:|---|
| RRF proxy scores are poorly calibrated | Good chunks may be filtered out or bad chunks may pass | Use a separate `PROXY_SCORE_FLOOR` and tune from logs |
| Gate skips BGE on ambiguous queries | Lower answer quality | Conservative initial thresholds and shadow evaluation |
| BM25 exact match over-trusts keyword matches | Wrong chunk selected due to token overlap | Require agreement signals, not BM25 alone |
| BGE lazy loading creates occasional cold-start latency | One request may become slower | Optional warmup in larger environments; keep `RERANK_WARMUP` configurable |
| Observability is insufficient | Cannot tune gate safely | Log decision features and score source for every retrieval |

## Final Recommendation

Do **not** fully remove the BGE reranker yet.

Implement **Contextual Hybrid RAG + gated rerank**:

```text
Hybrid retrieval first.
RRF confidence gate second.
BGE only when needed.
RRF proxy scoring when skipped.
Separate score floors.
Full observability and staged rollout.
```

This gives the project the latency and memory benefits of skipping BGE for easy requests while preserving BGE’s ranking quality for hard or ambiguous requests.

The most important immediate change is to stop unconditional BGE warmup when gated or off mode is enabled. That directly reduces startup memory pressure and avoids loading the local model for requests that do not need it.
