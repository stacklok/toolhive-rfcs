# RFC-0080: XAA Identity Chaining Intermediary

- **Status**: Draft
- **Author(s)**: Jakub Hrozek (@jhrozek)
- **Created**: 2026-07-01
- **Last Updated**: 2026-07-02
- **Target Repository**: toolhive
- **Related Issues**: [stacklok/toolhive#5218](https://github.com/stacklok/toolhive/issues/5218)

## Summary

[THV-0079](./THV-0079-cross-app-access-grant.md) makes the vMCP an XAA *client*: the user authenticates interactively at ToolHive's embedded AS (one browser login at ToolHive, separate from any login the MCP client already required), and ToolHive holds the captured upstream ID token to perform the two-step ID-JAG protocol on every backend call. This RFC addresses a different topology: the MCP client — an AI agent, an automation pipeline, or an application like Claude.ai — already holds the user's ID token from its own login and wants to present it directly to ToolHive's vMCP token endpoint. Under this model the user logs in once (to the MCP client); the client forwards the identity to ToolHive programmatically, with no second browser redirect at ToolHive.

This RFC is split into two independent phases:

- **Phase 1 — Inbound ID-JAG Acceptance.** ToolHive's embedded AS gains a new `jwt-bearer` grant handler. An external client presents an ID-JAG-THV addressed to ToolHive's AS issuer URI; ToolHive validates it and issues a vMCP session token. No browser redirect required. Works immediately with any backend using `header_injection`, `token_exchange`, or `upstream_inject` outgoing auth.

- **Phase 2 — Identity Chaining Intermediary.** For backends that themselves use the `xaa` outgoing strategy, ToolHive performs outbound XAA on the user's behalf: it derives a subject token from the Phase 1 session and uses it to obtain a backend-specific ID-JAG-SaaS via a fresh Exchange-SaaS. Phase 2 is additive on top of Phase 1 and can be shipped independently.

---

## 1. Background

This section describes what exists in ToolHive and in the OAuth ecosystem today. It contains no proposal content.

### 1.1 What THV-0079 Solves and What It Does Not

[THV-0079](./THV-0079-cross-app-access-grant.md) establishes the `xaa` outgoing-auth strategy for the vMCP. When a user authenticates via a browser OIDC flow, the embedded auth server ([THV-0053](./THV-0053-vmcp-embedded-authserver.md)) captures their upstream ID token. The `xaa` strategy reads that ID token and performs the two-step ID-JAG protocol on every backend call: Exchange-THV at the front-door IdP to mint an ID-JAG, Grant-THV at the resource AS to redeem it for a backend access token.

This path requires a browser login at ToolHive to seed the upstream ID token — a second authentication step that occurs after any login the MCP client already required of the user. It covers the interactive user case but leaves two gaps: (1) users must authenticate twice (once to the MCP client, once to ToolHive), and (2) automated clients — AI agents, CI pipelines, service-to-service calls — cannot drive a browser redirect at all. They already hold an identity credential from their own IdP, and they want to present that credential directly to ToolHive's vMCP token endpoint, with no redirect, and receive a session token they can use to call tools.

### 1.2 The Inbound Gap

The embedded auth server today issues tokens only to clients that have completed the OIDC authorization code flow. It has no grant handler for `urn:ietf:params:oauth:grant-type:jwt-bearer`. A non-interactive caller with a valid ID-JAG — an assertion that the user's IdP already minted and that ToolHive's AS could validate — has no entry point. The session token endpoint returns `unsupported_grant_type`.

Closing this gap is the first phase of this RFC: add an inbound ID-JAG acceptance path to the embedded AS (acting as Resource AS). This alone enables the browser-less use case for backends that use `header_injection`, `token_exchange`, `upstream_inject`, or any outgoing strategy that does not itself require an upstream ID token from a browser login.

### 1.3 The Chaining Scenario

For backends that are themselves protected via XAA — that is, backends whose `MCPExternalAuthConfig` is type `xaa` — the `xaa` outgoing strategy expects an upstream ID token in `Identity.UpstreamIDTokens`. After Phase 1, that map is empty for an inbound ID-JAG session: no browser login happened, so no upstream ID token was captured and stored. Phase 2 fills this gap by deriving a subject token that the `xaa` outgoing strategy (Exchange-SaaS) can use, without requiring the external client to supply any additional credential.

The full four-step chain and ToolHive's role in each:

| Step | Who performs it | Role |
|------|-----------------|------|
| **Exchange-THV** — external client exchanges its ID token at the IdP for ID-JAG-THV (aud=ToolHive AS) | External client (AI agent) | Client |
| **Grant-THV** — external client presents ID-JAG-THV to ToolHive's AS; ToolHive issues a vMCP session token | ToolHive | Resource Authorization Server |
| **Exchange-SaaS** — ToolHive exchanges a derived subject token at the IdP for ID-JAG-SaaS (aud=backend AS) | ToolHive | Client |
| **Grant-SaaS** — ToolHive presents ID-JAG-SaaS to the backend AS; backend AS issues an access token | ToolHive | Client |

ToolHive is absent from Exchange-THV — that step is the external client's responsibility, performed before any interaction with ToolHive.

`draft-ietf-oauth-identity-assertion-authz-grant-04` §9.3 explicitly permits a single piece of software to serve both roles: "This does not preclude a single piece of software from being both an IdP issuing ID-JAGs as well as a Resource Authorization Server consuming ID-JAGs." This RFC uses the term **Identity Chaining Intermediary** for ToolHive in this dual role. It is not a term from the draft — it is defined locally here to describe a deployment topology that the draft explicitly allows but does not name.

### 1.4 Vocabulary

Base vocabulary (Client, Resource AS, IdP AS, ID-JAG, upstream ID token, backend access token) is defined in [THV-0079 §1.5](./THV-0079-cross-app-access-grant.md). The following terms are new to this RFC:

| Term | Source | Meaning |
|------|--------|---------|
| **ID-JAG-THV** | ToolHive | The inbound ID-JAG presented by the external client to ToolHive's AS at Grant-THV. Its `aud` is ToolHive's AS issuer URI. |
| **ID-JAG-SaaS** | ToolHive | The outbound ID-JAG obtained by ToolHive at Exchange-SaaS and used at Grant-SaaS. Its `aud` is the backend AS issuer URI. |
| **vMCP session token** | ToolHive | The JWT issued by ToolHive's embedded AS to the external client after validating ID-JAG-THV. Authorizes subsequent tool calls. |
| **Identity Chaining Intermediary** | ToolHive | ToolHive in a deployment where it accepts an inbound ID-JAG-THV (Resource AS role) and issues outbound ID-JAG-SaaS requests (Client role) on the same request path. |
| **act claim** | RFC 8693 §4.1 | JWT claim recording the actor that performed a delegation step; used to preserve the full identity chain when ToolHive issues derived tokens in Phase 2. |
| **Backend AS** | ToolHive | The resource AS protecting a specific backend MCP server; the Grant-SaaS target. |

### 1.5 The Two-Step ID-JAG Protocol

The full wire format for Steps A and B is defined in [THV-0079 §1.3](./THV-0079-cross-app-access-grant.md). In summary:

- **Exchange-THV / Exchange-SaaS** (`grant_type=token-exchange` at the IdP): the Client presents a user ID token as `subject_token` and receives an ID-JAG with `token_type=N_A` and `issued_token_type=urn:ietf:params:oauth:token-type:id-jag`. The ID-JAG is a signed JWT with `typ: oauth-id-jag+jwt`, bound to a specific Resource AS via `aud` and to a specific API via `resource`.
- **Grant-THV / Grant-SaaS** (`grant_type=jwt-bearer` at the Resource AS): the Client presents the ID-JAG as `assertion` and receives a standard Bearer access token.

In this RFC:
- The **inbound path** is a Grant-THV call *into* ToolHive's embedded AS — ToolHive is the Resource AS, and the ID-JAG it receives is ID-JAG-THV.
- The **outbound path** on a Phase 2 tool call is Exchange-SaaS (ToolHive requests ID-JAG-SaaS from the IdP) followed by Grant-SaaS (ToolHive presents ID-JAG-SaaS to the backend AS).

---

## 2. Design Goals

### 2.0 Phase Definitions

| Phase | Scope | Shippable independently? | Backends it unlocks |
|-------|-------|--------------------------|---------------------|
| **Phase 1** — Inbound ID-JAG acceptance | Embedded AS gains a `jwt-bearer` grant handler; validates ID-JAG-THV, issues vMCP session token | Yes | `header_injection`, `token_exchange`, `upstream_inject` |
| **Phase 2** — Identity Chaining Intermediary | Derives a subject token from the Phase 1 session; performs outbound Exchange-SaaS + Grant-SaaS | Requires Phase 1 | `xaa` (backends that themselves use ID-JAG) |

Phase 1 alone is useful and safe to ship. Phase 2 is an additive extension and can follow in a subsequent release.

### 2.1 Goals

- Add an inbound RFC 7523 JWT bearer grant handler to the vMCP embedded AS so that external clients can obtain a vMCP session token by presenting a valid ID-JAG-THV, with no browser redirect (Phase 1).
- Validate ID-JAG-THV against a configurable per-issuer JWKS trust registry so the AS can trust assertions from multiple IdPs without trusting all of them.
- Enforce `jti` replay prevention on accepted ID-JAGs₁ so that a single assertion cannot be reused within its validity window.
- Expose Phase 1 acceptance in the AS discovery document so that clients can discover the supported grant types.
- Enable the full Identity Chaining Intermediary topology for backends using the `xaa` outgoing strategy by deriving a valid subject token for Exchange-SaaS from the inbound session, using one of the mechanisms described in §3.3 (Phase 2).
- Preserve the full identity chain in all derived tokens via the `act` claim (RFC 8693 §4.1), so that every downstream AS can observe "user X via agent Y via ToolHive".
- Stay strictly additive with respect to [THV-0079](./THV-0079-cross-app-access-grant.md): the browser-login path is unchanged, and the `xaa` outgoing strategy continues to work without modification for vMCP sessions seeded by a browser login.

### 2.2 Non-Goals

- **Replacing the browser-login path.** The OIDC authorization code flow and the `xaa` outgoing strategy for browser-seeded sessions ([THV-0079](./THV-0079-cross-app-access-grant.md)) are unchanged. This RFC adds a *parallel* entry point, not a replacement.
- **User-delegated chaining that requires interactive consent.** Some IdPs restrict delegated identity flows to interactive grant types where the user must be present. This RFC targets non-interactive clients; any chaining path that requires a browser redirect at Phase 2 is out of scope.
- **Implementing a full OIDC provider.** Phase 1 adds one new grant type to the existing token endpoint. The AS does not gain userinfo, dynamic client registration, or any endpoint beyond what it already exposes. Phase 2 Direct Backend Trust mints RFC 7523 assertions, not OIDC ID tokens; ToolHive does not become a general-purpose OIDC provider.
- **Client cooperation (forwarding credentials from the inbound client).** The external client presents ID-JAG-THV and nothing more. Phase 2 must work without the client supplying a refresh token, a second ID token, or any additional credential beyond what ID-JAG-THV already asserts.
- **Interactive step-up re-authentication.** Out of scope for the same reasons given in [THV-0079](./THV-0079-cross-app-access-grant.md) §2.2.
- **Multi-actor chaining beyond one delegation level.** This RFC carries one user identity and one agent identity per inbound token. Nested `act.act…` chains are not modelled.

---

## 3. Solution Design

### 3.1 Architecture Overview

The two phases of this RFC layer cleanly on the existing vMCP embedded AS architecture. Phase 1 is purely an inbound grant-handler addition; it does not touch the outgoing-auth layer at all. Phase 2 adds a derivation step between inbound session establishment and outgoing-auth strategy invocation.

The full Phase 2 flow, end to end:

```mermaid
sequenceDiagram
    participant Agent as External AI Agent
    participant IdP as Front-door IdP AS
    participant TH_AS as ToolHive Embedded AS<br>(Resource AS + Client)
    participant Backend_AS as Backend AS
    participant Backend as Backend MCP Server

    Note over Agent,IdP: Pre-step: agent obtains ID-JAG-THV (one-time)
    Agent->>IdP: POST /token<br>grant=token-exchange<br>subject_token=agent ID token<br>requested_token_type=id-jag<br>audience=ToolHive AS issuer URI
    IdP-->>Agent: ID-JAG-THV (aud=ToolHive AS issuer URI)

    Note over Agent,TH_AS: Phase 1: inbound ID-JAG acceptance
    Agent->>TH_AS: POST /token<br>grant=jwt-bearer<br>assertion=ID-JAG-THV
    TH_AS->>TH_AS: Validate ID-JAG-THV signature (IdP JWKS)<br>Check aud, exp, jti (replay cache)
    TH_AS-->>Agent: vMCP session token (JWT)

    Note over Agent,Backend: Diagram shows the generic Phase 1 path. Okta Agent-to-Agent (§3.3.1)<br>replaces Phase 1 entirely: the Agent obtains T3 from the IdP's custom AS<br>directly, and TH_AS validates T3 via OIDC middleware — it never issues a<br>vMCP session token from the flow above for that path.

    Note over Agent,Backend: Phase 2: identity chaining on tool call
    Agent->>TH_AS: tools/call<br>Authorization: Bearer vMCP session token

    alt Okta Agent-to-Agent (§3.3.1)
        TH_AS->>IdP: POST /token (Exchange-SaaS)<br>grant=token-exchange<br>subject_token=T3 (obtained in §3.3.1's Phase 1)<br>requested_token_type=id-jag<br>audience=Backend AS
        IdP-->>TH_AS: ID-JAG-SaaS (aud=Backend AS)
        TH_AS->>Backend_AS: POST /token (Grant-SaaS)<br>grant=jwt-bearer<br>assertion=ID-JAG-SaaS
        Backend_AS-->>TH_AS: backend access token
    else Direct Backend Trust (§3.3.2)
        TH_AS->>TH_AS: Mint RFC 7523 assertion<br>(sub=user, aud=backend token endpoint)
        TH_AS->>Backend_AS: POST /token<br>grant=jwt-bearer<br>assertion=ToolHive-minted JWT
        Backend_AS-->>TH_AS: backend access token
    end

    TH_AS->>Backend: tools/call<br>Authorization: Bearer backend access token
    Backend-->>TH_AS: tool result
    TH_AS-->>Agent: tool result
```

### 3.2 Phase 1 — Inbound ID-JAG Acceptance

Phase 1 adds a single new grant handler to the vMCP embedded AS. The handler accepts requests of the form:

```http
POST /token HTTP/1.1
Host: auth.example.com
Authorization: Basic <client credentials>
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-bearer
&assertion=<ID-JAG-THV>
```

The handler performs the following validation steps before issuing any token:

1. **Parse the assertion.** Extract the JWT header and claims without signature verification. Confirm `typ` is `oauth-id-jag+jwt`.
2. **Resolve the issuer trust entry.** Look up `iss` in the per-issuer JWKS trust registry (§3.4). Reject with `invalid_grant` if the issuer is not configured.
3. **Verify the signature** against the resolved JWKS. Reject with `invalid_grant` on any JOSE error.
4. **Validate standard JWT claims.** `exp` not in the past, `nbf` if present not in the future, `iat` present and recent (bounded clock skew, configurable up to 300 s). Reject with `invalid_grant` on any failure.
5. **Validate `aud`.** Performed by fosite's `rfc7523.Handler` against `AuthorizationServerConfig.GetTokenURLs()`. Fosite's default `GetTokenURLs()` returns only the AS's token endpoint URL (`<issuer>/oauth/token`); this RFC overrides it (§3.2.2) to also accept the bare AS issuer URI, since that is the value this RFC documents external IdPs should use for `aud`. Reject with `invalid_grant` (fosite's built-in behavior) if the audience matches neither.
6. **Check `jti` replay.** Look up `jti` in the replay-prevention cache (§3.5). Reject with `invalid_grant` if the `jti` has been seen before. Insert on first acceptance, with TTL equal to the assertion's `exp − now`.
7. **Extract identity.** Read `sub` and `iss` from the validated claims. Optionally read an email claim if the IdP includes one.
8. **Issue vMCP session token.** Mint a standard signed JWT using the embedded AS's signing key, with `sub` from the assertion, `iss` equal to the AS issuer URI, `aud` equal to the configured MCP resource URL, and `exp` equal to `min(ID-JAG-THV.exp, now + configured_session_lifetime)`. Include an `act` claim recording the original IdP `iss` and `sub`. Mint a fresh `tsid` (token-session-id) claim, matching the shape used for OIDC-callback sessions, and write a session record to the same session storage backend (§3.6) keyed by that `tsid`, with `ExpiresAt` equal to the token's own `exp`.

No upstream ID token is stored in session storage for Phase 1 sessions. The session record exists only to validate subsequent incoming bearer tokens; the `UpstreamIDTokens` map is empty for Phase 1 sessions (relevant to Phase 2 — see §3.3). ID-JAG-THV itself is never written to session storage: it is validated in process memory and discarded once the vMCP session token is issued. This RFC initially considered persisting it (referenced by `tsid`) to support the Audience Alignment mechanism (§3.3.4), but that mechanism is invalidated and no other Phase 2 solution needs a stored copy of ID-JAG-THV, so no such field exists.

The existing fosite-based token endpoint ([THV-0053](./THV-0053-vmcp-embedded-authserver.md)) gains a new fosite `compose.Factory` (§3.2.2), added to the `compose.Compose(...)` call in `pkg/authserver/server_impl.go`'s `createProvider()`, alongside the factories that construct the AS's other grant handlers today (`compose.OAuth2AuthorizeExplicitFactory`, `compose.OAuth2RefreshTokenGrantFactory`, `compose.OAuth2PKCEFactory`). Note this is `pkg/authserver/server_impl.go`, not `pkg/authserver/server/provider.go`'s `NewAuthorizationServer(..., factories...)` — that function and its own `Factory` type exist in the codebase but are exercised only by `provider_test.go`; the AS actually served in production is constructed by `createProvider()`'s direct call to fosite's own `compose.Compose`. Phase 1 does **not** register `compose.RFC7523AssertionGrantFactory` unmodified — two of the validation steps above (the `typ` check in step 1 and the `allowedClientIDs` filtering in step 2) are not reachable through fosite's `rfc7523.Handler` as shipped: its storage callbacks receive only `(issuer, subject, keyId)`, never the raw assertion, and it performs no `typ` check at all. §3.2.2 describes the integration this RFC actually implements.

The AS discovery document (`/.well-known/oauth-authorization-server`) is updated to include `urn:ietf:params:oauth:grant-type:jwt-bearer` in the `grant_types_supported` array.

#### 3.2.1 Trust Registry Configuration

The per-issuer JWKS trust registry is expressed as a new field on the vMCP embedded AS configuration:

```yaml
authServerConfig:
  issuer: https://auth.example.com/vmcp
  # ... existing fields ...
  inboundJWKSTrust:
    - issuer: https://idp.xaa.dev
      jwksUri: https://idp.xaa.dev/jwks
      # Optional: restrict which client IDs in ID-JAG-THV this entry accepts.
      allowedClientIDs:
        - client_3a2341f833945fbb
```

Multiple trust entries are permitted to support federated deployments where different external clients authenticate against different IdPs.

The operator CRD equivalent is a new `InboundJWKSTrust []InboundJWKSTrustEntry` field on the embedded AS spec, with `InboundJWKSTrustEntry` carrying `Issuer`, `JWKSUri`, and `AllowedClientIDs`.

JWKS content is fetched at registration time and cached with a configurable TTL (default 1 h), with refresh-on-verify-failure to handle key rotation. The fetch URL must be HTTPS; plain HTTP is rejected at configuration validation time.

#### 3.2.2 Fosite Integration

Phase 1 registers a custom fosite `compose.Factory` (matching the `func(config fosite.Configurator, storage interface{}, strategy interface{}) interface{}` signature `pkg/authserver/server_impl.go`'s `createProvider()` already passes to `compose.Compose(...)` for the other grant handlers) built from two pieces:

1. **`InboundTrustKeyStorage`** implements fosite's `rfc7523.RFC7523KeyStorage` interface against the trust registry (§3.4):
   - `GetPublicKey`/`GetPublicKeys(ctx, issuer, subject, ...)` ignore `subject` entirely and resolve the cached JWKS by `issuer` alone. This is a deliberate mismatch with `rfc7523`'s original design target (per-subject pre-registered service-account keys): ID-JAG-THV is trusted by *issuer*, not by an individually registered subject, so `subject` carries no meaning here. Unconfigured issuers return an error, which fosite maps to `invalid_grant`.
   - `GetPublicKeyScopes` is called unconditionally by fosite, but its result is only consulted inside a loop over `request.GetRequestedScopes()`, and Phase 1's Grant-THV wire format (§3.2) never includes a `scope` parameter — Phase 1 explicitly rejects any Grant-THV request that supplies one, with `invalid_scope`. The method's return value is therefore dead by construction, not an unspecified behavior; it is implemented to satisfy the interface and its result never meaningfully affects a Phase 1 request.
   - `IsJWTUsed`/`MarkJWTUsedForTime` **are** the `jti` replay-prevention cache described in §3.5 — fosite's `rfc7523.Handler.validateTokenClaims` calls these two methods directly. §3.5 is not a separate hook layered on top of the fosite integration; it is this interface, backed by the storage described there (in-memory LRU or Redis).

2. **A thin `fosite.TokenEndpointHandler` decorator** wrapping a `*rfc7523.Handler{Storage: InboundTrustKeyStorage{...}}`. Before delegating, the decorator re-parses the `assertion` form parameter — the same string fosite's own handler parses internally — to check: (a) the JWT `typ` header equals `oauth-id-jag+jwt` (step 1 of §3.2), rejecting with `invalid_grant` if not; (b) if the resolved trust entry's `allowedClientIDs` is non-empty, the assertion's `client_id` claim is a member (step 2), rejecting with `invalid_grant` if not. Both checks run on unverified claims purely to fail fast before the more expensive JWKS/signature path; the decorator then delegates to the wrapped `*rfc7523.Handler`, which performs the security-critical checks (signature, `aud`, `exp`/`nbf`/`iat`, `jti`) using fosite's own, already-reviewed validation logic. `CanHandleTokenEndpointRequest` and `CanSkipClientAuth` on the decorator pass through to the wrapped handler unchanged.

`compose.Compose(...)` passes a single opaque `storage interface{}` argument to every factory in its call list; each factory type-asserts out the interface it needs (`compose.RFC7523AssertionGrantFactory` itself does `storage.(rfc7523.RFC7523KeyStorage)`). Consequently, whatever storage value `createProvider()` currently passes to `compose.Compose` must additionally satisfy `rfc7523.RFC7523KeyStorage` for this factory to type-assert successfully — either by implementing the interface directly on the existing storage type, or by composing a small wrapper that embeds the existing storage and adds the `InboundTrustKeyStorage` methods. This RFC does not require changing what storage backend the other grant handlers use; it only requires that backend to additionally answer the five `RFC7523KeyStorage` methods.

This is more implementation work than "register a factory," but it is the only way to enforce `typ` and `allowedClientIDs` given that fosite's storage callbacks never expose the raw assertion. The checks that are security-critical — signature verification, `aud`, `exp`/`nbf`/`iat`, `jti` replay — still run entirely inside fosite's own `rfc7523.Handler`; the decorator only adds two additional fast-fail checks in front of it, matching the same shape fosite already uses for pluggable client-authentication and scope-strategy hooks elsewhere in the provider.

### 3.3 Phase 2 — Subject Token Derivation for Outbound XAA

When the `xaa` outgoing strategy runs for a Phase 1 session, `Identity.UpstreamIDTokens` is empty: no browser login captured an upstream ID token. Exchange-SaaS of the outgoing `xaa` strategy requires a `subject_token` (§4.3 of the draft) — typically an ID token. Without one, the strategy returns `ErrUpstreamTokenNotFound` and the call fails.

Phase 2 introduces a **subject token derivation** hook invoked between session validation and outgoing-strategy execution for Phase 1 sessions. The hook produces a subject token that the `xaa` strategy can present at Exchange-SaaS without any additional credential from the external client. The supported mechanisms are §3.3.1 and §3.3.2 below; two earlier candidates were evaluated and dropped — see §3.3.4.

#### 3.3.1 Okta Agent-to-Agent (IdP-Specific EA Feature)

**Mechanism.** Certain IdPs provide an Early Access "agent-to-agent" capability in which an AI agent application is registered with a dedicated custom authorization server hosted by the IdP. Under this model, Phase 1 proceeds entirely between the MCP client and the IdP's endpoints — ToolHive's embedded Fosite AS plays no role in Phase 1.

The MCP client calls the IdP's custom AS provisioned for ToolHive's agent registration to obtain T3 — a standard access token bound to ToolHive's resource and carrying the full `act` chain (`sub=user`, `act={ client_id=<MCP client app> }`). The Grant-THV step in Phase 1 targets the IdP's custom AS URL (`https://{idp-domain}/oauth2/{thv-as-id}/v1/token`), not ToolHive's embedded AS token endpoint. ToolHive acts as a pure OAuth resource server in Phase 1: its OIDC middleware validates incoming Bearer T3 against the IdP custom AS's JWKS.

At Exchange-SaaS, ToolHive presents T3 as `subject_token` with `subject_token_type=urn:ietf:params:oauth:token-type:access_token` (RFC 8693 §2.1) at the IdP's org AS. Because T3 is signed by the IdP's own infrastructure, the org AS accepts it and mints ID-JAG-SaaS scoped to the backend — no external trust configuration is required.

**Requirements:**
- ToolHive registered as an AI agent application at the IdP with a dedicated custom AS.
- Delegation link configured at the IdP between the MCP client's application and ToolHive's application.
- Resource connection configured at the IdP between ToolHive's application and the backend resource application.
- `client_assertion` (signed JWT) required on every IdP token endpoint call.
- The IdP's org AS (`/oauth2/v1/token`) handles Exchange-SaaS steps; the IdP's custom AS (`/oauth2/{id}/v1/token`) handles Grant-THV.

**Operational constraints:**
- Requires the agent-to-agent EA feature enabled on the IdP org; no GA equivalent exists.
- Four admin configurations required: AI agent app registration, backend resource app registration, delegation link (MCP client → ToolHive), and resource connection (ToolHive → backend).
- T3 lifetime is fixed by the IdP at token issuance time and is not configurable by ToolHive. After T3 expires, the full Phase 1 exchange must be re-run; tool calls attempted with an expired T3 fail with `invalid_grant`.
- Entirely specific to IdPs that support this EA capability; does not generalize to other IdPs.

**Configuration signal.** `subjectTokenSource: okta_agent_to_agent` on the `xaa` backend config.

#### 3.3.2 Direct Backend Trust (Gateway-as-OAuth-AS, RFC 7523)

**Mechanism.** ToolHive's embedded AS mints a short-lived JWT assertion directly for each configured backend at call time. The backend resource server is configured out-of-band to trust ToolHive's JWKS as a trusted issuer — the same trust establishment used for any OIDC issuer. No Exchange-SaaS step is involved: ToolHive makes a single RFC 7523 `jwt-bearer` call per tool call directly to the backend resource AS. The IdP AS is not involved in Phase 2.

**JWT assertion claims (ToolHive-minted):**

| Claim | Value |
|-------|-------|
| `iss` | ToolHive AS issuer URI |
| `sub` | User subject from the validated Phase 1 session |
| `aud` | Backend RS token endpoint URL — the RS must accept its own token endpoint URL as audience (RFC 7523 §3); do not use the issuer URI |
| `act` | `{ "sub": "<ToolHive AS client identifier>" }` — identifies ToolHive as the delegating actor (RFC 8693 §4.4) |
| `scope` | Backend-specific scopes from `MCPExternalAuthConfig` — REQUIRED; omitting scope may cause the RS to grant full access or reject the request |
| `exp` | `now + 60s` |
| `jti` | Random UUID — RS SHOULD track used values within the assertion's validity window (RFC 7523 §3) |

**Security properties:**
- ToolHive becomes the root of trust for all backends configured for this solution. A signing key compromise affects every backend that trusts that key.
- **Mitigation:** use a distinct signing key pair per backend, each with its own JWKS endpoint. This limits the blast radius of a key compromise to a single backend.
- There is no cryptographic binding between the vMCP session and the minted assertion. A subject-injection bug in session handling is indistinguishable from key compromise at the backend; the backend cannot detect it independently.
- The `act` claim makes the delegation chain visible in the assertion, but most resource server implementations do not enforce actor restrictions. Backend-side actor policy must be implemented explicitly if actor restriction is required.

**Operational properties:**
- Works with any backend that supports RFC 7523 `jwt-bearer` grant.
- No IdP-specific requirements; no EA features or custom AS registrations needed.
- Backend configuration: register ToolHive's JWKS endpoint once per ToolHive deployment, or once per backend if using per-backend signing keys.
- Key rotation: publish the new key in the JWKS before first use; retain the old key in the JWKS until RS JWKS caches have expired (typically 24 h). SHOULD use `kid`-keyed overlap rotation to avoid verification failures during cache refresh windows.
- Phase 1 is independent — any login mechanism works (browser OIDC flow or the jwt-bearer grant described in §3.2).

**Configuration signal.** `subjectTokenSource: gateway_direct_trust` on the `xaa` backend config.

#### 3.3.3 Solution Selection Summary

| Deployment context | Phase 1 works? | Phase 2 path | Notes |
|-----|----------------|--------------|-------|
| Any IdP; backend supports RFC 7523 | Yes | Direct Backend Trust (§3.3.2) | No IdP involvement in Phase 2; ToolHive holds root of trust for backends |
| IdP with agent-to-agent EA feature | Yes | Okta Agent-to-Agent (§3.3.1) | IdP-specific; requires four admin configurations |
| External IdP (user-delegated chain) | Yes | Not yet supported | User-delegated chain via interactive flow; out of scope per §2.2 |

#### 3.3.4 Alternatives Considered

Two mechanisms were evaluated and dropped before the solutions above. Neither is retained as a reserved, forward-compatible config surface: both fail for structural reasons — a spec design choice in one case, a federation-architecture mismatch in the other — that no near-term draft revision or IdP feature is likely to change, so there is no credible trigger that would ever move either from "invalidated" to "operational." Carrying a reserved enum value or a dedicated session-storage field for either is complexity paid against a mechanism with no activation path; if the standards or vendor landscape ever does shift here, the resulting design would need its own follow-up RFC informed by whatever the new mechanism actually looks like, rather than matching a guess made today.

**Audience Alignment.** The inbound ID-JAG-THV could be presented directly at Exchange-SaaS as `subject_token`. This fails for two independent reasons: XAA §4.3 permits only `id_token`, `saml2`, and `refresh_token` as `subject_token_type` inputs — the `id-jag` URN is an output type, not a valid exchange input; and audience alignment requires the operator to set `client_id` equal to the AS issuer URI, which IdP-managed registration typically does not permit. This is not a temporary spec gap: draft §9.3 explicitly states an ID-JAG "MUST NOT" be reused as the authorization grant for any Resource AS other than the one named in its `aud`, and open WG discussion of delegation semantics (e.g. issue #73 on the oauth-wg tracker) preserves this two-party trust boundary rather than proposing a re-presentable ID-JAG. `audience_alignment` is not implemented and no config surface is reserved for it.

**ToolHive-Issued Internal ID Token.** ToolHive could mint a short-lived OIDC ID token from its own issuer URI and present it at Exchange-SaaS, with the backend IdP configured to trust ToolHive as an OIDC upstream. This fails because IdP "trusted OIDC upstream" configuration is designed for redirect-based user federation, not M2M token exchange — IdPs do not accept machine-minted tokens as `subject_token` at their token exchange endpoint through this trust path. This is architectural, not a maturity gap: the closest real-world analog, Entra ID Workload Identity Federation, authenticates the calling *service* itself via `client_assertion` in a `client_credentials` flow — it carries no user/`subject_token` semantics and explicitly disallows using Entra-issued tokens as the federated credential. `th_issued_id_token` is not implemented and no config surface is reserved for it.

### 3.4 JWKS Trust Registry

The trust registry is consulted during Phase 1 inbound assertion validation to resolve the signing JWKS for ID-JAG-THV.

The registry is implemented as an in-process map keyed by issuer URI. Each entry holds:

- The configured `jwksUri`.
- The cached JWKS (fetched at startup, refreshed on TTL expiry and on signature-verification failure).
- A list of `allowedClientIDs` (empty = accept any `client_id` in the assertion).

JWKS refresh uses the standard ToolHive HTTP client with TLS, a configurable timeout (default 10 s), and exponential backoff. A failed refresh does not immediately invalidate the cached key set; validation continues against the stale cache until the cache entry expires (configurable, default 15 m), preventing a JWKS endpoint outage from locking out all inbound clients.

The registry is initialized from operator configuration at AS startup and is read-only at runtime. Dynamic registration is not supported.

### 3.5 Replay Prevention Cache

The `jti` replay-prevention cache prevents a presented ID-JAG-THV from being accepted more than once within its validity window. It is implemented as the `IsJWTUsed`/`MarkJWTUsedForTime` methods of the `InboundTrustKeyStorage` adapter described in §3.2.2 — fosite's `rfc7523.Handler` calls these directly during claim validation; there is no separate replay-check code path to build. Implementation requirements:

- Keyed by `(iss, jti)` — the issuer is included so that two IdPs issuing the same `jti` value do not collide. (`MarkJWTUsedForTime`'s signature only takes `jti`; the adapter prefixes it with the assertion's `iss` before writing.)
- TTL per entry equals `exp − now` at insertion time.
- Must be shared across all instances in a horizontally scaled deployment (§3.6).
- In single-instance deployments a local in-memory LRU cache (bounded by max-assertions-in-flight × max-assertion-lifetime) is sufficient.
- In multi-instance deployments the same Redis backend used for upstream token storage ([THV-0035](./THV-0035-auth-server-redis-storage.md)) is used, with a key shaped to match that backend's existing schema: `thv:auth:{<server-name>}:jti:<iss>:<jti>`, following the same `thv:auth:{<server-name>}:<type>:<identifier>` convention used elsewhere (e.g. the existing client-assertion replay cache at `thv:auth:{<server-name>}:jwt:<jti>`), rather than introducing a differently-shaped prefix.

The cache type (in-memory vs. Redis) is determined by the same `storageConfig` field that governs the rest of the AS session storage; no separate configuration key is introduced.

### 3.6 Horizontal Scaling

[THV-0047](./THV-0047-vmcp-proxyrunner-horizontal-scaling.md) describes the Redis-backed session storage that enables multiple vMCP proxy-runner pods to share session state. Phase 1 of this RFC integrates naturally with that model: the `jti` replay cache (§3.5) uses the same Redis backend, and the vMCP session tokens issued in Phase 1 are validated by the same token-reader used for browser-login sessions. No per-instance state is introduced.

Phase 2 Direct Backend Trust adds one new at-rest artifact per backend: the signing key used to mint RFC 7523 assertions. When per-backend keys are chosen (the recommended configuration for blast-radius limiting), each key must be stored in a location readable by all replicas — Kubernetes Secrets, mounted via the existing signing-key secret mechanism.

### 3.7 Configuration

New fields introduced by this RFC (full YAML examples are in the operator guide):

**`VirtualMCPServer.spec.authServerConfig.inboundJWKSTrust[]`** (Phase 1) — list of trusted inbound ID-JAG issuers. Each entry has `issuer` (URI, required), `jwksUri` (HTTPS, required), and `allowedClientIDs` (optional whitelist). An empty list disables Phase 1.

**`MCPExternalAuthConfig.spec.xaa.subjectTokenSource`** (Phase 2) — selects the subject token derivation mechanism:

| Value | Description |
|---|---|
| `upstream_id_token` | (default, THV-0079) Browser-login upstream ID token |
| `okta_agent_to_agent` | IdP agent-to-agent EA path (§3.3.1); T3 is IdP-issued |
| `gateway_direct_trust` | ToolHive-minted RFC 7523 assertion (§3.3.2); direct backend trust |

Two earlier candidates, "audience alignment" and "ToolHive-issued internal ID token," were evaluated and dropped for structural reasons unlikely to change (§3.3.4); no enum values or config fields are reserved for them.

**`MCPExternalAuthConfig.spec.xaa.backendTokenEndpoint`** (Phase 2 Direct Backend Trust) — the backend RS token endpoint URL, used as `aud` in ToolHive-minted RFC 7523 assertions. REQUIRED for `gateway_direct_trust`; must be the full token endpoint URL, not the issuer URI (RFC 7523 §3).

The issued vMCP session token includes an `act` claim tracing back to ID-JAG-THV's `iss` and `sub` so the full delegation chain is auditable.

---

## 4. Security Considerations

Identity chaining requires ToolHive to assert user identity it did not directly authenticate via a browser interaction. Every step in this chain is a site where a failure — misconfiguration, validation gap, or implementation error — can make ToolHive a confused deputy: an entity that does something on a user's behalf that the user did not intend or that the downstream system should not have been asked to do. The subsections below cover each identified threat and the corresponding mandatory mitigations.

### 4.1 Identity Authority Without Direct Authentication

**Threat.** ToolHive issues a vMCP session token (Phase 1) and potentially derived credentials in Phase 2 (RFC 7523 assertions for Direct Backend Trust) for a user it has never interactively authenticated. Its authority to do so rests entirely on validating ID-JAG-THV. If that validation is incomplete or skipped, ToolHive can be induced to issue tokens for arbitrary users.

**Mitigations:**
- The full validation chain (signature, `aud`, `exp`, `jti`, issuer trust registry) in §3.2 is mandatory. Each step is independently necessary; none may be skipped or soft-degraded.
- Issuance is gated on a configured trust registry entry for the assertion's `iss`. Assertions from issuers not in the registry are rejected at the first validation step, before any JOSE operation.
- Phase 2 Direct Backend Trust assertions include the `act` claim identifying ToolHive as the actor. An assertion without `act` MUST NOT be issued; the issuance path fails closed.

### 4.2 Revocation Gap

**Threat.** After ToolHive has accepted ID-JAG-THV and issued a vMCP session token, the user's account may be disabled, their session revoked, or the IdP's grant policy may change. ToolHive has no ongoing visibility into the IdP's revocation state during the session lifetime.

**Mitigations:**
- The vMCP session token's `exp` is hard-bound to `min(ID-JAG-THV.exp, now + configured_session_lifetime)`. ToolHive cannot issue a session that outlives the original assertion.
- Phase 2 Direct Backend Trust assertions have `exp = now + 60s` and are single-use — they expire within 60 seconds and carry a unique `jti`. This minimizes the window during which ToolHive can act on behalf of a revoked user at the outbound chain.
- Operators should configure `configured_session_lifetime` conservatively (e.g., 1 h) relative to their IdP's assertion lifetimes and rotation policy.
- There is currently no mechanism for the IdP to push revocation events to ToolHive mid-session. This is a known limitation; introspection-based revocation polling is a potential future extension.

### 4.3 Replay Attack on ID-JAG-THV

**Threat.** A valid ID-JAG-THV intercepted during transit (e.g., through a TLS-stripping proxy or via log exfiltration) could be replayed against ToolHive's token endpoint to create additional sessions for the same user within the assertion's validity window.

**Mitigations:**
- The `jti` replay-prevention cache (§3.5) ensures each `jti` value is accepted exactly once per issuer within the assertion's validity window.
- In horizontally scaled deployments, the replay cache uses Redis so that no two instances accept the same `jti` independently.
- ToolHive's token endpoint MUST be served over TLS only (this is an existing requirement; not changed by this RFC).

### 4.4 Additional Controls

- **§9.3 compliance.** ToolHive MUST NOT forward ID-JAG-THV as an authorization grant to any downstream AS. All Phase 2 solutions produce a new credential for Grant-SaaS (ID-JAG-SaaS for Okta Agent-to-Agent; a ToolHive-minted RFC 7523 assertion for Direct Backend Trust); ID-JAG-THV is never forwarded.
- **Derived token replay.** Phase 2 Direct Backend Trust assertions carry a unique `jti` and `exp = now + 60s`. Backend resource servers SHOULD maintain a `jti` use cache (RFC 7523 §3). Each tool call produces a fresh assertion; ToolHive does not reuse minted assertions across parallel requests.
- **Audit logging.** The following events MUST be emitted as structured records: ID-JAG-THV accepted/rejected (with `iss`, `sub`, `jti`); subject token derived (with solution used); Exchange-SaaS/B' succeeded/failed (with `sub`, target IdP/AS, scopes or error code). Token values MUST NOT appear in logs.
- **JWKS SSRF.** `jwksUri` entries in the trust registry are HTTPS-only; this is enforced at config validation time. Consistent with the `idpTokenUrl` posture in [THV-0079](./THV-0079-cross-app-access-grant.md) §4.8.
- **Clock skew.** `exp`/`nbf` validation uses a configurable tolerance (default 60 s, maximum 300 s).
- **No at-rest copy of ID-JAG-THV.** ID-JAG-THV is validated in process memory during Grant-THV and discarded once the vMCP session token is issued (§3.2 step 8) — it is never written to session storage and never embedded in the vMCP session token handed to the external client. A compromised or leaked vMCP session token, or a compromised session store, therefore cannot disclose ID-JAG-THV: there is no persisted copy to disclose. This is a deliberate design choice, not an incidental property.

---

## 5. Alternatives Considered

### 5.1 Forward ID-JAG-THV Directly to Backend AS

The simplest conceivable Phase 2 would forward ID-JAG-THV to each backend AS directly as a jwt-bearer assertion. This violates §9.3 of the draft (the `aud` of ID-JAG-THV is ToolHive's issuer URI, not the backend AS issuer URI) and would be rejected by a spec-compliant backend AS with `invalid_grant`. It also creates a single bearer credential that, if intercepted, grants access to both ToolHive and every backend — unacceptable blast radius. Not viable.

### 5.2 Require the Client to Supply Upstream ID Tokens Separately

Rather than deriving a subject token, require the external client to also include the user's upstream ID token in the Phase 1 request. This violates the "no client cooperation" non-goal (§2.2): agents that obtained ID-JAG-THV via their own IdP may not have access to the user's upstream ID token from the front-door IdP — that token belongs to the user's browser session. Requiring it would restrict the pattern to clients that have special-cased ToolHive and would negate the benefit of the standard ID-JAG protocol.

### 5.3 Store ID-JAG-THV and Use It Directly at Exchange-SaaS

Store ID-JAG-THV in the session record and present it as `subject_token` at Exchange-SaaS with a new `subject_token_type` URN. This conflates the use of ID-JAG-THV as a session-establishment credential with its use as a step-A' input. The draft defines no `subject_token_type` URN for ID-JAGs; XAA draft §4.3 accepts only `id_token`, `saml2`, and `refresh_token` as input types, so IdPs reject requests carrying the `id-jag` type with `invalid_grant`. This approach is not viable as either a production path or an alternative to the implemented solutions.

### 5.4 OAuth 2.0 Token Introspection for Revocation

Add a revocation-polling loop that periodically introspects the vMCP session token against the original IdP's introspection endpoint and revokes ToolHive's session if the IdP reports the credential as inactive. This would address §4.2's revocation gap. It was considered and deferred: introspection is not universally supported (and many IdPs restrict it to confidential clients); the polling interval introduces a revocation window regardless; and the operational complexity is high relative to the short session lifetimes already enforced. Deferred to a follow-up RFC.

---

## 6. Compatibility

### 6.1 Backward Compatibility

This RFC is strictly additive:

- Phase 1 adds a new grant type to the token endpoint behind configuration (`inboundJWKSTrust`). When `inboundJWKSTrust` is empty (the default), the new grant handler is not registered and behavior is identical to today.
- Phase 2 adds `subjectTokenSource` to `XAAConfig`. An absent field means the existing browser-login path applies — `Identity.UpstreamIDTokens` is populated from the browser-captured ID token, and `xaa` works as described in [THV-0079](./THV-0079-cross-app-access-grant.md). No existing `xaa` deployment is affected.
- The vMCP session tokens issued in Phase 1 are standard JWTs with an additional `act` claim. Existing incoming-auth middleware validates them identically to browser-login tokens (signature + `aud` + `exp`); the `act` claim is ignored by the middleware but available for downstream inspection.
- `InboundJWKSTrust` is a new CRD field with zero value = disabled; existing CRD instances need no migration.

### 6.2 Forward Compatibility

- The Phase 2 solution is selected via a `subjectTokenSource` string field. Additional solutions can be added as new enum values without breaking existing configurations — adding a value later is non-breaking regardless of whether it was pre-reserved, so the two dropped candidates in §3.3.4 do not need reserved placeholders today.
- The trust registry `InboundJWKSTrustEntry` struct is extensible: additional filtering fields (e.g., `allowedScopes`, `allowedSubjectPatterns`) can be added without breaking existing entries.
- If `draft-ietf-oauth-identity-assertion-authz-grant` or RFC 8693 ever gains a standardized way to re-present an ID-JAG as exchange input, adopting it would need its own design work informed by that mechanism's actual shape (draft §9.3's current MUST-NOT-reuse language makes this unlikely — see §3.3.4) — a new `subjectTokenSource` value at that point is a non-breaking addition, not a migration.

---

## 7. Implementation Plan

### Phase 1 — Inbound ID-JAG Acceptance (Resource AS)

Goal: ToolHive accepts ID-JAG-THV as a grant and issues a vMCP session token. No browser login required.

- Implement `InboundTrustKeyStorage` (fosite's `rfc7523.RFC7523KeyStorage`) against the trust registry, and the `fosite.TokenEndpointHandler` decorator that wraps `*rfc7523.Handler` with the `typ`/`allowedClientIDs` pre-checks (§3.2.2). Register the decorator's factory in `pkg/authserver/server_impl.go`'s `createProvider()` `compose.Compose(...)` call, alongside the existing grant-handler factories — not `compose.RFC7523AssertionGrantFactory` unmodified, and not `pkg/authserver/server/provider.go`'s `NewAuthorizationServer`/`Factory` abstraction, which is unused outside its own tests. Confirm the storage value passed to `compose.Compose` satisfies `rfc7523.RFC7523KeyStorage` (directly or via a wrapper) — see §3.2.2.
- Override `AuthorizationServerConfig.GetTokenURLs()` to accept both the token endpoint URL and the bare AS issuer URI as valid `aud` values (§3.2, Critical Issue resolved in review).
- Implement `InboundJWKSTrust` config parsing and JWKS-fetch/cache machinery.
- Wire `IsJWTUsed`/`MarkJWTUsedForTime` on `InboundTrustKeyStorage` to the `jti` replay store (in-memory for single-instance; §3.5 describes the Redis path for multi-instance) — this *is* the replay cache, not an add-on.
- Emit vMCP session token with `act` and `tsid` claims; write the corresponding session record (§3.2 step 8).
- Update AS discovery document.
- Add `InboundJWKSTrust` CRD field and operator converter.

#### Dependencies

- [THV-0053](./THV-0053-vmcp-embedded-authserver.md) — embedded AS; fosite token endpoint, `pkg/authserver/server_impl.go`'s `createProvider()`/`compose.Compose(...)` construction.
- [THV-0035](./THV-0035-auth-server-redis-storage.md) — Redis session storage; used for the multi-instance replay cache (§3.5).
- `github.com/ory/fosite/handler/rfc7523` (a subpackage of `ory/fosite`, already a direct dependency) — the server-side JWT-bearer grant handler this RFC wraps. Not to be confused with `pkg/oauthproto/jwtbearer` (THV-0079), which is a *client* package for outbound Step B and is not reused on this inbound path.

### Phase 2 — Identity Chaining Intermediary (Client + Resource AS)

Goal: inbound sessions can drive outbound `xaa` by deriving a valid subject token.

- Add `subjectTokenSource` field to `XAAConfig` and CRD `XAASpec`, with `okta_agent_to_agent` and `gateway_direct_trust` as the only accepted values.
- Implement Okta Agent-to-Agent: IdP-specific agent-to-agent EA path; T3 is validated by ToolHive's OIDC middleware (ToolHive is a resource server, not the token issuer); T3 presented as `access_token` at Exchange-SaaS.
- Implement Direct Backend Trust: ToolHive AS mints short-lived RFC 7523 JWT assertions per tool call; per-backend signing key management; `kid`-keyed JWKS publication; `jti` generation and scope injection from `MCPExternalAuthConfig`.
- Unit tests covering Okta Agent-to-Agent and Direct Backend Trust with mock IdP / backend AS servers.

#### Dependencies

- Phase 1 of this RFC (inbound acceptance).
- [THV-0079](./THV-0079-cross-app-access-grant.md) — `xaa` outgoing strategy; Phase 2 extends its `XAAConfig`, not replaces it.

#### Documentation

- Update the vMCP embedded AS operator guide to document `inboundJWKSTrust` configuration.
- Add `docs/arch/xaa-identity-chaining-intermediary.md` covering the dual-role topology, the Phase 2 solutions (C and D), and the deployment compatibility table.
- Add a runbook section covering `jti` replay cache eviction for Redis deployments and JWKS endpoint failure response.

---

## 8. Testing Strategy

### 8.1 Unit Tests

- Phase 1 grant handler against `httptest` mock assertion issuers: happy path, missing `inboundJWKSTrust`, invalid signature, wrong `aud`, expired assertion, duplicate `jti` (replay), unknown issuer.
- `InboundTrustKeyStorage`: `GetPublicKey`/`GetPublicKeys` resolve by issuer regardless of `subject`; `IsJWTUsed`/`MarkJWTUsedForTime` round-trip against the same store backing §3.5.
- Decorator pre-checks: assertion with wrong `typ` header rejected before signature verification; assertion whose `client_id` is absent from a configured `allowedClientIDs` rejected; both checks are bypassed correctly when `allowedClientIDs` is empty.
- `GetTokenURLs()` override: assertions with `aud` = token endpoint URL and `aud` = bare issuer URI both accepted; an unrelated `aud` rejected.
- JWKS cache: TTL expiry triggers refresh, refresh-on-verify-failure, stale-cache grace period during outage.
- Phase 2 Okta Agent-to-Agent: T3 validated by OIDC middleware rather than fosite embedded AS; T3 presented with `subject_token_type=access_token` in Exchange-SaaS request.
- Phase 2 Direct Backend Trust: minted assertion claims (`iss`, `sub`, `aud` = token endpoint URL, `act`, `scope`, `exp`, `jti`); `exp` pinned to 60 s; `jti` uniqueness across parallel tool calls; per-backend signing key selection.
- vMCP session token `act` claim round-trip: accept inbound → issue session → validate session in middleware, `act` preserved.

### 8.2 Integration Tests

- Phase 1 end-to-end against the xaa.dev sandbox: external agent obtains ID-JAG-THV from `idp.xaa.dev`, presents to ToolHive's AS, receives session token, calls `tools/list`.
- Phase 2 Direct Backend Trust end-to-end: same session calls a backend configured with `subjectTokenSource: gateway_direct_trust`; assert that the backend returns a tool result and that the jwt-bearer grant request to the backend RS includes correct `sub`, `aud` (token endpoint URL), `scope`, and `act` claims.

### 8.3 Security Tests

- `jti` replay: present the same ID-JAG-THV twice in rapid succession; the second must be rejected.
- Expired ID-JAG-THV (1 s past `exp`, zero clock-skew tolerance): rejected.
- Wrong `aud` (ID-JAG-THV addressed to a different AS): rejected.
- Direct Backend Trust assertion `aud` set to issuer URI instead of token endpoint URL: configuration validation returns error before issuance.
- Direct Backend Trust assertion configured without `scope`: configuration validation returns error.
- Direct Backend Trust `jti` replay: backend RS receives the same assertion `jti` twice within 60 s; second presentation MUST be rejected (requires RS-side test harness).

---

## Open Questions

1. **Direct Backend Trust per-backend key management UX.** The recommended configuration uses per-backend signing keys to limit blast radius; this requires operators to configure and rotate multiple JWKS endpoints. Should ToolHive provide tooling (CLI subcommand or operator controller) to automate per-backend JWKS endpoint publication and `kid`-keyed key rotation, or is a single shared signing key the default with per-backend keys as an explicit opt-in?

2. **Direct Backend Trust assertion lifetime.** The minted assertion `exp` is fixed at `now + 60s`. Is 60 s too short for high-latency backend token endpoint calls in some deployments? A configurable ceiling (default 60 s) is proposed; feedback welcome.

3. **Phase 1 with plain OIDC ID tokens.** Should the Phase 1 handler also accept a standard OIDC `id_token` (not an ID-JAG)? This would enable browser-less access for clients whose IdP cannot issue ID-JAGs. It deviates from strict XAA semantics but is valid under RFC 7523.

4. **Okta Agent-to-Agent T3 re-authentication signal.** When T3 expires, the MCP client must re-run the full Phase 1 exchange to obtain a new T3. Should ToolHive's OIDC middleware return a specific `WWW-Authenticate` challenge (e.g., with `error=invalid_token` and a machine-readable hint) to trigger MCP client re-authentication proactively, rather than returning a generic `401 Unauthorized` that the client cannot distinguish from authorization failure?

---

## References

- IETF: *Identity Assertion JWT Authorization Grant*, `draft-ietf-oauth-identity-assertion-authz-grant-04`, revised May 2026. <https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-assertion-authz-grant/>
- [RFC 8693 — OAuth 2.0 Token Exchange](https://datatracker.ietf.org/doc/html/rfc8693) (Exchange-THV / Exchange-SaaS).
- [RFC 7523 — JWT Profile for OAuth 2.0 Client Authentication and Authorization Grants](https://datatracker.ietf.org/doc/html/rfc7523) (Grant-THV / Grant-SaaS / Phase 1 grant handler).
- [RFC 7521 — Assertion Framework for OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc7521).
- [RFC 7519 — JSON Web Token (JWT)](https://datatracker.ietf.org/doc/html/rfc7519).
- [RFC 9470 — OAuth 2.0 Step Up Authentication Challenge Protocol](https://datatracker.ietf.org/doc/html/rfc9470).
- *xaa.dev cross-app access sandbox*: <https://xaa.dev>.
- *Okta AI Agent docs — cross-app access*: <https://developer.okta.com/docs/guides/configure-cross-app-access/main/>.
- [THV-0007](./THV-0007-token-exchange-middleware.md) — Token-exchange middleware (RFC 8693 outbound foundation).
- [THV-0031](./THV-0031-auth-server-integration.md) — Embedded Authorization Server in the Proxy Runner.
- [THV-0035](./THV-0035-auth-server-redis-storage.md) — Redis-backed AS session storage.
- [THV-0047](./THV-0047-vmcp-proxyrunner-horizontal-scaling.md) — vMCP proxy-runner horizontal scaling.
- [THV-0053](./THV-0053-vmcp-embedded-authserver.md) — vMCP Embedded Authorization Server.
- [THV-0054](./THV-0054-vmcp-upstream-inject-strategy.md) — vMCP `upstream_inject` outgoing-auth strategy.
- [THV-0079](./THV-0079-cross-app-access-grant.md) — XAA outgoing strategy (ID-JAG client, browser-login path).
- ToolHive code: `pkg/oauthproto/jwtbearer/`, `pkg/auth/identity.go`, `pkg/vmcp/auth/strategies/xaa.go`, `pkg/vmcp/auth/converters/xaa.go`, `cmd/thv-operator/api/v1beta1/mcpexternalauthconfig_types.go`.

---

## RFC Lifecycle

<!-- This section is maintained by RFC reviewers -->

### Review History

| Date | Reviewer | Decision | Notes |
|------|----------|----------|-------|
| 2026-07-01 | @jhrozek | Draft | Initial submission |

### Implementation Tracking

| Repository | PR | Status |
|------------|-----|--------|
| toolhive | — | Not started |
