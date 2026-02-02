# Force Promote for Primary-Secondary Failover

**Date:** 2026-02-02

## Overview

Add a `force_promote` flag to the `UpdateReplicateConfiguration` API that allows a secondary cluster to immediately become a standalone primary when the original primary is unavailable. This enables active-passive failover for Milvus cross-cluster replication.

## Motivation

In Milvus cross-cluster replication, a secondary cluster applies configuration changes by waiting for the primary cluster to broadcast the `AlterReplicateConfigMessage` via CDC. If the primary becomes unreachable, the secondary blocks indefinitely because it can never receive the replicated message.

Operators need a mechanism to:
- Promote a secondary cluster to primary during disaster recovery
- Resume service without waiting for the unreachable primary
- Handle incomplete transactions and broadcasts left in an inconsistent state after failover

## Current Replication Flow

**Primary Cluster:**
1. Receives `UpdateReplicateConfiguration` request
2. Broadcasts `AlterReplicateConfigMessage` to all pchannels
3. Returns after broadcast completes
4. Configuration persisted via broadcast callback

**Secondary Cluster:**
1. Receives `UpdateReplicateConfiguration` request
2. Attempts broadcast but fails with `ErrNotPrimary`
3. Waits for CDC to replicate the `AlterReplicateConfigMessage` from primary
4. Returns only when configuration matches

**Problem:** Secondary clusters block indefinitely if the primary is unreachable.

## Design

### API Change

Add `force_promote` field to the existing `UpdateReplicateConfigurationRequest`:

```protobuf
// In milvus.proto
message UpdateReplicateConfigurationRequest {
  common.ReplicateConfiguration replicate_configuration = 1;
  bool force_promote = 2;  // Immediately promote secondary to standalone primary
}
```

Add `force_promote` field to the internal message header:

```protobuf
// In messages.proto
message AlterReplicateConfigMessageHeader {
  common.ReplicateConfiguration replicate_configuration = 1;
  bool force_promote = 2;
}
```

Add metadata fields to track force-promoted configurations:

```protobuf
// In streaming.proto
message ReplicateConfigurationMeta {
  common.ReplicateConfiguration replicate_configuration = 1;
  bool force_promoted = 2;
  uint64 force_promote_timestamp = 3;
}
```

### Force Promote Constraints

For safety, force promote is restricted to failover scenarios only:

| Constraint | Validation | Rationale |
|---|---|---|
| Secondary cluster only | Attempt broadcast lock; reject if successful (we are primary) | Only secondary clusters need emergency promotion |
| Single cluster config | `len(config.Clusters) == 1` | Promoted cluster becomes standalone |
| Current cluster only | `config.Clusters[0].ClusterId == currentClusterID` | Cannot promote a different cluster |
| No topology | `len(config.CrossClusterTopology) == 0` | Standalone cluster has no replication edges |

### Force Promote Flow

```
Client SDK
    │  UpdateReplicateConfiguration(config, force_promote=true)
    ▼
Proxy
    │  Forward to StreamingCoord
    ▼
StreamingCoord (Assignment Service)
    │  1. Validate constraints (secondary, single cluster, no topology)
    │  2. Bypass broadcaster lock (use Broadcast().Append() directly)
    │  3. Append AlterReplicateConfigMessage with ForcePromote=true
    ▼
StreamingNode (Replicate Interceptor)
    │  4. Detect ForcePromote in message header
    │  5. Roll back all in-flight transactions via TxnManager
    │  6. Switch replication mode
    ▼
StreamingCoord (Broadcast Callback)
    │  7. Persist config with ForcePromoted=true flag
    │  8. Supplement incomplete broadcast messages to missing vchannels
    ▼
Done — cluster is now standalone primary
```

### Handling Incomplete Messages

When force promote completes, incomplete messages from the old topology must be handled:

#### Transaction Rollback (Automatic per StreamingNode)

When each StreamingNode processes the forced `AlterReplicateConfigMessage`:

1. Replicate interceptor detects `ForcePromote == true` in the message header
2. Calls `TxnManager.RollbackAllInFlightTransactions()`
3. All active transaction sessions are cleaned up
4. Rollback happens **after** the message is persisted in WAL (atomic)

```go
// TxnManager new method
func (m *txnManagerImpl) RollbackAllInFlightTransactions() {
    m.mu.Lock()
    defer m.mu.Unlock()
    for txnID, session := range m.sessions {
        session.Cleanup()
        delete(m.sessions, txnID)
    }
}
```

No remote detection or coordinator intervention is needed — each node handles its own transactions.

#### Broadcast Supplementation (In Callback on StreamingCoord)

In the `alterReplicateConfiguration()` callback:

1. Detect force promote from the message header flag
2. Query broadcaster for pending broadcast messages (`GetPendingBroadcastMessages()`)
3. Re-append pending messages to their target vchannels via WAL
4. Ensures partially-broadcasted DDL operations complete

```go
// Broadcaster new interface method
type Broadcaster interface {
    // ...existing methods...
    GetPendingBroadcastMessages() []message.MutableMessage
}
```

## Files Modified

### Proto Changes
- `pkg/proto/messages.proto` — Add `force_promote` to `AlterReplicateConfigMessageHeader`
- `pkg/proto/streaming.proto` — Add `force_promoted`, `force_promote_timestamp` to `ReplicateConfigurationMeta`

### Core Implementation
- `internal/streamingcoord/server/service/assignment.go` — Add `handleForcePromote()`, `validateForcePromoteConfiguration()`, `supplementIncompleteBroadcasts()`
- `internal/streamingcoord/server/balancer/channel/manager.go` — Persist force promote flag in configuration meta
- `internal/streamingnode/server/wal/interceptors/replicate/replicate_interceptor.go` — Detect force promote and trigger transaction rollback
- `internal/streamingnode/server/wal/interceptors/txn/txn_manager.go` — Add `RollbackAllInFlightTransactions()`
- `internal/streamingcoord/server/broadcaster/broadcast_manager.go` — Add `GetPendingBroadcastMessages()`
- `internal/streamingcoord/server/broadcaster/broadcaster.go` — Add method to `Broadcaster` interface

### Client & Proxy
- `internal/proxy/impl.go` — Pass through `force_promote` flag
- `client/milvusclient/replicate_builder.go` — Add `WithForcePromote()` builder method
- `internal/distributed/streaming/replicate_service.go` — Accept request object
- `internal/distributed/streaming/streaming.go` — Update `ReplicateService` interface
- `pkg/util/replicateutil/util.go` — Add logging helper

### Tests
- `internal/streamingcoord/server/service/assignment_test.go` — Force promote validation tests
- `internal/streamingnode/server/wal/interceptors/replicate/replicate_interceptor_test.go` — Force promote rollback tests
- `internal/streamingnode/server/wal/interceptors/txn/session_test.go` — RollbackAllInFlightTransactions tests
- `tests/integration/replication/force_promote_test.go` — Integration tests

## Edge Cases

1. **Primary cluster rejection** — Force promote explicitly rejected on primary clusters
2. **Concurrent force promotes** — Catalog atomic saves prevent corruption
3. **Idempotency** — `proto.Equal()` check skips duplicate updates
4. **Append failure** — No transaction rollback if message fails to persist (atomic guarantee)
5. **Nil TxnManager** — Gracefully handled (no-op) when TxnManager is not initialized
6. **Empty pending broadcasts** — Supplementation is a no-op when nothing is pending

## Alternatives Considered

### 1. Append RollbackTxn messages to WAL for each transaction
Rejected: Requires enumerating all in-flight transactions at the coordinator level and appending individual rollback messages. The programmatic `TxnManager.RollbackAllInFlightTransactions()` approach is simpler and avoids remote detection complexity.

### 2. Handle transaction rollback during WAL recovery
Rejected: Force promote is not a WAL recovery event. The `AlterReplicateConfigMessage` propagates naturally through the WAL to each StreamingNode, making the interceptor the correct place to trigger rollback.

### 3. Separate API endpoint for force promote
Rejected: Force promote is a specialized mode of `UpdateReplicateConfiguration`. Adding a separate endpoint would duplicate validation logic and complicate the client SDK.

## Related Issues

- https://github.com/milvus-io/milvus/issues/47351
- https://github.com/milvus-io/milvus/pull/47352
