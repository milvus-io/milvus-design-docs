# MEP: SHA-3 Support for Internal Hashing

- **Created:** 2026-04-14
- **Author(s):** @kailash360
- **Status:** Under Review
- **Component:** Proxy | Coordinator | Other
- **Related Issues:** #25970
- **Released:** 2.6.0

## Summary

Add configurable SHA-3 (SHA3-256) hashing support alongside the existing SHA-256 algorithm for internal Milvus operations. A new configuration parameter `common.security.hashAlgorithm` allows operators to choose between `sha256` (default) and `sha3` at deployment time. All internal hashing call sites — telemetry fingerprints and config digests — are unified behind a single `ComputeHash` abstraction.

## Motivation

Milvus currently hard-codes SHA-256 for all internal hashing operations (telemetry client IDs, config digests). While SHA-256 remains a valid choice, NIST has standardized SHA-3 (FIPS 202) as a structurally different and future-proof alternative. Organizations with strict cryptographic compliance requirements (e.g., government, finance, healthcare) may require or prefer SHA-3. This change makes Milvus's internal hashing algorithm configurable without requiring code changes, enabling adoption in compliance-sensitive environments.

## Public Interfaces

**New configuration parameter** in `configs/milvus.yaml` and paramtable:

```yaml
common:
  security:
    hashAlgorithm: sha256  # supported values: sha256 (default), sha3
```

**New Go package** `internal/util/crypto`:

```go
type HashType string

const (
    HashSHA256 HashType = "sha256"
    HashSHA3   HashType = "sha3"
)

// ComputeHash returns the hex-encoded digest of data using the specified algorithm.
// Unrecognized hashType values fall back to SHA-256.
func ComputeHash(data []byte, hashType HashType) string
```

No existing public API (gRPC/REST) is changed.

## Design Details

### Centralized Hash Abstraction

A new package `internal/util/crypto` provides `ComputeHash(data []byte, hashType HashType) string`. It uses:
- `crypto/sha256` (standard library) for SHA-256
- `golang.org/x/crypto/sha3` for SHA3-256

Unknown `HashType` values fall back to SHA-256, making the function safe against misconfiguration.

### Configuration

A new `ParamItem` named `HashAlgorithm` is added to `CommonConfig` in `pkg/util/paramtable/component_param.go`:

- Key: `common.security.hashAlgorithm`
- Default: `sha256`
- Formatter: validates input — only `"sha3"` passes through; everything else becomes `"sha256"`
- Exported and documented in `configs/milvus.yaml`

### Call Sites Updated

Two telemetry functions that previously used inline `crypto/sha256` are updated to read the algorithm from paramtable at call time:

| File | Function | Change |
|---|---|---|
| `internal/rootcoord/telemetry/manager.go` | `getOrCreateClientID` | Legacy client ID fingerprint |
| `internal/rootcoord/telemetry/manager.go` | `computeClientConfigHash` | Config digest for command delivery |
| `internal/rootcoord/telemetry/command_store.go` | `computeConfigHashFromConfigs` | Stored config hash |

Reading the algorithm at call time (rather than at startup) means the value can be changed via hot-reload without restarting Milvus.

## Compatibility, Deprecation, and Migration Plan

- **Existing deployments are unaffected.** The default value is `sha256`, identical to the current hard-coded behavior. No action required on upgrade.
- **Changing `hashAlgorithm` after deployment invalidates previously computed hashes.** Client IDs and config digests derived from SHA-256 will differ from those derived from SHA-3. This means:
  - Telemetry client IDs for legacy clients (those without a `client_id` in `Reserved`) will change, potentially creating duplicate entries during the transition window.
  - Config digests will be recomputed on the next heartbeat, triggering a one-time full config push to connected clients.
- **No migration tooling is needed** — hashes are ephemeral operational values, not persistent user data. There is no data loss.
- The config is documented as: *"Changing this value will invalidate any previously computed hashes."*

## Test Plan

### Unit Tests (`internal/util/crypto/hash_test.go`)
20 test cases with 100% statement coverage:
- Known NIST test vectors for both SHA-256 and SHA-3
- Output length, hex validity, determinism, fallback behavior, large inputs, binary data, nil input, truncation patterns

### Paramtable Tests (`pkg/util/paramtable/component_param_test.go`)
7 sub-tests covering the `Formatter` function:
- Default value is `sha256`
- Setting `sha3` returns `sha3`
- Invalid values (`""`, `"md5"`, `"SHA256"`, `"SHA3"`, `"sha512"`, `"unknown"`) all fall back to `sha256`

### Telemetry Unit Tests (`internal/rootcoord/telemetry/manager_test.go`)
4 new test functions exercising all 3 SHA-3 call sites:
- `TestGetOrCreateClientIDWithSHA3` — verifies SHA-256 and SHA-3 produce different legacy client IDs with correct format and determinism
- `TestComputeClientConfigHashWithSHA3` — verifies config hash differs between algorithms; tests nil/empty edge cases
- `TestComputeConfigHashFromConfigsWithSHA3` — same for the command store hash function
- `TestHashAlgorithmSwitchDuringHeartbeat` — verifies `HandleHeartbeat` works correctly when algorithm is switched at runtime

All unit tests run without Docker, C++ core, or external services.

## Rejected Alternatives

**Hard-coding SHA-3 as replacement for SHA-256**
Rejected because it would break all existing deployments that have computed hashes stored or cached, with no migration path.

**Making the algorithm a startup flag instead of config**
Rejected because the paramtable system already supports hot-reload. Reading from paramtable at call time is idiomatic in Milvus and requires no additional infrastructure.

**Supporting MD5 or SHA-1 for legacy compatibility**
Rejected because both are cryptographically broken. The formatter intentionally rejects any value other than `sha256` and `sha3` to prevent insecure configurations.

**Per-component algorithm selection**
Rejected as over-engineering. A single cluster-wide setting is operationally simpler and sufficient for compliance use cases.

## References

- [GitHub Issue #25970 — Feature: SHA-3 Support in Milvus](https://github.com/milvus-io/milvus/issues/25970)
- [NIST FIPS 202 — SHA-3 Standard](https://csrc.nist.gov/publications/detail/fips/202/final)
- [NIST SP 800-185 — SHA-3 Derived Functions](https://csrc.nist.gov/publications/detail/sp/800-185/final)
- [`golang.org/x/crypto/sha3`](https://pkg.go.dev/golang.org/x/crypto/sha3)