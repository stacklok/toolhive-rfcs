# RFC-0043: Registry Authentication

- **Status**: Draft
- **Author(s)**: Chris Burns (@ChrisJBurns)
- **Created**: 2026-02-20
- **Last Updated**: 2026-03-04
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

The design uses `oauth2.TokenSource` from `golang.org/x/oauth2` — the same
interface used throughout ToolHive's existing auth infrastructure — plugged
into the registry HTTP client via the standard `oauth2.Transport`
`http.RoundTripper`. When registry auth is configured, every HTTP request to
the registry transparently acquires and attaches a bearer token. The token
lifecycle (cache → restore → browser flow → refresh) is handled internally.

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
| Token source | `pkg/registry/auth/oauth_token_source.go` | **New.** Token lifecycle: cache → restore → browser flow. Implements `oauth2.TokenSource`. |
| Token source factory | `pkg/registry/auth/auth.go` | **New.** Factory for constructing the appropriate `oauth2.TokenSource` from registry config |
| Auth configurator | `pkg/registry/auth_configurator.go` | **New.** Business logic for set/unset/get auth config |
| Token persistence | `pkg/auth/remote/persisting_token_source.go` | Existing token persistence wrapper (reused) |
| Secrets manager | `pkg/secrets/` | Existing encrypted storage for refresh tokens (reused) |
| Config storage | `pkg/config/config.go` | Extended with `RegistryAuth` section |

The `pkg/registry/auth/` package is deliberately separate from `pkg/auth/`
because registry auth has different concerns than MCP server auth — different
config model, different token persistence keys, and different lifecycle. The
package is a thin orchestration layer that composes from existing primitives
in `pkg/auth/oauth/` and `pkg/auth/remote/` without re-implementing the
OAuth lifecycle. It does not define its own `TokenSource` interface — it uses
`oauth2.TokenSource` from `golang.org/x/oauth2` directly, and the standard
`oauth2.Transport` serves as the auth-injecting `http.RoundTripper`.

#### API Changes

**Token source type:** The registry auth package uses `oauth2.TokenSource`
from `golang.org/x/oauth2` directly — no custom interface is defined. The
existing ToolHive auth infrastructure (`PersistingTokenSource`,
`BearerTokenSource`, `MonitoredTokenSource`, `Flow.TokenSource()`) already
implements `oauth2.TokenSource`, so the registry auth layer composes from
these without adapter code. The standard `oauth2.Transport` is used as the
`http.RoundTripper` to inject `Authorization: Bearer` headers.

**Modified signatures:**

```go
// pkg/registry/api/client.go — added tokenSource parameter
func NewClient(baseURL string, allowPrivateIp bool, tokenSource oauth2.TokenSource) (Client, error)

// pkg/registry/provider_api.go — added tokenSource parameter
func NewAPIRegistryProvider(apiURL string, allowPrivateIp bool, tokenSource oauth2.TokenSource) (*APIRegistryProvider, error)

// pkg/registry/provider_cached.go — added tokenSource parameter
func NewCachedAPIRegistryProvider(apiURL string, allowPrivateIp bool, usePersistent bool, tokenSource oauth2.TokenSource) (*CachedAPIRegistryProvider, error)
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
- The `Audience` field is an IdP-specific parameter. It is **not** the
  RFC 8707 `resource` parameter, which requires an absolute `https://` URI.
  The audience is wired into the OAuth authorization request via
  `OAuthParams["audience"]`, which `buildAuthURL()` in `pkg/auth/oauth/flow.go`
  passes as a query parameter. Per-IdP configuration:

  | Identity Provider | Audience/Resource Mechanism | RFC-0043 Configuration |
  |-------------------|---------------------------|------------------------|
  | Auth0 | `audience` query param (authorize + token request) | `--audience <api-identifier>` |
  | Okta (Custom AS) | `audience` query param in authorization request | `--audience <custom-as-audience>` |
  | Okta (Org AS) | Server-side only, not configurable by client | `--audience` not needed; omit |
  | Azure AD / Entra | Resource encoded in scopes: `api://<id>/.default` | `--scopes api://<id>/.default` (not `--audience`) |
  | Keycloak | Audience mapper configured server-side | Typically not needed as query param |
- The secret key for `CachedRefreshTokenRef` is derived from the registry
  URL and issuer (e.g., `REGISTRY_OAUTH_<hex(sha256(registry_url + "\0" + issuer)[:4])>`)
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

**Authentication:**

```bash
# Explicitly authenticate to the configured registry (opens browser)
thv registry login
# → Opening browser for authentication...
# → Authenticated to https://registry.company.com

# Remove cached tokens (auth config is preserved for re-login)
thv registry logout
```

The `thv registry login` command eagerly triggers the browser OAuth flow and
confirms success. It is functionally equivalent to what happens implicitly on
the first registry API call, but provides an explicit entry point for:
- Bootstrapping `thv serve` (authenticate before starting the API server)
- Verifying auth configuration works after setup
- Re-authenticating when refresh tokens expire
- Scripting: `thv registry login && thv serve`

The `thv registry logout` command clears cached tokens from the secrets
manager (deleting the stored refresh token) and resets the
`CachedRefreshTokenRef` field in config to empty. The auth configuration
(issuer, client ID, scopes, audience) is preserved, allowing the user to
re-authenticate with `thv registry login` without reconfiguring. To remove
auth configuration entirely, use `thv config unset-registry-auth`.

**Pre-loaded configuration (MDM/enterprise):** Organizations using MDM
solutions (Jamf, Intune, etc.) can deploy `~/.config/toolhive/config.yaml`
with the `registry_auth` section pre-populated (issuer, client_id, scopes,
audience). The first `thv registry login` or registry command completes the
OAuth flow. For fully non-interactive scenarios (service accounts, CI/CD),
Phase 2 bearer token support provides the path.

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
| `thv serve` (headless context) | Uses cached tokens from secrets manager. If no cached tokens exist, returns a structured `registry_auth_required` JSON error directing the user to run `thv registry login`. Browser flow is never triggered from API server context. |
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
While `pkg/auth/` is a platform-level auth capability (not MCP-specific),
registry auth has different concerns than MCP server auth: different config
model (`RegistryOAuthConfig` vs `remote.Config`), different secret key
derivation (URL-hashed keys vs per-workload keys), and different lifecycle
(CLI session vs long-running MCP workload). The `pkg/registry/auth/` package
is a thin orchestration layer that composes from existing primitives —
`oauth.NewFlow`, `oauth.CreateOAuthConfigFromOIDC`,
`remote.NewPersistingTokenSource` — without re-implementing them. It uses
`oauth2.TokenSource` directly (no custom interface), so all existing token
source implementations work without adapters. Phase 2 bearer token support
will import `pkg/auth/remote.BearerTokenSource` rather than duplicating it.

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

**Why nonce is not used (OIDC ID tokens):**
The `openid` scope is included by default to obtain an OIDC-compatible token
exchange. However, the implementation does not consume or validate the ID
token — only the access token (for registry API calls) and refresh token
(for persistence) are used. Per OIDC Core §3.1.2.1, the `nonce` parameter
is OPTIONAL for Authorization Code flow and only mandatory when consuming
ID token claims. If a future version adds user identity display (e.g.,
"Authenticated as user@company.com"), a `nonce` MUST be generated, sent in
the authorization request, and validated in the returned ID token before
trusting the `sub`/`email` claims.

**Why `thv registry login` instead of `thv login`:**
ToolHive has multiple authentication contexts: registry auth (this RFC) and
remote MCP server auth (`pkg/auth/remote/`). A top-level `thv login` would
be ambiguous about which auth target it addresses, and would conflict with
any future `thv login <mcp-server-url>` command for MCP server-level auth.
`thv registry login` is scoped under the `registry` subcommand group,
co-located with `thv registry list` and `thv registry info`.

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
triggered from within an API server handling HTTP requests. `thv serve`
will never call `browser.OpenURL()` — the "interactive context" check in
the token lifecycle is a boolean flag set during token source construction,
not runtime detection. When `thv serve` needs to access an authenticated
registry (e.g., ToolHive Studio requests server list), it relies on cached
tokens obtained from a prior CLI session. If no cached tokens exist or the
refresh token has expired, the API server returns a structured JSON error:

```json
{
  "code": "registry_auth_required",
  "message": "Registry authentication required. Run 'thv registry login' to authenticate."
}
```

The user must run `thv registry login` in a terminal first to complete the
browser flow and cache tokens. Phase 2 bearer token support will provide a
non-interactive alternative for `thv serve` deployments.

**`thv serve` auth status API:**
The existing `GET /api/v1beta/registry` response is extended to include auth
status, enabling ToolHive Studio to display the correct UI state without
implementing its own auth logic:

```json
{
  "name": "default",
  "type": "api",
  "source": "https://registry.company.com/api",
  "auth_status": "configured",
  "auth_type": "oauth"
}
```

The `auth_status` field has four possible values:
- `"none"` — no registry auth configured
- `"configured"` — auth configured but no cached tokens (user needs to login)
- `"authenticated"` — auth configured with valid cached tokens
- `"expired"` — auth configured but cached tokens have expired

The `auth_type` field is `"oauth"`, `"bearer"` (Phase 2), or `""` (none).
**Tokens are never exposed via this API** — only status and non-secret
metadata. Studio should use `thv serve` as an authenticated proxy to the
registry, never obtaining tokens directly.

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
| OIDC discovery hijack | DNS hijack redirects discovery to attacker-controlled endpoints | Low (HTTPS + issuer comparison) | High |

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
  config. This confirms the URL is reachable, serves a valid
  `.well-known/openid-configuration` document, and — critically — that the
  `issuer` field in the returned document matches the configured issuer URL
  exactly (per OIDC Discovery §4.3 and RFC 8414 §3.3). A mismatch is
  rejected as a potential MITM or DNS hijack. The issuer URL **must** use
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
| OIDC discovery hijack | Issuer claim in discovery document must exactly match the configured issuer URL per OIDC Discovery §4.3; mismatch causes hard failure. HTTPS on the discovery request prevents in-transit tampering. |

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

## Implementation Guide

This section provides implementation-ready code for the four gaps that would
otherwise block a team from translating the RFC into working Go. All snippets
are grounded in the actual signatures of the existing primitives — do not
substitute hypothetical wrappers.

### Gap 1: Constructor Signature for `NewOAuthTokenSource`

The constructor must supply everything that the three underlying primitives
require. Reading their actual signatures:

```go
// pkg/auth/oauth/flow.go
func NewFlow(config *Config) (*Flow, error)

// oauth.Config fields that must be populated:
//   ClientID     string   (required, validates != "")
//   AuthURL      string   (required, validates != "")
//   TokenURL     string   (required, validates != "")
//   Scopes       []string
//   UsePKCE      bool
//   CallbackPort int
//   OAuthParams  map[string]string  // used to pass audience

// pkg/auth/remote/persisting_token_source.go
func NewPersistingTokenSource(source oauth2.TokenSource, persister TokenPersister) *PersistingTokenSource
// TokenPersister = func(refreshToken string, expiry time.Time) error

// pkg/auth/remote/persisting_token_source.go
func CreateTokenSourceFromCached(
    config *oauth2.Config,
    refreshToken string,
    expiry time.Time,
) oauth2.TokenSource
// Requires an *oauth2.Config with ClientID, Endpoint.AuthURL, Endpoint.TokenURL already set.
```

`NewOAuthTokenSource` must therefore receive:
1. The `RegistryOAuthConfig` (issuer, client ID, scopes, audience, callback
   port, cached token reference).
2. A `secrets.Provider` to retrieve and persist the refresh token — **may be
   nil** (see Gap 4).
3. The registry URL — needed to derive the secret storage key and to pass the
   `TokenPersister` closure enough context to call `SetSecret`.
4. An `interactive bool` — controls whether the browser flow is permitted (see
   Gap 3).
5. A `context.Context` — required by OIDC discovery
   (`oauth.CreateOAuthConfigFromOIDC` calls `discoverOIDCEndpointsWithClient`
   which takes a `ctx`), and by `secrets.Provider.GetSecret` /
   `secrets.Provider.SetSecret`.

Proposed signature in `pkg/registry/auth/oauth_token_source.go`:

```go
// NewOAuthTokenSource constructs an oauth2.TokenSource for a registry that
// requires OAuth/OIDC authentication.
//
// ctx is used for OIDC discovery and secrets operations during construction.
// oauthCfg provides issuer, client ID, scopes, audience, and the cached
// refresh-token reference written by a prior login.
// registryURL is the base URL of the registry (used to derive the secret key
// for token storage).
// secretsProvider may be nil when the secrets manager is not configured; in
// that case tokens are never persisted across invocations.
// interactive controls whether a browser OAuth flow is permitted; set to false
// for thv serve and other headless contexts.
func NewOAuthTokenSource(
    ctx         context.Context,
    oauthCfg    *config.RegistryOAuthConfig,
    registryURL string,
    secretsProvider secrets.Provider,  // may be nil
    interactive bool,
) (oauth2.TokenSource, error)
```

The function returns a bare `oauth2.TokenSource` (not the concrete
`*oauthTokenSource`) so callers depend only on the standard interface. The
internal struct is unexported:

```go
// oauthTokenSource implements oauth2.TokenSource for registry authentication.
type oauthTokenSource struct {
    oauthCfg        *config.RegistryOAuthConfig
    secretKey       string          // derived: REGISTRY_OAUTH_<hash>
    secretsProvider secrets.Provider // nil if not configured
    interactive     bool

    // populated lazily by the first Token() call via OIDC discovery
    oauth2Cfg       *oauth2.Config  // token-endpoint config for refresh calls
    inner           oauth2.TokenSource
    mu              sync.Mutex
}
```

`NewOAuthTokenSource` does **not** call OIDC discovery eagerly — it stores the
config and returns the struct. Discovery is deferred to the first `Token()`
call so that construction never blocks and never opens a browser. This mirrors
the pattern in `remote.Handler.Authenticate`, where `buildOAuthFlowConfig` and
`performOAuthFlow` are invoked only when a token is actually needed.

---

### Gap 2: `Token()` Method — Composing the Three Primitives

The `Token()` method implements the full cache → restore → browser flow
lifecycle shown in the RFC flowchart. The model is `handler.go`'s
`tryRestoreFromCachedTokens` + `performOAuthFlow` + `wrapWithPersistence`,
adapted to the simpler registry-auth config model (no DCR, no RFC 9728
discovery — issuer is always explicit).

```go
// Token implements oauth2.TokenSource.
// Thread-safe: the first call triggers OIDC discovery and, if needed, the
// full OAuth flow. Subsequent calls use the in-memory token source cached in
// ts.inner, which handles silent refresh via oauth2.ReuseTokenSource.
func (ts *oauthTokenSource) Token() (*oauth2.Token, error) {
    ts.mu.Lock()
    defer ts.mu.Unlock()

    // Fast path: in-memory token source already initialised (covers both
    // "restored from secrets" and "completed browser flow" cases).
    if ts.inner != nil {
        return ts.inner.Token()
    }

    // ── Step 1: OIDC discovery ──────────────────────────────────────────
    // oauth.CreateOAuthConfigFromOIDC contacts the issuer's
    // .well-known/openid-configuration endpoint, validates the document,
    // and returns an *oauth.Config with AuthURL and TokenURL populated.
    //
    // Signature (pkg/auth/oauth/oidc.go):
    //   func CreateOAuthConfigFromOIDC(
    //       ctx context.Context,
    //       issuer, clientID, clientSecret string,
    //       scopes []string,
    //       usePKCE bool,
    //       callbackPort int,
    //       resource string,
    //   ) (*Config, error)
    //
    // Notes:
    //   - clientSecret is "" (public client, PKCE only).
    //   - usePKCE is always true (mandatory per RFC, not configurable).
    //   - resource is "" unless the IdP requires RFC 8707; the audience
    //     parameter is passed via OAuthParams instead (see Step 3).
    ctx := context.Background() // long-lived; not tied to a request context
    flowCfg, err := oauth.CreateOAuthConfigFromOIDC(
        ctx,
        ts.oauthCfg.Issuer,
        ts.oauthCfg.ClientID,
        "",    // no client secret — public client
        ts.oauthCfg.Scopes,
        true,  // usePKCE — always S256, not configurable
        ts.oauthCfg.CallbackPort,
        "",    // resource — empty; audience sent via OAuthParams
    )
    if err != nil {
        return nil, fmt.Errorf("registry auth: OIDC discovery failed for %s: %w",
            ts.oauthCfg.Issuer, err)
    }

    // Build the minimal *oauth2.Config needed by CreateTokenSourceFromCached
    // and by oauth2.Config.TokenSource for refresh calls.
    ts.oauth2Cfg = &oauth2.Config{
        ClientID: ts.oauthCfg.ClientID,
        Scopes:   ts.oauthCfg.Scopes,
        Endpoint: oauth2.Endpoint{
            AuthURL:  flowCfg.AuthURL,
            TokenURL: flowCfg.TokenURL,
        },
    }

    // ── Step 2: Try to restore from cached refresh token ───────────────
    // Mirrors handler.go tryRestoreFromCachedTokens, but simpler:
    // no DCR, no dynamic issuer discovery.
    if ts.oauthCfg.CachedRefreshTokenRef != "" && ts.secretsProvider != nil {
        tokenSource, err := ts.tryRestoreFromCache(ctx)
        if err != nil {
            // Invalid or expired cached tokens — fall through to browser flow.
            slog.Warn("registry auth: cached tokens invalid, will re-authenticate",
                "error", err)
        } else {
            slog.Debug("registry auth: restored session from cached tokens")
            ts.inner = tokenSource
            return ts.inner.Token()
        }
    }

    // ── Step 3: Browser OAuth flow (interactive only) ──────────────────
    if !ts.interactive {
        return nil, fmt.Errorf(
            "registry authentication required but no interactive context is " +
            "available. Run 'thv registry login' to authenticate, then restart " +
            "thv serve")
    }

    tokenSource, err := ts.performBrowserFlow(ctx, flowCfg)
    if err != nil {
        return nil, fmt.Errorf("registry auth: OAuth flow failed: %w", err)
    }

    ts.inner = tokenSource
    return ts.inner.Token()
}

// tryRestoreFromCache mirrors handler.go's tryRestoreFromCachedTokens.
//
// It calls remote.CreateTokenSourceFromCached (pkg/auth/remote/persisting_token_source.go):
//   func CreateTokenSourceFromCached(
//       config *oauth2.Config,
//       refreshToken string,
//       expiry time.Time,
//   ) oauth2.TokenSource
//
// Then wraps the result in remote.NewPersistingTokenSource so that any token
// rotation by the IdP is written back to the secrets manager automatically.
func (ts *oauthTokenSource) tryRestoreFromCache(ctx context.Context) (oauth2.TokenSource, error) {
    // Retrieve the refresh token from the secrets manager.
    refreshToken, err := ts.secretsProvider.GetSecret(ctx, ts.secretKey)
    if err != nil {
        return nil, fmt.Errorf("failed to retrieve cached refresh token: %w", err)
    }

    // CreateTokenSourceFromCached constructs a ReuseTokenSource that triggers
    // an immediate refresh (access token is intentionally empty) and then
    // silently refreshes when the access token expires.
    baseSource := remote.CreateTokenSourceFromCached(
        ts.oauth2Cfg,
        refreshToken,
        time.Time{}, // expiry unknown; access token is empty so refresh fires immediately
    )

    // Eagerly validate that the refresh token is still accepted by the IdP.
    if _, err := baseSource.Token(); err != nil {
        return nil, fmt.Errorf("cached tokens are invalid or expired: %w", err)
    }

    // Wrap with PersistingTokenSource so IdP-rotated refresh tokens are saved.
    // remote.NewPersistingTokenSource signature:
    //   func NewPersistingTokenSource(
    //       source oauth2.TokenSource,
    //       persister TokenPersister,      // func(refreshToken string, expiry time.Time) error
    //   ) *PersistingTokenSource
    return remote.NewPersistingTokenSource(baseSource, ts.makePersister(ctx)), nil
}

// performBrowserFlow runs the interactive PKCE OAuth flow via oauth.NewFlow.
//
// oauth.NewFlow signature (pkg/auth/oauth/flow.go):
//   func NewFlow(config *Config) (*Flow, error)
//
// flow.Start signature:
//   func (f *Flow) Start(ctx context.Context, skipBrowser bool) (*TokenResult, error)
//
// flow.TokenSource signature:
//   func (f *Flow) TokenSource() oauth2.TokenSource
//
// After the flow completes, the refresh token is persisted immediately (the
// PersistingTokenSource handles future rotations).
func (ts *oauthTokenSource) performBrowserFlow(
    ctx context.Context,
    flowCfg *oauth.Config,
) (oauth2.TokenSource, error) {
    // Inject the audience as an extra OAuth parameter if configured.
    // oauth.Config.OAuthParams is passed verbatim as AuthCodeOption values in
    // buildAuthURL, so any key/value pair lands in the authorization request.
    if ts.oauthCfg.Audience != "" {
        if flowCfg.OAuthParams == nil {
            flowCfg.OAuthParams = make(map[string]string)
        }
        flowCfg.OAuthParams["audience"] = ts.oauthCfg.Audience
    }

    flow, err := oauth.NewFlow(flowCfg)
    if err != nil {
        return nil, fmt.Errorf("failed to create OAuth flow: %w", err)
    }

    // Start opens a browser and blocks until the callback is received or ctx
    // is cancelled. skipBrowser=false means we call browser.OpenURL().
    result, err := flow.Start(ctx, false /* skipBrowser */)
    if err != nil {
        return nil, err
    }

    slog.Info("registry auth: OAuth flow completed successfully")

    // Persist the refresh token immediately so the next CLI invocation can
    // skip the browser flow.
    if result.RefreshToken != "" && ts.secretsProvider != nil {
        if err := ts.secretsProvider.SetSecret(ctx, ts.secretKey, result.RefreshToken); err != nil {
            slog.Warn("registry auth: failed to persist refresh token", "error", err)
            // Non-fatal: token works for this session even without persistence.
        } else {
            slog.Debug("registry auth: refresh token persisted", "key", ts.secretKey)
        }
    }

    // flow.TokenSource() returns the oauth2.ReuseTokenSource constructed in
    // processToken — it already holds the access token and will refresh silently.
    baseSource := flow.TokenSource()

    // Wrap with PersistingTokenSource to capture IdP-rotated refresh tokens.
    if ts.secretsProvider != nil {
        return remote.NewPersistingTokenSource(baseSource, ts.makePersister(ctx)), nil
    }
    return baseSource, nil
}

// makePersister returns a TokenPersister closure that writes the refresh token
// to the secrets manager under the registry-specific secret key.
//
// remote.TokenPersister type (pkg/auth/remote/persisting_token_source.go):
//   type TokenPersister func(refreshToken string, expiry time.Time) error
func (ts *oauthTokenSource) makePersister(ctx context.Context) remote.TokenPersister {
    return func(refreshToken string, _ time.Time) error {
        if err := ts.secretsProvider.SetSecret(ctx, ts.secretKey, refreshToken); err != nil {
            return fmt.Errorf("registry auth: failed to persist rotated refresh token: %w", err)
        }
        slog.Debug("registry auth: persisted rotated refresh token", "key", ts.secretKey)
        return nil
    }
}
```

**Secret key derivation** (used to set `ts.secretKey` in
`NewOAuthTokenSource`):

```go
// deriveSecretKey returns a stable, collision-resistant key for storing the
// registry's refresh token in the secrets manager.
//
// Format: REGISTRY_OAUTH_<hex(sha256(registryURL + "\x00" + issuer)[:4])>
// The null byte prevents ambiguous concatenation (e.g. "ab" + "cd" vs "a" + "bcd").
// 4 bytes = 8 hex characters, matching the existing precedent in
// provider_cached.go (which uses hash[:4] for cache file names).
func deriveSecretKey(registryURL, issuer string) string {
    h := sha256.Sum256([]byte(registryURL + "\x00" + issuer))
    return fmt.Sprintf("REGISTRY_OAUTH_%x", h[:4])
}
```

---

### Gap 3: Threading the `interactive` Boolean

The `interactive` flag must be set at construction time — not detected at
runtime — because the token source is created once and reused for the lifetime
of the provider. The two call sites are:

- **CLI commands** (`thv registry list`, `thv run`, `thv registry login`):
  `interactive = true`
- **`thv serve`** (headless API server): `interactive = false`

**Step 1: `NewRegistryProvider` gains an `interactive` parameter.**

The current signature in `pkg/registry/factory.go` is:

```go
// Current (no auth support)
func NewRegistryProvider(cfg *config.Config) (Provider, error)
```

Proposed change:

```go
// Updated — accepts interactive flag so the token source is built correctly
// for both CLI (interactive=true) and thv serve (interactive=false).
func NewRegistryProvider(cfg *config.Config, interactive bool) (Provider, error) {
    // ...
    if cfg != nil && len(cfg.RegistryApiUrl) > 0 {
        tokenSource, err := resolveTokenSource(cfg, interactive)
        if err != nil {
            return nil, fmt.Errorf("failed to resolve registry token source: %w", err)
        }
        // When a tokenSource is non-nil, skip the validation probe:
        // the probe would trigger the browser flow inside a timeout and fail.
        if tokenSource != nil {
            return NewCachedAPIRegistryProvider(cfg.RegistryApiUrl, cfg.AllowPrivateRegistryIp, true, tokenSource)
        }
        provider, err := NewCachedAPIRegistryProvider(cfg.RegistryApiUrl, cfg.AllowPrivateRegistryIp, true, nil)
        if err != nil {
            return nil, fmt.Errorf("custom registry API at %s is not reachable: %w", cfg.RegistryApiUrl, err)
        }
        return provider, nil
    }
    // ... rest of existing priority chain unchanged
}
```

**Step 2: `resolveTokenSource` passes `interactive` into `NewOAuthTokenSource`.**

```go
// resolveTokenSource reads registry auth configuration and constructs the
// appropriate oauth2.TokenSource, or returns nil if auth is not configured.
// This is an unexported helper called only from NewRegistryProvider.
func resolveTokenSource(cfg *config.Config, interactive bool) (oauth2.TokenSource, error) {
    if cfg.RegistryAuth == nil || cfg.RegistryAuth.Type == "" {
        return nil, nil // no auth configured — backward compatible
    }

    switch cfg.RegistryAuth.Type {
    case "oauth":
        if cfg.RegistryAuth.OAuth == nil {
            return nil, fmt.Errorf("registry_auth.type is 'oauth' but oauth config is missing")
        }
        secretsProvider := resolveSecretsProvider(cfg) // see Gap 4
        ctx := context.Background()
        return registryauth.NewOAuthTokenSource(
            ctx,
            cfg.RegistryAuth.OAuth,
            cfg.RegistryApiUrl,
            secretsProvider, // may be nil
            interactive,
        )
    default:
        return nil, fmt.Errorf("unsupported registry_auth.type: %q", cfg.RegistryAuth.Type)
    }
}
```

**Step 3: Call sites pass the correct `interactive` value.**

```go
// cmd/thv/app/root.go (or wherever the registry provider is created for CLI use)
provider, err := registry.NewRegistryProvider(cfg, true /* interactive */)

// cmd/thv/app/serve.go (thv serve — headless, no browser)
provider, err := registry.NewRegistryProvider(cfg, false /* interactive */)
```

The existing `GetDefaultProvider` / `GetDefaultProviderWithConfig` singleton
helpers must also be updated. Because they are used in CLI paths, they should
pass `interactive = true`:

```go
func GetDefaultProviderWithConfig(configProvider config.Provider) (Provider, error) {
    defaultProviderOnce.Do(func() {
        cfg, err := configProvider.LoadOrCreateConfig()
        if err != nil {
            defaultProviderErr = err
            return
        }
        // CLI singleton: interactive = true
        defaultProvider, defaultProviderErr = NewRegistryProvider(cfg, true)
    })
    return defaultProvider, defaultProviderErr
}
```

For `thv serve`, the provider should not use the singleton — it should call
`NewRegistryProvider(cfg, false)` directly, since the serve context is
distinct from CLI invocations.

---

### Gap 4: Secrets Provider Optional Handling

The secrets manager may not be configured (the user has never run `thv secret
setup`). The RFC requires graceful degradation: tokens work for the current
session in memory but are not persisted. This must be handled in
`resolveSecretsProvider`, called from `resolveTokenSource`.

```go
// resolveSecretsProvider attempts to create a secrets provider from config.
// Returns nil (not an error) when secrets are not configured, so the caller
// can degrade gracefully to in-memory-only token storage.
//
// config.Secrets.GetProviderType() returns secrets.ErrSecretsNotSetup when
// SetupCompleted is false and TOOLHIVE_SECRETS_PROVIDER is not set — that is
// the "not configured" sentinel defined in pkg/secrets/factory.go.
func resolveSecretsProvider(cfg *config.Config) secrets.Provider {
    providerType, err := cfg.Secrets.GetProviderType()
    if err != nil {
        if errors.Is(err, secrets.ErrSecretsNotSetup) {
            // Secrets manager not configured — token persistence disabled.
            // The user will be asked to authenticate on every CLI invocation
            // until they run 'thv secret setup'.
            slog.Debug("registry auth: secrets manager not configured; " +
                "refresh tokens will not be persisted across CLI invocations. " +
                "Run 'thv secret setup' to enable persistence.")
            return nil
        }
        // Misconfigured provider (e.g. invalid provider type in env var) —
        // log a warning and degrade rather than failing the entire command.
        slog.Warn("registry auth: failed to determine secrets provider type; " +
            "refresh tokens will not be persisted", "error", err)
        return nil
    }

    // secrets.CreateSecretProvider signature (pkg/secrets/factory.go):
    //   func CreateSecretProvider(managerType ProviderType) (Provider, error)
    provider, err := secrets.CreateSecretProvider(providerType)
    if err != nil {
        // Provider configured but initialization failed (e.g. keyring locked,
        // 1Password CLI not running). Degrade to in-memory-only with a warning.
        slog.Warn("registry auth: failed to initialize secrets provider; " +
            "refresh tokens will not be persisted", "error", err)
        return nil
    }

    return provider
}
```

**Usage in `NewOAuthTokenSource`** — the nil check is applied at every
secrets read/write site inside `oauthTokenSource`:

```go
// Inside tryRestoreFromCache — skip if no provider:
if ts.secretsProvider == nil {
    return nil, fmt.Errorf("no secrets provider available; cannot restore cached tokens")
}
refreshToken, err := ts.secretsProvider.GetSecret(ctx, ts.secretKey)

// Inside performBrowserFlow — skip persistence if no provider:
if result.RefreshToken != "" && ts.secretsProvider != nil {
    if err := ts.secretsProvider.SetSecret(ctx, ts.secretKey, result.RefreshToken); err != nil {
        slog.Warn("registry auth: failed to persist refresh token", "error", err)
        // Non-fatal — continue with in-memory token.
    }
}

// Inside makePersister — only called when ts.secretsProvider != nil (see
// performBrowserFlow and tryRestoreFromCache above), so no extra nil check
// is needed inside the closure.
```

**Consequence on `Token()` control flow** when `secretsProvider` is nil:

```
Token() called
  └─ ts.inner == nil → proceed to OIDC discovery
  └─ CachedRefreshTokenRef != "" but secretsProvider == nil
       → skip restore attempt (guarded by "&& ts.secretsProvider != nil")
  └─ interactive == true → performBrowserFlow
       → persist skipped (guarded by "&& ts.secretsProvider != nil")
       → wrap with plain baseSource (no PersistingTokenSource)
  └─ ts.inner = baseSource (in-memory only, valid for this process lifetime)
```

This matches the behavior described in the RFC's cross-invocation table:
"Secrets manager not configured → tokens work for the current session
(in-memory) but are not persisted across invocations — each CLI run triggers
the browser flow."

---

### Gap 5: `NewClient` — Wiring `oauth2.Transport` into the HTTP Client

The existing `NewClient` in `pkg/registry/api/client.go` builds an
`*http.Client` from `networking.NewHttpClientBuilder`. The transport chain
must be extended so that when a `tokenSource` is non-nil, every outgoing
request gets an `Authorization: Bearer <token>` header injected by
`oauth2.Transport` before it hits the network.

Current signature:
```go
func NewClient(baseURL string, allowPrivateIp bool) (Client, error)
```

Updated signature:
```go
// NewClient creates a new MCP Registry API client.
// tokenSource may be nil — if nil, no Authorization header is added and the
// client behaves identically to the pre-auth implementation.
func NewClient(baseURL string, allowPrivateIp bool, tokenSource oauth2.TokenSource) (Client, error) {
```

Full updated function body:

```go
func NewClient(baseURL string, allowPrivateIp bool, tokenSource oauth2.TokenSource) (Client, error) {
    // Build the base HTTP client with security controls (private IP + HTTPS
    // enforcement). This is unchanged from the pre-auth implementation.
    builder := networking.NewHttpClientBuilder().WithPrivateIPs(allowPrivateIp)
    if allowPrivateIp {
        builder = builder.WithInsecureAllowHTTP(true)
    }
    baseHTTPClient, err := builder.Build()
    if err != nil {
        return nil, fmt.Errorf("failed to build HTTP client: %w", err)
    }

    // When a token source is provided, wrap the existing transport with
    // oauth2.Transport. oauth2.Transport calls tokenSource.Token() on every
    // request, adds "Authorization: Bearer <token>", and handles token
    // expiry + refresh transparently.
    //
    // IMPORTANT: oauth2.Transport.Base must be set to the builder's transport
    // (not nil). If Base is nil, oauth2.Transport falls back to
    // http.DefaultTransport, bypassing the ValidatingTransport that enforces
    // private IP blocking and HTTPS requirements.
    var transport http.RoundTripper = baseHTTPClient.Transport
    if transport == nil {
        transport = http.DefaultTransport
    }
    if tokenSource != nil {
        transport = &oauth2.Transport{
            Source: tokenSource,
            Base:   transport, // preserves ValidatingTransport in the chain
        }
    }

    httpClient := &http.Client{
        Transport: transport,
        Timeout:   baseHTTPClient.Timeout,
    }

    // Trim trailing slash (unchanged from original).
    if len(baseURL) > 0 && baseURL[len(baseURL)-1] == '/' {
        baseURL = baseURL[:len(baseURL)-1]
    }

    return &mcpRegistryClient{
        baseURL:        baseURL,
        httpClient:     httpClient,
        allowPrivateIp: allowPrivateIp,
        userAgent:      versions.GetUserAgent(),
    }, nil
}
```

The import added to `client.go`:

```go
import (
    "golang.org/x/oauth2"
    // ... existing imports
)
```

The resulting round-tripper chain when auth is configured:

```
outgoing request
    → oauth2.Transport          (adds Authorization: Bearer <token>)
        → ValidatingTransport   (enforces HTTPS + private IP rules)
            → net/http default  (actual TLS + TCP)
```

When `tokenSource` is nil, the chain is simply:

```
outgoing request
    → ValidatingTransport
        → net/http default
```

---

### Gap 6: `thv registry login` and `thv registry logout` — Cobra Command Structure

Both commands belong in `cmd/thv/app/registry.go` alongside the existing
`registryListCmd` and `registryInfoCmd`. This follows the existing pattern
where all `thv registry *` subcommands live in one file.

```go
// cmd/thv/app/registry.go

var registryLoginCmd = &cobra.Command{
    Use:   "login",
    Short: "Authenticate to the configured registry",
    Long: `Authenticate to the configured registry using the configured OAuth provider.
Opens a browser window for user authentication. On success, tokens are cached
in the secrets manager for subsequent CLI invocations (no browser needed).

This command is equivalent to what happens implicitly on the first registry API
call, but provides an explicit entry point for:
  - Bootstrapping thv serve: run 'thv registry login' before 'thv serve'
  - Verifying auth configuration works after setup
  - Re-authenticating when refresh tokens expire
  - Scripting: thv registry login && thv serve

Example:
  thv registry login`,
    RunE: registryLoginCmdFunc,
}

var registryLogoutCmd = &cobra.Command{
    Use:   "logout",
    Short: "Remove cached registry credentials",
    Long: `Remove cached OAuth tokens from the secrets manager and clear the
registry_auth section from configuration. The registry URL itself is preserved.

Example:
  thv registry logout`,
    RunE: registryLogoutCmdFunc,
}

func init() {
    rootCmd.AddCommand(registryCmd)

    // Existing subcommands (unchanged)
    registryCmd.AddCommand(registryListCmd)
    registryCmd.AddCommand(registryInfoCmd)

    // New auth subcommands
    registryCmd.AddCommand(registryLoginCmd)
    registryCmd.AddCommand(registryLogoutCmd)

    AddFormatFlag(registryListCmd, &registryFormat)
    registryListCmd.Flags().BoolVar(&refreshRegistry, "refresh", false, "Force refresh registry cache")
    registryListCmd.PreRunE = ValidateFormat(&registryFormat)

    AddFormatFlag(registryInfoCmd, &registryFormat)
    registryInfoCmd.Flags().BoolVar(&refreshRegistry, "refresh", false, "Force refresh registry cache")
    registryInfoCmd.PreRunE = ValidateFormat(&registryFormat)
}
```

The command functions. These use the same config and service patterns as the
existing `setRegistryCmdFunc` / `unsetRegistryCmdFunc` in
`cmd/thv/app/config.go` — load config, call a business-logic layer, reset the
provider cache.

```go
// cmd/thv/app/registry.go

func registryLoginCmdFunc(_ *cobra.Command, _ []string) error {
    cfg, err := config.LoadOrCreateConfig()
    if err != nil {
        return fmt.Errorf("failed to load config: %w", err)
    }

    if cfg.RegistryAuth == nil || cfg.RegistryAuth.Type == "" {
        return fmt.Errorf(
            "no registry auth is configured.\n\n" +
                "Configure authentication first with:\n" +
                "  thv config set-registry-auth --issuer <issuer-url> --client-id <client-id>")
    }

    if cfg.RegistryApiUrl == "" {
        return fmt.Errorf(
            "no registry URL is configured.\n\n" +
                "Configure the registry first with:\n" +
                "  thv config set-registry <url>")
    }

    // Obtain the secrets provider (may be nil if secrets not set up).
    // resolveSecretsProvider is the same helper used by factory.go,
    // extracted to pkg/registry/auth or duplicated here to avoid import cycles.
    secretsProvider := resolveSecretsProviderForCLI(cfg)

    // Build a token source with interactive=true.
    // Calling Token() eagerly exercises the full lifecycle:
    // cache miss → restore miss → browser flow → persist.
    ts, err := registryauth.NewOAuthTokenSource(
        context.Background(),
        cfg.RegistryAuth.OAuth,
        cfg.RegistryApiUrl,
        secretsProvider,
        true, // interactive — browser flow allowed from CLI
    )
    if err != nil {
        return fmt.Errorf("failed to initialize token source: %w", err)
    }

    token, err := ts.Token()
    if err != nil {
        return fmt.Errorf("authentication failed: %w", err)
    }
    if !token.Valid() {
        return fmt.Errorf("authentication succeeded but returned an invalid token")
    }

    fmt.Printf("Authenticated to %s\n", cfg.RegistryApiUrl)

    // Reset the provider cache so subsequent commands pick up the new tokens.
    registry.ResetDefaultProvider()
    return nil
}

func registryLogoutCmdFunc(_ *cobra.Command, _ []string) error {
    cfg, err := config.LoadOrCreateConfig()
    if err != nil {
        return fmt.Errorf("failed to load config: %w", err)
    }

    if cfg.RegistryAuth == nil || cfg.RegistryAuth.Type == "" {
        fmt.Println("No registry auth is currently configured.")
        return nil
    }

    // Delete the cached refresh token from the secrets manager.
    if cfg.RegistryAuth.OAuth != nil && cfg.RegistryAuth.OAuth.CachedRefreshTokenRef != "" {
        secretsProvider := resolveSecretsProviderForCLI(cfg)
        if secretsProvider != nil {
            key := cfg.RegistryAuth.OAuth.CachedRefreshTokenRef
            if err := secretsProvider.DeleteSecret(context.Background(), key); err != nil {
                // Non-fatal: the token may already be gone or the secret key
                // may have changed. Log at debug so the logout still succeeds.
                slog.Debug("failed to delete cached refresh token from secrets manager",
                    "key", key, "error", err)
            }
        }
    }

    // Clear the registry_auth section from config (using the same UpdateConfig
    // pattern as usageMetricsCmdFunc in config.go).
    if err := config.UpdateConfig(func(c *config.Config) {
        c.RegistryAuth = nil
    }); err != nil {
        return fmt.Errorf("failed to update config: %w", err)
    }

    registry.ResetDefaultProvider()
    fmt.Println("Registry credentials cleared.")
    return nil
}

// resolveSecretsProviderForCLI is the cmd-layer equivalent of factory.go's
// resolveSecretsProvider. It is duplicated here rather than imported from
// pkg/registry to avoid a circular import between cmd/thv/app and pkg/registry.
// Consider moving this helper to pkg/registry/auth if both packages need it.
func resolveSecretsProviderForCLI(cfg *config.Config) secrets.Provider {
    providerType, err := cfg.Secrets.GetProviderType()
    if err != nil {
        slog.Debug("secrets not configured; refresh tokens will not be persisted", "reason", err)
        return nil
    }
    provider, err := secrets.CreateSecretProvider(providerType)
    if err != nil {
        slog.Debug("failed to create secrets provider", "error", err)
        return nil
    }
    return provider
}
```

The required new imports for `registry.go`:

```go
import (
    "context"
    "fmt"
    "log/slog"

    "github.com/spf13/cobra"

    "github.com/stacklok/toolhive/pkg/config"
    "github.com/stacklok/toolhive/pkg/registry"
    registryauth "github.com/stacklok/toolhive/pkg/registry/auth"
    "github.com/stacklok/toolhive/pkg/secrets"
    // ... existing imports
)
```

---

## API Integration Guidance

This section provides concrete Go code for the four integration points that
connect the token-source layer (Gaps 1–6 above) to the existing API server
and provider chain in `pkg/api/v1/registry.go` and `pkg/registry/`. All file
paths are relative to the `toolhive` repository root.

---

### API-1: Auth Status Fields in `registryInfo`

**File:** `pkg/api/v1/registry.go`

Add two fields to `registryInfo`. The `omitempty` tags make the fields absent
from responses when no auth is configured, preserving backward compatibility:

```go
type registryInfo struct {
    Name        string       `json:"name"`
    Version     string       `json:"version"`
    LastUpdated string       `json:"last_updated"`
    ServerCount int          `json:"server_count"`
    Type        RegistryType `json:"type"`
    Source      string       `json:"source"`
    // AuthStatus is one of: "none", "configured", "authenticated".
    // Absent from the response when no auth is configured.
    AuthStatus string `json:"auth_status,omitempty"`
    // AuthType is "oauth", "bearer" (Phase 2), or absent when no auth.
    AuthType string `json:"auth_type,omitempty"`
}
```

Add a pure-config helper — no network calls, no secrets reads. The config
singleton is already loaded in memory after the first call to
`configProvider.LoadOrCreateConfig()`:

```go
const (
    authStatusNone          = "none"
    authStatusConfigured    = "configured"
    authStatusAuthenticated = "authenticated"
)

// resolveAuthStatus reads the in-memory config and returns the auth_status
// and auth_type strings for the API response.
//
// Status logic:
//   - RegistryAuth nil or Type=="" → ("", "") — fields omitted entirely
//   - Type=="oauth", CachedRefreshTokenRef=="" → ("configured", "oauth")
//     (issuer/client-id set but user has not yet run `thv registry login`)
//   - Type=="oauth", CachedRefreshTokenRef!="" → ("authenticated", "oauth")
//     (a refresh token reference exists — a prior login succeeded)
//
// Note: "expired" cannot be determined from config alone — it requires
// calling the token source. Return "authenticated" whenever a token ref is
// present; Studio should treat a subsequent 401 from a registry request as
// a signal to prompt re-login.
func resolveAuthStatus(cfg *config.Config) (authStatus, authType string) {
    if cfg == nil || cfg.RegistryAuth == nil || cfg.RegistryAuth.Type == "" {
        return "", ""
    }
    switch cfg.RegistryAuth.Type {
    case "oauth":
        if cfg.RegistryAuth.OAuth == nil ||
            cfg.RegistryAuth.OAuth.CachedRefreshTokenRef == "" {
            return authStatusConfigured, "oauth"
        }
        return authStatusAuthenticated, "oauth"
    default:
        // Unknown type — surface it so Studio can inform the user.
        return authStatusConfigured, cfg.RegistryAuth.Type
    }
}
```

Update `listRegistries` to populate the new fields and handle the
`registryAuthRequiredError` sentinel (defined in API-2):

```go
func (rr *RegistryRoutes) listRegistries(w http.ResponseWriter, _ *http.Request) {
    provider, ok := rr.getCurrentProvider(w)
    if !ok {
        return // getCurrentProvider already wrote the response
    }

    reg, err := provider.GetRegistry()
    if err != nil {
        var authErr *registryAuthRequiredError
        if errors.As(err, &authErr) {
            writeRegistryAuthRequiredError(w)
            return
        }
        http.Error(w, "Failed to get registry", http.StatusInternalServerError)
        return
    }

    registryType, source := rr.getRegistryInfo()

    // Config load is cheap (in-memory singleton after first call).
    cfg, _ := rr.configProvider.LoadOrCreateConfig()
    authStatus, authType := resolveAuthStatus(cfg)

    registries := []registryInfo{
        {
            Name:        defaultRegistryName,
            Version:     reg.Version,
            LastUpdated: reg.LastUpdated,
            ServerCount: len(reg.Servers),
            Type:        registryType,
            Source:      source,
            AuthStatus:  authStatus,
            AuthType:    authType,
        },
    }

    w.Header().Set("Content-Type", "application/json")
    response := registryListResponse{Registries: registries}
    if err := json.NewEncoder(w).Encode(response); err != nil {
        http.Error(w, "Failed to encode response", http.StatusInternalServerError)
    }
}
```

Apply the same `resolveAuthStatus` population to `getRegistry` for symmetry.

---

### API-2: Structured `registry_auth_required` JSON Error

**File:** `pkg/api/v1/registry.go`

The existing handlers use `http.Error` (plain text). The auth error requires a
structured JSON body so Studio can machine-read the `code` field and display
the correct remediation UI without parsing a free-form string.

The `registry.go` handlers do not yet use the `apierrors.ErrorHandler` wrapper
(unlike `workloads.go`, which uses `httperr.WithCode`). Add a targeted helper
that writes the JSON body inline — no need to refactor all handlers in one PR:

```go
// registryAuthRequiredError is a sentinel returned by the registry provider
// chain when no valid token is available and the context is non-interactive
// (i.e., thv serve cannot open a browser).
// Define this in pkg/registry/errors.go (alongside the existing RegistryError
// in pkg/config/errors.go) and import it here. Defining it in pkg/registry
// avoids a circular dependency because pkg/registry does not import pkg/api.
type registryAuthRequiredError struct {
    RegistryURL string
}

func (e *registryAuthRequiredError) Error() string {
    return fmt.Sprintf("registry at %s requires authentication; "+
        "run 'thv registry login' to authenticate", e.RegistryURL)
}

// registryAuthErrorResponse is the JSON body for HTTP 503 auth-required errors.
type registryAuthErrorResponse struct {
    Code    string `json:"code"`
    Message string `json:"message"`
}

// writeRegistryAuthRequiredError writes a structured JSON 503 response.
// Callers must not have written any headers before calling this function.
//
// HTTP 503 Service Unavailable is the correct status here: the incoming client
// (Studio) is authenticated to the thv serve API — the problem is that thv
// serve itself lacks a valid registry credential. HTTP 401 would incorrectly
// imply Studio needs to send credentials; 503 correctly signals that a
// server-side dependency (the CLI credential cache) is temporarily unavailable.
// Studio can detect this specific condition by checking the "code" field.
func writeRegistryAuthRequiredError(w http.ResponseWriter) {
    body := registryAuthErrorResponse{
        Code:    "registry_auth_required",
        Message: "Registry authentication required. Run 'thv registry login' to authenticate.",
    }
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusServiceUnavailable) // 503, not 401
    _ = json.NewEncoder(w).Encode(body) // header already written; encode errors not actionable
}
```

---

### API-3: 401/403 Detection in the HTTP Client and Provider

**File A:** `pkg/registry/api/client.go`

The three HTTP methods (`GetServer`, `fetchServersPage`, `SearchServers`)
currently return `fmt.Errorf("API returned status %d: %s", ...)` on non-2xx
responses. Replace with a typed sentinel so callers can use `errors.Is`
without string parsing:

```go
// ErrRegistryUnauthorized is returned when the registry responds with
// HTTP 401 Unauthorized or 403 Forbidden.
var ErrRegistryUnauthorized = errors.New("registry requires authentication")

// registryHTTPError is returned for all non-2xx responses.
// When StatusCode is 401 or 403, Unwrap returns ErrRegistryUnauthorized so
// callers can use errors.Is(err, api.ErrRegistryUnauthorized).
type registryHTTPError struct {
    StatusCode int
    Body       string
}

func (e *registryHTTPError) Error() string {
    return fmt.Sprintf("registry returned HTTP %d: %s", e.StatusCode, e.Body)
}

func (e *registryHTTPError) Unwrap() error {
    if e.StatusCode == http.StatusUnauthorized ||
        e.StatusCode == http.StatusForbidden {
        return ErrRegistryUnauthorized
    }
    return nil
}
```

In `GetServer`, `fetchServersPage`, and `SearchServers`, replace all three
occurrences of:

```go
// Before (three identical sites)
return nil, fmt.Errorf("API returned status %d: %s", resp.StatusCode, string(body))

// After
return nil, &registryHTTPError{StatusCode: resp.StatusCode, Body: string(body)}
```

**File B:** `pkg/registry/provider_api.go`

The validation probe in `NewAPIRegistryProvider` must be skipped when auth
is configured (the probe would invoke the token source before a browser is
available), and must produce an actionable message when a probe does run and
the registry returns 401:

```go
func NewAPIRegistryProvider(
    apiURL string,
    allowPrivateIp bool,
    tokenSource oauth2.TokenSource, // new parameter — nil for no auth
) (*APIRegistryProvider, error) {
    client, err := api.NewClient(apiURL, allowPrivateIp, tokenSource)
    if err != nil {
        return nil, fmt.Errorf("failed to create API client: %w", err)
    }

    p := &APIRegistryProvider{
        apiURL:         apiURL,
        allowPrivateIp: allowPrivateIp,
        client:         client,
    }
    p.BaseProvider = NewBaseProvider(p.GetRegistry)

    // Skip the validation probe when a token source is configured.
    // Running the probe would invoke the token source (potentially triggering
    // a browser flow) inside a 10-second timeout that cannot complete
    // interactively. The first real API call performs implicit validation.
    if tokenSource != nil {
        return p, nil
    }

    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    _, err = client.ListServers(ctx, &api.ListOptions{Limit: 1})
    if err != nil {
        if errors.Is(err, api.ErrRegistryUnauthorized) {
            // Produce the actionable message specified in the RFC.
            return nil, fmt.Errorf(
                "registry at %s returned 401 Unauthorized.\n\n"+
                    "If this registry requires authentication, configure it with:\n"+
                    "  thv config set-registry-auth "+
                    "--issuer <issuer-url> --client-id <client-id>",
                apiURL,
            )
        }
        return nil, fmt.Errorf("API endpoint not functional: %w", err)
    }

    return p, nil
}
```

**File C:** `pkg/registry/provider_api.go` — wrapping 401 as `*registryAuthRequiredError`

When a live registry call returns 401 in the API server context, wrap it as a
`*registryAuthRequiredError` so the handler in API-2 catches it without
needing to understand `api.ErrRegistryUnauthorized`:

```go
func (p *APIRegistryProvider) GetRegistry() (*types.Registry, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 60*time.Second)
    defer cancel()

    servers, err := p.client.ListServers(ctx, nil)
    if err != nil {
        if errors.Is(err, api.ErrRegistryUnauthorized) {
            return nil, &registryAuthRequiredError{RegistryURL: p.apiURL}
        }
        return nil, fmt.Errorf("failed to list servers from API: %w", err)
    }
    // ... remainder unchanged
}
```

The same `errors.Is(err, api.ErrRegistryUnauthorized)` guard belongs in
`GetServer` and `SearchServers` if those methods are reachable from API
server paths.

---

### API-4: `getCurrentProvider()` and the `interactive=false` Contract

**File:** `pkg/api/v1/registry.go` and `pkg/registry/factory.go`

The `interactive` flag is set at token source **construction time** — it is a
boolean field on `oauthTokenSource`, not a runtime check. The singleton
provider created by `GetDefaultProviderWithConfig` serves the API server, so
it must be constructed with `interactive=false`.

**Updated `getCurrentProvider`** — adds auth-error detection:

```go
// getCurrentProvider returns the current registry provider. If the provider
// cannot be created because registry auth is required and no cached token is
// available (thv serve context, interactive=false), it writes the structured
// JSON 401 response and returns (nil, false).
func (rr *RegistryRoutes) getCurrentProvider(w http.ResponseWriter) (regpkg.Provider, bool) {
    provider, err := regpkg.GetDefaultProviderWithConfig(rr.configProvider)
    if err != nil {
        var authErr *registryAuthRequiredError
        if errors.As(err, &authErr) {
            writeRegistryAuthRequiredError(w)
            return nil, false
        }
        http.Error(w, "Failed to get registry provider", http.StatusInternalServerError)
        slog.Error("failed to get registry provider", "error", err)
        return nil, false
    }
    return provider, true
}
```

**Updated `GetDefaultProviderWithConfig`** in `pkg/registry/factory.go`:

```go
// GetDefaultProviderWithConfig returns the singleton registry provider used
// by the API server (thv serve). The provider is always constructed with
// interactive=false — the API server must never open a browser.
//
// CLI commands that need interactive auth (thv registry list, thv run,
// thv registry login) must call NewRegistryProvider(cfg, true) directly,
// bypassing this singleton. Using this singleton from a CLI context will
// cause the browser flow to never trigger.
func GetDefaultProviderWithConfig(configProvider config.Provider) (Provider, error) {
    defaultProviderOnce.Do(func() {
        cfg, err := configProvider.LoadOrCreateConfig()
        if err != nil {
            defaultProviderErr = err
            return
        }
        // interactive=false: API server (thv serve) never opens a browser.
        // CLI commands call NewRegistryProvider(cfg, true) directly.
        defaultProvider, defaultProviderErr = NewRegistryProvider(cfg, false)
    })
    return defaultProvider, defaultProviderErr
}
```

**CLI call sites** must bypass the singleton to get `interactive=true`. This
is the critical correctness constraint — failing to do this causes the browser
flow to silently never trigger:

```go
// cmd/thv/app/registry.go — thv registry list, thv run, thv registry login
cfg, err := config.LoadOrCreateConfig()
if err != nil {
    return fmt.Errorf("failed to load config: %w", err)
}
// interactive=true: CLI context, browser OAuth flow is permitted.
provider, err := registry.NewRegistryProvider(cfg, true)
if err != nil {
    return err
}
```

**`interactive` flag contract summary:**

| Call site | `interactive` | Behavior when no cached token |
|---|---|---|
| `GetDefaultProviderWithConfig` (API server singleton) | `false` | Returns `*registryAuthRequiredError` → HTTP 503 JSON |
| CLI: `thv registry list`, `thv run` | `true` | Opens browser OAuth flow |
| CLI: `thv registry login` | `true` (explicit) | Eagerly warms the token cache |
| `thv serve` startup | `false` (via singleton) | Returns error immediately with actionable message |

---

### Full Error Flow: `GET /api/v1beta/registry` in `thv serve`

When Studio calls `GET /api/v1beta/registry` and the registry requires auth
but no cached token exists, the call traverses the following path:

```
listRegistries()
  └─ getCurrentProvider(w)
       └─ GetDefaultProviderWithConfig()          [interactive=false singleton]
            └─ NewRegistryProvider(cfg, false)
                 └─ resolveTokenSource(cfg, false) → oauth2.TokenSource{interactive:false}
                 └─ NewCachedAPIRegistryProvider(url, ..., tokenSource)
                      └─ NewAPIRegistryProvider(url, ..., tokenSource)
                           └─ tokenSource != nil → validation probe SKIPPED
            ← provider returned (construction succeeds; no network call yet)
  └─ provider.GetRegistry()
       └─ CachedAPIRegistryProvider.GetRegistry()
            └─ APIRegistryProvider.GetRegistry()
                 └─ client.ListServers(ctx, nil)
                      └─ oauth2.Transport.RoundTrip()
                           └─ tokenSource.Token()   [interactive=false]
                                └─ in-memory cache miss
                                └─ secretsProvider has no token
                                └─ interactive==false → return error immediately
                      OR: token acquired but registry returns HTTP 401
                 └─ errors.Is(err, api.ErrRegistryUnauthorized) → true
            └─ return &registryAuthRequiredError{RegistryURL: ...}
  └─ errors.As(err, &authErr) → true
  └─ writeRegistryAuthRequiredError(w)
       └─ HTTP 503
          Content-Type: application/json
          {"code":"registry_auth_required",
           "message":"Registry authentication required. Run 'thv registry login' to authenticate."}
```

---

## Security Implementation Constraints

This section contains binding constraints for implementing agents. Each
constraint resolves an ambiguity in the Implementation Guide above or adds
a guardrail the compiler cannot enforce. Deviating from these rules
introduces a security defect.

### S1. Secret Key Derivation — Exact Formula

**The secret key uses `h[:4]` (8 hex chars) to match the existing codebase
precedent.**

`pkg/registry/provider_cached.go` (lines 70–71) already uses this pattern
for hash-derived storage keys in the registry package:

```go
// Existing precedent — provider_cached.go
hash := sha256.Sum256([]byte(apiURL))
cacheFile := fmt.Sprintf("toolhive/%s/registry-%x.json", persistentCacheSubdir, hash[:4])
//  └─ 4 raw bytes → 8 lowercase hex characters
```

The registry auth secret key MUST use the same slice length. The full key
is `REGISTRY_OAUTH_<8 hex chars>` (24 characters total). This provides
2^32 ≈ 4 billion distinct values — sufficient for a user with a handful of
registries — and stays within all keyring backend key-length limits.

**Canonical `deriveSecretKey` (replaces the `h[:6]` version in Gap 2,
place in `pkg/registry/auth/auth.go`):**

```go
import (
    "crypto/sha256"
    "fmt"
    "net/url"
    "strings"
)

// deriveSecretKey returns a deterministic secret manager key for a
// registry's refresh token.
//
// Format: REGISTRY_OAUTH_<8 lowercase hex chars>
// Example: REGISTRY_OAUTH_a3f1c802
//
// Normalization applied before hashing:
//   1. Lowercase scheme and host (matches normalizeResourceURI in
//      pkg/auth/remote/config.go)
//   2. Strip trailing slash from path
//   3. Strip URL fragment (no identity meaning)
//   4. Null-byte separator between registry URL and issuer prevents
//      hash collisions from ambiguous concatenation:
//      "ab\x00cd" cannot collide with "abc\x00d"
func deriveSecretKey(registryURL, issuerURL string) (string, error) {
    normalizedRegistry, err := normalizeURLForKey(registryURL)
    if err != nil {
        return "", fmt.Errorf("invalid registry URL %q: %w", registryURL, err)
    }
    normalizedIssuer, err := normalizeURLForKey(issuerURL)
    if err != nil {
        return "", fmt.Errorf("invalid issuer URL %q: %w", issuerURL, err)
    }
    input := normalizedRegistry + "\x00" + normalizedIssuer
    h := sha256.Sum256([]byte(input))
    return fmt.Sprintf("REGISTRY_OAUTH_%x", h[:4]), nil // 4 bytes = 8 hex chars
}

// normalizeURLForKey applies the same normalizations as normalizeResourceURI
// in pkg/auth/remote/config.go, plus trailing-slash stripping on the path.
func normalizeURLForKey(raw string) (string, error) {
    if raw == "" {
        return "", fmt.Errorf("URL must not be empty")
    }
    parsed, err := url.Parse(raw)
    if err != nil {
        return "", err
    }
    parsed.Scheme = strings.ToLower(parsed.Scheme)
    parsed.Host = strings.ToLower(parsed.Host)
    parsed.Path = strings.TrimRight(parsed.Path, "/")
    parsed.Fragment = ""
    return parsed.String(), nil
}
```

**Required unit test in `pkg/registry/auth/auth_test.go`:**

```go
func TestDeriveSecretKey(t *testing.T) {
    canonical, err := deriveSecretKey(
        "https://registry.company.com/api",
        "https://auth.company.com",
    )
    require.NoError(t, err)
    require.Regexp(t, `^REGISTRY_OAUTH_[0-9a-f]{8}$`, canonical)

    // All of the following must produce the same key as canonical.
    sameInputs := []struct{ reg, iss string }{
        {"HTTPS://Registry.Company.COM/api", "https://auth.company.com"},  // uppercase normalised
        {"https://registry.company.com/api/", "https://auth.company.com"}, // trailing slash stripped
        {"https://registry.company.com/api", "https://auth.company.com#x"}, // fragment stripped
    }
    for _, tc := range sameInputs {
        got, err := deriveSecretKey(tc.reg, tc.iss)
        require.NoError(t, err)
        require.Equal(t, canonical, got, "expected same key for %v", tc)
    }

    // Different issuer or registry must produce a different key.
    differentInputs := []struct{ reg, iss string }{
        {"https://registry.company.com/api", "https://other-idp.com"},
        {"https://other-registry.company.com/api", "https://auth.company.com"},
    }
    for _, tc := range differentInputs {
        got, err := deriveSecretKey(tc.reg, tc.iss)
        require.NoError(t, err)
        require.NotEqual(t, canonical, got, "expected different key for %v", tc)
    }
}
```

### S2. Secrets Provider Degradation — Exact Error Check

The `resolveSecretsProvider` function in Gap 4 correctly uses
`errors.Is(err, secrets.ErrSecretsNotSetup)` for the graceful degradation
path. **Do not substitute string matching or type-assertion on the wrapped
error.** Use `errors.Is` only.

The sentinel is defined in `pkg/secrets/factory.go`:

```go
// pkg/secrets/factory.go (existing, do not modify)
var ErrSecretsNotSetup = httperr.WithCode(
    errors.New("secrets provider not configured. "+
        "Please run 'thv secret setup' to configure a secrets provider first"),
    http.StatusNotFound,
)
```

`httperr.WithCode` wraps the inner error; `errors.Is` unwraps it correctly.

**Correct check pattern:**

```go
providerType, err := cfg.Secrets.GetProviderType()
if err != nil {
    if errors.Is(err, secrets.ErrSecretsNotSetup) {
        // Expected: user has not run 'thv secret setup'. Degrade silently.
        // Log at DEBUG, not WARN — this is not a user error.
        slog.Debug("registry auth: secrets manager not configured; " +
            "refresh tokens will not persist across CLI invocations")
        return nil // callers must nil-guard every use of this return value
    }
    // Unexpected misconfiguration. Degrade with a warning, do not fail the
    // entire command — registry auth is an enhancement, not a hard dependency.
    slog.Warn("registry auth: secrets provider unavailable; " +
        "refresh tokens will not persist", "error", err)
    return nil
}
```

**Forbidden alternatives:**

```go
// WRONG: string matching bypasses error wrapping
if strings.Contains(err.Error(), "not configured") { ... }

// WRONG: type assertion on httperr internals is fragile
if _, ok := err.(*httperr.CodedError); ok { ... }

// WRONG: treating ErrSecretsNotSetup as fatal blocks all registry commands
// for users who have not yet run 'thv secret setup'
if err != nil { return nil, err }
```

**Nil-guard contract:** Every call site that uses the `secrets.Provider`
from `resolveSecretsProvider` MUST check for nil before calling any method.
The three call sites in `oauthTokenSource` are:

```go
// Site 1: tryRestoreFromCache — guard at entry
if ts.oauthCfg.CachedRefreshTokenRef == "" || ts.secretsProvider == nil {
    return nil, fmt.Errorf("no secrets provider: cannot restore cached tokens")
}

// Site 2: performBrowserFlow — guard around persist call
if result.RefreshToken != "" && ts.secretsProvider != nil {
    _ = ts.secretsProvider.SetSecret(ctx, ts.secretKey, result.RefreshToken)
}

// Site 3: makePersister — only constructed when ts.secretsProvider != nil,
// so no additional nil check is needed inside the closure body.
```

### S3. Interactive Context Enforcement

The implementation uses a single `oauthTokenSource` struct with an
`interactive bool` field, consistent with the existing codebase pattern
(e.g., `SkipBrowser bool` in `pkg/auth/remote/config.go` and
`pkg/auth/discovery/discovery.go`). The boolean is set once at
construction time and governs the `if !ts.interactive` guard in `Token()`.

**Sentinel error for non-interactive context:**

```go
// pkg/registry/auth/oauth_token_source.go

// ErrRegistryAuthRequired is returned by Token() when no cached token is
// available and the context is non-interactive (thv serve). Callers must
// use errors.Is to detect it. The API handler maps this to HTTP 503.
var ErrRegistryAuthRequired = errors.New(
    "registry authentication required: run 'thv registry login' to authenticate",
)
```

When `interactive == false` and no cached token can be restored, `Token()`
returns `ErrRegistryAuthRequired` instead of attempting the browser flow.

**HTTP 503 for `ErrRegistryAuthRequired` — not 401:**

- HTTP 401 signals that the incoming client (Studio) failed to authenticate
  to the `thv serve` API. Studio is not expected to have registry
  credentials — `thv serve` is the authenticated proxy.
- HTTP 503 correctly signals that a server-side dependency (the cached
  credential store) is temporarily unavailable and that a human must
  run `thv registry login` to remediate.

```go
// In the thv serve API handler:
if errors.Is(err, registryauth.ErrRegistryAuthRequired) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusServiceUnavailable) // 503, not 401
    _ = json.NewEncoder(w).Encode(map[string]string{
        "code":    "registry_auth_required",
        "message": "Registry authentication required. Run 'thv registry login' to authenticate.",
    })
    return
}
```

**Call-site mapping:**

| Context | `interactive` | Behavior |
|---------|--------------|----------|
| `thv registry list`, `thv run` | `true` | Browser flow if no cached token |
| `thv registry login` | `true` | Browser flow (eager) |
| `thv serve` | `false` | Cached tokens only; returns `ErrRegistryAuthRequired` if unavailable |
| Tests | must be explicit | must match path under test |

### S4. Token Logging Safety

Token values (access tokens, refresh tokens, client secrets) must never
appear in log output at any level. This is not compiler-enforceable; every
code review of this feature must check for violations explicitly.

**Permitted log content by level:**

| Level | Permitted | Forbidden |
|-------|-----------|-----------|
| INFO | "Registry OAuth flow completed", "Registry credentials cleared" | Any token value |
| DEBUG | Secret key name, issuer URL, token expiry timestamp, boolean token presence, "cached token retrieved", "token persisted" | Token values; `%+v` of `oauth2.Token` |
| WARN | Persist failure (key name + error only); secrets degradation | Token values |
| ERROR | Fatal auth failures | Token values |

**Safe patterns:**

```go
// Key name, not value
slog.Debug("Retrieving cached registry refresh token",
    slog.String("key", cfg.CachedRefreshTokenRef),
)

// Boolean presence, not value
slog.Debug("Cached registry refresh token status",
    slog.String("key", cfg.CachedRefreshTokenRef),
    slog.Bool("present", refreshToken != ""),
)

// Expiry, not token
slog.Debug("Registry access token obtained",
    slog.Time("expires_at", token.Expiry),
)

// Persist success: key only
slog.Debug("Registry refresh token persisted",
    slog.String("key", ts.secretKey),
)

// Persist failure: key + error, no value
slog.Warn("Failed to persist registry refresh token",
    slog.String("key", ts.secretKey),
    slog.String("error", err.Error()),
)
```

**Forbidden patterns (code review must reject these):**

```go
// Token value in log
slog.Debug("Got token", "access_token", token.AccessToken)

// Refresh token value in log
slog.Debug("Refresh token", "value", refreshToken)

// fmt.Sprintf embedding a token
slog.Debug(fmt.Sprintf("token: %s", token.AccessToken))

// %+v on oauth2.Token — exposes AccessToken and RefreshToken fields
// because oauth2.Token has no fmt.Stringer implementation
slog.Debug("Token", "t", fmt.Sprintf("%+v", token))

// slog.Any on oauth2.Token — reflect prints all exported fields
slog.Debug("Token", slog.Any("t", token))

// Token value embedded in an error message
return fmt.Errorf("invalid token %s: %w", token.AccessToken, err)
```

**Struct protection:** Any struct in `pkg/registry/auth/` that stores token
metadata (not the value itself) MUST implement `fmt.Stringer` with only
safe fields, so that `%v` / `%+v` in log calls elsewhere cannot accidentally
leak a token:

```go
// tokenRef stores metadata about a persisted token reference.
// The token value is never stored in this struct — it lives only as a
// local variable in the function that retrieved it from the secrets manager.
type tokenRef struct {
    key    string    // secret manager key name (safe to log)
    expiry time.Time // token expiry (safe to log)
}

func (t tokenRef) String() string {
    return fmt.Sprintf("tokenRef{key=%q, expires=%s}",
        t.key, t.expiry.Format(time.RFC3339))
}
```

**Persister callback:** The callback passed to `remote.NewPersistingTokenSource`
receives `refreshToken string` as its first argument. The existing
`PersistingTokenSource.Token()` method (line 66,
`pkg/auth/remote/persisting_token_source.go`) logs safely without values:

```go
slog.Debug("Successfully persisted refreshed OAuth token")
```

The registry auth persister MUST follow this pattern and MUST NOT log the
`refreshToken` parameter under any circumstances:

```go
// CORRECT persister pattern:
persister := func(refreshToken string, expiry time.Time) error {
    // DO NOT log refreshToken.
    if err := secretProvider.SetSecret(ctx, secretKey, refreshToken); err != nil {
        slog.Warn("Failed to persist registry refresh token",
            slog.String("key", secretKey),
            slog.String("error", err.Error()),
            // refreshToken intentionally omitted
        )
        return err
    }
    // PersistingTokenSource already logs success at DEBUG; no duplicate log here.
    return nil
}
```

---

## Implementation Plan

### Phase 1: OAuth/OIDC with PKCE (Current)

- Add `RegistryAuth` and `RegistryOAuthConfig` structs to `pkg/config/config.go`
- Create `pkg/registry/auth/` package:
  - `auth.go` — `NewTokenSource` factory returning `oauth2.TokenSource`
  - `oauth_token_source.go` — OAuth token lifecycle (cache → restore → browser flow), implements `oauth2.TokenSource`
  - Uses standard `oauth2.Transport` as the auth-injecting `http.RoundTripper`
- Create `pkg/registry/auth_configurator.go` — business logic for CLI commands
- Add `set-registry-auth` and `unset-registry-auth` commands to `cmd/thv/app/config.go`
- Add `thv registry login` and `thv registry logout` commands
- Update `pkg/registry/factory.go` with `resolveTokenSource()` integration
- Thread `tokenSource` through `NewCachedAPIRegistryProvider` → `NewAPIRegistryProvider` → `NewClient`
- Skip API validation probe when `tokenSource` is non-nil
- Detect 401/403 during validation and provide actionable error messages
- Update `get-registry` output to show auth status
- Extend `GET /api/v1beta/registry` response with `auth_status` and `auth_type` fields
- Return structured `registry_auth_required` JSON error from `thv serve` when auth is needed
- Derive secret keys from registry URL hash to prevent token clobbering

### Phase 2: Bearer Token Authentication (Future)

- Import `pkg/auth/remote.BearerTokenSource` (existing) as the `oauth2.TokenSource` for bearer auth
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
  instead of the fixed port 8666, eliminating port collision risk. However,
  this requires the IdP to permit dynamic redirect URIs — Okta and Azure AD
  (in standard configuration) require exact redirect URI pre-registration
  per RFC 6749 §3.1.2, making ephemeral ports incompatible with those
  providers. Auth0 supports wildcard patterns (`http://localhost:*`). The
  fixed port 8666 (configurable via `callback_port`) is the correct default
  for enterprise deployments targeting Okta/Azure AD.
- **Client ID Metadata Documents:** MCP 2025-11-25 recommends this as
  the preferred client registration mechanism. Would eliminate the need
  for manual `--client-id` configuration.
- **Registry-served auth discovery document:** Similar to Terraform's
  `/.well-known/terraform.json`, `toolhive-registry-server` could serve a
  `GET /.well-known/toolhive-registry` endpoint (unauthenticated —
  bootstrap requirement) containing the issuer, client ID, scopes, and
  audience. This would enable zero-config setup: `thv config set-registry
  https://registry.company.com` auto-discovers auth requirements without
  requiring `--issuer` or `--client-id`. This is complementary to RFC 9728
  (which provides standard 401-triggered discovery but does not include the
  client ID) and higher priority because it requires only a registry-server
  endpoint addition and a small CLI enhancement.
- **Unified `thv login` / `thv logout` commands:** As ToolHive potentially
  gains multiple authentication contexts (registry, cloud account), a
  top-level `thv login` may provide a more platform-like experience
  consistent with tools like `gh auth login` and `az login`. Deferred
  until there is more than one authentication domain to unify.
- **Studio-initiated auth via `thv serve` API:** An async
  `POST /api/v1beta/registry/{name}/auth/initiate` endpoint that starts
  the browser OAuth flow in a background goroutine, returning `202 Accepted`
  with a `flow_id` and `poll_url`. Studio polls
  `GET /api/v1beta/registry/{name}/auth/status` for completion
  (`pending`/`completed`/`failed`/`expired`). If a flow is already in
  progress, `/initiate` returns `409 Conflict`. A synchronous browser flow
  from an API handler is rejected (blocks request, phishing/DoS vector).
  Security controls required before this endpoint can be added:
  (1) **One-at-a-time mutex** — refuse if a flow is already in progress;
  (2) **Restricted CORS** — `Access-Control-Allow-Origin` restricted to
  Desktop App origin, not `*`;
  (3) **Opt-in flag** — `thv serve --enable-browser-auth` (off by default)
  so only intentionally-configured deployments expose the endpoint;
  (4) **No credentials in response** — status endpoint never returns tokens.
  Long-term, a pre-shared launch token between Studio and `thv serve`
  provides stronger per-caller authentication.
- **Config validation at load time:** Validate issuer URL scheme and
  optionally re-verify OIDC discovery when config is loaded from disk
  (not just when set via CLI command). This closes the gap where MDM or
  manual file editing bypasses `set-registry-auth` validation.

### Dependencies

- No new external dependencies. Reuses existing `golang.org/x/oauth2`,
  `pkg/auth/oauth/`, `pkg/auth/remote/`, and `pkg/secrets/` packages.

## Documentation

- Update CLI help text for `thv config` subcommands and `thv registry login/logout`
- Add registry authentication guide to user documentation
- Document MDM/enterprise deployment patterns (pre-loaded config)
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
| 2026-03-03 | Expert review panel | Revised | Address PR review comments: switch to `oauth2.TokenSource` (drop custom interface), clarify `pkg/registry/auth/` separation rationale, add `thv registry login/logout` commands, add auth status API for `thv serve`/Studio, add structured `registry_auth_required` errors, add Studio async auth flow to future considerations, document MDM/enterprise pre-loaded config support |
| 2026-03-03 | Expert review panel | Revised | Add Implementation Guide with concrete code for agent implementation: constructor signatures, Token() orchestration pseudocode, interactive flag threading, secrets provider degradation, oauth2.Transport wiring, CLI command structure, API integration guidance (auth status fields, structured 503 errors, 401 detection, getCurrentProvider), security constraints (secret key derivation h[:4], sentinel error, token logging safety). Reconciled conflicts: h[:4] matches provider_cached.go precedent, single struct with interactive bool matches existing codebase patterns, HTTP 503 for server-side auth dependency |
| 2026-03-04 | Expert review panel | Revised | Address @jhrozek feedback: add issuer-binding validation assertion (OIDC Discovery §4.3), add OIDC discovery hijack to threat model, add per-IdP audience configuration table, add nonce/ID token design decision (nonce not needed since ID tokens not consumed), fix logout to clear tokens only (not config), add `thv registry login` naming rationale, strengthen ephemeral ports note (Okta exact-match incompatibility), add registry-served discovery document future consideration, add `thv login` alias future consideration, expand Studio POST /initiate with concrete API contract and security controls |

### Implementation Tracking

| Repository | PR | Status |
|------------|-----|--------|
| toolhive | [#3908](https://github.com/stacklok/toolhive/pull/3908) | In Progress |
