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

#### New CRD: MCPAuthServer

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

  upstreamIdp:
    type: oidc
    oidc:
      issuer: "https://accounts.google.com"
      clientId: "..."
      clientSecretRef:
        name: authserver-secrets
        key: oidc-client-secret

  signingKey:
    secretRef:
      name: authserver-signing-key
      key: private.pem
    algorithm: RS256

  tls:
    # Server certificate for authserver's HTTPS endpoint
    certificateRef:
      name: mcp-authserver-tls  # cert-manager Certificate

    # mTLS: Client CA and validation rules for proxyrunner authentication
    clientAuth:
      # CA bundle for validating proxyrunner client certificates
      caBundle:
        configMapRef:
          name: toolhive-mtls-ca-bundle
          key: ca.crt

      # Allowed client certificate patterns (for access control)
      allowedSubjects:
        # Allow proxyrunners from specific namespaces
        organizationalUnits:
          - "toolhive-system"
          - "mcp-servers"
          - "mcp-production"
        # CN pattern: {mcpserver-name}.proxyrunner.{namespace}.toolhive.local
        commonNamePattern: "^[a-z0-9-]+\\.proxyrunner\\.[a-z0-9-]+\\.toolhive\\.local$"
```

**MCPAuthServer CRD mTLS Types:**

```go
// AuthServerTLSConfig configures TLS and mTLS for the authserver
type AuthServerTLSConfig struct {
    // CertificateRef references a cert-manager Certificate for server TLS
    // +kubebuilder:validation:Required
    CertificateRef CertificateReference `json:"certificateRef"`

    // ClientAuth configures mTLS client certificate validation
    // +optional
    ClientAuth *ClientAuthConfig `json:"clientAuth,omitempty"`
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
type AllowedSubjects struct {
    // OrganizationalUnits is a list of allowed OU values (typically namespaces)
    // Client cert must have at least one matching OU
    // +optional
    OrganizationalUnits []string `json:"organizationalUnits,omitempty"`

    // CommonNamePattern is a regex pattern for allowed CN values
    // +optional
    CommonNamePattern string `json:"commonNamePattern,omitempty"`
}
```

#### Resources Created by Controller
1. **Deployment** - Authserver pods with mTLS configuration
2. **Service** - ClusterIP for internal access (port 443 → 8443)
3. **ConfigMap** - Runtime configuration
4. **ServiceAccount** - Kubernetes RBAC identity

#### MCPServer CRD Updates for mTLS Client Certificate

Add a new field to `MCPServerSpec` for configuring the client certificate used when the proxyrunner communicates with the authserver:

**CRD Addition** ([cmd/thv-operator/api/v1alpha1/mcpserver_types.go](cmd/thv-operator/api/v1alpha1/mcpserver_types.go)):

```go
// MCPServerSpec defines the desired state of MCPServer
type MCPServerSpec struct {
    // ... existing fields ...

    // AuthServerClientConfig configures how the proxyrunner authenticates to the authserver
    // +optional
    AuthServerClientConfig *AuthServerClientConfig `json:"authServerClientConfig,omitempty"`
}

// AuthServerClientConfig configures mTLS client authentication to the authserver
type AuthServerClientConfig struct {
    // URL is the authserver base URL
    // +kubebuilder:validation:Required
    URL string `json:"url"`

    // ClientCertificateRef references a cert-manager Certificate for mTLS client auth
    // If specified, the controller creates the Certificate and mounts it to the pod
    // +optional
    ClientCertificateRef *ClientCertificateConfig `json:"clientCertificateRef,omitempty"`

    // CABundleRef references a ConfigMap containing the CA bundle for verifying authserver
    // Reuses existing CABundleSource type (defined at mcpserver_types.go:493-499)
    // +optional
    CABundleRef *CABundleSource `json:"caBundleRef,omitempty"`
}

// ClientCertificateConfig configures automatic client certificate provisioning
type ClientCertificateConfig struct {
    // IssuerRef references the cert-manager ClusterIssuer to use
    // +kubebuilder:validation:Required
    IssuerRef CertManagerIssuerReference `json:"issuerRef"`

    // Duration is the certificate validity period (default: 2160h / 90 days)
    // +kubebuilder:default="2160h"
    // +optional
    Duration string `json:"duration,omitempty"`

    // RenewBefore is when to renew before expiry (default: 360h / 15 days)
    // +kubebuilder:default="360h"
    // +optional
    RenewBefore string `json:"renewBefore,omitempty"`
}

// CertManagerIssuerReference references a cert-manager Issuer or ClusterIssuer
type CertManagerIssuerReference struct {
    // Name of the issuer
    Name string `json:"name"`

    // Kind is "Issuer" or "ClusterIssuer"
    // +kubebuilder:validation:Enum=Issuer;ClusterIssuer
    // +kubebuilder:default=ClusterIssuer
    Kind string `json:"kind,omitempty"`
}
```

**Example MCPServer with mTLS:**

```yaml
apiVersion: toolhive.stacklok.dev/v1alpha1
kind: MCPServer
metadata:
  name: github-tools
  namespace: mcp-servers
spec:
  image: ghcr.io/example/github-mcp:latest

  # OIDC config points to the authserver (for token validation)
  oidcConfig:
    type: inline
    resourceUrl: "https://github-tools.example.com/"
    inline:
      issuer: "https://mcp-authserver.toolhive-system.svc.cluster.local"
      audience: "github-tools"

  # NEW: mTLS client config for proxyrunner → authserver communication
  authServerClientConfig:
    url: "https://mcp-authserver.toolhive-system.svc.cluster.local"

    # Controller will create a cert-manager Certificate with:
    # - CN: github-tools.proxyrunner.mcp-servers.toolhive.local
    # - O: ToolHive ProxyRunner
    # - OU: mcp-servers
    clientCertificateRef:
      issuerRef:
        name: toolhive-mtls-ca
        kind: ClusterIssuer
      duration: "2160h"    # 90 days
      renewBefore: "360h"  # 15 days

    # CA bundle for verifying authserver's server certificate
    caBundleRef:
      configMapRef:
        name: toolhive-mtls-ca-bundle
        key: ca.crt
```

**Controller Behavior:**

When `authServerClientConfig.clientCertificateRef` is specified, the MCPServer controller:

1. **Creates a cert-manager Certificate:**
   ```yaml
   apiVersion: cert-manager.io/v1
   kind: Certificate
   metadata:
     name: github-tools-proxyrunner-client
     namespace: mcp-servers
   spec:
     secretName: github-tools-proxyrunner-client-tls
     commonName: github-tools.proxyrunner.mcp-servers.toolhive.local
     subject:
       organizations: ["ToolHive ProxyRunner"]
       organizationalUnits: ["mcp-servers"]
     usages: ["client auth"]
     issuerRef:
       name: toolhive-mtls-ca
       kind: ClusterIssuer
   ```

2. **Mounts the Secret to the proxyrunner pod:**
   ```yaml
   volumes:
     - name: authserver-client-cert
       secret:
         secretName: github-tools-proxyrunner-client-tls
   volumeMounts:
     - name: authserver-client-cert
       mountPath: /etc/toolhive/authserver-mtls
       readOnly: true
   ```

3. **Sets environment variables or runconfig:**
   ```json
   {
     "authserver_config": {
       "url": "https://mcp-authserver.toolhive-system.svc.cluster.local",
       "client_cert_path": "/etc/toolhive/authserver-mtls/tls.crt",
       "client_key_path": "/etc/toolhive/authserver-mtls/tls.key",
       "ca_bundle_path": "/etc/toolhive/authserver-mtls/ca.crt"
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

The authserver stores **upstream IDP tokens** (access tokens, refresh tokens) and links them to the JWTs it issues via the `tsid` (token session ID) claim. When a proxyrunner receives a client request with a JWT, it may need to:

1. **Retrieve the upstream access token** to pass to backend MCP servers that require the original IDP token
2. **Exchange tokens** (RFC 8693) to get a backend-specific token
3. **Refresh expired upstream tokens** on behalf of the client

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
│ CN: mcp-auth  │      │ CN: a.proxy   │      │ CN: b.proxy   │
│     server... │      │    runner...  │      │    runner...  │
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

#### How Proxyrunner Authenticates to Authserver

1. **Client Certificate Identity**:
   ```
   CN=myserver.proxyrunner.mcp-servers.toolhive.local
   O=ToolHive ProxyRunner
   OU=mcp-servers  (namespace)
   ```

2. **mTLS Handshake**:
   - Proxyrunner presents client certificate signed by shared CA
   - Authserver verifies certificate chain against trusted CA
   - Authserver extracts identity from certificate subject

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

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: mcp-authserver-tls
  namespace: toolhive-system
spec:
  secretName: mcp-authserver-tls
  duration: 8760h    # 1 year
  renewBefore: 720h  # 30 days
  issuerRef:
    name: toolhive-mtls-ca
    kind: ClusterIssuer
  commonName: mcp-authserver.toolhive-system.svc.cluster.local
  dnsNames:
    - mcp-authserver.toolhive-system.svc.cluster.local
    - mcp-authserver.toolhive-system.svc
    - mcp-authserver
  usages:
    - server auth
```

#### ProxyRunner Client Certificate (per MCPServer)

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ${MCPSERVER_NAME}-proxyrunner-client
  namespace: ${NAMESPACE}
spec:
  secretName: ${MCPSERVER_NAME}-proxyrunner-client-tls
  duration: 2160h    # 90 days
  renewBefore: 360h  # 15 days
  issuerRef:
    name: toolhive-mtls-ca
    kind: ClusterIssuer
  commonName: ${MCPSERVER_NAME}.proxyrunner.${NAMESPACE}.toolhive.local
  subject:
    organizations:
      - "ToolHive ProxyRunner"
    organizationalUnits:
      - ${NAMESPACE}
  usages:
    - client auth
```

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

#### Why Token Exchange is Needed

When a client authenticates, the authserver:
1. Redirects to upstream IDP (Google, GitHub, etc.)
2. Receives upstream tokens (access token, refresh token, ID token)
3. **Stores these tokens** linked to a session ID (`tsid`)
4. Issues its own JWT containing the `tsid` claim

The client's JWT **does not contain the upstream tokens** - only a reference to them. When the proxyrunner needs to call a backend that requires the upstream token (e.g., a GitHub MCP server needs a GitHub access token), it must:
1. Extract `tsid` from the client's JWT
2. Call authserver to retrieve the upstream access token
3. Use that token to authenticate to the backend

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

#### Proxyrunner Token Exchange Endpoint (new)

Add to authserver handlers:

```go
// pkg/authserver/server/handlers/proxyrunner_exchange.go

// ProxyRunnerIdentity represents the identity extracted from a proxyrunner's mTLS client certificate
type ProxyRunnerIdentity struct {
    // Name is the MCPServer name (extracted from CN before ".proxyrunner")
    Name string
    // Namespace is the Kubernetes namespace (from OU)
    Namespace string
    // FullCN is the complete Common Name
    FullCN string
    // CertificateSerial is the certificate serial number (for audit logging)
    CertificateSerial string
}

// extractProxyRunnerIdentity extracts the proxyrunner identity from the mTLS client certificate.
// Returns an error if no client certificate is present or if the certificate doesn't match
// the expected proxyrunner certificate format.
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

    // Extract namespace from OU (Organizational Unit)
    // Certificate format: OU=mcp-servers (the namespace)
    var namespace string
    if len(cert.Subject.OrganizationalUnit) > 0 {
        namespace = cert.Subject.OrganizationalUnit[0]
    } else {
        return nil, errors.New("client certificate missing OU (namespace)")
    }

    // Extract MCPServer name from CN
    // Certificate format: CN=github-tools.proxyrunner.mcp-servers.toolhive.local
    cn := cert.Subject.CommonName
    if cn == "" {
        return nil, errors.New("client certificate missing CN")
    }

    // Parse CN to extract MCPServer name
    // Expected format: {mcpserver-name}.proxyrunner.{namespace}.toolhive.local
    parts := strings.Split(cn, ".")
    if len(parts) < 4 || parts[1] != "proxyrunner" {
        return nil, fmt.Errorf("invalid CN format: expected {name}.proxyrunner.{ns}.toolhive.local, got %s", cn)
    }
    mcpServerName := parts[0]

    // Verify namespace in CN matches OU
    cnNamespace := parts[2]
    if cnNamespace != namespace {
        return nil, fmt.Errorf("CN namespace (%s) doesn't match OU namespace (%s)", cnNamespace, namespace)
    }

    return &ProxyRunnerIdentity{
        Name:              mcpServerName,
        Namespace:         namespace,
        FullCN:            cn,
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

// SubjectValidator validates client certificate subjects against allowed patterns
type SubjectValidator struct {
    // allowedOUs is a set of allowed Organizational Unit values (typically namespaces)
    allowedOUs map[string]bool
    // cnPattern is a compiled regex for validating Common Name format
    cnPattern *regexp.Regexp
}

// NewSubjectValidator creates a validator from MCPAuthServer CRD configuration
func NewSubjectValidator(allowedSubjects *AllowedSubjects) (*SubjectValidator, error) {
    if allowedSubjects == nil {
        // No restrictions - all valid certificates are allowed
        return &SubjectValidator{}, nil
    }

    validator := &SubjectValidator{}

    // Build allowed OU set
    if len(allowedSubjects.OrganizationalUnits) > 0 {
        validator.allowedOUs = make(map[string]bool, len(allowedSubjects.OrganizationalUnits))
        for _, ou := range allowedSubjects.OrganizationalUnits {
            validator.allowedOUs[ou] = true
        }
    }

    // Compile CN pattern
    if allowedSubjects.CommonNamePattern != "" {
        pattern, err := regexp.Compile(allowedSubjects.CommonNamePattern)
        if err != nil {
            return nil, fmt.Errorf("invalid commonNamePattern: %w", err)
        }
        validator.cnPattern = pattern
    }

    return validator, nil
}

// validateSubjectAllowed checks if a proxyrunner identity is allowed based on
// the allowedSubjects configuration from the MCPAuthServer CRD.
//
// Returns nil if allowed, error if rejected.
func (v *SubjectValidator) validateSubjectAllowed(identity *ProxyRunnerIdentity) error {
    // If no restrictions configured, allow all
    if v.allowedOUs == nil && v.cnPattern == nil {
        return nil
    }

    // Check OU (namespace) restriction
    if v.allowedOUs != nil {
        if !v.allowedOUs[identity.Namespace] {
            return fmt.Errorf("namespace %q is not in allowed list", identity.Namespace)
        }
    }

    // Check CN pattern restriction
    if v.cnPattern != nil {
        if !v.cnPattern.MatchString(identity.FullCN) {
            return fmt.Errorf("CN %q does not match allowed pattern", identity.FullCN)
        }
    }

    return nil
}

// validateSessionAudience verifies that the proxyrunner is authorized to access this session.
// The session's audience (from the JWT) should match the MCPServer identified by the client cert.
//
// This prevents a compromised proxyrunner in namespace A from requesting tokens for sessions
// that were intended for a different MCPServer in namespace B.
//
// Audience matching rules:
// 1. If JWT has "aud" claim, it must match the proxyrunner's MCPServer name
// 2. The namespace/name combination provides additional binding
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
    // - Service URL format: "https://github-tools.mcp-servers.svc.cluster.local"
    expectedAudiences := map[string]bool{
        identity.Name: true,
        fmt.Sprintf("%s.%s", identity.Name, identity.Namespace): true,
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

    return fmt.Errorf("token audience %v does not match proxyrunner %s/%s",
        audiences, identity.Namespace, identity.Name)
}

func (h *Handler) ProxyRunnerTokenExchangeHandler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    // 1. Extract proxyrunner identity from mTLS client cert
    identity, err := extractProxyRunnerIdentity(r)
    if err != nil {
        logger.Warnf("mTLS identity extraction failed: %v", err)
        h.writeError(w, fosite.ErrAccessDenied.WithHint("mTLS client certificate required"))
        return
    }

    logger.Infof("Token exchange request from proxyrunner %s/%s (cert serial: %s)",
        identity.Namespace, identity.Name, identity.CertificateSerial)

    // 2. Validate proxyrunner is allowed based on allowedSubjects from MCPAuthServer CRD
    //    h.subjectValidator is initialized from config at startup
    if err := h.subjectValidator.validateSubjectAllowed(identity); err != nil {
        logger.Warnf("Proxyrunner %s/%s rejected by subject policy: %v",
            identity.Namespace, identity.Name, err)
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
    if err != nil {
        logger.Warnf("Failed to retrieve upstream tokens for tsid %s: %v", tsid, err)
        h.writeError(w, fosite.ErrInvalidRequest.WithHint("session not found or expired"))
        return
    }

    // 6. Verify proxyrunner is authorized for this specific session
    //    The session's audience should match the MCPServer making the request
    if err := h.validateSessionAudience(claims, identity); err != nil {
        logger.Warnf("Session audience mismatch for proxyrunner %s/%s: %v",
            identity.Namespace, identity.Name, err)
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
```

#### ProxyRunner Client-Side Token Exchange (MCPServer)

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

    // Add CA bundle for server certificate validation
    if cfg.CABundlePath != "" {
        clientBuilder = clientBuilder.WithCABundle(cfg.CABundlePath)
    }

    // Add client certificate for mTLS
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
        logger.Debugf("Token cache hit for tsid %s", tsid)
        return cached, nil
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

    resp, err := c.httpClient.Do(req)
    if err != nil {
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

**Integration with tokenexchange middleware:**

Update the existing token exchange middleware to use the authserver client when configured:

```go
// pkg/auth/tokenexchange/middleware.go (updated)

// In the middleware function, check for authserver config
func (m *Middleware) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // ... existing code to extract identity and subject token ...

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
| `cmd/thv-operator/api/v1alpha1/mcpauthserver_types.go` | MCPAuthServer CRD with mTLS config |
| `cmd/thv-operator/controllers/mcpauthserver_controller.go` | MCPAuthServer reconciler |
| `pkg/authserver/server/handlers/proxyrunner_exchange.go` | Token exchange endpoint for proxyrunners |
| `pkg/authserver/server/handlers/subject_validator.go` | mTLS subject validation (allowedSubjects) |
| `pkg/auth/authserver/client.go` | Client for proxyrunner → authserver mTLS calls |
| `pkg/auth/authserver/cache.go` | Token cache for exchanged tokens |

### Modified Files

| File | Changes |
|------|---------|
| `cmd/thv-operator/api/v1alpha1/mcpserver_types.go` | Add `AuthServerClientConfig` for mTLS client certs |
| `cmd/thv-operator/controllers/mcpserver_controller.go` | Create cert-manager Certificate for proxyrunner |
| `pkg/networking/http_client.go` | Add `WithClientCertificate()` to HttpClientBuilder |
| `pkg/authserver/server/handlers/handler.go` | Add `subjectValidator` field, update `NewHandler()` |
| `pkg/authserver/storage/types.go` | Add `GetUpstreamTokens(ctx, tsid)` method |
| `pkg/runner/config.go` | Add `AuthServerConfig` struct |
| `pkg/auth/tokenexchange/middleware.go` | Integrate authserver client for token exchange |

---

## Implementation Phases

### Phase 1: CRD and Operator Foundation

**MCPAuthServer CRD:**
1. Create `mcpauthserver_types.go` with CRD types:
   - `MCPAuthServerSpec` (issuer, replicas, upstreamIdp, signingKey, tls)
   - `AuthServerTLSConfig` (certificateRef, clientAuth)
   - `ClientAuthConfig` (caBundle, allowedSubjects)
   - `AllowedSubjects` (organizationalUnits, commonNamePattern)
2. Run `make generate` and `make manifests`
3. Create `mcpauthserver_controller.go` with basic reconciliation

**MCPServer CRD Updates:**
4. Add `AuthServerClientConfig` to `MCPServerSpec`
5. Add `ClientCertificateConfig` and `CertManagerIssuerReference` types
6. Run `make generate` and `make manifests`

### Phase 2: Authserver Server-Side mTLS

**Handler Updates (`pkg/authserver/server/handlers/`):**
1. Create `subject_validator.go`:
   - `SubjectValidator` struct
   - `NewSubjectValidator()` function
   - `validateSubjectAllowed()` method
2. Update `handler.go`:
   - Add `subjectValidator` field to Handler
   - Update `NewHandler()` to accept `allowedSubjects`
3. Create `proxyrunner_exchange.go`:
   - `ProxyRunnerIdentity` struct
   - `extractProxyRunnerIdentity()` function
   - `validateSessionAudience()` function
   - `ProxyRunnerTokenExchangeHandler()` handler
4. Update `handler.go` Routes() to register `/internal/token-exchange`

**Storage Updates (`pkg/authserver/storage/`):**
5. Add `GetUpstreamTokens(ctx, tsid)` to `UpstreamTokenStorage` interface
6. Implement in `memory.go` (and later Redis)

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
3. Add unit tests

### Phase 5: RunConfig and Middleware Integration

**RunConfig (`pkg/runner/config.go`):**
1. Add `AuthServerConfig` field to `RunConfig`
2. Add `AuthServerClientConfig` struct

**Token Exchange Middleware (`pkg/auth/tokenexchange/`):**
3. Add `authServerClient` field to middleware
4. Update middleware factory to create authserver client if configured
5. Update `ServeHTTP` to use authserver client when available
6. Add fallback to direct exchange if authserver fails

### Phase 6: MCPServer Controller Integration

**Controller Updates (`cmd/thv-operator/controllers/mcpserver_controller.go`):**
1. Check for `authServerClientConfig` in spec
2. If `clientCertificateRef` specified:
   - Create cert-manager `Certificate` resource
   - Wait for certificate to be ready
3. Mount client cert Secret to proxyrunner pod:
   - Add volume from Secret
   - Add volumeMount to `/etc/toolhive/authserver-mtls`
4. Configure runconfig with authserver client paths

### Phase 7: MCPAuthServer Controller

**Controller (`cmd/thv-operator/controllers/mcpauthserver_controller.go`):**
1. Create Deployment with authserver container
2. Configure server TLS from `certificateRef`
3. Mount client CA bundle for mTLS verification
4. Create ClusterIP Service
5. Create ConfigMap for authserver configuration
6. Handle status conditions and reconciliation

### Phase 8: Testing and Production Readiness

**Testing:**

1. Unit tests for all new packages
2. Integration tests for mTLS handshake
3. E2E test: client → proxyrunner → authserver → upstream token

**Production Features:**

4. Redis-backed storage for distributed sessions
5. Metrics for token exchange latency and cache hit rate
6. Audit logging for mTLS identity and token access
7. Certificate rotation handling
