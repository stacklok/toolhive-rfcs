# RFC-0043: Registry Authentication

- **Status**: Draft
- **Author(s)**: Chris Burns (@ChrisJBurns)
- **Created**: 2026-02-20
- **Last Updated**: 2026-03-02
- **Target Repository**: toolhive
- **Related Issues**: [toolhive#2962](https://github.com/stacklok/toolhive/issues/2962)

## Summary

Add OAuth/OIDC authentication support to the ToolHive CLI for accessing
remote MCP server registries that require authentication. Phase 1 implements
browser-based OAuth with PKCE — the CLI opens a browser for user login,
receives tokens via a local callback, and injects them into all subsequent
registry API requests. Phase 2 adds static bearer token support for CI/CD
and headless environments (including `thv serve`).

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

**Note on MCP spec alignment:** The official MCP Registry at
`registry.modelcontextprotocol.io` is unauthenticated for reads. This RFC
targets *private* `toolhive-registry-server` instances behind corporate
identity providers. There is currently no MCP standard for authenticating
to a private registry's read API — this design is a general-purpose HTTP
API auth layer for the registry, not an implementation of the MCP
authorization spec (which governs MCP server-level auth, not registry-level
auth).

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
- **Confidential client support.** Not in scope for Phase 1. Phase 1 uses
  public clients with PKCE only.
- **Dynamic Client Registration (RFC 7591).** Auto-registering the CLI as an
  OAuth client would eliminate manual `client-id` configuration but is not
  universally supported by identity providers. Deferred.
- **Kubernetes registry auth.** This RFC targets the CLI (`thv`). Kubernetes
  operator registry access uses service accounts and is out of scope.
- **`RemoteRegistryProvider` authentication.** Auth applies to the API
  registry provider only. Organizations hosting a static JSON registry file
  behind authentication should migrate to the API provider
  (`toolhive-registry-server`). See [Limitations](#limitations).
- **MCP Protected Resource Metadata (RFC 9728).** Automatic auth discovery
  from `401 WWW-Authenticate` headers is deferred. Phase 1 uses manual
  issuer configuration. See [Future Considerations](#future-considerations).

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

**Note:** Auth is only supported for the API registry provider. The
`RemoteRegistryProvider` (static JSON file over HTTP) does not support
authentication. See [Limitations](#limitations).

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
    callback_port: 8666
    # Populated automatically after first login:
    cached_refresh_token_ref: "REGISTRY_OAUTH_<hash>"
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
    Scopes       []string `yaml:"scopes,omitempty"`
    Audience     string   `yaml:"audience,omitempty"`       // IdP audience parameter
    CallbackPort int      `yaml:"callback_port,omitempty"`

    CachedRefreshTokenRef string `yaml:"cached_refresh_token_ref,omitempty"`
}
```

**Changes from original design:**
- Removed `UsePKCE` field — PKCE with S256 is always enabled (mandatory per
  OAuth 2.1 for public clients). Not user-configurable.
- Removed `CachedTokenExpiry` — access tokens are memory-only and their
  expiry should not be persisted. The `oauth2` library handles expiry
  internally via the token source.
- The `Audience` field is an IdP-specific parameter (e.g., Auth0's
  `audience`, Azure AD's `resource`). It is **not** the RFC 8707 `resource`
  parameter, which requires an absolute `https://` URI. Most IdPs use a
  proprietary audience parameter, so this field accommodates that.
- The secret key for `CachedRefreshTokenRef` is derived from the registry
  URL and issuer (e.g., `REGISTRY_OAUTH_<sha256(registry_url + "\0" + issuer)[:12]>`)
  to prevent token clobbering when switching between registries. A null byte
  delimiter separates the URL and issuer in the hash input to prevent
  ambiguous concatenation.

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
| `--issuer` | Yes | — | OIDC issuer URL (must be `https://`, except `localhost`) |
| `--client-id` | Yes | — | OAuth client ID |
| `--scopes` | No | `openid` | OAuth scopes (comma-separated) |
| `--audience` | No | — | OAuth audience parameter (IdP-specific) |

The `set-registry-auth` command validates the issuer by performing OIDC
discovery before saving the configuration. This catches typos and
unreachable issuers early. The issuer URL **must** use the `https://` scheme
(with an exception for `localhost`/`127.0.0.1` for development), per the
OIDC Discovery specification Section 4.1.

**`get-registry` output changes:**

The existing `thv config get-registry` command is updated to display auth
status alongside the registry URL:

```
# OAuth configured but no cached tokens yet
Current registry: https://registry.company.com/api (API endpoint, OAuth configured)

# OAuth configured with cached tokens (authenticated)
Current registry: https://registry.company.com/api (API endpoint, OAuth authenticated)

# No auth configured (unchanged from current behavior)
Current registry: https://registry.company.com/api (API endpoint)
```

The suffix is determined by checking `RegistryAuth.Type` and whether
`CachedRefreshTokenRef` is populated (indicating a prior successful login).

#### Token Lifecycle

```mermaid
flowchart TD
    A["Token() called"] --> B{"In-memory<br/>token valid?"}
    B -->|Yes| C["Return access token"]
    B -->|No| D{"Refresh token<br/>in secrets manager?"}
    D -->|Yes| E["Refresh token exchange"]
    E -->|Success| C
    E -->|Failure| F{"Interactive<br/>context?"}
    D -->|No| F
    F -->|Yes| G["Browser OAuth flow"]
    F -->|No| H["Return error:<br/>run thv config set-registry-auth"]
    G --> I["Open browser → issuer authorize endpoint<br/>(with PKCE S256 challenge)"]
    I --> J["User authenticates in browser"]
    J --> K["Callback on localhost:8666<br/>with auth code"]
    K --> L["Exchange code + PKCE verifier<br/>for access + refresh tokens"]
    L --> M["Persist refresh token<br/>in secrets manager"]
    M --> N["Update config with<br/>token reference"]
    N --> C
```

**Cross-invocation behavior:**

| Scenario | What happens |
|----------|-------------|
| First request (no cached tokens) | Browser OAuth flow → user authenticates → tokens persisted |
| Subsequent CLI invocations | Restore refresh token from secrets → silent token refresh (no browser) |
| Within same process | In-memory token source → auto-refresh via `oauth2` library |
| Refresh token expired/revoked | Falls back to browser OAuth flow (same as first request) |
| Secrets manager not configured | `resolveTokenSource` logs a debug message and continues with a `nil` secrets provider. Tokens work for the current session (in-memory) but are not persisted across invocations — each CLI run triggers the browser flow. |
| `thv serve` (headless context) | Uses cached tokens from secrets manager. If no cached tokens exist, returns an error directing the user to authenticate via `thv registry list` first. Browser flow is never triggered from API server context. |
| Refresh token rotation | When the IdP returns a new refresh token during token refresh, the `PersistingTokenSource` automatically stores the new token, replacing the previous one. |

**PKCE enforcement:** PKCE with `S256` code challenge method is always
used. The `code_challenge_method` is hardcoded to `S256` — there is no
fallback to `plain`. This is mandatory per OAuth 2.1 for public clients.

**State parameter:** The `state` parameter is generated as a
cryptographically random value (32 bytes from `crypto/rand`) and validated
on callback to prevent CSRF attacks.

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

    Note over CLI,Registry: auth.Transport intercepts outgoing request
    CLI->>TokenSrc: auth.Transport calls Token()
    TokenSrc->>TokenSrc: Check in-memory cache (miss)
    TokenSrc->>Secrets: Get refresh token
    Secrets-->>TokenSrc: No cached token

    TokenSrc->>IDP: OIDC Discovery
    IDP-->>TokenSrc: Auth + token endpoints
    TokenSrc->>TokenSrc: Generate PKCE verifier/challenge (S256)
    TokenSrc->>TokenSrc: Start local server on 127.0.0.1:8666
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

**Why PKCE is mandatory (not configurable):**
CLI applications are public clients — they cannot securely store a client
secret. PKCE (RFC 7636) with S256 is the only mechanism preventing
authorization code interception attacks for public clients. OAuth 2.1
mandates PKCE for all authorization code grants. Allowing it to be
disabled would create a critical security vulnerability.

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

**Why share callback port 8666 with remote MCP auth:**
Registry auth and remote MCP server auth do not run simultaneously. Registry
auth happens during `thv registry list` / `thv run` (before any MCP server
is running), while remote MCP auth happens when the MCP server starts. The
port is reused intentionally via `remote.DefaultCallbackPort` to keep the
configuration simple. The `callback_port` config field allows overriding if
needed. The callback server binds to `127.0.0.1` (not `0.0.0.0`) to prevent
network-based interception.

**Why `audience` instead of RFC 8707 `resource`:**
RFC 8707 requires the `resource` parameter to be an absolute `https://` URI.
In practice, most identity providers (Okta, Auth0, Azure AD) use a
proprietary `audience` parameter that accepts arbitrary strings like
`api://my-registry`. The `audience` field accommodates real-world IdP
behavior. If an IdP requires RFC 8707 `resource` semantics, the `audience`
field can be set to the full `https://` URI of the registry.

**Why derive secret keys from registry URL:**
Using a fixed secret key (`REGISTRY_OAUTH_REFRESH_TOKEN`) would cause
token clobbering when switching between registries — the second registry's
refresh token silently overwrites the first. Deriving the key from the
registry URL and issuer (e.g., `REGISTRY_OAUTH_<hash>`) allows multiple
sets of credentials to coexist in the secrets manager, even though only one
registry can be active at a time.

### Limitations

**`thv serve` (API server context):**
The browser-based OAuth flow requires user interaction and cannot be
triggered from within an API server handling HTTP requests. When `thv serve`
needs to access an authenticated registry (e.g., ToolHive Studio requests
server list), it relies on cached tokens obtained from a prior CLI session.
If no cached tokens exist or the refresh token has expired, the API server
returns an error indicating that authentication is required. The user must
run `thv registry list` (or any registry command) in a terminal first to
complete the browser flow and cache tokens. Phase 2 bearer token support
will provide a non-interactive alternative for `thv serve` deployments.

**`RemoteRegistryProvider` (static JSON file):**
Auth applies only to the `CachedAPIRegistryProvider` (the API registry
provider). The `RemoteRegistryProvider`, which fetches a static JSON
registry file over HTTP, does not support authentication. Organizations
that host registry files behind authentication should migrate to the API
registry provider (`toolhive-registry-server`).

**Single active registry:**
Only one registry can be authenticated at a time (the one configured via
`registry_api_url`). However, credentials for previously authenticated
registries are preserved in the secrets manager (keyed by registry URL hash)
and restored automatically when switching back.

**401/403 error handling:**
When a registry returns `401 Unauthorized` during the API validation probe,
the error message should be actionable:

```
Error: registry at https://registry.company.com returned 401 Unauthorized.

If this registry requires authentication, configure it with:
  thv config set-registry-auth --issuer <issuer-url> --client-id <client-id>
```

This requires detecting HTTP 401/403 status codes in the API provider and
enhancing the error message, rather than surfacing the generic "API endpoint
not functional" error.

## Security Considerations

### Threat Model

| Threat | Description | Likelihood | Impact |
|--------|-------------|------------|--------|
| Token interception | Access token intercepted in transit | Low (HTTPS enforced) | High |
| Auth code interception | Authorization code stolen during callback | Low (PKCE + state mitigate) | High |
| Refresh token theft | Refresh token extracted from storage | Low (encrypted at rest) | High |
| Credential leakage | Tokens appear in logs or shell history | Medium | High |
| Callback port hijack | Malicious process listens on :8666 | Low | Medium |
| Config file exposure | Config readable by other users | Medium (depends on umask) | Medium |
| Phishing via issuer | Malicious issuer URL redirects to phishing page | Low | High |

### Authentication and Authorization

- Phase 1 adds OAuth/OIDC authentication for registry API requests.
- No changes to MCP server authentication or the embedded auth server.
- The identity provider controls authorization (scopes, audience). The CLI
  is a relying party only.
- The `audience` flag allows the identity provider to issue
  registry-scoped tokens, preventing token misuse across services.
  **Note:** If `--audience` is omitted, tokens may be valid for multiple
  services sharing the same IdP. Operators should configure audience
  restrictions at the IdP level.

### Data Security

- **Access tokens** are held in memory only and never persisted to disk.
- **Refresh tokens** are stored in the secrets manager (encrypted at rest
  via keyring or 1Password), not in plaintext config. The config only stores
  a *reference key* (`cached_refresh_token_ref`), not the token value.
- **No tokens in logs.** Token values never appear in debug logs. Only token
  *presence* is logged (e.g., "using cached refresh token").
- **Config file permissions.** The `config.yaml` file is programmatically
  created with mode `0600` (owner read/write only) via `os.OpenFile`. While
  the file does not contain tokens, it contains the issuer URL, client ID,
  and token reference key, which are organizational infrastructure details.

### Input Validation

- The `--issuer` URL is validated via OIDC discovery before saving to
  config. This confirms the URL is reachable and serves a valid
  `.well-known/openid-configuration` document. The issuer URL **must** use
  `https://` (with a `localhost` exception for development).
- OAuth callback parameters (authorization code, state) are validated by
  the existing `pkg/auth/oauth/` flow implementation. The `state` parameter
  is generated with `crypto/rand` (32 bytes) and compared on callback.
- The `--scopes`, `--audience`, and `--client-id` values are passed through
  to the identity provider, which performs its own validation.

### Secrets Management

- Refresh tokens are stored using ToolHive's existing `pkg/secrets/`
  infrastructure, which supports keyring (macOS Keychain, Linux
  secret-service) and 1Password backends.
- The secret key is derived from the registry URL and issuer
  (`REGISTRY_OAUTH_<hash>`) to prevent clobbering when switching registries.
- No client secrets are stored in Phase 1 (public client with PKCE).
- When the IdP rotates refresh tokens during token refresh, the
  `PersistingTokenSource` automatically stores the new token, replacing the
  previous one.

### Audit and Logging

- OAuth flow initiation and completion logged at INFO level.
- Token refresh (silent) logged at DEBUG level.
- Authentication failures (401/403 from registry) produce actionable error
  messages with remediation instructions.
- No token values are ever logged.

### Mitigations

| Threat | Mitigation |
|--------|------------|
| Token interception | HTTPS enforced by `ValidatingTransport`; `--allow-private-ip` relaxes both private IP blocking and HTTPS enforcement for registry connections (enabling `http://localhost` testing). This means bearer tokens are sent over plain HTTP when this flag is used — it should only be used for local development. This flag does not affect TLS requirements for IdP communication (token endpoint, authorization endpoint). |
| Auth code interception | PKCE with S256 (mandatory, not configurable) on all authorization code flows |
| Refresh token theft | Encrypted at rest via secrets manager (keyring/1Password); not stored in plaintext config |
| Credential leakage | Token values excluded from all log levels; no tokens in URLs or shell history |
| Callback port hijack | Local callback server binds to `127.0.0.1` (not `0.0.0.0`); accepts a single callback then shuts down; `state` parameter (crypto/rand, 32 bytes) prevents CSRF. The browser flow has a default timeout of 120 seconds — if the user does not authenticate within this window, the callback server shuts down and an error is returned. |
| Config file exposure | Config created with `0600` permissions; no token values in config |
| Phishing via issuer | Issuer validated via OIDC discovery at configuration time; HTTPS required |

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

### Alternative 6: Automatic Auth Discovery via 401 + RFC 9728

- **Description**: When the registry returns `401 Unauthorized`, parse the
  `WWW-Authenticate` header to discover the authorization server via
  Protected Resource Metadata (RFC 9728), as MCP 2025-11-25 mandates for
  MCP server auth.
- **Pros**: Zero-configuration experience — the CLI auto-discovers auth
  requirements. Aligns with MCP spec direction.
- **Cons**: Requires the registry server to implement RFC 9728 metadata
  endpoints. No current `toolhive-registry-server` support. Adds complexity
  to the initial implementation.
- **Why not chosen**: Deferred to a future phase. Manual `--issuer`
  configuration is sufficient for Phase 1 and avoids a dependency on
  registry server changes.

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
- Detect 401/403 during validation and provide actionable error messages
- Update `get-registry` output to show auth status
- Derive secret keys from registry URL hash to prevent token clobbering

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
- This phase unblocks `thv serve` with authenticated registries (no browser
  flow required)

### Future Considerations

- **Automatic auth discovery (RFC 9728):** Parse `WWW-Authenticate` headers
  from 401 responses to auto-discover the authorization server, eliminating
  manual `--issuer` configuration.
- **Token revocation on `unset-registry-auth`:** Attempt to call the IdP's
  revocation endpoint (RFC 7009) when clearing auth config. Currently,
  `unset-registry-auth` only clears local state — the refresh token remains
  valid at the IdP until it naturally expires.
- **Ephemeral callback ports:** Use OS-assigned ephemeral ports (`:0`)
  instead of the fixed port 8666, reducing collision risk and improving
  security. Requires IdPs that allow dynamic redirect URIs.
- **Client ID Metadata Documents:** MCP 2025-11-25 recommends this as
  the preferred client registration mechanism. Would eliminate the need
  for manual `--client-id` configuration.

### Dependencies

- No new external dependencies. Reuses existing `golang.org/x/oauth2`,
  `pkg/auth/oauth/`, `pkg/auth/remote/`, and `pkg/secrets/` packages.

## Documentation

- Update CLI help text for `thv config` subcommands
- Add registry authentication guide to user documentation
- Document identity provider setup requirements (public client, PKCE,
  callback URL `http://127.0.0.1:8666/callback`)

## Open Questions

1. **Multiple registry auth configurations.** If a user switches between
   registries with different auth requirements, should we support per-registry
   auth profiles? Currently auth is global (tied to whatever registry URL is
   configured), though credentials are preserved in the secrets manager
   keyed by registry URL hash.

## References

- [MCP Registry API Specification](https://github.com/modelcontextprotocol/registry)
- [ToolHive Registry Server](https://github.com/stacklok/toolhive-registry-server)
- [RFC 7636: PKCE](https://tools.ietf.org/html/rfc7636)
- [RFC 6750: Bearer Token Usage](https://tools.ietf.org/html/rfc6750)
- [RFC 9728: Protected Resource Metadata](https://datatracker.ietf.org/doc/html/rfc9728)
- [OAuth 2.0 Security BCP](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics)
- [toolhive#2962: Registry authentication support](https://github.com/stacklok/toolhive/issues/2962)
- [toolhive#3908: CLI auth reg implementation](https://github.com/stacklok/toolhive/pull/3908)
- Existing auth infrastructure: `pkg/auth/oauth/`, `pkg/auth/remote/`, `pkg/secrets/`

---

## RFC Lifecycle

### Review History

| Date | Reviewer | Decision | Notes |
|------|----------|----------|-------|
| 2026-02-20 | @ChrisJBurns | Draft | Initial submission based on design doc |
| 2026-03-02 | @ChrisJBurns | Revised | Address review feedback: fix sequence diagram, resolve callback port question, add secrets degradation behavior, document get-registry output, clarify callback timeout, remove ClientSecret field |
| 2026-03-02 | Expert review panel | Revised | Address expert panel findings: remove use_pkce toggle (PKCE mandatory), document thv serve limitation, fix audience/resource terminology, add 401 error handling, derive secret keys from registry URL, add config file permissions, document refresh token rotation, add HTTPS issuer enforcement, document RemoteRegistryProvider limitation, add MCP spec alignment note |

### Implementation Tracking

| Repository | PR | Status |
|------------|-----|--------|
| toolhive | [#3908](https://github.com/stacklok/toolhive/pull/3908) | In Progress |
