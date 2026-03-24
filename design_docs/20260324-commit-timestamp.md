# MEP: commit_timestamp for Correct MVCC/TTL/GC on Import and CDC Segments

- **Created:** 2026-03-24
- **Author(s):** @bigsheeper
- **Status:** Under Review
- **Component:** DataCoord | QueryNode | DataNode | Storage
- **Related Issues:** [milvus-io/milvus#48471](https://github.com/milvus-io/milvus/issues/48471)
- **Released:** TBD

## Summary

Add a `commit_timestamp` field to `SegmentInfo` (DataCoord) and `SegmentLoadInfo` (QueryCoord → QueryNode → C++ segcore). When non-zero, `commit_timestamp` is the effective transaction time for the segment, overriding `start_position.Timestamp` / `dml_position.Timestamp` for all temporal decisions: MVCC snapshot visibility, GC eligibility, TTL compaction triggering, and delete-buffer anchoring.

## Motivation

When Milvus imports a bulk-load batch (or replays CDC), each row carries its original `row.timestamp` from the source system — which may be hours, days, or years old. Every time-based check in the system uses `start_position.Timestamp` or `dml_position.Timestamp` (derived from these old row timestamps), seeing `T_old` instead of the actual commit time `T_commit`. This produces a family of correctness bugs:

| Bug | Root cause |
|-----|-----------|
| Snapshot over-inclusion | `GenSnapshot` filters by `start_position.Timestamp`; import segment with `T_old=1000` appears at `snapshotTs=3000` before it was logically committed at `T_commit=5000` |
| Premature L0 compaction | L0 target selection uses `start_position.Timestamp`; import segment is eligible for L0 runs before `T_commit` |
| Wrong TTL trigger | TTL compaction trigger uses `binlog.TimestampFrom/TimestampTo`; import binlogs with old row timestamps look expired immediately after `isImporting` is cleared |
| Wrong TTL query masking | C++ `mask_with_timestamps` uses raw `row.ts` for TTL; import rows with `row.ts = T_old` are falsely expired in live queries |
| Wrong MVCC visibility | C++ `mask_with_timestamps` uses raw `row.ts` for MVCC; a query at `mvcc_ts = T_mid` (where `T_old < T_mid < T_commit`) can see import rows that should be invisible until `T_commit` |
| Premature GC | GC eligibility uses `dml_position.Timestamp`; recently committed import segment with old `dml_position` may be garbage-collected |
| Wrong channel truncation | `TruncateChannelByTime` drops segments by `dml_position.Timestamp`; import segment dropped prematurely |
| Delete leakage | Delete-buffer anchor uses `start_position.Timestamp = T_old`; deletes issued between `T_old` and `T_commit` may be incorrectly applied to import rows |

## Public Interfaces

### Proto field additions

**`pkg/proto/data_coord.proto` — `SegmentInfo`:**
```protobuf
// commit_timestamp is the effective transaction timestamp for import segments.
// When non-zero, overrides start_position.Timestamp / dml_position.Timestamp
// for all temporal decisions. Zero = normal segment. Cleared to 0 on compaction.
uint64 commit_timestamp = 35;
```

**`pkg/proto/query_coord.proto` — `SegmentLoadInfo`:**
```protobuf
// commit_timestamp mirrors data_coord.SegmentInfo.commit_timestamp.
// Used by QueryNode for delete-buffer pinning and ListAfter calls.
uint64 commit_timestamp = 25;
```

**`pkg/proto/segcore.proto` — `SegmentLoadInfo`:**
```protobuf
// commit_timestamp propagated from QueryCoord to C++ segcore.
// Used in LoadFieldData to overwrite the timestamp column for import segments.
uint64 commit_timestamp = 22;
```

### New C API

```c
// Sets commit_ts_ on a sealed segment before LoadFieldData is called.
// Must be called BEFORE loading field data.
CStatus
SetCommitTimestamp(CSegmentInterface c_segment, uint64_t commit_ts);
```

## Design Details

### Core Idea

An import batch is an atomic transaction. From the collection's perspective, every row in the batch came into existence at `T_commit` — the moment `isImporting` was cleared. `commit_timestamp` stores this value on the segment and acts as the authoritative time for all temporal decisions when non-zero.

Two helper functions in DataCoord encapsulate the override:

```go
// segmentEffectiveTs returns start_position.Timestamp, or commit_timestamp when non-zero.
func segmentEffectiveTs(seg *datapb.SegmentInfo) uint64 {
    if ts := seg.GetCommitTimestamp(); ts != 0 {
        return ts
    }
    return seg.GetStartPosition().GetTimestamp()
}

// segmentEffectiveDmlTs returns dml_position.Timestamp, or commit_timestamp when non-zero.
func segmentEffectiveDmlTs(seg *datapb.SegmentInfo) uint64 {
    if ts := seg.GetCommitTimestamp(); ts != 0 {
        return ts
    }
    return seg.GetDmlPosition().GetTimestamp()
}
```

### When `commit_timestamp` Is Set

`commit_timestamp` is set **atomically** with `isImporting = false` in `import_checker.go:unsetSegmentImporting()`. Both fields are written to the same `SegmentInfo` proto key in a single `MultiSave` etcd transaction. There is no window where `isImporting=false` but `commit_timestamp=0` can be observed by any observer.

```go
nowTs, _ := c.alloc.AllocTimestamp(c.ctx)
c.meta.UpdateSegmentsInfo(c.ctx,
    UpdateIsImporting(segmentID, false),
    UpdateCommitTimestamp(segmentID, uint64(nowTs)),
)
```

### When `commit_timestamp` Is Cleared

After major compaction rewrites all row timestamps, the output segment has `commit_timestamp = 0` (proto zero value). No explicit clearing is needed.

### DataCoord Fix Sites

| File | Site | Old | Fix |
|------|------|-----|-----|
| `handler.go` | `GenSnapshot` filter | `info.GetStartPosition().GetTimestamp()` | `segmentEffectiveTs(info.SegmentInfo)` |
| `compaction_task_l0.go` | L0 target selection | `info.GetStartPosition().GetTimestamp()` | `segmentEffectiveTs(info.SegmentInfo)` |
| `meta.go` | `TruncateChannelByTime` | `segment.GetDmlPosition().GetTimestamp()` | `segmentEffectiveDmlTs(segment.SegmentInfo)` |
| `garbage_collector.go` | GC eligibility | `segment.GetDmlPosition().GetTimestamp()` | `segmentEffectiveDmlTs(segment.SegmentInfo)` |
| `compaction_trigger.go` | TTL trigger | `binlog.TimestampTo / TimestampFrom` | `max(binlogTs, commit_ts)` for both fields |

### QueryNode Fix Sites

A local helper mirrors the DataCoord pattern:

```go
func segmentEffectiveTs(info *querypb.SegmentLoadInfo) uint64 {
    if ts := info.GetCommitTimestamp(); ts != 0 {
        return ts
    }
    return info.GetStartPosition().GetTimestamp()
}
```

Applied at 4 call sites in `delegator_data.go`:
- `deleteBuffer.Pin(segmentEffectiveTs(info), ...)` — anchor point for streaming deletes
- `deleteBuffer.Unpin(segmentEffectiveTs(info), ...)` — symmetric unpin on segment release
- `deleteBuffer.ListAfter(segmentEffectiveTs(info))` — replay deletes since commit time
- `catchUpTs = segmentEffectiveTs(info)` — snapshot catch-up for empty-snapshot case

### C++ Segcore: Approach A (Timestamp Column Overwrite at Load Time)

**Rationale:** Overwriting `row.ts → commit_ts` during `LoadFieldData` makes the existing `mask_with_timestamps` (MVCC + TTL) and delete filtering work correctly without any modification to the query hot path.

**Implementation:** In `ChunkedSegmentSealedImpl`, `commit_ts_` is set from the deserialized proto in `SetLoadInfo`. Two loading paths (`load_system_field_internal` for storage v1, `load_column_group_data_internal` for storage v2) overwrite the timestamp vector with `std::fill(commit_ts_)` when `commit_ts_ != 0`:

```cpp
// Storage v1 path (load_system_field_internal):
if (commit_ts_ != 0) {
    std::fill(timestamps.begin(), timestamps.end(), commit_ts_);
}
init_timestamp_index_owned(std::move(timestamps), num_rows);

// Storage v2 path (load_column_group_data_internal):
if (commit_ts_ != 0) {
    std::vector<Timestamp> ts(num_rows, commit_ts_);
    init_timestamp_index_owned(std::move(ts), num_rows);
} else {
    auto col = get_column(TimestampFieldID);
    init_timestamp_index_from_column(col, num_rows);
}
```

**Effect on query correctness** (with all `row.ts = commit_ts`):

| Check | Condition | Result |
|-------|-----------|--------|
| MVCC visibility | `row.ts > mvcc_ts` | Invisible for `mvcc_ts < commit_ts`, visible for `mvcc_ts ≥ commit_ts` ✓ |
| TTL expiry | `row.ts ≤ collection_ttl` | Uses `commit_ts` as logical age ✓ |
| Delete (pre-commit, query_ts < commit_ts) | row invisible by MVCC | Delete irrelevant ✓ |
| Delete (post-commit, query_ts ≥ commit_ts) | row visible | Delete applied by timestamp ✓ |

**Trade-off:** Original `row.ts` values are no longer visible in the in-memory segment. They remain intact in on-disk binlogs. After compaction, `commit_ts` is baked as the physical row timestamp — semantically correct for a non-import segment.

**`SegmentGrowingImpl` is unaffected:** Growing segments receive rows from live DML where `row.ts ≈ current_timetick`. The `T_old ≪ T_commit` gap cannot occur.

### Propagation Chain

```
DataCoord:SegmentInfo.commit_timestamp
  → PackSegmentLoadInfo (querycoordv2/utils/types.go)
  → querypb.SegmentLoadInfo.commit_timestamp
  → ConvertToSegcoreSegmentLoadInfo (util/segcore/segment.go)
  → segcorepb.SegmentLoadInfo.commit_timestamp
  → NewSegmentWithLoadInfo (C API)
  → ChunkedSegmentSealedImpl::SetLoadInfo → commit_ts_
  → LoadFieldData: std::fill(timestamps, commit_ts_)
```

### Sites Confirmed Safe (No Change Needed)

| Site | Reason |
|------|--------|
| `handler.go:196` deleteCheckPoint | L0 segments only; conservative (earlier) checkpoint is safe |
| `meta.go:2219` `GetEarliestStartPositionOfGrowingSegments` | Growing segments only |
| `segment_allocation_policy.go` | Growing-segment sealing logic; import segments already sealed |
| `compaction_task_mix.go`, `compaction_task_clustering.go` | Neither uses `start_position.Timestamp` for selection |
| `delegator_data.go:854` `zap.Time` log field | Diagnostic output only |
| `compaction_trigger.go` `ShouldCompactExpiryWithTTLField` | Uses `GetExpirQuantiles()` (physical wall-clock); returns false for import segments which have no quantiles |

## Compatibility, Deprecation, and Migration Plan

- **Backward compatible:** `commit_timestamp` is a new proto field with default value 0. All existing non-import segments have `commit_timestamp = 0`, and all code paths fall back to the original `start_position.Timestamp` / `dml_position.Timestamp` logic. Behavior for normal segments is identical to pre-change.
- **Rolling upgrade safe:** The field is optional. A coordinator running the new code alongside a node running the old code (or vice versa) simply ignores the field — old nodes see `commit_timestamp = 0` and use existing logic unchanged.
- **No migration required:** Existing import segments already completed before this change will have `commit_timestamp = 0`. They benefit from this fix only for new import operations after the upgrade.

## Test Plan

**Unit tests (Go):**
- `TestSegmentEffectiveTs_*` / `TestSegmentEffectiveDmlTs_*` — helper function correctness for import and normal segments
- `TestUpdateCommitTimestamp_*` — meta operator sets field correctly
- `TestGenSnapshot_ImportSegment_*` — excluded before commit_ts, included after
- `TestDeleteBuffer_PinsAtCommitTs` — delete buffer anchor uses commit_ts, not start_position.ts
- `Test_compactionTrigger_shouldDoSingleCompaction_CommitTimestamp` — TTL trigger uses commit_ts for import segments
- `TestUnsetSegmentImporting_SetsCommitTimestampAtomically` — atomic set verification

**C++ tests (`test_commit_timestamp.cpp`):**
- `MVCC_RowsInvisibleBeforeCommitTs` — queries at `ts < commit_ts` see 0 rows; queries at `ts ≥ commit_ts` see all rows
- `TTL_RowsNotExpiredWhenCommitTsAboveTtl` — import segment not TTL-expired when `commit_ts > ttl_threshold`; control (no overwrite) correctly expires
- `Delete_PreCommitDeleteAppliedAfterCommit` — MVCC masks rows before commit_ts; delete applies correctly after commit_ts
- `NormalSegment_BehaviorUnchanged` — segments with `commit_ts=0` behave identically to pre-change

**Integration tests (`CommitTimestampSuite`):**
- `TestImport_CommitTimestampSetAfterCompletion` — after bulk-insert import completes, all segments have `CommitTimestamp > 0`
- `TestImport_DataQueryableAfterCommit` — Strong-consistency query returns all imported rows after import

## Rejected Alternatives

### Approach B: Overlay in `mask_with_timestamps`

Instead of overwriting the timestamp column at load time, apply a per-row `max(row.ts, commit_ts)` in the `mask_with_timestamps` hot path:

```cpp
auto effective_ts = (commit_ts_ != 0) ? std::max(val, commit_ts_) : val;
mask[i] = effective_ts > timestamp;
```

**Rejected because:**
- Adds a branch + conditional max to every row in every query — measurable hot-path overhead.
- Requires changes in both the MVCC lambda and the TTL lambda, increasing the blast radius.
- Still does not fix the DataCoord-side bugs (snapshot, GC, compaction trigger) — those need the `segmentEffectiveTs` helpers regardless.
- The load-time overwrite (Approach A) achieves the same correctness with zero query hot-path overhead and a single implementation point.

### Approach C: Dual-timestamp segment (store both `row.ts` and `commit_ts`)

Keep original `row.ts` intact; thread `commit_ts` as a separate per-segment value through all query logic.

**Rejected because:**
- Every temporal decision site needs to be aware of two timestamps and pick the right one — far larger diff, higher risk of missed sites.
- No query semantics require access to the original `row.ts` after commit. The in-memory value is transient.
- Original `row.ts` is preserved on disk in binlogs if needed for future use.

## References

- Implementation PR: [milvus-io/milvus#48472](https://github.com/milvus-io/milvus/pull/48472)
- Tracking issue: [milvus-io/milvus#48471](https://github.com/milvus-io/milvus/issues/48471)
