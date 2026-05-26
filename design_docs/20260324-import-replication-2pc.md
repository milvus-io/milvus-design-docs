# MEP: Import Two-Phase Commit for Primary/Secondary Replication

- **Created:** 2026-03-24
- **Author(s):** @bigsheeper
- **Status:** Under Review
- **Component:** DataCoord | StreamingNode | Proxy
- **Related Issues:** [milvus-io/milvus#48525](https://github.com/milvus-io/milvus/issues/48525)
- **Released:** TBD

## Summary

Add Two-Phase Commit (2PC) support to Import so that bulk-loaded data becomes visible only after a per-vchannel WAL commit fence, consistently across all clusters in a primary/secondary replication setup (GlobalCluster / disaster recovery). Data stays invisible until an explicit commit signal is delivered via WAL, ensuring primary and secondary clusters reach the same visible state at the same logical WAL position.

## Motivation

Milvus 2.6 supports primary/secondary disaster recovery and cross-region replication (GlobalCluster). Currently, Import operations are rejected when replication is enabled, because there is no way to ensure imported data becomes visible at the same logical time on all clusters.

The core problem is a **timing conflict between DML and Import**: a DELETE issued on the primary after Import starts but before it completes may arrive on the secondary at a different logical time relative to when the import data becomes visible — resulting in primary/secondary divergence.

The solution is to hold import data invisible until an explicit commit signal (`CommitImportMessage`) is delivered via WAL. Since CDC replicates all WAL messages to secondary clusters verbatim, every cluster processes the commit at the same logical position in the stream, eliminating the divergence window.

## Public Interfaces

### New RESTful Endpoints

| Method | Path | Body | Description |
|--------|------|------|-------------|
| POST | `/v2/vectordb/jobs/import/commit` | `{"jobID": "123"}` | Commit an Uncommitted import job |
| POST | `/v2/vectordb/jobs/import/abort` | `{"jobID": "123"}` | Abort a non-terminal import job |

These endpoints are **RESTful only** — not added to the public gRPC `MilvusService`.

`GetImportProgress` now surfaces two new states: `Uncommitted` and `Committing`.

### New `auto_commit` Import Option

```
options: [{"key": "auto_commit", "value": "false"}]
```

- Default `true` — existing behavior; ImportChecker auto-commits when the job reaches `Uncommitted`
- `false` — platform controls commit timing; used by replication clusters

### New ImportJobState Values

```protobuf
enum ImportJobState {
  // ...existing values (None=0 through Sorting=7)...
  Uncommitted = 8;  // data ingested + indexed, invisible, awaiting commit
  Committing  = 9;  // CommitImportMessage written to WAL, waiting for vchannels
}
```

State machine:
```
Pending → PreImporting → Importing → Sorting → IndexBuilding → Uncommitted → Committing → Completed
                                                                    ↓
                                                                  Failed (any stage)
```

### New WAL Message Types

```protobuf
enum MessageType {
  // ...existing values through DropSnapshotsByCollection=44...
  CommitImport   = 45;
  RollbackImport = 46;
}

message CommitImportMessageHeader {
    int64 collection_id = 1;
    int64 job_id        = 2;
}

message RollbackImportMessageHeader {
    int64 collection_id = 1;
    int64 job_id        = 2;
}
```

### New DataCoord RPCs (internal only)

```protobuf
service DataCoord {
    rpc CommitImport(CommitImportRequest)             returns (common.Status) {}
    rpc AbortImport(AbortImportRequest)               returns (common.Status) {}
    rpc HandleCommitVchannel(HandleCommitVchannelRequest) returns (common.Status) {}
}
```

## Design Details

### Segment Visibility

Segment visibility is controlled by the existing `is_importing` flag:
- `is_importing = true` — data invisible (set at segment creation, unchanged through `Uncommitted`)
- `is_importing = false` — data visible (set by `HandleCommitVchannel` after `CommitImportMessage` consumed)

No new visibility mechanism is introduced.

Visibility is committed per vchannel, not as a single job-wide metadata flip. `HandleCommitVchannel` first clears `is_importing=false` for the segments on that vchannel, then records the vchannel in `ImportJob.committed_vchannels`. Therefore, a vchannel's imported data can be visible while the job is still `Committing`; the job reaches `Completed` only after all vchannels have been recorded. This ordering is intentional because the commit fence is consumed independently on each vchannel.

### CommitImport Flow

**Phase 1 — RPC → WAL broadcast (DataCoord)**

```
Platform
  │  POST /v2/vectordb/jobs/import/commit {"jobID": "123"}
  ▼
Proxy (converts "123" → int64(123))
  → DataCoord.CommitImport(job_id=123)
       │
       ├─ Validate: job state == Uncommitted; Committing/Completed are idempotent success
       ├─ Reject manual commit if job.auto_commit == true
       ├─ Acquire per-job KeyLock keyed by job_id
       ├─ Re-read and re-validate job state
       ├─ Broadcast CommitImportMessage{collection_id, job_id} to job.vchannels (WAL)
       └─ Release per-job KeyLock
```

**Phase 2 — DDL ack callback (DataCoord, fires once for all vchannels)**

The DDL broadcast ack callback fires once when the message has been successfully written to all vchannels' WALs.

```
DDL ack callback:
  if job.state == Uncommitted:
      UpdateJobState(Committing)
  else:
      no-op
```

No compare-and-swap is needed here. `CommitImport` and `RollbackImport` messages are exclusive collection-level broadcast messages, so the broadcaster resource-key lock serializes their ack callbacks for the same collection. The state guard above determines the winner when commit and abort race.

**Phase 3 — Per-vchannel processing (StreamingNode WAL flusher)**

Each vchannel's StreamingNode intercepts `CommitImportMessage` in `wal_flusher.dispatch()`:

```
case CommitImportMessage:
  1. wbMgr.FlushChannel(channel, msg.TimeTick())  // trigger async DML flush
  2. DataCoord.HandleCommitVchannel(job_id, vchannel)
```

**Phase 4 — HandleCommitVchannel (DataCoord, idempotent)**

```
HandleCommitVchannel(job_id, vchannel):
  if vchannel in job.committed_vchannels:
    no-op (idempotent — handles WAL replay on SN restart)

  - set is_importing=false for all segments in this vchannel
  - append vchannel to committed_vchannels (persist to etcd)
```

The segment visibility update happens before the job metadata update. If the visibility callback fails, the vchannel is not recorded and WAL replay retries the same commit. If the job metadata write fails after visibility has been cleared, the retry runs the idempotent visibility callback again and then records the vchannel.

**Phase 5 — ImportChecker (background, ticker)**

```go
case ImportJobState_Committing:
    if len(job.CommittedVchannels) == len(job.Vchannels) {
        updateJobState(ImportJobState_Completed)
    }
    // else: wait for remaining vchannels (WAL delivery guaranteed)
```

### AbortImport Flow

`AbortImport` follows the same broadcast pattern and broadcasts `RollbackImportMessage` to `job.vchannels`. The DDL ack callback:
1. If the job is not `Committing`, `Completed`, or already `Failed`, call `UpdateJobState(Failed)`
2. No-op if the job is already `Committing`, `Completed`, or `Failed`

`UpdateJobState(Failed)` is the single place that releases the requested disk quota (`RequestedDiskSize = 0`) and sets `CleanupTs = now + retention` for GC eligibility. Import segment cleanup remains on the existing failed-job cleanup path: `ImportChecker` marks this job's import tasks as failed, then `ImportInspector.processFailed` marks the task's import segments as `SegmentState_Dropped` and clears segment IDs from the task metadata. Keeping segment cleanup in the inspector avoids a second segment-drop path in the rollback ack callback.

A **no-op handler** is registered for `RollbackImport = 46` in `wal_flusher.dispatch()` to prevent unknown-message errors as flowgraph consumes WAL messages.

### auto_commit Checker

The existing `ImportChecker` `ticker1` loop gains two new cases:

```go
case ImportJobState_Uncommitted:
    if job.GetAutoCommit() {
        commitImport(ctx, job)  // same code path as CommitImport RPC
    }
    // else: wait for explicit CommitImport from platform

case ImportJobState_Committing:
    if len(job.CommittedVchannels) == len(job.Vchannels) {
        updateJobState(ImportJobState_Completed)
    }
```

`checkUncommittedJob` may be invoked more than once while an auto-commit broadcast is in flight. This is safe: the broadcaster's exclusive collection resource key serializes overlapping commit/rollback broadcasts, the ack callbacks are state-guarded, and `HandleCommitVchannel` is idempotent through `committed_vchannels`.

### Race Safety

| Scenario | Protection |
|----------|------------|
| Concurrent CommitImport + AbortImport RPCs | Per-job KeyLock serializes each job's RPC handler; broadcaster exclusive collection resource key serializes ack callbacks |
| Commit ack fires before abort ack | Commit ack moves `Uncommitted → Committing`; abort ack sees `Committing` → no-op |
| Abort ack fires before commit ack | Abort ack moves job to `Failed`; commit ack sees non-`Uncommitted` → no-op |
| Duplicate `CommitImportMessage` (WAL replay on SN restart) | `committed_vchannels` guard in `HandleCommitVchannel` → idempotent no-op |
| Duplicate auto-commit broadcasts before ack | Resource-key lock + ack state guard make duplicates harmless; only the first successful commit ack moves the job to `Committing` |

### Platform Workflow (Replication Clusters)

1. Copy import files to all clusters' object storage
2. Call `ImportV2` on **primary** with `auto_commit=false`; CDC replicates `ImportMessage` to secondaries
3. Poll `GetImportProgress` on **all** clusters until all report `Uncommitted`
4. Call `POST /v2/vectordb/jobs/import/commit` on **primary**; `CommitImportMessage` written to primary WAL, CDC replicates to secondary WALs; all clusters process independently
5. Poll until all clusters report `Completed`

**Abort conditions**: call `POST /v2/vectordb/jobs/import/abort` on each cluster independently if any cluster's job fails, times out before reaching `Uncommitted`, or user requests cancellation. Once `Committing` is reached, abort is no longer possible.

### Implementation Notes

- `CommitImport`/`RollbackImport` messages are handled in `wal_flusher.dispatch()` (same pattern as `CreateCollection`/`DropCollection`), not in `flow_graph_dd_node.go`. This avoids requiring additions to the external milvus-proto `commonpb.MsgType`.
- Commit and rollback messages must be broadcast to the job's data vchannels, not only to the WAL control channel. A control-channel-only, non-pchannel-level message is dropped by the WAL flusher before the `CommitImport`/`RollbackImport` cases can run, so it cannot drive per-vchannel commit handling.
- No timeout is applied to the `Committing` state. WAL delivery is guaranteed; upon StreamingNode restart the WAL replays `CommitImportMessage` and `HandleCommitVchannel` idempotency handles the duplicate.
- `auto_commit` is parsed from `options` KV at job creation time and stored as `bool auto_commit` on the `ImportJob` proto (not re-parsed at check time).

## Compatibility, Deprecation, and Migration Plan

| Scenario | Behavior |
|----------|----------|
| Non-replication cluster, no `auto_commit` option | `auto_commit=true` by default; Checker auto-commits → identical to existing externally-visible behavior |
| Non-replication cluster, `auto_commit=false` | Platform controls commit timing; useful for multi-job atomic visibility |
| Rolling upgrade (secondary upgraded first, primary still old) | Old primary rejects Import in GlobalCluster; no compatibility issue triggered |

No migration is required. The `Uncommitted` and `Committing` state values (8, 9) are additive to the existing `ImportJobState` enum.

## Test Plan

- Unit tests for `IsAutoCommit` helper (`internal/util/importutilv2`)
- Unit tests for `HandleCommitVchannel` idempotency (duplicate vchannel, nil job, WAL replay)
- Unit tests for `CommitImport`/`AbortImport` RPC handlers: state validation, per-job KeyLock, broadcast happy path
- Unit tests for DDL ack callbacks: commit/abort race under resource-key-lock semantics, state guards, failed-state side effects
- Unit tests for `ImportChecker` `Uncommitted`/`Committing` cases: auto_commit trigger, repeated auto-commit safety, all-vchannels-confirmed transition
- Unit tests for CommitImport/RollbackImport broadcast targets: messages target `job.vchannels`, not the control channel
- Unit tests for WAL flusher dispatch: `CommitImportMessage` calls `HandleCommitVchannel`, `RollbackImportMessage` is no-op
- Unit tests for RESTful handlers: valid `jobId`, invalid `jobId` returns error code 1100
- Integration: non-replication cluster with default `auto_commit=true` import completes externally like pre-2PC, while internally passing through `Uncommitted → Committing → Completed`
- Integration: `auto_commit=false` import keeps data invisible at `Uncommitted`, exposes data after commit, and keeps data invisible after abort

## Rejected Alternatives

**Approach: set `is_importing=false` at the DDL ack callback (DataCoord only, no per-vchannel roundtrip)**

This would simplify the implementation by transitioning all segments to visible in a single step on the DataCoord side. However, it creates a window where QueryNode sees `is_importing=false` before the StreamingNode has flushed pending DML for the channel, allowing stale reads. The per-vchannel `HandleCommitVchannel` roundtrip ensures segments become visible only after the DML flush has been triggered for that vchannel.

**Approach: use a dedicated Commit RPC on MilvusService (public gRPC)**

Adding `CommitImport`/`AbortImport` to `MilvusService` would require changes to the external milvus-proto repository, coupling the release cycle. RESTful-only endpoints allow shipping this feature independently.

## References

- Implementation PR: [milvus-io/milvus#48524](https://github.com/milvus-io/milvus/pull/48524)
- Related: commit_timestamp MEP (`20260324-commit-timestamp.md`) for full flush-before-commit ordering guarantee
