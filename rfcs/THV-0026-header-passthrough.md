# THV-0020: Configurable Header Passthrough for Remote MCP Servers

- **Status**: Draft
- **Author(s)**: Jakub Hrozek (@jhrozek)
- **Created**: 2026-01-21
- **Last Updated**: 2026-01-21
- **Target Repository**: toolhive
- **Related Issues**: [toolhive#3316](https://github.com/stacklok/toolhive/issues/3316)

## Summary

Add server-side configuration for injecting additional HTTP headers into
requests forwarded to remote MCP servers.

Currently, if specific headers need to be sent to remote MCP servers, clients
must configure them individually. This approach is brittle and doesn't scale
as different MCP clients have varying configuration mechanisms, and managing
header settings across many clients is error-prone.

This RFC proposes configuring headers at the ToolHive proxy/server level,
allowing operators to specify header name-value pairs that are automatically
injected into every request to the remote server. This removes the burden
from clients and provides a single, centralized configuration point.

## Problem Statement

### Current Behavior

ToolHive's transparent proxy (using Go's `httputil.ReverseProxy`) passes
through most HTTP headers from client requests to remote MCP servers by
default. This includes:

- Standard headers (Accept, Content-Type, etc.)
- Custom headers (X-Request-ID, X-Correlation-ID, etc.)
- Authentication headers (which are then optionally overriden by token injection or token exchange middleware)

### Affected User Scenarios

- Users proxying to remote MCP servers via `thv proxy`
- Users running remote MCP servers via `thv run http://...`
- Kubernetes operators deploying `MCPRemoteProxy` resources

## Goals

- Provide CLI flags (`--remote-forward-headers`) for `thv proxy` and `thv run` commands to inject headers
- Add `headerForward` field to the `MCPRemoteProxy` Kubernetes CRD
- Enable server-side header configuration, removing the need for client-side configuration
- Make header injection intentional, documented, and auditable
- Maintain backward compatibility with existing deployments

## Non-Goals

- Filtering or blocking headers from client requests (only adding headers)
- Modifying response headers (only request headers)
- Supporting header injection for container-based (non-remote) MCP servers
- Dynamic header values (headers are static configuration)

## Proposed Solution

### High-Level Design

The solution uses a middleware that injects configured headers into requests
before they are forwarded to the remote server. This leverages Go's
`httputil.ReverseProxy` behavior where headers on the request are cloned to
the outgoing request.

```mermaid
flowchart LR
    Client[Client Request] --> MW[Header Forward<br/>Middleware]
    MW -->|Injects configured<br/>headers| Auth[Auth Middleware]
    Auth --> Proxy[Transparent Proxy]
    Proxy -->|ReverseProxy.Clone copies<br/>all headers| Remote[Remote MCP Server]
```

This approach follows the existing `token_injection` middleware pattern, which
sets headers on the request that are then forwarded to the backend.

### Detailed Design

#### Configuration Structure

```go
// HeaderForwardConfig defines configuration for injecting headers into requests to remote servers
type HeaderForwardConfig struct {
    // AddHeaders is a map of header names to values to inject into requests.
    // These headers are added server-side, so clients don't need to configure them.
    AddHeaders map[string]string `json:"addHeaders,omitempty" yaml:"addHeaders,omitempty"`
}
```

#### Middleware Implementation

The middleware injects configured headers into every request before forwarding to the remote server.

```go
// CreateHeaderForwardMiddleware creates middleware that injects configured headers
// into requests before they are forwarded to the remote server.
// This allows server-side configuration of headers, removing the need for clients
// to configure them individually.
func CreateHeaderForwardMiddleware(addHeaders map[string]string) func(http.Handler) http.Handler {
    if len(addHeaders) == 0 {
        return func(next http.Handler) http.Handler { return next }
    }

    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            for name, value := range addHeaders {
                r.Header.Set(name, value)
                logger.Debugf("Injected header %s for remote server", name)
            }
            next.ServeHTTP(w, r)
        })
    }
}
```

#### Middleware Ordering

The header forward middleware is added **last** in the middleware chain, right before the proxy handler:

```
Auth Middleware → Token Injection/Exchange → Header Forward → Proxy Handler
```

This ordering ensures:
1. Headers are only injected for authenticated requests
2. Injected headers are the final state before forwarding (can't be accidentally overwritten)
3. Clear separation: all "outgoing request modification" happens at the end

#### Why This Works

Go's `httputil.ReverseProxy` clones the incoming request before forwarding:

```go
// Inside ReverseProxy.ServeHTTP:
outreq := req.Clone(ctx)  // All headers are copied, including those set by middleware
```

This is the same pattern used by `token_injection.go`:
```go
r.Header.Set("Authorization", fmt.Sprintf("Bearer %s", token.AccessToken))
next.ServeHTTP(w, r)  // Header reaches backend via Clone()
```

Headers set by middleware are included in the clone and forwarded to the backend.

#### CLI Changes

**thv proxy command:**

```bash
thv proxy my-server --target-uri http://api.example.com \
  --remote-forward-headers "X-Tenant-ID=tenant123" \
  --remote-forward-headers "X-Custom-Header=custom-value"
```

**thv run command (remote):**

```bash
thv run http://remote-mcp.example.com \
  --remote-forward-headers "X-API-Key=secret123" \
  --remote-forward-headers "X-Environment=production"
```

#### Kubernetes CRD Changes

```yaml
apiVersion: toolhive.stacklok.io/v1alpha1
kind: MCPRemoteProxy
metadata:
  name: my-remote-proxy
spec:
  remoteURL: https://api.example.com/mcp
  oidcConfig:
    type: kubernetes
  headerForward:
    addHeaders:
      X-Tenant-ID: "tenant123"
      X-Environment: "production"
      X-Custom-Header: "custom-value"
```

#### RunConfig Changes

The `RunConfig` struct gains a new field:

```go
type RunConfig struct {
    // ... existing fields ...

    // HeaderForward contains configuration for injecting headers into requests to remote servers.
    // These headers are added server-side to every request forwarded to the remote MCP server.
    HeaderForward *HeaderForwardConfig `json:"header_forward,omitempty" yaml:"header_forward,omitempty"`
}

// Usage: config.HeaderForward.AddHeaders["X-Tenant-ID"] = "tenant123"
```

## Security Considerations

### Authentication and Authorization

- The `Authorization` header is handled separately by the token injection middleware
- Passthrough headers are applied **after** authentication middleware, so they cannot bypass auth
- The middleware only captures headers explicitly configured by the operator

### Input Validation

- Header names are normalized using `http.CanonicalHeaderKey()` for consistent matching
- No validation is performed on header values (passed through as-is)
- Configuration validates that header names are provided as a list of strings

### Secrets Management

- This feature does not handle secrets directly
- Operators should not configure sensitive headers (like API keys) as passthrough headers
- Authentication should use the existing token injection middleware instead

### Audit and Logging

- Header passthrough configuration is logged at startup
- Header capture events are logged at DEBUG level
- No sensitive header values are logged

### Mitigations

1. **Explicit opt-in**: Headers only pass through if explicitly configured
2. **Additive mode default**: Existing behavior is preserved; configuration adds to it
3. **Separation from auth**: Authentication headers are handled by dedicated middleware
4. **Debug-only logging**: Header names (not values) logged only at DEBUG level

### Backward Compatibility

This change is fully backward compatible:

- Default behavior (no configuration) remains unchanged
- Existing deployments continue to work without modification
- The `mode` field defaults to `additive`, preserving transparent proxy behavior

### Forward Compatibility

The configuration structure supports future enhancements:

- `mode: allowlist` can be implemented later to restrict headers
- Additional fields can be added to `HeaderPassthroughConfig` as needed
- The middleware pattern allows for future header transformation features

## Documentation

- Update `thv proxy --help` with `--remote-forward-headers` flag documentation
- Update `thv run --help` with `--remote-forward-headers` flag documentation
- Add example to MCPRemoteProxy CRD reference documentation for `headerForward` field
- Document security considerations for header forwarding
