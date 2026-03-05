# RFC-0016: Shared Key Authentication for Local MCP Servers

> **Note**: This was originally [THV-2570](https://github.com/stacklok/toolhive/pull/2570) in the toolhive repository.

- **Status**: Draft
- **Author(s)**: Juan Antonio Osorio (@JAORMX)
- **Created**: 2025-11-13
- **Last Updated**: 2025-11-13
- **Target Repository**: toolhive
- **Related Issues**: [toolhive#2570](https://github.com/stacklok/toolhive/pull/2570)

## Summary

This proposal introduces a lightweight authentication mechanism for localhost MCP server deployments that provides defense-in-depth security without requiring external identity providers. It generates cryptographically-secure shared keys per workload, stores them in the OS keychain, and validates them transparently through the stdio bridge.

## Problem Statement

Local ToolHive deployments bind MCP servers to localhost (`127.0.0.1`) for network-level security, but have no authentication layer. While localhost binding prevents remote access, customers want defense-in-depth protection against:

- Malicious local processes connecting to open localhost ports
- Port forwarding or SSH tunneling accidentally exposing endpoints
- Privilege escalation attacks targeting unauthenticated local services
- Compliance requirements mandating authentication even for localhost

Current authentication options (OIDC, local user auth) require external identity providers or manual password management, making them unsuitable for single-user local deployments.

## Goals

- Provide lightweight authentication for localhost MCP server deployments
- Require zero or minimal configuration from users
- Leverage existing ToolHive infrastructure (OS keychain, middleware, stdio bridge)
- Work transparently with the `thv proxy stdio` bridge used by MCP clients
- Maintain backward compatibility (opt-in feature)

## Non-Goals

- Replace network isolation or container-based security
- Support multi-user local deployments (use OIDC instead)
- Protect against root/administrator access or OS keychain compromise
- Store credentials in configuration files

## Proposed Solution

Implement a shared key authentication middleware that generates unique cryptographic keys per MCP server workload, stores them in the OS keychain, and validates them on incoming requests.

### High-Level Design

```
+-------------------------------------------------------------+
|                     MCP Client                               |
|                 (Claude Desktop, etc.)                       |
+----------------------+--------------------------------------+
                       | stdio
                       v
+-------------------------------------------------------------+
|              thv proxy stdio Bridge                          |
|  - Reads shared key from OS keychain                         |
|  - Injects X-ToolHive-Auth header in HTTP requests          |
+----------------------+--------------------------------------+
                       | HTTP + X-ToolHive-Auth header
                       v
+-------------------------------------------------------------+
|              ToolHive HTTP Proxy                             |
|  - Shared Key Auth Middleware validates header              |
|  - Constant-time comparison with env variable               |
|  - Other middleware (Parser, Authz, Audit)                  |
+----------------------+--------------------------------------+
                       | Forwarded request
                       v
+-------------------------------------------------------------+
|              MCP Server Container                            |
|  - Receives TOOLHIVE_SHARED_KEY env variable                 |
+-------------------------------------------------------------+
```

### Detailed Design

#### Component Changes

**1. Key Generation and Storage** (`pkg/auth/sharedkey/`)
- Generate cryptographically-secure 32-byte keys using `crypto/rand`
- Store in OS keychain via existing encrypted secrets provider
- Key naming: `workload:<workload-name>:shared-key`
- Lifecycle: Generate on deployment, delete on workload removal

**2. Shared Key Authentication Middleware** (`pkg/auth/sharedkey/middleware.go`)
- Implements standard `types.Middleware` interface
- Validates `X-ToolHive-Auth` header against `TOOLHIVE_SHARED_KEY` environment variable
- Uses `crypto/subtle.ConstantTimeCompare` to prevent timing attacks
- Returns `401 Unauthorized` for missing/invalid keys
- Adds identity to context for audit trail

**3. Workload Integration** (`pkg/workloads/manager.go`)
- Generate and store shared key when `SharedKeyAuth.Enabled` is set in RunConfig
- Inject key into container and proxy via `TOOLHIVE_SHARED_KEY` environment variable
- Configure middleware chain automatically
- Clean up key on workload deletion

**4. Stdio Bridge Enhancement** (`cmd/thv/app/proxy_stdio.go`)
- Retrieve shared key from secrets provider on startup
- Inject `X-ToolHive-Auth` header in all HTTP requests via custom transport wrapper
- Handle 401 responses with informative error messages

#### Configuration Changes

```go
type SharedKeyAuthConfig struct {
    Enabled bool `json:"enabled"`
    KeyRotationDays int `json:"keyRotationDays,omitempty"` // 0 = disabled
}

// Add to RunConfig struct
SharedKeyAuth *SharedKeyAuthConfig `json:"sharedKeyAuth,omitempty"`
```

### Request Flow

1. **Workload Deployment**: User runs `thv run my-server --shared-key-auth`
2. **Key Generation**: Workload manager generates 32-byte random key
3. **Key Storage**: Key stored as `workload:my-server:shared-key` in OS keychain
4. **Key Injection**: Key injected via `TOOLHIVE_SHARED_KEY` environment variable to container and proxy
5. **Middleware Configuration**: Shared key auth middleware added to chain
6. **Client Configuration**: MCP client configured to use `thv proxy stdio my-server`

### Authentication Flow

1. **Client Request**: MCP client sends JSON-RPC over stdio to `thv proxy stdio`
2. **Key Retrieval**: Stdio bridge retrieves key from OS keychain
3. **Header Injection**: Bridge adds `X-ToolHive-Auth: <key>` to HTTP request
4. **Validation**: Middleware validates header against environment variable
5. **Forwarding**: On success, request forwarded to MCP server container
6. **Rejection**: On failure, return `401 Unauthorized`

## Security Considerations

### Threat Model

**Protected Against**:
- Unauthorized local process connections
- Port scanning attacks
- Accidental credential exposure in config files
- Replay attacks (keys are workload-scoped)

**Not Protected Against**:
- Root/administrator access to keychain
- Process memory inspection
- Physical machine access
- Compromised OS keychain

### Authentication and Authorization

- Keys are 256-bit (32 bytes) cryptographically secure random values
- Validation uses constant-time comparison to prevent timing attacks
- Each workload has a unique key (no key sharing)
- Keys are automatically rotated when workloads are recreated

### Data Security

- **Key Strength**: 32 bytes (256 bits), base64-encoded
- **Key Generation**: Crypto-secure random via `crypto/rand`
- **Storage**: AES-256-GCM encrypted in OS keychain (existing provider)
- **Comparison**: Constant-time to prevent timing attacks
- **Uniqueness**: Independent key per workload
- **Lifecycle**: Keys deleted with workload

### Secrets Management

- Keys stored in OS keychain using existing encrypted secrets provider
- Key naming convention: `workload:<workload-name>:shared-key`
- Keys automatically cleaned up on workload deletion
- No plaintext storage in configuration files

### Audit and Logging

- Authentication attempts logged with workload context
- Failed authentication attempts logged with source information
- Successful authentications add identity to request context for downstream audit

### Mitigations

Shared key authentication adds a layer to existing security:

1. Network isolation (localhost binding)
2. Container isolation
3. **Shared key authentication** - New layer
4. Authorization (Cedar policies)
5. Audit logging

Each layer provides independent protection.

## Alternatives Considered

### Alternative 1: Certificate-Based Authentication (mTLS)

- **Pros**: Industry-standard, strong crypto
- **Cons**: Complex (certificate generation, validation, expiry), TLS infrastructure overhead, overkill for localhost
- **Why not chosen**: Rejected due to complexity

### Alternative 2: Unix Domain Sockets

- **Pros**: Filesystem-based permissions, OS-level access control
- **Cons**: Not cross-platform (Windows), breaks HTTP architecture, requires significant changes
- **Why not chosen**: Rejected due to platform limitations

### Alternative 3: API Keys in Config Files

- **Pros**: Simple implementation, easy debugging
- **Cons**: Plaintext storage, git commit risk, no leverage of existing secrets infrastructure
- **Why not chosen**: Rejected due to poor security properties

### Alternative 4: Time-Based One-Time Passwords (TOTP)

- **Pros**: Strong authentication, industry-standard
- **Cons**: Time synchronization complexity, poor UX for automated clients, overkill for localhost
- **Why not chosen**: Rejected due to complexity and poor UX

## Compatibility

### Backward Compatibility

- Opt-in feature via flag or RunConfig
- Existing workloads continue working without changes
- No breaking changes to configurations
- Gradual migration: enable per-workload incrementally

### Forward Compatibility

- Key format can be extended (e.g., versioned keys)
- Additional authentication methods can be added alongside
- Multi-workload scenarios work transparently with unique keys per workload

## Implementation Plan

### Phase 1: Core Infrastructure
- Create `pkg/auth/sharedkey/` package
- Implement key generation and storage
- Implement shared key authentication middleware
- Add middleware factory registration
- Extend RunConfig schema

### Phase 2: Workload Integration
- Modify workloads manager for key lifecycle
- Add CLI flag `--shared-key-auth`
- Configure middleware chain automatically
- Implement key cleanup on deletion

### Phase 3: Stdio Bridge Enhancement
- Enhance `thv proxy stdio` to retrieve keys
- Implement header injection in HTTP transport
- Add error handling for auth failures
- Update client configuration generation

### Phase 4: Testing and Documentation
- Unit tests for middleware and key management
- Integration tests for end-to-end flows
- Security testing (timing attacks, etc.)
- Update architecture documentation
- Create user guide

## Testing Strategy

- **Unit tests**: Key generation, middleware validation, constant-time comparison
- **Integration tests**: End-to-end auth flow with stdio bridge
- **Security tests**: Timing attack resistance, key uniqueness
- **E2E tests**: Full workload lifecycle with authentication

## Documentation

- User guide for enabling shared key authentication
- Architecture documentation updates
- Security considerations documentation
- Troubleshooting guide for authentication failures

## Open Questions

1. Should key rotation be automatic or manual?
2. Should we support multiple keys per workload for rotation overlap?

## References

- [ToolHive Middleware Architecture](https://github.com/stacklok/toolhive/blob/main/docs/middleware.md)
- [Go crypto/rand documentation](https://pkg.go.dev/crypto/rand)
- [Go crypto/subtle documentation](https://pkg.go.dev/crypto/subtle)

---

## RFC Lifecycle

### Review History

| Date | Reviewer | Decision | Notes |
|------|----------|----------|-------|
| 2025-11-13 | - | Draft | Ported from toolhive PR #2570 |

### Implementation Tracking

| Repository | PR | Status |
|------------|-----|--------|
| toolhive | - | Pending |
