# RFC-0073: vMCP Session Binding by Identity Tuple

- **Status**: Draft
- **Author(s)**: Jakub Hrozek (@jhrozek)
- **Created**: 2026-05-20
- **Last Updated**: 2026-05-21
- **Target Repository**: toolhive
- **Related Issues**: [toolhive#5306](https://github.com/stacklok/toolhive/issues/5306)
- **Supersedes**: the session-binding portion of [THV-0038](./THV-0038-session-scoped-client-lifecycle.md); updates [THV-0047](./THV-0047-vmcp-proxyrunner-horizontal-scaling.md) to use the new binding format

## Summary

The Virtual MCP server (vMCP) currently binds each MCP transport session to an HMAC-SHA256 hash of the raw bearer-token bytes presented at session creation, then rejects any subsequent request whose token bytes hash differently. This terminates the session on every legitimate OAuth refresh because the new access token has different bytes for the same identity. This RFC replaces the binding with a stable `(iss, sub)` identity tuple extracted from the OIDC identity's `Claims`, eliminates the HMAC plumbing entirely, and migrates legacy sessions via an invalidate-on-read scheme.

## Problem Statement

[THV-0038](./THV-0038-session-scoped-client-lifecycle.md) (Session-Scoped Architecture for vMCP) introduced session-hijack prevention with the requirement:

> "Store a secure cryptographic hash of the original authentication token in the session during creation. To prevent offline attacks if session state is leaked (e.g., from Redis/Valkey), prefer a keyed hash (e.g., `HMAC-SHA256` with a server-managed secret and a per-session salt). [...] On each request, validate the current auth token against the session's bound hash using a constant-time comparison to prevent timing attacks. If mismatch, reject with 'session authentication mismatch' error and terminate session."

[THV-0047](./THV-0047-vmcp-proxyrunner-horizontal-scaling.md) (Horizontal Scaling) operationalised this: the `HijackPreventionDecorator` persists `boundTokenHash` and `tokenSalt` in `MetadataKeyTokenHash` / `MetadataKeyTokenSalt` so the binding survives cross-pod restore.

Both RFCs are correct about the threat they defend — a stolen `Mcp-Session-Id` paired with the attacker's own token must not be accepted. Both are silent on one specific operation: **OAuth `refresh_token` exchange**. RFC 6749 §6 produces a new access token with different bytes for the same End-User. Under the HMAC-byte binding, every refresh fails the hijack check and the user is logged out — typically once per access-token TTL window (e.g. every 5 minutes for short-lived ATs).

The bug surfaces as:

```
HTTP 401 / JSON-RPC isError=true
  body:  "Unauthorized: caller identity does not match session owner"
  vMCP:  WARN  token validation failed: token hash mismatch  reason=token_hash_mismatch
```

The bug reproduces under the standard MCP flow: DCR → PKCE authorization → MCP `initialize` → `tools/list` (succeeds) → OAuth `refresh_token` exchange → `tools/call` with the refreshed access token and the same `Mcp-Session-Id` (fails).

The root cause is a conflation of two distinct invariants:

- **What changes legitimately during a session**: the access-token bytes, on each refresh.
- **What is stable for a given principal**: the `(iss, sub)` tuple from the OIDC token's claims.

The old binding pinned the first; it should have pinned the second.

## Goals

1. The vMCP session binding survives any legitimate OAuth `refresh_token` exchange for the original principal.
2. The binding continues to reject stolen `Mcp-Session-Id` reuse by a *different* identity (the original threat model).
3. The binding works under cross-pod restore (the THV-0047 invariant).
4. No per-deployment shared secret is required for the binding mechanism. Operator deploys become simpler.
5. Sessions written under the legacy schema invalidate cleanly at deploy with one forced re-auth per user (the standard MCP re-initialise signal), rather than entering a security-degraded migration window.

## Non-Goals

- **Per-request replay defence within a single identity.** Same-identity replay of a captured `Mcp-Session-Id` is not a property of the binding layer; it is the job of the token-validation layer above (signature, `exp`, `nbf`, audience).
- **Per-anonymous-user isolation.** `AnonymousMiddleware` emits identical claims for every request by construction; per-identity hijack prevention is impossible in that mode and is explicitly out of scope. The mode is documented as dev-only.
- **Encrypting bindings at rest in the session store.** The application-layer encryption secret has the same leak surface as the binding it would protect. Defence belongs at the store (Redis ACL, NetworkPolicy).
- **Audit-trail of binding mismatches.** Surfaced as structured `WARN` log lines for now; a formal audit event is left as a follow-up if a consumer needs it.

## Proposed Solution

### High-level design

The session metadata key `vmcp.token.hash` (the HMAC of bytes) is replaced by `vmcp.identity.binding` (a canonical encoding of `iss` and `sub`). The decorator, factory, and session manager migrate to the new key. Sessions carrying only the legacy key are treated as "not found" and re-initialise. The CLI's `VMCP_SESSION_HMAC_SECRET` reader is retained for one deploy cycle (logged at DEBUG and otherwise ignored) so operator-side env-var injection can be removed later without a coordinated cut-over.

### Detailed design

**Binding format.** The on-the-wire form is `iss + "\x00" + sub`, packaged as a small leaf API:

```go
const UnauthenticatedSentinel = "unauthenticated"

var ErrInvalidBinding = errors.New("invalid identity binding")

// Format returns iss + "\x00" + sub. Rejects empty halves and NUL injection.
func Format(iss, sub string) (string, error)

// Parse splits into (iss, sub). Returns ok=true only when s contains exactly
// one NUL and both halves are non-empty. Rejects the unauthenticated sentinel
// and any stray NUL beyond the first.
func Parse(s string) (iss, sub string, ok bool)

func IsUnauthenticated(s string) bool
```

The NUL separator is rejected from either input because no real OIDC issuer emits one, and accepting it would let a corrupted or adversarial value re-split during `Parse`. `Parse` mirrors `Format` strictly: any value `Format` would reject must also fail `Parse`.

**Identity extraction.** Both halves are read from `Identity.Claims["iss"]` and `Identity.Claims["sub"]` — from the same canonical layer (see [B7](#footnotes)). Extraction fails closed on nil identity, missing claim, non-string claim, or any pair `Format` rejects, and never returns a partial binding.

**Decorator and factory.** The bound identity is computed before any metadata write, so a session has either a valid binding or no metadata at all — never a stale value left over from a failed creation. Validation uses `subtle.ConstantTimeCompare`. All HMAC primitives (salt, token hashing, per-deploy secret) are deleted. The binding becomes the single source of identity at rest; on restore the factory parses it once and reconstructs `Identity.Subject`, `Claims["iss"]`, and `Claims["sub"]` so downstream code reading any of the three sees a consistent value.

Restore has three branches:

1. New binding key present → normal restore.
2. New key absent, legacy token-hash key present → return the bare `transportsession.ErrSessionNotFound` sentinel, triggering the standard MCP "session not found" client signal.
3. Neither key present → return a "corrupted metadata" error (distinct from "not found").

The *bare* sentinel return at branch 2 is load-bearing: the session manager matches it with `errors.Is`; a wrapped error would surface as HTTP 500 instead of a clean re-init.

**Session manager.** The Phase-2 marker (used to distinguish a fully-initialised session from a `Generate()`-only placeholder) switches from the legacy token-hash key to the new binding key; the placeholder-vs-full distinction is preserved.

**`ShouldAllowAnonymous` tightening.** The previous definition was `identity == nil || identity.Token == ""`. `LocalUserMiddleware` emits identities with `Token == ""` and a real `(iss, sub)` pair — Alice and Bob were therefore both treated as anonymous and could swap session-ids freely. The new definition treats any identity from which `Format(iss, sub)` succeeds as bound, even when `Token` is empty. If either claim is present but non-string the identity is treated as bound (not anonymous) with a `WARN` log.

**CLI.** `VMCP_SESSION_HMAC_SECRET` is read once at startup; if set, a `DEBUG` line records that the value is ignored. Operator-side env-var removal happens in a later cleanup. A `WARN` is emitted at startup when `IncomingAuth.Type == anonymous`: "per-identity session-hijack prevention is disabled in this mode; intended for development only."

### Comparison with THV-0038 (HMAC binding) and THV-0047 (horizontal scaling)

| Axis | THV-0038 / THV-0047 (HMAC of token bytes) | This RFC (`iss + sub`) |
|---|---|---|
| Threat: stolen session-id + different identity | Rejected | Rejected |
| Legitimate OAuth refresh | **Falsely rejected** | Accepted |
| Per-deploy shared secret | Required | Not required |
| Cross-pod restore | Hash + salt + same secret on all replicas | Persist binding; no secret coordination |
| At-rest representation | HMAC-SHA256 hash + salt | Plaintext `iss\x00sub` |
| Identifying a user from a Redis dump | Requires brute-forcing the HMAC secret | Immediate (PII leak) |

THV-0038's core threat reasoning is correct: an attacker with a stolen `Mcp-Session-Id` plus a different identity's token must be rejected. The HMAC was the wrong mechanism for that defence — it bound the access-token bytes, which change legitimately on every refresh. THV-0047's persistence design (write to Redis metadata, reapply decorator on restore) is preserved verbatim; only the value being persisted changes.

## Security Considerations

### Threat model — what the binding defends

The binding's purpose is to prevent **per-identity session-id replay**: a caller who obtains another user's `Mcp-Session-Id` (via logs, a malicious proxy, browser leak, etc.) must not be able to use it from their own authenticated session. The binding is a per-session invariant checked against a freshly-validated request token; it is not itself a credential and provides no freshness signal.

**Both old and new defend**: stolen `Mcp-Session-Id` + attacker's token for a *different* identity. Old: token-hash mismatch on HMAC compare. New: `(iss, sub)` mismatch on constant-time compare.

**New defends, old did not**: legitimate OAuth `refresh_token` exchange. RFC 6749 §6 produces a new access-token byte sequence for the same End-User; the old binding rejected it because the bytes differed. The new binding is invariant under refresh.

**Old defended, new does not**: nothing in the threat model. The HMAC-SHA256 of the token offered at-rest opacity in the session store, but it never defeated an actual attack — an in-memory attacker who had the token already could compute the hash, and the old scheme was non-functional on every legitimate refresh, so the opacity protected zero working sessions.

### Trade-off: HMAC → plaintext PII at rest

A reader of Redis/Valkey previously saw `hex(HMAC-SHA256(salt, token))`. Without the per-deployment HMAC secret, identifying the user required offline brute-force across the candidate principal set. Now a reader sees `iss\x00sub` plaintext and identifies the user immediately. The binding is **not** a credential — possession does not enable impersonation — but `(iss, sub)` is personally-identifying information.

This is a real downgrade in at-rest opacity, scoped to the session-store blast radius. It is acceptable because:

1. The old scheme rejected every legitimate refresh, so its opacity protected zero working sessions.
2. Operators control the session store, which is now documented as an identity-bearing system (`docs/arch/13-vmcp-scalability.md`).

Operator mitigation, documented in the arch doc:

- Redis ACL with a dedicated `vmcp` user and `requirepass` on the listener.
- Kubernetes `NetworkPolicy` restricting Redis ingress to vMCP pods.
- TLS for Redis transport when not on a confined network.
- Treat session-store dumps as an identity-data incident, not opaque-token loss.

### OAuth and OIDC correctness

The binding deliberately uses `(iss, sub)` and *only* `(iss, sub)`:

- **Refresh invariance**: OIDC Core §2 requires `sub` to be stable per End-User within an issuer; RFC 6749 refresh does not change the principal. Bindings survive refresh.
- **Global uniqueness**: `(iss, sub)` is globally unique under OIDC Core §2 — `sub` is only locally unique within `iss`, but the `iss\x00sub` concatenation lifts that to global.
- **Token exchange (RFC 8693)**: The binding is at the front-door token, not at any upstream-exchanged token. RFC 8693 §4.1 impersonation preserves `sub`; delegation adds `act` but `sub` still describes the principal the token is *about*. No interaction with the binding.
- **Why not `aud`**: step-up auth and audience-scoped tokens for different upstreams must not force a new vMCP session. `aud` validation belongs in the token-validation layer, one above the session boundary.
- **Why not `acr`/`amr`**: step-up enforcement is Cedar's job (authz layer), not the session layer.
- **Why not `act`**: describes who is using the token, not who it is about — wrong axis for session identity.

### AnonymousMiddleware scoping

`AnonymousMiddleware` emits identical `(iss="toolhive-local", sub="anonymous")` for every request. Consequently every anonymous user collapses into one binding equivalence class — per-identity hijack prevention is impossible *by construction* in anonymous deployments. This is not a regression: the old token-hash scheme produced the empty-string sentinel for the same case and exhibited identical behaviour. We surface the limitation at startup with a `WARN` and document the mode as dev-only. The session decorator still defends against the *upgrade attack* — a token presented to an anonymous session is rejected.

### LocalUserMiddleware: a prior gap closed

Before this change, `LocalUserMiddleware` set `Token == ""`, which made the old `ShouldAllowAnonymous` return true for every local user — Alice and Bob shared the anonymous equivalence class and could swap session-ids freely. The tightened `ShouldAllowAnonymous` now reads `Claims["iss"]` and `Claims["sub"]` (which `LocalUserMiddleware` populates with the username as `sub`), so Alice and Bob produce distinct bindings. This is a strict improvement in the local-user mode.

### Fail-closed defaults

- Identity extraction rejects nil identity, missing `iss`/`sub`, non-string `iss`/`sub`, and any pair that fails to format.
- The anonymous-check returns "bound" (not anonymous) for present-but-non-string claims and logs `WARN` for ops visibility.
- `Format` rejects NUL injection in either half; `Parse` rejects trailing NULs as defence-in-depth.
- Session creation computes the binding before any metadata write, so a failed extraction leaves no partial state.
- Restore rejects any stored value that is neither the unauthenticated sentinel nor parseable.

### Timing side channel

The caller check uses `subtle.ConstantTimeCompare`, which is constant-time over content but short-circuits on length mismatch. Leaking binding length is acceptable: `iss` is the OIDC issuer URL (public, in the discovery document), and `sub` has per-issuer-fixed length for almost every IdP (UUIDs, Entra GUIDs, Okta opaque strings, Google numeric). Neither component is secret.

### Forward-looking: RFC 7662 introspection

If a future incoming-auth type uses RFC 7662 token introspection (it currently doesn't — the `IncomingAuthConfig` validator allows `oidc`, `local`, `anonymous` only), identity extraction requires the introspection response to contain `iss` and `sub`. RFC 7662 §2.2 marks both OPTIONAL, so a misconfigured IdP would silently break the binding. A startup probe that fails fast on incompatible IdPs is recommended when that mode is added.

### Explicit non-goals (so they are not mistaken for regressions)

- **Cross-anonymous-user hijack**: out of scope. AnonymousMiddleware is dev-only.
- **Cross-tenant hijack**: defended (`iss` differs).
- **Replay of an `Mcp-Session-Id` captured from the *same identity***: not defended at this layer. The binding is a per-identity invariant, not a per-request nonce. Same-identity replay is the job of the token-validation layer above the session boundary.
- **Encrypted-at-rest bindings**: explicitly rejected (see Trade-off).
- **Audit-trail of binding mismatches**: present as `WARN` log lines with structured `reason` fields; a formal audit event is left for a follow-up if needed.

## Alternatives Considered

**Bind to `tsid` (the embedded-AS internal session-id claim).** Survives refresh, but `tsid` is intentionally filtered out of `*auth.Identity.Claims` to keep it off the webhook attack surface, and external IdPs (Keycloak, Entra, Okta, Auth0) don't emit it — so any production deployment would fall back to `(iss, sub)` anyway. The extra granularity `tsid` would buy is same-user-vs-self, which is not in the threat model.

**Bind to `(iss, sub, aud)`.** Over-segments legitimate flows: step-up auth and audience-scoped tokens for different upstreams in the same vMCP session would each force a restart. `aud` validation already lives in the token-validation layer, one above the session boundary.

**Encrypted-at-rest binding.** An application-layer encryption secret colocated with vMCP pods has the same leak surface as the binding it protects. Layered defence belongs at the store (ACLs, NetworkPolicy), not at the application — this is the same class of misplaced complexity as the original HMAC-secret approach.

**Hybrid migration ("read both, write new").** Re-creates the exact attack the binding defends against: an attacker presenting a stolen legacy session-id plus their own bearer token would cause the session to adopt the *attacker's* identity on the rebind. Invalidate-on-read fails closed at the cost of one re-auth at deploy.

**Keep the HMAC scheme, hash `(iss, sub)` instead of token bytes.** Survives refresh and preserves at-rest opacity, but the HMAC adds operational complexity (per-deploy secret, cross-replica coordination) for a property that is *not in the threat model* — the binding is not a credential, so identifying the user from a Redis dump does not enable impersonation. Removing the HMAC produces simpler, more honest code and reduces operator-side complexity (Goal 4).

## Compatibility

### Backward compatibility

- Sessions under the legacy schema are invalidated on first read after deploy (see [Migration](#migration)). The user-visible cost is one forced re-auth per active session at the time of deploy.
- `VMCP_SESSION_HMAC_SECRET` is still read and ignored with a DEBUG log; pods with the variable set will not fail to start. Operator-side env-var injection is removed in the cleanup phase.
- No public API change: the affected packages are all internal.
- Legacy metadata keys remain defined for one release cycle so the migration's legacy-detection branch can read them; they are removed in the cleanup phase.

### Forward compatibility

- RFC 7662 introspection-based incoming auth would require the IdP to emit `iss` and `sub` in introspection responses. A startup probe that fails fast on incompatible IdPs is recommended when that mode is added — it's not a hard precondition because the failure mode is loud (extraction errors on every request) and surfaces immediately in operator logs.
- `Format` and `Parse` are total functions over their inputs; the format is fixed. Future extensions (multiple sub-claims, federated identifiers) would land as new sentinel values or a new metadata key, not changes to the existing format.

## Implementation Plan

The change rolls out in two phases:

**Phase 1 — the binding swap.**

- New leaf package owning the binding format.
- New metadata key for the identity binding; tightened anonymous-mode check.
- Decorator and factory refactor; the session manager's Phase-2 marker switches to the new key.
- CLI: drop HMAC config, add anonymous-mode WARN.
- Arch doc updates and a reproducer harness for the bug.
- Unit + integration test catch-up.

**Phase 2 — cleanup (deletion-only, after legacy sessions have expired via the Redis TTL).**

- Delete the operator-side HMAC machinery (Secret creation, controller helpers, env-var injection).
- Delete the HMAC env-var reader and the legacy metadata-key constants from the binding code.

## Migration

**Invalidate-on-read.** At deploy, sessions in Redis carry the legacy keys but not `MetadataKeyIdentityBinding`. The next request that hits the session manager triggers `RestoreSession`, which detects the absence of the new key plus the presence of the legacy key and returns the bare `transportsession.ErrSessionNotFound` sentinel. The MCP client receives the standard re-initialise signal and starts a fresh session. A `WARN` log line with `reason=legacy_session_missing_identity_binding` is emitted once per legacy session encountered.

**Why not "read both, write new":** see [Alternative 4](#alternative-4-hybrid-migration-read-both-write-new).

**Operator-visible deploy procedure:**

1. Deploy the new vMCP image. No coordination with operator-side env-var changes required — the env-var is silently ignored.
2. Active sessions re-init once. Browser-based MCP clients will see one OAuth flow per active session; CLI clients see their next request return a "session not found" and re-initialise transparently.
3. After the Redis session TTL has elapsed across the cluster (default 30 minutes), no legacy-format sessions remain. The cleanup phase can land at any time after this point.

**Rolling-deploy note:** during a rolling restart, replicas of the old and new image coexist briefly. A request hitting an old replica creates a legacy-format session; a follow-up request to a new replica invalidates it. Net effect: one extra re-init per straddled session. Documented in the arch doc; recommend session affinity at the load balancer during the rollout.

## Testing Strategy

**Unit tests:**

- **Binding format:** `Format`/`Parse` round-trip, NUL rejection on both sides (including trailing-NUL defence-in-depth), sentinel discrimination, byte-identical output across JWT and introspection claim shapes.
- **Decorator:** the refresh-token regression for #5306 (primary acceptance test), cross-identity rejection, fail-closed behaviour on missing / non-string / NUL-injected `iss`/`sub` at both creation and caller paths, session-upgrade attack, nil-caller, and a concurrent-refresh race with 20 goroutines sharing `(iss, sub)` and distinct tokens.
- **Restore:** bound round-trip, unauthenticated round-trip, corrupted-binding rejection, empty-string and nil-session rejection.
- **Factory:** five identity shapes (OIDC, nil, LocalUser, anonymous, generated-only), legacy-token-hash returns bare `ErrSessionNotFound`, absent metadata returns corrupted-metadata error, restored session populates `Subject` and both claims consistently.
- **`ShouldAllowAnonymous`:** all identity shapes including the non-string-claim fail-closed path.
- **Session manager:** Phase-2 marker swap, including a documentation test that locks the legacy-session `Terminate` placeholder-path behaviour.

**Integration tests:**

- Cross-replica restore continues to work without HMAC coordination; the binding survives a simulated pod restart.
- **Rolling-deploy race:** write a session via the legacy code path (or fabricate legacy metadata), then read via the new factory. Asserts the bare `ErrSessionNotFound` sentinel is returned and an MCP re-init follows.
- Three HMAC-only tests (cross-replica secret mismatch, etc.) are deleted as they no longer apply.

**End-to-end (manual):**

- Run a kind-cluster end-to-end test that performs DCR → PKCE → MCP `initialize` → `tools/list` → `refresh_token` → `tools/call` with the refreshed access token reusing the original `Mcp-Session-Id`. Expected: the post-refresh `tools/call` returns HTTP 200; logs show no identity-binding mismatch on the legitimate refresh path.
- **Negative test:** tamper with the `Authorization` header to a different user's token between the `refresh_token` and the post-refresh `tools/call`; confirms a true cross-identity hijack is still rejected.
- **`LocalUserMiddleware` regression:** Alice creates a session, Bob attempts to use Alice's `Mcp-Session-Id` with his own credentials. Old code allowed it (shared empty-binding equivalence class); new code rejects.
- **`AnonymousMiddleware` smoke:** vMCP starts under anonymous mode and emits the documented startup `WARN` exactly once.

## References

- [toolhive#5306](https://github.com/stacklok/toolhive/issues/5306) — the issue this RFC addresses.
- [THV-0038](./THV-0038-session-scoped-client-lifecycle.md) — original session-scoped architecture RFC; the session-binding portion is superseded by this one.
- [THV-0047](./THV-0047-vmcp-proxyrunner-horizontal-scaling.md) — horizontal scaling design; the binding-persistence mechanism (RC-15) is preserved with the new key.
- [RFC 6749 §6](https://datatracker.ietf.org/doc/html/rfc6749#section-6) — OAuth 2.0 refresh-token grant.
- [RFC 7519 §4.1.2](https://datatracker.ietf.org/doc/html/rfc7519#section-4.1.2) — JWT `sub` claim.
- [OpenID Connect Core §2](https://openid.net/specs/openid-connect-core-1_0.html#IDToken) — `iss` and `sub` requirements.
- [RFC 8693](https://datatracker.ietf.org/doc/html/rfc8693) — OAuth 2.0 Token Exchange (impersonation vs delegation).
- [RFC 7662 §2.2](https://datatracker.ietf.org/doc/html/rfc7662#section-2.2) — Token Introspection response (`iss` and `sub` are OPTIONAL).

## Footnotes

**B7**: the choice to read both `iss` and `sub` from `Claims` (rather than reading `sub` from `Identity.Subject` and `iss` from `Claims["iss"]`) defends against a subtle canonicalisation mismatch: `Identity.Subject` is populated by the OIDC validator from `claims["sub"]`, while `Claims["iss"]` is populated separately. If the JWT path and the RFC 7662 introspection path (or any future validator) canonicalised `sub` differently — different whitespace handling, different unicode normalisation — a refresh through one path against an original through another would produce mismatched bindings and re-introduce the bug. Reading both from the same canonical layer eliminates this risk by construction.

## Review History

| Date | Reviewer | Decision | Notes |
|---|---|---|---|
| | | | |

## Implementation Tracking

Status is tracked via the linked issue ([toolhive#5306](https://github.com/stacklok/toolhive/issues/5306)).
