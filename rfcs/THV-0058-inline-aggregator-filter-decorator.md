# RFC-0058: Inline aggregator into session factory and extract advertising filter as session decorator

- **Status**: Draft
- **Author(s)**: Jeremy Drouillard (@jerm-dro)
- **Created**: 2026-03-19
- **Last Updated**: 2026-03-19
- **Target Repository**: toolhive
- **Related Issues**: [toolhive#4287](https://github.com/stacklok/toolhive/issues/4287)

## Summary

Decompose the monolithic `Aggregator` interface by inlining its pipeline (per-backend overrides, conflict resolution, routing table construction) into the session factory, and extracting the advertising filter into a session decorator. This fixes a bug where composite tool workflows fail type coercion for non-advertised backend tools, and simplifies the vMCP architecture by removing a layer that sits awkwardly between middleware and session.

## Problem Statement

### Bug: composite tool type coercion fails for non-advertised tools

When backend tools are excluded from advertising via `excludeAll` or `filter`, composite tool workflows fail. The workflow engine's `getToolInputSchema()` searches the session's tool list for `InputSchema` to coerce string arguments to their schema types. Non-advertised tools are absent from this list, so coercion is skipped and backends reject string-typed arguments (e.g. `pullNumber` as `"42"` instead of `float64(42)`).

The root cause: the advertising filter runs inside the aggregator **before** session creation. The session's `tools` field only contains advertised tools. The composite tools decorator builds its workflow engine from `sess.Tools()`, so it never sees schemas for non-advertised tools.

### Architectural issue: the aggregator is a monolithic middleman

The `Aggregator` interface has 7 responsibilities: backend capability querying, per-backend overrides, conflict resolution, advertising filtering, routing table construction, name reversal, and discovery pipeline orchestration. It sits between the HTTP middleware and the session object — operating on data that logically belongs to session construction.

The discovery path (`AggregateCapabilities`) is dead code — `WithSessionScopedRouting()` bypasses it entirely. Only `ProcessPreQueriedCapabilities` is called in production, during session creation.

As the aggregator's responsibilities shift (e.g. removing the filter), it becomes increasingly confusing to reason about what it does and when.

## Goals

- Fix composite tool type coercion for non-advertised backend tools
- Eliminate the `Aggregator` interface and `defaultAggregator` struct
- Move advertising filtering to a session decorator in the correct position in the decorator stack
- Keep conflict resolvers and per-backend override utilities as a reusable library
- No breaking changes to configuration or client-visible behavior

## Non-Goals

- Moving renaming/conflict-resolution into a session decorator (not feasible — routing requires private session state)
- Changing the `MultiSession` interface
- Refactoring Cedar authorization from HTTP middleware to a session decorator
- Removing the conflict resolver implementations or changing their behavior

## Proposed Solution

### High-Level Design

```
Decorator stack (top → bottom):

  optimizer            — replaces tool list with find_tool/call_tool
   filter              — NEW: advertising filter decorator
    composite tools    — workflow engine sees ALL renamed tools (fix!)
     base session      — routing table built by factory with resolved names
```

The session factory directly orchestrates: per-backend overrides → conflict resolution → routing table construction → advertising set computation. No `Aggregator` object. The advertising filter becomes a session decorator that sits above composite tools, so composite tools see all tool schemas while clients only see advertised tools.

### Detailed Design

#### Session factory inlines aggregator pipeline

`pkg/vmcp/session/factory.go`

Merge `buildRoutingTable` and `buildRoutingTableWithAggregator` into a single function:

```go
func buildRoutingTable(
    ctx context.Context,
    results []initResult,
    conflictResolver aggregator.ConflictResolver,
    toolConfigMap map[string]*config.WorkloadToolConfig,
    excludeAllTools bool,
) (*vmcp.RoutingTable, []vmcp.Tool, []vmcp.Resource, []vmcp.Prompt, map[string]bool, error)
```

When `conflictResolver` or `toolConfigMap` is non-nil:

1. Group tools by `BackendID`
2. Apply `aggregator.ProcessBackendTools()` per backend (overrides)
3. Call `conflictResolver.ResolveToolConflicts()` (conflict resolution)
4. Build routing table keyed by resolved names, with `OriginalCapabilityName` set via `aggregator.ActualBackendCapabilityName()`
5. Build `allTools` from ALL resolved tools (no filtering)
6. Compute `advertisedSet map[string]bool` keyed by resolved name — evaluates the filter logic inline

When both are nil: current first-writer-wins logic, `advertisedSet = nil`.

The `advertisedSet` is stored on `defaultMultiSession` as a private field.

#### Filter decorator

`pkg/vmcp/session/filterdec/decorator.go` (new)

```go
type filterDecorator struct {
    sessiontypes.MultiSession
    advertisedSet map[string]bool
}

func (d *filterDecorator) Tools() []vmcp.Tool {
    all := d.MultiSession.Tools()
    result := make([]vmcp.Tool, 0, len(all))
    for _, t := range all {
        // Composite tools (empty BackendID) always pass through
        if t.BackendID == "" || d.advertisedSet[t.Name] {
            result = append(result, t)
        }
    }
    return result
}
```

The decorator extracts `advertisedSet` from the base session via a private interface:

```go
type advertisedSetProvider interface {
    AdvertisedSet() map[string]bool
}
```

`CallTool`, `ReadResource`, `GetPrompt` pass through unchanged — the filter only affects what's advertised via `Tools()`, not what's callable.

#### Decorator wiring

`pkg/vmcp/server/sessionmanager/factory.go` (assumes PR #4231 landed)

Insert filter decorator between composite tools and optimizer:

```go
decorators = append(decorators, compositeToolsDecorator(...))
decorators = append(decorators, filterDecoratorFn())  // extracts advertisedSet from session
decorators = append(decorators, optimizerDecoratorFn(...))
```

#### Aggregator package becomes a utility library

Keep:
- `ConflictResolver` interface + 3 implementations (prefix, priority, manual)
- `ProcessBackendTools` (exported) — applies per-backend overrides
- `ActualBackendCapabilityName` (exported) — reverses overrides for backend forwarding
- `ResolvedTool`, `BackendCapabilities`, `ResolvedCapabilities` types

Delete:
- `Aggregator` interface, `defaultAggregator` struct
- `ProcessPreQueriedCapabilities`, `AggregateCapabilities`, `QueryCapabilities`, `QueryAllCapabilities`, `MergeCapabilities`
- `shouldAdvertiseTool` (logic inlined into factory)
- `AggregatedCapabilities`, `AggregationMetadata`, `BackendDiscoverer` types

#### Why renaming can't be a decorator

`defaultMultiSession.CallTool` uses the private `connections map[string]backend.Session` field for routing. A decorator wrapping `MultiSession` can't access this. If two backends both expose a tool named `fetch`, the base session's routing table (keyed by name) can only store one entry. A renaming decorator couldn't route `backend-a_fetch` to the correct backend by delegating `CallTool("fetch")` — the base session would route to whichever backend won first-writer-wins. The routing table must be built with conflict-resolved names at factory time.

### API Changes

**Removed**: `WithAggregator(agg aggregator.Aggregator)` session factory option

**Added**:
- `WithConflictResolver(cr aggregator.ConflictResolver)` — optional conflict resolver
- `WithToolConfig(cfg *config.AggregationConfig)` — optional aggregation config (overrides, filters)

Or a single `WithAggregationConfig` option that carries both.

**Exported**:
- `aggregator.ProcessBackendTools` (was `processBackendTools`)
- `aggregator.ActualBackendCapabilityName` (was `actualBackendCapabilityName`)

### Configuration Changes

None. All existing configuration fields (`aggregation.excludeAllTools`, `aggregation.tools[].excludeAll`, `aggregation.tools[].filter`, `aggregation.tools[].overrides`) continue to work identically. The implementation moves from the aggregator to the factory + decorator.

### Data Model Changes

`defaultMultiSession` gains a private `advertisedSet map[string]bool` field. Not exposed on any public interface.

## Security Considerations

### Threat Model

This change does not introduce new attack surface. The advertising filter continues to hide backend tools from clients — the mechanism changes (decorator vs aggregator) but the client-visible behavior is identical.

### Authentication and Authorization

No changes. Cedar authorization remains as HTTP middleware. The filter decorator operates below the auth layer.

### Data Security

No sensitive data handling changes.

### Input Validation

No new user input paths.

### Secrets Management

No changes.

### Audit and Logging

No changes. Filter decisions are evaluated at session creation time and are deterministic from the configuration.

### Mitigations

The `advertisedSet` is computed once at session creation and is immutable. The filter decorator is a simple set-membership check with no complex logic.

## Alternatives Considered

### Alternative 1: Add `AllRoutableTools()` to MultiSession interface

- Description: Add a new method that returns all tools including non-advertised ones
- Pros: Minimal change, composite tools call the new method
- Cons: Bloats the `MultiSession` interface with a method that exists only to work around the filter's position. Rejected during design review.

### Alternative 2: Store tool definitions on RoutingTable

- Description: Add `ToolDefinitions map[string]vmcp.Tool` to `RoutingTable`
- Pros: Avoids interface changes
- Cons: Duplicates data already on `vmcp.Tool`. Creates redundant state that must be kept in sync. Rejected during design review.

### Alternative 3: Filter decorator only (keep aggregator)

- Description: Move only the advertising filter to a decorator, keep the aggregator for everything else
- Pros: Smallest change, fixes the bug
- Cons: The aggregator continues to exist as a confusing middleman with shrinking responsibilities. Its remaining role (overrides + conflict resolution + routing table) is pure session factory logic.

### Alternative 4: Renaming as a session decorator

- Description: Move all aggregator responsibilities including renaming into decorators
- Pros: Full decomposition, no aggregator at all
- Cons: Not feasible — `CallTool` routing requires private session state (`connections` map). A renaming decorator can't route calls to the correct backend when multiple backends share a tool name.

## Compatibility

### Backward Compatibility

Fully backward compatible. All configuration fields work identically. Client-visible `tools/list` output is unchanged. The `Aggregator` interface is removed but it's an internal interface — no external consumers.

### Forward Compatibility

The decorator-based architecture makes it straightforward to add new session-level transforms (e.g. moving Cedar authorization from HTTP middleware to a decorator in the future).

## Implementation Plan

### Dependencies

- PR #4231 (`issue-3872-v3`) must land first — it introduces `DecoratingFactory` and `buildDecoratingFactory` in `sessionmanager/factory.go`

### Phase 1: Core changes

1. Inline aggregator pipeline into `buildRoutingTable` in `pkg/vmcp/session/factory.go`
2. Export `ProcessBackendTools` and `ActualBackendCapabilityName` from aggregator package
3. Create filter decorator at `pkg/vmcp/session/filterdec/decorator.go`
4. Wire filter decorator in `pkg/vmcp/server/sessionmanager/factory.go`
5. Update server wiring in `cmd/vmcp/app/commands.go` and `pkg/vmcp/server/server.go`

### Phase 2: Cleanup

6. Delete `Aggregator` interface, `defaultAggregator`, and associated methods
7. Delete dead discovery types (`AggregatedCapabilities`, `BackendDiscoverer`, etc.)
8. Update tests

## Testing Strategy

- **Unit tests**: Filter decorator (`filterdec/decorator_test.go`) — `Tools()` filtering, composite tool pass-through, `CallTool` pass-through
- **Unit tests**: Unified `buildRoutingTable` — with/without conflict resolver, `advertisedSet` computation
- **Regression test**: Workflow engine coercion — non-advertised tool receives `float64(42)` not `"42"` (`composer/workflow_engine_test.go`)
- **Existing tests**: Conflict resolver tests remain unchanged. Aggregator tests for deleted methods are removed.

```bash
go vet ./pkg/vmcp/... ./cmd/vmcp/...
go test ./pkg/vmcp/... ./cmd/vmcp/...
task lint-fix
```

## Documentation

- Update `docs/arch/` if aggregator is referenced in architecture docs
- No user-facing documentation changes (configuration is unchanged)

## Open Questions

1. Should the `shouldAdvertiseTool` comment (incorrectly says "before overrides" but actually receives post-override names) be fixed as part of this work, or is it moot since the method is deleted?
2. Should we add a Filter+Override combined test before deleting the aggregator, to document the actual behavior?

## References

- [toolhive#4287](https://github.com/stacklok/toolhive/issues/4287) — Bug: composite tool type coercion fails for non-advertised tools
- [toolhive#4231](https://github.com/stacklok/toolhive/pull/4231) — PR: Move decoration into DecoratingFactory (dependency)
- [THV-0008](THV-0008-virtual-mcp-server.md) — Virtual MCP Server RFC

---

## RFC Lifecycle

### Review History

| Date | Reviewer | Decision | Notes |
|------|----------|----------|-------|
| 2026-03-19 | @jerm-dro | Draft | Initial submission |

### Implementation Tracking

| Repository | PR | Status |
|------------|-----|--------|
| toolhive | TBD | Pending |
