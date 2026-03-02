# Deterministic Terms Aggregation — Architecture Overview

## Selected Approach: Adaptive Multi-Round TPUT (Approach 3)

Add `"mode": "exact"` to the existing `terms` aggregation. Phase 1 runs unchanged (existing single-round). If the coordinator detects error in the results, it adaptively triggers TPUT Phases 2/3 as follow-up shard requests. Sub-aggregations are fully re-collected on shards for terms discovered in Phase 2/3.

### Why This Approach

| Factor | Rationale |
|--------|-----------|
| DFS phase is unsuitable | DFS only handles term frequency stats for scoring, not aggregation counts. [Elastic community confirms](https://discuss.elastic.co/t/use-dfs-query-then-fetch-with-aggregation/115210) it doesn't improve aggregation accuracy. |
| Fast-path optimization | Most queries with default `shard_size = size * 1.5 + 10` already produce exact results on low-to-medium cardinality fields → 0 extra round-trips. |
| Proven pattern | Composite aggregation validates multi-round shard communication for exact results in Elasticsearch. |
| Minimal risk | Phase 1 is literally the existing unchanged implementation. New code only activates when `mode == "exact"` AND `docCountError > 0`. |

---

## 1. Component Diagram

```
                              ┌─────────────────────────────────────────────┐
                              │         Coordinating Node                   │
                              │                                             │
                              │  ┌─────────────────────────────────────┐    │
   REST API                   │  │      TransportSearchAction          │    │
  ─────────────►              │  │                                     │    │
  { "aggs": {                 │  │  SearchPhase Chain:                 │    │
    "top_terms": {            │  │                                     │    │
      "terms": {              │  │  1. [DFS Phase] (optional)          │    │
        "field": "status",    │  │         │                           │    │
        "size": 10,           │  │         ▼                           │    │
  ──►   "mode": "exact"       │  │  2. Query Phase                    │    │
      }                       │  │     (each shard returns top         │    │
    }                         │  │      shard_size terms)              │    │
  }}                          │  │         │                           │    │
                              │  │         ▼                           │    │
                              │  │  3. TermsAggregationReducer         │    │
                              │  │     (merge shard results)           │    │
                              │  │         │                           │    │
                              │  │         ▼                           │    │
                              │  │  ┌──────────────────────┐          │    │
                              │  │  │ Error check:         │          │    │
                              │  │  │ docCountError == 0?  │──YES──►  │    │
                              │  │  │ otherDocCount == 0?  │   DONE   │    │
                              │  │  └─────────┬────────────┘ (1 RT)   │    │
                              │  │            │NO                     │    │
                              │  │            ▼                       │    │
                              │  │  4. ★ AggregationRefinementPhase   │    │
                              │  │     (NEW — TPUT Phases 2 & 3)      │    │
                              │  │         │                           │    │
                              │  │         ▼                           │    │
                              │  │  5. Fetch Phase                    │    │
                              │  │         │                           │    │
                              │  │         ▼                           │    │
                              │  │  6. Expand Phase                   │    │
                              │  └─────────────────────────────────────┘    │
                              └─────────────────────────────────────────────┘
                                         │              ▲
                     Phase 1 (existing)   │              │  Phase 2/3 (new)
                     ────────────────────►│              │◄─────────────────
                                         ▼              │
              ┌──────────────────────────────────────────────────────────┐
              │                      Data Nodes (Shards)                 │
              │                                                          │
              │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
              │  │  Shard 0    │  │  Shard 1    │  │  Shard N    │     │
              │  │             │  │             │  │             │     │
              │  │ Phase 1:    │  │ Phase 1:    │  │ Phase 1:    │     │
              │  │ Top shard_  │  │ Top shard_  │  │ Top shard_  │     │
              │  │ size terms  │  │ size terms  │  │ size terms  │     │
              │  │ + sub-aggs  │  │ + sub-aggs  │  │ + sub-aggs  │     │
              │  │             │  │             │  │             │     │
              │  │ Phase 2:    │  │ Phase 2:    │  │ Phase 2:    │     │
              │  │ All terms   │  │ All terms   │  │ All terms   │     │
              │  │ with count  │  │ with count  │  │ with count  │     │
              │  │ >= T, minus │  │ >= T, minus │  │ >= T, minus │     │
              │  │ seen terms  │  │ seen terms  │  │ seen terms  │     │
              │  │ + sub-aggs  │  │ + sub-aggs  │  │ + sub-aggs  │     │
              │  └─────────────┘  └─────────────┘  └─────────────┘     │
              └──────────────────────────────────────────────────────────┘
```

---

## 2. Data Flow

### Fast Path (common case — 1 round-trip, zero overhead)

```
Client ──► Coordinator ──► Shards (Phase 1: top shard_size terms + sub-aggs)
                │
                ◄─────── Shard results (each returns shard_size buckets)
                │
                │  Reduce: merge, compute docCountError
                │
                │  docCountError == 0 && otherDocCount == 0?  YES
                │  (This means shard_size was large enough to capture
                │   all globally-relevant terms on every shard)
                │
                ▼
           Return exact results (1 round-trip, zero overhead)
```

### Refinement Path (high cardinality + skewed distribution — 2-3 round-trips)

```
Client ──► Coordinator ──► Shards (Phase 1: top shard_size terms + sub-aggs)
                │
                ◄─────── Shard results
                │
                │  Reduce: merge, compute docCountError
                │  docCountError > 0?  YES → compute threshold T
                │
                │  τ₁ = min doc_count among provisional top-k after merge
                │  T  = τ₁ / num_shards
                │  seenTerms[shard_i] = { terms shard_i already sent }
                │
                ──► Shards (Phase 2: TermsRefinementRequest)
                │   { query, field, sub-aggs, threshold=T, seenTerms }
                │
                │   Each shard:
                │     1. Re-opens search context (same query/filter)
                │     2. Runs ThresholdTermsAggregator:
                │        - Collects ALL terms with doc_count >= T
                │        - Excludes terms in seenTerms (already sent)
                │        - Runs FULL sub-aggregation pipeline for new terms
                │        - Also returns corrected counts for seen terms
                │     3. Returns InternalTerms with sub-agg results
                │
                ◄─── TermsRefinementResponse { newBuckets + sub-aggs }
                │
                │  Merge Phase 1 + Phase 2 buckets:
                │  - New terms: add to global bucket map
                │  - Existing terms: update doc_count with corrections
                │  - Sub-aggs: merge using InternalAggregations.reduce()
                │
                │  Any terms still have incomplete shard coverage?
                │  (edge case: term above T on one shard but below T on others,
                │   yet still in global top-k when summed)
                │
                │  YES → Phase 3: point requests for specific (term, shard) pairs
                │  NO  → proceed
                │
                ▼
           Return exact results (docCountError = 0)
```

---

## 3. API Changes

### Request — New `mode` Parameter

```json
{
  "aggs": {
    "top_status": {
      "terms": {
        "field": "status",
        "size": 10,
        "mode": "exact"
      }
    }
  }
}
```

| Parameter | Values | Default | Description |
|-----------|--------|---------|-------------|
| `mode` | `"approximate"`, `"exact"` | `"approximate"` | `approximate` = current behavior. `exact` = TPUT-based deterministic mode with zero error guarantee. |

### Response — Guaranteed Zero Error

```json
{
  "aggregations": {
    "top_status": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 4500,
      "buckets": [
        {
          "key": "active",
          "doc_count": 12500,
          "avg_response_time": { "value": 234.5 }
        },
        {
          "key": "pending",
          "doc_count": 8200,
          "avg_response_time": { "value": 456.7 }
        }
      ]
    }
  }
}
```

When `mode: "exact"`:
- `doc_count_error_upper_bound` is always `0`
- `doc_count` values are exact global counts
- Sub-aggregation values are computed from complete data
- `sum_other_doc_count` is the exact count of documents in terms not in the top-k

### New Java Enum

```java
// In TermsAggregationBuilder.java or new file
public enum TermsAggregationMode implements Writeable {
    APPROXIMATE,  // default — current single-round behavior
    EXACT;        // TPUT-based multi-round deterministic mode

    // StreamInput/StreamOutput serialization
    // TransportVersion gating for BWC
}
```

### New Internal Transport Action

```
Action name: "indices:data/read/search[phase/terms_refinement]"

TermsRefinementShardRequest implements Writeable {
    ShardId shardId;
    SearchSourceBuilder searchSource;     // original query + filter
    String fieldName;
    ValuesSourceConfig valuesSourceConfig;
    AggregatorFactories.Builder subAggregations;
    long threshold;                        // T = τ₁ / m
    Set<Object> seenTerms;                // terms already sent by this shard
}

TermsRefinementShardResponse implements Writeable {
    List<InternalTerms.Bucket> additionalBuckets;  // new terms + sub-aggs
    Map<Object, Long> correctedCounts;              // updated counts for seen terms
}
```

---

## 4. Sub-Aggregation Strategy: Full Re-Collection

**Decision**: Phase 2/3 shard requests re-execute the full aggregation pipeline (including sub-aggs) for newly discovered terms.

### How It Works

```
Phase 2 Shard Request:
┌──────────────────────────────────────────────────────────────────┐
│  TermsRefinementShardRequest                                      │
│                                                                    │
│  ● searchSource             (query, filter, field, sub-aggs)      │
│  ● threshold T              (minimum doc_count to collect)         │
│  ● seenTerms[]              (terms this shard already sent)        │
│  ● shardSearchRequest       (shard routing, index context)         │
└──────────────────────────────────────────────────────────────────┘

Phase 2 Shard Execution:
┌──────────────────────────────────────────────────────────────┐
│  On each shard:                                               │
│                                                               │
│  1. Re-open the search context (same query/filter)            │
│  2. Create a ThresholdTermsAggregator:                        │
│     - Collects ALL terms with doc_count >= T                  │
│     - Excludes terms in seenTerms[] (already in Phase 1)      │
│     - Runs full sub-aggregation pipeline for each new bucket  │
│     - No shard_size limit (bounded by threshold instead)      │
│  3. Return InternalTerms with sub-agg results                 │
└──────────────────────────────────────────────────────────────┘

Phase 2 Shard Response:
┌──────────────────────────────────────────────────────────────┐
│  TermsRefinementShardResponse                                 │
│                                                               │
│  ● List<InternalTerms.Bucket> newBuckets                      │
│    (each bucket has doc_count + full sub-aggregation results)  │
│  ● Map<Object, Long> correctedCounts                          │
│    (exact counts for terms that were in Phase 1 but may have  │
│     been undercounted due to shard_size truncation)            │
└──────────────────────────────────────────────────────────────┘
```

### Coordinator Merge Logic

```
1. Start with Phase 1 reduced buckets (existing merge)

2. For each shard's Phase 2 response:
   a. For each new bucket (term not in Phase 1):
      - Add to global bucket map
      - Include sub-aggregation data
   b. For each corrected count:
      - Update the existing bucket's doc_count contribution from this shard
      - Re-merge sub-aggregations if provided

3. Re-reduce all buckets:
   - Merge sub-aggs using InternalAggregations.reduce()
   - Re-sort by order (count desc, key, etc.)
   - Take top-k
   - Compute exact sum_other_doc_count

4. Set docCountError = 0 (guaranteed)
```

### Why Re-Collection Is The Right Choice

| Alternative | Problem |
|-------------|---------|
| Skip sub-aggs for Phase 2 terms | Users would see `null` sub-agg values for some top-k buckets. Confusing and limits usefulness. |
| Pre-collect all sub-aggs in Phase 1 | Would require unbounded shard_size, defeating the purpose of the optimization. |
| Post-collect sub-aggs in a Phase 4 | Adds a 4th round-trip. Re-collecting in Phase 2 is more efficient. |

---

## 5. Database / Index Changes

**None.** This is entirely a search-time optimization. No changes to:
- Index mappings
- Stored data format
- Cluster state
- Index settings

---

## 6. Integration Points & Blast Radius

### Files Modified (Existing Code)

| File | Risk | Changes |
|------|------|---------|
| `TermsAggregationBuilder.java` | **Low** | Add `mode` ParseField, `TermsAggregationMode` enum, serialization. Existing behavior unchanged when mode != exact. |
| `TermsAggregator.java` | **Low** | Add `TermsAggregationMode` to `BucketCountThresholds`. No logic changes for approximate mode. |
| `AbstractInternalTerms.java` | **Medium** | `TermsAggregationReducer.get()` gains a branch: if `mode == exact && docCountError > 0` → signal that refinement is needed. Existing code path completely untouched. |
| `TermsAggregatorFactory.java` | **Low** | Pass mode through to aggregator construction. `adjustBucketCountThresholds` unchanged. |
| `InternalTerms.java` / `StringTerms.java` / `LongTerms.java` / `DoubleTerms.java` | **Low** | Add `mode` field to serialization. Carry mode through to reduce phase. |
| `TransportSearchAction.java` or `SearchPhaseController.java` | **Medium** | Insert `AggregationRefinementPhase` into the phase chain. Conditional — only when exact-mode aggregations are present. |

### Files Added (New Code)

| File | Purpose |
|------|---------|
| `TermsAggregationMode.java` | Enum: `APPROXIMATE`, `EXACT` with `Writeable` serialization |
| `AggregationRefinementPhase.java` | New `SearchPhase` — orchestrates TPUT Phase 2/3 |
| `TermsRefinementCoordinator.java` | Computes threshold T, tracks seenTerms, merges deltas |
| `TermsRefinementShardRequest.java` | Transport request: threshold + seenTerms + search context |
| `TermsRefinementShardResponse.java` | Transport response: new buckets + corrected counts |
| `TransportTermsRefinementAction.java` | Shard-level action handler |
| `ThresholdTermsAggregator.java` | New aggregator variant: collects terms above threshold T |

### Blast Radius Summary

```
                         BLAST RADIUS MAP
                         ================

  ┌─────────────────────────────────────────────────────────┐
  │                    SAFE ZONE (zero risk)                │
  │                                                         │
  │  All existing terms aggregation queries                 │
  │  (mode defaults to "approximate" — completely unchanged)│
  │                                                         │
  │  All other aggregation types                            │
  │  (filter, histogram, date_histogram, etc.)              │
  │                                                         │
  │  All non-aggregation search functionality               │
  └─────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────┐
  │               CAUTION ZONE (medium risk)                │
  │                                                         │
  │  Search phase chain insertion point                     │
  │  (must be a clean no-op when no exact-mode aggs exist)  │
  │                                                         │
  │  Mixed-version clusters                                 │
  │  (older nodes don't understand "mode" — need BWC gate)  │
  │                                                         │
  │  Memory under high cardinality                          │
  │  (Phase 2 could return many terms — enforce max_buckets)│
  └─────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────┐
  │            ATTENTION ZONE (needs careful design)         │
  │                                                         │
  │  Sub-aggregation re-collection in Phase 2               │
  │  (most complex new code — full agg pipeline re-run)     │
  │                                                         │
  │  ThresholdTermsAggregator                               │
  │  (new aggregator variant — needs thorough testing)      │
  └─────────────────────────────────────────────────────────┘
```

---

## 7. Backward Compatibility

### TransportVersion Gating

```java
// In TermsAggregationBuilder serialization:
if (out.getTransportVersion().onOrAfter(TransportVersions.EXACT_TERMS_MODE)) {
    mode.writeTo(out);
} else {
    // Older node: silently degrade to approximate mode
}

// In coordinator:
if (anyShardRunsOlderVersion()) {
    // Cannot guarantee exact results — fall back to approximate
    // Log warning: "Exact terms mode requires all nodes on version X+"
    effectiveMode = APPROXIMATE;
}
```

### REST API Compatibility

- `mode` parameter is ignored by older node versions (unknown parameters are silently ignored in ES aggregation parsing)
- No breaking changes to existing response format
- `doc_count_error_upper_bound: 0` is already a valid value in the current response

---

## 8. Performance Characteristics

| Scenario | Round-Trips | Overhead vs Current |
|----------|------------|-------------------|
| Low cardinality (< shard_size unique terms) | 1 | Zero — fast path, results already exact |
| Medium cardinality, even distribution | 1 | Zero — `shard_size * 1.5 + 10` captures all top-k |
| High cardinality, even distribution | 2 | +1 RT for Phase 2. Phase 2 returns few terms (high T). |
| High cardinality, skewed distribution | 2-3 | +1-2 RTs. Phase 2 may return moderate terms. Phase 3 rarely needed. |
| Very high cardinality (millions of unique terms) | 2 | Phase 2 bounded by T. max_buckets enforced. |

### Memory Bounds

- **Phase 1**: Same as current (`shard_size` buckets per shard)
- **Phase 2**: Bounded by threshold T. If T is high, few terms qualify. If T is low (meaning top-k scores are low), more terms qualify but `max_buckets` enforces an upper bound.
- **Coordinator**: Must hold Phase 1 + Phase 2 buckets in memory during merge. Bounded by `max_buckets * num_shards` in the worst case.

### Network Bounds

- **Phase 1**: Same as current
- **Phase 2**: Sends `seenTerms` set (O(shard_size) per shard) + receives delta terms (typically much smaller than Phase 1)
- **Phase 3**: Only specific (term, shard) pairs — very small

---

## References

- [TPUT Paper — Pei Cao, Stanford (PDF)](https://crypto.stanford.edu/~cao/topk.pdf)
- [PODC '04 Proceedings — ACM](https://dl.acm.org/doi/10.1145/1011767.1011798)
- [Distributed top-k aggregation queries at large — Springer](https://link.springer.com/article/10.1007/s10619-009-7041-z)
- [Cache-Efficient Top-k Aggregation (VLDB 2024)](https://dl.acm.org/doi/10.14778/3636218.3636222)
- [Elasticsearch Terms Aggregation Docs](https://www.elastic.co/docs/reference/aggregations/search-aggregations-bucket-terms-aggregation)
- [Elastic Blog: DFS Query Then Fetch](https://www.elastic.co/blog/understanding-query-then-fetch-vs-dfs-query-then-fetch)
- [Terms Aggregation Pitfalls (Towards AI)](https://towardsai.net/p/l/the-hidden-pitfalls-of-elasticsearch-terms-aggregation-and-how-to-fix-them)
