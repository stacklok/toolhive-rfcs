# RFC-0043: Registry Authentication

- **Status**: Draft
- **Author(s)**: Chris Burns (@ChrisJBurns)
- **Created**: 2026-02-20
- **Last Updated**: 2026-02-20
- **Target Repository**: toolhive
- **Related Issues**: [toolhive#2962](https://github.com/stacklok/toolhive/issues/2962)

## Summary

Add OAuth/OIDC authentication support to the ToolHive CLI for accessing
remote MCP server registries that require authentication. Phase 1 implements
browser-based OAuth with PKCE — the CLI opens a browser for user login,
receives tokens via a local callback, and injects them into all subsequent
registry API requests. Phase 2 adds static bearer token support for CI/CD
environments.

## Problem Statement

Organizations hosting private MCP server registries need to restrict access
to authorized users. The ToolHive CLI currently has no mechanism to attach
credentials when communicating with remote registries. This means:

- **Private registries behind authentication cannot be used.** Organizations
  deploying `toolhive-registry-server` with identity provider protection
  (Okta, Azure AD, etc.) have no way to authenticate `thv` CLI requests.
- **Organizations cannot control who discovers their MCP servers.** Without
  registry-level auth, any user who knows the registry URL can enumerate
  available servers.

ToolHive already has mature OAuth/OIDC infrastructure for remote MCP *server*
authentication (`pkg/auth/oauth/`, `pkg/auth/remote/`) and a secrets manager
for credential persistence (`pkg/secrets/`). The registry auth feature reuses
this infrastructure rather than building a parallel implementation.

## Goals

- Allow users to authenticate to remote registries via browser-based
  OAuth/OIDC with PKCE
- Cache and refresh tokens transparently across CLI invocations
- Store credentials securely using the existing secrets infrastructure
  (keyring / 1Password)
- Maintain backward compatibility — unauthenticated registries continue to
  work with no configuration changes
- Provide a clean CLI experience for configuring and managing registry auth

## Non-Goals

- **Per-server authentication.** Auth is per-registry, not per-MCP-server
  within a registry.
- **Client secret management in Phase 1.** Phase 1 uses public clients with
  PKCE only. Confidential client support (client secrets) may be added later.
- **Dynamic Client Registration (RFC 7591).** Auto-registering the CLI as an
  OAuth client would eliminate manual `client-id` configuration but is not
  universally supported by identity providers. Deferred.
- **Kubernetes registry auth.** This RFC targets the CLI (`thv`). Kubernetes
  operator registry access uses service accounts and is out of scope.

## Proposed Solution

### High-Level Design

```mermaid
flowchart TD
    User["User"]
    CLI["thv CLI"]
    Config["config.yaml<br/>(registry_auth section)"]
    Secrets["Secrets Manager<br/>(keyring / 1Password)"]
    IDP["Identity Provider<br/>(Okta, Azure AD, etc.)"]
    Registry["Registry Server<br/>(toolhive-registry-server)"]

    User -->|"thv config set-registry-auth"| CLI
    CLI -->|"OIDC discovery + save"| Config
    User -->|"thv registry list"| CLI
    CLI -->|"resolve token"| TokenSource
    TokenSource -->|"restore refresh token"| Secrets
    TokenSource -->|"browser OAuth flow"| IDP
    IDP -->|"access + refresh tokens"| TokenSource
    TokenSource -->|"persist refresh token"| Secrets
    CLI -->|"Authorization: Bearer"| Registry

    subgraph TokenSource["Token Resolution"]
        Cache["In-memory cache"]
        Restore["Restore from secrets"]
        Browser["Browser OAuth flow"]
        Cache --> Restore --> Browser
    end
```

The design introduces a `TokenSource` abstraction that plugs into the
existing registry HTTP client via a custom `http.RoundTripper`. When
registry auth is configured, every HTTP request to the registry
transparently acquires and attaches a bearer token. The token lifecycle
(cache → restore → browser flow → refresh) is handled internally.

### Detailed Design

#### Registry Provider Chain

The existing registry provider factory (`pkg/registry/factory.go`) gains
a `resolveTokenSource()` step that reads auth configuration and creates the
appropriate `TokenSource`. This is injected through the provider chain:

```
thv config set-registry <url>
thv config set-registry-auth --issuer <url> --client-id <id>
        │
        ▼
┌─────────────────────┐
│  pkg/registry/      │
│  factory.go         │──▶ Priority: API > Remote > Local > Embedded
│                     │
│  NewRegistryProvider│──▶ resolveTokenSource() from config
└────────┬────────────┘
         │
         ├──▶ CachedAPIRegistryProvider ──▶ api/client.go ──▶ HTTP + auth.Transport
         ├──▶ RemoteRegistryProvider    ──▶ HttpClientBuilder ──▶ HTTP GET (no auth)
         └──▶ LocalRegistryProvider     ──▶ (no network)
```

#### Component Changes

| Component | Location | Role |
|-----------|----------|------|
| OAuth flow | `pkg/auth/oauth/flow.go` | Existing browser-based PKCE flow with local callback server (reused) |
| OIDC discovery | `pkg/auth/oauth/oidc.go` | Existing auto-discovery of auth/token endpoints from issuer (reused) |
| Token source | `pkg/registry/auth/oauth_token_source.go` | **New.** Token lifecycle: cache → restore → browser flow |
| Auth transport | `pkg/registry/auth/transport.go` | **New.** Injects `Authorization: Bearer` on HTTP requests |
| Token source interface | `pkg/registry/auth/auth.go` | **New.** `TokenSource` interface and factory |
| Auth configurator | `pkg/registry/auth_configurator.go` | **New.** Business logic for set/unset/get auth config |
| Token persistence | `pkg/auth/remote/persisting_token_source.go` | Existing token persistence wrapper (reused) |
| Secrets manager | `pkg/secrets/` | Existing encrypted storage for refresh tokens (reused) |
| Config storage | `pkg/config/config.go` | Extended with `RegistryAuth` section |

The `pkg/registry/auth/` package is deliberately separate from `pkg/auth/`
because registry auth has different concerns than MCP server auth — different
config model, different token persistence keys, and different lifecycle. The
package reuses the underlying OAuth primitives without coupling the two
domains.

#### API Changes

**New interface** (`pkg/registry/auth/auth.go`):

```go
// TokenSource provides authentication tokens for registry HTTP requests.
type TokenSource interface {
    // Token returns a valid access token string, or empty string if no auth.
    // Implementations handle token refresh transparently.
    Token(ctx context.Context) (string, error)
}
```

**Modified signatures:**

```go
// pkg/registry/api/client.go — added tokenSource parameter
func NewClient(baseURL string, allowPrivateIp bool, tokenSource auth.TokenSource) (Client, error)

// pkg/registry/provider_api.go — added tokenSource parameter
func NewAPIRegistryProvider(apiURL string, allowPrivateIp bool, tokenSource auth.TokenSource) (*APIRegistryProvider, error)

// pkg/registry/provider_cached.go — added tokenSource parameter
func NewCachedAPIRegistryProvider(apiURL string, allowPrivateIp bool, usePersistent bool, tokenSource auth.TokenSource) (*CachedAPIRegistryProvider, error)
```

#### Configuration Changes

The `Config` struct in `pkg/config/config.go` gains a `RegistryAuth` field:

```yaml
# ~/.config/toolhive/config.yaml
registry_api_url: "https://registry.company.com"
registry_auth:
  type: "oauth"
  oauth:
    issuer: "https://auth.company.com"
    client_id: "toolhive-cli"
    scopes: ["openid"]
    audience: "api://my-registry"
    use_pkce: true
    callback_port: 8666
    # Populated automatically after first login:
    cached_refresh_token_ref: "REGISTRY_OAUTH_REFRESH_TOKEN"
    cached_token_expiry: "2026-02-20T12:00:00Z"
```

**Config structs:**

```go
type RegistryAuth struct {
    Type  string              `yaml:"type,omitempty"`   // "oauth" or "" (none)
    OAuth *RegistryOAuthConfig `yaml:"oauth,omitempty"`
}

type RegistryOAuthConfig struct {
    Issuer       string   `yaml:"issuer"`
    ClientID     string   `yaml:"client_id"`
    ClientSecret string   `yaml:"client_secret,omitempty"` // For future confidential clients
    Scopes       []string `yaml:"scopes,omitempty"`
    Audience     string   `yaml:"audience,omitempty"`       // RFC 8707 resource indicator
    UsePKCE      bool     `yaml:"use_pkce"`
    CallbackPort int      `yaml:"callback_port,omitempty"`

    CachedRefreshTokenRef string    `yaml:"cached_refresh_token_ref,omitempty"`
    CachedTokenExpiry     time.Time `yaml:"cached_token_expiry,omitempty"`
}
```

#### CLI Commands

**Setup:**

```bash
# Configure the registry URL
thv config set-registry https://registry.company.com/api

# Configure OAuth authentication
thv config set-registry-auth \
    --issuer https://auth.company.com \
    --client-id toolhive-cli \
    --audience api://my-registry

# View current configuration
thv config get-registry
# → Current registry: https://registry.company.com/api (API endpoint, OAuth configured)

# Remove auth (keeps registry URL)
thv config unset-registry-auth
```

**Flags for `set-registry-auth`:**

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--issuer` | Yes | — | OIDC issuer URL |
| `--client-id` | Yes | — | OAuth client ID |
| `--scopes` | No | `openid` | OAuth scopes (comma-separated) |
| `--audience` | No | — | OAuth audience / resource indicator (RFC 8707) |
| `--use-pkce` | No | `true` | Enable PKCE (recommended for public clients) |

The `set-registry-auth` command validates the issuer by performing OIDC
discovery before saving the configuration. This catches typos and
unreachable issuers early.

#### Token Lifecycle

```mermaid
flowchart TD
    A["Token() called"] --> B{"In-memory<br/>token valid?"}
    B -->|Yes| C["Return access token"]
    B -->|No| D{"Refresh token<br/>in secrets manager?"}
    D -->|Yes| E["Refresh token exchange"]
    E -->|Success| C
    E -->|Failure| F["Browser OAuth flow"]
    D -->|No| F
    F --> G["Open browser → issuer authorize endpoint<br/>(with PKCE challenge)"]
    G --> H["User authenticates in browser"]
    H --> I["Callback on localhost:8666<br/>with auth code"]
    I --> J["Exchange code + PKCE verifier<br/>for access + refresh tokens"]
    J --> K["Persist refresh token<br/>in secrets manager"]
    K --> L["Update config with<br/>token reference + expiry"]
    L --> C
```

**Cross-invocation behavior:**

| Scenario | What happens |
|----------|-------------|
| First request (no cached tokens) | Browser OAuth flow → user authenticates → tokens persisted |
| Subsequent CLI invocations | Restore refresh token from secrets → silent token refresh (no browser) |
| Within same process | In-memory token source → auto-refresh via `oauth2` library |
| Refresh token expired/revoked | Falls back to browser OAuth flow (same as first request) |

#### Data Flow

```mermaid
sequenceDiagram
    participant User
    participant CLI as thv CLI
    participant Factory as registry/factory.go
    participant TokenSrc as oauthTokenSource
    participant Secrets as Secrets Manager
    participant IDP as Identity Provider
    participant Registry as Registry Server

    User->>CLI: thv registry list
    CLI->>Factory: NewRegistryProvider(cfg)
    Factory->>Factory: resolveTokenSource(cfg)
    Factory->>Factory: NewCachedAPIRegistryProvider(url, tokenSource)
    Note over Factory: Skip validation probe<br/>(auth requires user interaction)

    CLI->>Registry: GET /servers (via auth.Transport)
    Registry-->>TokenSrc: Transport calls Token()
    TokenSrc->>TokenSrc: Check in-memory cache (miss)
    TokenSrc->>Secrets: Get refresh token
    Secrets-->>TokenSrc: No cached token

    TokenSrc->>IDP: OIDC Discovery
    IDP-->>TokenSrc: Auth + token endpoints
    TokenSrc->>TokenSrc: Generate PKCE verifier/challenge
    TokenSrc->>TokenSrc: Start local server on :8666
    TokenSrc->>User: Open browser → authorize endpoint
    User->>IDP: Authenticate
    IDP->>TokenSrc: Redirect to localhost:8666/callback
    TokenSrc->>IDP: Exchange code + PKCE verifier
    IDP-->>TokenSrc: Access + refresh tokens
    TokenSrc->>Secrets: Persist refresh token
    TokenSrc-->>Registry: Request with Authorization: Bearer <token>
    Registry-->>CLI: Server list response
```

#### Design Decisions

**Why skip API validation when auth is configured:**
The API provider normally validates the endpoint on creation by making a test
request. When OAuth is configured, this test request would trigger the
browser flow within a 10-second timeout, which cannot complete. Instead,
validation is deferred to the first real API call.

**Why PKCE without client secret:**
CLI applications are public clients — they cannot securely store a client
secret. PKCE (RFC 7636) provides equivalent security without requiring a
secret. The identity provider must be configured as a "Native App" (public
client) to accept requests without client authentication.

**Why reuse `pkg/auth/oauth/` instead of a new implementation:**
ToolHive already has a battle-tested OAuth flow for remote MCP server
authentication. The registry auth reuses the same `oauth.NewFlow`,
`oauth.CreateOAuthConfigFromOIDC`, and `remote.NewPersistingTokenSource`
functions, ensuring consistency and avoiding duplication.

**Why a separate `pkg/registry/auth/` package:**
Registry auth has different concerns than MCP server auth (different config
model, different token persistence keys, different lifecycle). A separate
package keeps the boundaries clean while reusing the underlying OAuth
primitives.

## Security Considerations

### Threat Model

| Threat | Description | Likelihood | Impact |
|--------|-------------|------------|--------|
| Token interception | Access token intercepted in transit | Low (HTTPS enforced) | High |
| Auth code interception | Authorization code stolen during callback | Low (PKCE mitigates) | High |
| Refresh token theft | Refresh token extracted from storage | Low (encrypted at rest) | High |
| Credential leakage | Tokens appear in logs or shell history | Medium | High |
| Callback port hijack | Malicious process listens on :8666 | Low | Medium |

### Authentication and Authorization

- Phase 1 adds OAuth/OIDC authentication for registry API requests.
- No changes to MCP server authentication or the embedded auth server.
- The identity provider controls authorization (scopes, audience). The CLI
  is a relying party only.
- The `audience` flag (RFC 8707 resource indicator) allows the identity
  provider to issue registry-scoped tokens, preventing token misuse across
  services.

### Data Security

- **Access tokens** are held in memory only and never persisted to disk.
- **Refresh tokens** are stored in the secrets manager (encrypted at rest
  via keyring or 1Password), not in plaintext config. The config only stores
  a *reference key* (`cached_refresh_token_ref`), not the token value.
- **No tokens in logs.** Token values never appear in debug logs. Only token
  *presence* is logged (e.g., "using cached refresh token").

### Input Validation

- The `--issuer` URL is validated via OIDC discovery before saving to
  config. This confirms the URL is reachable and serves a valid
  `.well-known/openid-configuration` document.
- OAuth callback parameters (authorization code, state) are validated by
  the existing `pkg/auth/oauth/` flow implementation.
- The `--scopes`, `--audience`, and `--client-id` values are passed through
  to the identity provider, which performs its own validation.

### Secrets Management

- Refresh tokens are stored using ToolHive's existing `pkg/secrets/`
  infrastructure, which supports keyring (macOS Keychain, Linux
  secret-service) and 1Password backends.
- The secret key (`REGISTRY_OAUTH_REFRESH_TOKEN`) is a fixed identifier.
  Changing identity providers requires `thv config unset-registry-auth`
  which clears the cached token reference.
- No client secrets are stored in Phase 1 (public client with PKCE).

### Audit and Logging

- OAuth flow initiation and completion logged at INFO level.
- Token refresh (silent) logged at DEBUG level.
- Authentication failures (401/403 from registry) produce actionable error
  messages with remediation instructions.
- No token values are ever logged.

### Mitigations

| Threat | Mitigation |
|--------|------------|
| Token interception | HTTPS enforced by `ValidatingTransport`; HTTP only allowed with `--allow-private-ip` for localhost testing |
| Auth code interception | S256 PKCE challenges on all authorization code flows |
| Refresh token theft | Encrypted at rest via secrets manager (keyring/1Password); not stored in plaintext config |
| Credential leakage | Token values excluded from all log levels; no tokens in URLs or shell history |
| Callback port hijack | Local callback server accepts a single callback then shuts down; state parameter prevents CSRF |

## Alternatives Considered

### Alternative 1: Bearer Tokens First, OAuth Second

- **Description**: Implement static bearer token auth first, add OAuth later.
- **Pros**: Simpler initial implementation. Covers CI/CD use case first.
- **Cons**: The primary use case is corporate registries behind identity
  providers (Okta, Azure AD). Browser-based OAuth provides a better UX for
  interactive users ("just log in") compared to manually obtaining and
  rotating bearer tokens.
- **Why not chosen**: OAuth is the more impactful feature. Bearer tokens
  are deferred to Phase 2 for CI/CD.

### Alternative 2: Embed Auth in Registry URL

- **Description**: Support `https://user:token@registry.com` URL format.
- **Pros**: No config changes needed.
- **Cons**: Credentials appear in logs, shell history, and process listings.
  No token refresh capability.
- **Why not chosen**: Security risk and poor UX.

### Alternative 3: Add Auth to HttpClientBuilder Directly

- **Description**: Extend the existing `HttpClientBuilder` with auth support.
- **Pros**: Fewer new types.
- **Cons**: The builder is stateless and builds a client once. Registry auth
  needs dynamic token resolution (OAuth refresh, secrets manager lookups).
- **Why not chosen**: A `TokenSource`-based `RoundTripper` is more flexible
  and follows the `oauth2.TokenSource` pattern from the Go ecosystem.

### Alternative 4: Separate Auth Config File

- **Description**: Store registry auth in a dedicated file.
- **Pros**: Separation of concerns.
- **Cons**: Adds complexity. The existing `config.yaml` already stores
  registry configuration and is the natural place for registry auth.
- **Why not chosen**: Unnecessary indirection.

### Alternative 5: Dynamic Client Registration (RFC 7591)

- **Description**: Auto-register the CLI as an OAuth client with the IDP.
- **Pros**: Eliminates manual `--client-id` configuration.
- **Cons**: Not all identity providers support DCR. Adds complexity.
- **Why not chosen**: Deferred. May revisit if multiple registries with
  different IdPs become a common use case.

## Compatibility

### Backward Compatibility

- **Fully backward compatible.** Unauthenticated registries continue to
  work with no configuration changes.
- When `registry_auth` is absent or empty in config, the `resolveTokenSource`
  function returns `nil`, and the provider chain behaves identically to today.
- The `tokenSource` parameter added to `NewClient`, `NewAPIRegistryProvider`,
  and `NewCachedAPIRegistryProvider` is `nil`-safe — passing `nil` results
  in no auth headers.

### Forward Compatibility

- The `RegistryAuth.Type` field enables future auth types (e.g., `"bearer"`)
  without changing the config structure.
- The `TokenSource` interface is auth-mechanism agnostic — Phase 2 bearer
  token support implements the same interface.
- The `ClientSecret` field in `RegistryOAuthConfig` is present but unused
  in Phase 1, ready for confidential client support.

## Implementation Plan

### Phase 1: OAuth/OIDC with PKCE (Current)

- Add `RegistryAuth` and `RegistryOAuthConfig` structs to `pkg/config/config.go`
- Create `pkg/registry/auth/` package:
  - `auth.go` — `TokenSource` interface and `NewTokenSource` factory
  - `oauth_token_source.go` — OAuth token lifecycle (cache → restore → browser flow)
  - `transport.go` — `http.RoundTripper` wrapper that injects `Authorization: Bearer`
- Create `pkg/registry/auth_configurator.go` — business logic for CLI commands
- Add `set-registry-auth` and `unset-registry-auth` commands to `cmd/thv/app/config.go`
- Update `pkg/registry/factory.go` with `resolveTokenSource()` integration
- Thread `tokenSource` through `NewCachedAPIRegistryProvider` → `NewAPIRegistryProvider` → `NewClient`
- Skip API validation probe when `tokenSource` is non-nil
- Update `get-registry` output to show auth status

### Phase 2: Bearer Token Authentication (Future)

- Add `bearerTokenSource` implementing `TokenSource`
- Extend config with `bearer` type:
  ```yaml
  registry_auth:
    type: "bearer"
    bearer_token_ref: "REGISTRY_TOKEN"     # Secret manager reference
    bearer_token_file: "/path/to/token"    # Alternative: path to token file
  ```
- Token resolution order:
  1. Environment variable `TOOLHIVE_REGISTRY_AUTH_TOKEN` (highest priority)
  2. Token file from `bearer_token_file` config field
  3. Secrets manager via `bearer_token_ref` config field
- Add `--token` and `--token-file` flags to `set-registry-auth`
- Add actionable 401/403 error messages with remediation instructions

### Dependencies

- No new external dependencies. Reuses existing `golang.org/x/oauth2`,
  `pkg/auth/oauth/`, `pkg/auth/remote/`, and `pkg/secrets/` packages.

## Testing Strategy

- **Unit tests**: `oauthTokenSource` with mocked secrets provider and OIDC
  endpoints. `Transport` round-tripper with mock token sources.
- **Auth configurator tests**: Verify OIDC discovery validation, config
  read/write operations.
- **Integration tests**: End-to-end OAuth flow with a test OIDC server
  (e.g., `ory/fosite` or mock HTTP server returning OIDC discovery documents).
- **Backward compatibility tests**: Verify unauthenticated registries
  continue to work when `registry_auth` is absent.
- **Token lifecycle tests**: Verify cache → restore → refresh → browser
  flow fallback chain.

## Documentation

- Update CLI help text for `thv config` subcommands
- Add registry authentication guide to user documentation
- Document identity provider setup requirements (public client, PKCE,
  callback URL `http://localhost:8666/callback`)

## Open Questions

1. **Multiple registry auth configurations.** If a user switches between
   registries with different auth requirements, should we support per-registry
   auth profiles? Currently auth is global (tied to whatever registry URL is
   configured).

2. **Callback port conflicts.** The callback server uses port 8666, which
   is the same as the remote MCP auth callback. Should registry auth use a
   different port to avoid conflicts if both flows run simultaneously?

## References

- [MCP Registry API Specification](https://github.com/modelcontextprotocol/registry)
- [ToolHive Registry Server](https://github.com/stacklok/toolhive-registry-server)
- [RFC 7636: PKCE](https://tools.ietf.org/html/rfc7636)
- [RFC 8707: Resource Indicators for OAuth 2.0](https://tools.ietf.org/html/rfc8707)
- [RFC 6750: Bearer Token Usage](https://tools.ietf.org/html/rfc6750)
- [toolhive#2962: Registry authentication support](https://github.com/stacklok/toolhive/issues/2962)
- [toolhive#3908: CLI auth reg implementation](https://github.com/stacklok/toolhive/pull/3908)
- Existing auth infrastructure: `pkg/auth/oauth/`, `pkg/auth/remote/`, `pkg/secrets/`

---

## RFC Lifecycle

### Review History

| Date | Reviewer | Decision | Notes |
|------|----------|----------|-------|
| 2026-02-20 | @ChrisJBurns | Draft | Initial submission based on design doc |

### Implementation Tracking

| Repository | PR | Status |
|------------|-----|--------|
| toolhive | [#3908](https://github.com/stacklok/toolhive/pull/3908) | In Progress |
