# Cross-Cutting Conventions

Shared patterns, naming conventions, and standards used consistently across Kuadrant repositories.

## API Groups and Versioning

| API Group | Owner repo | Versions | CRDs |
|-----------|-----------|----------|------|
| `kuadrant.io` | kuadrant-operator | v1, v1beta1, v1alpha1 | AuthPolicy, RateLimitPolicy, DNSPolicy, TLSPolicy, TokenRateLimitPolicy, Kuadrant |
| `kuadrant.io` | dns-operator | v1alpha1 | DNSRecord, DNSHealthCheckProbe |
| `authorino.kuadrant.io` | authorino | v1beta3, v1beta2 | AuthConfig |
| `operator.authorino.kuadrant.io` | authorino-operator | v1beta1 | Authorino |
| `limitador.kuadrant.io` | limitador-operator | v1alpha1 | Limitador |

**Stability convention**:
- `v1` — stable, no breaking changes
- `v1beta1` — beta, may change
- `v1alpha1` — alpha, expect breaking changes

No conversion webhooks — version upgrades are handled in-place.

## Labels

### Well-known labels (`kuadrant.io/`)

| Label | Value | Purpose | Used by |
|-------|-------|---------|---------|
| `kuadrant.io/managed-by` | `kuadrant` | Marks operator-managed resources | kuadrant-operator |
| `kuadrant.io/auth` | `true` | Auth-related resources (AuthConfig) | kuadrant-operator |
| `kuadrant.io/ratelimit` | `true` | Rate-limit resources | kuadrant-operator |
| `kuadrant.io/tokenratelimit` | `true` | Token rate-limit resources | kuadrant-operator |
| `kuadrant.io/tracing` | `true` | Tracing resources | kuadrant-operator |
| `kuadrant.io/listener-name` | `<name>` | Listener name on DNSRecords | kuadrant-operator |
| `kuadrant.io/topology` | — | Topology-related objects | kuadrant-operator |
| `kuadrant.io/observability` | — | Observability resources | kuadrant-operator |
| `kuadrant.io/developerportal` | — | Developer portal resources | kuadrant-operator |
| `kuadrant.io/default-provider` | `true` | Default DNS provider Secret | dns-operator |
| `kuadrant.io/multicluster-kubeconfig` | `true` | Multi-cluster connection Secrets | dns-operator |
| `kuadrant.io/authoritative-record` | `true` | Authoritative DNSRecord (primary) | dns-operator |
| `kuadrant.io/health-probes-owner` | `<name>` | Links probes to DNSRecords | dns-operator |
| `authorino.kuadrant.io/managed-by` | `authorino` | Secrets managed by Authorino | authorino |

### Standard Kubernetes labels

Applied via `CommonLabels()` in kuadrant-operator:

```yaml
app: kuadrant
app.kubernetes.io/component: kuadrant
app.kubernetes.io/managed-by: kuadrant-operator
app.kubernetes.io/instance: kuadrant
app.kubernetes.io/name: kuadrant
app.kubernetes.io/part-of: kuadrant
```

### Policy-affected condition labels

Dynamic label pattern on Gateway API resources: `kuadrant.io/{PolicyKind}Affected` (e.g., `kuadrant.io/AuthPolicyAffected`).

## Annotations

| Annotation | Purpose | Used by |
|-----------|---------|---------|
| `kuadrant.io/delete` | Marks resource for deletion cleanup | kuadrant-operator |
| `extensions.kuadrant.io/trigger-time` | Extension trigger timestamp | kuadrant-operator |
| `extensions.kuadrant.io/trigger-reason` | Extension trigger reason | kuadrant-operator |

## Finalizers

**Naming pattern**: `kuadrant.io/{resource-name}`

| Finalizer | Resource | Repo |
|-----------|----------|------|
| `kuadrant.io/dns-record` | DNSRecord | dns-operator |
| `kuadrant.io/dnshealthcheckprobe` | DNSHealthCheckProbe | dns-operator |
| `kuadrant.io/extensions` | Extension resources | kuadrant-operator |
| `kuadrant.io/developerportal` | Developer portal | kuadrant-operator |

All use `controllerutil.AddFinalizer()` / `controllerutil.RemoveFinalizer()`.

## Status Conditions

### Standard condition types

| Type | Meaning | Used by |
|------|---------|---------|
| `Accepted` | Policy validated and attached to target | All policy CRDs |
| `Enforced` | Sub-resources created and ready | All policy CRDs |
| `SubResourcesHealthy` | Health probes passing | DNSPolicy |
| `Ready` | Resource fully reconciled | AuthConfig, Limitador, Authorino, DNSRecord |
| `Available` | Resource has linked hosts | AuthConfig |
| `Healthy` | Health checks passing | DNSRecord |
| `ReadyForDelegation` | Ready for multi-cluster delegation | DNSRecord |
| `Active` | Member of active failover group | DNSRecord |

### Standard condition reasons

Defined in `internal/kuadrant/conditions.go`:

```go
PolicyReasonEnforced             // fully enforced
PolicyReasonOverridden           // overridden by another policy
PolicyReasonUnknown              // unknown state
PolicyReasonMissingDependency    // required CRD/operator not installed
PolicyReasonMissingResource      // required resource not found
PolicyReasonInvalidCelExpression // CEL validation failure
```

From Gateway API:
```go
PolicyReasonTargetNotFound  // targetRef doesn't resolve
PolicyReasonInvalid         // validation error
PolicyReasonConflicted      // older policy already controls target
```

### Condition helpers

```go
AcceptedCondition(policy, err) *metav1.Condition
EnforcedCondition(policy, err, fullyEnforced) *metav1.Condition
meta.SetStatusCondition(&conditions, condition)
meta.IsStatusConditionTrue(conditions, "Type")
```

## Error Handling

### PolicyError interface

All policy errors implement `PolicyError` with a `Reason()` method mapping to Gateway API condition reasons:

| Error type | Reason | When |
|-----------|--------|------|
| `ErrTargetNotFound` | TargetNotFound | Policy target missing from topology |
| `ErrInvalid` | Invalid | Validation failure (e.g., issuer not found) |
| `ErrConflict` | Conflicted | Older policy already controls same target |
| `ErrDependencyNotInstalled` | MissingDependency | Required CRD not installed |
| `ErrSystemResource` | MissingResource | Required system resource missing |
| `ErrCelValidation` | InvalidCelExpression | CEL expression validation failure |
| `ErrOverridden` | Overridden | Policy overridden by higher-priority policy |
| `ErrNoRoutes` | Unknown | Policy not in path to any routes |
| `ErrOutOfSync` | Unknown | Components not synced |

### Wrapping pattern

```go
var policyErr PolicyError
if !errors.As(err, &policyErr) {
    policyErr = NewErrUnknown(p.Kind(), err)
}
```

## Container Images

### Registry

All images published to `quay.io/kuadrant/`:

| Image | Repo |
|-------|------|
| `quay.io/kuadrant/kuadrant-operator` | kuadrant-operator |
| `quay.io/kuadrant/authorino` | authorino |
| `quay.io/kuadrant/authorino-operator` | authorino-operator |
| `quay.io/kuadrant/limitador` | limitador |
| `quay.io/kuadrant/limitador-operator` | limitador-operator |
| `quay.io/kuadrant/dns-operator` | dns-operator |
| `quay.io/kuadrant/wasm-shim` | wasm-shim |
| `quay.io/kuadrant/console-plugin` | console-plugin |

**Bundle images**: `{image}-bundle:TAG`
**Catalog images**: `{image}-catalog:TAG`

### Tagging convention

```makefile
# Semantic version → v-prefixed tag
VERSION=1.2.3 → IMAGE_TAG=v1.2.3

# Non-semantic → dev tag
VERSION=main → IMAGE_TAG=dev (or latest)

# Default (no version) → latest
VERSION=0.0.0 → IMAGE_TAG=latest
```

## Toolchain

### Go repositories

| Convention | Value |
|-----------|-------|
| Go version | 1.25.5 |
| Controller framework | controller-runtime |
| Test framework | Ginkgo v2 + Gomega |
| Linter | golangci-lint |
| Code generation | controller-gen (kubebuilder) |
| Build | `go build` with ldflags for version injection |

### Rust repositories (limitador, wasm-shim)

| Convention | Value |
|-----------|-------|
| Workspace | Multi-crate workspace with resolver v2 |
| Release profile | LTO enabled, `codegen-units = 1` |
| Test | `cargo test --all-features` |
| Benchmarks | Criterion |

## Makefile Targets

Standard across all operator repos:

| Target | Purpose |
|--------|---------|
| `build` | Compile binary |
| `test` / `test-unit` | Unit tests |
| `test-integration` | Integration tests (requires cluster) |
| `test-e2e` | End-to-end tests |
| `generate` | Generate DeepCopy/CRD code |
| `manifests` | Generate CRD/RBAC manifests |
| `fmt` / `vet` / `lint` | Code quality |
| `docker-build` / `docker-push` | Container image build/push |
| `install` / `uninstall` | CRD install/remove |
| `deploy` / `undeploy` | Operator deploy/remove |
| `bundle` | Generate OLM bundle |
| `helm-build` | Generate Helm chart |
| `local-setup` / `local-cleanup` | Kind cluster lifecycle |
| `verify-manifests` | CI: ensure manifests up-to-date |
| `verify-bundle` | CI: ensure bundle up-to-date |
| `verify-fmt` | CI: check code formatting |

## Environment Variables

### kuadrant-operator

| Variable | Default | Purpose |
|----------|---------|---------|
| `LOG_LEVEL` | `info` | Logging level |
| `LOG_MODE` | `production` | Log format (production/development) |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | (empty) | OpenTelemetry collector endpoint |
| `OTEL_EXPORTER_OTLP_INSECURE` | `false` | Allow insecure OTLP connection |
| `OTEL_SERVICE_NAME` | `kuadrant-operator` | Service name for telemetry |
| `AUTH_SERVICE_TIMEOUT` | `200ms` | WASM auth service timeout |
| `AUTH_SERVICE_FAILURE_MODE` | `deny` | WASM auth failure behavior |
| `DNS_DEFAULT_TTL` | `300` | Default DNS record TTL (seconds) |
| `DNS_DEFAULT_LB_TTL` | `60` | Default load-balanced DNS TTL (seconds) |

### Common operator flags

| Flag | Default | Purpose |
|------|---------|---------|
| `--metrics-bind-address` | `:8080` | Prometheus metrics endpoint |
| `--health-probe-bind-address` | `:8081` | Health/readiness probes |
| `--leader-elect` | `false` | Enable leader election |

## Observability

### Metrics (Prometheus)

```go
kuadrant_policies_total{kind}              // gauge: total policies by kind
kuadrant_policies_enforced{kind, status}   // gauge: enforced policies by kind and status
```

Exposed on `:8080/metrics` via controller-runtime.

### Tracing (OpenTelemetry)

- W3C Trace Context propagation
- OTLP exporter (gRPC) when `OTEL_EXPORTER_OTLP_ENDPOINT` set
- Spans on reconciliation workflows via `traceReconcileFunc()`
- Span attributes include: git SHA, dirty state, Go version

### Logging

- Dual output: Zap (console) + OTLP (remote, if configured)
- Structured JSON in production mode
- Standard `logr.Logger` interface throughout

## CEL (Common Expression Language)

### Root bindings (available in all CEL expressions)

```go
request       // request attributes (method, path, headers, host, etc.)
source        // source attributes
destination   // destination attributes
connection    // connection attributes
```

### Wasm-shim specific functions

```
requestBodyJSON("/path")       // parse request body JSON
responseBodyJSON("/path")      // parse response body JSON (TokenRateLimitPolicy)
queryMap(request.query)        // parse query string to map
```

### Validation

CEL expressions are validated at reconcile time. Invalid expressions produce `ErrCelValidation` errors with reason `InvalidCelExpression`.

## Conflict Resolution

All policies follow the same conflict resolution rule: **oldest policy wins** (by `metadata.creationTimestamp`). Newer conflicting policies get `Accepted=False` with reason `Conflicted`.

## Owner References

All generated sub-resources set `ownerReferences` to the creating policy/operator resource. This enables:
- Automatic garbage collection on parent deletion
- Ownership tracking in topology
- `utils.IsOwnedBy()` checks for link resolution

## Project Layout (Go operators)

```
├── api/
│   ├── v1/                    # stable CRD types
│   ├── v1beta1/               # beta CRD types
│   └── v1alpha1/              # alpha CRD types
├── cmd/main.go                # entrypoint, flag parsing, logger setup
├── internal/
│   ├── controller/            # reconcilers, workflows, state_of_the_world
│   └── {component}/           # component-specific helpers
├── pkg/                       # shared packages (exported)
├── config/
│   ├── crd/                   # generated CRD manifests
│   ├── manager/               # operator Deployment
│   ├── rbac/                  # RBAC manifests
│   └── samples/               # example CRs
├── bundle/                    # OLM bundle
├── charts/                    # Helm charts
├── tests/
│   ├── common/                # shared test helpers
│   ├── istio/                 # Istio-specific integration tests
│   ├── envoygateway/          # EnvoyGateway-specific integration tests
│   └── bare_k8s/              # pure K8s tests
├── Makefile
├── go.mod
└── Dockerfile
```

## Test Conventions

### Timeouts

```go
TimeoutShort        = 5s
TimeoutMedium       = 10s
TimeoutLongerMedium = 15s
TimeoutLong         = 30s
RetryIntervalMedium = 250ms
```

### Async assertions

```go
Eventually(func(g Gomega) {
    obj := &v1.SomeResource{}
    g.Expect(client.Get(ctx, key, obj)).To(Succeed())
    g.Expect(obj.Status.Conditions).To(ContainElement(
        MatchFields(IgnoreExtras, Fields{
            "Type":   Equal("Ready"),
            "Status": Equal(metav1.ConditionTrue),
        }),
    ))
}).WithContext(ctx).Should(Succeed())
```

### Build tags

```go
//go:build integration  // integration tests
//go:build unit          // unit tests (if separated)
```

## Build Info Injection

All Go operators inject build metadata via ldflags:

```go
var (
    gitSHA  string  // git commit SHA
    dirty   string  // "true" if uncommitted changes
    version string  // semantic version
)
```

Used in OTEL resource attributes and operator status reporting.
