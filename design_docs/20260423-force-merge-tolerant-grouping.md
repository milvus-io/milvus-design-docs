# MEP: Force-Merge Tolerant Grouping — Parallel Compaction Tasks for Large Collections

Current state: In Progress

ISSUE: [feat: force-merge tolerant grouping to enable parallel compaction tasks #49280](https://github.com/milvus-io/milvus/issues/49280)

Keywords: Compaction, ForceMerge, DataCoord, Parallelism, DP

Released: N/A

## Summary

On production workloads, `ManualCompaction` with force-merge enabled collapses into one
very large compaction task even when a collection has enough segments and data volume to
benefit from parallel execution.

Observed case (collection `465308834425238548`): 21 L1 segments, total memory size
**55.61 GB**, force-merge target **4 GB**. Current planner emits **1 task** that ingests
all 55.61 GB and produces 14 outputs.

The root cause is a degenerate DP objective in `maxFullSegmentsGrouping`:

1. `hasTail` threshold at `0.01` **bytes** (line 289 of `compaction_view_forcemerge.go`)
   is absurdly tight — on real inputs the remainder is never below this, so every group
   is scored as having `tails=1`. Once `tails` is a constant across all candidates, the
   tiebreak order `more full > fewer tails > more groups` degenerates to
   `more full > fewer groups` — the opposite of the parallelism goal.
2. The grouping function in `largerGroupingSegments` also uses a misnamed local constant
   `defaultToleranceMB = 0.05` which is actually a ratio, not megabytes.

Consequences: no cluster-level parallelism, large retry blast radius, high per-task
memory pressure.

## Goals

Produce force-merge plans that:
1. Use multiple tasks when total input is much larger than target size.
2. Keep total output count close to `ceil(total/target)`.
3. Keep each output close to target size (within configurable tolerance).
4. Bound per-task input size via a configurable size multiplier.
5. Keep behavior configurable without API/proto changes.

## Non-Goals

- Adding new fields to `ManualCompactionRequest` proto or any client-facing API.
- Non-contiguous bin-packing algorithms (contiguous partitioning is sufficient and
  validated by the 21-segment reference workload).
- Changes to clustering, level-zero, or sort compaction policies.
- Any change to `MultiSegmentWriter` or datanode writer behavior.

## Design

### Algorithm: Tail-Quality DP with Input-Size Cap

Replace `maxFullSegmentsGrouping`'s degenerate objective with a new
`tolerantGroupingDP` function that scores each group candidate by tail quality:

```
For a candidate group of size G with target T:
  N = ceil(G / T)                      # number of output segments
  r = (G mod T) / T                    # tail fill ratio in [0, 1)

  if N == 1 or r == 0:
    loss = 0                           # one-output group or perfectly full tail
  elif r >= (1 - alpha):
    loss = 0                           # tail is within tolerance alpha of full
  else:
    loss = (1 - r) / N                 # penalize poorly-filled tails, amortized over N
```

Where `alpha` = `dataCoord.compaction.forceMerge.segmentSizeToleranceRatio` (default
`0.05`, clamped to `[0.0, 0.20]`).

DP objective: **minimize total loss** across groups; tiebreak with **more groups**
(parallelism).

Hard cap: a group is rejected if `groupSize > maxInputSizeMultiplier × targetSize`,
where `maxInputSizeMultiplier` = `dataCoord.compaction.forceMerge.maxInputSizeMultiplier`
(default `4.0`, clamped to `[1.0, 20.0]`). This prevents a single task from ingesting
unbounded input data. When total collection size exceeds cap × target (which triggers
splitting), the cap ensures the DP explores multi-group splits.

Contiguous partitioning: segments are grouped in their input order — no bin-packing
reorder. This is validated as sufficient by the 21-segment reference workload.

### Integration: Replace `maxFullSegmentsGrouping` in `adaptiveGroupSegments`

Current call chain:
```
ForceTriggerAll → adaptiveGroupSegments → maxFullSegmentsGrouping  (n ≤ threshold)
                                        → largerGroupingSegments   (n > threshold)
```

After this change:
```
ForceTriggerAll → adaptiveGroupSegments → tolerantGroupingDP  (n ≤ threshold)
                                        → largerGroupingSegments  (n > threshold)
```

Only `maxFullSegmentsGrouping` is replaced. `largerGroupingSegments` (large-n O(n) path)
is unchanged. Blast radius is bounded to the small-n compaction path.

### Configuration

Two new `DataCoordCfg` parameters:

| Config Key | Go Field Name | Default | Clamp Range | Semantics |
|---|---|---|---|---|
| `dataCoord.compaction.forceMerge.segmentSizeToleranceRatio` | `CompactionForceMergeSegmentSizeToleranceRatio` | `0.05` | `[0.0, 0.20]` | Fraction of target size by which a tail output may be under-full without penalty |
| `dataCoord.compaction.forceMerge.maxInputSizeMultiplier` | `CompactionForceMergeMaxInputSizeMultiplier` | `4.0` | `[1.0, 20.0]` | Maximum per-task input size as a multiple of target size |

Note: `segmentSizeToleranceRatio` default is `0.05` — the same numeric value as the
old `defaultToleranceMB` constant, but the semantics differ entirely. The old constant
was a break threshold in `largerGroupingSegments`; the new parameter is a dead-zone
width in the DP loss function.

### Naming Fix

Rename the local constant `defaultToleranceMB` to `defaultLargerGroupingBreakRatio` to
clarify that it is a ratio used as a break threshold in `largerGroupingSegments`, not a
megabyte value.

### Planner/Writer Semantics (Unchanged)

`CompactionTask.MaxSize` (proto field 26, `max_size`) already carries per-task planned
output segment size from the planner to datanode. `MultiSegmentWriter` already uses it
as the rotation size. No writer behavior change is required.

### Observability

Per force-merge plan (`INFO` level):
- collection / partition / channel
- total input size, target size
- tolerance ratio (α) and config key, max input size multiplier
- task count and output count summary

Per task (`DEBUG` level):
- input segment count, input size, planned output count, planned segment size, deviation

## Affected Files

### `pkg/util/paramtable/component_param.go`
- Add `CompactionForceMergeSegmentSizeToleranceRatio` ParamItem (float64, key
  `dataCoord.compaction.forceMerge.segmentSizeToleranceRatio`, default `0.05`).
- Add `CompactionForceMergeMaxInputSizeMultiplier` ParamItem (float64, key
  `dataCoord.compaction.forceMerge.maxInputSizeMultiplier`, default `4.0`).

### `configs/milvus.yaml`
- Add `dataCoord.compaction.forceMerge.segmentSizeToleranceRatio: 0.05` under the
  `compaction:` section.
- Add `dataCoord.compaction.forceMerge.maxInputSizeMultiplier: 4.0` under the
  `compaction:` section.

### `internal/datacoord/compaction_view_forcemerge.go`
- Add `tolerantGroupingDP(segments []*SegmentView, targetSize, toleranceRatio,
  maxInputSizeMultiplier float64) [][]*SegmentView` — new DP grouping function.
- Replace `maxFullSegmentsGrouping` call in `adaptiveGroupSegments` with
  `tolerantGroupingDP`.
- Rename `defaultToleranceMB` → `defaultLargerGroupingBreakRatio` with an explanatory
  comment.
- Pass config values from `paramtable.Get().DataCoordCfg` into `tolerantGroupingDP`.
- Add structured log lines (INFO per plan, DEBUG per task).

### `internal/datacoord/compaction_view_forcemerge_test.go`
- Add `TestForceMerge_Real21_TolerantDP`: 21-segment / 55.61 GB reference case asserts
  `numGroups > 1`.
- Update `TestForceMerge_Real21_MaxFull` and `TestForceMerge_Real21_Adaptive` (once
  added) to assert `numGroups > 1` after the fix.
- Add `TestForceMerge_ToleranceRatioConfig`: set `segmentSizeToleranceRatio` to two
  different values via `paramtable.Save/Reset`, verify groupings differ on a
  deterministic input.
- Add `TestForceMerge_MaxInputSizeMultiplierCap`: verify that groups never exceed
  `maxInputSizeMultiplier × targetSize`.
- Add edge cases: single segment, all segments fit in one group within cap, total size
  exactly at boundary.

## Not Touched

- `internal/mocks/` — auto-generated, not hand-edited
- `mock_*.go` — auto-generated, not hand-edited
- `pkg/proto/*.pb.go` — auto-generated, not hand-edited
- `internal/proto/*.pb.go` — auto-generated, not hand-edited
- `milvus-proto` submodule — no external proto changes
- `milvuspb.ManualCompactionRequest` — no client API changes
- `internal/datacoord/compaction_view.go` — `GetBinlogMemorySizeAsBytes` rename is
  already complete in current code; no further changes needed

## Validation

Reference workload: 21 L1 segments, 55.61 GB total, 4 GB target.

| Metric | Before | After |
|---|---|---|
| Number of compaction tasks | 1 | ≥ 2 (parallelism) |
| Total output segment count | 14 | 14 (unchanged) |
| Max per-task input size | 55.61 GB | ≤ 4.0 × 4 GB = 16 GB |

## Test Plan

### Unit Tests
- Reference 21-segment workload: `numGroups > 1`, output count equals theoretical minimum.
- Config-driven tolerance: two values produce different groupings on same deterministic input.
- Input-size cap: no group exceeds cap × target.
- Edge cases: single segment, empty, all fit in one group.

### Integration Tests
N/A — this change is internal to the force-merge planner; no new RPC surface.

## References

- GitHub Issue: https://github.com/milvus-io/milvus/issues/49280
- `internal/datacoord/compaction_view_forcemerge.go`
- `internal/datacoord/compaction_view_forcemerge_test.go`
