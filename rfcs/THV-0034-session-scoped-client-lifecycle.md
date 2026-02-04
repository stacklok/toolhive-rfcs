# RFC: Session-Scoped MCP Client Lifecycle Management

- **Status**: Draft
- **Author(s)**: @yrobla, @jerm-dro
- **Created**: 2026-02-04
- **Last Updated**: 2026-02-04
- **Target Repository**: `toolhive`
- **Related Issues**: #3062

## Summary

Move MCP backend client lifecycle from per-request creation/destruction to session-scoped management. Clients will be created once during session initialization, reused throughout the session lifetime, and closed during session cleanup. This simplifies the client pooling architecture and ensures consistent backend state preservation.

## Problem Statement

### Current Limitations

The current `httpBackendClient` implementation creates and closes MCP clients on every request. Each method (`CallTool`, `ReadResource`, `GetPrompt`, `ListCapabilities`) follows the same pattern:

1. Create a new client via `clientFactory()`
2. Defer client closure with `c.Close()`
3. Initialize the client with MCP handshake
4. Perform the requested operation
5. Close the client when the function returns

**Problems with Per-Request Client Lifecycle**:

1. **Connection Overhead**: Every tool call incurs TCP handshake, TLS negotiation, and MCP protocol initialization overhead.

2. **State Loss**: MCP backends that maintain stateful contexts (Playwright browser sessions, database transactions, conversation history) lose state between requests. While sessions preserve routing information, the underlying connections are recreated each time.

3. **Redundant Initialization**: Capability discovery during session creation (`AfterInitialize` hook) establishes which backends exist, but clients are created fresh for every subsequent request.

4. **Authentication Overhead**: Each client creation re-resolves authentication strategies and validates credentials, even though this information doesn't change within a session.

5. **Resource Waste**: Creating and destroying clients repeatedly wastes CPU cycles and network bandwidth, especially problematic for high-throughput scenarios.

### Affected Parties

- **vMCP Server Performance**: Every tool call creates/closes connections, multiplying latency by the number of requests
- **Backend Servers**: Repeated connection churn increases load on backend MCP servers
- **Stateful Backends**: Backends relying on persistent state (Playwright, databases) lose context between calls
- **High-Throughput Scenarios**: Connection overhead becomes a bottleneck for applications making many tool calls

### Why This Matters

MCP backends often maintain stateful contexts (browser sessions, database connections, conversation history). The current per-request pattern means:
- A browser automation workflow must navigate to the same page repeatedly
- Database transactions cannot span multiple tool calls
- Conversation context in AI assistants is lost between interactions

Sessions already exist and have defined lifetimes (TTL-based expiration). Aligning client lifecycle with session lifecycle is a natural fit that simplifies the architecture while enabling better state preservation.

## Goals & Non-Goals

### Goals

1. **Align Lifecycle with Sessions**: Move client lifecycle from per-request to session-scoped
2. **Eager Initialization**: Create all backend clients during session initialization alongside capability discovery
3. **Enable Stateful Workflows**: Allow backends to maintain state (browser contexts, DB transactions) across multiple tool calls within a session
4. **Reduce Overhead**: Eliminate redundant connection creation, TLS negotiation, and MCP initialization
5. **Simplify Code**: Remove repeated client creation/closure boilerplate from every method

### Non-Goals

- **Connection Pooling Within Clients**: Individual MCP clients may internally pool connections, but that's outside this RFC's scope
- **Multi-Session Client Sharing**: Clients remain session-scoped and are not shared across sessions
- **Lazy Backend Discovery**: Backend discovery remains eager (current behavior)
- **Client Versioning**: Handling MCP protocol version negotiation is out of scope

## Proposed Solution

### High-Level Design

```
┌─────────────────────────────────────────────────────────────┐
│ Session Manager                                             │
│                                                             │
│  ┌────────────────────────────────────────────────────┐   │
│  │ VMCPSession (TTL-scoped)                           │   │
│  │                                                     │   │
│  │  - Session ID                                      │   │
│  │  - Routing Table                                   │   │
│  │  - Tools List                                      │   │
│  │  - Backend Clients Map[WorkloadID]*client.Client ◄├───┼─── Created during AfterInitialize
│  │                                                     │   │
│  │  Lifecycle:                                        │   │
│  │    1. Create session (Generate)                   │   │
│  │    2. Initialize clients (AfterInitialize)        │   │
│  │    3. Use clients (CallTool/ReadResource/...)     │   │
│  │    4. Close clients (TTL expiry/Delete/Stop)      │   │
│  └────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
              ┌────────────────────────┐
              │ Backend Client         │
              │ (mcp-go/client.Client) │
              │                        │
              │ - Initialized once     │
              │ - Reused per session   │
              │ - Closed with session  │
              └────────────────────────┘
```

**Key Changes**:
- Add `backendClients` map to `VMCPSession` for storing initialized clients
- Modify `AfterInitialize` hook to create and initialize clients alongside capability discovery
- Change `httpBackendClient` methods to retrieve clients from session instead of creating them
- Add `Close()` method to `VMCPSession` to close all clients on session expiration
- Remove per-request `defer c.Close()` pattern from all client methods

### Detailed Design

#### 1. VMCPSession Structure Changes

**Add Backend Clients Map**:
- Add `backendClients map[string]*client.Client` field to `VMCPSession` struct
- This map stores initialized MCP clients keyed by workload ID
- Protected by existing mutex for thread-safe access

**New Methods**:
- `GetBackendClient(workloadID string)`: Retrieves an initialized client for the given backend
- `SetBackendClients(clients map[string]*client.Client)`: Stores the client map during session initialization
- `Close()`: Iterates through all clients and closes them, returning any errors encountered

**Location**: `pkg/vmcp/session/vmcp_session.go`

#### 2. Client Initialization During Session Setup

**Extend AfterInitialize Hook**:

The `AfterInitialize` hook in the discovery middleware already performs capability discovery for each backend. Extend this to also create and initialize MCP clients:

1. After capability discovery completes, iterate through all backends in the group
2. For each backend, create a client using the backend target configuration
3. Initialize the client immediately (MCP handshake)
4. Store successful clients in a map keyed by workload ID
5. For failed initializations: log warning and continue (partial initialization is acceptable)
6. Store the complete client map in the session via `SetBackendClients()`

**Error Handling**:
- Individual client initialization failures don't fail session creation
- Failed backends are marked unhealthy via existing health monitoring
- Subsequent requests to failed backends return "no client" errors

**Location**: `pkg/vmcp/server/middleware/discovery/discovery.go`

#### 3. Refactored BackendClient Implementation

**Add Session Manager Reference**:
- Add `sessionManager` field to `httpBackendClient` struct to enable session lookup
- Pass session manager to constructor during server initialization

**New Helper Methods**:
- `getSessionClient(workloadID)`: Retrieves pre-initialized client from session
  - Extracts session ID from request context
  - Looks up VMCPSession from manager
  - Returns the client for the requested backend
- `createAndInitializeClient(target)`: Creates and initializes a client
  - Uses existing `clientFactory` to create the client
  - Immediately initializes with MCP handshake
  - Returns initialized client ready for use
  - Called only during session setup, not per-request

**Refactor Request Methods**:
- `CallTool`, `ReadResource`, `GetPrompt`: Replace client creation pattern with:
  - Call `getSessionClient()` to retrieve pre-initialized client
  - Remove `defer c.Close()` (client managed by session)
  - Remove `initializeClient()` call (already initialized)
  - Use client directly for the operation
- `ListCapabilities`: Keep existing per-request pattern (only called during session init)

**Location**: `pkg/vmcp/client/client.go`

#### 4. Session Cleanup Integration

The transport layer's `LocalStorage` implementation already calls `Close()` on sessions before deletion (implemented in a previous PR). We leverage this existing mechanism to close backend clients.

**VMCPSession.Close() Implementation**:
- Iterate through all clients in the `backendClients` map
- Call `Close()` on each client
- Collect any errors encountered
- Return combined errors if any failures occurred

**Cleanup Triggers**:

The existing session cleanup infrastructure ensures clients are closed when:
- **TTL Expiration**: Session manager's cleanup worker calls `DeleteExpired()`, which calls `Close()` on expired sessions
- **Explicit Deletion**: Manual session deletion via `Delete()` calls `Close()` before removing the session
- **Manager Shutdown**: `Stop()` calls `Close()` on all remaining sessions

No changes needed to the transport layer - it already has the hooks in place.

#### 5. Error Handling

**Connection Failures During Session Creation**:
- Log warning and skip that backend
- Backend marked unhealthy via existing health monitor
- Session creation succeeds with partial backend connectivity

**Connection Failures During Tool Calls**:
- Return error to client (existing behavior)
- Health monitor marks backend unhealthy
- Subsequent sessions won't attempt to initialize unhealthy backends

**Client Already Closed**:
- If session expired and client is closed, return clear error
- Client should refresh session via `/initialize` endpoint

## Security

### Threat Model

No new security boundaries are introduced. This is a refactoring of existing client lifecycle management.

### Authentication & Authorization

**No Changes**: Authentication flow remains identical:
1. Client → vMCP: Validate incoming token (existing)
2. vMCP → Backend: Use outgoing auth configured per backend (existing)
3. Backend → External APIs: API-level validation (existing)

The only difference is **when** clients are created (session init vs first use), not **how** they authenticate.

### Data Protection

**Session Isolation**: Each session has its own client map. No cross-session data leakage risk.

**Connection Security**: TLS configuration and certificate validation remain unchanged.

### Secrets Management

**No Changes**: Outgoing auth secrets are still retrieved via `OutgoingAuthRegistry` during client creation. The timing changes (session init vs first request) but the mechanism is identical.

### Audit Logging

**New Log Event**: Add audit log entry for client initialization during session setup:
```json
{
  "event": "backend_client_initialized",
  "session_id": "sess-123",
  "workload_id": "github-mcp",
  "timestamp": "2026-02-04T10:30:00Z"
}
```

**Existing Events Unchanged**: Tool call logs, authentication logs, and session lifecycle logs remain the same.

## Alternatives Considered

### Alternative 1: Keep Pooling but Decouple from httpBackendClient

**Approach**: Make `pooledBackendClient` accept an interface instead of embedding `*httpBackendClient`.

**Pros**:
- Less refactoring required
- Maintains lazy initialization pattern

**Cons**:
- Still has pooling complexity
- Doesn't address first-call latency
- Doesn't eliminate redundant initialization logic

**Decision**: Rejected. Decoupling addresses tight coupling but not the fundamental architectural issue.

### Alternative 2: Lazy Pool Initialization with Eager Client Creation

**Approach**: Keep pool but create all clients immediately when pool is first accessed.

**Pros**:
- Minimal code changes
- Backward compatible with existing pool interface

**Cons**:
- Pool abstraction still adds unnecessary indirection
- First tool call per session still incurs initialization cost
- Doesn't simplify architecture

**Decision**: Rejected. This is a middle ground that retains most of the complexity.

### Alternative 3: Global Client Pool Across Sessions

**Approach**: Share clients across sessions to amortize initialization cost.

**Pros**:
- Maximum connection reuse
- Lowest per-session overhead

**Cons**:
- **State Pollution**: Backend state (Playwright contexts, DB transactions) would leak across sessions
- **Security Risk**: Potential for cross-session data exposure
- **Complexity**: Requires complex client reset/sanitization logic

**Decision**: Rejected. Session isolation is a core requirement for vMCP.

## Backward Compatibility

### Breaking Changes

**None for External APIs**: The `/vmcp/v1/*` HTTP API remains unchanged. Clients see identical behavior.

**Internal API Changes**:
- `BackendClient` interface: No signature changes, but behavior changes from "create on demand" to "retrieve from session"
- Components depending on `pooledBackendClient` or `BackendClientPool` types will need updates

### Forward Compatibility

Future enhancements are easier with this design:

- **Connection Warming**: Pre-connect to backends during idle time
- **Health-Based Initialization**: Skip unhealthy backends during session setup
- **Client Version Negotiation**: Perform protocol version negotiation once per session

## Implementation

### Phase 1: Session Client Storage

Add client storage capabilities to VMCPSession:

- Add `backendClients map[string]*client.Client` field to `VMCPSession` struct
- Implement `GetBackendClient(workloadID string)` method for retrieving clients
- Implement `SetBackendClients(clients map[string]*client.Client)` for storing client map
- Implement `Close()` method to iterate and close all clients

**Files to modify**:
- `pkg/vmcp/session/vmcp_session.go`

**Testing**:
- Unit tests for client storage/retrieval
- Unit tests for Close() error handling

### Phase 2: Client Initialization at Session Creation

Extend the discovery middleware to initialize clients during session setup:

- Modify `AfterInitialize` hook in discovery middleware
- After capability discovery, create clients for each backend
- Initialize each client immediately (MCP handshake)
- Store successful clients in session via `SetBackendClients()`
- Log warnings for failed initializations but continue (partial initialization acceptable)

**Files to modify**:
- `pkg/vmcp/server/middleware/discovery/discovery.go`

**Testing**:
- Integration tests verifying clients created during session init
- Tests for partial backend initialization (some backends fail)
- Tests confirming failed backends don't block session creation

### Phase 3: Refactor Backend Client Methods

Modify httpBackendClient to retrieve clients from session instead of creating per-request:

- Add `sessionManager` field to `httpBackendClient` struct
- Pass session manager to `NewHTTPBackendClient()` constructor
- Create `getSessionClient(ctx, workloadID)` helper method
- Create `createAndInitializeClient(ctx, target)` method for session init use
- Refactor `CallTool` to use `getSessionClient()` instead of `clientFactory()`
- Refactor `ReadResource` to use `getSessionClient()`
- Refactor `GetPrompt` to use `getSessionClient()`
- Remove `defer c.Close()` calls from all methods
- Remove `initializeClient()` calls from all methods (except `ListCapabilities`)

**Files to modify**:
- `pkg/vmcp/client/client.go`

**Testing**:
- Integration tests confirming clients reused across multiple tool calls
- Tests verifying no client leaks (clients closed on session expiration)
- Tests for "no client found" errors when backend unavailable

### Phase 4: Observability & Monitoring

Add observability for client lifecycle:

- Add audit log event for client initialization during session setup
- Add metrics for client creation success/failure per backend
- Add metrics for client lifetime (session duration)
- Add traces spanning session init → client creation → first tool call
- Consider health-based filtering: skip unhealthy backends during initialization

**Files to create/modify**:
- Add logging in discovery middleware
- Add metrics in client initialization path

**Testing**:
- Verify audit logs contain session ID and backend ID
- Verify metrics show client init success/failure rates
- Verify traces show reduced latency for subsequent tool calls

### Dependencies

- Existing transport session cleanup infrastructure (sessions already call `Close()` on expiration)
- Existing discovery middleware (`AfterInitialize` hook)
- Existing health monitoring system

### Testing Strategy

**Unit Tests**:
- Session client map operations (get, set, iterate)
- Client closure on session cleanup with error handling
- Error handling for missing/uninitialized clients

**Integration Tests**:
- Session creation initializes clients for all configured backends
- Multiple tool calls within same session reuse clients
- Session expiration triggers client closure
- Partial initialization doesn't fail session creation
- Failed backend initialization marked in health system

**End-to-End Tests**:
- Full vMCP workflow with multiple backends
- Backend state preservation across tool calls (e.g., Playwright browser session)
- Client resource cleanup on TTL expiration
- High-throughput scenarios verify no connection leaks

## Review History

| Date | Reviewer | Status | Comments |
|------|----------|--------|----------|
| 2026-02-04 | @yrobla | Author | Initial draft |
| 2026-02-04 | @jerm-dro | Contributor | Original proposal |

## Implementation Tracking

| Repository | PR | Status | Notes |
|------------|-------|--------|-------|
| toolhive | #TBD | Not Started | Phase 1: Session client storage |
| toolhive | #TBD | Not Started | Phase 2: Client initialization at session creation |
| toolhive | #TBD | Not Started | Phase 3: Refactor backend client methods |
| toolhive | #TBD | Not Started | Phase 4: Observability & monitoring |
