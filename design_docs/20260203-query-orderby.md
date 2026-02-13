# Query ORDER BY

## 1. Overview

Query ORDER BY pushes sorting into the segcore execution pipeline. Each segment filters, projects, sorts locally, and returns top-K results. The QueryNode merges results from multiple segments, and the proxy performs global deduplication, merge-sort, pagination, and field re-mapping before returning to the client.

This differs fundamentally from Search ORDER BY, which operates at the proxy level after vector search requery. Query ORDER BY has no vector search step — it is a pure filter-project-sort pipeline.

### End-to-End Data Flow

```
Client SDK
  │  order_by=["price:asc", "rating:desc"]
  ▼
Proxy (task_query.go)
  │  Parse order_by strings → OrderByField protos
  │  Validate field types, build QueryPlanNode
  ▼
QueryNode (segment-level)
  │  Segcore pipeline: Filter → Project → OrderBy → output
  │  FillOrderByResult: set field_id, fetch deferred fields, populate system fields
  │  MergeSegcoreRetrieveResults: merge results from multiple segments
  ▼
Proxy Pipeline (query_pipeline.go)
  │  ReduceByPK → OrderOperator → SliceOperator → RemapOperator
  │  Populate FieldName from schema, filter to user's output_fields
  ▼
Client SDK
  │  Results with correct field names and user-requested field order
```

### Supported Data Types

| Sortable | Not Sortable |
|----------|-------------|
| Bool, Int8, Int16, Int32, Int64 | FloatVector, BFloat16Vector, ... |
| Float, Double | Array, JSON (without path) |
| String, VarChar | Timestamptz (not yet supported) |

## 2. API & Proto

### Client API (PyMilvus)

Format: `"field_name:direction"` where direction is `asc` (default) or `desc`. Multiple fields specify multi-key sort priority.

### Proto Definition (plan.proto)

`OrderByField` message carries field_id, ascending (bool, default true), and nulls_first (bool, default false). `QueryPlanNode` includes a repeated `OrderByField order_by_fields` field alongside predicates, limit, group_by, and aggregates.

## 3. Proxy: Request Parsing & Validation

**File**: `internal/proxy/task_query.go`

### 3.1 Parsing order_by Strings

`translateOrderByFields` parses user-provided strings like `"price:desc"` into OrderByField protos:

- Splits on `:` or space → field name + direction
- Looks up field_id from collection schema
- Validates the field type is sortable via `isSortableFieldType`
- Default direction: ascending. Default nulls: NULLS LAST.

### 3.2 Validation Rules

1. **Type check**: Only sortable types are allowed (see table above). Vectors, arrays, JSON (without path) are rejected.
2. **GROUP BY compatibility**: When GROUP BY is present, ORDER BY can only reference columns in the GROUP BY clause or aggregate function results (e.g., `sum(price)`).
3. **limit required**: ORDER BY without limit is rejected (unbounded sort is too expensive).

### 3.3 Plan Assembly

The parsed OrderByField list is set on the QueryPlanNode and sent to the QueryNode via gRPC.

## 4. Segcore: Plan Creation & Pipeline

**File**: `internal/core/src/query/PlanProto.cpp`

### 4.1 Pipeline Structure

For `output_fields=["A","B","C"], order_by=["B:asc","C:desc"]`:

**Single-project mode** (all non-sort output fields are fixed-width):

```
FilterBitsNode → MvccNode → ProjectNode[pk, B, C, A, SegOffset] → OrderByNode(B, C, limit)
```

**Two-project mode** (any non-sort output field is variable-width, e.g., VARCHAR):

```
FilterBitsNode → MvccNode → ProjectNode[pk, B, C, SegOffset] → OrderByNode(B, C, limit)
                                                                       │
                                        deferred fields [A] fetched later by FillOrderByResult
```

### 4.2 Two-Project Mode: Why It Matters

VARCHAR values require heap allocation per row during RowContainer storage. Consider a segment with 10,000 matching rows and `limit=10`:

- **Single-project**: Heap-allocates 10,000 strings, sorts, discards 9,990. Wasteful.
- **Two-project**: Only projects fixed-width columns into the sort buffer. After TopK selects 10 winners, bulk-fetches the 10 VARCHAR values via segment offsets. 1000x fewer allocations.

### 4.3 Mode Decision Logic

For each non-sort output column: fixed-width types (bool, int8..int64, float, double) are included in the ProjectNode; variable-width types (VARCHAR, STRING) trigger two-project mode.

Any variable-width column causes ALL non-sort output columns to be deferred. This "all or nothing" deferral simplifies the positional layout: pipeline always produces `[pk, orderby_fields, SegOffset]` in two-project mode, regardless of how many fixed-width fields exist.

### 4.4 Positional Layout Contract

The pipeline returns columns in a fixed positional order. SegmentOffsetFieldID is always appended as the last column.

```
Pipeline output: [pk, B, C, (A if single-project), SegOffset]
                  ^   ^----^   ^                      ^
                  |   |        |                      +-- segment offset for deferred fetch
                  |   |        +-- non-sort output (single-project only)
                  |   +---------- ORDER BY fields (sort key order)
                  +-------------- PK (for proxy reduce/dedup)
```

If a field appears in both ORDER BY and output_fields, it occupies only the ORDER BY position and is not repeated.

### 4.5 OrderByNode & SortBuffer

The OrderByNode wraps a SortBuffer:

- **Storage**: All projected columns stored in RowContainer. Fixed-width values are inline; VARCHAR values are heap-allocated pointers.
- **Sorting**: Sorts row pointers (8 bytes each), not row data. Uses sort_keys_ to compare — non-sort columns are completely ignored during comparison.
- **TopK optimization**: When `limit < n/2`, uses partial_sort O(n log k) instead of full sort O(n log n).
- **Output**: Extracts columns from sorted row pointers via copy into new ColumnVectors.

## 5. Segcore: Result Assembly (FillOrderByResult)

**File**: `internal/core/src/segcore/SegmentInterface.cpp`

After the pipeline produces sorted top-K rows, FillOrderByResult assembles the final RetrieveResults proto in five steps:

### Step 1: Move Pipeline Columns

Move all columns except the last (SegmentOffsetFieldID) into results.fields_data. Set field_id on each DataArray using pipeline_field_ids_ — this is critical because pipeline-produced DataArrays have field_id=0 by default, and the QN-side merge matches by field_id.

### Step 2: Extract Segment Offsets

The last pipeline column contains segment-level row offsets. These are stored in results.offset() for deferred field fetching (Step 3) and QN-side merge validation (MergeSegcoreRetrieveResults checks offset length > 0).

### Step 3: Bulk-Fetch Deferred Fields (Two-Project Mode)

For each deferred field, bulk_subscript is called with the segment offsets to fetch exactly the top-K rows. Special handling exists for:
- **Dynamic fields**: JSON subfield projection via target_dynamic_fields_
- **Schema evolution**: Fields absent in this segment get default values via bulk_subscript_not_exist_field
- **Array type**: element_type is set on the output DataArray

### Step 4: Populate System Fields

Fetch system fields (e.g., TimestampField) via segment offsets. The timestamp field is required by QN-side pk+ts deduplication logic.

### Step 5: Set IDs from PK Column

Extract PK values from position 0 and populate results.ids (either int_id or str_id). This is required by the proxy's ReduceByPK operator.

## 6. ShouldIgnoreNonPk Bypass

**File**: `internal/core/src/segcore/plan_c.cpp`

Normal queries use a two-phase retrieval optimization: first fetch only PKs, then re-fetch full fields via offsets. This is controlled by ShouldIgnoreNonPk.

ORDER BY queries **must bypass** this optimization because:
- The pipeline returns data in a positional layout [pk, orderby, remaining]
- The proxy's Remap operator depends on this exact positional order
- Two-phase retrieval would re-fetch via FillTargetEntry in field_ids_ order, breaking the positional contract

When has_order_by_ is true, ShouldIgnoreNonPk returns false so the pipeline output is used directly.

## 7. QueryNode: Segment Merge

**File**: `internal/querynodev2/segments/result.go`

MergeSegcoreRetrieveResults merges results from multiple segments on the same QueryNode:

- Uses PrepareResultFieldData to create output containers, copying field schema from the first non-empty segment result (field_id, field_name, field_type)
- AppendFieldData matches fields by field_id (not position) using a map keyed by FieldId
- Preserves the positional layout from segcore — the merged result has the same column order as individual segment results

**Critical invariant**: Since AppendFieldData matches by field_id, FillOrderByResult must set correct field_id values on pipeline-produced DataArrays (Step 1 above). Without this, all columns would have field_id=0 and data would be corrupted during merge.

## 8. Proxy Pipeline

**File**: `internal/proxy/query_pipeline.go`

The proxy-side pipeline processes the merged results from all QueryNodes:

```
input → ReduceByPK → OrderOperator → SliceOperator → RemapOperator → output
```

### 8.1 ReduceByPK

Deduplicates rows across QueryNodes using the PK column (position 0). Rows with the same PK but older timestamps are discarded.

### 8.2 OrderOperator

**File**: `internal/util/queryutil/order_op.go`

Global merge-sort of results from all QueryNodes.

- Uses **positional comparison**: ORDER BY field i is at FieldsData[i+1] (position 0 is PK)
- When offset + limit < rowCount, uses **heap-based partial sort** O(N log K) instead of full sort O(N log N)
- Supports null handling with configurable NULLS FIRST / NULLS LAST
- Stable sort: equal rows preserve original order

### 8.3 SliceOperator

Applies user-specified offset and limit to the sorted result. For pagination: page 2 with limit=10 uses offset=10, limit=10.

### 8.4 RemapOperator

**File**: `internal/util/queryutil/remap_op.go`

Reorders FieldsData from segcore's positional layout to the user's requested output field order.

BuildRemapIndices reconstructs the segcore positional layout (mirroring PlanProto.cpp logic), then computes an index mapping for each user output field:

```
Segcore layout: [pk(pos=0), price(pos=1), rating(pos=2), name(pos=3), category(pos=4)]
User wants:     [name, category, price]
Remap indices:  [3, 4, 1]
```

Fields that are only in ORDER BY (not in user's output_fields) and the implicit PK are stripped by this step.

## 9. Proxy: Response Assembly

**File**: `internal/proxy/task_query.go`

After the pipeline completes, two final steps prepare the response:

### 9.1 FieldName Population

Segcore does not set field_name on DataArray protos (only field_id). PyMilvus uses field_data.field_name (not response.output_fields) to build result dict keys. Without this step, all fields would have field_name="".

The proxy iterates over all FieldsData entries, looks up each field_id in the collection schema, and sets FieldName, Type, and IsDynamic accordingly. This mirrors the search pipeline's endOperator behavior.

### 9.2 Output Field Filtering

Filter FieldsData to only include fields the user requested in output_fields. Removes any internal fields (PK if not requested, sort-only fields, system fields) that leaked through the pipeline.

## 10. GROUP BY + ORDER BY Interaction

When GROUP BY is present, the pipeline becomes:

```
FilterBitsNode → MvccNode → ProjectNode → AggregationNode → OrderByNode
```

Key differences from plain ORDER BY:
- **No implicit PK projection**: PK is a non-group-by field and is NOT projected
- **ORDER BY validation**: Can only reference GROUP BY columns or aggregate function results
- **No RemapOperator**: The aggregation pipeline produces results in a different layout; remap is not needed
- **No ReduceByPK**: Uses ReduceByGroups instead

## 11. Key Implementation Files

| Layer | File | Purpose |
|-------|------|---------|
| Proto | `pkg/proto/plan.proto` | OrderByField, QueryPlanNode definition |
| Proxy | `internal/proxy/task_query.go` | Parse, validate, plan creation, FieldName population |
| Proxy | `internal/proxy/query_pipeline.go` | Pipeline construction (reduce → order → slice → remap) |
| Proxy | `internal/util/queryutil/order_op.go` | OrderOperator: global merge-sort with partial sort optimization |
| Proxy | `internal/util/queryutil/remap_op.go` | RemapOperator: positional → user field order mapping |
| Proxy | `internal/util/reduce/orderby/types.go` | OrderByField type definition, sortable type check |
| C++ | `internal/core/src/query/PlanProto.cpp` | BuildOrderByProjectNode, BuildOrderByNode |
| C++ | `internal/core/src/query/PlanNode.h` | RetrievePlanNode: deferred_field_ids_, pipeline_field_ids_ |
| C++ | `internal/core/src/segcore/SegmentInterface.cpp` | FillOrderByResult: deferred fetch, system fields, IDs |
| C++ | `internal/core/src/segcore/plan_c.cpp` | ShouldIgnoreNonPk bypass for ORDER BY |
| C++ | `internal/core/src/exec/SortBuffer.h` | RowContainer sort with TopK optimization |
| QN | `internal/querynodev2/segments/result.go` | MergeSegcoreRetrieveResults: field_id-based merge |
