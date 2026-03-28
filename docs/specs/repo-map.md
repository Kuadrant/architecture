# Kuadrant Repo Map

Cross-repository dependency map for the Kuadrant project. Describes what each repo owns, how repos relate, and the contracts between them.

## Dependency Flow

```
gateway-api (external)
    |
policy-machinery (library)
    |
kuadrant-operator (orchestrator)
    |
    +-- authorino-operator --> authorino
    +-- limitador-operator --> limitador
    +-- dns-operator
    +-- wasm-shim (injected as Envoy WASM filter)
    +-- cert-manager (external)
    +-- console-plugin (OpenShift UI)
```

## Repositories

### kuadrant-operator
**Role**: Central orchestrator. Translates user-facing policy CRDs into component-specific resources.

**Defines CRDs** (`kuadrant.io`):
| Kind | API Version | Targets |
|------|-------------|---------|
| AuthPolicy | v1 | Gateway, HTTPRoute |
| RateLimitPolicy | v1 | Gateway, HTTPRoute |
| DNSPolicy | v1 | Gateway |
| TLSPolicy | v1 | Gateway |
| TokenRateLimitPolicy | v1alpha1 | Gateway, HTTPRoute |
| Kuadrant | v1beta1 | (cluster config) |

**Consumes CRDs**:
| Kind | API Group | From Repo |
|------|-----------|-----------|
| AuthConfig | authorino.kuadrant.io/v1beta3 | authorino |
| Limitador | limitador.kuadrant.io/v1alpha1 | limitador-operator |
| DNSRecord | kuadrant.io/v1alpha1 | dns-operator |
| Certificate | cert-manager.io/v1 | cert-manager (external) |
| Gateway, HTTPRoute | gateway.networking.k8s.io/v1 | gateway-api (external) |
| WasmPlugin | extensions.istio.io | istio (external) |

**Go module imports**:
| Module | Version |
|--------|---------|
| github.com/kuadrant/authorino | v0.22.0 |
| github.com/kuadrant/authorino-operator | v0.21.0 |
| github.com/kuadrant/limitador-operator | v0.15.0 |
| github.com/kuadrant/dns-operator | (commit-pinned) |
| github.com/kuadrant/policy-machinery | v0.7.1 |

**Key controllers**:
- AuthConfigsReconciler: AuthPolicy -> AuthConfig
- LimitadorLimitsReconciler: RateLimitPolicy -> Limitador limits (via ConfigMap)
- IstioExtensionReconciler: Policies -> WasmPlugin (Istio)
- EnvoyGatewayExtensionReconciler: Policies -> EnvoyGateway extensions
- DNSPolicyReconciler: DNSPolicy -> DNSRecord
- TLSPolicyReconciler: TLSPolicy -> cert-manager Certificate

---

### authorino
**Role**: Authorization service. Envoy ext_authz gRPC server.

**Defines CRDs** (`authorino.kuadrant.io`):
| Kind | API Version |
|------|-------------|
| AuthConfig | v1beta3 (storage), v1beta2 (legacy) |

**Protocol**: Envoy `ext_authz` gRPC on port 50051, raw HTTP on port 5001.

**Auth methods**: API key, JWT/OIDC, OAuth2 introspection, K8s TokenReview, x509 mTLS, OPA/Rego, SpiceDB, pattern matching.

**Go module imports**: No Kuadrant dependencies. Imports envoyproxy/go-control-plane, OPA, SpiceDB SDKs.

---

### authorino-operator
**Role**: Deploys and manages Authorino instances.

**Defines CRDs** (`operator.authorino.kuadrant.io`):
| Kind | API Version |
|------|-------------|
| Authorino | v1beta1 |

**Manages**: Deployment, Services (auth/OIDC/metrics), ServiceAccount, ClusterRole/RoleBinding for Authorino.

**Go module imports**: No Kuadrant dependencies. Manages Authorino purely as a container image.

---

### limitador
**Role**: Rate-limiting service (Rust). Implements Envoy RLS protocol.

**gRPC API** (port 8081):
- `CheckRateLimit` / `Report` (Kuadrant RLS extension: `kuadrantrls.proto`)
- Standard Envoy `envoy.service.ratelimit.v3.RateLimitService`

**HTTP API** (port 8080):
- `/check_and_report`, `/check`, `/report` (rate limit operations)
- `/limits/{namespace}`, `/counters/{namespace}` (inspection)
- `/status`, `/metrics`

**Config format** (YAML, hot-reloadable):
```yaml
- namespace: example.org
  max_value: 100
  seconds: 60
  conditions: ["descriptors[0]['req.method'] == 'POST'"]
  variables: ["descriptors[0]['user_id']"]
```

**Storage backends**: memory, disk, redis, redis_cached.

---

### limitador-operator
**Role**: Deploys and manages Limitador instances.

**Defines CRDs** (`limitador.kuadrant.io`):
| Kind | API Version |
|------|-------------|
| Limitador | v1alpha1 |

**Manages**: Deployment, Service (headless), ConfigMap (limits YAML), PVC (disk storage), PDB.

**Translates**: CRD spec -> Limitador CLI args + mounted ConfigMap at `/home/limitador/etc/limitador-config.yaml`.

**Go module imports**: No Kuadrant dependencies. Manages Limitador purely as a container image.

---

### dns-operator
**Role**: Manages DNS records across cloud providers (AWS Route53, Google Cloud DNS, Azure DNS, CoreDNS).

**Defines CRDs** (`kuadrant.io`):
| Kind | API Version |
|------|-------------|
| DNSRecord | v1alpha1 |
| DNSHealthCheckProbe | v1alpha1 |

**Features**: Multi-cluster delegation, health-aware DNS, TXT registry for ownership.

**Go module imports**: `sigs.k8s.io/external-dns` (Kuadrant fork), `sigs.k8s.io/multicluster-runtime`.

---

### policy-machinery
**Role**: Shared Go library. Provides the policy attachment framework used by kuadrant-operator.

**Exports**:
- `Policy` interface (policies that target objects and merge)
- `Targetable` interface (Gateways, HTTPRoutes)
- `MergeStrategy` function type
- Topology builder (DAG of Gateway -> HTTPRoute -> Policy relationships)

**Consumed by**: kuadrant-operator only.

**Go module imports**: `sigs.k8s.io/gateway-api` v1.2.1, controller-runtime.

---

### wasm-shim
**Role**: Proxy-Wasm Rust module (compiled to WASM). Deployed as an Envoy filter via Istio WasmPlugin or EnvoyGateway extension.

**Config format** (JSON, injected by kuadrant-operator):
```json
{
  "actionSets": [{
    "name": "...",
    "routeRuleConditions": {
      "hostnames": ["api.example.com"],
      "predicates": ["request.method == 'GET'"]
    },
    "actions": [{
      "service": "ratelimit-service",
      "scope": "my-namespace/my-rlp",
      "predicates": ["..."],
      "conditionalData": [{"key": "...", "expression": "..."}]
    }]
  }]
}
```

**Dispatches gRPC to**:
- limitador: `CheckRateLimit` / `Report` (Kuadrant RLS)
- authorino: Envoy `ext_authz` `Check`

**CEL predicates**: Evaluated per-request to determine which actions fire.

---

### console-plugin
**Role**: OpenShift Console dynamic plugin (React/TypeScript). Provides UI for Kuadrant resources.

**Displays CRDs**:
- Gateway, HTTPRoute (gateway-api)
- AuthPolicy, RateLimitPolicy, DNSPolicy, TLSPolicy (kuadrant.io/v1)
- PlanPolicy, APIProduct, APIKeyRequest (extensions.kuadrant.io/v1alpha1)

**Features**: Policy topology visualization, form/YAML editors, gateway health dashboard.

---

### architecture
**Role**: RFCs and design documents. Central coordination for cross-repo architecture decisions.

**Key RFCs**: Policy Machinery (0011), RLP v2 (0001), DNSPolicy (0003/0005), mTLS (0012), DNS Failover (0014).

---

## Cross-Repo Contracts

### kuadrant-operator <-> authorino
- **Object**: `AuthConfig` CR (authorino.kuadrant.io/v1beta3)
- **Producer**: kuadrant-operator (AuthConfigsReconciler)
- **Consumer**: authorino (watches AuthConfig, builds in-memory index)
- **Contract**: kuadrant-operator creates AuthConfig CRs with host matching, auth rules, and metadata. Authorino enforces them on ext_authz requests.

### kuadrant-operator <-> limitador-operator
- **Object**: Limitador CR (limitador.kuadrant.io/v1alpha1) + limits ConfigMap
- **Producer**: kuadrant-operator (LimitadorLimitsReconciler)
- **Consumer**: limitador-operator (reconciles Deployment + ConfigMap)
- **Contract**: kuadrant-operator writes rate limit definitions to the Limitador CR's `spec.limits`. limitador-operator syncs these to a ConfigMap mounted into Limitador pods.

### kuadrant-operator <-> wasm-shim
- **Object**: WasmPlugin CR (extensions.istio.io) containing JSON config
- **Producer**: kuadrant-operator (IstioExtensionReconciler)
- **Consumer**: wasm-shim (parses JSON config at filter init)
- **Contract**: kuadrant-operator serializes ActionSets with CEL predicates, service references, and conditional data into the WasmPlugin's `spec.pluginConfig`. wasm-shim uses these to route auth/ratelimit calls.
- **Invariant**: Descriptor keys in wasm-shim config MUST match limit definitions written to Limitador. Scope values MUST match between wasm-shim actions and Limitador limit namespaces.

### kuadrant-operator <-> dns-operator
- **Object**: `DNSRecord` CR (kuadrant.io/v1alpha1)
- **Producer**: kuadrant-operator (DNSPolicyReconciler)
- **Consumer**: dns-operator (reconciles DNS records with cloud providers)
- **Contract**: kuadrant-operator creates DNSRecord CRs with endpoints, provider references, and health check config. dns-operator syncs these to the configured DNS provider.

### kuadrant-operator <-> cert-manager
- **Object**: `Certificate` CR (cert-manager.io/v1)
- **Producer**: kuadrant-operator (TLSPolicyReconciler)
- **Consumer**: cert-manager (issues/renews TLS certificates)
- **Contract**: kuadrant-operator creates Certificate CRs referencing an Issuer/ClusterIssuer. cert-manager provisions certificates and stores them in Secrets.

### wasm-shim <-> limitador
- **Protocol**: gRPC (Kuadrant RLS extension)
- **Service**: `kuadrant.service.ratelimit.v1.RateLimitService`
- **Methods**: `CheckRateLimit`, `Report`
- **Data**: Scope + descriptors (key-value pairs from CEL expressions) + hits addend

### wasm-shim <-> authorino
- **Protocol**: gRPC (Envoy ext_authz)
- **Service**: `envoy.service.auth.v3.Authorization`
- **Method**: `Check`
- **Data**: HTTP request attributes (method, path, headers, host)

## Gateway Provider Integration

kuadrant-operator supports two gateway providers, each with its own extension mechanism:

| Provider | Extension CRD | Reconciler | wasm-shim Injection |
|----------|--------------|------------|---------------------|
| Istio | WasmPlugin (extensions.istio.io) | IstioExtensionReconciler | Via Istio WasmPlugin CR |
| Envoy Gateway | EnvoyExtensionPolicy | EnvoyGatewayExtensionReconciler | Via EG extension policy |

Both paths produce equivalent wasm-shim configurations; only the delivery mechanism differs.
