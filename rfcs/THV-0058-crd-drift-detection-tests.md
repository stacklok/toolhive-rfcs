# RFC: Drift Detection Tests for CRD-to-Runtime Type Mappings

## Status

Draft

## Context

ToolHive's operator defines CRD types (e.g., `MCPTelemetryOTelConfig`) that are structurally distinct from their runtime counterparts (e.g., `telemetry.Config`). This is by design — the CRD types use a nested structure for better UX, exclude CLI-only fields, and add K8s-specific fields like `sensitiveHeaders`. A thin conversion layer maps between them.

This separation provides important benefits:
- **No field leakage** — CLI-only fields like `environmentVariables` and per-server fields like `serviceName` don't appear in the CRD schema
- **Independent evolution** — adding a CLI flag doesn't change the CRD's API surface
- **Clean CRD UX** — nested `openTelemetry`/`prometheus` structure vs flat config

However, it introduces a maintenance risk: when a field is added to either the CRD type or the runtime config type, the conversion layer and the other type may not be updated. This has historically caused silent bugs (PR #3118, issue #3125) where configuration appeared to work in tests but failed through the CRD path.

## Problem

Today, the only way to catch a missed field mapping is:
1. Manual code review (unreliable for large PRs)
2. An integration or e2e test that exercises the specific field end-to-end (expensive to write, easy to miss)

Neither approach scales. A developer adding a field to `telemetry.Config` for a CLI feature has no signal that the CRD conversion layer needs attention — the code compiles, unit tests pass, and the drift is invisible until a user hits it.

## Proposal

Add **reflection-based drift detection tests** alongside each CRD conversion function. These tests enumerate the JSON fields on both the CRD type and the runtime type, then assert that every field is either:
- **Mapped** in the conversion function, or
- **Explicitly excluded** with a documented reason

This turns a silent drift into a compile-time-like failure: add a field to either type and the test fails immediately, forcing the developer to make a conscious decision.

### Example: MCPTelemetryConfig

```go
func TestTelemetryConfigFieldCoverage(t *testing.T) {
    t.Parallel()

    // Fields in telemetry.Config intentionally NOT in the CRD.
    // Adding a field here requires a justification comment.
    excludedFromCRD := map[string]string{
        "serviceName":          "per-server field, set via MCPTelemetryConfigReference",
        "serviceVersion":       "resolved at runtime from binary version",
        "environmentVariables": "CLI-only, not applicable to CRD-managed telemetry",
        "customAttributes":     "replaced by resourceAttributes in the CRD",
    }

    // Mapping from telemetry.Config JSON field -> CRD location.
    // Documents the conversion performed by NormalizeMCPTelemetryConfig.
    mappedToCRD := map[string]string{
        "endpoint":                    "MCPTelemetryOTelConfig.Endpoint",
        "tracingEnabled":              "OpenTelemetryTracingConfig.Enabled",
        "metricsEnabled":              "OpenTelemetryMetricsConfig.Enabled",
        "samplingRate":                "OpenTelemetryTracingConfig.SamplingRate",
        "headers":                     "MCPTelemetryOTelConfig.Headers",
        "insecure":                    "MCPTelemetryOTelConfig.Insecure",
        "enablePrometheusMetricsPath": "PrometheusConfig.Enabled",
        "useLegacyAttributes":         "MCPTelemetryOTelConfig.UseLegacyAttributes",
    }

    configFields := jsonFieldNames(reflect.TypeOf(telemetry.Config{}))

    for _, field := range configFields {
        if _, excluded := excludedFromCRD[field]; excluded {
            continue
        }
        if _, mapped := mappedToCRD[field]; mapped {
            continue
        }
        t.Errorf("telemetry.Config field %q is not mapped in "+
            "NormalizeMCPTelemetryConfig and not in the exclusion list", field)
    }
}
```

### What this catches

| Scenario | Without drift test | With drift test |
|----------|-------------------|-----------------|
| New field added to `telemetry.Config` for CLI | Silent -- field missing from CRD conversion, users never see it | **Test fails** -- developer must add to mapping or exclusion list |
| New field added to CRD type | Works, but no validation it maps to runtime correctly | **Test fails** -- developer must document the mapping |
| Field renamed in `telemetry.Config` | Conversion breaks silently -- old name mapped, new name ignored | **Test fails** -- old mapping name no longer matches |
| Field removed from `telemetry.Config` | Conversion references non-existent field -- compile error (already caught) | Same -- compile error |

### The exclusion list as documentation

The `excludedFromCRD` map serves double duty:
1. **Test assertion** -- the test passes only if every field is accounted for
2. **Living documentation** -- explains *why* each field is excluded, reviewable in PRs

```go
excludedFromCRD := map[string]string{
    "serviceName":          "per-server field, set via MCPTelemetryConfigReference",
    "environmentVariables": "CLI-only, not applicable to CRD-managed telemetry",
}
```

Adding a field to this map requires a justification string. This forces the conversation during code review: *"Why is this field excluded from the CRD?"*

## Scope

This pattern applies to any CRD type that has a conversion layer to a runtime config type:

| CRD Type | Runtime Type | Conversion Function |
|----------|-------------|-------------------|
| `MCPTelemetryOTelConfig` | `telemetry.Config` | `NormalizeMCPTelemetryConfig` |
| `TelemetryConfig` (inline) | `telemetry.Config` | `ConvertTelemetryConfig` |
| Future CRDs with separate runtime types | -- | -- |

CRDs that directly embed their runtime type (e.g., VirtualMCPServer embedding `config.Config`) don't need this pattern -- they have no conversion layer to drift.

## Implementation

1. Add a `jsonFieldNames(reflect.Type) []string` helper to a shared test utilities package
2. Add a `TestTelemetryConfigFieldCoverage` test in `pkg/spectoconfig/` alongside the conversion function
3. Document the pattern in `.claude/rules/testing.md` so future CRD authors follow it

## Alternatives Considered

### Inline embedding with CEL guards
Embed `telemetry.Config` directly and use CEL rules to block unwanted fields. Rejected because:
- Go cannot override embedded struct tags -- fields leak into the CRD schema
- CEL blocks at admission but fields remain visible in `kubectl explain`
- Coupling means CLI changes silently change the CRD API surface

### `conversion-gen` from k8s code-generator
Auto-generates conversion functions between API versions. Not applicable here because:
- Designed for API version conversion (v1alpha1 <-> v1beta1), not CRD-to-runtime mapping
- Requires matching field names/structure -- ours are structurally different (nested vs flat)
- Heavyweight infrastructure (scheme registration, hub/spoke pattern)

### Shared canonical type with CLI split
Restructure `telemetry.Config` into shared + CLI-only structs, enabling embedding without leaks. Valid long-term approach but:
- Epic-sized refactor touching CLI, VirtualMCPServer, all telemetry consumers
- Doesn't eliminate the need for drift tests (fields can still be added to the wrong struct)

## Decision

Adopt reflection-based drift detection tests as the standard pattern for CRD conversion layers. This is lightweight (no new dependencies, ~50 lines per test), catches the most common failure mode (missed field mappings), and produces clear error messages pointing to the exact field and required action.
