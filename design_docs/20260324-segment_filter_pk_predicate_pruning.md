# MEP: Delegator-Side Segment Filtering with PK Predicate Pruning

- **Created:** 2026-03-24
- **Author(s):** @xiaofan-luan
- **Status:** Draft
- **Component:** Proxy, QueryNode
- **Related Issues:** #47804
- **Released:** TBD

## Summary

Add a two-stage segment pruning mechanism at the QueryNode delegator level that leverages primary key (PK) predicates in search/query requests to skip irrelevant segments before dispatching work to QueryNode workers. Stage 1 uses PK min/max statistics for range expression pruning; Stage 2 uses bloom filters for PK term/equality pruning. A lightweight proxy-side hint avoids unnecessary plan deserialization when no PK predicate exists.

## Motivation

In Milvus, each search or query request is dispatched by the shard delegator to all segments assigned to that shard. For collections with many sealed segments, this can lead to significant unnecessary computation when the query contains PK-based predicates (e.g., `pk in [1, 2, 3]` or `pk > 100`).

**Current behavior:** The delegator sends the request to every segment regardless of whether it could possibly contain matching PKs. Bloom filters and PK statistics already exist on each segment for delete deduplication, but they are not leveraged during search/query segment dispatch.

**Key observation:** Many real-world workloads filter by PK (point lookups, range scans). For these queries, we can use existing per-segment metadata to prune segments at the delegator — before any vector search or data scan occurs. This is especially impactful for:

- **Point queries** (`pk = X` or `pk in [list]`): Bloom filters can definitively exclude segments that don't contain the target PKs.
- **Range queries** (`pk > X`, `pk < Y`): Min/max PK statistics can exclude segments whose PK range doesn't overlap the predicate.

The pruning happens at the delegator (Go layer), avoiding the overhead of dispatching to C++ segcore for segments that provably contain no matching rows.

## Public Interfaces

### New Configuration Parameters

| Parameter | Key | Default | Description |
|-----------|-----|---------|-------------|
| `EnableSegmentFilter` | `queryNode.enableSegmentFilter` | `true` | Enable delegator-side segment filtering using PK predicates (min/max + bloom filter) |

### Proto Changes

**`internal.proto`** — New field on `SearchRequest` and `RetrieveRequest`:

```protobuf
message SearchRequest {
    // ... existing fields ...
    int32 pk_filter = N;  // Proxy-set hint: 0 = unknown, 1 = no PK predicate, 2 = has PK predicate
}

message RetrieveRequest {
    // ... existing fields ...
    int32 pk_filter = N;  // Same semantics as above
}
```

**Constants** (in `pkg/common/common.go`):

```go
const (
    PkFilterUnknown    = int32(0) // Not analyzed (backward compat with old proxies)
    PkFilterNoPkFilter = int32(1) // Proxy confirmed no optimizable PK predicate
    PkFilterHasPkFilter = int32(2) // Proxy confirmed optimizable PK predicate exists
)
```

### New Metrics

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `milvus_querynode_segment_filter_hit_segment_num` | Histogram | nodeID, collectionID, queryType | Number of segments with bloom filter hits (not pruned) |
| `milvus_querynode_segment_filter_skipped_segment_num` | Histogram | nodeID, collectionID, queryType | Number of segments pruned by the filter |

## Design Details

### Architecture Overview

```
┌─────────┐     ┌────────────────────┐     ┌──────────────────────┐
│  Proxy  │────▶│ Shard Delegator    │────▶│  QueryNode Workers   │
│         │     │                    │     │                      │
│ Analyze │     │ Stage 0: Hint      │     │  (only receives      │
│ plan,   │     │ Stage 1: Min/Max   │     │   non-pruned         │
│ set     │     │ Stage 2: Bloom     │     │   segments)          │
│ PkFilter│     │ Filter segments    │     │                      │
└─────────┘     └────────────────────┘     └──────────────────────┘
```

### Stage 0: Proxy-Side Hint (Avoid Unnecessary Unmarshal)

The proxy analyzes the query plan at compile time and sets the `PkFilter` hint field:

- **`PkFilterNoPkFilter` (1):** No PK predicate found → delegator skips all filtering (no plan unmarshal needed).
- **`PkFilterHasPkFilter` (2):** Optimizable PK predicate found → delegator proceeds with filtering.
- **`PkFilterUnknown` (0):** Old proxy or mixed-version rolling upgrade → delegator falls back to attempting unmarshal.

The proxy walks the expression tree looking for PK predicates (TermExpr, UnaryRangeExpr) and handles logical operators:
- `AND(pk_pred, other)` → optimizable
- `OR(pk_pred, non_pk_pred)` → **not** optimizable (non-PK side is unconstrained)
- `NOT(pk_pred)` → **not** optimizable (negation invalidates pruning)

This avoids the cost of `proto.Unmarshal` on the delegator for the majority of queries that don't filter by PK.

### Stage 1: Min/Max Pruning (Range Expressions)

For PK range predicates (`pk > X`, `pk < Y`, `pk >= X`, `pk <= Y`, `pk = X`), the delegator checks each sealed segment's PK min/max statistics:

- If the segment's `[minPK, maxPK]` range is entirely outside the predicate range, the segment is skipped.
- Conjunction (`AND`) of multiple range predicates is evaluated conjunctively — a segment must satisfy all conditions to survive.
- Disjunction (`OR`) and other complex expressions are treated conservatively (segment kept).

**Only sealed segments** are pruned. Growing segments are excluded because:
1. Their min/max stats are mutable (concurrent inserts).
2. Growing segments are typically small, so the pruning benefit is minimal.

### Stage 2: Bloom Filter Pruning (Term/Equality Expressions)

For PK point predicates (`pk IN [values]` or `pk = value`), the delegator uses per-segment bloom filters:

1. **Extract PK constraint:** Walk the expression tree to collect the set of PK values, handling:
   - `TermExpr(pk, [v1, v2, ...])` → PK in set
   - `UnaryRangeExpr(pk, Equal, v)` → PK = v (treated as single-element set)
   - `AND(left, right)` → intersection of PK sets
   - `OR(left, right)` → union of PK sets (only if both sides have PK constraints)
   - `NOT(inner)` → PK constraint from inner is ignored (conservative)

2. **Batch bloom filter check:** For each sealed segment, test all extracted PK values against the segment's bloom filter.

3. **Pruning decision:** If **no** PK value has a bloom filter hit for a segment, the segment is skipped (bloom filters have no false negatives for membership testing).

### Partition Filtering

Before PK pruning, segments are also filtered by requested partitions. This is a simple set membership check that reduces the number of segments entering the PK pruning stages.

### Data Flow

```
Search/Query Request
│
├── Proxy: computeSegmentFilter(plan) → set PkFilter hint
│
├── Delegator: buildAndApplySegmentFilters()
│   ├── Check EnableSegmentFilter config flag
│   ├── Check PkFilter hint (early exit if NoPkFilter)
│   ├── proto.Unmarshal(serializedPlan)
│   ├── Filter segments by partitions
│   ├── Stage 1: buildSkippedSegmentsByPredicates() [min/max]
│   │   └── evalExprWithCandidate() for each sealed segment
│   ├── Stage 2: buildSegmentFilterFromPredicates() [bloom filter]
│   │   ├── extractPkConstraint() → pkConstraint{dataType, values}
│   │   ├── BatchGetFromSealedSegments() → per-segment BF hit results
│   │   └── Segments with no BF hits → skipped
│   └── Report metrics
│
└── Dispatch to QueryNode workers (pruned segment lists)
```

### Correctness Guarantees

- **No false negatives:** Bloom filter testing is safe — a negative result guarantees the PK is not in the segment. Only positive results may be false positives (segment is kept when it might not need to be).
- **Min/max is conservative:** If min/max stats are unavailable, the segment is kept.
- **Growing segments untouched:** Growing segment lists are never pruned by PK, avoiding races with concurrent inserts.
- **Empty intersection:** If AND produces an empty PK set (impossible predicate), all sealed segments are skipped — the query will return empty results as expected.

## Compatibility, Deprecation, and Migration Plan

- **Rolling upgrade safe:** The `PkFilter` field defaults to `0` (Unknown). Old proxies that don't set the hint will cause the delegator to attempt unmarshal and analyze on its own. New proxies with old delegators simply have the hint field ignored.
- **Feature flag:** `queryNode.enableSegmentFilter` defaults to `true` but can be disabled at runtime if issues arise.
- **No API changes:** This is purely an internal optimization. External search/query APIs are unchanged.

## Test Plan

- **Unit tests:** `segment_filter_test.go` covers:
  - PK constraint extraction for all expression types (Term, Equal, AND, OR, NOT, nested combinations)
  - Bloom filter pruning with multiple segments and PK values
  - Min/max range pruning for all comparison operators
  - Partition filtering
  - Edge cases: nil plans, empty predicates, unsupported types, offline segments
  - End-to-end `buildAndApplySegmentFilters` with config flag toggling
- **Proxy-side tests:** `task_search_pk_hint_test.go` covers `computeSegmentFilter()` for all plan types.
- **Benchmark tests:** `segment_filter_bench_test.go` measures proto marshal/unmarshal overhead for plans with 2000 PKs.
- **E2E:** Existing search/query integration tests ensure correctness is preserved — the optimization only affects which segments are scanned, not results.

## Rejected Alternatives

### Pass PK Values from Proxy to Delegator via Proto

An earlier design considered having the proxy extract PK values and pass them explicitly in the SearchRequest/QueryRequest proto (via a `SegmentPkHint` message). This was rejected because:
- It duplicates data already in the serialized plan, increasing message size.
- For large IN-lists (thousands of PKs), the overhead of serializing PK values twice is wasteful.
- The delegator already needs to unmarshal the plan for other purposes in some code paths.

### Prune Growing Segments Too

Growing segments have bloom filters and PK statistics, but they are mutable under concurrent inserts. Pruning them would require either locking (latency cost) or accepting race conditions (correctness risk). Since growing segments are typically small, the benefit doesn't justify the complexity.

### Always Unmarshal Plan at Delegator

Without the proxy hint, the delegator would unmarshal every query plan to check for PK predicates. Benchmarks showed this adds ~20-50µs per request even for plans with no PK predicate. The hint avoids this overhead for the common case.

## References

- PR: https://github.com/milvus-io/milvus/pull/47805
- Issue: https://github.com/milvus-io/milvus/issues/47804
