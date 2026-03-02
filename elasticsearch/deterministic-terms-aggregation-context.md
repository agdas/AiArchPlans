# Deterministic Terms Aggregation — Context Summary

## Feature Goal

Make Elasticsearch's `terms` aggregation deterministic with zero error margin using an efficient distributed Top-K algorithm (TPUT).

---

## 1. Repository Involved

Only **elasticsearch** — all aggregation code lives in the `server` module:

```
server/src/main/java/org/elasticsearch/search/aggregations/
├── bucket/
│   ├── terms/                    # Core terms aggregation code
│   │   ├── TermsAggregationBuilder.java
│   │   ├── TermsAggregatorFactory.java
│   │   ├── TermsAggregator.java
│   │   ├── AbstractInternalTerms.java
│   │   ├── InternalTerms.java
│   │   ├── GlobalOrdinalsStringTermsAggregator.java
│   │   ├── MapStringTermsAggregator.java
│   │   ├── NumericTermsAggregator.java
│   │   ├── AbstractStringTermsAggregator.java
│   │   ├── RareTermsAggregationBuilder.java
│   │   └── ...
│   ├── BucketUtils.java          # shard_size heuristic
│   └── IteratorAndCurrent.java   # Merge-sort helper
├── TopBucketBuilder.java         # Priority queue top-N selection
├── DelayedBucket.java            # Lazy bucket merging
├── AggregatorReducer.java        # Standard reduce interface
├── AggregationReduceContext.java # Partial vs Final reduce context
└── ...
```

---

## 2. Current Terms Aggregation Implementation

### How It Works Today (Single-Round Approximate)

```
┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
│   Shard 1        │   │   Shard 2        │   │   Shard N        │
│                  │   │                  │   │                  │
│ Collect top      │   │ Collect top      │   │ Collect top      │
│ shard_size terms │   │ shard_size terms │   │ shard_size terms │
└────────┬─────────┘   └────────┬─────────┘   └────────┬─────────┘
         │                      │                       │
         │    shard_size terms   │    shard_size terms   │
         └──────────────────────┼───────────────────────┘
                                │
                    ┌───────────▼───────────┐
                    │   Coordinator Node    │
                    │                       │
                    │  Merge all shard      │
                    │  results, pick top    │
                    │  `size` buckets       │
                    │                       │
                    │  Report error bounds: │
                    │  doc_count_error_     │
                    │  upper_bound          │
                    └───────────────────────┘
```

### Shard-Side (Data Nodes)

- **Base class**: `TermsAggregator` extends `DeferableBucketAggregator`
- **Concrete implementations**:
  - `GlobalOrdinalsStringTermsAggregator` — keyword/string fields using global ordinals
  - `MapStringTermsAggregator` — string fields using hash map
  - `NumericTermsAggregator` — numeric/date/boolean fields
- **`shard_size`**: Each shard collects this many top terms
  - Default: `size * 1.5 + 10` (from `BucketUtils.suggestShardSideQueueSize`)
  - User-configurable via `shard_size` parameter
- **Configuration**: `TermsAggregator.BucketCountThresholds` record holds:
  - `requiredSize` (the `size` parameter)
  - `shardSize`
  - `minDocCount`
  - `shardMinDocCount`

### Coordinator-Side (Reduce)

- **`AbstractInternalTerms.TermsAggregationReducer`** — merges results from all shards
  - Implements `AggregatorReducer` interface
  - Uses **merge-sort** when buckets are key-ordered (since v7.10)
  - Falls back to **HashMap-based merging** (`reduceLegacy`) for other orders
- **`TopBucketBuilder`** — selects the final top-N buckets
  - `PriorityQueueTopBucketBuilder` for small N (< 1024)
  - `BufferingTopBucketBuilder` for large N (>= 1024)
- **Two-phase reduce** via sealed `AggregationReduceContext`:
  - `ForPartial` — used on data nodes for partial reduction
  - `ForFinal` — used on coordinator for final reduction
- **Error tracking**:
  - `sumDocCountError` — accumulated error from all shards
  - `otherDocCount` — documents that didn't make it into top-N
  - `doc_count_error_upper_bound` — reported to the user

### The Error Source

Each shard only sends `shard_size` terms. Terms that don't make it into `shard_size` on a shard are **invisible** to the coordinator. This means:

1. A term that's popular globally but only moderately popular on each shard may be missed entirely
2. The `doc_count` for returned terms may be underestimated (missing contributions from shards where it fell below `shard_size`)
3. The coordinator reports `doc_count_error_upper_bound` as an upper bound on the error, but the actual error is unknown

**Example**: With 5 shards, `size=10`, `shard_size=25`:
- Term "foo" has 100 docs on shard 1 but only 5 docs on each of shards 2-5
- If "foo" doesn't make it into the top-25 on shards 2-5, the coordinator sees only 100 docs instead of the true 120
- Worse: "foo" might not appear in the final top-10 at all if other terms had higher counts on shard 1

---

## 3. Architecture Patterns

| Pattern | Where Used | Description |
|---------|-----------|-------------|
| Builder → Factory → Aggregator | `TermsAggregationBuilder` → `TermsAggregatorFactory` → concrete aggregator | Standard aggregation lifecycle |
| `AggregatorReducer` interface | `AbstractInternalTerms.TermsAggregationReducer` | Standard reduce pattern with `accept()` + `get()` |
| `ValuesSourceRegistry` | `TermsAggregatorFactory.registerAggregators` | Type-dispatching: keyword → bytes supplier, numeric → numeric supplier |
| `ExecutionMode` enum | `TermsAggregatorFactory.ExecutionMode` | MAP vs GLOBAL_ORDINALS selection |
| Sealed class hierarchy | `AggregationReduceContext.ForPartial` / `ForFinal` | Partial vs final reduce context |
| Stream serialization | All `InternalTerms`, `Bucket`, etc. | `Writeable` interface for node-to-node transfer |
| `ParseField` + `ObjectParser` | `TermsAggregationBuilder.PARSER` | REST API request parsing |

---

## 4. Key Files and Their Roles

### Core Implementation

| File | Role |
|------|------|
| `TermsAggregationBuilder.java` (498 lines) | REST API parsing, configuration, builder |
| `TermsAggregatorFactory.java` (629 lines) | Creates the right aggregator type, adjusts shard_size |
| `TermsAggregator.java` (274 lines) | Base class with `BucketCountThresholds` |
| `AbstractInternalTerms.java` (390 lines) | **Core reduce logic** — `TermsAggregationReducer` |
| `InternalTerms.java` | Concrete internal terms with serialization |
| `GlobalOrdinalsStringTermsAggregator.java` | String aggregation using global ordinals |
| `MapStringTermsAggregator.java` | String aggregation using hash map |
| `NumericTermsAggregator.java` | Numeric/date/boolean aggregation |

### Supporting Utilities

| File | Role |
|------|------|
| `BucketUtils.java` (36 lines) | `suggestShardSideQueueSize`: default `shard_size = size * 1.5 + 10` |
| `TopBucketBuilder.java` (218 lines) | Priority queue / buffering builder for top-N selection |
| `DelayedBucket.java` | Lazy bucket merging to avoid unnecessary work |
| `IteratorAndCurrent.java` | Merge-sort helper for sorted bucket lists |
| `AggregationReduceContext.java` | Partial vs final reduce context with cancellation |
| `AggregatorReducer.java` | Standard reduce interface (`accept` + `get`) |

### Test Files

| File | Base Class |
|------|-----------|
| `TermsAggregatorTests.java` | `AggregatorTestCase` |
| `KeywordTermsAggregatorTests.java` | `AggregatorTestCase` |
| `NumericTermsAggregatorTests.java` | `AggregatorTestCase` |
| `BinaryTermsAggregatorTests.java` | `AggregatorTestCase` |
| `TermsAggregatorFactoryTests.java` | `ESTestCase` |
| `InternalTermsTestCase.java` | Base for internal terms tests |
| `AbstractTermsTestCase.java` (framework) | Integration test base for terms accuracy |

---

## 5. Conventions to Follow

- **Class naming**: `[Type]TermsAggregator`, `Internal[Type]Terms`, `[Type]TermsAggregationBuilder`
- **Test naming**: `*Tests.java` suffix, extend `AggregatorTestCase` or `ESTestCase`
- **Serialization**: All stream-serializable objects implement `Writeable` with `StreamInput`/`StreamOutput`
- **Error handling**: `ElasticsearchStatusException` with `RestStatus` codes
- **License header**: Triple-license (Elastic 2.0, AGPL v3, SSPL v1)
- **Deprecation**: `LoggingDeprecationHandler` for deprecated parameter handling
- **Parallel collection**: Must check `supportsParallelCollection` for concurrent shard-level execution

---

## 6. Reusable Code and Utilities

| Utility | Location | Reuse Purpose |
|---------|----------|---------------|
| `BucketUtils.suggestShardSideQueueSize` | `bucket/BucketUtils.java` | Default shard_size heuristic (can be adapted for Phase 1) |
| `TopBucketBuilder` | `aggregations/TopBucketBuilder.java` | Priority queue / buffering for top-N selection (reusable in final phase) |
| `DelayedBucket` | `aggregations/DelayedBucket.java` | Lazy bucket merging (reusable in multi-phase reduce) |
| `IteratorAndCurrent` | `aggregations/bucket/IteratorAndCurrent.java` | Merge-sort helper |
| `AggregatorReducer` | `aggregations/AggregatorReducer.java` | Standard reduce interface to implement |
| `AggregationReduceContext` | `aggregations/AggregationReduceContext.java` | Partial vs final reduce context |
| `BucketOrder` | `aggregations/BucketOrder.java` | Bucket ordering (count, key, sub-aggregation) |
| `IncludeExclude` | `bucket/terms/IncludeExclude.java` | Term filtering (include/exclude patterns) |

---

## 7. TPUT Algorithm Research

### Background

The current terms aggregation is a **single-round heuristic**: each shard sends `shard_size` terms, coordinator merges. This can produce incorrect results when term distributions are skewed across shards.

### TPUT (Three-Phase Uniform Threshold)

From [Pei Cao et al., Stanford (PODC '04)](https://dl.acm.org/doi/10.1145/1011767.1011798):

```
Phase 1: Initial Top-K Collection
┌──────────┐    top-k terms    ┌─────────────┐
│  Shard 1  │ ───────────────► │             │
│  Shard 2  │ ───────────────► │ Coordinator │ → computes τ₁ (min aggregate
│  Shard N  │ ───────────────► │             │   score among provisional top-k)
└──────────┘                   └─────────────┘

Phase 2: Threshold-Based Expansion
┌─────────────┐   T = τ₁/m    ┌──────────┐
│             │ ◄───────────── │  Shard 1  │
│ Coordinator │   threshold    │  Shard 2  │ → each shard sends all terms
│             │ ───────────────►│  Shard N  │   with local count ≥ T
└─────────────┘                └──────────┘

Phase 3: Resolution
┌─────────────┐  specific      ┌──────────┐
│ Coordinator │  requests for  │  Shards   │ → coordinator resolves
│             │ ◄────────────► │           │   remaining uncertainty
└─────────────┘  missing data  └──────────┘
```

**Key properties:**
- **Exact**: Guarantees correct top-k with zero error. Any item not reported by any shard in Phase 2 has global score < τ₁, so it provably cannot be in the top-k.
- **Fixed round-trips**: Always completes in exactly 3 phases regardless of data distribution.
- **Efficient**: Reduces network traffic by 1-2 orders of magnitude vs. naive "send everything".
- **Threshold formula**: `T = τ₁ / m` where `m` = number of shards, `τ₁` = minimum aggregate score among the provisional top-k after Phase 1.

### TPAT (Three-Phase Adaptive Threshold)

A variant that uses **summary statistics** (e.g., histograms) of shard data distributions to compute tighter thresholds in Phase 2. This can reduce the number of terms transferred in Phase 2, but:

- **More complex**: Requires shards to compute and send distribution summaries
- **Higher Phase 1 cost**: Summary statistics add overhead
- **Diminishing returns**: In practice, TPUT's uniform threshold is already tight enough for most distributions

### Recommendation: TPUT

TPUT is the better choice for Elasticsearch because:
1. **Simpler implementation** — no distribution estimation needed
2. **Predictable performance** — exactly 3 round-trips, always
3. **Lower Phase 1 overhead** — no summary statistics
4. **Well-studied** — proven correct in academic literature
5. **Natural fit** — maps directly to ES's existing shard → coordinator communication model

---

## 8. Mapping TPUT to Elasticsearch Architecture

| TPUT Concept | Elasticsearch Equivalent |
|-------------|-------------------------|
| Distributed node | Data node (shard) |
| Central manager | Coordinating node |
| Item score | `doc_count` of a term bucket |
| Top-k | `size` parameter |
| Phase 1 response | Existing shard-level aggregation response (top `shard_size` terms) |
| Phase 2 threshold T | New: threshold broadcast to shards |
| Phase 2 response | New: additional terms with count ≥ T |
| Phase 3 | New: specific term count requests to shards |
| Final merge | Existing `TermsAggregationReducer.get()` logic |

### Key Architectural Decisions Needed

1. **How to implement multi-round communication** — Elasticsearch's current aggregation framework is single-round (search phase → fetch phase). TPUT requires 3 rounds.
2. **Where to add the new parameter** — new `execution_hint` value, new boolean flag, or separate aggregation type?
3. **Backward compatibility** — must not break existing terms aggregation behavior
4. **Performance trade-off** — 3 round-trips vs. 1, but potentially less data transferred and guaranteed correctness

---

## References

- [TPUT Paper — Pei Cao, Stanford (PDF)](https://crypto.stanford.edu/~cao/topk.pdf)
- [PODC '04 Proceedings — ACM](https://dl.acm.org/doi/10.1145/1011767.1011798)
- [Distributed top-k aggregation queries at large — Springer](https://link.springer.com/article/10.1007/s10619-009-7041-z)
- [Optimizing Distributed Top-k Queries — Springer](https://link.springer.com/chapter/10.1007/978-3-540-85481-4_26)
- [Elasticsearch Terms Aggregation Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-bucket-terms-aggregation.html)
