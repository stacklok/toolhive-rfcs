# RFC-0055: MCPServerEntry CRD for Direct Remote MCP Server Backends

- **Status**: Draft
- **Author(s)**: Juan Antonio Osorio (@jaosorior)
- **Created**: 2026-03-12
- **Last Updated**: 2026-03-12
- **Target Repository**: toolhive
- **Related Issues**: [toolhive#3104](https://github.com/stacklok/toolhive/issues/3104), [toolhive#4109](https://github.com/stacklok/toolhive/issues/4109)

## Summary

Introduce a new `MCPServerEntry` CRD (short name: `mcpentry`) that allows
VirtualMCPServer to connect directly to remote MCP servers without deploying
MCPRemoteProxy infrastructure. MCPServerEntry is a lightweight, pod-less
configuration resource that declares a remote MCP endpoint and belongs to an
MCPGroup, enabling vMCP to reach remote servers with a single auth boundary
and zero additional pods.

## Problem Statement

vMCP currently relies on MCPRemoteProxy (which spawns `thv-proxyrunner` pods)
to reach remote MCP servers. This architecture creates three concrete problems:

### 1. Forced Authentication on Public Remotes (Issue #3104)

MCPRemoteProxy requires OIDC authentication configuration even when vMCP
already handles client authentication at its own boundary. This blocks
unauthenticated public remote MCP servers (e.g., context7, public API
gateways) from being placed behind vMCP without configuring unnecessary
auth on the proxy layer.

### 2. Dual Auth Boundary Confusion (Issue #4109)

MCPRemoteProxy's single `externalAuthConfigRef` field is used for both the
vMCP-to-proxy boundary AND the proxy-to-remote boundary. When vMCP needs
to authenticate to the remote server through the proxy, token exchange
becomes circular or broken because the same auth config serves two
conflicting purposes:

```
Client -> vMCP [boundary 1: client auth]
    -> MCPRemoteProxy [boundary 2: vMCP auth + remote auth on SAME config]
        -> Remote Server
```

The operator cannot express "use auth X for the proxy and auth Y for the
remote" because there is only one `externalAuthConfigRef`.

### 3. Resource Waste

Every remote MCP server behind vMCP requires a full Deployment + Service +
Pod just to make an HTTP call that vMCP could make directly. For
organizations with many remote MCP backends, this creates unnecessary
infrastructure cost and operational overhead.

### Who Is Affected

- **Platform teams** deploying vMCP with remote MCP backends in Kubernetes
- **Product teams** wanting to register external MCP services behind vMCP
- **Organizations** running public or unauthenticated remote MCP servers
  behind vMCP for aggregation

## Goals

- Enable vMCP to connect directly to remote MCP servers without
  MCPRemoteProxy in the path
- Eliminate the dual auth boundary confusion by providing a single,
  unambiguous auth config for the vMCP-to-remote boundary
- Allow unauthenticated remote MCP servers behind vMCP without workarounds
- Deploy zero additional infrastructure (no pods, services, or deployments)
  for remote backend declarations
- Follow existing Kubernetes patterns (groupRef, externalAuthConfigRef)
  consistent with MCPServer

## Non-Goals

- **Deprecating MCPRemoteProxy**: MCPRemoteProxy remains valuable for
  standalone proxy use cases with its own auth middleware, audit logging,
  and observability. MCPServerEntry is specifically for "behind vMCP" use
  cases.
- **Adding health probing from the operator**: The operator controller
  should NOT probe remote URLs. Reachability from the operator pod does not
  imply reachability from the vMCP pod, and probing expands the operator's
  attack surface. Health checking belongs in vMCP's existing runtime
  infrastructure (`healthCheckInterval`, circuit breaker).
- **Cross-namespace references**: MCPServerEntry follows the same
  namespace-scoped patterns as other ToolHive CRDs.
- **Supporting stdio or container-based transports**: MCPServerEntry is
  exclusively for remote HTTP-based MCP servers.
- **CLI mode support**: MCPServerEntry is a Kubernetes-only CRD. CLI mode
  already supports remote backends via direct configuration.

## Proposed Solution

### High-Level Design

Introduce a new `MCPServerEntry` CRD that acts as a catalog entry for a
remote MCP endpoint. The naming follows the Istio `ServiceEntry` pattern,
communicating "this is a catalog entry, not an active workload."

```mermaid
graph TB
    subgraph "Client Layer"
        Client[MCP Client]
    end

    subgraph "Virtual MCP Server"
        InAuth[Incoming Auth<br/>Validates: aud=vmcp]
        Router[Request Router]
        AuthMgr[Backend Auth Manager]
    end

    subgraph "Backend Layer (In-Cluster)"
        MCPServer1[MCPServer: github-mcp<br/>Pod + Service]
        MCPServer2[MCPServer: jira-mcp<br/>Pod + Service]
    end

    subgraph "Backend Layer (Remote)"
        Entry1[MCPServerEntry: context7<br/>No pods - config only]
        Entry2[MCPServerEntry: salesforce<br/>No pods - config only]
    end

    subgraph "External Services"
        Remote1[context7.com/mcp]
        Remote2[mcp.salesforce.com]
    end

    Client -->|Token: aud=vmcp| InAuth
    InAuth --> Router
    Router --> AuthMgr

    AuthMgr -->|In-cluster call| MCPServer1
    AuthMgr -->|In-cluster call| MCPServer2
    AuthMgr -->|Direct HTTPS<br/>+ externalAuthConfig| Remote1
    AuthMgr -->|Direct HTTPS<br/>+ externalAuthConfig| Remote2

    Entry1 -.->|Declares endpoint| Remote1
    Entry2 -.->|Declares endpoint| Remote2

    style Entry1 fill:#fff3e0,stroke:#ff9800
    style Entry2 fill:#fff3e0,stroke:#ff9800
    style MCPServer1 fill:#e3f2fd,stroke:#2196f3
    style MCPServer2 fill:#e3f2fd,stroke:#2196f3
```

The key insight is that MCPServerEntry deploys **no infrastructure**. It is
pure configuration that tells vMCP "there is a remote MCP server at this
URL, use this auth to reach it." VirtualMCPServer discovers MCPServerEntry
resources the same way it discovers MCPServer resources: via `groupRef`.

### Auth Flow Comparison

**Current (with MCPRemoteProxy) - Two boundaries, one config:**

```
Client -> (token: aud=vmcp) -> vMCP [incoming auth boundary]
    -> MCPRemoteProxy [deploys pod]
        externalAuthConfigRef used for BOTH:
          - vMCP-to-proxy auth (boundary 2a)
          - proxy-to-remote auth (boundary 2b)
        -> Remote Server
```

**Proposed (with MCPServerEntry) - One clean boundary:**

```
Client -> (token: aud=vmcp) -> vMCP [incoming auth boundary]
    -> MCPServerEntry: vMCP applies externalAuthConfigRef directly
        -> Remote Server
       (ONE boundary, ONE auth config, no confusion)
```

```mermaid
sequenceDiagram
    participant Client
    participant vMCP as Virtual MCP Server
    participant IDP as Identity Provider
    participant Remote as Remote MCP Server

    Client->>vMCP: MCP Request<br/>Authorization: Bearer token (aud=vmcp)

    Note over vMCP: Validate incoming token<br/>(existing auth middleware)

    Note over vMCP: Look up MCPServerEntry<br/>for target backend

    alt externalAuthConfigRef is set
        vMCP->>IDP: Token exchange request<br/>(per MCPExternalAuthConfig)
        IDP-->>vMCP: Exchanged token (aud=remote-api)
        vMCP->>Remote: Forward request<br/>Authorization: Bearer exchanged-token
    else No auth configured (public remote)
        vMCP->>Remote: Forward request<br/>(no Authorization header)
    end

    Remote-->>vMCP: MCP Response
    vMCP-->>Client: Response
```

### Detailed Design

#### MCPServerEntry CRD

```yaml
apiVersion: toolhive.stacklok.dev/v1alpha1
kind: MCPServerEntry
metadata:
  name: context7
  namespace: default
spec:
  # REQUIRED: URL of the remote MCP server
  remoteURL: https://mcp.context7.com/mcp

  # REQUIRED: Transport protocol
  # +kubebuilder:validation:Enum=streamable-http;sse
  transport: streamable-http

  # REQUIRED: Group membership (unlike MCPServer where it's optional)
  # An MCPServerEntry without a group is dead config - it cannot be
  # discovered by any VirtualMCPServer.
  groupRef: engineering-team

  # OPTIONAL: Auth configuration for reaching the remote server.
  # Omit entirely for unauthenticated public remotes (resolves #3104).
  # Single unambiguous purpose: auth to the remote (resolves #4109).
  externalAuthConfigRef:
    name: salesforce-auth

  # OPTIONAL: Header forwarding configuration.
  # Reuses existing pattern from MCPRemoteProxy (THV-0026).
  headerForward:
    addPlaintextHeaders:
      X-Tenant-ID: "tenant-123"
    addHeadersFromSecrets:
      - headerName: X-API-Key
        valueSecretRef:
          name: remote-api-credentials
          key: api-key

  # OPTIONAL: Custom CA bundle for private remote servers using
  # internal/self-signed certificates.
  caBundleRef:
    name: internal-ca-bundle
    key: ca.crt
```

**Example: Unauthenticated public remote (resolves #3104):**

```yaml
apiVersion: toolhive.stacklok.dev/v1alpha1
kind: MCPServerEntry
metadata:
  name: context7
spec:
  remoteURL: https://mcp.context7.com/mcp
  transport: streamable-http
  groupRef: engineering-team
  # No externalAuthConfigRef - public endpoint, no auth needed
```

**Example: Authenticated remote with token exchange:**

```yaml
apiVersion: toolhive.stacklok.dev/v1alpha1
kind: MCPServerEntry
metadata:
  name: salesforce-mcp
spec:
  remoteURL: https://mcp.salesforce.com
  transport: streamable-http
  groupRef: engineering-team
  externalAuthConfigRef:
    name: salesforce-token-exchange
---
apiVersion: toolhive.stacklok.dev/v1alpha1
kind: MCPExternalAuthConfig
metadata:
  name: salesforce-token-exchange
spec:
  type: tokenExchange
  tokenExchange:
    tokenUrl: https://keycloak.company.com/realms/myrealm/protocol/openid-connect/token
    clientId: salesforce-exchange
    clientSecretRef:
      name: salesforce-oauth
      key: client-secret
    audience: mcp.salesforce.com
    scopes: ["mcp:read", "mcp:write"]
```

**Example: Remote with static header auth:**

```yaml
apiVersion: toolhive.stacklok.dev/v1alpha1
kind: MCPServerEntry
metadata:
  name: internal-api-mcp
spec:
  remoteURL: https://internal-mcp.corp.example.com/mcp
  transport: sse
  groupRef: engineering-team
  headerForward:
    addHeadersFromSecrets:
      - headerName: Authorization
        valueSecretRef:
          name: internal-api-token
          key: bearer-token
  caBundleRef:
    name: corp-ca-bundle
    key: ca.crt
```

#### CRD Type Definitions

The `MCPServerEntry` CRD type is defined in
`cmd/thv-operator/api/v1alpha1/mcpserverentry_types.go`. It follows the
standard kubebuilder pattern with `Spec` and `Status` subresources.

The resource uses the short name `mcpentry` and exposes print columns for
URL, Transport, Group, Ready status, and Age.

**Spec fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `remoteURL` | string | Yes | URL of the remote MCP server. Must match `^https?://`. HTTPS enforced unless `toolhive.stacklok.dev/allow-insecure` annotation is set. |
| `transport` | enum | Yes | MCP transport protocol: `streamable-http` or `sse`. |
| `groupRef` | string | Yes | Name of the MCPGroup this entry belongs to (min length: 1). |
| `externalAuthConfigRef` | object | No | Reference to an MCPExternalAuthConfig in the same namespace. Omit for unauthenticated endpoints. |
| `headerForward` | object | No | Header forwarding configuration. Reuses existing `HeaderForwardConfig` type from MCPRemoteProxy. |
| `caBundleRef` | object | No | Reference to a Secret containing a custom CA certificate bundle for TLS verification. |

**Status fields:**

| Field | Type | Description |
|-------|------|-------------|
| `conditions` | []Condition | Standard Kubernetes conditions (see table below). |
| `observedGeneration` | int64 | Most recent generation observed by the controller. |

**Condition types:**

| Type | Purpose | When Set |
|------|---------|----------|
| `Ready` | Overall readiness | Always |
| `GroupRefValid` | Referenced MCPGroup exists | Always |
| `AuthConfigValid` | Referenced MCPExternalAuthConfig exists | Only when `externalAuthConfigRef` is set |
| `CABundleValid` | Referenced CA bundle exists | Only when `caBundleRef` is set |

There is intentionally **no `RemoteReachable` condition**. The controller
should NOT probe remote URLs because:

1. Reachability from the operator pod does not imply reachability from the
   vMCP pod (different network policies, egress rules, DNS resolution).
2. Probing external URLs from the operator expands its attack surface and
   requires egress network access it may not have.
3. It gives false confidence: a probe succeeding now doesn't mean it will
   succeed when vMCP makes the actual request.
4. vMCP already has health checking infrastructure (`healthCheckInterval`,
   circuit breaker) that operates at the right layer.

#### Status Example

```yaml
status:
  conditions:
    - type: Ready
      status: "True"
      reason: ValidationSucceeded
      message: "MCPServerEntry is valid and ready for discovery"
      lastTransitionTime: "2026-03-12T10:00:00Z"
    - type: GroupRefValid
      status: "True"
      reason: GroupExists
      message: "MCPGroup 'engineering-team' exists"
      lastTransitionTime: "2026-03-12T10:00:00Z"
    - type: AuthConfigValid
      status: "True"
      reason: AuthConfigExists
      message: "MCPExternalAuthConfig 'salesforce-auth' exists"
      lastTransitionTime: "2026-03-12T10:00:00Z"
  observedGeneration: 1
```

#### Component Changes

##### Operator: New CRD and Controller

The MCPServerEntry controller is intentionally simple. It performs
**validation only** and creates **no infrastructure**.

The reconciliation logic:

1. Fetches the MCPServerEntry resource (ignores not-found for deletions).
2. Validates that the referenced MCPGroup exists in the same namespace.
   Sets `GroupRefValid` condition accordingly.
3. If `externalAuthConfigRef` is set, validates that the referenced
   MCPExternalAuthConfig exists. Sets `AuthConfigValid` condition.
4. Validates the HTTPS requirement: if `remoteURL` does not use HTTPS,
   the controller checks for the `toolhive.stacklok.dev/allow-insecure`
   annotation. Without it, the `Ready` condition is set to false with
   reason `InsecureURL`.
5. If all validations pass, sets `Ready` to true with reason
   `ValidationSucceeded`.

The controller watches MCPGroup and MCPExternalAuthConfig resources via
`EnqueueRequestsFromMapFunc` handlers, so that changes to referenced
resources trigger re-validation of affected MCPServerEntry resources.

No finalizers are needed because MCPServerEntry creates no infrastructure
to clean up.

##### Operator: MCPGroup Controller Update

The MCPGroup controller must be updated to watch MCPServerEntry resources
in addition to MCPServer resources, so that `status.servers` and
`status.serverCount` reflect both types of backends in the group.

##### Operator: VirtualMCPServer Controller Update

**Static mode (`outgoingAuth.source: inline`):** The operator generates
the ConfigMap that vMCP reads at startup. This ConfigMap must now include
MCPServerEntry backends alongside MCPServer backends.

The controller discovers MCPServerEntry resources in the group and
serializes them as remote backend entries in the ConfigMap:

```yaml
# Generated ConfigMap content
backends:
  # From MCPServer resources (existing)
  - name: github-mcp
    url: http://github-mcp.default.svc:8080
    transport: sse
    type: container
    auth:
      type: token_exchange
      # ...

  # From MCPServerEntry resources (new)
  - name: context7
    url: https://mcp.context7.com/mcp
    transport: streamable-http
    type: entry   # New backend type
    # No auth - public endpoint

  - name: salesforce-mcp
    url: https://mcp.salesforce.com
    transport: streamable-http
    type: entry
    auth:
      type: token_exchange
      # ...
```

##### vMCP: Backend Type and Discovery

A new `BackendTypeEntry` constant (`"entry"`) is added to
`pkg/vmcp/types.go` alongside the existing `BackendTypeContainer` and
`BackendTypeProxy`.

The `ListWorkloadsInGroup()` function in `pkg/vmcp/workloads/k8s.go` is
extended to discover MCPServerEntry resources in addition to MCPServer
resources. For each MCPServerEntry in the group, vMCP:

1. Lists MCPServerEntry resources filtered by `spec.groupRef`.
2. Converts each entry to an internal `Backend` struct using the entry's
   `remoteURL`, `transport`, and name.
3. Resolves `externalAuthConfigRef` if set (using existing auth resolution
   logic).
4. Resolves `headerForward` configuration if set.
5. Resolves `caBundleRef` if set (fetching the CA certificate from the
   referenced Secret).
6. Appends the resulting backends alongside MCPServer-sourced backends.

##### vMCP: HTTP Client for External TLS

Backends of type `entry` connect to external URLs over HTTPS. The vMCP
HTTP client in `pkg/vmcp/client/client.go` must be updated to:

1. Use the system CA certificate pool by default (for public CAs).
2. Optionally append a custom CA bundle from `caBundleRef` (for private
   CAs) to the system pool.
3. Enforce a minimum TLS version of 1.2.
4. Apply the resolved `externalAuthConfigRef` credentials directly to
   outgoing requests.

##### vMCP: Dynamic Mode Reconciler Update

For dynamic mode (`outgoingAuth.source: discovered`), the reconciler
infrastructure from THV-0014 must be extended to watch MCPServerEntry
resources.

The `MCPServerEntryWatcher` follows the same reconciler pattern as the
existing `MCPServerWatcher` from THV-0014. It holds a reference to the
`DynamicRegistry` and the target `groupRef`. On reconciliation:

1. If the resource is deleted (not found), it removes the backend from the
   registry by namespaced name.
2. If the entry's `groupRef` doesn't match the watcher's group, it removes
   the backend (handles group reassignment).
3. Otherwise, it converts the MCPServerEntry to a `Backend` struct
   (resolving auth, headers, CA bundle) and upserts it into the registry.

The watcher also watches MCPExternalAuthConfig and Secret resources via
`EnqueueRequestsFromMapFunc` handlers, so changes to referenced auth
configs or secrets trigger re-reconciliation of affected entries.

##### vMCP: Static Config Parser Update

The static config parser must be updated to deserialize `type: entry`
backends from the ConfigMap and create appropriate HTTP clients with
external TLS support.

## Security Considerations

### Threat Model

| Threat | Description | Mitigation |
|--------|-------------|------------|
| Man-in-the-middle on remote connection | Attacker intercepts vMCP-to-remote traffic | HTTPS required by default; custom CA bundles for private CAs |
| Credential exposure in CRD spec | Auth secrets visible in CRD manifest | Credentials stored in K8s Secrets, referenced via `externalAuthConfigRef` and `headerForward.addHeadersFromSecrets`; never inline in CRD spec |
| SSRF via remoteURL | Operator configures URL pointing to internal services | Mitigated by RBAC (only authorized users create MCPServerEntry); annotation required for non-HTTPS; NetworkPolicy should restrict vMCP egress |
| Auth config confusion (existing issue) | Dual-boundary auth leading to wrong tokens sent to wrong endpoints | Eliminated: MCPServerEntry has exactly one auth boundary with one purpose |
| Operator probing external URLs | Controller making network requests to untrusted URLs | Eliminated: controller performs validation only, no network probing |

### Authentication and Authorization

- **No new auth primitives**: MCPServerEntry reuses the existing
  `MCPExternalAuthConfig` CRD and `externalAuthConfigRef` pattern.
- **Single boundary**: vMCP's incoming auth validates client tokens.
  MCPServerEntry's `externalAuthConfigRef` handles outgoing auth to
  the remote. These are cleanly separated.
- **RBAC**: Standard Kubernetes RBAC controls who can create/modify
  MCPServerEntry resources. This enables fine-grained access: platform
  teams manage VirtualMCPServer, product teams register MCPServerEntry
  backends.
- **No privilege escalation**: MCPServerEntry grants no additional
  permissions beyond what the referenced MCPExternalAuthConfig already
  provides.

### Data Security

- **In transit**: HTTPS required for remote connections (with annotation
  escape hatch for development).
- **At rest**: No sensitive data stored in MCPServerEntry spec. Auth
  credentials are in K8s Secrets, referenced indirectly.
- **CA bundles**: Custom CA certificates referenced via `caBundleRef`,
  stored in K8s Secrets/ConfigMaps with standard K8s encryption at rest.

### Input Validation

- **remoteURL**: Must match `^https?://` pattern. HTTPS enforced unless
  annotation override. Validated by both CRD CEL rules and controller
  reconciliation.
- **transport**: Enum validation (`streamable-http` or `sse`).
- **groupRef**: Required, validated to reference an existing MCPGroup.
- **externalAuthConfigRef**: When set, validated to reference an existing
  MCPExternalAuthConfig.
- **headerForward**: Uses the same restricted header blocklist and
  validation as MCPRemoteProxy (THV-0026).

### Secrets Management

- MCPServerEntry follows the same secret access patterns as MCPServer:
  - **Dynamic mode**: vMCP reads secrets at runtime via K8s API
    (namespace-scoped RBAC).
  - **Static mode**: Operator mounts secrets as environment variables.
- Secret rotation follows existing patterns:
  - **Dynamic mode**: Watch-based propagation, no pod restart needed.
  - **Static mode**: Requires pod restart (Deployment rollout).

### Audit and Logging

- vMCP's existing audit middleware logs all requests routed to
  MCPServerEntry backends, including user identity and target tool.
- The operator controller logs validation results (group existence,
  auth config existence) at standard log levels.
- No sensitive data (URLs with credentials, auth tokens) is logged.

### Mitigations

1. **HTTPS enforcement**: Default requires HTTPS; annotation override
   requires explicit operator action.
2. **No network probing**: Controller never connects to remote URLs.
3. **Single auth boundary**: Eliminates dual-boundary confusion.
4. **Existing patterns**: Reuses battle-tested secret access, RBAC,
   and auth patterns from MCPServer.
5. **NetworkPolicy recommendation**: Documentation recommends restricting
   vMCP pod egress to known remote endpoints.
6. **No new attack surface**: Zero additional pods deployed.

## Alternatives Considered

### Alternative 1: Add `remoteServerRefs` to VirtualMCPServer Spec

Embed remote server configuration directly in the VirtualMCPServer CRD.

```yaml
kind: VirtualMCPServer
spec:
  groupRef:
    name: engineering-team
  remoteServerRefs:
    - name: context7
      remoteURL: https://mcp.context7.com/mcp
      transport: streamable-http
    - name: salesforce
      remoteURL: https://mcp.salesforce.com
      transport: streamable-http
      externalAuthConfigRef:
        name: salesforce-auth
```

**Pros:**
- No new CRD needed
- Simple for small deployments

**Cons:**
- Violates separation of concerns: VirtualMCPServer manages aggregation,
  not backend declaration
- Breaks the `groupRef` discovery pattern: some backends discovered via
  group, others embedded inline
- Bloats VirtualMCPServer spec
- Prevents independent lifecycle management: adding/removing a remote
  backend requires editing the VirtualMCPServer, which may trigger
  reconciliation of unrelated configuration
- Prevents fine-grained RBAC: only VirtualMCPServer editors can manage
  remote backends

**Why not chosen:** Inconsistent with existing patterns and prevents the
RBAC separation that makes MCPServerEntry valuable (platform teams manage
vMCP, product teams register backends).

### Alternative 2: Extend MCPServer with Remote Mode

Add a `mode: remote` field to the existing MCPServer CRD.

```yaml
kind: MCPServer
spec:
  mode: remote
  remoteURL: https://mcp.context7.com/mcp
  transport: streamable-http
  groupRef: engineering-team
```

**Pros:**
- No new CRD
- Reuses existing MCPServer controller infrastructure

**Cons:**
- MCPServer is fundamentally a container workload resource. Adding a
  "don't deploy anything" mode creates confusing semantics: `spec.image`
  becomes optional, `spec.resources` is meaningless, status conditions
  designed for pod lifecycle don't apply.
- Controller logic becomes complex with conditional paths for
  container vs remote modes.
- Existing MCPServer watchers (MCPGroup controller, VirtualMCPServer
  controller) would need to handle both modes, adding complexity.
- The controller currently creates Deployments, Services, and ConfigMaps.
  Adding a mode that creates none of these is a significant semantic
  change.

**Why not chosen:** Overloading MCPServer with remote-mode semantics
increases complexity and confusion. A separate CRD with clear "this is
configuration only" semantics is cleaner.

### Alternative 3: Configure Remote Backends Only in vMCP Config

Handle remote backends entirely in vMCP's configuration (ConfigMap or
runtime discovery) without a CRD.

**Pros:**
- No CRD changes needed
- Simpler operator

**Cons:**
- No Kubernetes-native resource to represent remote backends
- No status reporting, no `kubectl get` visibility
- No RBAC for who can manage remote backends
- Breaks the pattern where all backends are discoverable via `groupRef`
- MCPGroup status cannot reflect remote backends

**Why not chosen:** Loses Kubernetes-native management, visibility, and
access control.

## Compatibility

### Backward Compatibility

MCPServerEntry is a purely additive change:

- **No changes to existing CRDs**: MCPServer, MCPRemoteProxy,
  VirtualMCPServer, MCPGroup, and MCPExternalAuthConfig are unchanged.
- **No changes to existing behavior**: VirtualMCPServer continues to
  discover MCPServer resources via `groupRef`. MCPServerEntry adds a
  new discovery source alongside the existing one.
- **MCPRemoteProxy still works**: Organizations using MCPRemoteProxy
  can continue to do so. MCPServerEntry is an alternative, not a
  replacement.
- **No migration required**: Existing deployments work without
  modification after the upgrade.

### Forward Compatibility

- **Extensibility**: The `MCPServerEntrySpec` can be extended with
  additional fields (e.g., rate limiting, tool filtering) without
  breaking changes.
- **API versioning**: Starts at `v1alpha1`, consistent with all other
  ToolHive CRDs.
- **Future deprecation path**: If MCPRemoteProxy use cases are eventually
  subsumed, MCPServerEntry provides a clean migration target.

## Implementation Plan

### Phase 1: CRD and Controller

1. Define `MCPServerEntry` CRD types
2. Implement validation-only controller
3. Generate CRD manifests
4. Update MCPGroup controller to watch MCPServerEntry resources
5. Add unit tests for controller validation logic

### Phase 2: Static Mode Integration

1. Update VirtualMCPServer controller to discover MCPServerEntry resources
   in the group
2. Update ConfigMap generation to include entry-type backends
3. Update vMCP static config parser to deserialize entry backends
4. Add `BackendTypeEntry` to vMCP types
5. Implement external TLS transport creation for entry backends
6. Integration tests with envtest

### Phase 3: Dynamic Mode Integration

1. Create MCPServerEntry reconciler for vMCP's dynamic registry
2. Register watcher in the K8s manager alongside MCPServerWatcher
3. Update workload discovery to include MCPServerEntry
4. Resolve auth configs for entry backends at runtime
5. Integration tests for dynamic discovery of entry backends

### Phase 4: Documentation and E2E

1. CRD reference documentation
2. User guide with examples (public remote, authenticated remote,
   private CA)
3. MCPRemoteProxy vs MCPServerEntry comparison guide
4. E2E Chainsaw tests for full lifecycle
5. E2E tests for mixed MCPServer + MCPServerEntry groups

### Dependencies

- THV-0014 (K8s-Aware vMCP) for dynamic mode support
- THV-0026 (Header Passthrough) for `headerForward` field reuse
- Existing MCPExternalAuthConfig CRD for auth configuration

## Testing Strategy

### Unit Tests

- Controller validation: groupRef exists, authConfigRef exists, HTTPS
  enforcement, annotation override
- CRD type serialization/deserialization
- Backend conversion from MCPServerEntry to internal Backend struct
- External TLS transport creation with and without custom CA bundles
- Static config parsing with entry-type backends

### Integration Tests (envtest)

- MCPServerEntry controller reconciliation with real API server
- VirtualMCPServer ConfigMap generation including entry backends
- MCPGroup status update with mixed MCPServer + MCPServerEntry members
- Dynamic mode: MCPServerEntry watcher reconciliation
- Auth config resolution for entry backends
- Secret change propagation to entry backends

### End-to-End Tests (Chainsaw)

- Full lifecycle: create MCPGroup, create MCPServerEntry, create
  VirtualMCPServer, verify vMCP routes to remote backend
- Mixed group: MCPServer (container) + MCPServerEntry (remote) in same
  group
- Unauthenticated public remote behind vMCP
- Authenticated remote with token exchange
- MCPServerEntry deletion removes backend from vMCP
- CA bundle configuration for private remotes

### Security Tests

- Verify HTTPS enforcement (HTTP URL without annotation is rejected)
- Verify RBAC separation (entry creation requires correct permissions)
- Verify no network probing from controller
- Verify secret values are not logged

## Documentation

- **CRD Reference**: Auto-generated CRD documentation for MCPServerEntry
  fields, validation rules, and status conditions
- **User Guide**: How to add remote MCP backends to vMCP using
  MCPServerEntry, with examples for common scenarios
- **Comparison Guide**: When to use MCPRemoteProxy vs MCPServerEntry:

  | Feature | MCPRemoteProxy | MCPServerEntry |
  |---------|---------------|----------------|
  | Deploys pods | Yes (proxy pod) | No |
  | Own auth middleware | Yes (oidcConfig, authzConfig) | No |
  | Own audit logging | Yes | No (uses vMCP's) |
  | Standalone use | Yes | No (only via VirtualMCPServer) |
  | GroupRef support | Yes (optional) | Yes (required) |
  | Primary use case | Standalone proxy with full observability | Backend declaration for vMCP |

- **Architecture Documentation**: Update `docs/arch/10-virtual-mcp-architecture.md`
  to describe MCPServerEntry as a backend type

## Open Questions

1. **Should `remoteURL` strictly require HTTPS?**
   Recommendation: Yes, with annotation override
   (`toolhive.stacklok.dev/allow-insecure: "true"`) for development.
   This prevents accidental plaintext credential transmission while
   allowing local development workflows.

2. **Should the CRD support custom CA bundles for private remote servers?**
   Recommendation: Yes, via `caBundleRef` field referencing a Secret or
   ConfigMap. This is essential for enterprises with internal CAs. The
   current design includes this field.

3. **Should there be a `disabled` field for temporarily removing an entry
   from discovery without deleting it?**
   This could be useful for maintenance windows or incident response.
   However, it adds complexity and can be achieved by removing the
   `groupRef` temporarily. Defer to post-implementation feedback.

4. **Should MCPServerEntry support `toolConfigRef` for tool filtering?**
   MCPRemoteProxy supports tool filtering via `toolConfigRef`.
   VirtualMCPServer also has its own tool filtering/override configuration
   in `spec.aggregation.tools`. For MCPServerEntry, tool filtering should
   be configured at the VirtualMCPServer level (where it already exists)
   rather than duplicating it on the entry. Defer unless there is a clear
   use case for entry-level filtering.

## References

- [THV-0008: Virtual MCP Server](./THV-0008-virtual-mcp-server.md) -
  VirtualMCPServer design, auth boundaries, capability aggregation
- [THV-0009: Remote MCP Server Proxy](./THV-0009-remote-mcp-proxy.md) -
  MCPRemoteProxy CRD design
- [THV-0010: MCPGroup CRD](./THV-0010-kubernetes-mcpgroup-crd.md) -
  Group-based backend discovery pattern
- [THV-0014: K8s-Aware vMCP](./THV-0014-vmcp-k8s-aware-refactor.md) -
  Dynamic vs static discovery modes, reconciler infrastructure
- [THV-0026: Header Passthrough](./THV-0026-header-passthrough.md) -
  `headerForward` configuration pattern
- [Istio ServiceEntry](https://istio.io/latest/docs/reference/config/networking/service-entry/) -
  Naming pattern inspiration
- [toolhive#3104](https://github.com/stacklok/toolhive/issues/3104) -
  MCPRemoteProxy forces OIDC auth on public remotes behind vMCP
- [toolhive#4109](https://github.com/stacklok/toolhive/issues/4109) -
  Dual auth boundary confusion with externalAuthConfigRef

---

## RFC Lifecycle

<!-- This section is maintained by RFC reviewers -->

### Review History

| Date | Reviewer | Decision | Notes |
|------|----------|----------|-------|
| 2026-03-12 | @jaosorior | Draft | Initial submission |

### Implementation Tracking

| Repository | PR | Status |
|------------|-----|--------|
| toolhive | TBD | Not started |
