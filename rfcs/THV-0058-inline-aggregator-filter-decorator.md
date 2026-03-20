# RFC-0058: Inline aggregator into session factory and extract advertising filter as session decorator

- **Status**: Draft
- **Author(s)**: Jeremy Drouillard (@jerm-dro)
- **Created**: 2026-03-19
- **Last Updated**: 2026-03-20
- **Target Repository**: toolhive
- **Related Issues**: [toolhive#4287](https://github.com/stacklok/toolhive/issues/4287)

## Summary

Decompose the monolithic `Aggregator` interface by inlining its pipeline (per-backend overrides, conflict resolution, routing table construction) into the session factory, and extracting the advertising filter into a session decorator. This fixes a bug where composite tool workflows fail type coercion for non-advertised backend tools, and simplifies the vMCP architecture by removing a layer that sits awkwardly between middleware and session.

## Problem Statement

### Bug: composite tool type coercion fails for non-advertised tools

When backend tools are excluded from advertising via `excludeAll` or `filter`, composite tool workflows fail. The workflow engine's `getToolInputSchema()` searches the session's tool list for `InputSchema` to coerce string arguments to their schema types. Non-advertised tools are absent from this list, so coercion is skipped and backends reject string-typed arguments (e.g. `pullNumber` as `"42"` instead of `float64(42)`).

The root cause: the advertising filter runs inside the aggregator **before** session creation. The session's `tools` field only contains advertised tools. The composite tools decorator builds its workflow engine from `sess.Tools()`, so it never sees schemas for non-advertised tools.

### Abstraction leak: workflow engine bypasses the session

The workflow engine directly uses `Router.RouteTool()` to resolve a `BackendTarget`, then calls `BackendClient.CallTool()` with that target — completely bypassing the session's `CallTool` method and every decorator in the stack. The session already encapsulates routing + backend dispatch in its own `CallTool`, making the engine's direct use of `Router` and `BackendClient` redundant. This leak is why `getToolInputSchema()` must maintain its own tool list rather than relying on the session — the engine doesn't call through the session at all.

### Architectural issue: the aggregator is a monolithic middleman

The `Aggregator` interface has 7 responsibilities: backend capability querying, per-backend overrides, conflict resolution, advertising filtering, routing table construction, name reversal, and discovery pipeline orchestration. It sits between the HTTP middleware and the session object — operating on data that logically belongs to session construction.

The discovery path (`AggregateCapabilities`) is dead code — `WithSessionScopedRouting()` bypasses it entirely. Only `ProcessPreQueriedCapabilities` is called in production, during session creation.

As the aggregator's responsibilities shift (e.g. removing the filter), it becomes increasingly confusing to reason about what it does and when.

## Goals

- Fix composite tool type coercion for non-advertised backend tools
- Eliminate the `Aggregator` interface and `defaultAggregator` struct
- Move advertising filtering to a session decorator in the correct position in the decorator stack
- Simplify the workflow engine by routing tool calls through the session instead of directly using `Router` + `BackendClient`
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

The workflow engine is simplified: instead of directly using `Router` and `BackendClient` to dispatch tool calls, it calls `sess.CallTool()`. The session already handles routing and backend dispatch. This eliminates the abstraction leak and means the engine's `getToolInputSchema()` can use `sess.Tools()` — which, at the composite tools layer (below the filter), includes all tools.

### Detailed Design

#### Session factory inlines aggregator pipeline

`pkg/vmcp/session/factory.go`

Merge `buildRoutingTable` and `buildRoutingTableWithAggregator` into a single function:

```go
func buildRoutingTable(
    ctx context.Context,
    results []initResult,
    conflictResolver ConflictResolver,
    toolConfigMap map[string]*config.WorkloadToolConfig,
    excludeAllTools bool,
) (*vmcp.RoutingTable, []vmcp.Tool, []vmcp.Resource, []vmcp.Prompt, error)
```

A conflict resolver is always provided — `NewConflictResolver(nil)` defaults to prefix strategy with `{workload}_` format, matching current behavior. The pipeline:

1. Group tools by `BackendID`
2. Apply `processBackendTools()` per backend (overrides)
3. Call `conflictResolver.ResolveToolConflicts()` (conflict resolution)
4. Build routing table keyed by resolved names, with `OriginalCapabilityName` set via `actualBackendCapabilityName()`
5. Build `allTools` from ALL resolved tools (no filtering)

The old `buildRoutingTable` (first-writer-wins, no conflict resolution) is removed entirely — the conflict resolver always handles name conflicts.

#### Filter decorator

`pkg/vmcp/session/filterdec/decorator.go` (new)

The filter decorator evaluates the advertising filter directly on `sess.Tools()` using the tool config. Each tool in the session has `BackendID` set, which is sufficient to evaluate per-workload `excludeAll` and `filter` rules. The filter config references post-override tool names, which match `tool.Name` after conflict resolution when no prefix was applied, or can be matched by stripping the known prefix.

```go
type filterDecorator struct {
    sessiontypes.MultiSession
    toolConfigMap   map[string]*config.WorkloadToolConfig
    excludeAllTools bool
}

func (d *filterDecorator) Tools() []vmcp.Tool {
    all := d.MultiSession.Tools()
    result := make([]vmcp.Tool, 0, len(all))
    for _, t := range all {
        // Composite tools (empty BackendID) always pass through
        if t.BackendID == "" || d.shouldAdvertise(t) {
            result = append(result, t)
        }
    }
    return result
}
```

`CallTool`, `ReadResource`, `GetPrompt` pass through unchanged — the filter only affects what's advertised via `Tools()`, not what's callable.

#### Workflow engine calls through the session

`pkg/vmcp/composer/workflow_engine.go`

The workflow engine currently takes a `Router` and `BackendClient` and dispatches tool calls directly:

```go
// Current: bypasses the session entirely
target, err := e.router.RouteTool(ctx, step.Tool)
result, err := e.backendClient.CallTool(ctx, target, step.Tool, args, nil)
```

This is replaced with a call through the session:

```go
// New: routes through the session's CallTool
result, err := e.session.CallTool(ctx, step.Tool, args, nil)
```

The session already handles routing table lookup, backend capability name reversal, and backend dispatch. The engine no longer needs `Router` or `BackendClient` interfaces — it only needs a reference to the `MultiSession` it decorates.

Similarly, `getToolInputSchema()` currently maintains its own `e.tools` list. It can instead call `e.session.Tools()`, which at the composite tools layer (below the filter) returns all tools including non-advertised ones. This naturally fixes the #4287 bug: the schema for every backend tool is available regardless of advertising.

The `ResolveToolName` call in `getToolInputSchema` (for dot-convention support) moves to the session's `CallTool` implementation, which already handles this via the session router.

#### Decorator wiring

`pkg/vmcp/server/sessionmanager/factory.go` (assumes PR #4231 landed)

Insert filter decorator between composite tools and optimizer. The filter decorator is always applied — it receives the tool config and evaluates advertising decisions against `sess.Tools()` at decoration time:

```go
decorators = append(decorators, compositeToolsDecorator(...))
decorators = append(decorators, filterDecoratorFn(toolConfigMap, excludeAllTools))
decorators = append(decorators, optimizerDecoratorFn(...))
```

#### Aggregator package becomes a utility library

Keep in aggregator package:
- `ConflictResolver` interface + 3 implementations (prefix, priority, manual)
- `NewConflictResolver` factory function
- `ResolvedTool`, `BackendCapabilities`, `ResolvedCapabilities` types

Move to session package (implementation detail):
- `processBackendTools` — used only by the session factory
- `actualBackendCapabilityName` — used only by the session factory

Delete:
- `Aggregator` interface, `defaultAggregator` struct
- `ProcessPreQueriedCapabilities`, `AggregateCapabilities`, `QueryCapabilities`, `QueryAllCapabilities`, `MergeCapabilities`
- `shouldAdvertiseTool` (logic moves to filter decorator)
- `AggregatedCapabilities`, `AggregationMetadata`, `BackendDiscoverer` types

#### Why renaming can't be a decorator

`defaultMultiSession.CallTool` uses the private `connections map[string]backend.Session` field for routing. A decorator wrapping `MultiSession` can't access this. If two backends both expose a tool named `fetch`, the base session's routing table (keyed by name) can only store one entry. A renaming decorator couldn't route `backend-a_fetch` to the correct backend by delegating `CallTool("fetch")` — the base session would route to whichever backend won first-writer-wins. The routing table must be built with conflict-resolved names at factory time.

### API Changes

**Removed**:
- `WithAggregator(agg aggregator.Aggregator)` session factory option
- `Router` and `BackendClient` dependencies from the workflow engine (tool calls route through the session)

**Added**:
- `WithAggregationConfig(cfg *config.AggregationConfig, cr ConflictResolver)` — required conflict resolver and aggregation config. The conflict resolver always exists (`NewConflictResolver` defaults to prefix strategy).

**Moved to session package** (implementation detail, not exported):
- `processBackendTools` — moves from `aggregator` to `session` package
- `actualBackendCapabilityName` — moves from `aggregator` to `session` package

### Configuration Changes

None. All existing configuration fields (`aggregation.excludeAllTools`, `aggregation.tools[].excludeAll`, `aggregation.tools[].filter`, `aggregation.tools[].overrides`) continue to work identically. The implementation moves from the aggregator to the factory + decorator.

### Data Model Changes

None. The filter decorator holds the tool config directly and evaluates advertising decisions at decoration time against `sess.Tools()`.

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

All configuration fields work identically. Client-visible behavior is unchanged. The `Aggregator` interface is removed but it's internal — no external consumers.

**Feature-by-feature analysis of the new decorator ordering:**

| Feature | Current flow | New flow | Impact |
|---|---|---|---|
| **Authentication (Cedar)** | HTTP middleware intercepts `tools/list` and `tools/call` at the JSON-RPC level, evaluates Cedar policies against resolved tool names | Unchanged — Cedar middleware sits above the entire session decorator stack, sees the same resolved tool names in requests/responses | None |
| **Composite tool calling** | Workflow engine receives `sess.Tools()` (advertised only) and `sess.GetRoutingTable()`. Calls `backendClient.CallTool()` directly (bypasses session decorators). `getToolInputSchema()` fails for non-advertised tools | Workflow engine calls `sess.CallTool()` — routes through the session instead of bypassing it. `getToolInputSchema()` uses `sess.Tools()` which, below the filter decorator, includes all tools | **Bug fix** — coercion works for non-advertised tools. **Simplification** — engine no longer depends on `Router` or `BackendClient` |
| **Advertising filter** | Aggregator applies `shouldAdvertiseTool` before session creation. `sess.Tools()` returns only advertised tools | Filter decorator applies after composite tools. `sess.Tools()` at the filter level returns only advertised tools + composite tools. Below the filter (where composite tools operate), `sess.Tools()` returns all tools | Same client-visible result; composite tools see more |
| **Optimizer** | Wraps session after composite tools, indexes `sess.Tools()` (advertised + composite) into find_tool/call_tool | Wraps session after filter, indexes `sess.Tools()` (advertised + composite) — same tools as before | None |
| **Tool call routing** | `defaultMultiSession.CallTool` looks up routing table → `GetBackendCapabilityName()` → backend HTTP call | Unchanged — routing table is built with the same resolved names and `OriginalCapabilityName` values, just by the factory instead of the aggregator | None |

### Forward Compatibility

The decorator-based architecture makes it straightforward to add new session-level transforms (e.g. moving Cedar authorization from HTTP middleware to a decorator in the future).

## Implementation Plan

### Dependencies

- PR #4231 (`issue-3872-v3`) must land first — it introduces `DecoratingFactory` and `buildDecoratingFactory` in `sessionmanager/factory.go`

### Phase 1: Core changes

1. Move `processBackendTools` and `actualBackendCapabilityName` from aggregator to session package
2. Inline aggregator pipeline into `buildRoutingTable` in `pkg/vmcp/session/factory.go` — always use conflict resolver
3. Create filter decorator at `pkg/vmcp/session/filterdec/decorator.go`
4. Wire filter decorator in `pkg/vmcp/server/sessionmanager/factory.go`
5. Simplify workflow engine: replace `Router` + `BackendClient` with `sess.CallTool()`; replace `e.tools` with `sess.Tools()`
6. Update server wiring in `cmd/vmcp/app/commands.go` and `pkg/vmcp/server/server.go`

### Phase 2: Cleanup

7. Delete `Aggregator` interface, `defaultAggregator`, and associated methods
8. Delete dead types (`AggregatedCapabilities`, `BackendDiscoverer`, etc.)
9. Delete aggregator tests for removed methods; remove `Router` and `BackendClient` interfaces from workflow engine

## Testing Strategy

All validation is at the K8s/Ginkgo E2E level (`test/e2e/thv-operator/virtualmcp/`). Tests deploy real MCPServers + VirtualMCPServers to a Kind cluster and exercise the full stack via MCP clients.

### Feature combination matrix

The features being modified interact with each other. The matrix below maps every relevant combination to an E2E test — either an existing one that provides coverage or a new one that must be written.

**Features**: **F** = Filter (per-workload), **EA** = ExcludeAll (per-workload or global), **R** = Renames/Overrides, **CR** = Conflict Resolution, **CT** = Composite Tools, **O** = Optimizer, **A** = Authn (non-anonymous)

| # | F | EA | R | CR | CT | O | A | E2E test | Status |
|---|---|---|---|---|---|---|---|----------|--------|
| 1 | | G | | prefix | | | anon | `virtualmcp_excludeall_global_test.go` | Existing |
| 2 | Y | Y | | prefix | | | anon | `virtualmcp_aggregation_filtering_test.go` | Existing |
| 3 | Y | Y | | prefix | Y | | anon | `virtualmcp_composite_hidden_tools_test.go` | Existing |
| 4 | | | | prefix/priority/manual | | | anon | `virtualmcp_conflict_resolution_test.go` | Existing |
| 5 | | | Y | prefix | | | anon | `virtualmcp_aggregation_overrides_test.go` | Existing |
| 6 | Y | | Y | prefix | | | anon | `virtualmcp_toolconfig_test.go` (MCPToolConfig CRD) | Existing |
| 7 | | | Y | prefix | Y | Y | anon | `virtualmcp_optimizer_composite_test.go` | Existing |
| 8 | Y | Y | | prefix | Y | | anon | **`virtualmcp_composite_coercion_test.go`** | **New** |
| 9 | Y | | Y | prefix | | | anon | **`virtualmcp_overrides_filter_test.go`** | **New** |
| 10 | Y | Y | Y | prefix | Y | Y | anon | **`virtualmcp_full_stack_test.go`** | **New** |
| 11 | | | | prefix | | | token | `virtualmcp_external_auth_test.go` | Existing |

**Legend**: G = global ExcludeAllTools, Y = feature active in test, blank = not exercised.

### Existing tests (regression gates)

All existing tests in the matrix must pass without modification after the refactor. Any failure is a regression.

- **Row 1**: Global `ExcludeAllTools: true` hides all backend tools; MCP server still responds.
- **Row 2**: Per-workload `Filter` + `ExcludeAll` applied to different backends; only filtered tools exposed.
- **Row 3**: Composite tool calls hidden backend tools (ExcludeAll + Filter). Validates #3636 fix. Only tests string params — does NOT exercise type coercion.
- **Row 4**: Prefix, priority, and manual conflict resolution with two backends sharing tool names.
- **Row 5**: Tool name/description overrides; overridden tool callable by new name.
- **Row 6**: MCPToolConfig CRD drives filtering + overrides simultaneously.
- **Row 7**: Optimizer wraps composite tools + overridden backend tools through `find_tool`/`call_tool`.
- **Row 11**: Token-based incoming auth with backend routing.

### New E2E tests

#### Row 8: Composite tool type coercion with hidden tools (regression test for #4287)

**File**: `test/e2e/thv-operator/virtualmcp/virtualmcp_composite_coercion_test.go`

The specific bug this RFC fixes. When a composite tool workflow step calls a hidden backend tool with a numeric parameter, `getToolInputSchema()` fails to find the schema, skips coercion, and the backend rejects `"42"` (string) instead of `float64(42)`.

**Setup**:
- Backend with a tool accepting an integer parameter (requires adding an `add` tool to yardstick: `{"a": integer, "b": integer}` → returns sum)
- `ExcludeAll: true` for the backend
- Composite tool whose workflow step calls the hidden tool via template expansion (`"{{ .params.a }}"`)

**Assertions**:
- `tools/list` returns only the composite tool
- Calling the composite tool succeeds — backend receives coerced numeric args
- Response contains correct output (the sum), proving type coercion worked

**Must fail before the refactor, pass after.**

#### Row 9: Overrides + filter on the same backend

**File**: `test/e2e/thv-operator/virtualmcp/virtualmcp_overrides_filter_test.go`

No existing test exercises overrides and filter together. The `shouldAdvertiseTool` comment bug (says "before overrides" but receives post-override name) was never caught because this combination is untested.

**Setup**:
- Backend with tool `echo`, overridden to `custom_echo`
- Filter configured with `["echo"]` (pre-override name)

**Assertions**:
- Document whether the filter matches pre- or post-override names
- Verify the overridden tool is callable by its new name
- Lock in the actual behavior so the refactor preserves it

#### Row 10: Full stack — all features combined

**File**: `test/e2e/thv-operator/virtualmcp/virtualmcp_full_stack_test.go`

Exercises the entire decorator stack with all features active simultaneously.

**Setup**:
- Two backends with conflicting tool names (prefix conflict resolution)
- Backend A: tool overridden (`echo` → `custom_echo`), plus filter hiding some tools
- Backend B: `ExcludeAll: true`
- Composite tool calling tools from both backends (including hidden + overridden)
- Optimizer enabled

**Assertions**:
- `tools/list` via optimizer returns only `find_tool`/`call_tool`
- `find_tool` returns the composite tool + filtered backend A tools (not excluded backend B tools, not hidden backend A tools)
- `call_tool` with composite tool succeeds — reaches both backends with correct original names
- Overridden tool callable by its new name through the optimizer
- Backend B tools reachable via composite tool despite `ExcludeAll`

### Unit tests for the filter decorator

**File**: `pkg/vmcp/session/filterdec/decorator_test.go`

Lightweight unit tests for the new filter decorator in isolation:
- `Tools()` with `excludeAll` → only composite tools returned
- `Tools()` with `filter` → only matching backend tools + composite tools
- `CallTool` passes through for both advertised and non-advertised tools
- No filter config → all tools pass through

### Cleanup

- Delete aggregator unit tests for removed methods (`ProcessPreQueriedCapabilities`, `MergeCapabilities`, `shouldAdvertiseTool`)
- Conflict resolver unit tests remain unchanged

### Verification

```bash
# Unit tests
go vet ./pkg/vmcp/... ./cmd/vmcp/...
go test ./pkg/vmcp/... ./cmd/vmcp/...
task lint-fix

# E2E tests (Kind cluster)
task test-e2e
```

## Documentation

- Update `docs/arch/` if aggregator is referenced in architecture docs
- No user-facing documentation changes (configuration is unchanged)

## Open Questions

None — all design questions resolved during review.

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
