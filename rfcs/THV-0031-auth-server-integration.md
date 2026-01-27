# RFC-0031: Embedded Authorization Server in Proxy Runner

- **Status**: Draft
- **Author(s)**: @tgrunnagle
- **Created**: 2026-01-27
- **Last Updated**: 2026-01-27
- **Target Repository**: toolhive
- **Related Issues**: #195

## Summary

Integrate the ToolHive authorization server (`pkg/authserver/`) into the proxy runner process for Kubernetes deployments. This enables MCP servers to have an embedded OAuth2/OIDC authorization server that authenticates users via upstream identity providers, issuing tokens that can be used to access the MCP server.

## Problem Statement

- **Current behavior**: MCP servers in Kubernetes require external authentication setup. The existing `MCPExternalAuthConfig` supports token exchange, header injection, and bearer tokens, but not a full OAuth2/OIDC authorization server for user authentication flows.

- **Who is affected**: Platform operators deploying MCP servers that need to authenticate end users via corporate identity providers (Okta, Azure AD, etc.) before granting access to MCP tools.

- **Why worth solving**: Enables secure, standards-compliant user authentication for MCP servers without requiring separate infrastructure deployment. Users can authenticate once via their corporate IDP and receive tokens to access MCP server capabilities.

## Goals

- Add a new `embeddedAuthServer` type to `MCPExternalAuthConfig` CRD
- Support signing key rotation via list of secret references (first is active, rest are on JWKS for verification)
- Support HMAC secret rotation for internal token encryption
- Support upstream identity providers (OIDC with discovery, OAuth2 with explicit endpoints)
- Mount signing keys and HMAC secrets as volumes (not environment variables) for better security
- Run the authorization server in-process with the proxy runner
- Expose OAuth2/OIDC endpoints (`/oauth/*`, `/.well-known/*`) on the proxy

## Non-Goals

- **Multi-replica support**: Memory-based storage limits to single replica (existing constraint)
- **Standalone auth server deployment**: This RFC focuses on in-process integration only
- **Multiple upstream IDPs**: Initially only one upstream provider is supported (error if multiple configured)
- **Local CLI support**: Kubernetes deployments only

---

## Proposed Solution

### High-Level Design

```mermaid
flowchart TB
    subgraph "Kubernetes Cluster"
        subgraph "Proxy Runner Pod"
            subgraph "Proxy Runner Container"
                subgraph "HTTP Server :8080"
                    AuthRoutes["/oauth/*, /.well-known/*<br/>(Auth Server Handler)"]
                    MCPRoutes["/mcp, /sse<br/>(MCP Proxy Handler)"]
                end
                AuthServer["Embedded Auth Server<br/>(fosite, storage, upstream)"]
                MCPProxy["MCP Transport<br/>(SSE/Streamable-HTTP)"]
            end
        end

        subgraph "Mounted Secrets"
            SK[Signing Keys]
            HS[HMAC Secrets]
        end

        subgraph "Mounted ConfigMaps"
            RC[RunConfig]
        end

        CS[Upstream Client Secret<br/>via Env Var]
    end

    UP[Upstream IDP<br/>Okta/Azure AD/etc.]
    Client[MCP Client]

    AuthRoutes --> AuthServer
    MCPRoutes --> MCPProxy

    Client -->|1. GET /oauth/authorize| AuthRoutes
    AuthServer -->|2. Redirect to IDP| UP
    UP -->|3. Callback with code| AuthRoutes
    AuthServer -->|4. Exchange code| UP
    AuthServer -->|5. Issue JWT access token| Client
    Client -->|6. MCP request + Bearer token| MCPRoutes

    SK -.->|Volume: /etc/toolhive/authserver/keys/| AuthServer
    HS -.->|Volume: /etc/toolhive/authserver/hmac/| AuthServer
    CS -.->|TOOLHIVE_UPSTREAM_CLIENT_SECRET| AuthServer
    RC -.->|Volume: /etc/runconfig/| MCPProxy
```

**Architecture Notes:**
- **Single container**: The proxy runner is one container running one HTTP server
- **Shared HTTP server**: Auth server routes (`/oauth/*`, `/.well-known/*`) and MCP proxy routes (`/mcp`, `/sse`) are mounted on the same HTTP server
- **Route priority**: Auth server routes are registered first, MCP proxy routes handle remaining paths
- **In-process**: Embedded auth server runs as Go code within the proxy runner process (not a separate sidecar)

### Detailed Design

#### Component Changes

**1. New CRD Type: `embeddedAuthServer`**

Add to [mcpexternalauthconfig_types.go](cmd/thv-operator/api/v1alpha1/mcpexternalauthconfig_types.go):

```go
const (
    ExternalAuthTypeEmbeddedAuthServer ExternalAuthType = "embeddedAuthServer"
)
```

**2. New Controller Utility: Volume Generation**

New file [cmd/thv-operator/pkg/controllerutil/authserver.go](cmd/thv-operator/pkg/controllerutil/authserver.go):
- `GenerateAuthServerVolumes()` - Creates volume and mount configs for signing keys and HMAC secrets
  - Follows pgpass pattern from [registryapi/podtemplatespec.go](cmd/thv-operator/pkg/registryapi/podtemplatespec.go)
  - Uses `corev1.SecretVolumeSource` with `Items[].KeyToPath` mapping
  - Sets `DefaultMode: 0400` for restrictive permissions
- `GenerateAuthServerEnvVars()` - Creates env var for upstream client secret
  - Follows pattern from [tokenexchange.go](cmd/thv-operator/pkg/controllerutil/tokenexchange.go) and [oidc.go](cmd/thv-operator/pkg/controllerutil/oidc.go)
  - Uses `SecretKeyRef` for `TOOLHIVE_UPSTREAM_CLIENT_SECRET`
- `AddEmbeddedAuthServerConfigOptions()` - Adds auth server config to runner options

**3. New Runner Component: Embedded Auth Server Wrapper**

New file [pkg/runner/authserver.go](pkg/runner/authserver.go):
- `EmbeddedAuthServer` struct wrapping auth server lifecycle
- `NewEmbeddedAuthServer()` - Initializes from RunConfig
- `Handler()` - Returns HTTP handler for routes
- `Close()` - Cleanup resources

**4. Runner Integration**

Modify [pkg/runner/runner.go](pkg/runner/runner.go):
- Start embedded auth server in `Run()` if configured
- Mount auth server handler on transport config
- Cleanup on shutdown

**5. Transport Integration**

Modify [pkg/transport/http.go](pkg/transport/http.go):
- Add `AuthServerHandler` field to transport config
- Mount auth server routes before MCP proxy routes

#### API Changes

**New CRD Fields in MCPExternalAuthConfigSpec:**

```go
type EmbeddedAuthServerConfig struct {
    // Issuer is the issuer URL (must be https, no trailing slash)
    // +kubebuilder:validation:Required
    // +kubebuilder:validation:Pattern=`^https://[^\s/]+[^\s/]$`
    Issuer string `json:"issuer"`

    // SigningKeys for JWT signing (first is active, rest for JWKS rotation)
    // +kubebuilder:validation:Required
    // +kubebuilder:validation:MinItems=1
    // +kubebuilder:validation:MaxItems=5
    SigningKeys []SigningKeySecretRef `json:"signingKeys"`

    // HMACSecretRefs for internal token encryption (first is current, rest rotated)
    // +kubebuilder:validation:Required
    // +kubebuilder:validation:MinItems=1
    HMACSecretRefs []SecretKeyRef `json:"hmacSecretRefs"`

    // TokenLifespans configuration (optional, has defaults)
    // +optional
    TokenLifespans *TokenLifespanConfig `json:"tokenLifespans,omitempty"`

    // UpstreamProviders (currently only ONE supported; error if multiple)
    // +kubebuilder:validation:Required
    // +kubebuilder:validation:MinItems=1
    UpstreamProviders []UpstreamProviderConfig `json:"upstreamProviders"`

    // AllowedAudiences for RFC 8707 resource parameter validation
    // +optional
    AllowedAudiences []string `json:"allowedAudiences,omitempty"`
}

type SigningKeySecretRef struct {
    SecretRef SecretKeyRef `json:"secretRef"`
}

type TokenLifespanConfig struct {
    AccessTokenLifespan  string `json:"accessTokenLifespan,omitempty"`  // Default: 15m
    RefreshTokenLifespan string `json:"refreshTokenLifespan,omitempty"` // Default: 7d
    AuthCodeLifespan     string `json:"authCodeLifespan,omitempty"`     // Default: 5m
}

type UpstreamProviderConfig struct {
    Name   string              `json:"name"`
    Type   string              `json:"type"` // "oidc" or "oauth2"
    OIDC   *OIDCUpstreamConfig   `json:"oidc,omitempty"`
    OAuth2 *OAuth2UpstreamConfig `json:"oauth2,omitempty"`
}

type OIDCUpstreamConfig struct {
    IssuerURL       string        `json:"issuerUrl"`
    ClientID        string        `json:"clientId"`
    ClientSecretRef *SecretKeyRef `json:"clientSecretRef,omitempty"`
    RedirectURI     string        `json:"redirectUri"`
    Scopes          []string      `json:"scopes,omitempty"`
}

type OAuth2UpstreamConfig struct {
    AuthorizationEndpoint string        `json:"authorizationEndpoint"`
    TokenEndpoint         string        `json:"tokenEndpoint"`
    UserInfoEndpoint      string        `json:"userInfoEndpoint,omitempty"`
    ClientID              string        `json:"clientId"`
    ClientSecretRef       *SecretKeyRef `json:"clientSecretRef,omitempty"`
    RedirectURI           string        `json:"redirectUri"`
    Scopes                []string      `json:"scopes,omitempty"`
}
```

#### Configuration Changes

**RunConfig Extension** ([pkg/runner/config.go](pkg/runner/config.go)):

```go
type RunConfig struct {
    // ... existing fields ...
    EmbeddedAuthServer *EmbeddedAuthServerConfig `json:"embedded_auth_server,omitempty"`
}

type EmbeddedAuthServerConfig struct {
    Enabled              bool                    `json:"enabled"`
    Issuer               string                  `json:"issuer"`
    KeysDir              string                  `json:"keys_dir"`              // Volume mount path
    SigningKeyFile       string                  `json:"signing_key_file"`       // Filename (relative to KeysDir)
    FallbackKeyFiles     []string                `json:"fallback_key_files,omitempty"` // Filenames for rotation
    HMACSecretFiles      []string                `json:"hmac_secret_files"`      // Volume mount paths
    AccessTokenLifespan  time.Duration           `json:"access_token_lifespan"`
    RefreshTokenLifespan time.Duration           `json:"refresh_token_lifespan"`
    AuthCodeLifespan     time.Duration           `json:"auth_code_lifespan"`
    AllowedAudiences     []string                `json:"allowed_audiences,omitempty"`
    UpstreamProvider     *UpstreamProviderConfig `json:"upstream_provider"`
}

type UpstreamProviderConfig struct {
    Type                  string   `json:"type"` // "oidc" or "oauth2"
    IssuerURL             string   `json:"issuer_url,omitempty"`             // OIDC only
    AuthorizationEndpoint string   `json:"authorization_endpoint,omitempty"` // OAuth2 only
    TokenEndpoint         string   `json:"token_endpoint,omitempty"`         // OAuth2 only
    UserInfoEndpoint      string   `json:"userinfo_endpoint,omitempty"`      // OAuth2 only
    ClientID              string   `json:"client_id"`
    ClientSecretEnvVar    string   `json:"client_secret_env_var,omitempty"`  // Env var name (follows existing pattern)
    RedirectURI           string   `json:"redirect_uri"`
    Scopes                []string `json:"scopes,omitempty"`
}
```

**Example CRD:**

```yaml
apiVersion: toolhive.stacklok.dev/v1alpha1
kind: MCPExternalAuthConfig
metadata:
  name: my-auth-config
  namespace: mcp-servers
spec:
  type: embeddedAuthServer
  embeddedAuthServer:
    issuer: "https://mcp.example.com"

    signingKeys:
      - secretRef:
          name: auth-signing-key
          key: private.pem
      - secretRef:
          name: auth-signing-key-old
          key: private.pem

    hmacSecretRefs:
      - name: auth-hmac-secrets
        key: current-hmac
      - name: auth-hmac-secrets-old
        key: previous-hmac

    tokenLifespans:
      accessTokenLifespan: "15m"
      refreshTokenLifespan: "7d"
      authCodeLifespan: "5m"

    upstreamProviders:
      - name: okta
        type: oidc
        oidc:
          issuerUrl: "https://dev-123456.okta.com"
          clientId: "0oa1234567890abcdef"
          clientSecretRef:
            name: okta-client-secret
            key: secret
          redirectUri: "https://mcp.example.com/oauth/callback"
          scopes: ["openid", "profile", "email"]

    allowedAudiences:
      - "https://api.example.com"
```

#### Data Model Changes

**Secret Mounting Strategy (follows existing precedent):**

ToolHive uses two patterns for secrets in K8s deployments:

1. **Environment Variables** (via `SecretKeyRef`): For simple string secrets
   - Examples: `TOOLHIVE_TOKEN_EXCHANGE_CLIENT_SECRET`, `TOOLHIVE_OIDC_CLIENT_SECRET`
   - Pattern: [tokenexchange.go](cmd/thv-operator/pkg/controllerutil/tokenexchange.go), [oidc.go](cmd/thv-operator/pkg/controllerutil/oidc.go)

2. **Secret Volumes**: For file-based secrets (PEM files, credential files)
   - Precedent: pgpass mounting in [registryapi/podtemplatespec.go](cmd/thv-operator/pkg/registryapi/podtemplatespec.go)
   - Uses `corev1.SecretVolumeSource` with `Items[].KeyToPath` mapping

For embedded auth server:
- **Signing keys** (PEM files): Must use volume mounts - multi-line binary content too large for env vars
- **HMAC secrets**: Use volume mounts for consistency with signing keys
- **Upstream client secret**: Use environment variable (simple string, follows existing pattern)

**Volume Mount Configuration:**

```go
// Following pgpass precedent from registryapi/podtemplatespec.go
volumes = append(volumes, corev1.Volume{
    Name: "signing-key-0",
    VolumeSource: corev1.VolumeSource{
        Secret: &corev1.SecretVolumeSource{
            SecretName: secretRef.Name,
            Items: []corev1.KeyToPath{{
                Key:  secretRef.Key,
                Path: "key-0.pem",
            }},
            DefaultMode: ptr.To(int32(0400)), // Read-only for owner
        },
    },
})
```

**Mount Paths:**

| Secret Type | Mount Method | Path/Variable |
|-------------|--------------|---------------|
| Signing Key (index N) | Volume | `/etc/toolhive/authserver/keys/key-{N}.pem` |
| HMAC Secret (index N) | Volume | `/etc/toolhive/authserver/hmac/hmac-{N}` |
| Upstream Client Secret | Env Var | `TOOLHIVE_UPSTREAM_CLIENT_SECRET` |

All secret volumes mounted with `0400` permissions (read-only for owner), following the pgpass precedent.

---

## Security Considerations

### Threat Model

| Threat | Description | Likelihood | Impact |
|--------|-------------|------------|--------|
| Compromised proxy runner | Attacker gains access to process memory | Medium | Access to upstream tokens for ONE MCP server |
| Compromised auth server (standalone) | Attacker gains access to centralized auth service | Medium | Access to upstream tokens for ALL MCP servers |
| Signing key compromise | Attacker can forge JWT tokens | Low | Can impersonate any user to this MCP server |
| HMAC secret compromise | Attacker can decrypt internal tokens | Low | Can forge auth codes and refresh tokens |

### Authentication and Authorization

- Users authenticate via upstream IDP (OIDC/OAuth2)
- Auth server issues JWT access tokens with claims: `sub`, `aud`, `client_id`, `tsid` (token session ID)
- MCP server validates tokens using existing auth middleware
- PKCE (S256) required for all authorization code flows

### Data Security

- **Signing keys**: Asymmetric (RSA/ECDSA/EdDSA), private key never exposed
- **HMAC secrets**: Symmetric, used only for internal token encryption
- **Upstream tokens**: Stored in memory only, lost on pod restart
- **Data in transit**: HTTPS required for issuer URL

### Input Validation

- Issuer URL validated (https, no trailing slash)
- Redirect URIs validated per RFC 8252 (loopback only for DCR)
- Audience URIs validated per RFC 8707
- PKCE challenge method restricted to S256 only

### Secrets Management

| Secret | Mount Method | Env Var / Path | Rotation Support |
|--------|--------------|----------------|------------------|
| Signing keys | Volume (SecretVolumeSource) | `/etc/toolhive/authserver/keys/` | Yes, via list (first active, rest for verification) |
| HMAC secrets | Volume (SecretVolumeSource) | `/etc/toolhive/authserver/hmac/` | Yes, via list (first current, rest for decryption) |
| Upstream client secret | Env Var (SecretKeyRef) | `TOOLHIVE_UPSTREAM_CLIENT_SECRET` | Manual (update secret, restart pod) |

**Alignment with existing patterns:**
- Upstream client secret uses env var pattern (same as `TOOLHIVE_OIDC_CLIENT_SECRET`)
- Signing keys use volume pattern (same as pgpass in registry API)
- HMAC secrets use volume pattern for consistency with signing keys

### Audit and Logging

- OAuth flow events logged at INFO level (code exchange, token issuance)
- Failed authentication attempts logged at WARN level
- No sensitive data (tokens, secrets) logged

### Mitigations

| Threat | Mitigation |
|--------|------------|
| Compromised proxy runner | **Per-workload isolation** - each MCP server has its own auth server; blast radius limited to one workload |
| Key compromise | **Key rotation** - old keys removed after token expiration; JWKS endpoint updated atomically |
| Memory exposure | **Ephemeral storage** - tokens lost on pod restart; no persistent state |
| Secret exposure | **Volume mounts** - secrets not in environment variables; restricted file permissions (0400) |

---

## Alternatives Considered

### Alternative 1: Standalone Auth Server Deployment

- **Description**: Deploy auth server as a separate K8s Deployment, accessed by proxy runners via network
- **Pros**:
  - Token isolation (upstream refresh tokens never reach proxy runner)
  - Can serve multiple MCP servers
- **Cons**:
  - Requires mTLS between auth server and proxy runners
  - Token exchange endpoint becomes additional attack surface
  - Centralized auth server = larger blast radius if compromised (ALL MCP servers' tokens)
  - Higher operational complexity
- **Why not chosen**: Worse security profile (centralized compromise affects all workloads) and higher operational burden outweigh token isolation benefit

### Alternative 2: External Auth Server Integration

- **Description**: Integrate with external OAuth2/OIDC provider directly (not self-hosted)
- **Pros**:
  - No auth server to manage
  - Leverages existing identity infrastructure
- **Cons**:
  - External IDPs don't support MCP-specific claims
  - No control over token lifespans or audiences
  - Cannot issue tokens bound to specific MCP servers
- **Why not chosen**: Need custom token issuance with MCP-specific claims and audiences

---

## Compatibility

### Backward Compatibility

- **Fully backward compatible**: New auth type added to existing enum
- Existing `tokenExchange`, `headerInjection`, `bearerToken`, `unauthenticated` types unchanged
- No migration required for existing MCPExternalAuthConfig resources

### Forward Compatibility

- **Upstream providers list**: Designed as list for future multi-IDP support (currently validated to require exactly one)
- **Signing keys list**: Supports rotation without schema changes
- **HMAC secrets list**: Supports rotation without schema changes

---

## Implementation Plan

### Phase 1: CRD Changes

- Add `EmbeddedAuthServerConfig` types to `mcpexternalauthconfig_types.go`
- Run `task operator-generate && task operator-manifests && task crdref-gen`
- Add webhook validation (ensure only one upstream provider)

### Phase 2: Controller Changes

- Create `pkg/controllerutil/authserver.go` with volume generation helpers
- Integrate volume mounting into `deploymentForMCPServer` and `deploymentForMCPRemoteProxy`
- Update RunConfig generation to include auth server config

### Phase 3: RunConfig Extension

- Add `EmbeddedAuthServerConfig` to `pkg/runner/config.go`
- Add `WithEmbeddedAuthServer()` builder option

### Phase 4: Proxy Runner Integration

- Create `pkg/runner/authserver.go` with `EmbeddedAuthServer` wrapper
- Modify `Runner.Run()` to start auth server and mount routes
- Add cleanup logic

### Phase 5: Transport Integration

- Add `AuthServerHandler` to transport config types
- Mount auth server routes in HTTP transport

### Dependencies

- Existing `pkg/authserver/` package (no changes needed)
- Existing `pkg/authserver/upstream/` for IDP integration

---

## Testing Strategy

- **Unit tests**:
  - CRD type validation
  - Volume generation for signing keys and HMAC secrets
  - RunConfig serialization/deserialization
  - Embedded auth server initialization

- **Integration tests**:
  - Controller creates correct volumes and mounts
  - RunConfig populated correctly from MCPExternalAuthConfig

- **E2E tests**:
  - Full OAuth flow: authorize → callback → token → MCP request
  - Key rotation (add fallback key, promote, remove old)
  - JWKS endpoint returns correct public keys
  - Token validation with issued tokens
  - Upstream IDP integration (mock IDP)

---

## Documentation

- User documentation: How to configure embedded auth server for MCP servers
- API documentation: MCPExternalAuthConfig CRD reference with `embeddedAuthServer` type
- Architecture documentation: Update `docs/arch/` with auth server integration
- Operational guides: Key rotation procedures, troubleshooting auth flows

---

## Open Questions

None - all design questions resolved.

---

## References

- [OAuth 2.0 RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749)
- [PKCE RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636)
- [Resource Indicators RFC 8707](https://datatracker.ietf.org/doc/html/rfc8707)
- [OAuth 2.0 Token Exchange RFC 8693](https://datatracker.ietf.org/doc/html/rfc8693)
- [MCP Specification - Authentication](https://modelcontextprotocol.io/specification)
- [pkg/authserver/](pkg/authserver/) - ToolHive auth server implementation
- [Fosite OAuth2 Library](https://github.com/ory/fosite)

---

## RFC Lifecycle

### Review History

| Date | Reviewer | Decision | Notes |
|------|----------|----------|-------|
| 2026-01-27 | TBD | Draft | Initial submission |

### Implementation Tracking

| Repository | PR | Status |
|------------|-----|--------|
| toolhive | TBD | Pending |
