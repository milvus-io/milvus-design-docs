# Fix Etcd Watch and Session Lifecycle Issues

**Date:** 2026-01-30

**Author(s):** @xiaofan-luan

**Status:** Implemented

**Component:** Coordinator

**Related Issues:** #7989

**Released:** TBD

## Summary

This document describes fixes for several reliability issues in the session utility module (`session_util.go`), addressing etcd watch channel management, keepalive timeout handling, and error recovery in active-standby mode.

## Motivation

The session utility is critical infrastructure that manages node registration and service discovery via etcd. Several edge cases were identified that could lead to:

1. **Silent failures**: Error from `checkIDExist()` was ignored, potentially leaving the system in an undefined state
2. **Infinite loops**: `getServerIDWithKey()` could loop forever if the context was cancelled
3. **Zombie sessions**: Keepalive channels could block indefinitely if etcd stopped responding, leaving sessions that appeared alive but were actually expired
4. **Resource leaks**: Watch channels used the parent session context instead of their own cancellable context, preventing proper cleanup on `Stop()`
5. **Panics**: DELETE events with nil `PrevKv` caused nil pointer dereferences
6. **Watch failures**: `ProcessActiveStandBy` did not handle `ErrCompacted`, causing standby nodes to stop watching

## Design Details

### 1. Error Handling in checkIDExist()

**Before:** Error from etcd transaction was silently ignored.

**After:** Error is returned and causes Init() to panic, failing fast rather than continuing with undefined state.

```go
func (s *Session) checkIDExist() error {
    _, err := s.etcdCli.Txn(s.ctx).If(...).Then(...).Commit()
    if err != nil {
        log.Ctx(s.ctx).Warn("failed to check/create ID key in etcd", zap.Error(err))
    }
    return err
}
```

### 2. Context Cancellation Check in getServerIDWithKey()

**Before:** Loop continued indefinitely even if context was cancelled.

**After:** Check context at each iteration to exit promptly.

```go
for {
    select {
    case <-s.ctx.Done():
        return -1, s.ctx.Err()
    default:
    }
    // ... rest of loop
}
```

### 3. Keepalive Timeout Protection

**Before:** The keepalive loop only exited when the channel closed, which might not happen if etcd stops responding.

**After:** Add a timeout based on session TTL. If no keepalive response is received within the TTL period, the session is considered expired and the process exits.

```go
keepaliveTimeout := time.Duration(s.sessionTTL) * time.Second
timer := time.NewTimer(keepaliveTimeout)
for {
    select {
    case _, ok := <-ch:
        if !ok {
            break keepaliveLoop
        }
        timer.Reset(keepaliveTimeout)
    case <-timer.C:
        log.Error("keepalive response timeout, session expired")
        os.Exit(exitCodeSessionLeaseExpired)
    case <-s.ctx.Done():
        return
    }
}
```

### 4. Watcher Context Management

**Before:** Watch calls used `s.ctx` (session context), so calling `watcher.Stop()` wouldn't cancel the watch.

**After:** Each watcher creates its own context derived from session context, stored in the watcher struct. Watch calls use this context, ensuring `Stop()` properly cancels the watch.

```go
type sessionWatcher struct {
    s         *Session
    ctx       context.Context  // Added: watcher's own context
    cancel    context.CancelFunc
    // ...
}

// In WatchServices:
ctx, cancel := context.WithCancel(s.ctx)
w := &sessionWatcher{
    ctx:    ctx,
    cancel: cancel,
    rch:    s.etcdCli.Watch(ctx, ...),  // Use ctx, not s.ctx
    // ...
}
```

### 5. Nil PrevKv Handling

**Before:** DELETE events assumed `ev.PrevKv` was always non-nil.

**After:** Check for nil and skip the event with a warning log.

```go
case mvccpb.DELETE:
    if ev.PrevKv == nil {
        log.Warn("DELETE event with nil PrevKv, skipping",
            zap.String("key", string(ev.Kv.Key)))
        continue
    }
    // ... process event
```

### 6. ErrCompacted Handling in ProcessActiveStandBy

**Before:** Watch errors in `ProcessActiveStandBy` caused the function to exit without recovery.

**After:** On `ErrCompacted`, fetch the latest revision and restart the watch.

```go
if wresp.Err() == v3rpc.ErrCompacted {
    log.Warn("watch ACTIVE key compacted, rewatch with latest revision")
    resp, err := s.etcdCli.Get(ctx, s.activeKey)
    if err != nil {
        break watchLoop
    }
    if resp.Count == 0 {
        // Active key deleted, try to register
        break watchLoop
    }
    revision = resp.Header.Revision + 1
    continue watchLoop  // Restart watch with new revision
}
```

## Test Plan

Comprehensive unit tests have been added covering:

1. **checkIDExist error handling**: Verify Init panics on etcd failure
2. **getServerIDWithKey context cancellation**: Verify prompt exit on context cancel
3. **Keepalive timeout**: Verify session exits when keepalive times out
4. **Watcher context isolation**: Verify Stop() cancels watch operations
5. **Nil PrevKv handling**: Verify DELETE events with nil PrevKv are handled gracefully
6. **ProcessActiveStandBy ErrCompacted**: Verify rewatch happens on compaction

Tests use synchronization primitives (channels, condition variables) instead of time-based waits for reliability.

## Compatibility

These changes are backward compatible:
- No API changes
- No configuration changes
- Behavior changes only affect error handling and edge cases

## References

- etcd clientv3 documentation: https://etcd.io/docs/v3.5/learning/api/
- gRPC keepalive: https://grpc.io/docs/guides/keepalive/
