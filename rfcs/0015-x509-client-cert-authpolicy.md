# X.509 Client Certificate Authentication in Kuadrant AuthPolicy

- Feature Name: `x509-client-cert-authpolicy`
- Status: **Draft**
- Start Date: 2026-03-03
- Issue tracking: [Kuadrant/architecture#140](https://github.com/Kuadrant/architecture/issues/140)

# Summary

Enable X.509 client certificate authentication in Kuadrant's AuthPolicy by allowing Authorino to extract client certificates from the X-Forwarded-Client-Cert (XFCC) header and leverage Gateway API v1.5's standardized frontend TLS validation configuration for mTLS termination at the gateway level.

# Goals

- Support X.509 client certificate authentication in Kuadrant AuthPolicy, making the inherited API from Authorino's AuthConfig functionally operational in Kuadrant deployments
- Provide a vendor-agnostic solution using Gateway API v1.5 standard features (instead of Istio-specific EnvoyFilter resources)
- Enable Authorino to extract and validate client certificates from the XFCC header populated by the proxy
- Allow users to configure which HTTP header contains the client certificate (defaulting to X-Forwarded-Client-Cert)
- Maintain backward compatibility with existing AuthConfig/AuthPolicy configurations
- Eliminate the need for manual workarounds involving OPA Rego policies and anonymous authentication

# Non-goals

- Support for certificate extraction directly from the TLS connection attributes in the wasm-shim (blocked by current technical limitations)
- Automatic injection of CA certificates into Gateway resources (this may be addressed in future iterations)
- Support for Gateway API versions prior to v1.4 (feature was Experimental before v1.5)
- Custom certificate validation logic beyond what Authorino already provides
- Certificate revocation checking (CRL/OCSP) - this remains the responsibility of the gateway proxy

# Problem Statement

Kuadrant's AuthPolicy API inherits the `authentication.x509` configuration from Authorino's AuthConfig CRD, creating the misleading appearance that X.509 client certificate authentication is supported in Kuadrant. However, this feature is non-functional in practice due to two critical gaps:

1. **Gateway TLS Configuration Gap**: Kuadrant does not configure the gateway listeners with TLS client certificate validation settings, meaning the gateway never requests or validates client certificates during the TLS handshake.

2. **Certificate Propagation Gap**: The wasm-shim cannot read the client certificate in PEM format from the TLS connection attributes, preventing it from populating the `attributes.source.certificate` field in the external authorization CheckRequest to Authorino.

As a result, users attempting to use `spec.rules.authentication.x509` in their AuthPolicy resources find that authentication fails because Authorino never receives the client certificate data.

## Current Workaround

The temporary workaround documented in issue #140 involves:
- Manually configuring TLS context validation in gateway listeners using Gateway API provider-specific resources (e.g. Istio EnvoyFilter)
- Using `anonymous` authentication in AuthPolicy
- Creating OPA Rego authorization policies that parse and validate certificates from the XFCC header
- Manually managing CA certificate ConfigMaps and updating gateway configurations

This workaround is fragile, vendor-specific (Istio, Envoy Gateway), and requires deep knowledge of Envoy configuration.

# Proposed Solution

The solution consists of three coordinated changes across the Kuadrant ecosystem:

## 1. Extend Authorino's AuthConfig API

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

## 2. Propagate API Changes Through the Stack

### Authorino Operator
- Update the AuthConfig CRD manifests hosted in the authorino-operator repository to include the new `source` field
- No controller logic changes required (Authorino itself handles the certificate extraction)

### Kuadrant Operator
- Update the AuthPolicy CRD to expose the new `authentication.x509.source` field
- Update the AuthPolicy-to-AuthConfig translation logic to propagate the `source` configuration
- No changes to the wasm-shim required (certificate extraction happens entirely at the Authorino layer)

## 3. Gateway TLS Configuration via Gateway API

Instead of using provider-specific resources to configure the Gateway (e.g. Istio EnvoyFilter), leverage Gateway API v1.5's standardized `spec.tls.frontend.default.validation` configuration:

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

**Key aspects**:
- `spec.tls.frontend.default.validation.caCertificateRefs` points to ConfigMap(s) containing trusted CA certificates (PEM format under `ca.crt` key)
- `spec.tls.frontend.default.validation.mode` specifies the validation mode (`AllowValidOnly` or `AllowInsecureFallback`)
- Supported in Gateway API v1.4 (Experimental) and v1.5+ (Standard)
- Vendor-agnostic: works with any Gateway API implementation (Istio, Envoy Gateway, etc.)
- Gateway implementation ensures the XFCC header is populated with the validated client certificate

**User Responsibility**:
Users must manually configure the Gateway resource with appropriate `spec.tls.frontend.default.validation` settings. This is intentional - gateway TLS configuration is infrastructure-level configuration that should be explicit and visible.

**Future Enhancement** (out of scope for this RFC):
A future iteration could introduce automatic injection of CA certificate references into Gateway configurations based on label selectors specified in AuthPolicy resources, similar to the approach proposed in the original issue #140.

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
│  │ - Validates client cert against CA    │  │
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

## Key Architectural Principles

1. **Separation of Concerns**: TLS termination and certificate validation at the gateway level; certificate-based authentication at the application level (Authorino)
2. **Standard Protocols**: Uses industry-standard XFCC header format for certificate propagation
3. **Vendor Neutrality**: Relies on Gateway API standard features, not vendor-specific extensions
4. **Explicit Configuration**: Gateway TLS configuration remains explicit and user-managed

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

1. **Gateway-Level Validation**: The gateway validates the client certificate against the CA bundle configured in `spec.tls.frontend.default.validation.caCertificateRefs`
   - Only certificates signed by trusted CAs are accepted
   - Invalid or expired certificates are rejected at the TLS layer

2. **Authorino-Level Validation**: Authorino performs additional application-level validation
   - Integration with Kubernetes Secret resources for trusted certificates, via label selectors
   - Certificate parsing and validation
   - CA trust verification (matches a trusted CA from selectors)

**Defense in Depth**: Both layers must succeed for authentication to pass.

## XFCC Header Security

**Critical Security Consideration**: The XFCC header must be trusted and cannot be spoofed by clients.

**Mitigation**:
1. The gateway proxy (Envoy/Istio) **must** be configured to:
   - Strip any XFCC headers from incoming client requests (prevent spoofing)
   - Only set XFCC header after successful TLS validation

2. This is standard behavior in Envoy when `forward_client_cert_details` is configured properly

3. Documentation must explicitly warn users:
   - XFCC header source must be the trusted gateway proxy
   - Do not use `source.header` if the header could be set by untrusted sources
   - When in doubt, verify proxy configuration

## CA Certificate Management

**User Responsibility**:
- Users must securely manage CA certificate ConfigMaps referenced in Gateway `spec.tls.frontend.default.validation`
- Rotation of CA certificates requires updating ConfigMaps and potentially restarting gateway pods
- Namespace isolation: CA ConfigMaps should be in the same namespace as the Gateway or explicitly allowed via ReferenceGrant

**Best Practices** (to be documented):
- Use cert-manager for automatic CA certificate management
- Implement certificate rotation strategies
- Monitor certificate expiration

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
- Works today without code changes
- Flexible policy language

**Cons**:
- Requires deep OPA knowledge
- Does not leverage Authorino's native x509 authentication
- Poor user experience

**Decision**: This is the current workaround but should be replaced by native support.

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
   - Create comprehensive example showing:
     - Gateway configuration with spec.tls.frontend.default.validation
     - CA certificate ConfigMap
     - AuthPolicy with x509.source.header
     - HTTPRoute binding
   - Document security considerations

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
   - Highlight Gateway API v1.5 requirement

2. **User Guides**:
   - Create step-by-step guide for enabling X.509 authentication
   - Include cert-manager integration example
   - Troubleshooting guide

3. **Migration Guide**:
   - Document migration from OPA workaround to native support

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

**Test Scenario 1: Happy Path**
- Setup:
  - Gateway with `spec.tls.frontend.default.validation` configured
  - CA certificate ConfigMap
  - AuthPolicy with `x509.source.header: "X-Forwarded-Client-Cert"`
  - HTTPRoute bound to AuthPolicy
- Test:
  - Client with valid certificate → 200 OK
  - Client without certificate → 401 Unauthorized
  - Client with certificate signed by wrong CA → 401 Unauthorized

**Test Scenario 2: Certificate Subject Validation**
- Setup: AuthPolicy with additionl authorization rule matching specific certificate subjects (using OPA or CEL)
- Test:
  - Client with matching subject → 200 OK
  - Client with valid cert but non-matching subject → 403 Forbidden

**Test Scenario 3: Certificate Chain Validation**
- Setup: Intermediate CA certificates
- Test: Client certificate signed by intermediate CA (full chain in XFCC)

**Test Scenario 4: Custom Header Name**
- Setup: AuthPolicy with `x509.source.header: "X-Custom-Client-Cert"`
- Modify gateway to use custom header
- Test: Verification with custom header

**Test Scenario 5: Backward Compatibility**
- Setup: AuthPolicy without `source` field
- Test: Verify existing behavior is not broken

**Test Scenario 6: Multi-Gateway**
- Setup: Multiple gateways with different CA configurations
- Test: Different AuthPolicies for different gateways

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

### Gateway Configuration Fixture
```python
@pytest.fixture
def gateway_with_client_cert_validation(cluster, module_label):
    """Gateway configured with spec.tls.frontend.default.validation for client certificates"""
    gateway = Gateway.from_dict({
        "metadata": {"name": "x509-gateway", "namespace": "kuadrant-system"},
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

6. **Automatic Gateway Configuration** (Future):
   - [ ] Should Kuadrant offer automatic injection of `spec.tls.frontend.default.validation` in a future version?
   - [ ] If so, what should the API look like?
   - [ ] How to avoid conflicts with user-managed Gateway configurations?

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
- [ ] Write user guide for X.509 authentication
- [ ] Create step-by-step tutorial with cert-manager
- [ ] Document troubleshooting steps
- [ ] Write migration guide from OPA workaround
- [ ] Update release notes
- [ ] Create architecture diagrams
- [ ] Document XFCC header format and security considerations

### Release
- [ ] Coordinate release versions across repos (Authorino, operators)
- [ ] Tag releases
- [ ] Publish release notes
- [ ] Announce feature in community channels
- [ ] Update official documentation site

## Future Enhancements

- [ ] **Automatic CA Injection**: Investigate automatic injection of CA certificates into Gateway `spec.tls.frontend.default.validation` based on AuthPolicy configuration
- [ ] **Certificate Revocation**: Add support for CRL or OCSP-based certificate revocation checking
- [ ] **Dynamic CA Updates**: Hot-reload CA certificates without gateway restarts
- [ ] **Certificate Metrics**: Expose Prometheus metrics for certificate validation (success/failure rates, expiration warnings)
- [ ] **Certificate Caching**: Cache parsed certificates to reduce CPU overhead
- [ ] **Multi-Header Support**: Support extracting certificates from multiple headers (e.g., for proxy chains)
- [ ] **Direct TLS Attribute Access**: Long-term goal to access certificates directly from TLS connection in wasm-shim
- [ ] **Gateway API Integration**: Contribute to Gateway API to standardize certificate forwarding behavior

---

## References

- [GitHub Issue #140](https://github.com/Kuadrant/architecture/issues/140)
- [Gateway API v1.5 Specification](https://gateway-api.sigs.k8s.io/reference/1.5/spec/)
- [Envoy XFCC Header Documentation](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/headers#x-forwarded-client-cert)
- [Authorino X.509 Authentication](https://github.com/Kuadrant/authorino/blob/main/docs/features.md#x509-client-certificate-authentication-identityx509)
- [Gateway API Frontend TLS Validation](https://gateway-api.sigs.k8s.io/geps/gep-91/)
