# RFC-XXXX: User-to-Agent Delegation via OAuth Token Exchange

- **Status**: Draft
- **Author(s)**: Jakub Hrozek (@jhrozek)
- **Created**: 2026-04-24
- **Last Updated**: 2026-04-24
- **Target Repository**: toolhive
- **Related Issues**: TBD

> **Note on numbering**: this file uses the placeholder `THV-XXXX` until the
> PR is opened, at which point the number will be set to the PR number per
> the [naming convention](../CONTRIBUTING.md#rfc-file-naming-format).

## Summary

Extend ToolHive's embedded authorization server with the [RFC 8693 OAuth 2.0
Token Exchange](https://datatracker.ietf.org/doc/html/rfc8693) grant so an
authenticated *agent* can present a user's JWT and receive a delegated JWT
that names the user as the principal (`sub`) and the agent as the acting
party (`act`). The exchange supports both same-trust-domain tokens (subject
tokens issued by ToolHive's own AS) and federated tokens issued by an
external OIDC provider, configured via a new trust-only upstream type
(`oidc-trust`). Cedar authorization is extended to read the `act` claim so
policies can scope what an agent may do *on behalf of* a user.

## Problem Statement

### What is "user-to-agent delegation"?

In an *agentic* deployment of MCP, a non-human caller — typically an LLM agent
— calls an MCP server and ultimately a backend resource. Two distinct
identities are present on every such call:

- the **user** who originally requested the work (e.g. "Alice")
- the **agent** that is performing the work on the user's behalf
  (e.g. "coding-agent")

Today, the embedded auth server (RFC-0019, RFC-0031) issues a JWT that
identifies *the user* and nothing else. The agent is invisible. This forces
a binary choice at the MCP server / backend:

1. **Pretend the agent is the user.** The agent sends the user's JWT
   verbatim. Authorization, audit, and rate-limiting see only the user. An
   audit log entry that says "Alice deleted the repo" cannot tell whether
   Alice typed the command herself or whether her agent did it on her behalf
   from a tool call that may itself have come from a malicious prompt
   injection.

2. **Treat the agent as the only identity.** The agent uses its own
   `client_credentials` token. The user is invisible. Per-user policy and
   audit are lost; the agent looks like a single super-user account.

Neither option is acceptable for production. We need a token format that
carries **both** identities at once, with a clear semantic distinction
between *principal* (whose data) and *actor* (who is acting).

### Why the existing token exchange middleware (RFC-0007) does not solve it

RFC-0007 (`pkg/oauth/tokenexchange`) is **outbound** middleware: it takes the
incoming token to the proxy and exchanges it at an *external* token endpoint
for an upstream-bound token before forwarding the request to a backend. It
does not change the identity of the principal — the result still names the
same user — and it never adds an actor. It cannot create a token that says
"this is Alice, acted upon by coding-agent" because the AS at the other end
of the exchange has no notion of the agent.

This RFC works on the **inbound** side: the *AS itself* learns to issue
delegated tokens. The outbound middleware then continues to work unchanged on
top.

### Who is affected

- **Platform operators** running ToolHive with an embedded AS in front of
  agentic workloads, who need per-agent audit and per-agent policy on top of
  per-user policy.
- **MCP server authors** who want to express authorization rules of the form
  "Alice may write to the repository, but coding-agent acting on Alice's
  behalf may only read".
- **Compliance / security teams** who require an auditable record of which
  agent acted on which user's behalf, distinct from the user's own actions.

### Why now

ToolHive is increasingly used to front agentic systems, and the existing
single-identity JWT model is the largest gap in our authorization story.
RFC 8693 has been an IETF standard since 2020 and is supported by every
major OAuth library, so the solution is well-trodden ground; the only novel
part is the integration with ToolHive's existing AS, upstream-provider
model, and Cedar authorization.

## Background

This section is for readers unfamiliar with RFC 8693 or with ToolHive's
auth-server architecture. Readers comfortable with both can skip ahead to
[Goals](#goals).

### RFC 8693 in 30 seconds

A token exchange request is an HTTP POST to the token endpoint with
`grant_type=urn:ietf:params:oauth:grant-type:token-exchange`. Its core
parameters are:

| Parameter | Meaning |
|-----------|---------|
| `subject_token` | The token representing **the principal** the new token is *for* |
| `subject_token_type` | The format of `subject_token` (typically `access_token` or `id_token`) |
| `actor_token` *(optional)* | The token representing **the acting party** |
| `actor_token_type` *(optional)* | The format of `actor_token` |
| `audience`, `scope`, `resource` | Standard scoping parameters |

The token endpoint validates both tokens, applies its own policy, and issues
a new token. RFC 8693 §4.1 specifies that delegated tokens carry an **`act`
claim**: a nested object whose `sub` field names the actor.

```json
{
  "iss":  "https://auth.example.com",
  "sub":  "alice@example.com",
  "act":  { "sub": "coding-agent" },
  "exp":  1714000000
}
```

The principal is `alice`. The actor is `coding-agent`. Backends authorise
against `sub`; audit records both.

### What the AS already does (after RFC-0019, RFC-0031, RFC-0052)

- Speaks OAuth 2.0 / OIDC against MCP clients on `/oauth/{authorize,token,
  callback,register}` and `/.well-known/{openid-configuration,
  oauth-authorization-server,jwks.json}`.
- Authenticates the user against one or more configured **upstream IDPs**
  (`oidc` for full OIDC discovery, `oauth2` for explicit-endpoint OAuth 2.0)
  and stores the resulting upstream tokens.
- Issues short-lived signed JWTs whose `tsid` claim links to the stored
  upstream tokens for later retrieval by vMCP outbound middleware.
- Runs **embedded** inside the proxy runner, configured by an
  `MCPExternalAuthConfig` of `type: embeddedAuthServer`.

Today the AS supports the `authorization_code`, `refresh_token`, and
`client_credentials` grants. It does **not** support token exchange.

## Goals

- Add the RFC 8693 token-exchange grant to the embedded AS so an
  authenticated client can exchange a user JWT for a delegated JWT that
  carries both identities.
- Support **two trust models** for the subject token:
  1. *Same-domain*: the subject token is an AS-issued JWT, validated
     against the AS's own JWKS.
  2. *Federated*: the subject token is issued by an external OIDC provider,
     trusted via a new `oidc-trust` upstream type and validated against
     that provider's JWKS.
- Bind the actor identity to the authenticated OAuth client to prevent
  replay of leaked actor tokens by a different client.
- Extend Cedar authorization so policies can read `act.sub` and write rules
  that depend on which agent is acting.
- Plumb operational details for in-cluster IDPs: a CA-bundle reference for
  private TLS and a flag to permit private-IP issuer endpoints.
- Stay strictly additive: deployments that do not configure delegation or
  trust upstreams are unaffected, and the existing `authorization_code`,
  `refresh_token`, and `client_credentials` flows continue to work
  unchanged.

## Non-Goals

- **Multiple actor chaining (`act.act.…`).** RFC 8693 §4.1 allows nested
  `act` claims to record an agent acting on behalf of another agent acting
  on behalf of a user. This RFC issues a single-level `act` claim only.
- **Issuing OIDC ID tokens or refresh tokens from the exchange.** Only an
  access token is returned. The exchange is one-shot; the agent re-exchanges
  on its own schedule.
- **Token Exchange between two different ToolHive AS instances** (trust
  federation across deployments). All federated trust is one-way: the AS
  trusts an external OIDC provider, not vice versa.
- **Redirect-flow login against `oidc-trust` upstreams.** The new upstream
  type contributes trust material only; it does not participate in browser
  redirect flows.
- **mTLS / SPIFFE-based client authentication.** A separate effort explored
  using the agent's mTLS identity as the actor; this RFC uses the
  authenticated OAuth client identity (HTTP Basic, `client_secret_post`, or
  similar). Adding mTLS-derived actors is left to a follow-up.
- **Step-up authentication** when the subject token's authentication context
  is insufficient for the requested scope.
- **CLI-mode (`thv proxy`) integration.** Like RFC-0031, this is an
  embedded-AS feature; the CLI does not expose an embedded AS to extend.

## Proposed Solution

### High-Level Design

The AS gains a new fosite handler for the token-exchange grant. The handler
sits alongside the existing handlers in the same fosite provider; nothing
about the existing endpoints or grant types changes.

```mermaid
flowchart TB
    subgraph AuthServer["Embedded Auth Server"]
        Token["/oauth/token"]
        subgraph Handlers["Fosite token-endpoint handlers"]
            AC["Authorization Code"]
            RT["Refresh Token"]
            CC["Client Credentials"]
            TE["Token Exchange (NEW)"]
        end
        subgraph Validators["Subject-token validators"]
            SV["Self-issued (own JWKS)"]
            MV["Multi-issuer<br>(self + trusted external)"]
        end
        subgraph Upstreams["Upstream provider types"]
            OIDC["oidc"]
            OAUTH2["oauth2"]
            TRUST["oidc-trust (NEW)"]
        end
    end

    Token --> AC
    Token --> RT
    Token --> CC
    Token --> TE
    TE --> MV
    MV --> SV
    MV --> TRUST
    TRUST -->|JWKS via OIDC discovery| External[External OIDC IdP]
```

#### Two flows: same-domain and federated

The exchange has two flows that differ only in which JWKS validates the
subject token. Everything downstream of validation is identical.

**Same-domain flow** — the user has an AS-issued JWT and the agent
exchanges it for a delegated AS-issued JWT.

```mermaid
sequenceDiagram
    participant User
    participant Agent
    participant AS as ToolHive AS
    participant Backend as MCP Server

    User->>AS: 1. Authorization code flow
    AS-->>User: 2. JWT-A (sub=alice)
    User->>Agent: 3. Pass JWT-A to agent
    Note over Agent: Agent authenticates as<br>OAuth client coding-agent

    Agent->>AS: 4. POST /oauth/token<br>grant_type=token-exchange<br>subject_token=JWT-A<br>(client_secret_basic auth)
    Note over AS: Validate JWT-A against own JWKS<br>Verify client = coding-agent<br>Build delegated session
    AS-->>Agent: 5. JWT-B<br>sub=alice, act={sub:coding-agent}

    Agent->>Backend: 6. Bearer JWT-B
    Note over Backend: Auth middleware validates JWT-B<br>Cedar: principal=alice,<br>actor=coding-agent
    Backend-->>Agent: 7. Response
```

**Federated flow** — the user authenticated against an external OIDC IDP
(say Keycloak) and holds a Keycloak-issued JWT. The agent presents that
token directly, without first exchanging it through the AS authorization
code flow.

```mermaid
sequenceDiagram
    participant User
    participant Agent
    participant Ext as External OIDC IdP<br>(Keycloak)
    participant AS as ToolHive AS
    participant Backend as MCP Server

    User->>Ext: Authenticate
    Ext-->>User: JWT-X (iss=keycloak, sub=alice)
    User->>Agent: Pass JWT-X

    Agent->>AS: POST /oauth/token<br>grant_type=token-exchange<br>subject_token=JWT-X
    Note over AS: peek iss claim → keycloak<br>route to MultiIssuerValidator<br>fetch keycloak JWKS via discovery<br>verify JWT-X<br>verify client = coding-agent
    AS-->>Agent: JWT-B<br>iss=toolhive-as<br>sub=alice<br>act={sub:coding-agent}

    Agent->>Backend: Bearer JWT-B
    Backend-->>Agent: Response
```

The federated flow lets organisations keep their existing IDP as the user's
authentication source while still using ToolHive's per-agent delegation and
audit semantics.

### Vocabulary

This RFC and the implementation use the following terms consistently. They
match RFC 8693 wherever the spec defines them.

| Term | Meaning |
|------|---------|
| **Principal** | The identity carried in `sub` of the issued token. The "on behalf of" party. |
| **Actor** | The identity carried in `act.sub` of the issued token. The party performing the action. |
| **Subject token** | The token presented as `subject_token`. Names the principal. |
| **Actor token** | The token presented as `actor_token`. Names the actor. Optional in this RFC. |
| **Authenticated client** | The OAuth client that authenticated to the token endpoint (HTTP Basic etc.). Used as the actor when no `actor_token` is supplied. |
| **Self-issued token** | A JWT whose `iss` matches the AS's own issuer URL. Validated against the AS's own JWKS. |
| **Trusted external token** | A JWT whose `iss` matches one of the configured `oidc-trust` upstreams. Validated against that upstream's JWKS. |
| **Delegated token** | The access token returned from a token exchange. Carries `sub` and `act`. |

### Detailed Design

#### 1. Token-exchange handler

A new package `pkg/authserver/server/tokenexchange` implements
`fosite.TokenEndpointHandler` for the `urn:ietf:params:oauth:grant-type:
token-exchange` grant.

The handler runs after fosite's client authentication step, so by the time
it executes, `requester.GetClient()` is the authenticated OAuth client.

Algorithm:

1. Reject if the client is public. Token exchange requires a confidential
   client.
2. Parse and validate the RFC 8693 form parameters (`subject_token`,
   `subject_token_type`, optional `actor_token`/`actor_token_type`).
   - Accepted subject token types: `access_token`, `jwt`, `id_token`.
   - Accepted actor token types: `access_token`, `jwt`. (`id_token`
     intentionally excluded — see *Security Considerations*.)
3. Validate the subject token via the configured `SubjectTokenValidator`
   (see §2). Returns canonical claims (`sub`, `name`, `email`, `exp`,…) on
   success.
4. Resolve the actor identity (see §3). Result is a single string: either
   the actor token's `sub` (after a binding check) or the authenticated
   client's `client_id`.
5. Validate that every requested scope is allowed for the client and that
   the requested audience is permitted.
6. Build a delegated `session.Session` with:
   - `sub` = subject token's `sub`
   - `act` = `{ "sub": <actor> }`
   - `name`, `email` propagated from the subject token (UI surfaces)
   - **No `tsid` claim.** Delegated tokens are not bound to a stored
     upstream session; the agent is acting on the *user's* identity but
     does not gain access to upstream IDP tokens stored under the user's
     `tsid`.
7. Cap the lifetime to `min(subject_remaining, configured_delegation_max)`.
   The configured maximum defaults to 15 minutes and is bounded above by 24
   hours. A delegated token may not outlive the user's own grant.
8. Hand off to fosite's standard `IssueAccessToken` to sign and return.

Pseudocode for the central decision in step 6:

```go
delegatedSession := session.New(
    validatedClaims.Subject,            // sub = principal (user)
    "",                                 // no upstream session link
    actorSub,                           // session.client_id field for audit
    session.UserClaims{Name, Email},
)
delegatedSession.JWTClaims.Extra["act"] = map[string]any{"sub": actorSub}
```

#### 2. Subject-token validation: same-domain vs federated

The handler depends on a `SubjectTokenValidator` interface:

```go
type SubjectTokenValidator interface {
    Validate(ctx context.Context, rawToken string) (*ValidatedClaims, error)
}
```

Two implementations:

- **`SelfIssuedTokenValidator`** — validates against the AS's own JWKS.
  Used when the deployment has only same-domain delegation.
- **`MultiIssuerTokenValidator`** — wraps a `SelfIssuedTokenValidator` plus
  zero or more `TrustedIssuer` entries. On `Validate(ctx, raw)`:
  1. Parse the JWT *without* signature verification to peek at `iss`.
  2. If `iss` equals the AS's own issuer, delegate to the self-issued
     validator.
  3. If `iss` matches one of the configured trusted issuers, fetch (and
     cache, with a 5-minute TTL) the issuer's JWKS via OIDC discovery and
     verify there.
  4. Otherwise, reject — unknown issuers are never trusted.

The factory selects which validator to install based on whether the AS
config contains any `oidc-trust` upstreams. Same-domain-only deployments
get the simpler self-issued validator with no external HTTP traffic.

```go
type TrustedIssuer struct {
    IssuerURL        string
    ExpectedAudience string  // required `aud` claim, may be empty to skip
}
```

The `MultiIssuerTokenValidator` uses an `*http.Client` injected at
construction time. The same client is used for both OIDC discovery and
JWKS retrieval, so a single CA bundle / private-IP-permission policy
applies to everything (see §6).

The discovery document is parsed into the existing
`pkg/oauthproto.OIDCDiscoveryDocument` type. The JWKS URL is validated
against an SSRF guard that requires HTTPS and rejects private/loopback
addresses unless `AllowPrivateIP` is enabled.

#### 3. Actor identity resolution

The actor is resolved by the handler in this order:

1. **If `actor_token` is supplied:**
   - Validate it against the AS's *own* JWKS — actor tokens are *always*
     self-issued in this RFC. This prevents an external IDP from
     unilaterally asserting "this is the agent".
   - Enforce the binding `actor_token.sub == authenticated_client.client_id`.
     Without this, a leaked actor token could be presented by a different
     client to claim its identity. The check is unconditional, regardless
     of how the client authenticated (basic, post, future mTLS).
   - On success the actor is `actor_token.sub`.

2. **If `actor_token` is absent:**
   - The actor is the authenticated client's `client_id`.

The `actor_token` path exists because some deployments want a richer actor
representation than just a `client_id` string — e.g., a JWT carrying
additional claims (deployment, region, agent version) that downstream
authorization or audit might consume. Today only `actor.sub` is consumed
by Cedar, but the audit record can still include the full claim set.

The `actor_token` validator is exposed on the handler as a separate
`SubjectTokenValidator` (`selfValidator`) so it can be unit-tested in
isolation. The factory wires `selfValidator = validator` for same-domain
deployments and reuses the same self-issued validator inside the
multi-issuer wrapper for federated deployments.

#### 4. New upstream provider type: `oidc-trust`

The existing upstream provider model (RFC-0019, RFC-0052) supports two
types: `oidc` (OIDC with discovery + redirect-flow login) and `oauth2`
(explicit-endpoint OAuth 2.0 with redirect-flow login). Both are designed
around participating in the user's *login* — they hold a `client_id` /
`client_secret` and a `redirect_uri`, and the AS uses them to exchange an
authorization code for tokens.

A federated trust upstream is fundamentally different: the AS never logs
the user in through it, never holds a client secret for it, and never has
a redirect URI. The AS only needs to verify a signature and an audience.
Reusing `oidc` would force unnecessary fields and make the validation logic
ambiguous about whether redirect-flow is supported.

Therefore: a third type `oidc-trust`. It holds:

- `issuer` (required) — the OIDC issuer URL, used to construct the
  discovery URL (`<issuer>/.well-known/openid-configuration`) and to
  match the `iss` claim of incoming tokens.
- `clientId` (optional) — used as the `ExpectedAudience` for incoming
  token validation. Empty disables the audience check (some IDPs do not
  populate `aud` on access tokens).
- `caBundleConfigMapRef` (optional) — see §6.
- `allowPrivateIP` (optional, default `false`) — see §6.

`oidc-trust` upstreams implement the same `OAuth2Provider` interface as
the others for storage compatibility, but the redirect-flow methods
(`AuthorizationURL`, `ExchangeCodeForIdentity`, `RefreshTokens`) return an
explicit "not supported" error.

The AS's `defaultUpstreamFactory` extracts every `oidc-trust` upstream
from the configured list and feeds them into the
`MultiIssuerTokenValidator` as `TrustedIssuer` entries. The remaining
upstreams (`oidc`, `oauth2`) continue to drive the redirect-flow login
chain established by RFC-0052.

#### 5. Cedar authorization: the `act` claim

ToolHive's Cedar authorizer (`pkg/authz/authorizers/cedar`) builds a
context map from the token's claims so policies can write rules like
`context.claim_email == "alice@example.com"`. Today the converter
silently drops nested-map claims, so the `act` claim — which is itself a
JSON object — is invisible to policies.

This RFC adds a `map[string]interface{}` case to `convertToCedarValue` so
nested claims are converted to Cedar Records. After the change, policies
can be written like:

```cedar
// Allow Alice to call any tool, but if she's acting through coding-agent,
// only the read-only tools are allowed.

permit (
  principal,
  action == Action::"call_tool",
  resource
)
when {
  context.claim_email == "alice@example.com" &&
  !(context has claim_act)
};

permit (
  principal,
  action == Action::"call_tool",
  resource
)
when {
  context.claim_email == "alice@example.com" &&
  context has claim_act &&
  context.claim_act.sub == "coding-agent" &&
  resource.readOnlyHint == true
};
```

The change is purely additive in Cedar: existing policies that do not
reference `claim_act` are unaffected.

#### 6. Operational plumbing: CA bundle and `AllowPrivateIP`

Two practical concerns appeared during implementation that do not change
the conceptual model but require new fields to surface to the operator.

**CA bundle (`caBundleConfigMapRef`)**: an `oidc-trust` upstream is most
commonly an in-cluster Keycloak (or similar) using TLS certificates issued
by a private cluster CA. The system trust store does not contain that CA,
so OIDC discovery and JWKS fetching fail with a TLS error. The new field
points at a `ConfigMap` containing the CA in PEM form. The operator
mounts the ConfigMap into the proxy runner pod, the runner passes the
filesystem path through `RunConfig`, and the AS uses
`pkg/networking.NewHttpClientBuilder().WithCABundle(path)` to construct
the HTTP client used by the multi-issuer validator. When unset, the system
trust store is used.

**`allowPrivateIP`**: an in-cluster issuer URL typically resolves to a
private (RFC 1918) cluster IP. ToolHive's networking package defaults to
rejecting private IPs as a defence-in-depth measure against SSRF. The new
boolean field opts a single `oidc-trust` upstream into permitting private
IP resolution for its discovery and JWKS fetches. When `false` (the
default), the existing rejection policy applies. The flag is per-upstream
because two different `oidc-trust` upstreams in the same AS may want
different policies (one in-cluster, one external).

Both fields aggregate at AS startup: if any configured `oidc-trust`
upstream has a CA bundle or requests private IPs, the HTTP client is
built once with the union of those settings and shared by the validator
across all upstreams. A future refinement could give each upstream its
own client.

#### 7. Wiring: `RunConfig` → AS

The `EmbeddedAuthServerConfig` `RunConfig` gains:

```go
type RunConfig struct {
    // ... existing fields ...

    // DelegationTokenLifespan caps the lifetime of delegated tokens
    // issued via RFC 8693 token exchange. Empty defaults to 15 minutes;
    // bounded above by 24h.
    DelegationTokenLifespan string

    Upstreams []UpstreamRunConfig // existing, but Type may now be "oidc-trust"
}

type OIDCUpstreamRunConfig struct {
    // ... existing fields ...

    // CABundlePath is the absolute filesystem path to a CA-bundle PEM.
    // Used only by oidc-trust upstreams for TLS verification of the
    // OIDC discovery and JWKS endpoints. Empty means "use the system
    // trust store".
    CABundlePath string

    // AllowPrivateIP permits OIDC discovery and JWKS endpoints to
    // resolve to private IP addresses. Off by default.
    AllowPrivateIP bool
}
```

The `authserver.Config.UpstreamProviderType` enum gains
`UpstreamProviderTypeOIDCTrust` alongside the existing `oidc` and
`oauth2`. Validation requires `oidc_config` (with at least an issuer URL)
and forbids `oauth2_config` for the new type.

The runner derives the `extraFactories` slice for the fosite provider:

```go
extraFactories := []oauthserver.Factory{
    tokenexchange.Factory(tokenexchange.FactoryConfig{
        DelegationLifespan: cfg.DelegationTokenLifespan,
        TrustedIssuers:     trustedIssuersFromOIDCTrustUpstreams(upstreams),
        HTTPClient:         buildHTTPClientFromCAandPrivateIPFlags(upstreams),
    }),
}
```

When no `oidc-trust` upstreams are configured, `TrustedIssuers` is empty
and the validator collapses back to the self-issued path.

The token-exchange grant type is advertised in
`/.well-known/oauth-authorization-server` unconditionally. (RFC 8414
discovery has no notion of "this client may use this grant"; the
authorization decision is made per-client by fosite at the token endpoint.)

#### 8. Operator / CRD changes

`MCPExternalAuthConfig.spec.embeddedAuthServer.upstreamProviders[].type`
gains the enum value `oidc-trust`. The validation function is reorganised
so that:

- `oidc` and `oauth2` continue to follow the existing XOR rule
  (`oidcConfig` xor `oauth2Config`).
- `oidc-trust` requires `oidcConfig` (with a valid `issuerURL`), forbids
  `oauth2Config`, and additionally accepts the optional
  `caBundleConfigMapRef` and `allowPrivateIP` fields.

`OIDCUpstreamConfig` gains:

```go
type OIDCUpstreamConfig struct {
    // ... existing fields ...

    // CABundleConfigMapRef references a ConfigMap containing the CA
    // certificate for verifying the OIDC issuer's TLS certificate. Used
    // for oidc-trust providers where the issuer uses a non-public CA
    // (e.g., internal PKI). When nil, the system trust store is used.
    // +optional
    CABundleConfigMapRef *CABundleSource `json:"caBundleConfigMapRef,omitempty"`

    // AllowPrivateIP allows OIDC discovery and JWKS endpoints to resolve
    // to private IP addresses. Required for in-cluster OIDC issuers
    // where the service DNS name resolves to a cluster-internal IP.
    // Default: false.
    // +optional
    AllowPrivateIP bool `json:"allowPrivateIP,omitempty"`
}
```

The `EmbeddedAuthServerConfig` gains
`delegationTokenLifespan *metav1.Duration`.

The operator's `controllerutil.GenerateAuthServerVolumes` is extended to
generate a CA-bundle volume mount per `oidc-trust` upstream that has a
`caBundleConfigMapRef`. The mount path follows the convention
`/etc/toolhive/authserver/upstream-ca/<upstream-name>/<key>` and is
threaded through `RunConfig` as `CABundlePath`.

### Configuration Changes

A complete example: an embedded AS with one `oidc` redirect-flow upstream
(corporate SSO for user login) and one `oidc-trust` upstream (Keycloak
for federated delegation), with the proxy runner mounting both the
Keycloak CA bundle and the agent's client secret.

```yaml
apiVersion: toolhive.stacklok.com/v1beta1
kind: MCPExternalAuthConfig
metadata:
  name: agentic-auth
  namespace: my-mcp
spec:
  type: embeddedAuthServer
  embeddedAuthServer:
    issuer: https://auth.toolhive.example.com
    delegationTokenLifespan: 15m   # cap delegated tokens
    signingKeys:
      - secretRef: { name: as-signing-keys, key: ec-key.pem }
    hmacSecretRefs:
      - { name: as-hmac, key: hmac-0 }
    upstreamProviders:
      # Existing-style upstream: corporate SSO for user login
      - name: corporate-sso
        type: oidc
        oidcConfig:
          issuerURL: https://sso.corp.example.com
          clientID: toolhive-as
          clientSecretRef: { name: corp-sso, key: client-secret }
          redirectURI: https://auth.toolhive.example.com/oauth/callback

      # NEW-style upstream: Keycloak whose tokens we trust as subject_tokens
      - name: keycloak
        type: oidc-trust
        oidcConfig:
          issuerURL: https://keycloak.svc.cluster.local:8443/realms/agents
          clientID: toolhive-resource          # used as ExpectedAudience
          caBundleConfigMapRef:
            configMapRef: { name: keycloak-ca, key: ca.crt }
          allowPrivateIP: true                  # in-cluster service IP
```

A delegation request from the agent to the AS:

```http
POST /oauth/token HTTP/1.1
Host: auth.toolhive.example.com
Authorization: Basic Y29kaW5nLWFnZW50OnMzY3JldA==
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Atoken-exchange
&subject_token=eyJhbG...
&subject_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Ajwt
&audience=https%3A%2F%2Fmcp.example.com
```

Successful response (`200 OK`):

```json
{
  "access_token":      "eyJhbGc...",
  "token_type":        "Bearer",
  "expires_in":        899,
  "issued_token_type": "urn:ietf:params:oauth:token-type:access_token"
}
```

Decoding `access_token`:

```json
{
  "iss": "https://auth.toolhive.example.com",
  "sub": "alice@corp.example.com",
  "act": { "sub": "coding-agent" },
  "aud": "https://mcp.example.com",
  "exp": 1714000899,
  "iat": 1714000000
}
```

## Security Considerations

### Threat Model

The token exchange grant is, by design, a privilege-relevant operation:
the AS is being asked to mint a token in *somebody else's* name. Threats:

- **Subject-token forgery / replay.** A caller crafts or steals a JWT and
  asks for a delegated token in that user's name.
- **Actor-token hijack.** A caller steals an actor token and presents it
  with their own client authentication to claim a different agent's
  identity.
- **External-IDP impersonation.** If the AS naively trusts whatever JWT it
  receives, a token from an *unconfigured* issuer could be accepted.
- **SSRF via discovery.** An attacker who controls an `oidc-trust`
  upstream's issuer URL could try to point the AS at internal infrastructure
  via the discovery / JWKS fetch.
- **Public-client misuse.** A public OAuth client (no secret) could attempt
  to exchange tokens.
- **Lifetime escalation.** A short-lived user token is exchanged for a
  long-lived delegated token, extending the effective grant.

### Authentication and Authorization

- **Confidential clients only.** The handler rejects any request from a
  client whose `Public()` is true. Public clients have no proof of identity
  and therefore cannot serve as actors.
- **Subject token signature.** Verified against the AS's own JWKS
  (same-domain) or against the trusted issuer's JWKS (federated). Standard
  claims (`iss`, `aud`, `exp`, `iat`, `nbf`, `sub`) are validated per RFC
  7519. Tokens signed with `none` are rejected by go-jose.
- **Issuer allowlist.** The multi-issuer validator rejects any token whose
  `iss` is neither the AS itself nor a configured `oidc-trust` upstream.
  There is no wildcard or regex matching; issuer URLs must match exactly.
- **Actor-token binding.** When `actor_token` is supplied, its `sub` must
  equal the authenticated client's `client_id`. This binds the actor
  identity to the authenticated party and prevents replay of leaked actor
  tokens by a different client. The check is unconditional and runs
  regardless of authentication method.
- **Standard fosite scope and audience checks** apply to the request: the
  client must be allowed each requested scope and each requested audience.

### Data Security

- **No new secrets at rest.** The handler does not persist actor tokens or
  subject tokens. Delegated tokens are signed in memory and returned over
  TLS; their contents (the user's `sub`, `name`, `email`) are no more
  sensitive than what the AS already issues today.
- **CA bundles** are referenced through Kubernetes `ConfigMap`s
  (non-secret) and mounted read-only into the proxy runner pod with
  `0400` mode.
- **JWKS caching** uses an in-memory map with a 5-minute TTL. No JWKS or
  discovery response is persisted.

### Input Validation

- **Form parameters.** Each RFC 8693 parameter is checked for presence and
  format. `subject_token_type` must be one of three known URIs;
  `actor_token_type` must be one of two (`id_token` is intentionally
  excluded — see below). Mismatched pairs (e.g., `actor_token_type`
  without `actor_token`) are rejected.
- **Why `id_token` is rejected as `actor_token_type`.** An ID token is an
  *identity assertion* about a user-driven login session, including a
  `nonce` and the user's authentication context. It is conceptually wrong
  to use it as a bearer credential proving "I am the agent". Allowing it
  would invite confused-deputy patterns where a user's ID token is replayed
  as an actor token.
- **JWT parsing.** Both validators use `github.com/go-jose/go-jose/v4`,
  which enforces strict JSON Web Algorithm allowlists (no `none`, no `HS*`
  on RSA/EC keys).
- **Issuer URL.** Discovery URLs are constructed from the configured
  issuer plus `/.well-known/openid-configuration`. The constructed URL is
  validated to be HTTPS; the `jwks_uri` returned by discovery is similarly
  validated and additionally checked against an SSRF allowlist (no
  loopback, no link-local, no private IP unless `allowPrivateIP=true` for
  that upstream).

### Secrets Management

- The agent's OAuth client secret is provided through the existing
  `clientSecretRef` mechanism on its registered client; this RFC does not
  introduce a new secret type.
- The AS's signing keys and HMAC secrets are unchanged from RFC-0031.
- No client secret is required for `oidc-trust` upstreams — they are
  trust-only and never participate in OAuth flows that would require one.

### Audit and Logging

The handler emits debug logs with the following fields on a successful
exchange:

| Field | Meaning |
|-------|---------|
| `subject` | The principal (user) `sub` from the subject token |
| `actor` | The actor `sub` (either authenticated `client_id` or `actor_token.sub`) |
| `lifetime` | The capped lifetime of the issued token |

On validation failure, the failure reason and the actor are logged at
debug. The token contents themselves are never logged. The fields chosen
allow an operator to reconstruct, from log output alone, "who acted as
whom and for how long" without exposing the raw JWTs.

A future RFC may add a structured audit-log emitter for delegation events
specifically; the current design relies on the existing `slog` plumbing
shared with the rest of the AS.

### Mitigations

- **Lifetime cap.** Delegated tokens are bounded by both the configured
  `DelegationTokenLifespan` (default 15m, max 24h) and the remaining
  lifetime of the subject token, whichever is shorter. This blunts the
  blast radius of a stolen delegated token.
- **No upstream-token access.** Delegated tokens have no `tsid` claim, so
  the outbound-swap middleware (RFC-0052) cannot use them to retrieve the
  user's stored upstream tokens. The agent's delegated token is good only
  for AS-issued downstream backends.
- **No refresh token.** The exchange returns only an access token. The
  agent must re-exchange when the token expires, giving the AS a chance to
  re-validate the subject token (which may by then have been revoked or
  expired).
- **Public-client reject** at the very first step.
- **Issuer allowlist** — see above.
- **TLS-by-default for discovery / JWKS** — `https://` required;
  exceptions explicit and per-upstream.

## Alternatives Considered

### Alternative 1: extend the existing outbound `tokenexchange` middleware

Add an "act-claim" mode to `pkg/oauth/tokenexchange` (the outbound
middleware introduced in RFC-0007) so it adds an actor when forwarding to
the upstream.

- **Pros**: no AS change; works without an embedded AS at all.
- **Cons**: the outbound middleware exchanges at an *external* token
  endpoint, which has no notion of the agent and would not produce a
  matching `act` claim. We'd be mis-using an outbound feature for an
  inbound concern. Also it would require *every* downstream IDP to
  implement RFC 8693 with `act` semantics, which most do not.
- **Why not chosen**: the trust boundary is wrong. The party that decides
  "this agent may act on this user's behalf" must be in our trust domain.
  The embedded AS is that party.

### Alternative 2: actor identity from mTLS / SPIFFE only, no `actor_token`

Always derive the actor from the agent's mTLS client certificate (via
SPIFFE ID extraction) and reject `actor_token` outright.

- **Pros**: cryptographically strong actor identity; no token replay
  concerns; aligns with WIMSE / agent-identity directions.
- **Cons**: requires every agent to have a mTLS-capable transport and a
  SPIFFE workload-attestation infrastructure. Excludes simpler deployments
  using `client_secret` auth. A prior iteration of this work bundled the
  mTLS path; we extracted it into a follow-up so the OAuth-only path can
  ship first.
- **Why not chosen now**: scope. The `actor_token` + binding-check pattern
  is RFC 8693's own answer and works without infrastructure changes. mTLS
  remains a future option; the `resolveActorIdentity` function is
  structured so an mTLS-context source can be added later as a third
  branch.

### Alternative 3: nest as a "type" inside the existing `oidc` upstream

Add a `mode: trust-only` field to `oidc` instead of introducing a third
type.

- **Pros**: smaller enum.
- **Cons**: the `oidc` type's existing fields (`clientSecretRef`,
  `redirectURI`, scopes) become contextually invalid; CRD validation
  becomes a nest of conditional XORs. Two distinct types are clearer.
- **Why not chosen**: explicit type wins over contextual modes.

### Alternative 4: validate trusted-issuer tokens only via OIDC userinfo

Instead of fetching JWKS, hit the issuer's `userinfo` endpoint with the
subject token to verify it.

- **Pros**: works for opaque (non-JWT) tokens.
- **Cons**: 1 extra round-trip per exchange; not all issuers expose
  userinfo; cannot extract `act` semantics. RFC 8693 is JWT-shaped.
- **Why not chosen**: scope is JWT subject tokens.

## Compatibility

### Backward Compatibility

This change is fully additive:

- The token-exchange grant is opt-in via `delegationTokenLifespan` /
  having clients permitted to use it. A deployment that does not configure
  any `oidc-trust` upstreams and has no agent clients keeps issuing
  exactly the same tokens as before.
- The `oidc-trust` upstream type is a new enum value. Existing
  `MCPExternalAuthConfig` resources never set it.
- `OIDCUpstreamConfig.caBundleConfigMapRef` and `allowPrivateIP` are new
  optional fields; existing resources omit them.
- The Cedar `act`-claim conversion is purely additive: existing policies
  do not reference `claim_act`, so their evaluation is unchanged.
- `EmbeddedAuthServerConfig.delegationTokenLifespan` is optional with a
  safe default.

### Forward Compatibility

- The `actor_token` validation lives behind the
  `SubjectTokenValidator` interface, so future actor sources (mTLS-derived
  SPIFFE IDs, asymmetric-key-bound tokens per RFC 7800, …) can be added
  by composing additional branches inside `resolveActorIdentity`.
- `TrustedIssuer` is a struct so future per-issuer policy fields (issuer-
  specific `aud` lists, allowed `sub` patterns, audit tags) can be added
  without breaking the constructor.
- Multi-actor chaining (`act.act…`) is structurally supported by the
  underlying claim shape; this RFC chooses not to populate it but does
  not preclude a follow-up.

## Implementation Plan

The work has already been prototyped and validated against a Keycloak +
in-cluster proxy runner test environment. The plan below splits the
change into reviewable PRs that can land independently. Each PR is
self-contained and the AS continues to build and pass existing tests at
every stage.

### Phase 1: core handler (foundation)

Goal: the AS understands the token-exchange grant for *same-domain*
subject tokens, with the actor being the authenticated client. No
federated trust, no `actor_token`, no Cedar changes.

- New package `pkg/authserver/server/tokenexchange` with
  `Handler`, `SelfIssuedTokenValidator`, `Factory`.
- Wire the factory into `createProvider` via `oauthserver.NewAuthorizationServer`'s
  `Factory` slice.
- Advertise the grant in `/.well-known/oauth-authorization-server`.
- Add `DelegationTokenLifespan` to `EmbeddedAuthServerConfig` and
  `RunConfig`, with default 15m and max 24h validation.

PR size: small to medium, almost entirely new code.

### Phase 2: `actor_token` and `id_token` subject-token type

- Add `actor_token` / `actor_token_type` parameter handling.
- Implement `resolveActorIdentity` with the binding check.
- Accept `id_token` as a valid `subject_token_type`.

### Phase 3: Cedar `act`-claim support

- Add `map[string]interface{}` case to `convertToCedarValue`.
- Tests verifying `claim_act.sub` is reachable from a policy.

This phase can land independently of phases 1-2 and parallel to phase 4.

### Phase 4: federated trust (`oidc-trust` upstream)

- Extract `SubjectTokenValidator` interface; implement
  `MultiIssuerTokenValidator` with OIDC discovery and JWKS caching.
- Add the `oidc-trust` upstream type to `pkg/authserver` config and to
  the `MCPExternalAuthConfig` CRD.
- Operator: extend `defaultUpstreamFactory`, validation, and the
  embedded-AS volume generation to handle the new type. Regenerate CRDs.
- Wire trusted issuers from the configured upstreams into the factory.

### Phase 5: operational plumbing

- Add `caBundleConfigMapRef` and `allowPrivateIP` to
  `OIDCUpstreamConfig` (CRD) and `OIDCUpstreamRunConfig`.
- Operator: generate per-upstream CA-bundle volume mounts.
- AS: aggregate CA-bundle paths and `allowPrivateIP` flags into a
  single `*http.Client` injected into the multi-issuer validator.

### Dependencies

- RFC-0019 (auth-server overview) — accepted, implemented.
- RFC-0031 (embedded AS in proxy runner) — accepted, implemented.
- RFC-0052 (multi-upstream IDP support) — referenced but **not** a hard
  dependency. The new `oidc-trust` type fits inside the multi-upstream
  model when it is in place; if RFC-0052 lands first, the integration is
  a no-op. If this RFC lands first, `oidc-trust` upstreams compose with
  the legacy single-upstream model by being separable from the redirect-
  flow login chain.

## Testing Strategy

- **Unit tests** for the token-exchange handler covering: valid same-domain
  exchange, valid federated exchange, missing parameters, invalid
  `subject_token_type`, expired subject token, public client rejection,
  scope mismatch, audience mismatch, lifetime capping by subject expiry,
  `actor_token` happy path, `actor_token` `sub` mismatch with client,
  `actor_token` without `actor_token_type` (and vice versa),
  `id_token` rejected as actor token type.
- **Unit tests** for `MultiIssuerTokenValidator`: routing on `iss`,
  rejection of unknown issuers, JWKS cache TTL behaviour, discovery
  failure surfacing as a token-exchange error.
- **Unit tests** for Cedar `claim_act.sub` reachability with table-driven
  policies.
- **Integration tests** wiring an in-process AS with one `oidc-trust`
  upstream pointing at a fake OIDC server (test fixture). Round-trip a
  forged-by-test token through the exchange and verify the `act` claim.
- **End-to-end test** in the operator e2e suite: Keycloak in a kind
  cluster, a `MCPExternalAuthConfig` with one `oidc` upstream and one
  `oidc-trust` upstream, an `MCPServer` with Cedar policies that
  distinguish direct vs delegated calls, and a script that exercises both
  paths.
- **Security tests** specifically covering: `actor_token` issued by an
  unrelated party rejected by binding check; `subject_token` with a forged
  `iss` rejected by issuer allowlist; private-IP issuer rejected when
  `allowPrivateIP=false`.

## Documentation

- **Architecture docs** (`docs/arch/`): a new section in the auth-server
  arch doc covering delegation, with the two sequence diagrams from this
  RFC.
- **Operator docs**: an example of an `MCPExternalAuthConfig` with the
  new upstream type, the new fields, and the resulting agent flow.
- **`stacklok/docs-website`**: a "delegating to agents" how-to that walks
  through the same Keycloak-based example end to end.
- **CRD reference** (auto-generated from kubebuilder markers): regenerated
  after the type changes.

## Open Questions

1. **Should `delegationTokenLifespan` default differently per
   environment?** 15 minutes is a reasonable default for production but
   may be inconvenient for development. Consider a separate
   `dev-mode` toggle that loosens the cap.
2. **Should the audit log get a dedicated structured event** (e.g.,
   `delegation_issued`) rather than relying on debug-level slog? A
   follow-up RFC on audit-event taxonomy would be a better home.
3. **Is `actor_token` issued by a *different* AS within the same trust
   federation in scope for a future iteration**, or is the binding to
   the authenticated client always the right call? Today we say "always";
   a federation use case may want to relax it.
4. **Does the multi-issuer validator need per-issuer client allowlists**
   — i.e., "tokens from issuer A may only be exchanged by clients from
   set X"? The RFC currently treats every authenticated client as eligible
   to exchange any trusted token; finer-grained policy could move into
   Cedar.

## References

- [RFC 8693 — OAuth 2.0 Token Exchange](https://datatracker.ietf.org/doc/html/rfc8693)
  (in particular §1.5 *Delegation vs. Impersonation*, §2 *Token Exchange
  Request and Response*, §4.1 *act (Actor) Claim*)
- [RFC 7519 — JSON Web Token](https://datatracker.ietf.org/doc/html/rfc7519)
- [RFC 8414 — OAuth 2.0 Authorization Server Metadata](https://datatracker.ietf.org/doc/html/rfc8414)
- [RFC 7591 — OAuth 2.0 Dynamic Client Registration](https://datatracker.ietf.org/doc/html/rfc7591)
- THV-0007 — Token exchange middleware (outbound counterpart)
- THV-0019 — OAuth Authorization Server (overview and design)
- THV-0031 — Embedded Authorization Server in Proxy Runner
- THV-0035 — Auth Server Redis Storage
- THV-0050 — Dedicated Auth Server Reference (`authServerRef`)
- THV-0052 — Multi-Upstream IDP Support in the Embedded Auth Server
- THV-0053 — vMCP Embedded Auth Server
- ToolHive auth server: `pkg/authserver/`
- Cedar authorizer: `pkg/authz/authorizers/cedar/`

---

## RFC Lifecycle

<!-- This section is maintained by RFC reviewers -->

### Review History

| Date | Reviewer | Decision | Notes |
|------|----------|----------|-------|
| 2026-04-24 | TBD | Draft | Initial submission |

### Implementation Tracking

| Repository | PR | Status |
|------------|-----|--------|
| toolhive | TBD | Pending — feature branch `token-delegation` |
