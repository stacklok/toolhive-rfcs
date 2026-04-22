# RFC-0070: `thv llm` Subcommand for LLM Gateway Authentication

- **Status**: Draft
- **Author(s)**: Jeremy Drouillard (@jerm-dro)
- **Created**: 2026-04-17
- **Last Updated**: 2026-04-17
- **Target Repository**: toolhive
- **Related Issues**: N/A

## Summary

ToolHive gains a `thv llm` command group that bridges AI coding tools to
OIDC-protected LLM gateways. Two authentication modes cover the full spectrum of
tools: a localhost reverse proxy for tools that only accept static API keys, and a
token helper for tools with native OIDC support. A single `thv llm setup` command
detects installed tools, configures them, and handles the OIDC login — mirroring
ToolHive's existing auto-wiring of MCP tools into client applications.

## Problem Statement

AI coding tools need API keys or tokens to make LLM requests. In enterprise
environments, these requests route through a gateway that authenticates developers
via OIDC. This creates a compatibility gap:

- **OIDC-capable tools** (Claude Code, Codex CLI, avante.nvim) support dynamic
  token commands — they can invoke an external program to fetch a fresh token before
  each request. These tools just need a token helper wired in and pointed at the
  gateway.
- **Static-key-only tools** (Roo Code, Cline, Cursor, Continue, Aider, Zed) accept
  a single API key and base URL at configuration time. They cannot perform OAuth
  flows, store refresh tokens, or handle token expiry. When the token expires, the
  tool breaks silently.

Today there is no standard way for developers to bridge this gap. They resort to
manual token copy-paste, custom scripts, or giving up on enterprise gateways
entirely.

ToolHive already auto-wires MCP tools into client applications — detecting installed
clients, writing their configuration, and managing the connection lifecycle.
Extending this to LLM gateway access is a natural next step. ToolHive also has OIDC
and OAuth infrastructure, a secrets provider for credential storage, and a CLI that
developers already use. It is the natural home for solving this problem.

## Goals

- Transparent OIDC authentication for static-key-only tools via a localhost reverse
  proxy that injects fresh tokens on every request.
- A token helper for OIDC-capable tools that prints a fresh access token to stdout,
  suitable for use as an `apiKeyHelper` or `auth.command`.
- Auto-detect installed AI coding tools and configure them to route through the
  proxy or directly to the gateway in a single command.
- A shared OIDC session across both modes — one browser login covers all tools.
- Upstream-agnostic design that works with any LLM gateway accepting OIDC JWTs.

## Non-Goals

- Backend gateway implementation details — this RFC covers the client-side CLI only.
  The upstream gateway is treated as an opaque HTTPS endpoint that accepts OIDC
  bearer tokens.
- Formal daemon lifecycle management — `proxy stop`, `proxy status`, auto-restart,
  process supervision. The proxy runs in the background when started by `setup` and
  is stopped by `teardown`, but richer daemon controls are future work.
- Exhaustive tool coverage — Claude Code and Cursor are explored in this RFC to
  illustrate the two authentication modes (direct token helper and proxy
  respectively). Support for additional tools will be evaluated on a case-by-case
  basis.
- Unified single sign-on across MCP and LLM gateway authentication — the OIDC flows
  for MCP tool authentication and LLM gateway access are fundamentally different.
  Unifying them into a single login would add significant complexity. Two separate
  logins is acceptable for now.
- HTTP API exposure — the config model and core logic are designed to be reusable,
  but this RFC does not define API endpoints for managing LLM gateway configuration.
  Exposing these settings via the HTTP API (e.g., for UI-driven setup) is future
  work.

## Proposed Solution

### User Experience

The primary entry point is `thv llm setup`. A developer configuring LLM gateway
access for the first time runs:

```
$ thv llm setup \
    --gateway-url https://llm.example.com \
    --issuer https://auth.example.com \
    --client-id my-client-id
```

This single command:

1. **Saves the gateway connection config** — gateway URL, OIDC issuer, client ID,
   and defaults are persisted in ToolHive's config file.
2. **Detects installed AI coding tools** — scans for known tool config directories
   (e.g., `~/.claude/`, `~/.cursor/`).
3. **Configures each detected tool**:
   - For OIDC-capable tools (Claude Code): writes a token helper that calls
     `thv llm token`, sets the gateway as the base URL.
   - For static-key-only tools (Cursor): sets the proxy's localhost address as the
     base URL with a placeholder API key.
4. **Starts the localhost proxy** in the background (if any static-key tool was
   configured).
5. **Triggers the OIDC browser login** — the developer authenticates once, and the
   refresh token is stored securely for future sessions.

After setup, all configured tools work immediately. Token refresh is automatic and
transparent.

To reverse the configuration:

```
$ thv llm teardown
```

This removes the LLM gateway configuration from each tool's config file, stops the
background proxy if running, and optionally purges cached OIDC tokens with
`--purge-tokens`. Teardown can also target specific tools:
`thv llm teardown cursor`.

### Command Structure

```
thv llm
├── setup          # Detect tools, configure them, start proxy, trigger OIDC login
├── teardown       # Reverse setup — remove added config keys, stop proxy
├── config
│   ├── set        # Manually adjust gateway URL, OIDC settings, ports
│   ├── show       # Display current LLM config (text or JSON)
│   └── reset      # Clear all LLM config and cached tokens
├── proxy
│   └── start      # Start the proxy in the foreground (for debugging)
└── token          # (hidden) Print fresh access token to stdout
```

**Why each command exists:**

- **`setup` / `teardown`** are the primary commands most users interact with. They
  handle the full lifecycle.
- **`config set/show/reset`** give advanced users manual control over gateway
  connection settings. Both `setup` and `config set` write to the same config —
  they are two paths to configure the `thv llm` commands. `config set` does not
  modify running proxy instances or already-wired tool configs.
- **`proxy start`** runs the proxy in the foreground with full log output. This is a
  debugging tool — when something isn't working, running the proxy interactively
  lets the developer see request/response flow and token acquisition in real time.
- **`token`** is a hidden subcommand. It is not intended to be called by users
  directly — it exists as the target for tool-specific token helpers
  (`apiKeyHelper`, `auth.command`). It prints a single JWT to stdout and exits.

### Token Source

The token source manages OIDC token acquisition and refresh with a three-tier
strategy:

1. **In-memory cache** — If a valid (non-expired) access token exists in memory,
   return it immediately. No I/O, no network.
2. **Secrets provider restore** — If no in-memory token exists, attempt to load the
   refresh token from the secrets provider (OS keyring or encrypted file). Use it to
   obtain a fresh access token from the OIDC provider.
3. **Browser OIDC+PKCE flow** — If no refresh token is available (first login or
   revoked), launch an authorization code flow with PKCE (S256) via the system
   browser. The user authenticates, and both access and refresh tokens are returned.

After any successful token acquisition:

- The access token is cached in memory only — never written to disk.
- The refresh token is stored in the secrets provider for future sessions.
- The `oauth2.TokenSource` wrapper handles automatic refresh before expiry.

**Preemptive refresh:** The token source refreshes proactively when the access token
is within 30 seconds of expiry, avoiding request failures at the boundary.

**Non-interactive mode:** The `thv llm token` command runs non-interactively — it
will not launch a browser flow. If no cached or restorable token is available, it
returns an error. The browser flow is only triggered during `setup` or `proxy start`
(interactive commands).

### Proxy

The localhost reverse proxy accepts unauthenticated HTTP requests from static-key
tools and forwards them to the upstream gateway with a fresh OIDC token injected.

**Request flow:**

1. AI tool sends request to `http://localhost:14000/v1/chat/completions` with a
   placeholder API key.
2. Proxy strips the incoming `Authorization` header.
3. Proxy calls `TokenSource.Token()` to get a fresh access token.
4. Proxy sets `Authorization: Bearer <access_token>` on the outgoing request.
5. Proxy forwards to the upstream gateway configured at startup (the destination is
   fixed in the config, never derived from the incoming request). The original
   request path and query string are appended to the configured gateway URL.
6. Response streams back to the tool — SSE streaming is preserved without buffering
   (`FlushInterval: -1`).

**Key properties:**

- **Loopback-only binding** — The proxy validates at startup that the listen address
  resolves to a loopback interface. Binding to `0.0.0.0` or non-loopback addresses
  is rejected.
- **Health endpoint** — `GET /healthz` returns `{"status":"ok","upstream":"..."}` for
  liveness checks.
- **OpenAI-compatible errors** — Token acquisition failures return HTTP 401 with an
  error body matching the OpenAI API schema, so tools display meaningful messages.
- **TLS skip verify** — An optional `--tls-skip-verify` flag for development
  environments with self-signed certificates. Disabled by default.
- **SSE streaming** — The proxy uses `httputil.ReverseProxy` with `FlushInterval: -1`
  to stream Server-Sent Events without buffering, which is critical for chat
  completion responses.

### Token Helper

`thv llm token` is a hidden subcommand that prints a single fresh JWT to stdout and
exits. It is the integration point for OIDC-capable tools:

- **Claude Code**: configured via `apiKeyHelper` in `~/.claude/settings.json` — Claude
  Code calls the helper before each API request.
- **Codex CLI**: configured via `auth.command` — same pattern.
- **avante.nvim**: configured via a `cmd:` prefix in the API key field.

The token helper is the preferred integration mode — it does not require the
background proxy to be running and handles token lifecycle independently. The command
runs non-interactively: it will use cached or refreshed tokens but will not launch a
browser flow. During execution, stdout is reserved exclusively for the JWT — all
other output (update checks, warnings, progress) is redirected to stderr to prevent
corrupting the token value.

### Setup and Teardown

#### Tool Detection

Setup scans for installed AI coding tools by checking for known configuration
directories and files. Each supported tool defines:

- **Detect** — How to determine if the tool is installed (e.g., does
  `~/.claude/settings.json` or `~/.claude/` exist?).
- **Apply** — How to configure the tool to use the gateway (what config keys to set,
  what values).
- **Revert** — How to remove the configuration added by setup on teardown.

#### Two Tool Kinds

**Direct (OIDC-capable tools like Claude Code):**

- A helper script is generated at `~/.config/toolhive/bin/thv-llm-token` that
  invokes `thv llm token`.
- The tool's config is patched to set the helper script as its token command
  (`apiKeyHelper` for Claude Code) and the gateway URL as its base URL
  (`ANTHROPIC_BASE_URL`).
- Requests go directly from the tool to the gateway — no proxy needed.

**Proxy (static-key-only tools like Cursor):**

- The tool's config is patched to set the localhost proxy address
  (`http://localhost:14000/v1`) as its base URL and a placeholder key
  (`sk-toolhive-proxy`) as its API key.
- The background proxy must be running for these tools to function.

#### Helper Script

OIDC-capable tools like Claude Code don't call `thv llm token` directly — they
invoke a helper script that wraps it. The indirection is needed because these tools
expect a path to an executable in their config (e.g., `apiKeyHelper` in Claude
Code's `settings.json`). The helper script provides a stable path that setup writes
once; if the `thv` binary moves or the invocation needs to change, only the script
is updated — no re-patching of tool configs required.

Setup generates the script at `~/.config/toolhive/bin/thv-llm-token`:

```bash
#!/bin/bash
exec thv llm token
```

#### Config Patching

Tool configuration files (JSON, JSONC) are modified in place using ToolHive's
existing config editing infrastructure (`pkg/client/config_editor.go`). This uses
hujson for lenient JSON parsing (preserves comments and formatting) and RFC 6902
JSON Patch operations for adds and removes. All writes are atomic (write to temp
file, then rename) and protected by dual-layer file locking (in-process mutex +
OS-level advisory lock).

This is the same mechanism ToolHive uses to auto-wire MCP servers into client
applications — no new patching approach is introduced.

#### Config Modification and Removal

Following ToolHive's existing convention for MCP auto-wiring, client config files
are modified in place without backups. Setup applies RFC 6902 "add" patches to set
the relevant keys, and teardown applies "remove" patches to undo them. This is
consistent with how `pkg/client/` handles MCP server registration and removal
today.

#### Background Proxy Lifecycle

When setup configures at least one proxy-mode tool, it starts the proxy as a
background process:

- The proxy is launched via `os/exec` as a detached process.
- The PID is written to `~/.config/toolhive/llm-proxy.pid`.
- Teardown sends SIGTERM to the process when no proxy-mode tools remain.

### Adding New Tool Support

Supporting a new AI coding tool requires:

1. **Define detection** — How to check if the tool is installed (config directory,
   binary path, etc.).
2. **Define apply** — What config keys to set and what values (base URL, API key or
   token helper path).
3. **Define revert** — How to undo the apply step (remove the keys that were added).
4. **Register the tool** — Add it to the tool registry so setup discovers it.

No changes to the setup/teardown orchestration, proxy, token source, or CLI commands
are needed. The orchestration iterates over all registered tools and calls their
Detect/Apply/Revert functions uniformly.

### Configuration Model

LLM gateway settings are stored in ToolHive's existing config file
(`~/.config/toolhive/config.yaml`) under a new `llm:` key, persisted via the same
`UpdateConfig()` mechanism used by runtime configs and registry auth — atomic updates
with file locking. The Go types live in `pkg/llm/config.go` to keep the LLM domain
self-contained; `pkg/config/config.go` references them via a field on the top-level
`Config` struct.

```yaml
llm:
  gateway_url: https://llm.example.com
  oidc:
    issuer: https://auth.example.com
    client_id: my-client-id
    scopes:
      - openid
      - offline_access
    audience: ""             # optional
    callback_port: 0         # 0 = ephemeral (auto-assigned)
  proxy:
    listen_port: 14000
  auth:
    cached_token_expiry: "2026-04-17T12:00:00Z"
  configured_tools:
    - tool: claude-code
      mode: direct
      config_path: /home/user/.claude/settings.json
    - tool: cursor
      mode: proxy
      config_path: /home/user/.cursor/settings.json
```

**Go types:**

```go
type LLMConfig struct {
    GatewayURL      string          `yaml:"gateway_url"`
    OIDC            LLMOIDCConfig   `yaml:"oidc"`
    Proxy           LLMProxyConfig  `yaml:"proxy"`
    Auth            LLMAuthState    `yaml:"auth"`
    ConfiguredTools []LLMToolConfig `yaml:"configured_tools"`
}

type LLMOIDCConfig struct {
    Issuer       string   `yaml:"issuer"`
    ClientID     string   `yaml:"client_id"`
    Scopes       []string `yaml:"scopes"`
    Audience     string   `yaml:"audience,omitempty"`
    CallbackPort int      `yaml:"callback_port,omitempty"`
}

type LLMProxyConfig struct {
    ListenPort int `yaml:"listen_port"`
}

type LLMAuthState struct {
    CachedTokenExpiry time.Time `yaml:"cached_token_expiry"`
}

type LLMToolConfig struct {
    Tool       string `yaml:"tool"`
    Mode       string `yaml:"mode"`
    ConfigPath string `yaml:"config_path"`
}
```

`IsConfigured()` validates that the minimum required fields (gateway URL, issuer,
client ID) are present. `Validate()` performs full validation including HTTPS
enforcement, port range checks, and OIDC field requirements.

### Package Layout

```
pkg/llm/
├── doc.go                        # Package documentation
├── llm.go                        # Public API surface (Setup, Teardown, NewProxy, NewTokenSource)
├── config.go                     # LLMConfig, LLMOIDCConfig, LLMProxyConfig, LLMAuthState, LLMToolConfig
└── internal/
    ├── errors.go                 # OpenAI-compatible error responses
    ├── errors_test.go
    ├── proxy.go                  # Reverse proxy with token injection
    ├── proxy_test.go
    ├── setup.go                  # Tool registry, detection, apply/revert
    ├── setup_test.go
    ├── token_source.go           # 3-tier OIDC token acquisition
    ├── token_source_test.go
    └── validate.go               # Loopback address validation

cmd/thv/app/
└── llm.go                        # CLI commands: setup, teardown, config, proxy start, token
```

Config types (`LLMConfig`, `LLMOIDCConfig`, etc.) live in `pkg/llm/config.go` rather
than `pkg/config/` — this keeps the LLM domain self-contained and avoids duplicative
types. Tests in `internal/` use these config types directly. The `internal/` package
contains the implementation details — proxy, token source, tool registry, validation,
and error formatting. The parent `pkg/llm/` exposes only the minimal public API
needed by `cmd/thv/app/llm.go`. Most tests live in `internal/` alongside the code
they exercise.

## Security Considerations

### Threat Model

The primary threat is unauthorized access to the LLM gateway via stolen tokens or
proxy misuse. The attack surface is:

- **Local process access** — Any process on the developer's machine can reach the
  localhost proxy.
- **Token extraction** — If an attacker gains access to the secrets provider, they
  could extract the refresh token.
- **Network exposure** — If the proxy were bound to a non-loopback address, remote
  hosts could use it as an authentication oracle.

### Authentication and Authorization

- OIDC Authorization Code flow with PKCE (S256) is used for all authentication.
  This is the recommended flow for public clients (no client secret).
- No client secret is required or stored.
- The proxy does not authenticate incoming requests — it relies on the localhost
  trust model (only local processes can reach it).

### Data Security

- **Access tokens** are held in memory only and never written to disk or logged.
- **Refresh tokens** are stored in the OS keyring (macOS Keychain, Linux
  Secret Service) via the secrets provider, falling back to AES-256-GCM encrypted
  file storage.
- **Token values are never logged** — log messages reference token metadata (expiry,
  scope) but never the token string itself.

### Input Validation

- Gateway URL must use HTTPS (enforced by `Validate()`).
- Listen address must resolve to a loopback interface — `ValidateLoopbackAddress()`
  rejects `0.0.0.0`, non-loopback IPs, and hostnames that resolve to non-loopback
  addresses.
- OIDC configuration fields (issuer, client ID) are validated as non-empty and
  well-formed.

### Secrets Management

- Refresh tokens are stored via ToolHive's existing secrets provider.
- Tokens are revocable — `thv llm config reset` and `thv llm teardown --purge-tokens`
  delete cached tokens from the secrets provider.
- The `thv llm token` command redirects all non-token output to stderr, preventing
  accidental token leakage into log files or terminal scrollback.

### Audit and Logging

Basic audit logging for security-relevant operations:

- **Proxy**: Logs each proxied request (method, path, upstream status code) without
  logging token values or request/response bodies.
- **Token execution**: Logs token acquisition events (cache hit, refresh, browser
  flow triggered) and failures, with token expiry metadata but never the token itself.
- **Setup/teardown**: Logs which tools were detected, configured, and removed.

### Mitigations

| Threat | Mitigation |
|--------|------------|
| Proxy accessible from network | Loopback-only binding enforced at startup |
| Token theft from disk | Access tokens memory-only; refresh tokens in OS keyring |
| Token leakage in logs | Token values never logged; stderr isolation in token helper |
| Stale tokens after offboarding | `teardown --purge-tokens` clears all cached credentials |
| Man-in-the-middle on gateway | HTTPS enforced for gateway URL; TLS verification on by default |

## Alternatives Considered

### Alternative 1: Standalone Binary

Build the proxy and token helper as a separate binary, independent of ToolHive.

- **Pros**: No dependency on ToolHive; simpler distribution for non-ToolHive users.
- **Cons**: Duplicates ToolHive's OIDC, secrets provider, and config infrastructure.
  Two binaries to install and update. Cannot leverage ToolHive's existing client
  auto-wiring patterns.
- **Why not chosen**: ToolHive already has every building block needed. A standalone
  binary would mean maintaining parallel implementations of OAuth flows, secret
  storage, and config management.

### Alternative 2: Manual Configuration

Do nothing — let developers manually configure each tool's base URL and API key to
point at the gateway, and handle token refresh themselves.

- **Pros**: No new code to write or maintain.
- **Cons**: Error-prone and tedious. Developers must understand each tool's config
  format, manually copy tokens, and deal with silent failures when tokens expire.
  Does not scale across teams or tools.
- **Why not chosen**: The manual approach is the status quo and is the problem this
  RFC solves.

## Compatibility

### Backward Compatibility

This is a purely additive change. The `thv llm` command group is new — no existing
commands are modified. The `LLM` field in the config struct is optional and defaults
to zero values (unconfigured).

Existing ToolHive functionality (MCP tool management, secrets, registry) is
unaffected.

### Forward Compatibility

- The tool registry is designed for extension — adding new tools requires
  implementing Detect/Apply/Revert functions with no changes to orchestration.
- The config model includes optional fields (audience, callback port) that
  accommodate different IdP requirements.
- Daemon mode, `proxy stop`/`status`, and richer tool lifecycle management can be
  layered on in future RFCs without breaking the foundation established here.

## Implementation Plan

### Phase 1: Core Infrastructure

Build the foundational components that all commands depend on:

- **Config types** — `LLMConfig`, `LLMOIDCConfig`, `LLMProxyConfig`, `LLMAuthState`
  in `pkg/llm/config.go`. Persisted via ToolHive's existing `UpdateConfig()` with
  file locking, stored in `config.yaml` under the `llm:` key — the same persistence
  pattern used by runtime configs and registry auth.
- **Token source** — Three-tier acquisition in `pkg/llm/internal/token_source.go`.
  Refresh tokens stored via the secrets provider.
- **Reverse proxy** — `pkg/llm/internal/proxy.go` with token injection, SSE
  streaming, loopback validation, health endpoint.
- **CLI commands** — `config set/show/reset`, `proxy start` (foreground), `token`
  (hidden).

### Phase 2: Auto-Wiring

Build the tool detection and auto-configuration layer on top of Phase 1:

- **Tool detection and registry** — `pkg/llm/internal/setup.go` with Detect/Apply/Revert
  functions for Claude Code (direct mode) and Cursor (proxy mode).
- **Helper script generation** — Write `thv-llm-token` script for direct-mode tools.
- **Client config patching** — Modify tool configs in place using ToolHive's existing
  `pkg/client/config_editor.go` infrastructure. Teardown removes added keys via
  RFC 6902 "remove" patches.
- **Background proxy management** — Start/stop proxy as a detached process, PID
  tracking.
- **CLI commands** — `setup` and `teardown`.
- **`configured_tools` tracking** — Record what setup configured in
  `Config.LLM.ConfiguredTools` so teardown knows exactly what to reverse.

### Dependencies

- Existing `pkg/auth/oauth` — OIDC authorization code flow with PKCE.
- Existing `pkg/secrets/` — Secrets provider (keyring + encrypted file fallback).
- Existing `pkg/config/` — Config loading, saving, and file-locked updates.
- Existing `pkg/client/` — Client config editing infrastructure.

## Testing Strategy

### Unit Tests

- **Proxy** — `httptest` server as upstream. Verify token injection, header
  stripping, SSE streaming passthrough, error handling (token failure → 401,
  upstream failure → 502), health endpoint.
- **Token source** — Mock OIDC provider and mock secrets provider. Verify the
  three-tier fallback chain, preemptive refresh behavior, non-interactive mode
  rejection.
- **Config validation** — Test `IsConfigured()`, `Validate()`, HTTPS enforcement,
  port range checks, default values.
- **Setup/teardown** — Filesystem fixtures (temporary directories with mock tool
  config files). Verify detection, config patching, key removal on teardown,
  helper script content.

### Adding Tests for New Tools

Each tool's Detect/Apply/Revert functions are independently testable. When adding
support for a new tool, the test pattern is:

1. Create a fixture directory mimicking the tool's config layout.
2. Test **Detect** — returns true when the fixture exists, false otherwise.
3. Test **Apply** — verify the config file is patched with the expected keys/values.
4. Test **Revert** — verify the added keys are removed from the config file.

This pattern is self-contained per tool. No changes to proxy, token source, or
orchestration tests are needed when adding a new tool.

### Integration Tests

- End-to-end proxy flow with a mock upstream server that validates the injected
  bearer token and returns streamed responses.
- Setup/teardown round-trip: run setup, verify tool configs are patched, run
  teardown, verify added keys are removed.

## Documentation

CLI help text for all `thv llm` commands. No additional user documentation in this
initial iteration.

## Open Questions

1. What is the priority of additional tool support beyond Claude Code and Cursor?

## References

None.

---

## RFC Lifecycle

<!-- This section is maintained by RFC reviewers -->

### Review History

| Date | Reviewer | Decision | Notes |
|------|----------|----------|-------|
| 2026-04-17 | @jerm-dro | Draft | Initial submission |

### Implementation Tracking

| Repository | PR | Status |
|------------|-----|--------|
| toolhive | TBD | Not started |
