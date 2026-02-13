# Milvus Query ORDER BY Feature Design Document

## Overview

Unlike Search ORDER BY (which operates at the proxy level after requery), Query ORDER BY pushes sorting into the segcore execution pipeline. Each segment sorts locally and returns top-K results; the proxy then merges, deduplicates, and re-maps fields.

## API

```python
# PyMilvus Example
collection.query(
    expr="age > 20",
    output_fields=["A", "B", "C"],
    order_by_fields=[
        {"field": "B", "order": "asc"},
        {"field": "C", "order": "desc"}
    ],
    limit=10
)
```

## Segcore Execution Pipeline

### Single-Project Mode (non-sort output columns are all fixed-width)

When all output columns that are NOT sort keys have fixed-width types (int, float, double, etc.), a single ProjectNode materializes all columns before sorting:

```
FilterBitsNode -> MvccNode -> ProjectNode[pk, B, C, A] -> OrderByNode(B, C, limit)
```

**Rationale**: Fixed-width values are stored inline in RowContainer (no per-row heap allocation), so including them in the sort buffer has negligible overhead.

### Two-Project Mode (non-sort output columns contain variable-width types)

When any output column that is NOT a sort key has a variable-width type (VARCHAR, STRING), a second ProjectNode is added after sorting:

```
FilterBitsNode -> MvccNode -> Project_1[pk, B, C] -> OrderByNode(B, C, limit) -> Project_2[A]
```

**Rationale**: VARCHAR values require `new std::string(...)` heap allocation per row during RowContainer storage. For a segment with 10,000 matching rows and `limit=10`, single-project mode would heap-allocate 10,000 strings only to discard 9,990 after TopK. Two-project mode defers VARCHAR materialization to after TopK selection, allocating only 10 strings.

### Decision Logic

```
for each column in outputFields:
    if column is already a sort key:
        skip (already in Project_1)
    else if column is fixed-width (bool, int8..int64, float, double):
        include in Project_1 (single-project)
    else if column is variable-width (VARCHAR, STRING):
        defer to Project_2 (two-project)
```

If all non-sort output columns are fixed-width -> single-project mode.
If any non-sort output column is variable-width -> two-project mode.

## Positional Field Layout Contract

Segcore returns columns in a fixed positional order - **no field_id is set on column vectors**. The proxy interprets columns purely by position.

For `SELECT A, B, C FROM table ORDER BY B, C`:

```
Segcore returns: [pk, B, C, A]
                  ^   ^----^  ^
                  |   |       +-- non-sort output fields (original user order)
                  |   +---------- ORDER BY fields (sort key order)
                  +-------------- PK (implicit, for proxy reduce/dedup)
```

**Layout rule**:
- Position 0: **PK** - always included implicitly for proxy-side reduce/deduplication
- Position 1..N: **ORDER BY fields** - in the order specified by the user's `order_by_fields`
- Position N+1..M: **Remaining output fields** - output fields not in ORDER BY, preserving the user's original `output_fields` order

**Deduplication**: If a field appears in both ORDER BY and output fields, it appears only once in the ORDER BY section and is not repeated in the output section.

## Proxy-Side Field Re-mapping

The proxy knows the positional layout and performs the final re-mapping before returning results to the client:

1. **Reduce**: Use position 0 (PK) for deduplication across segments
2. **Merge/Sort**: Use positions 1..N (ORDER BY fields) for cross-segment k-way merge
3. **Slice**: Apply user-specified offset/limit
4. **Re-map**: Reorganize field data from `[pk, orderby_fields, output_fields]` back into the user's requested `output_fields` order
5. **Strip**: Remove PK and sort-only fields (fields in ORDER BY but not in user's `output_fields`) from the final response

**Example**:

```
User request:   output_fields=["A", "B", "C"], order_by=["B ASC", "C DESC"]
Segcore returns: [pk, B, C, A]  (positional contract)

Proxy processing:
  1. Reduce by pk (position 0)
  2. K-way merge by B, C (positions 1, 2)
  3. Apply offset/limit
  4. Re-map to user order: [A, B, C]
  5. Strip pk (user didn't request it)
  -> Return to client: [A, B, C]
```

If B is only in ORDER BY but not in `output_fields`:

```
User request:   output_fields=["A", "C"], order_by=["B ASC"]
Segcore returns: [pk, B, A, C]

Proxy processing:
  -> Strip pk and B (sort-only)
  -> Return to client: [A, C]
```

## SortBuffer Internals

The `SortBuffer` (at `internal/core/src/exec/SortBuffer.h`) is the core sorting component:

- **Storage**: All columns (sort + non-sort) are stored in `RowContainer`. Fixed-width values are inline; VARCHAR values are heap-allocated pointers.
- **Sorting**: Only sorts `char*` row pointers (8 bytes each), not row data. Uses `sort_keys_` to compare - non-sort columns are completely ignored during comparison.
- **TopK**: When `limit < n/2`, uses `std::partial_sort` (O(n log k)) instead of `std::sort` (O(n log n)).
- **Output**: Extracts columns from sorted row pointers via copy into new `ColumnVector`s.

## Query ORDER BY with GROUP BY

For queries combining GROUP BY and ORDER BY:

```
FilterBitsNode -> MvccNode -> ProjectNode -> AggregationNode -> OrderByNode
```

In GROUP BY semantics:
- PK is a non-group-by field and should **NOT** be implicitly projected
- The aggregation node handles group-level reduction
- ORDER BY sorts the aggregated groups

## Query ORDER BY without GROUP BY

In simple ORDER BY without GROUP BY:
- PK **MUST** be projected (implicit "group by PK" semantics for deduplication)
- The proxy uses PK for cross-segment reduce/dedup
