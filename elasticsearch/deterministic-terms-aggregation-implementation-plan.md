# Feature Plan: Deterministic Terms Aggregation via TPUT

## Context Summary

**Repository**: elasticsearch (server module only)
**Feature**: Add `"mode": "exact"` to the existing `terms` aggregation, implementing the TPUT (Three-Phase Uniform Threshold) algorithm to guarantee zero-error top-k results.
**Algorithm**: TPUT — Phase 1 collects top shard_size terms (existing behavior), Phase 2 broadcasts threshold T=τ₁/m to shards for expansion, Phase 3 resolves remaining gaps.

**Key files analyzed**: `TermsAggregationBuilder.java` (498 lines), `AbstractInternalTerms.java` (390 lines), `TermsAggregatorFactory.java` (629 lines), `TermsAggregator.java` (274 lines), `BucketUtils.java` (36 lines), `TopBucketBuilder.java` (218 lines), `FetchSearchPhase.java` (287 lines), `TermsAggregatorTests.java` (2383 lines).

---

## Selected Approach

**Approach 3: Adaptive Multi-Round TPUT** — Phase 1 is the existing unchanged single-round aggregation. If `docCountError > 0`, the coordinator adaptively triggers TPUT Phases 2/3 as follow-up shard requests with full sub-aggregation re-collection. Fast path (most queries) completes in 1 round-trip with zero overhead.

---

## Codebase Patterns This Plan Follows

| Pattern | Found In | Applied To |
|---------|----------|------------|
| Builder → Factory → Aggregator lifecycle | `TermsAggregationBuilder` → `TermsAggregatorFactory` → concrete aggregator | `mode` field flows through the same chain |
| `ParseField` + `ObjectParser` for REST params | `TermsAggregationBuilder.PARSER` static block (line 60-101) | New `MODE_FIELD` ParseField |
| `Writeable` serialization with `StreamInput`/`StreamOutput` | `TermsAggregationBuilder.innerWriteTo` (line 209) | `TermsAggregationMode` enum serialization |
| `TransportVersion` gating for BWC | Throughout ES codebase | New `EXACT_TERMS_MODE` version constant |
| `AggregatorReducer` interface (`accept` + `get`) | `AbstractInternalTerms.TermsAggregationReducer` (line 286) | Refinement trigger in `get()` |
| `SearchPhase` chain (nextPhase pattern) | `FetchSearchPhase.nextPhase()` (line 73) | New `AggregationRefinementPhase` |
| Test pattern: `AggregatorTestCase` | `TermsAggregatorTests.java` (line 187) | New tests extend same base |
| `ElasticsearchStatusException` for errors | `TermsAggregatorFactory.doCreateInternal()` (line 316) | Validation errors in exact mode |

---

## Reusable Code & Utilities

| Utility | From | Used For |
|---------|------|----------|
| `BucketUtils.suggestShardSideQueueSize` | `bucket/BucketUtils.java` | Phase 1 shard_size (unchanged) |
| `TopBucketBuilder` | `aggregations/TopBucketBuilder.java` | Final top-N selection after merge |
| `IteratorAndCurrent` | `bucket/IteratorAndCurrent.java` | Merge-sort during bucket merging |
| `AggregationReduceContext` | `aggregations/AggregationReduceContext.java` | Partial vs final reduce context |
| `BucketOrder` | `aggregations/BucketOrder.java` | Bucket ordering in final output |
| `IncludeExclude` | `bucket/terms/IncludeExclude.java` | Term filtering (passed through to Phase 2) |

---

## Architecture Overview

See [`deterministic-terms-aggregation-architecture.md`](./deterministic-terms-aggregation-architecture.md) for the full architecture with component diagrams, data flow, API changes, and blast radius analysis.

**Summary**: New `AggregationRefinementPhase` inserted between query reduce and fetch. Computes threshold `T = τ₁ / num_shards`, sends `TermsRefinementShardRequest` to each shard, shards run `ThresholdTermsAggregator` collecting all terms with `doc_count >= T` (excluding already-seen terms) with full sub-aggregation pipeline. Coordinator merges Phase 1 + Phase 2 results for exact top-k.

---

## Implementation Breakdown

### Task 1: Add `TermsAggregationMode` Enum

**Files**: `server/src/main/java/org/elasticsearch/search/aggregations/bucket/terms/TermsAggregationMode.java` (new file)
**Type**: new file

**What to do**:
Create a `Writeable` enum representing the aggregation mode. Follows the same pattern as `SubAggCollectionMode` and `TermsAggregatorFactory.ExecutionMode`.

**Test first (Red)**:
```java
// In server/src/test/java/org/elasticsearch/search/aggregations/bucket/terms/TermsAggregationModeTests.java
public class TermsAggregationModeTests extends ESTestCase {

    public void testWriteAndRead() throws IOException {
        for (TermsAggregationMode mode : TermsAggregationMode.values()) {
            BytesStreamOutput out = new BytesStreamOutput();
            mode.writeTo(out);
            StreamInput in = out.bytes().streamInput();
            assertThat(TermsAggregationMode.readFromStream(in), equalTo(mode));
        }
    }

    public void testParseApproximate() {
        assertThat(TermsAggregationMode.parse("approximate"), equalTo(TermsAggregationMode.APPROXIMATE));
    }

    public void testParseExact() {
        assertThat(TermsAggregationMode.parse("exact"), equalTo(TermsAggregationMode.EXACT));
    }

    public void testParseInvalid() {
        expectThrows(IllegalArgumentException.class, () -> TermsAggregationMode.parse("invalid"));
    }

    public void testDefaultIsApproximate() {
        assertThat(TermsAggregationMode.DEFAULT, equalTo(TermsAggregationMode.APPROXIMATE));
    }
}
```

**Implement (Green)**:
```java
// server/src/main/java/org/elasticsearch/search/aggregations/bucket/terms/TermsAggregationMode.java
package org.elasticsearch.search.aggregations.bucket.terms;

import org.elasticsearch.common.io.stream.StreamInput;
import org.elasticsearch.common.io.stream.StreamOutput;
import org.elasticsearch.common.io.stream.Writeable;

import java.io.IOException;
import java.util.Locale;

/**
 * Determines the accuracy mode for terms aggregation.
 * <p>
 * {@link #APPROXIMATE} is the default single-round behavior with potential error.
 * {@link #EXACT} uses the TPUT algorithm for guaranteed zero-error top-k results.
 */
public enum TermsAggregationMode implements Writeable {
    APPROXIMATE,
    EXACT;

    public static final TermsAggregationMode DEFAULT = APPROXIMATE;

    public static TermsAggregationMode readFromStream(StreamInput in) throws IOException {
        return in.readEnum(TermsAggregationMode.class);
    }

    @Override
    public void writeTo(StreamOutput out) throws IOException {
        out.writeEnum(this);
    }

    public static TermsAggregationMode parse(String value) {
        return switch (value.toLowerCase(Locale.ROOT)) {
            case "approximate" -> APPROXIMATE;
            case "exact" -> EXACT;
            default -> throw new IllegalArgumentException(
                "Unknown terms aggregation mode: [" + value + "]. Valid values are [approximate, exact]"
            );
        };
    }

    @Override
    public String toString() {
        return name().toLowerCase(Locale.ROOT);
    }
}
```

**Verify**:
```bash
./gradlew :server:test --tests "org.elasticsearch.search.aggregations.bucket.terms.TermsAggregationModeTests"
```

**Commit**: `feat(aggs): add TermsAggregationMode enum for exact/approximate terms`

---

### Task 2: Add `mode` Field to `TermsAggregationBuilder`

**Files**: `server/src/main/java/org/elasticsearch/search/aggregations/bucket/terms/TermsAggregationBuilder.java` (modify existing)
**Type**: modify existing

**What to do**:
Add the `mode` field to the builder with REST API parsing, serialization, equals/hashCode, and XContent output. The field must be TransportVersion-gated for backward compatibility.

**Test first (Red)**:
```java
// Add to existing TermsAggregatorFactoryTests.java or create TermsAggregationBuilderModeTests.java
public class TermsAggregationBuilderModeTests extends ESTestCase {

    public void testDefaultModeIsApproximate() {
        TermsAggregationBuilder builder = new TermsAggregationBuilder("test");
        assertThat(builder.mode(), equalTo(TermsAggregationMode.APPROXIMATE));
    }

    public void testSetModeExact() {
        TermsAggregationBuilder builder = new TermsAggregationBuilder("test").mode(TermsAggregationMode.EXACT);
        assertThat(builder.mode(), equalTo(TermsAggregationMode.EXACT));
    }

    public void testModeSerializationRoundTrip() throws IOException {
        TermsAggregationBuilder original = new TermsAggregationBuilder("test")
            .field("status")
            .mode(TermsAggregationMode.EXACT);
        BytesStreamOutput out = new BytesStreamOutput();
        out.setTransportVersion(TransportVersion.current());
        original.writeTo(out);
        StreamInput in = out.bytes().streamInput();
        in.setTransportVersion(TransportVersion.current());
        TermsAggregationBuilder deserialized = new TermsAggregationBuilder(in);
        assertThat(deserialized.mode(), equalTo(TermsAggregationMode.EXACT));
    }

    public void testModeInEquals() {
        TermsAggregationBuilder approx = new TermsAggregationBuilder("test").field("f");
        TermsAggregationBuilder exact = new TermsAggregationBuilder("test").field("f")
            .mode(TermsAggregationMode.EXACT);
        assertNotEquals(approx, exact);
    }

    public void testModeInXContent() throws IOException {
        TermsAggregationBuilder builder = new TermsAggregationBuilder("test")
            .field("status")
            .mode(TermsAggregationMode.EXACT);
        XContentBuilder xContent = XContentFactory.jsonBuilder();
        builder.toXContent(xContent, ToXContent.EMPTY_PARAMS);
        String json = Strings.toString(xContent);
        assertThat(json, containsString("\"mode\":\"exact\""));
    }

    public void testModeFromParser() throws IOException {
        String json = """
            { "field": "status", "mode": "exact" }
            """;
        XContentParser parser = createParser(XContentType.JSON.xContent(), json);
        TermsAggregationBuilder parsed = TermsAggregationBuilder.PARSER.parse(parser, "test_agg");
        assertThat(parsed.mode(), equalTo(TermsAggregationMode.EXACT));
    }
}
```

**Implement (Green)**:

Changes to `TermsAggregationBuilder.java`:

1. **Add ParseField and field declaration** (after line 58):
```java
public static final ParseField MODE_FIELD = new ParseField("mode");
```

2. **Add field** (after line 115, with other field declarations):
```java
private TermsAggregationMode mode = TermsAggregationMode.DEFAULT;
```

3. **Add to PARSER static block** (after line 101, before the closing `}`):
```java
PARSER.declareString(
    (b, v) -> b.mode(TermsAggregationMode.parse(v)),
    MODE_FIELD
);
```

4. **Add to clone constructor** (after line 133):
```java
this.mode = clone.mode;
```

5. **Add getter/setter** (after the `shardMinDocCount()` getter around line 298):
```java
/**
 * Sets the aggregation mode. {@link TermsAggregationMode#EXACT} uses the TPUT algorithm
 * to guarantee zero-error top-k results at the cost of additional round-trips.
 */
public TermsAggregationBuilder mode(TermsAggregationMode mode) {
    this.mode = Objects.requireNonNull(mode);
    return this;
}

/**
 * Returns the current aggregation mode.
 */
public TermsAggregationMode mode() {
    return mode;
}
```

6. **Add to deserialization constructor** (after line 200, `excludeDeletedDocs = in.readBoolean()`):
```java
if (in.getTransportVersion().onOrAfter(TransportVersions.EXACT_TERMS_MODE)) {
    mode = TermsAggregationMode.readFromStream(in);
} else {
    mode = TermsAggregationMode.DEFAULT;
}
```

7. **Add to `innerWriteTo`** (after line 216, `out.writeBoolean(excludeDeletedDocs)`):
```java
if (out.getTransportVersion().onOrAfter(TransportVersions.EXACT_TERMS_MODE)) {
    mode.writeTo(out);
}
```

8. **Add to `doXContentBody`** (after line 444):
```java
if (mode != TermsAggregationMode.DEFAULT) {
    builder.field(MODE_FIELD.getPreferredName(), mode.toString());
}
```

9. **Add to `hashCode`** (line 461):
```java
return Objects.hash(
    super.hashCode(),
    bucketCountThresholds,
    collectMode,
    executionHint,
    includeExclude,
    order,
    showTermDocCountError,
    excludeDeletedDocs,
    mode  // ADD THIS
);
```

10. **Add to `equals`** (after line 485):
```java
&& Objects.equals(mode, other.mode)
```

11. **Add to `shallowCopy`** — the clone constructor handles it via step 4.

**Verify**:
```bash
./gradlew :server:test --tests "org.elasticsearch.search.aggregations.bucket.terms.TermsAggregationBuilderModeTests"
./gradlew :server:test --tests "org.elasticsearch.search.aggregations.bucket.terms.TermsAggregatorFactoryTests"
```

**Commit**: `feat(aggs): add mode field to TermsAggregationBuilder for exact/approximate selection`

---

### Task 3: Add `TransportVersion` Constant

**Files**: `server/src/main/java/org/elasticsearch/TransportVersions.java` (modify existing)
**Type**: modify existing

**What to do**:
Add a new `TransportVersion` constant for the exact terms mode feature. This gates serialization of the `mode` field so older nodes in a mixed-version cluster silently degrade to approximate mode.

**Test first (Red)**:
```java
// TransportVersion constants are validated by existing tests — just ensure the constant compiles
// and is used correctly in the serialization code from Task 2.
```

**Implement (Green)**:

Add at the end of the constant list in `TransportVersions.java`:
```java
public static final TransportVersion EXACT_TERMS_MODE = def(8_xxx_00_0);
// Use the next available version ID following the pattern in the file
```

> **Note**: The exact version number must follow the sequential pattern in the current file. Check the last entry and increment.

**Verify**:
```bash
./gradlew :server:test --tests "*TransportVersion*"
```

**Commit**: `feat(transport): add EXACT_TERMS_MODE transport version constant`

---

### Task 4: Propagate `mode` Through Factory and Aggregator

**Files**:
- `server/src/main/java/org/elasticsearch/search/aggregations/bucket/terms/TermsAggregatorFactory.java` (modify)
- `server/src/main/java/org/elasticsearch/search/aggregations/bucket/terms/TermsAggregator.java` (modify)
- `server/src/main/java/org/elasticsearch/search/aggregations/bucket/terms/TermsAggregatorSupplier.java` (modify)
**Type**: modify existing

**What to do**:
Pass the `mode` from the builder through the factory to the aggregator, so the shard-level aggregator knows it needs to produce error-tracking metadata even when not explicitly requested by `show_term_doc_count_error`.

**Test first (Red)**:
```java
// In TermsAggregatorTests.java, add:
public void testExactModeEnablesDocCountError() throws IOException {
    // When mode=exact, doc count error tracking should be enabled
    // even if show_term_doc_count_error is false
    try (Directory directory = newDirectory()) {
        try (RandomIndexWriter indexWriter = new RandomIndexWriter(random(), directory)) {
            indexWriter.addDocument(singleton(new SortedSetDocValuesField("field", new BytesRef("a"))));
            indexWriter.addDocument(singleton(new SortedSetDocValuesField("field", new BytesRef("b"))));
        }
        try (DirectoryReader indexReader = DirectoryReader.open(directory)) {
            TermsAggregationBuilder builder = new TermsAggregationBuilder("test")
                .field("field")
                .mode(TermsAggregationMode.EXACT);
            // mode should propagate through factory
            assertThat(builder.mode(), equalTo(TermsAggregationMode.EXACT));
        }
    }
}
```

**Implement (Green)**:

1. **`TermsAggregatorFactory` constructor** — add `mode` parameter, store it:
```java
private final TermsAggregationMode mode;

// In constructor:
this.mode = mode;
```

2. **`TermsAggregationBuilder.innerBuild()`** — pass `mode` to factory:
```java
// Around line 416, in the TermsAggregatorFactory constructor call, add mode parameter
return new TermsAggregatorFactory(
    name, config, order, includeExclude, executionHint,
    collectMode, bucketCountThresholds, showTermDocCountError,
    excludeDeletedDocs, mode, context, parent, subFactoriesBuilder, metadata
);
```

3. **`TermsAggregator.BucketCountThresholds`** — No change needed here. The `mode` travels as a separate field, not inside thresholds.

4. **`TermsAggregatorFactory.doCreateInternal()`** — when `mode == EXACT`, force `showTermDocCountError = true` internally so the reducer has error data to work with:
```java
// After line 311 (BucketCountThresholds adjusted = ...)
boolean effectiveShowError = showTermDocCountError || (mode == TermsAggregationMode.EXACT);

// Pass effectiveShowError instead of showTermDocCountError to aggregatorSupplier.build()
```

**Verify**:
```bash
./gradlew :server:test --tests "org.elasticsearch.search.aggregations.bucket.terms.TermsAggregator*"
./gradlew :server:test --tests "org.elasticsearch.search.aggregations.bucket.terms.TermsAggregatorFactory*"
```

**Commit**: `feat(aggs): propagate TermsAggregationMode through factory to aggregator`

---

### Task 5: Carry `mode` in Internal Terms (Serialization)

**Files**:
- `server/src/main/java/org/elasticsearch/search/aggregations/bucket/terms/InternalTerms.java` (modify)
- `server/src/main/java/org/elasticsearch/search/aggregations/bucket/terms/StringTerms.java` (modify)
- `server/src/main/java/org/elasticsearch/search/aggregations/bucket/terms/LongTerms.java` (modify)
- `server/src/main/java/org/elasticsearch/search/aggregations/bucket/terms/DoubleTerms.java` (modify)
- `server/src/main/java/org/elasticsearch/search/aggregations/bucket/terms/AbstractInternalTerms.java` (modify)
**Type**: modify existing

**What to do**:
The `InternalTerms` objects are serialized from data nodes to the coordinator. They need to carry the `mode` so the coordinator's reduce logic knows whether to trigger refinement. Add the `mode` field to `AbstractInternalTerms` and propagate through all concrete subclasses' serialization.

**Test first (Red)**:
```java
// In InternalTermsTestCase (or a new ExactModeInternalTermsTests.java):
public void testExactModeSerializationRoundTrip() throws IOException {
    // Create an InternalTerms (e.g., StringTerms) with mode=EXACT
    // Serialize and deserialize, verify mode is preserved
    StringTerms terms = createTestInstance();
    // Set mode to EXACT
    terms.setMode(TermsAggregationMode.EXACT);
    BytesStreamOutput out = new BytesStreamOutput();
    out.setTransportVersion(TransportVersion.current());
    terms.writeTo(out);
    StreamInput in = out.bytes().streamInput();
    in.setTransportVersion(TransportVersion.current());
    StringTerms deserialized = new StringTerms(in);
    assertThat(deserialized.getMode(), equalTo(TermsAggregationMode.EXACT));
}

public void testExactModePreservedThroughReduce() throws IOException {
    // Create multiple InternalTerms with mode=EXACT
    // Reduce them, verify the result also has mode=EXACT
}
```

**Implement (Green)**:

1. **`AbstractInternalTerms`** — add field and accessor:
```java
protected TermsAggregationMode mode = TermsAggregationMode.DEFAULT;

public TermsAggregationMode getMode() {
    return mode;
}

public void setMode(TermsAggregationMode mode) {
    this.mode = mode;
}
```

2. **`InternalTerms`** — add to serialization:
```java
// In constructor from StreamInput:
if (in.getTransportVersion().onOrAfter(TransportVersions.EXACT_TERMS_MODE)) {
    mode = TermsAggregationMode.readFromStream(in);
}

// In writeTo:
if (out.getTransportVersion().onOrAfter(TransportVersions.EXACT_TERMS_MODE)) {
    mode.writeTo(out);
}
```

3. **`StringTerms`, `LongTerms`, `DoubleTerms`** — ensure they call `super` serialization which handles the mode field, or add explicit serialization if the pattern requires it. Each concrete class's `create()` factory method must propagate `mode`:
```java
// In each concrete class's create() method:
@Override
protected StringTerms create(String name, List<Bucket> buckets, BucketOrder reduceOrder, long docCountError, long otherDocCount) {
    StringTerms result = new StringTerms(name, reduceOrder, order, bucketCountThresholds, requiredSize, minDocCount,
        metadata, format, shardSize, showTermDocCountError, otherDocCount, buckets, docCountError);
    result.setMode(this.mode);
    return result;
}
```

**Verify**:
```bash
./gradlew :server:test --tests "org.elasticsearch.search.aggregations.bucket.terms.Internal*"
./gradlew :server:test --tests "org.elasticsearch.search.aggregations.bucket.terms.StringTermsTests"
./gradlew :server:test --tests "org.elasticsearch.search.aggregations.bucket.terms.LongTermsTests"
```

**Commit**: `feat(aggs): carry TermsAggregationMode through InternalTerms serialization`

---

### Task 6: Add Refinement Trigger in `AbstractInternalTerms` Reduce

**Files**: `server/src/main/java/org/elasticsearch/search/aggregations/bucket/terms/AbstractInternalTerms.java` (modify)
**Type**: modify existing

**What to do**:
In `TermsAggregationReducer.get()` (line 286), after the existing reduce logic computes `docCountError`, add a branch that signals refinement is needed when `mode == EXACT && docCountError > 0`. The signal is carried as a new field on the returned `InternalTerms` object that the `AggregationRefinementPhase` will inspect.

**Test first (Red)**:
```java
// In a new ExactTermsReduceTests.java extending ESTestCase:
public void testExactModeSignalsRefinementWhenErrorPresent() {
    // Create multiple InternalTerms from different "shards" with mode=EXACT
    // where the reduce would produce docCountError > 0
    // Verify the reduced result has needsRefinement() == true
}

public void testExactModeFastPathWhenNoError() {
    // Create InternalTerms where each shard has all terms (no error)
    // Verify needsRefinement() == false even with mode=EXACT
}

public void testApproximateModeNeverSignalsRefinement() {
    // Create InternalTerms with mode=APPROXIMATE and docCountError > 0
    // Verify needsRefinement() == false (existing behavior)
}
```

**Implement (Green)**:

1. **Add `needsRefinement` flag to `AbstractInternalTerms`**:
```java
private boolean needsRefinement = false;

public boolean needsRefinement() {
    return needsRefinement;
}
```

2. **Modify `TermsAggregationReducer.get()`** — after line 340 (`return create(...)`):
```java
@Override
public InternalAggregation get() {
    // ... existing reduce logic (lines 287-339) ...

    long docCountError = -1;
    if (sumDocCountError != -1) {
        docCountError = size == 1 && reduceContext.hasBatchedResult() == false ? 0 : sumDocCountError;
    }

    AbstractInternalTerms<?, B> reduced = create(
        name, result,
        reduceContext.isFinalReduce() ? getOrder() : thisReduceOrder,
        docCountError, otherDocCount
    );

    // TPUT refinement trigger: if mode is EXACT and we detected error,
    // signal that AggregationRefinementPhase should run
    if (getMode() == TermsAggregationMode.EXACT
            && reduceContext.isFinalReduce()
            && (docCountError > 0 || (docCountError == -1))) {
        reduced.needsRefinement = true;
    }

    return reduced;
}
```

3. **Store provisional top-k data** needed by the refinement phase:
```java
// In AbstractInternalTerms, add fields to carry Phase 1 context:
private long provisionalMinDocCount = -1;  // τ₁ for threshold computation
private int numShards = -1;

public long getProvisionalMinDocCount() {
    return provisionalMinDocCount;
}

public int getNumShardsInReduce() {
    return numShards;
}
```

4. **In `TermsAggregationReducer.get()`**, compute τ₁:
```java
// After computing result list, before creating the reduced object:
if (getMode() == TermsAggregationMode.EXACT && !result.isEmpty()) {
    // τ₁ = minimum doc_count among the provisional top-k
    long minCount = Long.MAX_VALUE;
    for (B bucket : result) {
        minCount = Math.min(minCount, bucket.getDocCount());
    }
    reduced.provisionalMinDocCount = minCount;
    reduced.numShards = size;  // 'size' here is the count of shards accepted
}
```

**Verify**:
```bash
./gradlew :server:test --tests "org.elasticsearch.search.aggregations.bucket.terms.ExactTermsReduceTests"
./gradlew :server:test --tests "org.elasticsearch.search.aggregations.bucket.terms.TermsAggregator*"
```

**Commit**: `feat(aggs): add TPUT refinement trigger in AbstractInternalTerms reduce`

---

### Task 7: Create `TermsRefinementShardRequest` and `TermsRefinementShardResponse`

**Files**:
- `server/src/main/java/org/elasticsearch/search/aggregations/bucket/terms/TermsRefinementShardRequest.java` (new)
- `server/src/main/java/org/elasticsearch/search/aggregations/bucket/terms/TermsRefinementShardResponse.java` (new)
**Type**: new files

**What to do**:
Create the transport request/response objects for Phase 2 shard communication. These carry the threshold, seen terms, and search context to each shard, and return additional buckets with sub-aggregation data.

**Test first (Red)**:
```java
// TermsRefinementShardRequestTests.java
public class TermsRefinementShardRequestTests extends ESTestCase {

    public void testSerializationRoundTrip() throws IOException {
        ShardId shardId = new ShardId("test-index", "_na_", 0);
        TermsRefinementShardRequest request = new TermsRefinementShardRequest(
            shardId,
            new OriginalIndices(new String[] {"test-index"}, IndicesOptions.strictExpandOpen()),
            "status",             // field name
            1000L,                // threshold T
            Set.of(              // seen terms
                new BytesRef("active"),
                new BytesRef("pending")
            ),
            null,                 // search source (simplified for test)
            null                  // sub-aggregation factories
        );

        BytesStreamOutput out = new BytesStreamOutput();
        request.writeTo(out);
        StreamInput in = out.bytes().streamInput();
        TermsRefinementShardRequest deserialized = new TermsRefinementShardRequest(in);

        assertThat(deserialized.shardId(), equalTo(shardId));
        assertThat(deserialized.fieldName(), equalTo("status"));
        assertThat(deserialized.threshold(), equalTo(1000L));
        assertThat(deserialized.seenTerms(), hasSize(2));
    }
}

// TermsRefinementShardResponseTests.java
public class TermsRefinementShardResponseTests extends ESTestCase {

    public void testSerializationRoundTrip() throws IOException {
        // Create response with sample buckets and corrected counts
        // Serialize and deserialize, verify all data preserved
    }
}
```

**Implement (Green)**:

```java
// TermsRefinementShardRequest.java
package org.elasticsearch.search.aggregations.bucket.terms;

import org.elasticsearch.action.IndicesRequest;
import org.elasticsearch.action.OriginalIndices;
import org.elasticsearch.action.support.IndicesOptions;
import org.elasticsearch.common.io.stream.StreamInput;
import org.elasticsearch.common.io.stream.StreamOutput;
import org.elasticsearch.index.shard.ShardId;
import org.elasticsearch.search.builder.SearchSourceBuilder;
import org.elasticsearch.transport.TransportRequest;

import java.io.IOException;
import java.util.Set;

public class TermsRefinementShardRequest extends TransportRequest implements IndicesRequest {

    private final ShardId shardId;
    private final OriginalIndices originalIndices;
    private final String fieldName;
    private final long threshold;
    private final Set<Object> seenTerms;
    private final SearchSourceBuilder searchSource;

    public TermsRefinementShardRequest(
        ShardId shardId,
        OriginalIndices originalIndices,
        String fieldName,
        long threshold,
        Set<Object> seenTerms,
        SearchSourceBuilder searchSource
    ) {
        this.shardId = shardId;
        this.originalIndices = originalIndices;
        this.fieldName = fieldName;
        this.threshold = threshold;
        this.seenTerms = seenTerms;
        this.searchSource = searchSource;
    }

    public TermsRefinementShardRequest(StreamInput in) throws IOException {
        super(in);
        shardId = new ShardId(in);
        originalIndices = OriginalIndices.readOriginalIndices(in);
        fieldName = in.readString();
        threshold = in.readVLong();
        seenTerms = in.readCollectionAsSet(StreamInput::readGenericValue);
        searchSource = in.readOptionalWriteable(SearchSourceBuilder::new);
    }

    @Override
    public void writeTo(StreamOutput out) throws IOException {
        super.writeTo(out);
        shardId.writeTo(out);
        OriginalIndices.writeOriginalIndices(originalIndices, out);
        out.writeString(fieldName);
        out.writeVLong(threshold);
        out.writeCollection(seenTerms, StreamOutput::writeGenericValue);
        out.writeOptionalWriteable(searchSource);
    }

    public ShardId shardId() { return shardId; }
    public String fieldName() { return fieldName; }
    public long threshold() { return threshold; }
    public Set<Object> seenTerms() { return seenTerms; }
    public SearchSourceBuilder searchSource() { return searchSource; }

    @Override
    public String[] indices() { return originalIndices.indices(); }

    @Override
    public IndicesOptions indicesOptions() { return originalIndices.indicesOptions(); }
}
```

```java
// TermsRefinementShardResponse.java
package org.elasticsearch.search.aggregations.bucket.terms;

import org.elasticsearch.common.io.stream.StreamInput;
import org.elasticsearch.common.io.stream.StreamOutput;
import org.elasticsearch.search.aggregations.InternalAggregations;
import org.elasticsearch.transport.TransportResponse;

import java.io.IOException;
import java.util.List;
import java.util.Map;

public class TermsRefinementShardResponse extends TransportResponse {

    private final List<RefinementBucket> additionalBuckets;
    private final Map<Object, Long> correctedCounts;

    public record RefinementBucket(Object key, long docCount, InternalAggregations subAggregations) {}

    public TermsRefinementShardResponse(
        List<RefinementBucket> additionalBuckets,
        Map<Object, Long> correctedCounts
    ) {
        this.additionalBuckets = additionalBuckets;
        this.correctedCounts = correctedCounts;
    }

    public TermsRefinementShardResponse(StreamInput in) throws IOException {
        super(in);
        additionalBuckets = in.readCollectionAsList(i -> new RefinementBucket(
            i.readGenericValue(),
            i.readVLong(),
            InternalAggregations.readFrom(i)
        ));
        correctedCounts = in.readMap(StreamInput::readGenericValue, StreamInput::readVLong);
    }

    @Override
    public void writeTo(StreamOutput out) throws IOException {
        out.writeCollection(additionalBuckets, (o, b) -> {
            o.writeGenericValue(b.key());
            o.writeVLong(b.docCount());
            b.subAggregations().writeTo(o);
        });
        out.writeMap(correctedCounts, StreamOutput::writeGenericValue, StreamOutput::writeVLong);
    }

    public List<RefinementBucket> additionalBuckets() { return additionalBuckets; }
    public Map<Object, Long> correctedCounts() { return correctedCounts; }
}
```

**Verify**:
```bash
./gradlew :server:test --tests "org.elasticsearch.search.aggregations.bucket.terms.TermsRefinementShard*"
```

**Commit**: `feat(aggs): add TermsRefinementShardRequest/Response transport objects`

---

### Task 8: Create `ThresholdTermsAggregator`

**Files**: `server/src/main/java/org/elasticsearch/search/aggregations/bucket/terms/ThresholdTermsAggregator.java` (new file)
**Type**: new file

**What to do**:
Create a new aggregator variant for Phase 2 shard-side execution. Unlike the standard terms aggregator which keeps the top `shard_size` terms, this collects **all** terms with `doc_count >= threshold` while excluding terms in the `seenTerms` set. It runs the full sub-aggregation pipeline for each collected term.

This is the most complex new component. It extends the existing aggregator infrastructure but replaces the top-N collection logic with threshold-based collection.

**Test first (Red)**:
```java
// ThresholdTermsAggregatorTests.java extending AggregatorTestCase
public class ThresholdTermsAggregatorTests extends AggregatorTestCase {

    public void testCollectsTermsAboveThreshold() throws IOException {
        // Index: term "a" (100 docs), "b" (50 docs), "c" (10 docs), "d" (5 docs)
        // Threshold: 15
        // Expected: "a" and "b" are collected, "c" and "d" are not
        try (Directory directory = newDirectory()) {
            try (RandomIndexWriter indexWriter = new RandomIndexWriter(random(), directory)) {
                for (int i = 0; i < 100; i++) addDoc(indexWriter, "a");
                for (int i = 0; i < 50; i++) addDoc(indexWriter, "b");
                for (int i = 0; i < 10; i++) addDoc(indexWriter, "c");
                for (int i = 0; i < 5; i++) addDoc(indexWriter, "d");
            }
            try (DirectoryReader reader = DirectoryReader.open(directory)) {
                // Run ThresholdTermsAggregator with threshold=15, seenTerms={}
                // Assert results contain "a" (100) and "b" (50) only
            }
        }
    }

    public void testExcludesSeenTerms() throws IOException {
        // Same data as above, but seenTerms = {"a"}
        // Threshold: 15
        // Expected: only "b" is collected ("a" excluded, "c"/"d" below threshold)
    }

    public void testSubAggregationsExecuted() throws IOException {
        // Index terms with numeric sub-field
        // Run ThresholdTermsAggregator with avg sub-aggregation
        // Assert sub-agg results are present and correct
    }

    public void testEmptyResultWhenAllTermsBelowThreshold() throws IOException {
        // All terms below threshold → empty result
    }

    private void addDoc(RandomIndexWriter writer, String term) throws IOException {
        Document doc = new Document();
        doc.add(new SortedSetDocValuesField("field", new BytesRef(term)));
        writer.addDocument(doc);
    }
}
```

**Implement (Green)**:

```java
// ThresholdTermsAggregator.java
package org.elasticsearch.search.aggregations.bucket.terms;

import org.apache.lucene.index.SortedSetDocValues;
import org.apache.lucene.index.TermsEnum;
import org.apache.lucene.util.BytesRef;
import org.elasticsearch.search.aggregations.AggregatorFactories;
import org.elasticsearch.search.aggregations.CardinalityUpperBound;
import org.elasticsearch.search.aggregations.LeafBucketCollector;
import org.elasticsearch.search.aggregations.LeafBucketCollectorBase;
import org.elasticsearch.search.aggregations.bucket.DeferableBucketAggregator;
import org.elasticsearch.search.aggregations.support.AggregationContext;
import org.elasticsearch.search.aggregations.support.ValuesSourceConfig;

import java.io.IOException;
import java.util.Map;
import java.util.Set;

/**
 * A terms aggregator that collects all terms with doc_count >= threshold,
 * excluding terms in the seenTerms set. Used in TPUT Phase 2 for exact
 * terms aggregation.
 *
 * Unlike standard terms aggregators that maintain a top-N priority queue,
 * this aggregator uses a threshold-based cutoff: any term meeting the
 * threshold is collected with full sub-aggregations.
 */
public class ThresholdTermsAggregator extends DeferableBucketAggregator {

    private final long threshold;
    private final Set<Object> seenTerms;
    private final ValuesSourceConfig valuesSourceConfig;
    private final String fieldName;

    public ThresholdTermsAggregator(
        String name,
        AggregatorFactories factories,
        AggregationContext context,
        Aggregator parent,
        ValuesSourceConfig valuesSourceConfig,
        String fieldName,
        long threshold,
        Set<Object> seenTerms,
        Map<String, Object> metadata
    ) throws IOException {
        super(name, factories, context, parent, metadata);
        this.threshold = threshold;
        this.seenTerms = seenTerms;
        this.valuesSourceConfig = valuesSourceConfig;
        this.fieldName = fieldName;
    }

    // Implementation details:
    // 1. Iterate over all terms in the field's doc values
    // 2. For each term, count documents matching the query
    // 3. If count >= threshold AND term not in seenTerms, collect into a bucket
    // 4. Sub-aggregations are collected for each qualifying bucket
    //
    // The exact leaf collector implementation depends on field type
    // (global ordinals for keyword, numeric for numbers).
    // Start with keyword/string support using SortedSetDocValues.

    @Override
    protected LeafBucketCollector getLeafCollector(/* ... */) throws IOException {
        // Collect documents, grouping by term value
        // Post-collection: filter buckets by threshold and seenTerms
        // This follows the same pattern as GlobalOrdinalsStringTermsAggregator
        // but without the priority queue — all qualifying terms are kept
        return LeafBucketCollector.NO_OP_COLLECTOR; // placeholder
    }

    // buildAggregations() → iterate collected buckets, filter by threshold,
    // build InternalTerms with sub-agg results
}
```

> **Implementation note**: The full implementation of `ThresholdTermsAggregator` is the most complex task. It should closely follow the structure of `GlobalOrdinalsStringTermsAggregator` for string fields and `NumericTermsAggregator` for numeric fields, replacing the priority-queue-based top-N selection with a threshold-based filter. The key difference is:
> - Standard: collect all, keep top shard_size via priority queue
> - Threshold: collect all with count >= T, skip seenTerms, keep all qualifying

**Verify**:
```bash
./gradlew :server:test --tests "org.elasticsearch.search.aggregations.bucket.terms.ThresholdTermsAggregatorTests"
```

**Commit**: `feat(aggs): add ThresholdTermsAggregator for TPUT Phase 2 shard-side collection`

---

### Task 9: Create `TermsRefinementCoordinator`

**Files**: `server/src/main/java/org/elasticsearch/search/aggregations/bucket/terms/TermsRefinementCoordinator.java` (new file)
**Type**: new file

**What to do**:
Create the coordinator-side logic that orchestrates TPUT Phases 2 and 3. This class:
1. Computes threshold `T = τ₁ / numShards`
2. Determines which terms each shard has already sent (seenTerms per shard)
3. Builds `TermsRefinementShardRequest` for each shard
4. Merges Phase 2 responses with Phase 1 results
5. Determines if Phase 3 is needed (rare edge case)
6. Produces the final exact `InternalTerms`

**Test first (Red)**:
```java
// TermsRefinementCoordinatorTests.java
public class TermsRefinementCoordinatorTests extends ESTestCase {

    public void testThresholdComputation() {
        // provisionalMinDocCount = 100, numShards = 5
        // Expected T = 100 / 5 = 20
        TermsRefinementCoordinator coordinator = new TermsRefinementCoordinator(
            /* provisionalTopK */ ...,
            /* numShards */ 5,
            /* requiredSize */ 10
        );
        assertThat(coordinator.computeThreshold(), equalTo(20L));
    }

    public void testThresholdWithSingleShard() {
        // Single shard: T = τ₁ / 1 = τ₁ (no expansion needed, shouldn't be called)
        // This is a fast-path case — verify it handles gracefully
    }

    public void testMergePhase2Results() {
        // Phase 1 result: {a: 100, b: 80, c: 60}
        // Phase 2 shard responses: shard1 adds {d: 50}, shard2 corrects {c: +15}
        // Expected merged: {a: 100, b: 80, c: 75, d: 50}
    }

    public void testPhase3DetectionWhenNeeded() {
        // After Phase 2 merge, a term is in top-k but missing counts from some shards
        // Verify Phase 3 requests are generated
    }

    public void testPhase3NotNeededWhenComplete() {
        // After Phase 2 merge, all top-k terms have complete shard coverage
        // Verify no Phase 3 requests
    }

    public void testFinalResultHasZeroError() {
        // After full refinement, docCountError must be 0
    }
}
```

**Implement (Green)**:

```java
// TermsRefinementCoordinator.java
package org.elasticsearch.search.aggregations.bucket.terms;

import org.elasticsearch.search.aggregations.InternalAggregation;
import org.elasticsearch.search.aggregations.InternalAggregations;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.HashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;

/**
 * Orchestrates TPUT Phases 2 and 3 for exact terms aggregation.
 *
 * After Phase 1 (standard terms aggregation) produces results with
 * docCountError > 0, this coordinator computes the TPUT threshold,
 * builds shard requests, and merges responses to produce exact results.
 */
public class TermsRefinementCoordinator {

    private final long provisionalMinDocCount;  // τ₁
    private final int numShards;
    private final int requiredSize;
    private final Map<Object, Long> globalBucketCounts;  // merged Phase 1 counts
    private final Map<Integer, Set<Object>> perShardSeenTerms;  // shard → terms it sent

    public TermsRefinementCoordinator(
        long provisionalMinDocCount,
        int numShards,
        int requiredSize,
        Map<Object, Long> globalBucketCounts,
        Map<Integer, Set<Object>> perShardSeenTerms
    ) {
        this.provisionalMinDocCount = provisionalMinDocCount;
        this.numShards = numShards;
        this.requiredSize = requiredSize;
        this.globalBucketCounts = globalBucketCounts;
        this.perShardSeenTerms = perShardSeenTerms;
    }

    /**
     * Compute the TPUT threshold: T = τ₁ / m
     * Any term with global count >= τ₁ could be in the top-k.
     * A term with global count >= τ₁ must have local count >= τ₁/m
     * on at least one shard. So T = floor(τ₁ / m).
     */
    public long computeThreshold() {
        if (numShards <= 0) return 0;
        return provisionalMinDocCount / numShards;
    }

    /**
     * Build Phase 2 shard requests.
     */
    public List<TermsRefinementShardRequest> buildPhase2Requests(/* shard routing info */) {
        long threshold = computeThreshold();
        List<TermsRefinementShardRequest> requests = new ArrayList<>();
        for (int shardIdx = 0; shardIdx < numShards; shardIdx++) {
            Set<Object> seenTerms = perShardSeenTerms.getOrDefault(shardIdx, Set.of());
            // Build request with threshold and seenTerms
            // requests.add(new TermsRefinementShardRequest(...));
        }
        return requests;
    }

    /**
     * Merge Phase 2 responses with Phase 1 results.
     * Returns the merged bucket map and identifies any terms
     * that still need Phase 3 resolution.
     */
    public MergeResult mergePhase2Responses(
        Map<Integer, TermsRefinementShardResponse> shardResponses
    ) {
        Map<Object, Long> mergedCounts = new HashMap<>(globalBucketCounts);
        Map<Object, InternalAggregations> mergedSubAggs = new HashMap<>();

        for (var entry : shardResponses.entrySet()) {
            TermsRefinementShardResponse response = entry.getValue();

            // Add new buckets
            for (var bucket : response.additionalBuckets()) {
                mergedCounts.merge(bucket.key(), bucket.docCount(), Long::sum);
                // Merge sub-aggregations
            }

            // Apply corrected counts
            for (var correction : response.correctedCounts().entrySet()) {
                mergedCounts.put(correction.getKey(), correction.getValue());
            }
        }

        // Determine if Phase 3 is needed:
        // Sort by count desc, take top requiredSize
        // For each top-k term, check if all shards have reported
        // If any shard hasn't reported a top-k term (and didn't include
        // it in seenTerms or Phase 2), we need Phase 3
        Set<Object> needsPhase3 = identifyIncompleteTerms(mergedCounts);

        return new MergeResult(mergedCounts, mergedSubAggs, needsPhase3);
    }

    private Set<Object> identifyIncompleteTerms(Map<Object, Long> mergedCounts) {
        // Implementation: for each term in provisional top-k of mergedCounts,
        // check if every shard has reported it (in Phase 1 seenTerms or Phase 2 response)
        // If not, it needs Phase 3 point queries
        return Set.of(); // placeholder
    }

    public record MergeResult(
        Map<Object, Long> mergedCounts,
        Map<Object, InternalAggregations> mergedSubAggs,
        Set<Object> needsPhase3
    ) {}
}
```

**Verify**:
```bash
./gradlew :server:test --tests "org.elasticsearch.search.aggregations.bucket.terms.TermsRefinementCoordinatorTests"
```

**Commit**: `feat(aggs): add TermsRefinementCoordinator for TPUT threshold computation and merge`

---

### Task 10: Create `TransportTermsRefinementAction`

**Files**: `server/src/main/java/org/elasticsearch/action/search/TransportTermsRefinementAction.java` (new file)
**Type**: new file

**What to do**:
Create the transport action that handles Phase 2 shard requests. This is executed on data nodes. It:
1. Receives a `TermsRefinementShardRequest`
2. Opens an index searcher for the target shard
3. Runs the original query to get the matching document set
4. Creates a `ThresholdTermsAggregator` with the threshold and seenTerms
5. Executes the aggregator against the matching documents
6. Returns a `TermsRefinementShardResponse`

**Test first (Red)**:
```java
// TransportTermsRefinementActionTests.java
// Use MockTransportService pattern from existing ES tests
public class TransportTermsRefinementActionTests extends ESTestCase {

    public void testShardExecutionReturnsNewTerms() {
        // Set up a shard with known data
        // Send refinement request with threshold
        // Verify response contains terms above threshold
    }

    public void testShardExecutionExcludesSeenTerms() {
        // Send refinement request with seenTerms
        // Verify those terms are not in the response
    }

    public void testShardExecutionIncludesSubAggregations() {
        // Send refinement request with sub-agg factories
        // Verify sub-agg results are present in response buckets
    }
}
```

**Implement (Green)**:

```java
// TransportTermsRefinementAction.java
package org.elasticsearch.action.search;

import org.elasticsearch.action.ActionType;
import org.elasticsearch.action.support.ActionFilters;
import org.elasticsearch.action.support.single.shard.TransportSingleShardAction;
import org.elasticsearch.cluster.ClusterState;
import org.elasticsearch.cluster.metadata.IndexNameExpressionResolver;
import org.elasticsearch.cluster.routing.ShardsIterator;
import org.elasticsearch.cluster.service.ClusterService;
import org.elasticsearch.common.io.stream.Writeable;
import org.elasticsearch.index.shard.ShardId;
import org.elasticsearch.indices.IndicesService;
import org.elasticsearch.search.aggregations.bucket.terms.TermsRefinementShardRequest;
import org.elasticsearch.search.aggregations.bucket.terms.TermsRefinementShardResponse;
import org.elasticsearch.search.aggregations.bucket.terms.ThresholdTermsAggregator;
import org.elasticsearch.threadpool.ThreadPool;
import org.elasticsearch.transport.TransportService;

public class TransportTermsRefinementAction extends TransportSingleShardAction<
    TermsRefinementShardRequest,
    TermsRefinementShardResponse
> {

    public static final String NAME = "indices:data/read/search[phase/terms_refinement]";
    public static final ActionType<TermsRefinementShardResponse> TYPE = new ActionType<>(NAME);

    private final IndicesService indicesService;

    // Constructor with @Inject annotation for DI
    // Standard TransportSingleShardAction pattern

    @Override
    protected TermsRefinementShardResponse shardOperation(
        TermsRefinementShardRequest request,
        ShardId shardId
    ) {
        // 1. Get IndexShard from indicesService
        // 2. Acquire searcher
        // 3. Build query from request.searchSource()
        // 4. Create ThresholdTermsAggregator with request.threshold(), request.seenTerms()
        // 5. Execute aggregator
        // 6. Build and return TermsRefinementShardResponse
        return null; // placeholder
    }

    @Override
    protected Writeable.Reader<TermsRefinementShardResponse> getResponseReader() {
        return TermsRefinementShardResponse::new;
    }

    @Override
    protected boolean resolveIndex(TermsRefinementShardRequest request) {
        return true;
    }

    @Override
    protected ShardsIterator shards(ClusterState state, InternalRequest request) {
        // Route to the specific shard from the request
        return null; // placeholder
    }
}
```

**Verify**:
```bash
./gradlew :server:test --tests "org.elasticsearch.action.search.TransportTermsRefinementActionTests"
```

**Commit**: `feat(aggs): add TransportTermsRefinementAction for Phase 2 shard execution`

---

### Task 11: Create `AggregationRefinementPhase`

**Files**: `server/src/main/java/org/elasticsearch/action/search/AggregationRefinementPhase.java` (new file)
**Type**: new file

**What to do**:
Create the new `SearchPhase` that sits between query reduce and fetch. It inspects the reduced aggregation results for any `InternalTerms` with `needsRefinement == true`, then orchestrates the TPUT refinement using `TermsRefinementCoordinator`.

**Test first (Red)**:
```java
// AggregationRefinementPhaseTests.java
public class AggregationRefinementPhaseTests extends ESTestCase {

    public void testSkipsWhenNoRefinementNeeded() {
        // Create reduced results with needsRefinement=false
        // Verify phase is a no-op and proceeds to next phase immediately
    }

    public void testTriggersPhase2WhenRefinementNeeded() {
        // Create reduced results with needsRefinement=true
        // Verify Phase 2 shard requests are dispatched
    }

    public void testFinalResultHasZeroDocCountError() {
        // End-to-end: Phase 1 with error → Phase 2 → merged result has error=0
    }

    public void testMultipleExactTermsAggsInSameSearch() {
        // Two terms aggs with mode=exact in same search request
        // Verify both are refined
    }
}
```

**Implement (Green)**:

```java
// AggregationRefinementPhase.java
package org.elasticsearch.action.search;

import org.elasticsearch.search.SearchPhaseResult;
import org.elasticsearch.search.aggregations.InternalAggregation;
import org.elasticsearch.search.aggregations.InternalAggregations;
import org.elasticsearch.search.aggregations.bucket.terms.AbstractInternalTerms;
import org.elasticsearch.search.aggregations.bucket.terms.TermsRefinementCoordinator;
import org.elasticsearch.search.internal.SearchContext;
import org.elasticsearch.transport.TransportService;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.atomic.AtomicArray;

/**
 * Search phase that performs TPUT refinement for exact terms aggregations.
 *
 * Inserted between query reduce and fetch phase. Inspects reduced
 * aggregation results for any InternalTerms with needsRefinement=true.
 * If found, executes TPUT Phases 2 (and optionally 3) to produce
 * exact results before proceeding.
 *
 * When no exact-mode aggregations need refinement, this phase is a
 * no-op that immediately delegates to the next phase.
 */
public class AggregationRefinementPhase extends SearchPhase {

    static final String NAME = "agg_refinement";

    private final AbstractSearchAsyncAction<?> context;
    private final SearchPhaseController.ReducedQueryPhase reducedQueryPhase;
    private final AtomicArray<SearchPhaseResult> queryPhaseResults;
    private final TransportService transportService;

    AggregationRefinementPhase(
        AbstractSearchAsyncAction<?> context,
        SearchPhaseController.ReducedQueryPhase reducedQueryPhase,
        AtomicArray<SearchPhaseResult> queryPhaseResults,
        TransportService transportService
    ) {
        super(NAME);
        this.context = context;
        this.reducedQueryPhase = reducedQueryPhase;
        this.queryPhaseResults = queryPhaseResults;
        this.transportService = transportService;
    }

    @Override
    protected void run() {
        // 1. Scan reducedQueryPhase.aggregations for any AbstractInternalTerms
        //    with needsRefinement() == true
        List<AbstractInternalTerms<?, ?>> toRefine = findTermsNeedingRefinement(
            reducedQueryPhase.aggregations()
        );

        if (toRefine.isEmpty()) {
            // Fast path: no refinement needed, proceed to fetch
            proceedToFetchPhase();
            return;
        }

        // 2. For each terms agg needing refinement:
        //    a. Create TermsRefinementCoordinator
        //    b. Compute threshold T
        //    c. Send Phase 2 requests to all shards
        //    d. Collect responses
        //    e. Merge results
        //    f. Check if Phase 3 needed
        //    g. If so, send Phase 3 point requests
        //    h. Produce final InternalTerms with docCountError=0

        // 3. Replace the refined aggregations in reducedQueryPhase

        // 4. Proceed to fetch phase with updated results
        proceedToFetchPhase();
    }

    private List<AbstractInternalTerms<?, ?>> findTermsNeedingRefinement(
        InternalAggregations aggregations
    ) {
        List<AbstractInternalTerms<?, ?>> result = new ArrayList<>();
        if (aggregations == null) return result;
        for (InternalAggregation agg : aggregations.asList()) {
            if (agg instanceof AbstractInternalTerms<?, ?> terms && terms.needsRefinement()) {
                result.add(terms);
            }
            // Also check nested aggregations (terms inside other aggs)
        }
        return result;
    }

    private void proceedToFetchPhase() {
        // Delegate to FetchSearchPhase with the (potentially updated) reducedQueryPhase
        context.executeNextPhase(
            this,
            new FetchSearchPhase(context, reducedQueryPhase, queryPhaseResults)
        );
    }
}
```

**Verify**:
```bash
./gradlew :server:test --tests "org.elasticsearch.action.search.AggregationRefinementPhaseTests"
```

**Commit**: `feat(aggs): add AggregationRefinementPhase for TPUT orchestration in search pipeline`

---

### Task 12: Insert `AggregationRefinementPhase` into Search Phase Chain

**Files**: `server/src/main/java/org/elasticsearch/action/search/FetchSearchPhase.java` (modify) OR `server/src/main/java/org/elasticsearch/action/search/SearchPhaseController.java` (modify)
**Type**: modify existing

**What to do**:
Insert the `AggregationRefinementPhase` between query reduce and fetch. The insertion point is the `nextPhase()` method in `FetchSearchPhase` or the phase chain construction in the search action. The phase must be a clean no-op when no exact-mode aggs exist.

**Test first (Red)**:
```java
// SearchPhaseChainTests.java (or add to existing FetchSearchPhaseTests)
public void testAggRefinementPhaseInsertedWhenExactModePresent() {
    // Simulate a search with mode=exact terms agg
    // Verify AggregationRefinementPhase runs before FetchSearchPhase
}

public void testAggRefinementPhaseSkippedWhenApproximateMode() {
    // Simulate a search with mode=approximate (default)
    // Verify AggregationRefinementPhase is either not created or is a no-op
}
```

**Implement (Green)**:

The cleanest insertion point is to modify the phase chain so that after the query phase reduces, before fetch runs, we check for refinement. The approach depends on the exact current phase chain construction, but the key change is:

```java
// In the phase chain construction (e.g., where FetchSearchPhase is created
// after query reduce), wrap it:

SearchPhase fetchPhase = new FetchSearchPhase(context, reducedQueryPhase, queryResults);

// If any exact-mode terms aggs exist, insert refinement before fetch:
if (hasExactModeTermsAggs(reducedQueryPhase)) {
    SearchPhase refinementPhase = new AggregationRefinementPhase(
        context, reducedQueryPhase, queryResults, transportService
    );
    // refinementPhase.run() will internally proceed to fetchPhase when done
} else {
    fetchPhase.run();
}
```

Alternatively, modify `FetchSearchPhase.nextPhase()` to return the refinement phase first:

```java
// In FetchSearchPhase, before the existing nextPhase:
// This won't work since FetchSearchPhase.nextPhase returns what comes AFTER fetch.
// The insertion must be BEFORE fetch, not after.
```

The correct approach: modify the code that creates `FetchSearchPhase` (in `AbstractSearchAsyncAction` or `SearchQueryThenFetchAsyncAction`) to first create and run `AggregationRefinementPhase`, which then creates `FetchSearchPhase` as its next phase.

**Verify**:
```bash
./gradlew :server:test --tests "org.elasticsearch.action.search.FetchSearchPhaseTests"
./gradlew :server:test --tests "org.elasticsearch.action.search.SearchPhaseChainTests"
```

**Commit**: `feat(aggs): wire AggregationRefinementPhase into search phase chain`

---

### Task 13: End-to-End Integration Tests

**Files**: `server/src/test/java/org/elasticsearch/search/aggregations/bucket/terms/ExactTermsAggregationIT.java` (new file)
**Type**: new file

**What to do**:
Write integration tests that validate the entire flow from REST API to exact results. These tests should use the test infrastructure's multi-shard index setup to exercise the distributed behavior.

**Test first (Red)**:
```java
// ExactTermsAggregationIT.java
public class ExactTermsAggregationIT extends ESIntegTestCase {

    @Override
    protected int numberOfShards() {
        return 5; // Multiple shards to trigger cross-shard error
    }

    public void testExactModeProducesZeroError() throws Exception {
        // Index data with skewed distribution across shards
        // (e.g., routing to ensure specific terms land on specific shards)
        createIndex("test");
        for (int i = 0; i < 10000; i++) {
            // Create skewed distribution: "common" on all shards,
            // "shard_local_X" only on shard X
            indexDoc("test", String.valueOf(i),
                "status", randomFrom("active", "pending", "closed", "archived",
                    "draft", "review", "approved", "rejected", "expired", "suspended"));
        }
        refresh("test");

        // Approximate mode: may have error
        SearchResponse approxResponse = client().prepareSearch("test")
            .addAggregation(
                new TermsAggregationBuilder("statuses")
                    .field("status")
                    .size(5)
                    .shardSize(5)  // Force small shard_size to induce error
            )
            .get();

        // Exact mode: guaranteed zero error
        SearchResponse exactResponse = client().prepareSearch("test")
            .addAggregation(
                new TermsAggregationBuilder("statuses")
                    .field("status")
                    .size(5)
                    .shardSize(5)
                    .mode(TermsAggregationMode.EXACT)
            )
            .get();

        Terms exactTerms = exactResponse.getAggregations().get("statuses");
        assertThat(exactTerms.getDocCountError(), equalTo(0L));

        // Verify exact counts match a full scan
        SearchResponse fullScan = client().prepareSearch("test")
            .addAggregation(
                new TermsAggregationBuilder("statuses")
                    .field("status")
                    .size(100)  // get all terms
            )
            .get();

        Terms fullTerms = fullScan.getAggregations().get("statuses");
        for (Terms.Bucket exactBucket : exactTerms.getBuckets()) {
            Terms.Bucket fullBucket = fullTerms.getBucketByKey(exactBucket.getKeyAsString());
            assertThat(
                "Exact count for " + exactBucket.getKeyAsString() + " should match full scan",
                exactBucket.getDocCount(),
                equalTo(fullBucket.getDocCount())
            );
        }
    }

    public void testExactModeWithSubAggregations() throws Exception {
        // Index data with numeric field for avg sub-agg
        // Run exact terms with avg sub-agg
        // Verify sub-agg values are correct
    }

    public void testExactModeFastPath() throws Exception {
        // Index low-cardinality data (fewer terms than shard_size)
        // Verify only 1 round-trip (no Phase 2/3)
        // Result should still have docCountError=0
    }

    public void testExactModeWithNumericField() throws Exception {
        // Test with long/double field types
    }

    public void testExactModeRESTAPI() throws Exception {
        // Test via REST client with JSON:
        // {"aggs": {"top": {"terms": {"field": "status", "mode": "exact"}}}}
    }

    public void testApproximateModeUnchanged() throws Exception {
        // Verify default behavior is completely unchanged
        // No extra round-trips, same error bounds as before
    }
}
```

**Verify**:
```bash
./gradlew :server:internalClusterTest --tests "org.elasticsearch.search.aggregations.bucket.terms.ExactTermsAggregationIT"
```

**Commit**: `test(aggs): add end-to-end integration tests for exact terms aggregation`

---

### Task 14: REST API Spec and YAML Tests

**Files**:
- `rest-api-spec/src/main/resources/rest-api-spec/api/search.json` (modify if needed)
- `rest-api-spec/src/yamlRestTest/resources/rest-api-spec/test/search.aggregation/terms_exact_mode.yml` (new file)
**Type**: new file + modify existing

**What to do**:
Add YAML REST tests that verify the `mode` parameter is accepted and produces correct results via the REST API.

**Implement**:
```yaml
# terms_exact_mode.yml
---
"Terms aggregation with exact mode":
  - do:
      indices.create:
        index: test
        body:
          settings:
            number_of_shards: 3
          mappings:
            properties:
              status:
                type: keyword

  - do:
      bulk:
        refresh: true
        body:
          - index: { _index: test }
          - { status: "active" }
          - index: { _index: test }
          - { status: "pending" }
          - index: { _index: test }
          - { status: "active" }
          - index: { _index: test }
          - { status: "closed" }
          - index: { _index: test }
          - { status: "active" }

  - do:
      search:
        index: test
        body:
          size: 0
          aggs:
            top_status:
              terms:
                field: status
                size: 10
                mode: exact

  - match: { aggregations.top_status.doc_count_error_upper_bound: 0 }
  - length: { aggregations.top_status.buckets: 3 }
  - match: { aggregations.top_status.buckets.0.key: "active" }
  - match: { aggregations.top_status.buckets.0.doc_count: 3 }

---
"Terms aggregation default mode is approximate":
  - do:
      search:
        index: test
        body:
          size: 0
          aggs:
            top_status:
              terms:
                field: status
                size: 10

  # Default mode should work as before (no "mode" field in response)
  - is_true: aggregations.top_status.buckets
```

**Verify**:
```bash
./gradlew :rest-api-spec:yamlRestTest --tests "*terms_exact_mode*"
```

**Commit**: `test(aggs): add REST API YAML tests for exact terms mode`

---

## Cross-Repo Integration Points

| Source Repo | Target Repo | Integration Type | Risk Level | Notes |
|---|---|---|---|---|
| elasticsearch (server) | elasticsearch (server) | Internal — all changes within same module | **Low** | No cross-repo dependencies |
| elasticsearch (rest-api-spec) | elasticsearch (server) | REST API spec ↔ Server | **Low** | New optional parameter, backward compatible |
| elasticsearch (client) | elasticsearch (server) | Java High-Level REST Client | **Low** | Client auto-discovers new params from spec |

**This feature is entirely self-contained within the elasticsearch repository.** No other repositories are affected.

---

## Migration & Rollback Plan

### Feature Flag
- **`mode` parameter defaults to `"approximate"`** — exact mode is opt-in only
- No feature flag needed beyond the parameter itself
- Users must explicitly request `"mode": "exact"` to activate

### Mixed-Version Cluster Behavior
- **TransportVersion gating** ensures older nodes ignore the `mode` field
- If any shard in the cluster runs an older version, the coordinator silently degrades to approximate mode and logs a deprecation warning
- No data migration, no index changes, no cluster state changes

### Rollback Procedure
1. **Immediate**: Users stop sending `"mode": "exact"` in queries → instant rollback to approximate behavior
2. **Node rollback**: If nodes are downgraded, the `mode` field is silently ignored (unknown fields are ignored in aggregation parsing)
3. **No data cleanup**: No persistent state is created by this feature

### Performance Safety Valve
- `max_buckets` setting (default: 65,535) bounds the total number of buckets created in Phase 2
- If Phase 2 would exceed `max_buckets`, the search fails with a clear error message rather than consuming unbounded memory
- Circuit breaker (`indices.breaker.request.limit`) provides additional protection

---

## Open Questions

1. **Phase 3 implementation priority**: Phase 3 (point queries for specific term-shard pairs) handles a rare edge case. Should it be implemented in v1, or should v1 use a simpler fallback (e.g., re-run Phase 2 with lower threshold)?

2. **`order` interaction**: When `order` is not `_count desc` (e.g., `_key asc`), the TPUT threshold based on doc_count may not directly apply. Should `mode: "exact"` be restricted to `_count` ordering initially?

3. **`min_doc_count` interaction**: When `min_doc_count > 1`, should exact mode guarantee that no term above min_doc_count is missed, or should it guarantee the full top-k is correct regardless?

4. **Composite aggregation interaction**: Should exact mode work inside composite aggregation pages? This would require persisting refinement state across pages.

5. **Performance benchmarking**: What is the actual overhead of Phase 2 on realistic workloads? Should there be a configurable threshold for "acceptable error" below which refinement is skipped?

6. **Action registration**: The `TransportTermsRefinementAction` needs to be registered in the appropriate module plugin class (`SearchModule` or `ActionModule`). Verify the exact registration pattern.

---

## Task Dependency Graph

```
Task 1: TermsAggregationMode enum
  │
  ├──► Task 2: mode in TermsAggregationBuilder (depends on Task 1)
  │     │
  │     ├──► Task 3: TransportVersion constant (depends on Task 2)
  │     │
  │     └──► Task 4: Propagate mode through Factory/Aggregator (depends on Task 2)
  │
  ├──► Task 5: Carry mode in InternalTerms serialization (depends on Task 1)
  │     │
  │     └──► Task 6: Refinement trigger in reduce (depends on Task 5)
  │
  ├──► Task 7: Request/Response transport objects (depends on Task 1)
  │     │
  │     ├──► Task 8: ThresholdTermsAggregator (depends on Task 7)
  │     │
  │     ├──► Task 9: TermsRefinementCoordinator (depends on Task 7)
  │     │
  │     └──► Task 10: TransportTermsRefinementAction (depends on Tasks 7, 8)
  │
  └──► Task 11: AggregationRefinementPhase (depends on Tasks 6, 9, 10)
        │
        └──► Task 12: Wire into search phase chain (depends on Task 11)
              │
              ├──► Task 13: Integration tests (depends on Task 12)
              │
              └──► Task 14: REST API YAML tests (depends on Task 12)
```

**Critical path**: Tasks 1 → 2 → 5 → 6 → 11 → 12 → 13

**Parallelizable**:
- Tasks 3 and 4 can run in parallel after Task 2
- Tasks 7, 8, 9 can run in parallel after Task 1
- Task 10 depends on Tasks 7 and 8
- Tasks 13 and 14 can run in parallel after Task 12

---

## Estimated Complexity

| Task | Complexity | Notes |
|------|-----------|-------|
| Task 1: TermsAggregationMode | Low | Simple enum |
| Task 2: mode in Builder | Low | Follow existing field pattern |
| Task 3: TransportVersion | Low | Single constant |
| Task 4: Factory propagation | Low | Pass-through |
| Task 5: InternalTerms serialization | Medium | Multiple concrete classes |
| Task 6: Refinement trigger | Medium | Core algorithm logic |
| Task 7: Transport objects | Medium | Serialization of complex types |
| Task 8: ThresholdTermsAggregator | **High** | New aggregator, complex Lucene integration |
| Task 9: RefinementCoordinator | **High** | Core TPUT algorithm, merge logic |
| Task 10: Transport action | Medium | Standard pattern but needs search context |
| Task 11: RefinementPhase | **High** | Async phase orchestration |
| Task 12: Phase chain wiring | Medium | Integration with existing chain |
| Task 13: Integration tests | Medium | Multi-shard test setup |
| Task 14: YAML tests | Low | Declarative tests |
