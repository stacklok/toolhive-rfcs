# RFC-0015: Add Tool Definitions to Publisher Extensions

> **Note**: This was originally [THV-2733](https://github.com/stacklok/toolhive/pull/2733) in the toolhive repository.

- **Status**: Draft
- **Author(s)**: Juan Antonio Osorio (@JAORMX)
- **Created**: 2025-11-25
- **Last Updated**: 2025-11-25
- **Target Repository**: toolhive-registry
- **Related Issues**: [toolhive#2733](https://github.com/stacklok/toolhive/pull/2733)

## Summary

Add comprehensive tool metadata to the existing ToolHive publisher extension (`io.github.stacklok`) in the upstream MCP Registry format. This extends beyond tool names to include descriptions, input/output schemas, and annotations.

## Problem Statement

The current registry stores only tool names (`tools: []string`), which limits discoverability and tooling capabilities. Users and AI agents need richer metadata to:
- Understand what each tool does before running a server
- Validate tool inputs/outputs programmatically
- Enable better tool selection and filtering in aggregation scenarios (e.g., Virtual MCP Server)
- Support IDE/editor integrations with type information

## Goals

- Store full MCP tool definitions in the registry without breaking existing consumers
- Reuse the existing `io.github.stacklok` publisher extension namespace
- Align with the MCP specification's tool schema
- Maintain backward compatibility with the existing `tools` field

## Non-Goals

- Not changing how tools are discovered at runtime (still via `tools/list`)
- Not requiring `tool_definitions` for registry entries
- Not adding new publisher extension namespaces
- Not modifying the upstream MCP Registry schema

## Proposed Solution

Add a new `tool_definitions` field to the existing ToolHive publisher extension structure. This field contains an array of tool objects matching the MCP specification.

### High-Level Design

The `tool_definitions` field is added alongside existing fields in the `io.github.stacklok` extension:

```json
{
  "meta": {
    "publisher_provided": {
      "io.github.stacklok": {
        "<image_or_url>": {
          "status": "active",
          "tier": "Official",
          "tools": ["get_weather", "search_location"],
          "tool_definitions": [
            {
              "name": "get_weather",
              "title": "Weather Information",
              "description": "Get current weather for a location",
              "inputSchema": {
                "type": "object",
                "properties": {
                  "location": {
                    "type": "string",
                    "description": "City name or coordinates"
                  }
                },
                "required": ["location"]
              },
              "outputSchema": {
                "type": "object",
                "properties": {
                  "temperature": { "type": "number" },
                  "conditions": { "type": "string" }
                }
              },
              "annotations": {
                "readOnly": true
              }
            }
          ]
        }
      }
    }
  }
}
```

### Detailed Design

#### Tool Definition Fields

Per the [MCP specification](https://modelcontextprotocol.io/specification/2025-06-18/server/tools):

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | `string` | Yes | Unique identifier for the tool |
| `title` | `string` | No | Human-readable display name |
| `description` | `string` | No | Human-readable description of functionality |
| `inputSchema` | `object` | No | JSON Schema defining expected parameters |
| `outputSchema` | `object` | No | JSON Schema defining expected output structure |
| `annotations` | `object` | No | Properties describing tool behavior |

#### Annotations

Tool annotations provide hints about tool behavior:

| Annotation | Type | Description |
|------------|------|-------------|
| `readOnly` | `boolean` | Tool only reads data, no side effects |
| `destructive` | `boolean` | Tool may perform destructive operations |
| `idempotent` | `boolean` | Repeated calls with same args have same effect |
| `openWorld` | `boolean` | Tool interacts with external entities |

#### Data Model Changes

Add a new type to `pkg/registry/registry/registry_types.go`:

```go
// ToolDefinition represents the full metadata for an MCP tool
type ToolDefinition struct {
    Name         string         `json:"name"`
    Title        string         `json:"title,omitempty"`
    Description  string         `json:"description,omitempty"`
    InputSchema  map[string]any `json:"inputSchema,omitempty"`
    OutputSchema map[string]any `json:"outputSchema,omitempty"`
    Annotations  map[string]any `json:"annotations,omitempty"`
}
```

Update `BaseServerMetadata` to include the new field:

```go
type BaseServerMetadata struct {
    // ... existing fields ...
    Tools           []string          `json:"tools" yaml:"tools"`
    ToolDefinitions []*ToolDefinition `json:"tool_definitions,omitempty" yaml:"tool_definitions,omitempty"`
    // ... remaining fields ...
}
```

#### Converter Updates

**ToolHive to Upstream (`toolhive_to_upstream.go`)**:

Add `tool_definitions` to the extension creation functions:

```go
if len(metadata.ToolDefinitions) > 0 {
    extensions["tool_definitions"] = metadata.ToolDefinitions
}
```

**Upstream to ToolHive (`upstream_to_toolhive.go`)**:

Extract `tool_definitions` from the extension data:

```go
if toolDefs, ok := extensions["tool_definitions"].([]interface{}); ok {
    metadata.ToolDefinitions = remarshalToType[[]*ToolDefinition](toolDefs)
}
```

## Security Considerations

### Threat Model

This change introduces minimal security risk as it only adds metadata storage.

- **Data integrity**: Tool definitions are provided by MCP server publishers. Malicious publishers could provide misleading metadata (e.g., claiming a tool is read-only when it isn't).
- **Schema injection**: Input/output schemas are stored as JSON objects. These could potentially contain malicious content if rendered unsafely.

### Data Security

- Tool definitions are public metadata, similar to existing tool names
- No sensitive data is stored in tool definitions
- Schemas follow JSON Schema standard, which is data-only (no executable code)

### Input Validation

- Tool names must be valid identifiers (alphanumeric, underscores)
- Schemas should be validated as proper JSON Schema format
- Size limits should be enforced to prevent oversized definitions

### Mitigations

- Validate schema structure on ingestion
- Sanitize any user-facing display of descriptions
- Enforce reasonable size limits on tool definitions

## Alternatives Considered

### Alternative 1: Separate Extension Namespace

Create a new extension namespace specifically for tool definitions.

- **Pros**: Cleaner separation, independent evolution
- **Cons**: More complexity, duplicated tool references
- **Why not chosen**: Reusing existing namespace is simpler and maintains consistency

### Alternative 2: External Tool Definition Registry

Store tool definitions in a separate registry or service.

- **Pros**: Independent scaling, specialized service
- **Cons**: Additional infrastructure, synchronization challenges
- **Why not chosen**: Adds operational complexity without clear benefit

## Compatibility

### Backward Compatibility

- The `tools` field remains unchanged and continues to store tool names as strings
- Consumers that only read `tools` are unaffected
- The `tool_definitions` field is optional; servers without it continue to work
- When both fields exist, `tool_definitions` is the authoritative source; `tools` serves as a quick lookup

### Forward Compatibility

- Schema is designed to accommodate additional MCP tool fields as the spec evolves
- Using `map[string]any` for schemas allows flexibility without breaking changes

## Implementation Plan

### Phase 1: Type Definitions

- Add `ToolDefinition` type to registry types
- Update `BaseServerMetadata` with new field
- Add JSON/YAML serialization support

### Phase 2: Converter Updates

- Update ToolHive-to-upstream converter
- Update upstream-to-ToolHive converter
- Add unit tests for round-trip conversion

### Phase 3: CLI Integration

- Add `--show-tools` flag to `thv list` command
- Display tool descriptions in output

## Testing Strategy

- **Unit tests**: Serialization/deserialization of tool definitions
- **Integration tests**: Round-trip conversion between formats
- **E2E tests**: Registry operations with tool definitions

## Documentation

- Update registry format documentation
- Add examples of tool definitions in registry entries
- Document use cases for tool definitions

## Open Questions

1. Should we validate that `tools` array matches `tool_definitions[*].name`?
2. What size limits should be enforced on tool definitions?

## References

- [MCP Specification - Tools](https://modelcontextprotocol.io/specification/2025-06-18/server/tools)
- [JSON Schema Specification](https://json-schema.org/)

---

## RFC Lifecycle

### Review History

| Date | Reviewer | Decision | Notes |
|------|----------|----------|-------|
| 2025-11-25 | - | Draft | Ported from toolhive PR #2733 |

### Implementation Tracking

| Repository | PR | Status |
|------------|-----|--------|
| toolhive | - | Pending |
