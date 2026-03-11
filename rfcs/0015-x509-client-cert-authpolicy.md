# X.509 Client Certificate Authentication in Kuadrant AuthPolicy

- Feature Name: `x509-client-cert-authpolicy`
- Status: **Draft**
- Start Date: 2026-03-03
- Issue tracking: [Kuadrant/architecture#140](https://github.com/Kuadrant/architecture/issues/140)

# Summary

Enable X.509 client certificate authentication in Kuadrant's AuthPolicy by extending Authorino's AuthConfig API to extract client certificates from the X-Forwarded-Client-Cert (XFCC) header. This allows users to leverage Authorino's native x509 authentication features (multi-CA trust, certificate selection via label selectors) in Kuadrant deployments, replacing existing workarounds.

# Goals

- Support X.509 client certificate authentication in Kuadrant AuthPolicy, making the inherited API from Authorino's AuthConfig functionally operational in Kuadrant deployments
- Enable Authorino to extract and validate client certificates from the XFCC header populated by the proxy
- Allow users to leverage Authorino's native x509 authentication features, particularly:
  - Multi-CA trust via label selectors (vs. single root CA in proxy configuration)
  - Fine-grained certificate selection and validation
  - Integration with Kubernetes Secret resources for trusted CAs
- Allow users to configure which HTTP header contains the client certificate (defaulting to X-Forwarded-Client-Cert)
- Maintain backward compatibility with existing AuthConfig/AuthPolicy configurations
- Eliminate the need for OPA-based workarounds that bypass Authorino's authentication phase

# Non-goals

- Support for certificate extraction directly from the TLS connection attributes in the wasm-shim (blocked by current technical limitations)
- Automatic injection of CA certificates into Gateway resources (this may be addressed in future iterations)
- Prescribing a specific method for configuring proxy-level TLS client certificate validation (users choose based on their Gateway API provider and version)
- Custom certificate validation logic beyond what Authorino already provides
- Certificate revocation checking (CRL/OCSP) - this remains the responsibility of the gateway proxy or Authorino's existing capabilities

# Problem Statement

Kuadrant's AuthPolicy API inherits the `authentication.x509` configuration from Authorino's AuthConfig CRD, creating the misleading appearance that X.509 client certificate authentication is supported in Kuadrant. However, this feature is non-functional in practice due to a critical technical limitation:

**Certificate Propagation Gap**: The wasm-shim cannot read the client certificate in PEM format from the TLS connection attributes, preventing it from populating the `attributes.source.certificate` field in the external authorization CheckRequest to Authorino.

As a result, users attempting to use `spec.rules.authentication.x509` in their AuthPolicy resources find that authentication fails because Authorino never receives the client certificate data, even when the proxy is properly configured to perform TLS client certificate validation.

## Current Workaround

The temporary workaround documented in issue #140 involves:
- Configuring TLS client certificate validation in the gateway proxy (using Gateway API provider-specific resources like Istio EnvoyFilter or Gateway API's `spec.tls.frontend.default.validation` in v1.5+)
- **Skipping the authentication phase entirely** by using `anonymous` authentication in AuthPolicy
- **Using OPA Rego authorization policies** to manually parse and validate certificates from the XFCC header
- Managing CA certificate ConfigMaps

## Main Pain Point

The critical problem with this workaround is not the proxy configuration method—it's that **users cannot leverage Authorino's native x509 authentication features**:

- **Loss of multi-CA trust**: Authorino's x509 authentication supports trusting multiple intermediate CAs via label selectors, enabling fine-grained certificate revocation and trust management. The workaround points users to configure a single root CA in the proxy, losing this flexibility.
- **Complexity**: Users must implement certificate parsing and validation logic in OPA Rego instead of using Authorino's built-in, well-tested x509 authenticator.
- **Inconsistency**: The `authentication.x509` API exists but doesn't work, creating a confusing user experience.
- **Fragility**: OPA policies for certificate validation are custom code that must be maintained separately from the standard AuthPolicy configuration.

# Proposed Solution

The solution is to extend Authorino's AuthConfig API to support extracting client certificates from HTTP headers (specifically the XFCC header), and propagate this capability to Kuadrant's AuthPolicy. This allows users to leverage Authorino's native x509 authentication features when certificates are forwarded via XFCC.

## Core API Extension

### Extend Authorino's AuthConfig API

Add a new field to the `authentication.x509` configuration to specify the source of the client certificate:

```yaml
apiVersion: authorino.kuadrant.io/v1beta3
kind: AuthConfig
spec:
  authentication:
    "x509-authn":
      x509:
        # New field: specify the source of the certificate
        source:
          # Option 1: Extract from HTTP header (default: X-Forwarded-Client-Cert)
          header: "X-Forwarded-Client-Cert"
          # Option 2: Extract from CheckRequest attributes (current behavior)
          # attribute: "source.certificate"

        # Existing x509 validation fields
        selector:
          matchLabels:
            group: admins
        allNamespaces: true
```

**Behavior**:
- When `source.header` is specified, Authorino extracts the certificate from the named HTTP request header
- When `source.attribute` is specified (or neither is specified for backward compatibility), Authorino uses the current behavior of reading from `attributes.source.certificate`
- The XFCC header format follows the [Envoy specification](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/headers#x-forwarded-client-cert)
- Certificate validation (leaf maps to a trusted CA) remains unchanged once the certificate is extracted

### Propagate API Changes Through the Stack

**Authorino Operator:**
- Update the AuthConfig CRD manifests hosted in the authorino-operator repository to include the new `source` field
- No controller logic changes required (Authorino itself handles the certificate extraction)

**Kuadrant Operator:**
- Update the AuthPolicy CRD to expose the new `authentication.x509.source` field – i.e., upgrade the version of the embedded AuthConfig types to one that includes the new field
- Update the AuthPolicy-to-AuthConfig translation logic to propagate the `source` configuration
- No changes to the wasm-shim required (certificate extraction happens entirely at the Authorino layer)

## Proxy Configuration for XFCC Header Population

**This is a prerequisite, not part of this RFC's deliverables.** Users must configure their gateway proxy to perform TLS client certificate validation and populate the XFCC header. There are three approaches, listed from recommended to exceptional use cases:

### Tier 1: Gateway API `spec.tls.frontend.default.validation` (Recommended)

Use Gateway API v1.5+'s standardized `spec.tls.frontend.default.validation` configuration:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: istio
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    hostname: "*.example.com"
    tls:
      mode: Terminate
      certificateRefs:
      - name: gateway-cert
        kind: Secret
  # Configure client certificate validation
  tls:
    frontend:
      default:
        validation:
          caCertificateRefs:
          - name: client-ca-bundle
            kind: ConfigMap
            group: ""
          mode: AllowValidOnly
```

**Advantages**:
- Standardized, vendor-agnostic approach
- Declarative Gateway API resource (no low-level Envoy configuration)
- Supported in Gateway API v1.5+ (Standard)
- Gateway implementation automatically populates the XFCC header with validated client certificates

**Configuration**:
- `spec.tls.frontend.default.validation.caCertificateRefs`: Points to ConfigMap(s) containing trusted CA certificates (PEM format under `ca.crt` key)
- `spec.tls.frontend.default.validation.mode`: Validation mode (`AllowValidOnly` requires valid certs; `AllowInsecureFallback` permits connections without certs)

**When to use**: This is the **recommended approach** for users with Gateway API v1.5+ support.

### Tier 2: EnvoyFilter or Provider-Specific Resources (Alternative)

For users whose Gateway API implementation does not yet support `spec.tls.frontend.default.validation` (pre-v1.5), use provider-specific resources to configure TLS client certificate validation.

**Example with Istio EnvoyFilter**:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: gateway-client-cert-validation
  namespace: istio-system
spec:
  workloadSelector:
    labels:
      istio: ingressgateway
  configPatches:
  - applyTo: LISTENER
    match:
      context: GATEWAY
      listener:
        portNumber: 443
    patch:
      operation: MERGE
      value:
        filter_chains:
        - transport_socket:
            name: envoy.transport_sockets.tls
            typed_config:
              '@type': type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext
              require_client_certificate: true
              common_tls_context:
                validation_context:
                  trusted_ca:
                    filename: /etc/certs/ca-bundle.pem
```

**Advantages**:
- Works with older Gateway API versions
- Fine-grained control over Envoy configuration

**Disadvantages**:
- Provider-specific (not portable across different Gateway implementations)
- Requires understanding of underlying proxy configuration (e.g., Envoy)
- More brittle (changes to Gateway implementation may break configuration)

**When to use**: Use this approach when Gateway API v1.5+ is not available but you need TLS client certificate validation.

### Tier 3: User-Specified XFCC Forwarding (Exceptional Cases)

In exceptional scenarios where TLS client certificate validation **cannot** be configured at the gateway level (e.g., gateway is managed by another team, or validation must happen exclusively at L7), users can configure the proxy to **forward user-specified XFCC headers without validation**.

**Example with Istio Gateway annotation**:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
  annotations:
    proxy.istio.io/config: '{"gatewayTopology": {"forwardClientCertDetails": "ALWAYS_FORWARD_ONLY"}}'
spec:
  # ... listeners without frontendValidation
```

**Behavior**:
- The proxy **forwards** the XFCC header from the incoming request without validating the certificate
- Clients must provide their certificate in the XFCC header (typically as a PEM-encoded string)
- **No TLS-level certificate validation occurs** at the gateway

**Security Implications**:
- **L4 → L7 validation shift**: TLS authentication moves entirely from Layer 4 (transport) to Layer 7 (application). The proxy no longer validates certificates against trusted CAs during the TLS handshake.
- **Authorino becomes the sole validator**: Only Authorino validates certificates against trusted CAs. There is no defense-in-depth.
- **Client key possession not verified**: Since the certificate is passed as a header (not via TLS handshake), there's no cryptographic proof that the client possesses the private key corresponding to the certificate.
- **Spoofing risk**: If the XFCC header reaches the proxy from an untrusted source (e.g., through another proxy in the chain), it could be spoofed.

**When to use**: Use this approach **only** in exceptional cases where:
- Gateway-level TLS validation is not possible or not desired
- You accept the security trade-offs of L7-only certificate validation
- You trust the source of the XFCC header (e.g., it comes from a trusted upstream proxy)
- You understand that certificate private key possession is not cryptographically verified

**Not recommended** for production security-critical applications.

## Gateway Configuration Is User Responsibility

Regardless of which tier is used, **configuring the gateway for TLS client certificate validation is the user's responsibility** and is not automated by this RFC. Gateway TLS configuration is infrastructure-level configuration that should be explicit and visible.

**Future Enhancement** (out of scope for this RFC): A future iteration could introduce opt-in automatic injection of CA certificate references into Gateway configurations based on AuthPolicy settings.

# Architecture Changes

## Component Interaction Flow

```
┌─────────────┐
│   Client    │
│ (with cert) │
└──────┬──────┘
       │ HTTPS + mTLS
       ↓
┌─────────────────────────────────────────────┐
│         Gateway (Envoy/Istio)               │
│  ┌───────────────────────────────────────┐  │
│  │ TLS Termination                       │  │
│  │ - Validates client cert (Tier 1/2)*   │  │
│  │   OR forwards XFCC header (Tier 3)    │  │
│  │ - Populates XFCC header               │  │
│  └───────────────────────────────────────┘  │
└──────┬──────────────────────────────────────┘
       │ HTTP + XFCC header
       ↓
┌─────────────────────────────────────────────┐
│         Wasm-Shim (in Gateway)              │
│  ┌───────────────────────────────────────┐  │
│  │ - Forwards request + XFCC to          │  │
│  │   Authorino for ext_authz             │  │
│  └───────────────────────────────────────┘  │
└──────┬──────────────────────────────────────┘
       │ CheckRequest + XFCC in headers
       ↓
┌─────────────────────────────────────────────┐
│            Authorino                        │
│  ┌───────────────────────────────────────┐  │
│  │ X.509 Authentication                  │  │
│  │ - Extracts cert from XFCC header      │  │
│  │ - Parses PEM certificate              │  │
│  │ - Validates against trusted CAs       │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

**Note**: The gateway's role in certificate validation depends on the proxy configuration tier:
- **Tier 1/2**: Gateway validates client certificates during TLS handshake and populates XFCC header with validated certificate
- **Tier 3**: Gateway forwards user-provided XFCC header without TLS-level validation; Authorino is the sole validator

## Key Architectural Principles

1. **Separation of Concerns**: TLS termination at the gateway level; certificate-based authentication at the application level (Authorino). Note: Tier 3 shifts TLS validation entirely to L7 (Authorino only).
2. **Standard Protocols**: Uses industry-standard XFCC header format for certificate propagation between proxy and authorization service
3. **Flexibility**: Supports multiple proxy configuration approaches (Gateway API, EnvoyFilter, XFCC forwarding) based on user environment and requirements
4. **Explicit Configuration**: Gateway/proxy TLS configuration remains explicit and user-managed, not automated by Kuadrant
5. **Defense in Depth** (Tier 1/2): Gateway validates certificates at TLS layer; Authorino provides fine-grained application-level validation with multi-CA trust

# API Changes

## Authorino AuthConfig CRD

**New field**: `spec.authentication.<name>.x509.source`

```go
// X509AuthenticationSpec defines x509 client certificate authentication
type X509AuthenticationSpec struct {
    // Source defines where to extract the client certificate from
    // +optional
    Source *X509CertificateSource `json:"source,omitempty"`

    // Selector defines rules for selecting trusted certificates
    // +optional
    Selector *metav1.LabelSelector `json:"selector,omitempty"`

    // AllNamespaces allows searching for trusted certificates across all namespaces
    // +optional
    AllNamespaces bool `json:"allNamespaces,omitempty"`
}

// X509CertificateSource defines the source from which to extract the client certificate
// Only one of Header or Attribute should be specified
type X509CertificateSource struct {
    // Header specifies the HTTP header name containing the client certificate in PEM format
    // Typically "X-Forwarded-Client-Cert" for certificates forwarded by the proxy
    // +optional
    Header string `json:"header,omitempty"`

    // Attribute specifies the CheckRequest attribute path containing the certificate
    // Default behavior if neither Header nor Attribute is specified
    // +optional
    Attribute string `json:"attribute,omitempty"`
}
```

**Default behavior**:
- If `source` is not specified: backward compatible behavior (read from `attributes.source.certificate`)
- If `source.header` is specified: extract certificate from the named HTTP header
- If both `header` and `attribute` are specified: `header` takes precedence

## Kuadrant AuthPolicy CRD

The AuthPolicy CRD mirrors the AuthConfig changes:

```yaml
apiVersion: kuadrant.io/v1
kind: AuthPolicy
metadata:
  name: my-auth-policy
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: my-route
  defaults:
    rules:
      authentication:
        "x509-from-gateway":
          x509:
            source:
              header: "X-Forwarded-Client-Cert"
            selector:
              matchLabels:
                app.kubernetes.io/name: trusted-client
```

# Changes to Kuadrant Components

## Authorino

**Repository**: https://github.com/Kuadrant/authorino

**Changes**:
1. **API Types** (`api/v1beta3/auth_config_types.go`):
   - Add `X509CertificateSource` struct
   - Add `Source` field to `X509AuthenticationSpec`

2. **X.509 Authenticator** (`pkg/evaluators/identity/x509.go`):
   - Add logic to check if `source.header` is configured
   - If configured, extract certificate from the HTTP request header instead of CheckRequest attributes
   - Parse XFCC header format (URL-encoded PEM, potentially with additional metadata)
   - Maintain existing certificate validation logic

3. **Tests**:
   - Unit tests for XFCC header parsing
   - Integration tests with mock XFCC headers
   - Backward compatibility tests (ensure existing configs still work)

**Estimated Complexity**: Low-Medium
- XFCC header parsing is straightforward (URL-encoded PEM)
- Existing certificate validation logic is reused
- Main risk: XFCC format variations across different proxies

## Authorino Operator

**Repository**: https://github.com/Kuadrant/authorino-operator

**Changes**:
1. **CRD Manifests** (`config/crd/bases/authorino.kuadrant.io_authconfigs.yaml`):
   - Update with new `source` field schema
   - Add OpenAPI validation rules

2. **API Version**:
   - Consider whether this requires a new API version (v1beta4) or can be added to v1beta3
   - Recommendation: Add to v1beta3 as an optional field (backward compatible)

**Estimated Complexity**: Low
- Primarily CRD manifest updates
- No controller logic changes

## Kuadrant Operator

**Repository**: https://github.com/Kuadrant/kuadrant-operator

**Changes**:
1. **AuthPolicy CRD** (`api/v1/authpolicy_types.go`):
   - Mirror the new `X509CertificateSource` types from Authorino
   - Update CRD manifests

2. **AuthPolicy-to-AuthConfig Translation** (`internal/controller/authconfigs_reconciler.go` or similar):
   - Propagate `authentication.<name>.x509.source` from AuthPolicy to generated AuthConfig
   - No additional business logic required

3. **Documentation**:
   - Update AuthPolicy examples to show X.509 authentication with XFCC header
   - Document the requirement for Gateway `spec.tls.frontend.default.validation` configuration
   - Provide complete working example combining Gateway config + AuthPolicy

**Estimated Complexity**: Low-Medium
- Straightforward API propagation
- Documentation is critical for user adoption

## Wasm-Shim

**Repository**: https://github.com/Kuadrant/wasm-shim

**Changes**: **NONE**

The wasm-shim already forwards HTTP headers (including XFCC) to Authorino as part of the CheckRequest. No changes are required.

**Verification needed**: Confirm that the XFCC header is not stripped or modified by the wasm-shim.

# Security Considerations

## Certificate Validation Trust Chain

The security model depends on which proxy configuration tier is used:

### Tier 1 & 2: Gateway-Level TLS Validation (Defense in Depth)

When using Gateway API `spec.tls.frontend.default.validation` or EnvoyFilter-based TLS validation:

1. **Gateway-Level Validation (L4/TLS)**:
   - The gateway validates the client certificate during the TLS handshake against the CA bundle configured in the proxy
   - Only certificates signed by trusted CAs are accepted
   - Invalid, expired, or untrusted certificates are rejected at the TLS layer
   - The proxy cryptographically verifies that the client possesses the private key corresponding to the certificate

2. **Authorino-Level Validation (L7/Application)**:
   - Authorino extracts the certificate from the XFCC header
   - Performs additional application-level validation:
     - Trusts certificates that match label selectors (supports multiple intermediate CAs)
     - Validates certificate chain against trusted CAs from Kubernetes Secrets
     - Can apply fine-grained rules (subject matching, organization, etc.)

**Defense in Depth**: Both layers must succeed for authentication to pass. The gateway validates the certificate cryptographically at TLS level; Authorino applies application-specific trust policies.

### Tier 3: L7-Only Validation (Authorino Only)

When using user-specified XFCC (forwarded by the proxy without TLS validation):

1. **No Gateway-Level Validation**: The proxy forwards the XFCC header without validating the certificate or performing a TLS handshake with client cert requirements.

2. **Authorino-Level Validation Only (L7/Application)**:
   - Authorino is the **sole validator** of the certificate
   - Validates the certificate against trusted CAs from Kubernetes Secrets
   - **Cannot verify** that the client possesses the private key (no TLS handshake)

**Security Trade-offs**:
- ❌ No cryptographic proof of private key possession
- ❌ No defense in depth (single point of failure)
- ⚠️ Potential spoofing if XFCC header comes from untrusted source
- ✅ Still validates certificate chain and trust against CAs
- ✅ Supports multi-CA trust via Authorino's label selectors

**Use only in exceptional cases** where gateway-level TLS validation is not feasible.

## XFCC Header Security

**Critical Security Consideration**: The XFCC header must originate from a trusted source and cannot be spoofed by clients.

### Tier 1 & 2: XFCC Set by Gateway After TLS Validation

When using gateway-level TLS validation, the proxy configuration **must** ensure:

1. **Strip incoming XFCC headers**: Any XFCC headers in incoming client requests must be stripped to prevent spoofing
2. **Set XFCC only after validation**: The proxy sets the XFCC header only after successfully validating the client certificate during the TLS handshake

**Envoy behavior**:
- When `forward_client_cert_details` is set to `SANITIZE_SET` or `APPEND_FORWARD` (recommended), Envoy:
  - Strips any incoming XFCC headers from untrusted clients
  - Sets/appends the XFCC header only after TLS validation succeeds
- This is the **standard and secure** behavior

**Istio default**: Istio's gateway proxies are typically configured with secure XFCC handling by default.

### Tier 3: XFCC Forwarded from Client

When using `forwardClientCertDetails: ALWAYS_FORWARD_ONLY`:

⚠️ **Security Warning**: The proxy forwards the XFCC header from the incoming request **without validation**. This means:
- Clients can potentially spoof the XFCC header
- The header must come from a **trusted source** (e.g., an upstream proxy that performed TLS validation)
- If clients can directly reach the gateway, **do not use this approach**

**When Tier 3 is acceptable**:
- XFCC header originates from a trusted upstream proxy
- Network topology prevents direct client access to the gateway
- You accept the risk of L7-only validation

### Documentation Warnings

Documentation must explicitly warn users:
- **Default assumption**: XFCC header is set by the gateway proxy after TLS validation (Tier 1/2)
- **Verify proxy configuration**: Ensure the proxy strips incoming XFCC headers and sets them only after validation
- **Do not use `source.header` in Tier 3** unless you trust the source and understand the security implications
- **When in doubt**: Use Tier 1 (Gateway API) or Tier 2 (EnvoyFilter) with proper TLS validation

## CA Certificate Management

### Gateway-Level CA Configuration (Tier 1 & 2)

**User Responsibility**:
- Users must securely manage CA certificates for gateway-level validation:
  - **Tier 1**: ConfigMaps referenced in Gateway `spec.tls.frontend.default.validation.caCertificateRefs`
  - **Tier 2**: CA certificates configured in EnvoyFilter or provider-specific resources
- Rotation of gateway CA certificates requires updating ConfigMaps/resources and potentially restarting gateway pods
- Namespace isolation: CA ConfigMaps should be in the same namespace as the Gateway or explicitly allowed via ReferenceGrant

### Authorino-Level CA Configuration (All Tiers)

**User Responsibility**:
- Users must manage CA certificates in Kubernetes Secrets with appropriate labels for Authorino's label selectors
- Rotation of Authorino-trusted CAs requires updating Secrets
- **Multi-CA trust**: Authorino can trust multiple intermediate CAs, providing fine-grained revocation control

**Key Difference**:
- **Gateway CAs** (Tier 1/2): Typically a single root CA, validated at TLS handshake
- **Authorino CAs**: Can be multiple intermediate CAs selected via label selectors, providing granular trust management and easier revocation

### Best Practices

**To be documented**:
- Use cert-manager for automatic CA certificate management (both gateway and Authorino)
- Implement certificate rotation strategies
- Monitor certificate expiration
- Consider using different CAs for gateway-level and application-level trust when defense in depth is required

## Comparison with Direct Certificate Access

**Why use XFCC instead of direct certificate access?**
- Wasm-shim technical limitation: cannot access TLS connection attributes in current architecture
- XFCC is a well-established standard (used by Envoy, NGINX, etc.)
- Separation of concerns: TLS termination at gateway, authentication at application layer

**Potential risks**:
- Dependency on correct proxy configuration (XFCC stripping/setting)
- Header size limits (certificates can be large, especially with cert chains)

# Alternatives Considered

## Alternative 1: Fix wasm-shim to read TLS connection attributes

**Approach**: Modify wasm-shim to access the client certificate directly from Envoy's TLS connection attributes and populate `attributes.source.certificate` in the CheckRequest.

**Pros**:
- No need for XFCC header
- No API changes required to Authorino
- Potentially more secure (no header-based trust)

**Cons**:
- Significant technical complexity in wasm-shim
- May require Envoy API changes or ABI extensions
- Wasm sandbox limitations may prevent direct TLS attribute access
- Longer implementation timeline
- Still requires Gateway TLS configuration for mTLS

**Decision**: Rejected due to technical complexity and unknown feasibility.

## Alternative 2: OPA-based workaround (current state)

**Approach**: Use anonymous authentication + OPA Rego policies to parse XFCC headers (as documented in the workaround).

**Pros**:
- Works today without code changes to Kuadrant components
- Flexible policy language (can implement custom validation logic)

**Cons**:
- **Bypasses authentication phase**: Uses `anonymous` authentication, losing Authorino's authentication features
- **No multi-CA trust via label selectors**: OPA policy must implement certificate validation logic manually, cannot leverage Authorino's integration with Kubernetes Secrets for trusted CAs
- Requires deep OPA knowledge and custom policy maintenance
- Complex to implement correctly (certificate parsing, chain validation, CA trust)
- Poor user experience (expected `authentication.x509` API doesn't work)

**Decision**: This workaround should be replaced by native support. The main pain point is not the complexity of OPA, but the **loss of Authorino's native x509 authentication features**, particularly multi-CA trust and label-based certificate selection.

## Alternative 3: Automatic Gateway configuration injection

**Approach**: Have Kuadrant Operator automatically inject `spec.tls.frontend.default.validation` configuration into Gateway resources based on AuthPolicy settings.

**Pros**:
- Seamless user experience (one resource to configure)
- Automatic CA certificate management

**Cons**:
- Gateway resource mutation is complex and may conflict with other controllers
- Requires sophisticated label selectors for CA certificate discovery
- Unclear ownership model (who controls the Gateway TLS config?)
- May surprise users with unexpected Gateway modifications

**Decision**: Rejected for initial implementation. This could be a future enhancement but should be opt-in, not automatic. For now, users explicitly configure both Gateway and AuthPolicy.

## Alternative 4: Support multiple certificate sources simultaneously

**Approach**: Allow AuthConfig to try multiple sources (XFCC header, then CheckRequest attributes, etc.) until a certificate is found.

**Pros**:
- Maximum flexibility
- Automatic fallback

**Cons**:
- Ambiguous behavior (which source is actually used?)
- Security risk (could allow bypasses via header injection)
- Complex validation logic

**Decision**: Rejected. Explicit configuration is clearer and more secure.

# Implementation Plan

## Phase 1: Authorino Core

1. **API Design**:
   - Finalize `X509CertificateSource` struct design
   - Review with maintainers
   - Update API documentation

2. **Implementation**:
   - Add types to `api/v1beta3/auth_config_types.go`
   - Implement XFCC header parsing in `pkg/evaluators/identity/x509.go`
   - Handle URL-encoded PEM format
   - Parse Envoy XFCC format (may include multiple fields like `By`, `Hash`, `Cert`, `Chain`, `Subject`, `URI`, `DNS`)

3. **Testing**:
   - Unit tests for XFCC parsing
   - Unit tests for backward compatibility
   - Integration tests with mock headers

4. **Documentation**:
   - Update AuthConfig documentation
   - Add examples of XFCC-based x509 authentication

**Deliverables**:
- PR to Authorino repository
- Updated CRD manifests
- Test coverage > 80% for new code

## Phase 2: Authorino Operator

1. **CRD Updates**:
   - Sync CRD manifests from Authorino
   - Update examples and documentation

2. **Testing**:
   - Verify CRD installation
   - Test example AuthConfigs with new fields

**Deliverables**:
- PR to Authorino Operator repository
- Updated CRD manifests

## Phase 3: Kuadrant Operator

1. **AuthPolicy API**:
   - Add `X509CertificateSource` types to AuthPolicy CRD
   - Update CRD manifests
   - Update OpenAPI validation

2. **Controller Updates**:
   - Update AuthPolicy-to-AuthConfig translation logic
   - Ensure `authentication.x509.source` is propagated correctly

3. **Testing**:
   - Unit tests for API types
   - Controller tests for translation logic
   - Integration tests (if applicable)

4. **Documentation**:
   - Update AuthPolicy documentation
   - Create comprehensive examples showing all three proxy configuration tiers:
     - **Tier 1**: Gateway with `spec.tls.frontend.default.validation` (recommended)
     - **Tier 2**: Gateway with EnvoyFilter for TLS client validation (alternative)
     - **Tier 3**: Gateway with XFCC forwarding annotation (exceptional cases)
   - For each tier, show:
     - Complete Gateway/proxy configuration
     - CA certificate ConfigMap (for Tier 1/2)
     - AuthPolicy with `x509.source.header`
     - HTTPRoute binding
   - Document security considerations and trade-offs of each tier
   - Document Authorino's multi-CA trust capabilities via label selectors

**Deliverables**:
- PR to Kuadrant Operator repository
- Updated CRD manifests
- Comprehensive documentation

## Phase 4: End-to-End Testing

1. **Test Suite Updates**:
   - Add e2e tests to Kuadrant testsuite
   - Test with real Gateway implementations (Istio, Envoy Gateway)
   - Verify XFCC header propagation
   - Test certificate validation (valid certs, invalid certs, expired certs, wrong CA)

2. **Multi-cluster testing** (if applicable):
   - Verify behavior in multi-cluster Kuadrant deployments

**Deliverables**:
- PR to testsuite repository
- Documented test scenarios

## Phase 5: Release and Documentation

1. **Release Notes**:
   - Document new feature in release notes
   - Highlight that `authentication.x509` in AuthPolicy now works via XFCC header extraction
   - Note that gateway proxy configuration is user responsibility (not automated)

2. **User Guides**:
   - Create step-by-step guide for enabling X.509 authentication with all three tiers
   - **Tier 1 guide** (recommended): Gateway API v1.5+ with `spec.tls.frontend.default.validation`
   - **Tier 2 guide** (alternative): EnvoyFilter for users with older Gateway API versions
   - **Tier 3 guide** (exceptional): XFCC forwarding with security warnings
   - Include cert-manager integration examples for both gateway and Authorino CA management
   - Document Authorino's multi-CA trust via label selectors
   - Troubleshooting guide (common issues: XFCC header not populated, certificate not trusted, etc.)

3. **Migration Guide**:
   - Document migration from OPA workaround to native `authentication.x509`
   - Show how to replace OPA Rego policies with Authorino's native certificate validation
   - Explain how to configure multi-CA trust using label selectors

**Deliverables**:
- Updated documentation site
- Release notes
- Migration guide

# Testing Strategy

## Unit Tests

### Authorino
- **XFCC Header Parsing**:
  - Valid PEM certificates in XFCC format
  - URL-encoded PEM parsing
  - Multiple certificates in XFCC (chain)
  - Malformed XFCC headers
  - Empty/missing XFCC header when configured

- **Backward Compatibility**:
  - AuthConfigs without `source` field (should use default behavior)
  - AuthConfigs with `source.attribute` field

- **Certificate Validation**:
  - Valid certificates matching label selectors
  - Valid certificates not matching selectors
  - Expired certificates
  - Self-signed certificates (when allowed)

### Kuadrant Operator
- **API Translation**:
  - AuthPolicy with `x509.source.header` → AuthConfig with same config
  - AuthPolicy without `source` → AuthConfig with default behavior
  - Complex scenarios with multiple authentication methods

## Integration Tests

### Authorino
- End-to-end authentication flow with mocked XFCC headers
- Integration with Kubernetes Secret resources for CA certificates
- Multi-namespace certificate discovery

### Kuadrant Operator
- AuthPolicy creation triggers AuthConfig creation with correct fields
- AuthPolicy updates propagate to AuthConfig
- Gateway targeting and route binding

## End-to-End Tests

### Testsuite Repository

**Test Scenario 1: Happy Path (Tier 1 - Gateway API)**
- Setup:
  - Gateway with `spec.tls.frontend.default.validation` configured
  - CA certificate ConfigMap for gateway validation
  - CA certificate Secret(s) with labels for Authorino validation
  - AuthPolicy with `x509.source.header: "X-Forwarded-Client-Cert"`
  - HTTPRoute bound to AuthPolicy
- Test:
  - Client with valid certificate → 200 OK
  - Client without certificate → 401 Unauthorized (rejected at TLS layer)
  - Client with certificate signed by wrong CA → 401 Unauthorized (rejected at gateway or Authorino)

**Test Scenario 1b: Happy Path (Tier 2 - EnvoyFilter)**
- Setup:
  - Gateway with EnvoyFilter configuring TLS client validation
  - CA certificate file mounted in gateway pod
  - CA certificate Secret(s) with labels for Authorino validation
  - AuthPolicy with `x509.source.header: "X-Forwarded-Client-Cert"`
  - HTTPRoute bound to AuthPolicy
- Test:
  - Same as Scenario 1

**Test Scenario 1c: L7-Only Validation (Tier 3 - XFCC Forwarding)**
- Setup:
  - Gateway with `forwardClientCertDetails: ALWAYS_FORWARD_ONLY` annotation
  - No gateway-level TLS client validation configured
  - CA certificate Secret(s) with labels for Authorino validation
  - AuthPolicy with `x509.source.header: "X-Forwarded-Client-Cert"`
  - HTTPRoute bound to AuthPolicy
- Test:
  - Client with valid certificate in XFCC header → 200 OK (validated by Authorino only)
  - Client without XFCC header → 401 Unauthorized
  - Client with certificate signed by wrong CA in XFCC header → 403 Forbidden (Authorino rejects)
  - Verify that gateway does NOT reject invalid certificates (they pass through to Authorino)

**Test Scenario 2: Multi-CA Trust via Label Selectors**
- Setup:
  - AuthPolicy trusting multiple intermediate CAs via label selectors
  - Multiple CA Secrets with different labels (e.g., `ca-team-a`, `ca-team-b`)
- Test:
  - Client cert signed by CA with matching label → 200 OK
  - Client cert signed by CA without matching label → 403 Forbidden
  - Demonstrates Authorino's fine-grained CA trust (vs. single root CA in gateway)

**Test Scenario 3: Certificate Subject Validation**
- Setup: AuthPolicy with additional authorization rule matching specific certificate subjects (using OPA or CEL)
- Test:
  - Client with matching subject → 200 OK
  - Client with valid cert but non-matching subject → 403 Forbidden

**Test Scenario 4: Certificate Chain Validation**
- Setup: Intermediate CA certificates in XFCC
- Test: Client certificate signed by intermediate CA (full chain in XFCC)

**Test Scenario 5: Custom Header Name**
- Setup: AuthPolicy with `x509.source.header: "X-Custom-Client-Cert"`
- Modify gateway to use custom header (implementation-specific)
- Test: Verification with custom header

**Test Scenario 6: Backward Compatibility**
- Setup: AuthPolicy without `source` field (legacy behavior)
- Test: Verify existing behavior is not broken (expects cert in `attributes.source.certificate`)

**Test Scenario 7: Multi-Gateway**
- Setup: Multiple gateways with different CA configurations and validation tiers
- Test: Different AuthPolicies for different gateways (Tier 1 on one gateway, Tier 2 on another)

### Gateway Implementation Matrix

Test with multiple Gateway implementations:
- **Istio** (primary target)
- **Envoy Gateway** (if time permits)
- **Other implementations** (as available)

Verify XFCC header format consistency across implementations.

## Performance Testing

- **Overhead**: Measure latency impact of XFCC header parsing vs. direct certificate access (baseline)
- **Scale**: Test with large certificate chains
- **Concurrency**: Verify behavior under high request load

# Changes to Kuadrant Testsuite

**Repository**: https://github.com/Kuadrant/testsuite

## New Test Files

1. **`testsuite/tests/singlecluster/auth/test_x509_xfcc.py`** (or similar):
   - X.509 authentication tests with XFCC header
   - Organized by scenario (happy path, validation failures, etc.)

2. **`testsuite/fixtures/certificates.py`**:
   - Fixtures for generating test certificates (CA, client certs, expired certs)
   - May use `cryptography` library or integrate with cert-manager

## Test Fixtures

### Gateway Configuration Fixtures

#### Tier 1: Gateway API Frontend Validation
```python
@pytest.fixture
def gateway_tier1_gatewayapi_validation(cluster, module_label):
    """Gateway configured with spec.tls.frontend.default.validation (Tier 1 - recommended)"""
    gateway = Gateway.from_dict({
        "metadata": {"name": "x509-gateway-tier1", "namespace": "kuadrant-system"},
        "spec": {
            "gatewayClassName": "istio",
            "listeners": [{
                "name": "https",
                "protocol": "HTTPS",
                "port": 443,
                "tls": {
                    "mode": "Terminate",
                    "certificateRefs": [{"name": "gateway-cert"}]
                }
            }],
            "tls": {
                "frontend": {
                    "default": {
                        "validation": {
                            "caCertificateRefs": [{"name": "client-ca-bundle", "kind": "ConfigMap"}],
                            "mode": "AllowValidOnly"
                        }
                    }
                }
            }
        }
    })
    cluster.create(gateway)
    yield gateway
    cluster.delete(gateway)
```

#### Tier 2: EnvoyFilter Configuration
```python
@pytest.fixture
def gateway_tier2_envoyfilter(cluster, module_label):
    """Gateway with EnvoyFilter for TLS client validation (Tier 2 - alternative)"""
    # Create basic gateway without frontendValidation
    gateway = Gateway.from_dict({
        "metadata": {"name": "x509-gateway-tier2", "namespace": "kuadrant-system"},
        "spec": {
            "gatewayClassName": "istio",
            "listeners": [{
                "name": "https",
                "protocol": "HTTPS",
                "port": 443,
                "tls": {
                    "mode": "Terminate",
                    "certificateRefs": [{"name": "gateway-cert"}]
                }
            }]
        }
    })

    # Create EnvoyFilter for client cert validation
    envoy_filter = cluster.apply_from_dict({
        "apiVersion": "networking.istio.io/v1alpha3",
        "kind": "EnvoyFilter",
        "metadata": {"name": "gateway-client-cert", "namespace": "istio-system"},
        "spec": {
            "workloadSelector": {"labels": {"istio": "ingressgateway"}},
            "configPatches": [{
                "applyTo": "LISTENER",
                "match": {"context": "GATEWAY", "listener": {"portNumber": 443}},
                "patch": {
                    "operation": "MERGE",
                    "value": {
                        # EnvoyFilter configuration for client cert validation
                        # (implementation details omitted for brevity)
                    }
                }
            }]
        }
    })

    cluster.create(gateway)
    yield gateway
    cluster.delete(gateway)
    cluster.delete(envoy_filter)
```

#### Tier 3: XFCC Forwarding
```python
@pytest.fixture
def gateway_tier3_xfcc_forwarding(cluster, module_label):
    """Gateway with XFCC forwarding, no TLS validation (Tier 3 - exceptional)"""
    gateway = Gateway.from_dict({
        "metadata": {
            "name": "x509-gateway-tier3",
            "namespace": "kuadrant-system",
            "annotations": {
                "proxy.istio.io/config": '{"gatewayTopology": {"forwardClientCertDetails": "ALWAYS_FORWARD_ONLY"}}'
            }
        },
        "spec": {
            "gatewayClassName": "istio",
            "listeners": [{
                "name": "https",
                "protocol": "HTTPS",
                "port": 443,
                "tls": {
                    "mode": "Terminate",
                    "certificateRefs": [{"name": "gateway-cert"}]
                }
            }]
            # Note: No frontendValidation - XFCC is forwarded without gateway validation
        }
    })
    cluster.create(gateway)
    yield gateway
    cluster.delete(gateway)
```

### CA Certificate Fixture
```python
@pytest.fixture
def client_ca_bundle(cluster):
    """ConfigMap containing CA certificates for client cert validation"""
    # Generate or load CA certificate
    ca_cert = generate_ca_certificate()

    cm = ConfigMap.from_dict({
        "metadata": {"name": "client-ca-bundle", "namespace": "kuadrant-system"},
        "data": {"ca.crt": ca_cert.public_bytes()}
    })
    cluster.create(cm)
    yield cm
    cluster.delete(cm)
```

### Client Certificate Fixtures
```python
@pytest.fixture
def valid_client_cert(client_ca_bundle):
    """Generate a valid client certificate signed by the CA"""
    # Generate certificate signed by CA
    return generate_client_certificate(
        ca=client_ca_bundle,
        subject={"CN": "test-client", "O": "kuadrant"}
    )

@pytest.fixture
def expired_client_cert(client_ca_bundle):
    """Generate an expired client certificate"""
    return generate_client_certificate(
        ca=client_ca_bundle,
        not_after=datetime.now() - timedelta(days=1)
    )
```

### AuthPolicy Fixture
```python
@pytest.fixture
def x509_auth_policy(route, module_label):
    """AuthPolicy configured for X.509 authentication via XFCC header"""
    policy = AuthPolicy.from_dict({
        "metadata": {"name": "x509-policy", "namespace": "kuadrant-system"},
        "spec": {
            "targetRef": {
                "group": "gateway.networking.k8s.io",
                "kind": "HTTPRoute",
                "name": route.name()
            },
            "defaults": {
                "rules": {
                    "authentication": {
                        "x509-authn": {
                            "x509": {
                                "source": {"header": "X-Forwarded-Client-Cert"},
                                "selector": {
                                    "matchLabels": {"app": "trusted-client"}
                                }
                            }
                        }
                    }
                }
            }
        }
    })
    return policy
```

## Test Implementation

### Basic Authentication Test
```python
def test_x509_authentication_valid_cert(gateway_with_client_cert_validation,
                                         x509_auth_policy,
                                         valid_client_cert):
    """Test successful authentication with valid client certificate"""
    response = requests.get(
        "https://example.com/api",
        cert=valid_client_cert,
        verify=False  # or use proper CA verification
    )
    assert response.status_code == 200

def test_x509_authentication_no_cert(gateway_with_client_cert_validation,
                                      x509_auth_policy):
    """Test authentication failure when no client certificate provided"""
    response = requests.get("https://example.com/api", verify=False)
    # May fail at TLS layer (depending on gateway config) or at auth layer
    assert response.status_code in [401, 403]

def test_x509_authentication_expired_cert(gateway_with_client_cert_validation,
                                           x509_auth_policy,
                                           expired_client_cert):
    """Test authentication failure with expired certificate"""
    # This may fail at TLS termination or auth validation
    with pytest.raises((SSLError, requests.exceptions.SSLError)):
        requests.get(
            "https://example.com/api",
            cert=expired_client_cert,
            verify=False
        )
```

## Test Environment Setup

**Prerequisites**:
- Gateway implementation with Gateway API v1.5 support
- cert-manager (optional, for automatic certificate generation)
- Test cluster with HTTPS ingress capability

**CI/CD Integration**:
- Add new test suite to CI pipeline
- Ensure Gateway API v1.5 is installed in test clusters
- May require separate test jobs for different gateway implementations

# Open Questions and TODOs

## Open Questions

1. **XFCC Format Variations**:
   - [ ] Verify XFCC header format consistency across Istio, Envoy Gateway, and other Gateway implementations
   - [ ] Document any implementation-specific quirks or limitations
   - [ ] Decision: Should Authorino support multiple XFCC formats, or standardize on Envoy format?

2. **API Versioning**:
   - [ ] Should the new `source` field be added to Authorino v1beta3 or require a new v1beta4?
   - [ ] Recommendation: Add to v1beta3 as optional field (backward compatible)
   - [ ] Get maintainer approval

3. **Certificate Chain Handling**:
   - [ ] How should Authorino handle certificate chains in XFCC?
   - [ ] Should it validate the entire chain or only the leaf certificate?
   - [ ] Current behavior with `attributes.source.certificate`?

4. **Header Size Limits**:
   - [ ] What are the practical limits for XFCC header size?
   - [ ] Should there be a warning/error for excessively large certificate chains?
   - [ ] Document recommended maximum certificate chain length

5. **Multi-Certificate Support**:
   - [ ] Can a client present multiple certificates?
   - [ ] How does XFCC represent this (comma-separated, multiple headers)?
   - [ ] Should Authorino support selecting specific certificates from a set?

6. **Proxy Configuration Guidance**:
   - [ ] Should documentation include decision tree for choosing between Tier 1/2/3?
   - [ ] Should there be a warning/validation in AuthPolicy when using `x509.source.header` without proper gateway configuration?
   - [ ] How can users verify their gateway is properly configured for XFCC?

7. **Automatic Gateway Configuration** (Future):
   - [ ] Should Kuadrant offer opt-in automatic injection of `spec.tls.frontend.default.validation` in a future version?
   - [ ] If so, what should the API look like (annotation, dedicated field)?
   - [ ] How to avoid conflicts with user-managed Gateway configurations?
   - [ ] Should it support both Tier 1 (Gateway API) and Tier 2 (EnvoyFilter) injection?

## Implementation TODOs

### Research Phase
- [ ] Analyze XFCC header format in Envoy documentation
- [ ] Test XFCC header with Istio gateway in local environment
- [ ] Test XFCC header with Envoy Gateway (if applicable)
- [ ] Verify wasm-shim forwards XFCC header without modification
- [ ] Identify certificate chain maximum size in XFCC

### Authorino Development
- [ ] Design API types for `X509CertificateSource`
- [ ] Implement XFCC header parser
- [ ] Implement certificate extraction from XFCC
- [ ] Add backward compatibility handling
- [ ] Write unit tests for XFCC parsing
- [ ] Write integration tests with mocked headers
- [ ] Update API documentation
- [ ] Update AuthConfig examples
- [ ] Submit PR to Authorino repository
- [ ] Address code review feedback

### Authorino Operator Development
- [ ] Sync CRD manifests from Authorino
- [ ] Update example AuthConfigs
- [ ] Test CRD installation and validation
- [ ] Submit PR to Authorino Operator repository

### Kuadrant Operator Development
- [ ] Add `X509CertificateSource` types to AuthPolicy API
- [ ] Update AuthPolicy CRD manifests
- [ ] Implement AuthPolicy-to-AuthConfig translation
- [ ] Write controller unit tests
- [ ] Create comprehensive example (Gateway + AuthPolicy)
- [ ] Write documentation for AuthPolicy x509 authentication
- [ ] Document security considerations
- [ ] Document Gateway API v1.5 requirement
- [ ] Submit PR to Kuadrant Operator repository

### Testing Development
- [ ] Design e2e test scenarios
- [ ] Implement certificate generation fixtures
- [ ] Implement Gateway configuration fixtures
- [ ] Write happy path tests
- [ ] Write failure scenario tests (invalid certs, expired, wrong CA)
- [ ] Write certificate subject validation tests
- [ ] Write certificate chain tests
- [ ] Test with Istio gateway
- [ ] Test with Envoy Gateway (if applicable)
- [ ] Verify performance impact
- [ ] Submit PR to testsuite repository

### Documentation
- [ ] Write user guide for X.509 authentication with all three tiers
- [ ] Create step-by-step tutorials:
  - [ ] Tier 1: Gateway API v1.5+ with `spec.tls.frontend.default.validation` (recommended)
  - [ ] Tier 2: EnvoyFilter for older Gateway API versions (alternative)
  - [ ] Tier 3: XFCC forwarding for exceptional cases (with security warnings)
- [ ] Document multi-CA trust configuration using label selectors
- [ ] Create decision tree for choosing proxy configuration tier
- [ ] Document cert-manager integration examples (gateway CAs + Authorino CAs)
- [ ] Document troubleshooting steps:
  - [ ] XFCC header not populated
  - [ ] Certificate not trusted by Authorino
  - [ ] Gateway vs. Authorino validation failures
- [ ] Write migration guide from OPA workaround to native `authentication.x509`
- [ ] Update release notes
- [ ] Create architecture diagrams (defense in depth vs. L7-only validation)
- [ ] Document XFCC header format and security considerations for each tier

### Release
- [ ] Coordinate release versions across repos (Authorino, operators)
- [ ] Tag releases
- [ ] Publish release notes
- [ ] Announce feature in community channels
- [ ] Update official documentation site

## Future Enhancements

- [ ] **Automatic Gateway Configuration** (opt-in): Investigate automatic injection of CA certificates and TLS validation configuration into Gateway resources based on AuthPolicy settings
  - Support both Tier 1 (Gateway API `spec.tls.frontend.default.validation`) and Tier 2 (EnvoyFilter) injection
  - Require explicit user opt-in to avoid unexpected mutations
  - Address ownership and conflict resolution with user-managed configurations
- [ ] **Configuration Validation**: Add validation/warnings to AuthPolicy when `x509.source.header` is configured but gateway appears to lack proper TLS validation
- [ ] **Certificate Revocation**: Add support for CRL or OCSP-based certificate revocation checking in Authorino
- [ ] **Dynamic CA Updates**: Hot-reload CA certificates without gateway restarts
- [ ] **Certificate Metrics**: Expose Prometheus metrics for certificate validation (success/failure rates, expiration warnings, which tier was used)
- [ ] **Certificate Caching**: Cache parsed certificates in Authorino to reduce CPU overhead
- [ ] **Multi-Header Support**: Support extracting certificates from multiple headers (e.g., for proxy chains)
- [ ] **Direct TLS Attribute Access**: Long-term goal to access certificates directly from TLS connection in wasm-shim (would eliminate need for XFCC entirely)
- [ ] **Gateway API Integration**: Contribute to Gateway API to standardize certificate forwarding behavior and XFCC header format

---

## References

- [GitHub Issue #140](https://github.com/Kuadrant/architecture/issues/140) - Original feature request and workaround documentation
- [PR #141 Comment on Tier 3 Approach](https://github.com/Kuadrant/architecture/pull/141#discussion_r2906277267) - Discussion of XFCC forwarding without TLS validation
- [Gateway API v1.5 Specification](https://gateway-api.sigs.k8s.io/reference/1.5/spec/) - Frontend TLS validation (Tier 1)
- [Gateway API Frontend TLS Validation GEP](https://gateway-api.sigs.k8s.io/geps/gep-91/) - Background on the standardization of client cert validation
- [Envoy XFCC Header Documentation](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/headers#x-forwarded-client-cert) - XFCC header format specification
- [Envoy forward_client_cert_details](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/transport_sockets/tls/v3/tls.proto#extensions-transport-sockets-tls-v3-tlscontext) - Configuration for XFCC header behavior
- [Authorino X.509 Authentication](https://github.com/Kuadrant/authorino/blob/main/docs/features.md#x509-client-certificate-authentication-identityx509) - Current x509 authentication documentation
