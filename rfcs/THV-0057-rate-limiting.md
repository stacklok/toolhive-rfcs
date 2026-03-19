# RFC-0057: Rate Limiting for MCP Servers

- **Status**: Draft
- **Author(s)**: Jeremy Drouillard (@jerm-dro)
- **Created**: 2026-03-18
- **Last Updated**: 2026-03-19
- **Target Repository**: toolhive
- **Related Issues**: None

## Summary

Enable rate limiting for `MCPServer` and `VirtualMCPServer`, supporting per-user and global limits at both the server and individual operation level. Rate limits are configured declaratively on the resource spec and enforced by a new middleware in the middleware chain, with Redis as the shared counter backend.

## Problem Statement

ToolHive currently has no mechanism to limit the rate of requests flowing through its proxy layer. This creates several risks for administrators:

1. **Noisy-neighbor problem**: A single authenticated user can consume unbounded resources, degrading the experience for all other users of a shared MCP server.
2. **Downstream overload**: Aggregate traffic spikes — even when no single user is misbehaving — can overwhelm the upstream MCP server or the external services it depends on (APIs with their own rate limits, databases, etc.).
3. **Agent data exfiltration**: AI agents can invoke tools in tight loops to export large volumes of data. Without per-tool or per-user limits, there is no mechanism to cap this behavior.

These risks grow as ToolHive deployments move to shared, multi-user Kubernetes environments. Without rate limiting, cluster administrators have no knob to turn between "fully open" and "take the server offline."

> **Scope**: This RFC targets Kubernetes-based deployments of ToolHive.

## Goals

- Allow administrators to configure **per-user** rate limits so that no single user can monopolize a server.
- Allow administrators to configure **global** rate limits so that total throughput stays within safe bounds for downstream services.
- Allow administrators to configure rate limits **per tool**, **per prompt**, or **per resource**, so that expensive or externally-constrained operations can have tighter limits than the server default.
- Provide a consistent configuration experience across `MCPServer` and `VirtualMCPServer` resources.
- Enforce rate limits in the existing proxy middleware chain with minimal latency overhead.
- Support correct enforcement across multiple replicas using Redis as the shared counter backend.

## Non-Goals

- **Adaptive / auto-tuning rate limits**: Automatically adjusting limits based on observed load or downstream health signals. Limits are static and administrator-configured.
- **Cost or billing integration**: Tracking usage for billing purposes. This is purely a protective mechanism.
- **Request queuing or throttling**: Requests that exceed the limit are rejected, not queued.
- **Rate limiting at the vMCP routing layer**: Limits are applied at the individual server proxy, not at the vMCP aggregation/routing level.

## Proposed Solution

### High-Level Design

Rate limiting is implemented as a new middleware in ToolHive's middleware chain. When a request arrives, the middleware checks the applicable limits (global, per-user, per-operation) and either allows the request to proceed or returns an appropriate error response.

```mermaid
flowchart LR
    Client -->|request| Auth
    Auth -->|identified user| RateLimit
    RateLimit -->|allowed| Downstream[Remaining Middleware + MCP Server]
    RateLimit -->|rejected| Error[Error Response]
```

The rate limit middleware sits after authentication (so user identity is available) and before the rest of the middleware chain.

Rate limit counters are stored in Redis, which ToolHive already depends on. This ensures accurate enforcement across multiple replicas in horizontally-scaled deployments.

### Configuration

Rate limits are configured via a `rateLimiting` field on the server spec. The same structure applies to both `MCPServer` and `VirtualMCPServer`.

#### Server-Level Limits

```yaml
apiVersion: toolhive.stacklok.dev/v1alpha1
kind: MCPServer
metadata:
  name: my-server
spec:
  # ... existing fields ...
  rateLimiting:
    # Global limit: total requests across all users
    global:
      requestsPerWindow: 1000
      windowSeconds: 60

    # Per-user limit: applied independently to each authenticated user
    perUser:
      requestsPerWindow: 100
      windowSeconds: 60
```

**Validation**: Per-user rate limits require authentication to be enabled. If `perUser` limits are configured with anonymous inbound auth, the server will raise an error at startup.

#### Per-Operation Limits

Individual tools, prompts, or resources can have their own limits that supplement the server-level defaults. Per-operation limits can be either global or per-user:

```yaml
spec:
  rateLimiting:
    perUser:
      requestsPerWindow: 100
      windowSeconds: 60

    tools:
      - name: "expensive_search"
        perUser:
          requestsPerWindow: 10
          windowSeconds: 60
      - name: "shared_resource"
        global:
          requestsPerWindow: 50
          windowSeconds: 60

    prompts:
      - name: "generate_report"
        perUser:
          requestsPerWindow: 5
          windowSeconds: 60

    resources:
      - name: "large_dataset"
        global:
          requestsPerWindow: 20
          windowSeconds: 60
```

When an operation-level limit is defined, it is enforced **in addition to** any server-level limits. A request must pass all applicable limits.

The `tools`, `prompts`, and `resources` lists all follow the same structure — each entry specifies an operation name and either a `global` or `perUser` limit (or both).

#### VirtualMCPServer

The same `rateLimiting` configuration is available on `VirtualMCPServer`. Limits configured here apply to the proxied traffic for each backend server independently.

```yaml
apiVersion: toolhive.stacklok.dev/v1alpha1
kind: VirtualMCPServer
metadata:
  name: my-vmcp
spec:
  config:
    rateLimiting:
      perUser:
        requestsPerWindow: 200
        windowSeconds: 60
      tools:
        - name: "backend_a/costly_tool"
          perUser:
            requestsPerWindow: 5
            windowSeconds: 60
      prompts:
        - name: "backend_b/heavy_prompt"
          global:
            requestsPerWindow: 30
            windowSeconds: 60
```

### Algorithm: Token Bucket

Rate limits are enforced using a **token bucket** algorithm. The configuration maps directly onto it:

- `requestsPerWindow` = bucket capacity (max tokens)
- `windowSeconds` = time to fully refill the bucket
- Refill rate = `requestsPerWindow / windowSeconds` tokens per second

Each token represents a single allowed request. The bucket starts full, refills at a steady rate, and each incoming request consumes one token. When the bucket is empty, requests are rejected.

Per-user limits work identically — each user gets their own bucket, keyed by identity. Redis keys follow a structure like:

- Global: `thv:rl:{server}:global`
- Per-user: `thv:rl:{server}:user:{userId}`
- Per-user per-tool: `thv:rl:{server}:user:{userId}:tool:{toolName}`

Each bucket is stored as a Redis hash with two fields: token count and last refill timestamp. The check-and-decrement runs as a single atomic operation, so there are no race conditions across replicas. Keys auto-expire when inactive, so no garbage collection is needed.

Storage is **O(1) per counter** (two fields per hash). For example, 500 users with 10 per-operation limits = 5,000 hashes — negligible for Redis.

**Burst behavior**: An idle user accumulates tokens up to the bucket capacity. This means they can send a full burst of `requestsPerWindow` requests at once after a period of inactivity. This is intentional — it handles legitimate traffic spikes — but administrators should understand it when setting capacity.

### Rejection Behavior

When a request is rate-limited, the middleware returns an MCP-appropriate error response. Because token bucket tracks the refill rate, the middleware can compute a precise `Retry-After` value (`1 / refill_rate`) telling the client exactly when the next token will be available.

## Open Questions

1. **Redis unavailability**: How should the middleware behave if Redis is unreachable? Fail open (allow all requests) or fail closed (reject all requests)?

## References

- [THV-0017: Dynamic Webhook Middleware](./THV-0017-dynamic-webhook-middleware.md) — mentions rate limiting as an external webhook use case
- [THV-0047: Horizontal Scaling for vMCP and Proxy Runner](./THV-0047-vmcp-proxyrunner-horizontal-scaling.md) — relevant to distributed rate limiting concerns
- [THV-0035: Auth Server Redis Storage](./THV-0035-auth-server-redis-storage.md) — existing Redis dependency
- [IETF RFC 6585](https://tools.ietf.org/html/rfc6585) — HTTP 429 Too Many Requests status code

---

## RFC Lifecycle

<!-- This section is maintained by RFC reviewers -->

### Review History

| Date | Reviewer | Decision | Notes |
|------|----------|----------|-------|
| 2026-03-18 | @jerm-dro | Draft | Initial submission |

### Implementation Tracking

| Repository | PR | Status |
|------------|-----|--------|
| toolhive | TBD | Not started |
