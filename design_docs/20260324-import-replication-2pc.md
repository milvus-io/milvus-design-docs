# MEP: Import Two-Phase Commit for Primary/Secondary Replication

- **Created:** 2026-03-24
- **Author(s):** @bigsheeper
- **Status:** Under Review
- **Component:** DataCoord | StreamingNode | Proxy
- **Related Issues:** [milvus-io/milvus#48525](https://github.com/milvus-io/milvus/issues/48525)
- **Released:** TBD

## Summary

Add Two-Phase Commit (2PC) support to Import so that bulk-loaded data can become visible atomically and consistently across all clusters in a primary/secondary replication setup (GlobalCluster / disaster recovery). Data stays invisible until an explicit commit signal is delivered via WAL, ensuring primary and secondary clusters reach the same visible state at the same logical WAL position.

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
  // ...existing values through RefreshExternalCollection=43...
  CommitImport   = 44;
  RollbackImport = 45;
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

### CommitImport Flow

**Phase 1 — RPC → WAL broadcast (DataCoord)**

```
Platform
  │  POST /v2/vectordb/jobs/import/commit {"jobID": "123"}
  ▼
Proxy (converts "123" → int64(123))
  → DataCoord.CommitImport(job_id=123)
       │
       ├─ Validate: job state == Uncommitted; return InvalidState otherwise
       ├─ Acquire per-job mutex (in-memory sync.Map keyed by job_id)
       ├─ Broadcast CommitImportMessage{collection_id, job_id} to all vchannels (WAL)
       └─ Release per-job mutex
```

**Phase 2 — DDL ack callback (DataCoord, fires once for all vchannels)**

The DDL broadcast ack callback fires once when the message has been successfully written to all vchannels' WALs.

```
DDL ack callback:
  CAS: Uncommitted → Committing
  (if already Failed: abort won the race → no-op)
```

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
  CAS: if vchannel NOT in job.committed_vchannels:
    - set is_importing=false for all segments in this vchannel
    - append vchannel to committed_vchannels (persist to etcd)
  else:
    no-op (idempotent — handles WAL replay on SN restart)
```

**Phase 5 — ImportChecker (background, ticker)**

```go
case ImportJobState_Committing:
    if len(job.CommittedVchannels) == len(job.Vchannels) {
        updateJobState(ImportJobState_Completed)
    }
    // else: wait for remaining vchannels (WAL delivery guaranteed)
```

### AbortImport Flow

`AbortImport` follows the same broadcast pattern. The DDL ack callback:
1. CAS: current state → `Failed` (no-op if already `Committing`/`Completed`)
2. Sets `RequestedDiskSize = 0` (disk quota released)
3. Sets `CleanupTs = now + retention` (GC eligible)
4. Marks all import segments for this job as `SegmentState_Dropped`

A **no-op handler** is registered for `RollbackImport = 45` in `wal_flusher.dispatch()` to prevent unknown-message errors as flowgraph consumes WAL messages.

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

### Race Safety

| Scenario | Protection |
|----------|------------|
| Concurrent CommitImport + AbortImport RPCs | Per-job mutex serializes broadcast; only one message enters WAL first |
| Commit ack fires before abort ack | Abort ack CAS sees `Committing` → no-op |
| Abort ack fires before commit ack | Commit ack CAS sees `Failed` → no-op |
| Duplicate `CommitImportMessage` (WAL replay on SN restart) | `committed_vchannels` CAS in `HandleCommitVchannel` → idempotent no-op |

### Platform Workflow (Replication Clusters)

1. Copy import files to all clusters' object storage
2. Call `ImportV2` on **primary** with `auto_commit=false`; CDC replicates `ImportMessage` to secondaries
3. Poll `GetImportProgress` on **all** clusters until all report `Uncommitted`
4. Call `POST /v2/vectordb/jobs/import/commit` on **primary**; `CommitImportMessage` written to primary WAL, CDC replicates to secondary WALs; all clusters process independently
5. Poll until all clusters report `Completed`

**Abort conditions**: call `POST /v2/vectordb/jobs/import/abort` on each cluster independently if any cluster's job fails, times out before reaching `Uncommitted`, or user requests cancellation. Once `Committing` is reached, abort is no longer possible.

### Implementation Notes

- `CommitImport`/`RollbackImport` messages are handled in `wal_flusher.dispatch()` (same pattern as `CreateCollection`/`DropCollection`), not in `flow_graph_dd_node.go`. This avoids requiring additions to the external milvus-proto `commonpb.MsgType`.
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
- Unit tests for `CommitImport`/`AbortImport` RPC handlers: state validation, per-job mutex, broadcast happy path
- Unit tests for DDL ack callbacks: commit CAS races (abort wins), rollback CAS races (commit wins), segment drop
- Unit tests for `ImportChecker` `Uncommitted`/`Committing` cases: auto_commit trigger, all-vchannels-confirmed transition
- Unit tests for WAL flusher dispatch: `CommitImportMessage` calls `HandleCommitVchannel`, `RollbackImportMessage` is no-op
- Unit tests for RESTful handlers: valid `jobId`, invalid `jobId` returns error code 1100
- Integration: non-replication cluster with default `auto_commit=true` import completes identically to pre-2PC

## Rejected Alternatives

**Approach: set `is_importing=false` at the DDL ack callback (DataCoord only, no per-vchannel roundtrip)**

This would simplify the implementation by transitioning all segments to visible in a single step on the DataCoord side. However, it creates a window where QueryNode sees `is_importing=false` before the StreamingNode has flushed pending DML for the channel, allowing stale reads. The per-vchannel `HandleCommitVchannel` roundtrip ensures segments become visible only after the DML flush has been triggered for that vchannel.

**Approach: use a dedicated Commit RPC on MilvusService (public gRPC)**

Adding `CommitImport`/`AbortImport` to `MilvusService` would require changes to the external milvus-proto repository, coupling the release cycle. RESTful-only endpoints allow shipping this feature independently.

## References

- Implementation PR: [milvus-io/milvus#48524](https://github.com/milvus-io/milvus/pull/48524)
- Related: commit_timestamp MEP (`20260324-commit-timestamp.md`) for full flush-before-commit ordering guarantee
