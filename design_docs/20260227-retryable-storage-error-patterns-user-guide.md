# Configurable Retryable Error Patterns for Object Storage

## Overview

Starting from v2.6.12, Milvus enhances its object storage retry logic with two improvements:

1. **Built-in transient network error detection**: Automatically detects and retries Go-level transient network errors (timeouts, connection refused, deadline exceeded) that were previously not retried.
2. **User-configurable retryable error patterns**: Allows operators to define custom error message substrings that should trigger retries, enabling support for non-standard S3-compatible backends without code changes.

## Background

Milvus relies on object storage (MinIO, AWS S3, Azure Blob, GCS, or S3-compatible backends) for persistent data storage. The existing retry logic handles vendor-specific errors (e.g., MinIO `ErrorResponse`, Azure `ResponseError`, Google `googleapi.Error`) but does not cover:

- Go standard library network errors (e.g., `net/http: timeout awaiting response headers`)
- Custom S3-compatible backends that return non-standard error codes or messages on transient failures

When these errors occur during bulk import or data read operations, Milvus incorrectly classifies them as non-retryable, causing unnecessary task failures.

## Configuration

### Parameter

| Parameter | Default | Dynamic | Description |
|---|---|---|---|
| `common.storage.retryableErrorPatterns` | `""` (empty) | Yes | Comma-separated list of error message substrings that should trigger retry for object storage operations. |

### How It Works

When Milvus encounters an error from object storage, it classifies the error through the following priority chain:

1. **Vendor-specific errors**: MinIO, Azure, GCS error responses are checked first using their native error types and HTTP status codes.
2. **Standard IO errors**: `io.ErrUnexpectedEOF` and too-many-requests errors are handled.
3. **Built-in transient network detection** (new): Go `net.Error` with `Timeout() == true`, `syscall.ECONNREFUSED`, and `os.ErrDeadlineExceeded` are automatically classified as retryable.
4. **User-configured patterns** (new): If the error message contains any of the configured substrings, it is classified as retryable.
5. **Fallback**: All other errors are classified as non-retryable.

Retryable errors are wrapped as `ErrIoTransient` (error code 1004) and will be retried with exponential backoff (200ms initial, up to 3s cap, default 10 attempts).

### Configuration Examples

**Example 1: Custom S3-compatible backend with throttling**

If your storage backend returns errors like `"server throttled: rate limit exceeded"`, configure:

```yaml
common:
  storage:
    retryableErrorPatterns: "server throttled"
```

**Example 2: Multiple patterns**

```yaml
common:
  storage:
    retryableErrorPatterns: "server throttled,custom busy,temporary unavailable"
```

**Example 3: Runtime update via Milvus client**

Since this parameter is dynamically refreshable, you can update it at runtime without restarting Milvus:

```python
from pymilvus import utility

utility.alter_config("common.storage.retryableErrorPatterns", "server throttled,custom busy")
```

## Built-in Transient Errors (No Configuration Needed)

The following errors are now automatically retried without any configuration:

| Error Type | Example | Previously Retried? |
|---|---|---|
| `net.Error` with `Timeout()` | `net/http: timeout awaiting response headers` | No |
| `syscall.ECONNREFUSED` | Connection refused to storage endpoint | No |
| `os.ErrDeadlineExceeded` | I/O deadline exceeded | No |
| `io.ErrUnexpectedEOF` | Unexpected EOF during read | Yes (existing) |
| MinIO `TooManyRequestsException` | S3 rate limiting | Yes (existing) |

## Usage Notes

- **Pattern matching is case-sensitive**: The error message substring must match exactly. For example, `"Server Throttled"` will not match `"server throttled"`.
- **Patterns are matched as substrings**: The pattern `"timeout"` will match any error containing the word "timeout" anywhere in the message.
- **Empty patterns are ignored**: Leading/trailing empty strings from comma splitting (e.g., `",pattern,"`) are safely ignored.
- **Applies to read-path operations**: This retry logic covers object storage reads (data imports, segment loading, binlog reads). Write operations have their own retry mechanisms.

## Troubleshooting

### Identifying non-retried storage errors

Search Milvus logs for `IO failed` errors during import or query operations:

```
grep "IO failed" milvus.log
```

If you see errors like:
```
IO failed[key=...]: Get "https://your-storage/...": net/http: timeout awaiting response headers
```

This error is now automatically retried in v2.6.12+. For other custom error messages, add the relevant substring to `retryableErrorPatterns`.

### Verifying retry behavior

Enable trace-level logging to see retry attempts:

```yaml
common:
  traceLogMode: 1
```

Look for repeated read attempts on the same key in the logs, followed by eventual success.

## Related Configuration

| Parameter | Default | Description |
|---|---|---|
| `common.storage.readRetryAttempts` | `10` | Maximum number of retry attempts for retryable storage read errors. |

## Version Compatibility

| Milvus Version | Built-in Transient Detection | Configurable Patterns |
|---|---|---|
| < 2.6.12 | Not supported | Not supported |
| >= 2.6.12 | Supported | Supported |
