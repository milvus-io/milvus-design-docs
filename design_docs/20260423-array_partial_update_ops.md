# MEP: Array Field Partial Update Operators

Current state: Under Discussion

ISSUE: [[Feature]: Field-level partial update operators for Array fields #49241](https://github.com/milvus-io/milvus/issues/49241)

PRs:
- proto: [milvus-io/milvus-proto#586](https://github.com/milvus-io/milvus-proto/pull/586)
- core: [milvus-io/milvus#49251](https://github.com/milvus-io/milvus/pull/49251)
- pymilvus: [milvus-io/pymilvus#3432](https://github.com/milvus-io/pymilvus/pull/3432)

Keywords: Upsert, Partial Update, Array, ARRAY_APPEND, ARRAY_REMOVE

Released: v2.7.0 (target)

## Summary

Introduce per-field partial-update operators on `UpsertRequest` so that
Array-typed fields can be mutated incrementally — **append** new elements
or **remove** matching elements — without the caller resending the full
array payload. Phase 1 supports two operators: `ARRAY_APPEND` and
`ARRAY_REMOVE`. `REPLACE` remains the default and matches current
upsert behavior.

## Motivation

Today, mutating one element of an Array column on an existing row requires
the application to:

1. query the row to fetch the current Array value,
2. compute the new array client-side,
3. send the full array back in an Upsert.

This round-trip model has three downsides for real-world workloads:

- **Bandwidth cost** — a 1 KB tag array round-trips on every single-tag
  change, even when the actual delta is one element.
- **Race condition surface** — steps 1-3 are not atomic. Two concurrent
  writers that both want to append to the same row race, and the late
  writer silently overwrites the early writer’s append.
- **Schema-forced read path** — read-only workloads that never need the
  existing tag set are forced to fetch it just to write.

Document stores (MongoDB `$push` / `$pull`) and relational systems
(Postgres `array_append`, `array_remove`) already expose this as a
first-class primitive. Milvus should as well, since Array is a native
element type in the collection schema.

## Public Interfaces

### Proto — `milvus-io/milvus-proto`

Add an enum + message in `schema.proto`:

```proto
message FieldPartialUpdateOp {
  enum OpType {
    REPLACE      = 0;  // default, full overwrite (legacy upsert)
    ARRAY_APPEND = 1;
    ARRAY_REMOVE = 2;

    // Reserved namespaces for future expansion:
    //   10-19  JSON_*
    //   20-29  NUM_*  (increment / decrement, etc.)
    //   30-39  SET_*
    //   40-49  STRING_*
  }

  string field_name = 1;  // references fields_data[*].field_name
  OpType op         = 2;
}
```

Extend `UpsertRequest` in `milvus.proto`:

```proto
message UpsertRequest {
  // ... existing fields ...
  repeated schema.FieldPartialUpdateOp field_ops = 11;
}
```

`FieldData` is **unchanged**. Placing the op on `UpsertRequest` (not on
`FieldData`) keeps `FieldData` a pure data carrier — it continues to
flow through Insert / Query / Search / msgstream without leaking an
upsert-only concept into those paths.

### Go SDK — `client/v2`

```go
opt := milvusclient.NewColumnBasedInsertOption("tbl").
    WithColumns(pkCol, tagsCol).
    WithArrayAppend("tags")            // or WithArrayRemove("tags")
mc.Upsert(ctx, opt)
```

`WithFieldPartialOp(field, op)` is also exposed for advanced callers. A
non-REPLACE op auto-promotes `partial_update=true` so callers do not
have to set both.

### pymilvus

```python
from pymilvus import FieldOp

client.upsert(
    collection_name="tbl",
    data=[{"pk": 1, "tags": [3, 4]}],
    field_ops={"tags": FieldOp.array_append()},
)
```

Accepted forms for each `field_ops` entry: `FieldOp` factory,
`FieldOpType` enum, raw int, or string alias (`"array_append"` /
`"array_remove"` / `"replace"`). `REPLACE` entries are dropped (no-op).

## Design Details

### 1. Semantics

| Op            | Base row          | Payload row    | Resulting row          |
|---------------|-------------------|----------------|------------------------|
| `REPLACE`     | `[a, b, c]`       | `[x, y]`       | `[x, y]`               |
| `ARRAY_APPEND`| `[a, b]`          | `[c, a]`       | `[a, b, c, a]`         |
| `ARRAY_REMOVE`| `[a, b, a, c, b]` | `[a, b]`       | `[c]`                  |

Properties:

- **ARRAY_APPEND preserves duplicates and order.** Base comes first,
  payload is concatenated after.
- **ARRAY_REMOVE removes every match.** Aligns with MongoDB `$pull` and
  Postgres `array_remove`. Base order is preserved for surviving
  elements.
- **Float / Double equality is strict IEEE-754.** `NaN != NaN`, so an
  ARRAY_REMOVE payload containing NaN cannot delete NaN elements from
  the base. This is an intentional language semantics, documented.
- **Per-row capacity enforcement.** Before merge, the proxy checks that
  `base_len + payload_len ≤ max_capacity` for ARRAY_APPEND, then enforces
  again at merge time. ARRAY_REMOVE cannot grow the array and is not
  capped.
- **Auto-promote.** A request that carries at least one non-REPLACE op
  is treated as a partial update even if `partial_update=false`.

### 2. Proxy flow

```
UpsertRequest
    │
    ▼
validateFieldPartialUpdateOps   <-- rejects:
    │                               · empty field_name
    │                               · duplicate field
    │                               · op on PK
    │                               · unknown field
    │                               · non-Array field
    │                               · element-type mismatch
    │                               · ARRAY_APPEND payload > max_capacity
    │                               · nil ArrayData in fields_data
    ▼
partial_update = true (auto)
    │
    ▼
queryPreExecute  ── fetches current rows from segments
    │
    ▼
per-field merge dispatch
    │
    ├─ REPLACE       → UpdateFieldDataByColumn         (legacy path)
    └─ ARRAY_APPEND/ → UpdateArrayFieldByColumnWithOp
       ARRAY_REMOVE    (typeutil, per-element-type)
    │
    ▼
rewrite as DELETE + INSERT  (existing upsert back end)
```

The merged `FieldsData` is then fed through the usual upsert path
(delete-then-insert), which means no changes are required to
DataNode / QueryNode / storage format.

### 3. `typeutil` primitives

Two pure, side-effect-free helpers in `pkg/util/typeutil/schema.go`:

```go
func UpdateArrayFieldByColumnWithOp(
    base, update *schemapb.FieldData,
    baseIndices, updateIndices []int64,
    op schemapb.FieldPartialUpdateOp_OpType,
    maxCapacity int,
) error

func ApplyArrayRowOp(
    base, update *schemapb.ScalarField,
    op schemapb.FieldPartialUpdateOp_OpType,
    elementType schemapb.DataType,
    maxCapacity int,
) (*schemapb.ScalarField, error)
```

Properties:

- REPLACE short-circuits to the legacy `UpdateFieldDataByColumn` path —
  zero overhead for the default case.
- Every element type (`Bool`, `Int8/16/32`, `Int64`, `Float`, `Double`,
  `VarChar`, `String`) has a dedicated code path.
- `-1` is the sentinel "no capacity gate", consistent between pre-check
  and merge.
- `ValidData` is synced after a successful merge so a null base row that
  receives a valid payload flips to `valid=true`. A null upsert payload
  (`update.ValidData[i]=false`) is a no-op for that row.

### 4. Forward / backward compatibility

- **Old server, new client** — the server sees `field_ops` as an unknown
  proto3 field and drops it silently. Behavior degrades to REPLACE. The
  SDK logs a one-time warning so users know their op was not applied.
- **New server, old client** — `field_ops` stays empty, behavior is
  identical to legacy upsert. Zero impact.
- **FieldData path unchanged** — because the op lives on
  `UpsertRequest`, anything that consumes `FieldData` (Insert,
  SearchResult, QueryResults, msgstream envelopes) is untouched.

### 5. Concurrency

The merge is **last-write-wins at the row level** under the existing
upsert concurrency model. Two concurrent ARRAY_APPEND requests targeting
the same row do not interleave at the element level; one of them sees
the pre-state produced by the other. This is the same guarantee that
today's plain upserts provide, and is sufficient for the motivating
single-writer-per-row workloads.

A future enhancement may introduce a lock-per-row or CAS primitive for
element-level atomicity, but that is out of scope for Phase 1.

### 6. Limitations (Phase 1)

- Array only. JSON / Set / numeric-increment operators are deliberately
  deferred — their enum ranges are reserved in the proto to prevent
  namespace collisions.
- Scalar types inside the Array follow Go-native equality (`==`). There
  is no user-defined comparator.
- `ARRAY_REMOVE` is O(base_len × payload_len) for Float / Double (to
  avoid NaN ambiguity in hash-based lookup) and O(base_len + payload_len)
  for hashable types. Acceptable because typical Array lengths are bound
  by `max_capacity` ≤ 1024.
- No `ARRAY_INSERT(index, value)` / `ARRAY_SET(index, value)` — those
  require the client to carry stable indices which is not a first-class
  concept in Milvus Arrays yet.

## Test Plan

- **typeutil unit tests** — 38 cases covering every element type plus
  edge cases (empty base, empty update, over-capacity, NaN semantics,
  nullable rows, nil ArrayData).
- **proxy unit tests** — 20+ cases covering every validate gate plus
  auto-promote logic and capacity pre-check.
- **SDK unit tests** — builder state machines for both row-based and
  column-based upsert, including REPLACE-clears-prior, unknown-field
  forwarding, and multi-field emission.
- **pymilvus unit tests** — 21 cases over `FieldOp` factories,
  `normalize_field_ops`, `Prepare.*` wiring, and proto regression
  guards.
- **Go SDK E2E** — live-cluster tests under
  `tests/go_client/testcases/upsert_array_partial_op_test.go` covering
  append, remove (including no-match rows), capacity overflow, and
  auto-promote.

## Rejected Alternatives

- **Attach op on `FieldData`.** Would pollute Insert / Query / Search /
  msgstream paths that also traffic in `FieldData`. Keeping the op on
  `UpsertRequest` preserves `FieldData` as a pure data carrier.
- **Overload `partial_update` bool to encode op.** Not extensible to
  future op types.
- **Expression-based update (`"tags = array_append(tags, 3)"`).** Much
  larger scope (parser + planner). Phase 1 keeps the surface minimal.
- **Remove only the first match.** Diverges from MongoDB `$pull` and
  Postgres `array_remove`. Remove-all is the more common request.
