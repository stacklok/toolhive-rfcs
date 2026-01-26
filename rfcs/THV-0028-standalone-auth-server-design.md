# Authserver Standalone Kubernetes Deployment Design

## Overview

This document describes how `pkg/authserver` could be deployed as a standalone Kubernetes service with mutual TLS (mTLS) authentication between the authserver and proxyrunner components.

---

## Current State

### Authserver ([pkg/authserver/](pkg/authserver/))
- Full OAuth 2.0/OIDC authorization server using Fosite
- Discovery endpoints: `/.well-known/openid-configuration`, `/.well-known/oauth-authorization-server`, `/.well-known/jwks.json`
- Authorization flow: `/oauth/authorize` → upstream IDP → `/oauth/callback` → issues own JWT tokens
- JWT sessions with `tsid` claim linking to stored upstream IDP tokens
- Signing key support: RSA/ECDSA/Ed25519
- In-memory storage only (no persistent backend)
- Key files:
  - [server/handlers/handler.go](pkg/authserver/server/handlers/handler.go) - HTTP routing
  - [server/handlers/discovery.go](pkg/authserver/server/handlers/discovery.go) - Discovery endpoints
  - [server/handlers/authorize.go](pkg/authserver/server/handlers/authorize.go) - Authorization handler
  - [server/handlers/callback.go](pkg/authserver/server/handlers/callback.go) - Callback with token exchange

### Proxyrunner ([cmd/thv-proxyrunner/](cmd/thv-proxyrunner/))
- Container runner wrapper (not an auth gateway)
- Uses middleware chain (auth, tokenexchange, authz, audit)
- Reads `runconfig.json` for configuration
- No mTLS currently - token-based auth only

### TLS/Certificate Patterns
- [pkg/networking/http_client.go](pkg/networking/http_client.go) - `HttpClientBuilder` with CA bundle support
- No client certificate support for mTLS
- Kubernetes uses ConfigMaps for CA distribution
- No cert-manager integration

---

## Proposed Architecture

### 1. Kubernetes Deployment Model

#### 1.1 MCPAuthServer (New)

A new CRD and controller for deploying the authserver as a standalone Kubernetes service.

**CRD Specification:**

Following the [MCPServer CRD pattern](cmd/thv-operator/api/v1alpha1/mcpserver_types.go):

```yaml
apiVersion: toolhive.stacklok.dev/v1alpha1
kind: MCPAuthServer
metadata:
  name: main-authserver
  namespace: toolhive-system
spec:
  issuer: "https://mcp-authserver.toolhive-system.svc.cluster.local"
  replicas: 2
  port: 8443

  # Upstream identity providers for user authentication
  # Currently supports a single IDP; multiple IDPs planned for vMCP use case
  upstreamIdps:
    - name: google  # Unique identifier for this IDP
      type: oidc
      oidc:
        issuer: "https://accounts.google.com"
        clientId: "..."
        clientSecretRef:
          name: authserver-secrets
          key: oidc-client-secret

  # Signing keys for JWT issuance and JWKS endpoint
  # First key is the active signing key; subsequent keys are advertised on JWKS for rotation
  # Key IDs (kid) are computed using RFC 7638 thumbprints
  signingKeys:
    - secretRef:
        name: authserver-signing-key
        key: private.pem
      algorithm: RS256
    # Previous key still advertised on JWKS during rotation period
    # - secretRef:
    #     name: authserver-signing-key-old
    #     key: private.pem
    #   algorithm: RS256

  tls:
    # Issuer for server certificate (controller creates the Certificate resource)
    serverCert:
      issuerRef:
        name: toolhive-mtls-ca
        kind: ClusterIssuer
      duration: "8760h"     # 1 year (optional, has default)
      renewBefore: "720h"   # 30 days (optional, has default)

    # mTLS: Client CA and validation rules for proxyrunner authentication
    clientAuth:
      # CA bundle for validating proxyrunner client certificates
      caBundle:
        configMapRef:
          name: toolhive-mtls-ca-bundle
          key: ca.crt

      # Allowed client certificate patterns (for access control)
      # Uses SPIFFE URI format: spiffe://{trust.domain}/ns/{namespace}/mcpserver/{name}
      allowedSubjects:
        # Trust domain for SPIFFE URIs (required)
        trustDomain: "toolhive.local"
        # Allow proxyrunners from specific namespaces (optional)
        allowedNamespaces:
          - "toolhive-system"
          - "mcp-servers"
          - "mcp-production"
        # Allow only specific MCPServer names (optional)
        # If not specified, all MCPServers in allowed namespaces are permitted
        allowedNames:
          - "github-tools"
          - "slack-bot"
```

**MCPAuthServer CRD Types:**

```go
// MCPAuthServerSpec defines the desired state of MCPAuthServer
type MCPAuthServerSpec struct {
    // Issuer is the OAuth 2.0/OIDC issuer URL for this authserver
    // This is the base URL used in token "iss" claims and discovery endpoints
    // +kubebuilder:validation:Required
    Issuer string `json:"issuer"`

    // Replicas is the number of authserver pod replicas
    // +kubebuilder:default=1
    // +optional
    Replicas *int32 `json:"replicas,omitempty"`

    // Port is the HTTPS port for the authserver (default: 8443)
    // +kubebuilder:default=8443
    // +optional
    Port int32 `json:"port,omitempty"`

    // UpstreamIdps configures upstream identity providers for user authentication
    // The authserver federates authentication to these IDPs
    // Currently only a single IDP is supported; multiple IDPs planned for vMCP use case
    // +kubebuilder:validation:MinItems=1
    // +kubebuilder:validation:MaxItems=1
    // +kubebuilder:validation:Required
    UpstreamIdps []UpstreamIdpConfig `json:"upstreamIdps"`

    // SigningKeys configures JWT signing keys for the authserver
    // The first key in the list is the active signing key used for new tokens
    // Subsequent keys are included in the JWKS endpoint to support key rotation
    // This allows clients to verify tokens signed with previous keys during rotation
    // Key IDs (kid) are computed using RFC 7638 JWK Thumbprints
    // +kubebuilder:validation:MinItems=1
    // +kubebuilder:validation:Required
    SigningKeys []SigningKeyConfig `json:"signingKeys"`

    // TLS configures TLS and mTLS for the authserver
    // +kubebuilder:validation:Required
    TLS AuthServerTLSConfig `json:"tls"`
}

// UpstreamIdpType represents the type of upstream identity provider
type UpstreamIdpType string

const (
    // UpstreamIdpTypeOIDC is for OpenID Connect providers
    UpstreamIdpTypeOIDC UpstreamIdpType = "oidc"
)

// UpstreamIdpConfig configures an upstream identity provider for user authentication
type UpstreamIdpConfig struct {
    // Name is a unique identifier for this IDP within the MCPAuthServer
    // Used to reference the IDP in logs and potentially in multi-IDP routing (future)
    // +kubebuilder:validation:Required
    // +kubebuilder:validation:Pattern=`^[a-z0-9]([a-z0-9-]*[a-z0-9])?$`
    Name string `json:"name"`

    // Type is the type of identity provider
    // +kubebuilder:validation:Enum=oidc
    // +kubebuilder:validation:Required
    Type UpstreamIdpType `json:"type"`

    // OIDC configures an OpenID Connect identity provider
    // Required when Type is "oidc"
    // +optional
    OIDC *OIDCIdpConfig `json:"oidc,omitempty"`
}

// OIDCIdpConfig configures an OIDC identity provider
type OIDCIdpConfig struct {
    // Issuer is the OIDC issuer URL (e.g., "https://accounts.google.com")
    // +kubebuilder:validation:Required
    Issuer string `json:"issuer"`

    // ClientID is the OAuth 2.0 client identifier
    // +kubebuilder:validation:Required
    ClientID string `json:"clientId"`

    // ClientSecretRef references a Kubernetes Secret containing the client secret
    // +kubebuilder:validation:Required
    ClientSecretRef SecretKeyRef `json:"clientSecretRef"`

    // Scopes is the list of OAuth 2.0 scopes to request (default: ["openid", "profile", "email"])
    // +optional
    Scopes []string `json:"scopes,omitempty"`
}

// SigningKeyConfig configures a JWT signing key for the authserver
// The key ID (kid) is computed using RFC 7638 JWK Thumbprint
type SigningKeyConfig struct {
    // SecretRef references a Kubernetes Secret containing the private key
    // +kubebuilder:validation:Required
    SecretRef SecretKeyRef `json:"secretRef"`

    // Algorithm is the JWT signing algorithm to use with this key
    // +kubebuilder:validation:Enum=RS256;RS384;RS512;ES256;ES384;ES512;EdDSA
    // +kubebuilder:validation:Required
    Algorithm string `json:"algorithm"`
}

// AuthServerTLSConfig configures TLS and mTLS for the authserver
type AuthServerTLSConfig struct {
    // ServerCert configures automatic server certificate provisioning
    // Controller creates a cert-manager Certificate using this issuer
    // +kubebuilder:validation:Required
    ServerCert ServerCertConfig `json:"serverCert"`

    // ClientAuth configures mTLS client certificate validation
    // +optional
    ClientAuth *ClientAuthConfig `json:"clientAuth,omitempty"`
}

// ServerCertConfig configures automatic server certificate provisioning
type ServerCertConfig struct {
    // IssuerRef references the cert-manager issuer to use
    // +kubebuilder:validation:Required
    IssuerRef CertManagerIssuerReference `json:"issuerRef"`

    // Duration is the certificate validity period (default: 8760h / 1 year)
    // +kubebuilder:default="8760h"
    // +optional
    Duration string `json:"duration,omitempty"`

    // RenewBefore is when to renew before expiry (default: 720h / 30 days)
    // +kubebuilder:default="720h"
    // +optional
    RenewBefore string `json:"renewBefore,omitempty"`
}

// ClientAuthConfig configures mTLS client verification for proxyrunners
type ClientAuthConfig struct {
    // CABundle references a ConfigMap containing the CA certificate for
    // validating proxyrunner client certificates
    // +kubebuilder:validation:Required
    CABundle CABundleSource `json:"caBundle"`

    // AllowedSubjects restricts which client certificates are accepted
    // If not specified, any certificate signed by the CA is accepted
    // +optional
    AllowedSubjects *AllowedSubjects `json:"allowedSubjects,omitempty"`
}

// AllowedSubjects defines which certificate subjects are allowed to connect
// Uses SPIFFE URI format: spiffe://{trustDomain}/ns/{namespace}/mcpserver/{name}
type AllowedSubjects struct {
    // TrustDomain is the SPIFFE trust domain for validating client certificate URIs
    // Example: "toolhive.local"
    // +kubebuilder:validation:Required
    TrustDomain string `json:"trustDomain"`

    // AllowedNamespaces is a list of Kubernetes namespaces whose MCPServers are allowed
    // The SPIFFE URI must match: spiffe://{trustDomain}/ns/{namespace}/mcpserver/{name}
    // If empty, all namespaces are allowed (only trustDomain is validated)
    // +optional
    AllowedNamespaces []string `json:"allowedNamespaces,omitempty"`

    // AllowedNames is a list of MCPServer names that are allowed to connect
    // When specified, only MCPServers with matching names are permitted
    // If empty, all MCPServer names are allowed (subject to namespace restrictions)
    // +optional
    AllowedNames []string `json:"allowedNames,omitempty"`
}
```

**MCPAuthServer Controller:**

The controller reconciles MCPAuthServer resources and creates/manages the following Kubernetes resources:

1. **cert-manager Certificate** - Server certificate for TLS:
   ```yaml
   # Created from tls.serverCert configuration
   apiVersion: cert-manager.io/v1
   kind: Certificate
   metadata:
     name: ${MCPAuthServer.name}-tls
     namespace: ${MCPAuthServer.namespace}
   spec:
     secretName: ${MCPAuthServer.name}-tls
     duration: ${tls.serverCert.duration}      # default: 8760h
     renewBefore: ${tls.serverCert.renewBefore} # default: 720h
     issuerRef: ${tls.serverCert.issuerRef}
     commonName: ${MCPAuthServer.name}.${namespace}.svc.cluster.local
     dnsNames:
       - ${MCPAuthServer.name}.${namespace}.svc.cluster.local
       - ${MCPAuthServer.name}.${namespace}.svc
       - ${MCPAuthServer.name}
     usages:
       - server auth
   ```
2. **Deployment** - Authserver pods running `thv-authserver` image:
   - Mounts server certificate Secret at `/etc/toolhive/server-tls`
   - Mounts client CA ConfigMap at `/etc/toolhive/client-ca`
   - Mounts ConfigMap for authserver configuration
   ```yaml
   volumes:
     - name: server-tls
       secret:
         secretName: ${MCPAuthServer.name}-tls
     - name: client-ca
       configMap:
         name: ${tls.clientAuth.caBundle.configMapRef.name}
   volumeMounts:
     - name: server-tls
       mountPath: /etc/toolhive/server-tls
       readOnly: true
     - name: client-ca
       mountPath: /etc/toolhive/client-ca
       readOnly: true
   ```
3. **Service** - ClusterIP for internal access (port 443 → 8443)
4. **ConfigMap** - Runtime configuration (issuer, signing key path, client CA path)
5. **ServiceAccount** - Kubernetes RBAC identity

The controller watches for changes to the MCPAuthServer CR and reconciles the dependent resources. It also monitors the cert-manager Certificate for readiness before creating the Deployment.

---

#### 1.2 MCPExternalAuthConfig Updates (authServer type)

Add a new external auth type `authServer` to `MCPExternalAuthConfig` for configuring the proxyrunner's mTLS client authentication to the standalone authserver. This leverages the existing `MCPExternalAuthConfig` pattern and allows the same authserver configuration to be shared across multiple MCPServer resources.

**CRD Addition** ([cmd/thv-operator/api/v1alpha1/mcpexternalauthconfig_types.go](cmd/thv-operator/api/v1alpha1/mcpexternalauthconfig_types.go)):

```go
// Add new auth type constant
const (
    // ExternalAuthTypeAuthServer is the type for standalone authserver mTLS authentication
    // Used when proxyrunner needs to exchange client JWTs for upstream tokens via the authserver
    ExternalAuthTypeAuthServer ExternalAuthType = "authServer"
)

// MCPExternalAuthConfigSpec defines the desired state of MCPExternalAuthConfig.
type MCPExternalAuthConfigSpec struct {
    // Type is the type of external authentication to configure
    // +kubebuilder:validation:Enum=tokenExchange;headerInjection;bearerToken;unauthenticated;authServer
    // +kubebuilder:validation:Required
    Type ExternalAuthType `json:"type"`

    // ... existing fields ...

    // AuthServer configures mTLS client authentication to a standalone authserver
    // Only used when Type is "authServer"
    // +optional
    AuthServer *AuthServerConfig `json:"authServer,omitempty"`
}

// AuthServerConfig configures mTLS client authentication to the standalone authserver
type AuthServerConfig struct {
    // URL is the authserver base URL
    // +kubebuilder:validation:Required
    URL string `json:"url"`

    // ClientCert configures automatic client certificate provisioning for mTLS
    // If specified, the MCPExternalAuthConfig controller creates the Certificate
    // and the MCPServer controller mounts it to the proxyrunner pod
    // +optional
    ClientCert *ClientCertificateConfig `json:"clientCert,omitempty"`

    // CABundle references a ConfigMap containing the CA bundle for verifying authserver
    // Reuses existing CABundleSource type
    // +optional
    CABundle *CABundleSource `json:"caBundle,omitempty"`
}

// ClientCertificateConfig configures automatic client certificate provisioning
type ClientCertificateConfig struct {
    // IssuerRef references the cert-manager ClusterIssuer to use
    // +kubebuilder:validation:Required
    IssuerRef CertManagerIssuerReference `json:"issuerRef"`

    // TrustDomain is the SPIFFE trust domain for the client certificate URI SAN
    // The certificate will include: spiffe://{trustDomain}/ns/{namespace}/mcpserver/{name}
    // +kubebuilder:validation:Required
    TrustDomain string `json:"trustDomain"`

    // Note: duration and renewBefore use reasonable defaults (90 days / 15 days)
    // and are not exposed in the CRD to keep the API simple
}

// CertManagerIssuerReference references a cert-manager Issuer or ClusterIssuer
// NOTE: This type is shared between MCPAuthServer and MCPExternalAuthConfig CRDs.
// Define in cmd/thv-operator/api/v1alpha1/certmanager_types.go (new file)
type CertManagerIssuerReference struct {
    // Name of the issuer
    Name string `json:"name"`

    // Kind is "Issuer" or "ClusterIssuer"
    // +kubebuilder:validation:Enum=Issuer;ClusterIssuer
    // +kubebuilder:default=ClusterIssuer
    Kind string `json:"kind,omitempty"`
}
```

**Example MCPExternalAuthConfig for authServer:**

```yaml
apiVersion: toolhive.stacklok.dev/v1alpha1
kind: MCPExternalAuthConfig
metadata:
  name: main-authserver-client
  namespace: mcp-servers
spec:
  type: authServer

  authServer:
    url: "https://mcp-authserver.toolhive-system.svc.cluster.local"

    # Controller will create a cert-manager Certificate with:
    # - CN: {mcpserver-name} (human-readable for audit logs)
    # - URI SAN: spiffe://{trustDomain}/ns/{namespace}/mcpserver/{name}
    # The certificate uses reasonable defaults: 90 days validity, renew at 15 days before expiry
    clientCert:
      # Trust domain for SPIFFE URIs in client certificates
      trustDomain: "toolhive.local"
      issuerRef:
        name: toolhive-mtls-ca
        kind: ClusterIssuer

    # CA bundle for verifying authserver's server certificate
    caBundle:
      configMapRef:
        name: toolhive-mtls-ca-bundle
        key: ca.crt
```

**Example MCPServer referencing the authServer config:**

```yaml
apiVersion: toolhive.stacklok.dev/v1alpha1
kind: MCPServer
metadata:
  name: github-tools
  namespace: mcp-servers
spec:
  image: ghcr.io/example/github-mcp:latest

  # OIDC config - proxyrunner validates JWTs using authserver's JWKS
  # The issuer also appears in /.well-known/oauth-protected-resource for client discovery
  oidcConfig:
    type: inline
    resourceUrl: "https://github-tools.example.com/"
    inline:
      issuer: "https://mcp-authserver.toolhive-system.svc.cluster.local"
      audience: "github-tools"

  # Reference the MCPExternalAuthConfig for authserver mTLS
  externalAuthConfigRef:
    name: main-authserver-client
```

**Controller Behavior:**

When an `MCPExternalAuthConfig` with `type: authServer` is referenced by an MCPServer:

1. **MCPExternalAuthConfig Controller creates a cert-manager Certificate** (if `clientCert` specified):
   ```yaml
   apiVersion: cert-manager.io/v1
   kind: Certificate
   metadata:
     name: github-tools-mtls
     namespace: mcp-servers
   spec:
     secretName: github-tools-mtls
     duration: 2160h    # 90 days (default)
     renewBefore: 360h  # 15 days (default)
     commonName: github-tools  # Human-readable for audit logs
     uris:
       - "spiffe://toolhive.local/ns/mcp-servers/mcpserver/github-tools"
     usages:
       - client auth
     issuerRef:
       name: toolhive-mtls-ca
       kind: ClusterIssuer
   ```

2. **MCPServer Controller mounts certificates to the proxyrunner pod:**
   ```yaml
   volumes:
     # Client certificate for mTLS (if clientCert specified)
     - name: authserver-client-cert
       secret:
         secretName: github-tools-mtls
     # CA bundle for verifying authserver (if caBundle specified)
     - name: authserver-ca-bundle
       configMap:
         name: toolhive-mtls-ca-bundle
   volumeMounts:
     - name: authserver-client-cert
       mountPath: /etc/toolhive/authserver-mtls
       readOnly: true
     - name: authserver-ca-bundle
       mountPath: /etc/toolhive/authserver-ca
       readOnly: true
   ```

3. **MCPServer Controller sets runconfig:**
   ```json
   {
     "authserver_config": {
       "url": "https://mcp-authserver.toolhive-system.svc.cluster.local",
       "client_cert_path": "/etc/toolhive/authserver-mtls/tls.crt",
       "client_key_path": "/etc/toolhive/authserver-mtls/tls.key",
       "ca_bundle_path": "/etc/toolhive/authserver-ca/ca.crt"
     }
   }
   ```

---

### 2. OAuth Discovery Flow (RFC 9728)

The client initially only knows about the MCP server (proxyrunner). Discovery happens via the RFC 9728 Protected Resource Metadata flow:

```
┌─────────────┐                    ┌─────────────────┐                  ┌──────────────────┐
│   Client    │                    │   ProxyRunner   │                  │    AuthServer    │
│   (Agent)   │                    │   (MCP Server)  │                  │                  │
└──────┬──────┘                    └────────┬────────┘                  └────────┬─────────┘
       │                                    │                                    │
       │ 1. MCP Request (no auth)           │                                    │
       │───────────────────────────────────►│                                    │
       │                                    │                                    │
       │ 2. 401 Unauthorized                │                                    │
       │    WWW-Authenticate: Bearer        │                                    │
       │    resource_metadata="/.well-known/oauth-protected-resource"            │
       │◄───────────────────────────────────│                                    │
       │                                    │                                    │
       │ 3. GET /.well-known/oauth-protected-resource                            │
       │───────────────────────────────────►│                                    │
       │                                    │                                    │
       │ 4. RFC 9728 Protected Resource Metadata                                 │
       │    { "authorization_servers": ["https://mcp-authserver..."],            │
       │      "resource": "...", "jwks_uri": "..." }                             │
       │◄───────────────────────────────────│                                    │
       │                                    │                                    │
       │ 5. GET /.well-known/openid-configuration (to authserver)                │
       │─────────────────────────────────────────────────────────────────────────►
       │                                    │                                    │
       │ 6. OIDC Discovery Document                                              │
       │    { "issuer": "...", "authorization_endpoint": "...",                  │
       │      "token_endpoint": "...", "jwks_uri": "..." }                       │
       │◄─────────────────────────────────────────────────────────────────────────
       │                                    │                                    │
       │ 7. OAuth Authorization Flow (PKCE) │                                    │
       │─────────────────────────────────────────────────────────────────────────►
       │                                    │                                    │
       │ 8. Access Token (JWT)              │                                    │
       │◄─────────────────────────────────────────────────────────────────────────
       │                                    │                                    │
       │ 9. MCP Request + Bearer Token      │                                    │
       │───────────────────────────────────►│                                    │
       │                                    │                                    │
       │ 10. Success                        │                                    │
       │◄───────────────────────────────────│                                    │
```

#### ProxyRunner's Protected Resource Metadata (Step 4)

The proxyrunner exposes `/.well-known/oauth-protected-resource` via [pkg/auth/token.go:NewAuthInfoHandler()](pkg/auth/token.go#L943):

```json
{
  "resource": "https://my-mcp-server.example.com/",
  "authorization_servers": ["https://mcp-authserver.toolhive-system.svc.cluster.local"],
  "bearer_methods_supported": ["header"],
  "jwks_uri": "https://mcp-authserver.../.well-known/jwks.json",
  "scopes_supported": ["openid", "mcp"]
}
```

The `authorization_servers` field tells clients where the authserver is located.

#### Authserver OIDC Discovery (Step 6)

```json
{
  "issuer": "https://mcp-authserver.toolhive-system.svc.cluster.local",
  "authorization_endpoint": "https://mcp-authserver.../oauth/authorize",
  "token_endpoint": "https://mcp-authserver.../oauth/token",
  "jwks_uri": "https://mcp-authserver.../.well-known/jwks.json",
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code", "refresh_token"],
  "code_challenge_methods_supported": ["S256"],
  "subject_types_supported": ["public"],
  "id_token_signing_alg_values_supported": ["RS256"]
}
```

#### How ProxyRunner Exposes `/.well-known/oauth-protected-resource`

**Existing wiring (already implemented):**

1. **CRD Configuration** ([cmd/thv-operator/api/v1alpha1/mcpserver_types.go:408-432](cmd/thv-operator/api/v1alpha1/mcpserver_types.go#L408-L432)):
   ```yaml
   apiVersion: toolhive.stacklok.dev/v1alpha1
   kind: MCPServer
   spec:
     oidcConfig:
       type: inline
       resourceUrl: "https://my-mcp-server.example.com/"  # For protected resource metadata
       inline:
         issuer: "https://mcp-authserver.toolhive-system.svc.cluster.local"
         audience: "my-mcp-server"
   ```

2. **Auth middleware creation** ([pkg/auth/utils.go:63-77](pkg/auth/utils.go#L63-L77)):
   ```go
   func GetAuthenticationMiddleware(ctx context.Context, oidcConfig *TokenValidatorConfig) (...) {
       jwtValidator, err := NewTokenValidator(ctx, *oidcConfig)
       // Creates the handler that returns RFC 9728 metadata
       authInfoHandler := NewAuthInfoHandler(
           oidcConfig.Issuer,          // → "authorization_servers" field
           jwtValidator.jwksURL,       // → "jwks_uri" field
           oidcConfig.ResourceURL,     // → "resource" field
           oidcConfig.Scopes,          // → "scopes_supported" field
       )
       return jwtValidator.Middleware, authInfoHandler, nil
   }
   ```

3. **Handler returns metadata** ([pkg/auth/token.go:943-993](pkg/auth/token.go#L943-L993)):
   ```go
   func NewAuthInfoHandler(issuer, jwksURL, resourceURL string, scopes []string) http.Handler {
       return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
           authInfo := RFC9728AuthInfo{
               Resource:             resourceURL,
               AuthorizationServers: []string{issuer},  // Points to authserver!
               BearerMethodsSupported: []string{"header"},
               JWKSURI:              jwksURL,
               ScopesSupported:      scopes,
           }
           json.NewEncoder(w).Encode(authInfo)
       })
   }
   ```

4. **Proxy wires up endpoint** ([pkg/transport/proxy/transparent/transparent_proxy.go:450-454](pkg/transport/proxy/transparent/transparent_proxy.go#L450-L454)):
   ```go
   // In Start() method
   if wellKnownHandler := auth.NewWellKnownHandler(p.authInfoHandler); wellKnownHandler != nil {
       mux.Handle("/.well-known/", wellKnownHandler)
       logger.Info("RFC 9728 OAuth discovery endpoints enabled")
   }
   ```

**What's already working:**
- ProxyRunner exposes `/.well-known/oauth-protected-resource` ✅
- Returns `authorization_servers` pointing to issuer ✅
- Clients can discover where to authenticate ✅

**What needs to be added for standalone authserver:**
- Currently the `issuer` in OIDC config points to the external IDP (e.g., Google)
- For standalone authserver, the `issuer` would point to the authserver instead
- The authserver then federates to the upstream IDP internally

---

### 3. Mutual Authentication Design

#### Why mTLS Between Proxyrunner and Authserver?

The authserver stores **upstream IDP tokens** (access tokens, refresh tokens) and links them to the JWTs it issues via the `tsid` (token session ID) claim. When a proxyrunner receives a client request with an authserver-issued JWT, it may need to:

1. **Retrieve the upstream access token** to pass to backend MCP servers that require the original IDP token
2. **Exchange tokens** (RFC 8693) to get a backend-specific token

Note: The proxyrunner doesn't refresh client JWTs - clients refresh their authserver-issued JWTs directly via the authserver's `/oauth/token` endpoint with `grant_type=refresh_token`. (This endpoint is currently TODO and needs implementation.) When the proxyrunner calls the token exchange endpoint, the authserver internally refreshes expired upstream IDP tokens if needed.

This is sensitive because:
- **Upstream tokens are valuable** - they grant access to external services (Google, GitHub, etc.)
- **Only authorized proxyrunners should access them** - a compromised or rogue service shouldn't be able to request tokens for arbitrary sessions
- **Token binding** - the authserver needs to verify that the proxyrunner requesting tokens is the one the client intended to use

**mTLS provides strong mutual authentication:**
- Authserver knows which proxyrunner is making the request (certificate identity)
- Proxyrunner can't be impersonated without the private key
- No shared secrets to manage or rotate (cert-manager handles lifecycle)

#### Certificate Trust Chain

Each MCPServer gets its own client certificate. This allows the authserver to identify which proxyrunner is making requests (useful for audit logging and access control).

```
                     ┌─────────────────────┐
                     │   cert-manager CA   │
                     │   (ClusterIssuer)   │
                     │  "toolhive-mtls-ca" │
                     └─────────┬───────────┘
                               │ signs all certs
                               │
      ┌────────────────────────┼────────────────────────┐
      │                        │                        │
      ▼                        ▼                        ▼
┌───────────────┐      ┌───────────────┐      ┌───────────────┐
│  AuthServer   │      │ MCPServer "A" │      │ MCPServer "B" │
│  Server Cert  │      │  Client Cert  │      │  Client Cert  │
│ (server auth) │      │ (client auth) │      │ (client auth) │
│               │      │               │      │               │
│ CN: mcp-auth  │      │ CN: server-a  │      │ CN: server-b  │
│     server... │      │ URI: spiffe://│      │ URI: spiffe://│
│               │      │  .../server-a │      │  .../server-b │
└───────────────┘      └───────────────┘      └───────────────┘
        │                      │                      │
        │                      │                      │
        └──────────────────────┴──────────────────────┘
                          mTLS connection
                    (proxyrunners → authserver)
```

**Why per-MCPServer certificates:**
- **Identity**: Authserver knows exactly which MCP server is requesting tokens
- **Audit**: Logs can show "MCPServer 'github-tools' retrieved token for session X"
- **Access control**: Could restrict which proxyrunners can access which sessions
- **Revocation**: Can revoke a single proxyrunner's access without affecting others

#### How Proxyrunner Verifies Authserver Identity

Before sending sensitive data (client JWTs, session IDs), the proxyrunner must verify it's connecting to a legitimate authserver:

1. **Server Certificate Verification**:
   - Authserver presents its server certificate during TLS handshake
   - Proxyrunner validates the certificate chain against the CA bundle (from `authServerClientConfig.caBundle`)
   - Certificate must have DNS names matching the authserver URL

2. **Trust Establishment**:
   - Both use the same cert-manager CA (`toolhive-mtls-ca`)
   - Proxyrunner's CA bundle contains the CA certificate that signed the authserver's cert
   - If verification fails, connection is rejected (prevents MITM attacks)

#### How Proxyrunner Authenticates to Authserver

1. **Client Certificate Identity (SPIFFE URI)**:
   ```
   CN=github-tools                                              # Human-readable for audit logs
   URI SAN=spiffe://toolhive.local/ns/mcp-servers/mcpserver/github-tools
   ```

   The SPIFFE URI encodes:
   - **Trust domain**: `toolhive.local` (organization-specific)
   - **Namespace**: `mcp-servers` (Kubernetes namespace)
   - **MCPServer name**: `github-tools`

2. **mTLS Handshake**:
   - Proxyrunner presents client certificate signed by shared CA
   - Authserver verifies certificate chain against trusted CA
   - Authserver extracts identity from SPIFFE URI SAN

3. **Identity Binding**:
   - Authserver binds proxyrunner identity to token exchange requests
   - Issued tokens include proxyrunner identity in claims

---

### 4. Cert-Manager Integration

#### CA Secret (Root of Trust)

**Option 1: Cert-manager self-signed CA (dev/test)**

For development and testing, let cert-manager generate a self-signed CA:

```yaml
# Bootstrap issuer (self-signed)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: toolhive-selfsigned-bootstrap
spec:
  selfSigned: {}
---
# CA Certificate (generated by bootstrap issuer)
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: toolhive-mtls-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: "ToolHive mTLS CA"
  subject:
    organizations:
      - ToolHive
    organizationalUnits:
      - Platform
  secretName: toolhive-mtls-ca-keypair
  duration: 87600h    # 10 years
  renewBefore: 8760h  # Renew 1 year before expiry
  privateKey:
    algorithm: RSA
    size: 4096
  issuerRef:
    name: toolhive-selfsigned-bootstrap
    kind: ClusterIssuer
```

**Option 2: External CA (production)**

For production, use an enterprise PKI or cloud-based CA service:

```yaml
# Example: HashiCorp Vault as CA
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: toolhive-mtls-ca
spec:
  vault:
    server: https://vault.example.com
    path: pki/sign/toolhive-mtls
    auth:
      kubernetes:
        role: cert-manager
        mountPath: /v1/auth/kubernetes
        secretRef:
          name: vault-token
          key: token
---
# Example: AWS Private CA
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: toolhive-mtls-ca
spec:
  acmPCA:
    arn: arn:aws:acm-pca:us-east-1:123456789:certificate-authority/abc-123
    region: us-east-1
```

Production considerations:
- Store CA private key in HSM or managed service (Vault, AWS Private CA, Google Cloud CA)
- Use short-lived certificates (90 days) with automatic renewal
- Implement certificate revocation (CRL or OCSP)
- Audit certificate issuance

#### ClusterIssuer for mTLS

Once the CA secret exists, create the ClusterIssuer that will sign all mTLS certificates:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: toolhive-mtls-ca
spec:
  ca:
    secretName: toolhive-mtls-ca-keypair
```

#### Authserver Server Certificate

The authserver's server certificate is **automatically generated by the MCPAuthServer controller** based on the `tls.serverCert` configuration. See **Section 1: MCPAuthServer CRD** for the configuration.

#### ProxyRunner Client Certificate (per MCPServer)

Client certificates for proxyrunners are **automatically generated by the MCPServer controller** when `authServerClientConfig.clientCert` is specified in the MCPServer CRD. See **Section 1: MCPServer CRD Updates** for the configuration and controller behavior.

#### HttpClientBuilder Extension

Extend [pkg/networking/http_client.go](pkg/networking/http_client.go):

```go
// Add to HttpClientBuilder
func (b *HttpClientBuilder) WithClientCertificate(certPath, keyPath string) *HttpClientBuilder {
    b.clientCertPath = certPath
    b.clientKeyPath = keyPath
    return b
}

// In Build() method
if b.clientCertPath != "" && b.clientKeyPath != "" {
    cert, err := tls.LoadX509KeyPair(b.clientCertPath, b.clientKeyPath)
    if err != nil {
        return nil, fmt.Errorf("failed to load client certificate: %w", err)
    }
    transport.TLSClientConfig.Certificates = []tls.Certificate{cert}
}
```

---

### 5. Token Flow

#### Overview: Why Token Exchange is Needed

When a client authenticates, the authserver:
1. Redirects to upstream IDP (Google, GitHub, etc.)
2. Receives upstream tokens (access token, refresh token, ID token)
3. **Stores these tokens** linked to a session ID (`tsid`)
4. Issues its own JWT containing the `tsid` claim

The client's JWT **does not contain the upstream tokens** - only a reference to them. When the proxyrunner needs to call a backend that requires the upstream token (e.g., a GitHub MCP server needs a GitHub access token), it must:
1. Extract `tsid` from the client's JWT
2. Call authserver to retrieve the upstream access token
3. Use that token to authenticate to the backend

**Token refresh responsibilities:**
- **Client JWT refresh**: The client is responsible for refreshing its authserver-issued JWT by calling the authserver's `/oauth/token` endpoint with `grant_type=refresh_token`. The proxyrunner returns 401 if the JWT is expired.
- **Upstream IDP token refresh**: The authserver automatically refreshes expired upstream tokens internally when the proxyrunner calls the token exchange endpoint. The proxyrunner never handles upstream refresh directly.

#### Complete Flow with Upstream Token Retrieval

```
┌─────────────┐     ┌─────────────────┐     ┌──────────────────┐     ┌──────────┐
│   Client    │     │   ProxyRunner   │     │    AuthServer    │     │ Upstream │
│   (Agent)   │     │                 │     │                  │     │   IDP    │
└──────┬──────┘     └────────┬────────┘     └────────┬─────────┘     └────┬─────┘
       │                     │                       │                    │
       │ == INITIAL AUTH (via discovery flow above) ========================
       │                     │                       │                    │
       │                     │                       │ 1. User auth flow  │
       │                     │                       │◄──────────────────►│
       │                     │                       │                    │
       │                     │                       │ 2. Authserver receives & STORES
       │                     │                       │    upstream tokens (linked to tsid)
       │                     │                       │                    │
       │◄────────────────────┼───────────────────────│                    │
       │ 3. Authserver JWT { sub, tsid, ... }        │                    │
       │    (tsid links to stored upstream tokens)   │                    │
       │                     │                       │                    │
       │ == SUBSEQUENT MCP REQUESTS =========================================
       │                     │                       │                    │
       │ 4. MCP Request      │                       │                    │
       │    + JWT            │                       │                    │
       │────────────────────►│                       │                    │
       │                     │                       │                    │
       │                     │ 5. Validate JWT,      │                    │
       │                     │    extract tsid       │                    │
       │                     │                       │                    │
       │                     │ 6. GET /internal/tokens/{tsid}             │
       │                     │    (mTLS client cert authenticates         │
       │                     │     proxyrunner identity)                  │
       │                     │───────────────────────►                    │
       │                     │                       │                    │
       │                     │ 7. Authserver verifies:                    │
       │                     │    - mTLS cert is valid proxyrunner        │
       │                     │    - tsid exists and is not expired        │
       │                     │    - (optional) proxyrunner is authorized  │
       │                     │      for this session                      │
       │                     │                       │                    │
       │                     │◄──────────────────────│                    │
       │                     │ 8. Upstream access token                   │
       │                     │    (the actual IDP token)                  │
       │                     │                       │                    │
       │                     │ 9. Call backend MCP server                 │
       │                     │    with upstream token │                   │
       │                     │───────────────────────┼───────────────────►│
       │                     │                       │                    │
       │◄────────────────────│                       │                    │
       │ 10. MCP Response    │                       │                    │
```

#### 5.1 MCPAuthServer: Token Exchange Endpoint

The authserver exposes an internal endpoint for proxyrunners to exchange client JWTs for upstream access tokens.

```go
// pkg/authserver/server/handlers/token_exchange.go

import (
    "github.com/spiffe/go-spiffe/v2/spiffeid"
)

// ProxyRunnerIdentity represents the identity extracted from a proxyrunner's mTLS client certificate
type ProxyRunnerIdentity struct {
    // SpiffeID is the parsed SPIFFE ID from the certificate URI SAN
    SpiffeID spiffeid.ID
    // TrustDomain is the SPIFFE trust domain (e.g., "toolhive.local")
    TrustDomain spiffeid.TrustDomain
    // Namespace is the Kubernetes namespace (from SPIFFE URI path)
    Namespace string
    // Name is the MCPServer name (from SPIFFE URI path)
    Name string
    // CommonName is the human-readable CN (for audit logs)
    CommonName string
    // CertificateSerial is the certificate serial number (for audit logging)
    CertificateSerial string
}

// extractProxyRunnerIdentity extracts the proxyrunner identity from the mTLS client certificate.
// The certificate must contain a SPIFFE URI SAN in the format:
// spiffe://{trust.domain}/ns/{namespace}/mcpserver/{name}
//
// Returns an error if no client certificate is present or if the certificate doesn't have
// a valid SPIFFE URI SAN.
func extractProxyRunnerIdentity(r *http.Request) (*ProxyRunnerIdentity, error) {
    // Check for TLS connection
    if r.TLS == nil {
        return nil, errors.New("connection is not TLS")
    }

    // Check for peer (client) certificates
    if len(r.TLS.PeerCertificates) == 0 {
        return nil, errors.New("no client certificate provided")
    }

    // Use the leaf certificate (first in chain)
    cert := r.TLS.PeerCertificates[0]

    // Find and parse SPIFFE ID from certificate URI SANs
    var spiffeID spiffeid.ID
    var found bool
    for _, uri := range cert.URIs {
        if uri.Scheme == "spiffe" {
            var err error
            spiffeID, err = spiffeid.FromURI(uri)
            if err != nil {
                return nil, fmt.Errorf("invalid SPIFFE URI in certificate: %w", err)
            }
            found = true
            break
        }
    }

    if !found {
        return nil, errors.New("client certificate missing SPIFFE URI SAN")
    }

    // Parse path segments: /ns/{namespace}/mcpserver/{name}
    // spiffeid.ID.Path() returns the path without leading slash
    pathSegments := strings.Split(spiffeID.Path(), "/")
    if len(pathSegments) != 4 || pathSegments[0] != "ns" || pathSegments[2] != "mcpserver" {
        return nil, fmt.Errorf("invalid SPIFFE ID path format: expected ns/{namespace}/mcpserver/{name}, got %s", spiffeID.Path())
    }

    namespace := pathSegments[1]
    mcpServerName := pathSegments[3]

    return &ProxyRunnerIdentity{
        SpiffeID:          spiffeID,
        TrustDomain:       spiffeID.TrustDomain(),
        Namespace:         namespace,
        Name:              mcpServerName,
        CommonName:        cert.Subject.CommonName, // For audit logging
        CertificateSerial: cert.SerialNumber.String(),
    }, nil
}

// Handler struct update (in pkg/authserver/server/handlers/handler.go)
// Add subjectValidator field for mTLS access control
type Handler struct {
    fositeProvider fosite.OAuth2Provider
    config         *server.AuthorizationServerConfig
    storage        storage.Storage
    upstreamIdP    upstream.OAuth2Provider

    // NEW: Validator for proxyrunner mTLS client certificates
    // Initialized from MCPAuthServer CRD's tls.clientAuth.allowedSubjects
    subjectValidator *SubjectValidator
}

// NewHandler initialization (in pkg/authserver/server/handlers/handler.go)
// Updated to accept mTLS configuration
func NewHandler(
    fositeProvider fosite.OAuth2Provider,
    config *server.AuthorizationServerConfig,
    storage storage.Storage,
    upstreamIdP upstream.OAuth2Provider,
    allowedSubjects *AllowedSubjects,  // NEW: from MCPAuthServer CRD
) (*Handler, error) {
    // Create subject validator from CRD config
    subjectValidator, err := NewSubjectValidator(allowedSubjects)
    if err != nil {
        return nil, fmt.Errorf("invalid allowedSubjects config: %w", err)
    }

    return &Handler{
        fositeProvider:   fositeProvider,
        config:           config,
        storage:          storage,
        upstreamIdP:      upstreamIdP,
        subjectValidator: subjectValidator,
    }, nil
}

// SubjectValidator validates client certificate SPIFFE IDs against allowed patterns
type SubjectValidator struct {
    // trustDomain is the required SPIFFE trust domain
    trustDomain spiffeid.TrustDomain
    // allowedNamespaces is a set of allowed Kubernetes namespaces (nil means all allowed)
    allowedNamespaces map[string]bool
    // allowedNames is a set of allowed MCPServer names (nil means all allowed)
    allowedNames map[string]bool
}

// NewSubjectValidator creates a validator from MCPAuthServer CRD configuration
func NewSubjectValidator(allowedSubjects *AllowedSubjects) (*SubjectValidator, error) {
    if allowedSubjects == nil {
        // No restrictions - all valid certificates are allowed
        return &SubjectValidator{}, nil
    }

    // Parse and validate trust domain
    td, err := spiffeid.TrustDomainFromString(allowedSubjects.TrustDomain)
    if err != nil {
        return nil, fmt.Errorf("invalid trustDomain %q: %w", allowedSubjects.TrustDomain, err)
    }

    validator := &SubjectValidator{
        trustDomain: td,
    }

    // Build allowed namespaces set
    if len(allowedSubjects.AllowedNamespaces) > 0 {
        validator.allowedNamespaces = make(map[string]bool, len(allowedSubjects.AllowedNamespaces))
        for _, ns := range allowedSubjects.AllowedNamespaces {
            validator.allowedNamespaces[ns] = true
        }
    }

    // Build allowed names set
    if len(allowedSubjects.AllowedNames) > 0 {
        validator.allowedNames = make(map[string]bool, len(allowedSubjects.AllowedNames))
        for _, name := range allowedSubjects.AllowedNames {
            validator.allowedNames[name] = true
        }
    }

    return validator, nil
}

// validateSubjectAllowed checks if a proxyrunner identity is allowed based on
// the allowedSubjects configuration from the MCPAuthServer CRD.
//
// Validates:
// 1. SPIFFE trust domain matches the configured trust domain
// 2. Namespace is in the allowed list (if configured)
// 3. MCPServer name is in the allowed list (if configured)
//
// Returns nil if allowed, error if rejected.
func (v *SubjectValidator) validateSubjectAllowed(identity *ProxyRunnerIdentity) error {
    // If no trust domain configured, allow all
    if v.trustDomain.IsZero() {
        return nil
    }

    // Check trust domain matches
    if !identity.TrustDomain.Equal(v.trustDomain) {
        return fmt.Errorf("trust domain %q does not match required %q",
            identity.TrustDomain, v.trustDomain)
    }

    // Check namespace restriction (if configured)
    if v.allowedNamespaces != nil {
        if !v.allowedNamespaces[identity.Namespace] {
            return fmt.Errorf("namespace %q is not in allowed list", identity.Namespace)
        }
    }

    // Check name restriction (if configured)
    if v.allowedNames != nil {
        if !v.allowedNames[identity.Name] {
            return fmt.Errorf("MCPServer name %q is not in allowed list", identity.Name)
        }
    }

    return nil
}

// validateSessionAudience verifies that the proxyrunner is authorized to access this session.
// The session's audience (from the JWT) should match the MCPServer identified by the SPIFFE ID.
//
// This prevents a compromised proxyrunner in namespace A from requesting tokens for sessions
// that were intended for a different MCPServer in namespace B.
//
// Audience matching rules:
// 1. If JWT has "aud" claim, it must match the proxyrunner's MCPServer name
// 2. The SPIFFE ID namespace/name combination provides additional binding
func (h *Handler) validateSessionAudience(claims map[string]interface{}, identity *ProxyRunnerIdentity) error {
    // Extract audience from JWT claims
    // Audience can be a string or array of strings
    var audiences []string
    switch aud := claims["aud"].(type) {
    case string:
        audiences = []string{aud}
    case []interface{}:
        for _, a := range aud {
            if s, ok := a.(string); ok {
                audiences = append(audiences, s)
            }
        }
    case nil:
        // No audience claim - skip validation
        // This is acceptable for tokens that don't specify a target MCPServer
        return nil
    default:
        return fmt.Errorf("unexpected audience claim type: %T", aud)
    }

    if len(audiences) == 0 {
        return nil // No audience restriction
    }

    // Build expected audience values for this proxyrunner
    // Accept either:
    // - Just the MCPServer name: "github-tools"
    // - Fully qualified: "github-tools.mcp-servers" (name.namespace)
    // - SPIFFE ID: "spiffe://toolhive.local/ns/mcp-servers/mcpserver/github-tools"
    // - Service URL format: "https://github-tools.mcp-servers.svc.cluster.local"
    expectedAudiences := map[string]bool{
        identity.Name: true,
        fmt.Sprintf("%s.%s", identity.Name, identity.Namespace): true,
        identity.SpiffeID.String(): true,
    }

    // Check if any JWT audience matches expected
    for _, aud := range audiences {
        // Direct match
        if expectedAudiences[aud] {
            return nil
        }
        // URL-based match: extract hostname and compare
        if u, err := url.Parse(aud); err == nil && u.Host != "" {
            hostname := strings.Split(u.Host, ".")[0]
            if hostname == identity.Name {
                return nil
            }
        }
    }

    return fmt.Errorf("token audience %v does not match proxyrunner SPIFFE ID %s",
        audiences, identity.SpiffeID)
}

func (h *Handler) TokenExchangeHandler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    // 1. Extract proxyrunner identity from mTLS client cert SPIFFE ID
    identity, err := extractProxyRunnerIdentity(r)
    if err != nil {
        logger.Warnf("SPIFFE identity extraction failed: %v", err)
        h.writeError(w, fosite.ErrAccessDenied.WithHint("mTLS client certificate with SPIFFE URI required"))
        return
    }

    logger.Infof("Token exchange request from proxyrunner %s (cert serial: %s)",
        identity.SpiffeID, identity.CertificateSerial)

    // 2. Validate proxyrunner SPIFFE ID is allowed based on allowedSubjects from MCPAuthServer CRD
    //    h.subjectValidator is initialized from config at startup
    if err := h.subjectValidator.validateSubjectAllowed(identity); err != nil {
        logger.Warnf("Proxyrunner %s rejected by subject policy: %v",
            identity.SpiffeID, err)
        h.writeError(w, fosite.ErrAccessDenied.WithHintf(
            "proxyrunner not allowed: %s", err.Error()))
        return
    }

    // 3. Validate incoming client JWT
    clientToken := r.FormValue("subject_token")
    if clientToken == "" {
        h.writeError(w, fosite.ErrInvalidRequest.WithHint("subject_token required"))
        return
    }

    claims, err := h.validateClientToken(ctx, clientToken)
    if err != nil {
        logger.Warnf("Client token validation failed: %v", err)
        h.writeError(w, fosite.ErrInvalidGrant.WithHint("invalid subject_token"))
        return
    }

    // 4. Extract session ID from JWT's tsid claim
    tsid, ok := claims["tsid"].(string)
    if !ok || tsid == "" {
        h.writeError(w, fosite.ErrInvalidGrant.WithHint("token missing tsid claim"))
        return
    }

    // 5. Retrieve upstream tokens using session ID
    upstreamTokens, err := h.storage.GetUpstreamTokens(ctx, tsid)
    if errors.Is(err, storage.ErrExpired) {
        // 5a. Tokens expired - attempt refresh using stored refresh token
        upstreamTokens, err = h.refreshUpstreamTokens(ctx, tsid)
        if err != nil {
            logger.Warnf("Failed to refresh upstream tokens for tsid %s: %v", tsid, err)
            h.writeError(w, fosite.ErrServerError.WithHint("upstream token refresh failed"))
            return
        }
    } else if err != nil {
        logger.Warnf("Failed to retrieve upstream tokens for tsid %s: %v", tsid, err)
        h.writeError(w, fosite.ErrInvalidRequest.WithHint("session not found"))
        return
    }

    // 6. Verify proxyrunner is authorized for this specific session
    //    The session's audience should match the MCPServer identified by the SPIFFE ID
    if err := h.validateSessionAudience(claims, identity); err != nil {
        logger.Warnf("Session audience mismatch for proxyrunner %s: %v",
            identity.SpiffeID, err)
        h.writeError(w, fosite.ErrAccessDenied.WithHintf(
            "proxyrunner not authorized for this session: %s", err.Error()))
        return
    }

    // 7. Return the upstream access token to the proxyrunner
    h.writeTokenResponse(w, &TokenExchangeResponse{
        AccessToken: upstreamTokens.AccessToken,
        TokenType:   "Bearer",
        ExpiresIn:   int(time.Until(upstreamTokens.ExpiresAt).Seconds()),
        // Don't return refresh token - proxyrunner should call this endpoint again
    })
}

type TokenExchangeResponse struct {
    AccessToken string `json:"access_token"`
    TokenType   string `json:"token_type"`
    ExpiresIn   int    `json:"expires_in,omitempty"`
}

// refreshUpstreamTokens refreshes expired upstream IDP tokens.
// Uses the stored refresh token to get new tokens from the upstream IDP.
func (h *Handler) refreshUpstreamTokens(ctx context.Context, tsid string) (*storage.UpstreamTokens, error) {
    // Get stored tokens (including refresh token) - bypass expiry check
    storedTokens, err := h.storage.GetUpstreamTokensForRefresh(ctx, tsid)
    if err != nil {
        return nil, fmt.Errorf("failed to get stored tokens: %w", err)
    }

    if storedTokens.RefreshToken == "" {
        return nil, errors.New("no refresh token available")
    }

    // Call upstream IDP to refresh tokens
    newTokens, err := h.upstreamIdP.RefreshTokens(ctx, storedTokens.RefreshToken)
    if err != nil {
        return nil, fmt.Errorf("upstream refresh failed: %w", err)
    }

    // Update stored tokens with refreshed values
    updatedTokens := &storage.UpstreamTokens{
        ProviderID:      storedTokens.ProviderID,
        AccessToken:     newTokens.AccessToken,
        RefreshToken:    newTokens.RefreshToken, // May be rotated
        IDToken:         newTokens.IDToken,
        ExpiresAt:       newTokens.ExpiresAt,
        UserID:          storedTokens.UserID,
        UpstreamSubject: storedTokens.UpstreamSubject,
        ClientID:        storedTokens.ClientID,
    }

    if err := h.storage.StoreUpstreamTokens(ctx, tsid, updatedTokens); err != nil {
        return nil, fmt.Errorf("failed to store refreshed tokens: %w", err)
    }

    logger.Infow("upstream tokens refreshed", "tsid", tsid)
    return updatedTokens, nil
}
```

#### 5.2 MCPServer: AuthServer Client

The proxyrunner needs a client to call the authserver's token exchange endpoint with mTLS and caching.

**New package: `pkg/auth/authserver/` (or `pkg/runner/authserver/`)**

Location options:
- `pkg/auth/authserver/` - Keeps auth-related code together (recommended)
- `pkg/runner/authserver/` - Closer to where proxyrunner uses it
- `pkg/authserver/client/` - Keeps authserver code together but creates circular dependency risk

```go
// pkg/auth/authserver/client.go (recommended location)

// AuthServerClient handles communication with the authserver for token exchange
type AuthServerClient struct {
    httpClient   *http.Client
    baseURL      string
    tokenCache   *TokenCache
    mu           sync.RWMutex
}

// Config holds configuration for the authserver client
type Config struct {
    // URL is the authserver base URL
    URL string
    // ClientCertPath is the path to the client certificate for mTLS
    ClientCertPath string
    // ClientKeyPath is the path to the client key for mTLS
    ClientKeyPath string
    // CABundlePath is the path to the CA bundle for verifying authserver
    CABundlePath string
    // CacheTTL is how long to cache tokens (default: 80% of token lifetime)
    CacheTTL time.Duration
}

// NewAuthServerClient creates a new client with mTLS configuration
func NewAuthServerClient(cfg *Config) (*AuthServerClient, error) {
    // Build HTTP client with mTLS using existing HttpClientBuilder
    clientBuilder := networking.NewHttpClientBuilder()

    // Add CA bundle for verifying authserver's server certificate
    // This ensures we only connect to an authserver with a cert signed by the trusted CA
    if cfg.CABundlePath != "" {
        clientBuilder = clientBuilder.WithCABundle(cfg.CABundlePath)
    }

    // Add client certificate for mTLS (authserver verifies this)
    if cfg.ClientCertPath != "" && cfg.ClientKeyPath != "" {
        clientBuilder = clientBuilder.WithClientCertificate(cfg.ClientCertPath, cfg.ClientKeyPath)
    }

    httpClient, err := clientBuilder.Build()
    if err != nil {
        return nil, fmt.Errorf("failed to create HTTP client: %w", err)
    }

    return &AuthServerClient{
        httpClient: httpClient,
        baseURL:    strings.TrimSuffix(cfg.URL, "/"),
        tokenCache: NewTokenCache(cfg.CacheTTL),
    }, nil
}

// ExchangeToken exchanges a client JWT for an upstream access token.
// Results are cached to avoid repeated calls for the same session.
func (c *AuthServerClient) ExchangeToken(ctx context.Context, clientJWT string) (*TokenExchangeResult, error) {
    // Extract tsid from JWT for cache key (without validating - authserver will validate)
    tsid, err := extractTSIDFromJWT(clientJWT)
    if err != nil {
        return nil, fmt.Errorf("failed to extract tsid from JWT: %w", err)
    }

    // Check cache first
    if cached := c.tokenCache.Get(tsid); cached != nil {
        // Verify the token hasn't expired (with 30s buffer)
        if !cached.IsExpired() {
            logger.Debugf("Token cache hit for tsid %s", tsid)
            return cached, nil
        }
        logger.Debugf("Token cache hit but token expired for tsid %s", tsid)
    }

    // Cache miss - call authserver
    logger.Debugf("Token cache miss for tsid %s, calling authserver", tsid)

    result, err := c.doTokenExchange(ctx, clientJWT)
    if err != nil {
        return nil, err
    }

    // Cache the result (with TTL based on token expiry)
    c.tokenCache.Set(tsid, result)

    return result, nil
}

// doTokenExchange makes the actual HTTP request to the authserver
func (c *AuthServerClient) doTokenExchange(ctx context.Context, clientJWT string) (*TokenExchangeResult, error) {
    // Build request body
    form := url.Values{}
    form.Set("grant_type", "urn:ietf:params:oauth:grant-type:token-exchange")
    form.Set("subject_token", clientJWT)
    form.Set("subject_token_type", "urn:ietf:params:oauth:token-type:jwt")

    req, err := http.NewRequestWithContext(ctx, http.MethodPost,
        c.baseURL+"/internal/token-exchange",
        strings.NewReader(form.Encode()))
    if err != nil {
        return nil, fmt.Errorf("failed to create request: %w", err)
    }

    req.Header.Set("Content-Type", "application/x-www-form-urlencoded")

    // TLS handshake verifies authserver's certificate against CA bundle (configured in NewAuthServerClient)
    // and presents client certificate for mTLS authentication
    resp, err := c.httpClient.Do(req)
    if err != nil {
        // Fails if: server cert invalid, client cert rejected, or connection error
        return nil, fmt.Errorf("token exchange request failed: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        body, _ := io.ReadAll(io.LimitReader(resp.Body, 1024))
        return nil, fmt.Errorf("token exchange failed with status %d: %s",
            resp.StatusCode, string(body))
    }

    var result TokenExchangeResult
    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return nil, fmt.Errorf("failed to decode response: %w", err)
    }

    // Compute absolute expiry time
    if result.ExpiresIn > 0 {
        result.ExpiresAt = time.Now().Add(time.Duration(result.ExpiresIn) * time.Second)
    }

    return &result, nil
}

// extractTSIDFromJWT extracts the tsid claim from a JWT without full validation
// (authserver will perform full validation)
func extractTSIDFromJWT(token string) (string, error) {
    parts := strings.Split(token, ".")
    if len(parts) != 3 {
        return "", errors.New("invalid JWT format")
    }

    // Decode payload (middle part)
    payload, err := base64.RawURLEncoding.DecodeString(parts[1])
    if err != nil {
        return "", fmt.Errorf("failed to decode JWT payload: %w", err)
    }

    var claims struct {
        TSID string `json:"tsid"`
    }
    if err := json.Unmarshal(payload, &claims); err != nil {
        return "", fmt.Errorf("failed to parse JWT claims: %w", err)
    }

    if claims.TSID == "" {
        return "", errors.New("JWT missing tsid claim")
    }

    return claims.TSID, nil
}

// TokenExchangeResult holds the result of a token exchange
type TokenExchangeResult struct {
    AccessToken string    `json:"access_token"`
    TokenType   string    `json:"token_type"`
    ExpiresIn   int       `json:"expires_in,omitempty"`
    ExpiresAt   time.Time `json:"-"` // Computed from ExpiresIn
}

// IsExpired checks if the token has expired (with 30s buffer)
func (r *TokenExchangeResult) IsExpired() bool {
    return time.Now().Add(30 * time.Second).After(r.ExpiresAt)
}
```

**Token Cache Implementation:**

```go
// pkg/auth/authserver/cache.go

// TokenCache provides thread-safe caching of exchanged tokens
type TokenCache struct {
    cache    map[string]*cacheEntry
    mu       sync.RWMutex
    defaultTTL time.Duration
}

type cacheEntry struct {
    result    *TokenExchangeResult
    expiresAt time.Time
}

func NewTokenCache(defaultTTL time.Duration) *TokenCache {
    if defaultTTL == 0 {
        defaultTTL = 5 * time.Minute // Conservative default
    }
    cache := &TokenCache{
        cache:      make(map[string]*cacheEntry),
        defaultTTL: defaultTTL,
    }
    // Start background cleanup goroutine
    go cache.cleanupLoop()
    return cache
}

// Get retrieves a cached token, returning nil if not found or expired
func (c *TokenCache) Get(tsid string) *TokenExchangeResult {
    c.mu.RLock()
    defer c.mu.RUnlock()

    entry, ok := c.cache[tsid]
    if !ok {
        return nil
    }

    // Check if expired
    if time.Now().After(entry.expiresAt) {
        return nil
    }

    return entry.result
}

// Set stores a token in the cache
// TTL is set to 80% of the token's remaining lifetime, or defaultTTL if no expiry
func (c *TokenCache) Set(tsid string, result *TokenExchangeResult) {
    c.mu.Lock()
    defer c.mu.Unlock()

    var ttl time.Duration
    if !result.ExpiresAt.IsZero() {
        // Use 80% of remaining lifetime to refresh before expiry
        remaining := time.Until(result.ExpiresAt)
        ttl = time.Duration(float64(remaining) * 0.8)
    }
    if ttl <= 0 {
        ttl = c.defaultTTL
    }

    c.cache[tsid] = &cacheEntry{
        result:    result,
        expiresAt: time.Now().Add(ttl),
    }
}

// cleanupLoop periodically removes expired entries
func (c *TokenCache) cleanupLoop() {
    ticker := time.NewTicker(1 * time.Minute)
    defer ticker.Stop()

    for range ticker.C {
        c.cleanup()
    }
}

func (c *TokenCache) cleanup() {
    c.mu.Lock()
    defer c.mu.Unlock()

    now := time.Now()
    for tsid, entry := range c.cache {
        if now.After(entry.expiresAt) {
            delete(c.cache, tsid)
        }
    }
}
```

**JWT Verification against MCPAuthServer JWKS:**

The existing auth middleware in [pkg/auth/token.go](pkg/auth/token.go) validates incoming JWTs against the authserver's JWKS. When `oidcConfig.issuer` points to the MCPAuthServer, the middleware fetches JWKS from `{issuer}/.well-known/jwks.json` and validates JWT signatures, issuer, audience, and expiration.

**Integration with tokenexchange middleware:**

After the auth middleware validates the JWT, the tokenexchange middleware exchanges it for upstream tokens:

```go
// pkg/auth/tokenexchange/middleware.go (updated)

func (m *Middleware) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // JWT already validated by auth middleware - extract from Authorization header
    subjectToken := extractBearerToken(r)

    // If authserver client is configured, use it to get upstream token
    if m.authServerClient != nil {
        result, err := m.authServerClient.ExchangeToken(r.Context(), subjectToken)
        if err != nil {
            logger.Errorf("Authserver token exchange failed: %v", err)
            // Fall back to direct token exchange if configured
            if m.directExchangeConfig != nil {
                // ... existing direct exchange logic ...
            } else {
                http.Error(w, "token exchange failed", http.StatusUnauthorized)
                return
            }
        } else {
            // Use the upstream token from authserver
            r.Header.Set("Authorization", "Bearer "+result.AccessToken)
        }
    }

    next.ServeHTTP(w, r)
}
```

#### RunConfig Extension

Extend [pkg/runner/config.go](pkg/runner/config.go):

```go
type RunConfig struct {
    // ... existing fields ...

    // AuthServerConfig configures connection to the authorization server
    AuthServerConfig *AuthServerClientConfig `json:"authserver_config,omitempty"`
}

type AuthServerClientConfig struct {
    URL            string `json:"url"`
    ClientCertPath string `json:"client_cert_path,omitempty"`
    ClientKeyPath  string `json:"client_key_path,omitempty"`
    CABundlePath   string `json:"ca_bundle_path,omitempty"`
}
```

---

## Key Files to Modify

### New Files

| File | Purpose |
|------|---------|
| `cmd/thv-authserver/main.go` | Authserver service entry point |
| `cmd/thv-authserver/app/serve.go` | Serve command with mTLS HTTP server |
| `cmd/thv-operator/api/v1alpha1/certmanager_types.go` | Shared types (CertManagerIssuerReference) |
| `cmd/thv-operator/api/v1alpha1/mcpauthserver_types.go` | MCPAuthServer CRD with mTLS config |
| `cmd/thv-operator/controllers/mcpauthserver_controller.go` | MCPAuthServer reconciler |
| `pkg/authserver/server/handlers/token_exchange.go` | Token exchange endpoint for proxyrunners |
| `pkg/authserver/server/handlers/subject_validator.go` | SPIFFE ID validation (allowedSubjects) |
| `pkg/auth/authserver/client.go` | Client for proxyrunner → authserver mTLS calls |
| `pkg/auth/authserver/cache.go` | Token cache for exchanged tokens |

### New Dependencies

| Package | Purpose |
|---------|---------|
| `github.com/spiffe/go-spiffe/v2/spiffeid` | SPIFFE ID parsing and validation for mTLS client certificates |

### Modified Files

| File | Changes |
|------|---------|
| `cmd/thv-operator/api/v1alpha1/mcpexternalauthconfig_types.go` | Add `authServer` type and `AuthServerConfig` struct for mTLS client certs |
| `cmd/thv-operator/controllers/mcpexternalauthconfig_controller.go` | Create cert-manager Certificate for proxyrunner when `type: authServer` |
| `cmd/thv-operator/controllers/mcpserver_controller.go` | Mount CA bundle and client cert Secret from MCPExternalAuthConfig |
| `pkg/networking/http_client.go` | Add `WithClientCertificate()` to HttpClientBuilder |
| `pkg/authserver/server/handlers/handler.go` | Add `subjectValidator` field, update `NewHandler()`, register token exchange route |
| `pkg/authserver/storage/types.go` | Add `GetUpstreamTokensForRefresh(ctx, tsid)` method (bypasses expiry check) |
| `pkg/authserver/storage/memory.go` | Implement `GetUpstreamTokensForRefresh()` |
| `pkg/runner/config.go` | Add `AuthServerConfig` struct |
| `pkg/auth/tokenexchange/middleware.go` | Integrate authserver client for token exchange |

---

## Implementation Phases

### Phase 1: MCPAuthServer CRD and Shared Types

**Shared Types:**

1. Create `certmanager_types.go` with shared types:
   - `CertManagerIssuerReference` (used by both MCPAuthServer and MCPExternalAuthConfig)

**MCPAuthServer CRD:**

2. Create `mcpauthserver_types.go` with CRD types:
   - `MCPAuthServerSpec` (issuer, replicas, upstreamIdps, signingKeys, tls)
   - `AuthServerTLSConfig` (serverCert, clientAuth)
   - `ServerCertConfig` (issuerRef, duration, renewBefore)
   - `ClientAuthConfig` (caBundle, allowedSubjects)
   - `AllowedSubjects` (trustDomain, allowedNamespaces, allowedNames) - uses SPIFFE URI patterns
3. Run `make generate` and `make manifests`

### Phase 2: Authserver Server-Side mTLS

**Handler Updates (`pkg/authserver/server/handlers/`):**

1. Create `subject_validator.go`:
   - `SubjectValidator` struct
   - `NewSubjectValidator()` function
   - `validateSubjectAllowed()` method
2. Update `handler.go`:
   - Add `subjectValidator` field to Handler
   - Update `NewHandler()` to accept `allowedSubjects`
3. Create `token_exchange.go`:
   - `ProxyRunnerIdentity` struct
   - `extractProxyRunnerIdentity()` function
   - `validateSessionAudience()` function
   - `TokenExchangeHandler()` handler
4. Update `handler.go` Routes() to register `/internal/token-exchange`

**Storage Updates (`pkg/authserver/storage/`):**

5. Add `GetUpstreamTokensForRefresh(ctx, tsid)` to `UpstreamTokenStorage` interface
6. Implement in `memory.go` (and later Redis)

**Unit Tests:**

7. Test `SubjectValidator` with various OU/CN patterns
8. Test `extractProxyRunnerIdentity()` with valid/invalid certs
9. Test `TokenExchangeHandler()` mTLS validation and token exchange
10. Test `GetUpstreamTokensForRefresh()` in storage

### Phase 3: HttpClientBuilder mTLS Support

**Networking Package (`pkg/networking/`):**

1. Add fields to `HttpClientBuilder`:
   - `clientCertPath string`
   - `clientKeyPath string`
2. Add `WithClientCertificate(certPath, keyPath)` method
3. Update `Build()` to load and configure client certificate:
   ```go
   cert, err := tls.LoadX509KeyPair(certPath, keyPath)
   transport.TLSClientConfig.Certificates = []tls.Certificate{cert}
   ```
4. Add unit tests for mTLS client configuration

### Phase 4: ProxyRunner Authserver Client

**New Package (`pkg/auth/authserver/`):**

1. Create `client.go`:
   - `Config` struct
   - `AuthServerClient` struct
   - `NewAuthServerClient()` constructor
   - `ExchangeToken()` with caching
   - `doTokenExchange()` HTTP implementation
   - `extractTSIDFromJWT()` helper
2. Create `cache.go`:
   - `TokenCache` struct
   - `NewTokenCache()` constructor
   - `Get()`, `Set()`, `cleanup()` methods

**Unit Tests:**

3. Test `AuthServerClient` token exchange with mock HTTP server
4. Test `TokenCache` expiry and cleanup
5. Test `extractTSIDFromJWT()` with valid/invalid JWTs

### Phase 5: RunConfig and Middleware Integration

**RunConfig (`pkg/runner/config.go`):**

1. Add `AuthServerConfig` field to `RunConfig`
2. Add `AuthServerClientConfig` struct

**Token Exchange Middleware (`pkg/auth/tokenexchange/`):**

3. Add `authServerClient` field to middleware
4. Update middleware factory to create authserver client if configured
5. Update `ServeHTTP` to use authserver client when available
6. Add fallback to direct exchange if authserver fails

**Unit Tests:**

7. Test middleware with mock authserver client
8. Test fallback behavior when authserver fails

### Phase 6: MCPExternalAuthConfig CRD Updates

**CRD Changes (`cmd/thv-operator/api/v1alpha1/mcpexternalauthconfig_types.go`):**

1. Add `ExternalAuthTypeAuthServer` constant to enum
2. Add `AuthServer *AuthServerConfig` field to `MCPExternalAuthConfigSpec`
3. Add `AuthServerConfig` struct (url, clientCert, caBundle)
4. Add `ClientCertificateConfig` type (issuerRef, trustDomain for SPIFFE URIs)
5. Run `make generate` and `make manifests`

**Controller Updates (`cmd/thv-operator/controllers/mcpexternalauthconfig_controller.go`):**

6. When `type: authServer`, create cert-manager `Certificate` for each referencing MCPServer:
   - Certificate name: `{mcpserver-name}-mtls`
   - CN: `{mcpserver-name}` (human-readable for audit logs)
   - URI SAN: `spiffe://{trustDomain}/ns/{namespace}/mcpserver/{name}`
   - Uses default duration (90 days) and renewBefore (15 days)
7. Track certificate readiness in status

### Phase 7: MCPServer Controller Integration

**Controller Updates (`cmd/thv-operator/controllers/mcpserver_controller.go`):**

1. When `externalAuthConfigRef` references an MCPExternalAuthConfig with `type: authServer`:
   - Resolve the MCPExternalAuthConfig
   - Get the client cert Secret name for this MCPServer
2. Mount client cert Secret and CA bundle ConfigMap to proxyrunner pod:
   - Add volume from Secret (client cert)
   - Add volume from ConfigMap (CA bundle)
   - Add volumeMounts to `/etc/toolhive/authserver-mtls` and `/etc/toolhive/authserver-ca`
3. Configure runconfig with authserver client paths

### Phase 8: Authserver Service Binary

**New Entry Point (`cmd/thv-authserver/`):**

1. Create `main.go` with Cobra CLI
2. Create `app/commands.go` with root command
3. Create `app/serve.go` with serve command:
   ```go
   // Loads configuration from:
   // - ConfigMap mounted at /etc/authserver/config.yaml
   // - Environment variables (AUTHSERVER_*)
   // - Command line flags

   // Creates:
   // - Upstream IDP provider (OAuth2/OIDC)
   // - Storage backend (memory or Redis)
   // - Fosite OAuth2 provider
   // - Handler with subject validator
   // - HTTP server with mTLS
   ```

**HTTP Server with mTLS (`pkg/authserver/server/`):**

4. Create `server.go`:
   - `Server` struct with `net/http.Server`
   - `NewServer(config, handler)` constructor
   - `Start()` with TLS and mTLS configuration:
     ```go
     tlsConfig := &tls.Config{
         MinVersion: tls.VersionTLS12,
         Certificates: []tls.Certificate{serverCert},
         ClientAuth: tls.RequireAndVerifyClientCert,
         ClientCAs: clientCAPool,
         VerifyPeerCertificate: subjectValidator.VerifyPeerCertificate,
     }
     ```
   - `Stop()` graceful shutdown
5. Add health check endpoint at `/healthz`
6. Add readiness endpoint at `/readyz`

**Container Image:**

7. Add Dockerfile at `cmd/thv-authserver/Dockerfile`
8. Add to `Taskfile.yaml` build targets
9. Push to container registry

### Phase 9: MCPAuthServer Controller

**Controller (`cmd/thv-operator/controllers/mcpauthserver_controller.go`):**

1. Create Deployment with authserver container
2. Configure server TLS from `certificateRef`
3. Mount client CA bundle for mTLS verification
4. Create ClusterIP Service
5. Create ConfigMap for authserver configuration
6. Handle status conditions and reconciliation

### Phase 10: Integration and E2E Testing

**Integration Tests:**

1. mTLS handshake between proxyrunner and authserver
2. Token exchange flow with upstream token refresh

**E2E Tests:**

3. Full flow: client → proxyrunner → authserver → upstream token
4. Kubernetes operator tests with cert-manager integration

**Production Features (future):**

5. Redis-backed storage for distributed sessions
6. Metrics for token exchange latency and cache hit rate
7. Audit logging for mTLS identity and token access
8. Certificate rotation handling
