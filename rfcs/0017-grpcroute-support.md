# Feature: GRPCRoute Support for Kuadrant Policies

- Feature Name: `grpcroute-support`
- Status: **Draft**
- Start Date: 2026-03-13
- Issue tracking: [Kuadrant/architecture#156](https://github.com/Kuadrant/architecture/issues/156)

## Summary

This design document outlines the implementation of GRPCRoute support in the Kuadrant ecosystem. GRPCRoute is a Gateway API resource (GA since v1.1.0) that provides gRPC-native routing semantics. This work enables all Kuadrant policies except TokenRateLimitPolicy to work with GRPCRoute resources:

- **Data plane policies** (AuthPolicy, RateLimitPolicy) can target GRPCRoute resources via the standard `targetRef` field
- **Extension policies** (OIDCPolicy, PlanPolicy, TelemetryPolicy) can target GRPCRoute resources via the standard `targetRef` field
- **Infrastructure policies** (DNSPolicy, TLSPolicy) continue to work when applied to Gateways with attached GRPCRoutes

TokenRateLimitPolicy is explicitly excluded as it requires protobuf response body parsing for token counting, which is beyond the scope of this proposal.

## Goals

1. Enable **data plane policies** (AuthPolicy, RateLimitPolicy) to target GRPCRoute resources via the standard `targetRef` field
2. Enable **extension policies** (OIDCPolicy, PlanPolicy, TelemetryPolicy) to target GRPCRoute resources via the standard `targetRef` field
3. Verify **infrastructure policies** (DNSPolicy, TLSPolicy) continue to work correctly when applied to Gateways with attached GRPCRoutes
4. Generate correct CEL predicates from GRPCRouteMatch (service/method matching)
5. Integrate GRPCRoutes into the existing topology graph
6. Provide example applications and documentation for gRPC use cases
7. Maintain full backward compatibility with existing HTTPRoute-based workflows

## Non-Goals

1. **TokenRateLimitPolicy support** - TokenRateLimitPolicy is explicitly excluded from GRPCRoute support. It requires response body parsing to extract token counts from LLM/AI responses. For gRPC, responses are protobuf-encoded rather than JSON, requiring WASM shim changes beyond this proposal's scope. Supporting TokenRateLimitPolicy on GRPCRoute would require:
   - Protobuf schema registration and parsing in the WASM shim
   - Response deserialization logic for arbitrary gRPC services
   - Token counting from protobuf message fields

   This is a significant effort that warrants its own RFC if there is future demand.

2. **Changes to backend services** - Authorino, Limitador, and WASM shim require no modifications
3. **New Well-Known Attributes** - RFC 0002 already supports HTTP/2; gRPC uses standard `request.url_path`
4. **gRPC-specific convenience attributes** - Optional future enhancement (`grpc.service`, `grpc.method`)
5. **APIProduct CRD** - Developer portal feature, separate domain from traffic policy

## Requirements

- **Minimum Gateway API version:** v1.1.0 (GRPCRoute reached GA in this release). The current dependency (`gateway-api v1.2.1`) already satisfies this.
- **sectionName targeting** requires Gateway API v1.2.0+ (GRPCRouteRule `name` field was added in v1.2.0). Already met by current dependency.
- The operator should log a warning if GRPCRoute CRD is not available at startup

## Design

### Backwards Compatibility

All changes are additive. Existing HTTPRoute-based policies continue to work unchanged. The core change is extending the topology to include GRPCRoute resources alongside HTTPRoutes.

### Architecture Changes

#### Key Insight: gRPC = HTTP/2

gRPC runs over HTTP/2, not alongside it. From Envoy's perspective, a gRPC request is an HTTP/2 request:

| Attribute | HTTP Request | gRPC Request |
|-----------|--------------|--------------|
| `request.method` | `GET`, `POST`, etc. | Always `POST` |
| `request.url_path` | `/api/users/123` | `/package.Service/Method` |
| `request.headers` | Standard headers | HTTP/2 headers + gRPC metadata |
| Protocol | HTTP/1.1 or HTTP/2 | HTTP/2 |

This means:
- **No changes needed to WASM shim** - CEL predicates against HTTP attributes work for gRPC
- **No changes needed to Authorino** - Receives AuthConfig CRDs with standard predicates
- **No changes needed to Limitador** - Counts requests by descriptors, protocol-agnostic

#### Data Flow

```
                            KUADRANT OPERATOR (changes here)
  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
  │ GRPCRoute   │    │ AuthPolicy/ │    │ CEL         │    │ AuthConfig/ │
  │ Match       │ +  │ RateLimit   │ →  │ Predicates  │ →  │ Limitador   │
  │             │    │ Policy      │    │             │    │ Limits      │
  │ service: X  │    │             │    │ url_path == │    │             │
  │ method: Y   │    │             │    │ '/X/Y'      │    │             │
  └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
                                                                 │
                                                                 ▼
                                         ┌───────────────────────────────┐
                                         │     ENVOY + WASM (no changes) │
                                         │  - Evaluates CEL predicates   │
                                         │  - Works for HTTP and gRPC    │
                                         └───────────────────────────────┘
                                                │                  │
                                                ▼                  ▼
                                         ┌─────────────┐    ┌─────────────┐
                                         │ AUTHORINO   │    │ LIMITADOR   │
                                         │ (no changes)│    │ (no changes)│
                                         └─────────────┘    └─────────────┘
```

#### Predicate Generation

GRPCRouteMatch translates to CEL predicates using standard HTTP attributes:

| GRPCRouteMatch | CEL Predicate |
|----------------|---------------|
| `service: "UserService", method: "GetUser"` | `request.url_path == '/UserService/GetUser'` |
| `service: "UserService"` | `request.url_path.startsWith('/UserService/')` |
| `headers: [{name: "x-tenant", value: "acme"}]` | `request.headers['x-tenant'] == 'acme'` |

### API Changes

#### Policy TargetRef Validation

All Kuadrant policies except TokenRateLimitPolicy will accept `GRPCRoute` as a valid `targetRef` kind. The policy structure is identical to HTTPRoute — only the `targetRef` changes.

**Policies supporting GRPCRoute:**
- **Data plane policies**: AuthPolicy, RateLimitPolicy
- **Extension policies**: OIDCPolicy, PlanPolicy, TelemetryPolicy
- **Infrastructure policies**: DNSPolicy, TLSPolicy (target Gateways, not routes directly)

**Policies excluded:**
- **TokenRateLimitPolicy**: Requires protobuf response body parsing (see Non-Goals)

**Example - RateLimitPolicy targeting HTTPRoute (existing):**
```yaml
apiVersion: kuadrant.io/v1
kind: RateLimitPolicy
metadata:
  name: toystore-ratelimit
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: toystore
  defaults:
    limits:
      global:
        rates:
          - limit: 10
            window: 10s
```

**For GRPCRoute, only the targetRef changes:**
```yaml
  targetRef:
    group: gateway.networking.k8s.io
    kind: GRPCRoute  # Only this changes
    name: grpcstore
```

The rest of the policy spec (rules, limits, authentication config, etc.) is identical. This pattern applies to all Kuadrant policies. Policies operate on the generated CEL predicates, not the route type directly.

#### Example GRPCRoute Configuration

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GRPCRoute
metadata:
  name: grpcstore
spec:
  parentRefs:
    - name: kuadrant-ingressgateway
      namespace: gateway-system
  hostnames: ["grpc.example.com"]
  rules:
    - matches:
        - method:
            service: "talker.TalkerService"
            method: "Echo"
      backendRefs:
        - name: grpcstore
          port: 9000
```

### Component Changes

#### Repositories Affected

| Component | Repository | Change Type |
|-----------|------------|-------------|
| policy-machinery | `kuadrant/policy-machinery` | Controller-layer wiring (machinery types/topology already exist) |
| kuadrant-operator | `kuadrant/kuadrant-operator` | Watchers, topology, predicates, reconcilers |
| testsuite | `kuadrant/testsuite` | GRPCRoute class, gRPC backend, E2E tests |

#### Affected Policies (kuadrant-operator)

All policies except TokenRateLimitPolicy will support GRPCRoute targets. Implementation requirements differ based on policy architecture:

| Policy Type | Policies | Implementation Changes |
|-------------|----------|----------------------|
| **Data Plane Policies** | AuthPolicy, RateLimitPolicy | targetRef validation + reconciler updates for GRPCRouteRule iteration + AuthConfig/Limitador generation |
| **Extension Policies** | OIDCPolicy, PlanPolicy, TelemetryPolicy | targetRef validation + event subscriptions (no reconciler logic changes needed) |
| **Infrastructure Policies** | DNSPolicy, TLSPolicy | No changes: target Gateways not routes, verification only |
| **Excluded Policies** | TokenRateLimitPolicy | Not supported: requires protobuf response body parsing (see Non-Goal #1) |

**Extension policies** use the same WASM shim and event-driven reconciliation as data plane policies, so GRPCRoute support works automatically once the operator infrastructure is in place. These policies only need targetRef validation and event subscription updates — no custom reconciliation logic changes are required.

#### Repositories NOT Affected

| Component | Why No Changes |
|-----------|----------------|
| Authorino / Authorino Operator | Receives AuthConfig CRDs - protocol agnostic |
| Limitador / Limitador Operator | Receives limit definitions - protocol agnostic |
| WASM Shim | Evaluates CEL against HTTP/2 attributes - works for gRPC |
| DNS Operator | Operates on Gateway listeners - route type irrelevant |
| RFC 0002 (Well-Known Attributes) | Already supports HTTP/2 |

#### Kuadrant Operator Changes

1. **Watcher Registration**: Register GRPCRoute watcher in state_of_the_world.go
2. **Topology Building**: Include GRPCRoutes via `machinery.WithGRPCRoutes()` and `machinery.ExpandGRPCRouteRules()`
3. **Path Extraction**: Abstract `ObjectsInRequestPath()` to handle both HTTP and gRPC routes
4. **Predicate Generation**: New `PredicatesFromGRPCRouteMatch()` function
5. **Policy Reconcilers**: Update effective policy reconcilers to iterate over GRPCRouteRules
6. **Extension Reconcilers**: Add GRPCRouteGroupKind subscriptions to Istio and Envoy Gateway reconcilers (WasmPlugin, EnvoyExtensionPolicy, auth/ratelimit cluster configs)
7. **Discoverability Reconcilers**: Define a `GRPCRoutePolicyDiscoverabilityReconciler` reconciler

### Design Decisions

#### Path Extraction Approach

The current `ObjectsInRequestPath()` returns hard-coded HTTPRoute types. Two approaches for supporting GRPCRoute:

1. **Unified struct** with `machinery.Targetable` interface fields and type-assertion helpers (`IsHTTPRoute()`, `AsGRPCRoute()`, etc.)
2. **Separate functions** (`ObjectsInHTTPRequestPath()`, `ObjectsInGRPCRequestPath()`) returning concrete types

Since policy reconcilers use separate iteration loops for HTTPRouteRules and GRPCRouteRules (see below), callers will already know the route type — making separate functions potentially cleaner. The final approach should be decided during implementation based on how many callers genuinely need route-type-agnostic access.

#### Reconciler Iteration Strategy

Effective policy reconcilers should use **separate iteration loops** for HTTPRouteRules and GRPCRouteRules, rather than combined iteration with type switching. This provides clearer separation and easier debugging.

#### Event Subscription Approach

Only `GRPCRouteGroupKind` should be added to event matchers — not `GRPCRouteRuleGroupKind`. This follows the existing HTTPRoute pattern where only route-level kinds are used in event subscriptions; rule-level kinds are only used in link functions and topology filtering.

Adding `GRPCRouteGroupKind` to the shared `dataPlaneEffectivePoliciesEventMatchers` list will also cause TokenRateLimitPolicy reconcilers to receive GRPCRoute events. These will be no-ops since TokenRateLimitPolicy does not support GRPCRoute targets (see Non-Goal #1).

#### GRPCRouteMatch Predicate Patterns

GRPCRouteMatch translates to CEL predicates using `request.url_path`:

| Pattern | CEL Predicate |
|---------|---------------|
| Exact service + exact method | `request.url_path == '/Service/Method'` |
| Exact service only | `request.url_path.startsWith('/Service/')` |
| Exact method only (no service) | `request.url_path.matches('^/[^/]+/Method$')` |
| Regex service + regex method | `request.url_path.matches('^/ServicePattern/MethodPattern$')` |
| Regex service only | `request.url_path.matches('^/ServicePattern/.*$')` |
| Regex method only | `request.url_path.matches('^/[^/]+/MethodPattern$')` |
| Mixed: exact service + regex method | `request.url_path.matches('^/Service/MethodPattern$')` |

Per Gateway API spec, "at least one of Service and Method MUST be a non-empty string". For Exact match type, values are expected to be valid gRPC identifiers (alphanumeric + dots), so regex escaping should not be needed in practice. For RegularExpression match type, values are user-provided regex patterns and should be used as-is.

An empty GRPCRouteMatch (no method, no headers) should return empty predicates, meaning "match all" — consistent with HTTPRoute behaviour.

#### GRPCRouteMatch Sorting Precedence

[Per Gateway API spec](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io/v1.GRPCRouteRule):

1. Largest number of characters in a matching non-wildcard hostname
2. Largest number of characters in a matching hostname
3. Largest number of characters in a matching service
4. Largest number of characters in a matching method
5. Largest number of header matches
6. Oldest Route based on creation timestamp (tie-breaker)
7. Alphabetical order by `{namespace}/{name}` (tie-breaker)

### Future: gRPC Convenience Attributes

The following attributes could be added as a future enhancement to improve UX for `when` clauses and `counters`:

- `request.grpc.service` — extracted from `request.url_path` (e.g., `UserService` from `/UserService/GetUser`)
- `request.grpc.method` — extracted from `request.url_path` (e.g., `GetUser` from `/UserService/GetUser`)

This would require WASM shim changes to detect gRPC requests (via `content-type: application/grpc`) and parse the URL path into components. The internal predicate generation should continue using `request.url_path` regardless, as using convenience attributes internally would create a hard dependency on WASM shim changes that must land before any GRPCRoute support can ship.

### Security Considerations

No additional security considerations. GRPCRoute support uses the same policy enforcement mechanisms as HTTPRoute. All authentication, authorization, and rate limiting policies apply identically — only the predicate generation differs.

## Implementation Plan

Work is organized into 9 tasks across 4 repositories. Each task delivers independently testable, working functionality and can be reviewed and merged separately while respecting dependencies.

Tasks are structured so that each policy-specific task includes full end-to-end functionality (predicate generation, reconciler updates, gateway provider wiring, and integration tests), ensuring that each completed task delivers working features rather than partial implementations.

## Testing Strategy

### Unit Tests

- Predicate generation for all GRPCRouteMatch types
- Path extraction with HTTP and gRPC routes
- Action set building for gRPC paths
- Sorting behavior for GRPCRouteMatchConfigs

### Integration Tests (kuadrant-operator)

**Data plane policies:**
- AuthPolicy targeting GRPCRoute
- RateLimitPolicy targeting GRPCRoute
- Real gRPC traffic with AuthPolicy enforcement
- Real gRPC traffic with RateLimitPolicy enforcement
- gRPC method-specific policies

**Extension policies:**
- OIDCPolicy targeting GRPCRoute
- PlanPolicy targeting GRPCRoute
- TelemetryPolicy targeting GRPCRoute

**Cross-policy scenarios:**
- Mixed HTTPRoute and GRPCRoute on same Gateway
- Policy inheritance (Gateway → GRPCRoute)
- DNSPolicy on Gateway with attached GRPCRoutes resolves hostnames correctly
- TLSPolicy on Gateway with attached GRPCRoutes provisions certificates correctly

**Providers:**
- Both Istio and Envoy Gateway providers

### E2E Tests (testsuite)

The `kuadrant/testsuite` repository provides the broader E2E test coverage following the existing HTTPRoute test patterns. This includes a `GRPCRoute` class implementing the `GatewayRoute` interface, a gRPC client wrapper, and test cases mirroring the existing HTTPRoute suite (auth enforcement, rate limiting, section targeting, deletion reconciliation, policy inheritance). See Task 10 for details.

## Execution

### Todo

- [ ] **Task 1: policy-machinery - GRPCRoute Controller Wiring**
  Controller-layer wiring for GRPCRoute resources. The machinery layer already has full support via [PR #16](https://github.com/Kuadrant/policy-machinery/pull/16).

- [ ] **Task 2: gRPC Backend Image for Testing & Examples**
  Select and validate a publicly available gRPC-capable image for use in integration tests, E2E tests, and examples.

- [ ] **Task 3: kuadrant-operator - Core GRPCRoute Infrastructure** (depends on Task 1)
  Register GRPCRoute watcher, update topology building, and abstract path extraction to support both HTTPRoute and GRPCRoute.

- [ ] **Task 4: kuadrant-operator - AuthPolicy Support + Data Plane Wiring** (depends on Tasks 2, 3)
  Enable AuthPolicy to target GRPCRoute with full end-to-end functionality including predicate generation, gateway provider reconcilers (WasmPlugin, auth cluster configs), and integration tests with real gRPC traffic.

- [ ] **Task 5: kuadrant-operator - RateLimitPolicy Support** (depends on Task 4)
  Enable RateLimitPolicy to target GRPCRoute including ratelimit cluster reconcilers and integration tests with real gRPC traffic.

- [ ] **Task 6: kuadrant-operator - Extension Policies Support** (depends on Task 4)
  Enable extension policies (OIDCPolicy, PlanPolicy, TelemetryPolicy) to target GRPCRoute with integration tests.

- [ ] **Task 7: kuadrant-operator - GRPCRoute Policy Discoverability Reconciler** (depends on Tasks 5, 6)
  Create GRPCRoutePolicyDiscoverabilityReconciler to add status conditions to GRPCRoutes showing which policies affect them.

- [ ] **Task 8: kuadrant-operator - Examples & Documentation** (depends on Tasks 2, 7)
  Create example manifests and user guide documentation for gRPC with all supported policies, including grpcurl verification commands.

- [ ] **Task 9: testsuite - GRPCRoute E2E Test Coverage** (depends on Tasks 2, 7)
  Add GRPCRoute support to the testsuite framework and E2E tests mirroring existing HTTPRoute test patterns.

### Completed

_(No tasks completed yet)_

---

## Alternatives Considered

### Alternative 1: GRPCRoute as HTTPRoute Adapter

Translate GRPCRoute to HTTPRoute internally instead of parallel code paths:

```go
func GRPCRouteMatchToHTTPRouteMatch(grpcMatch gwapiv1.GRPCRouteMatch) gwapiv1.HTTPRouteMatch {
    return gwapiv1.HTTPRouteMatch{
        Path: &gwapiv1.HTTPPathMatch{
            Type:  ptr(gwapiv1.PathMatchExact),
            Value: ptr("/" + grpcMatch.Method.Service + "/" + grpcMatch.Method.Method),
        },
        Method: ptr(gwapiv1.HTTPMethodPost), // gRPC is always POST
    }
}
```

| Pros | Cons |
|------|------|
| Reuses all existing HTTPRoute logic | Loses GRPCRoute semantics |
| Simpler maintenance - one code path | Harder to add gRPC-specific features later |
| | Obscures what's actually happening |

**Verdict:** Not recommended. The parallel approach is cleaner and more extensible.

### Alternative 2: Generic Route Interface

Create a common interface for both route types:

```go
type RouteWrapper interface {
    Hostnames() []string
    ParentRefs() []gwapiv1.ParentReference
    Rules() []RouteRuleWrapper
}
```

| Pros | Cons |
|------|------|
| Truly unified handling | Significant refactor of existing code |
| Extensible to TCPRoute, UDPRoute | Over-engineering for current needs |

**Verdict:** Not recommended for initial implementation. Could be considered as a future refactor if TCPRoute/UDPRoute support is needed.

---

## Appendix: Common Misconceptions

### "gRPC needs separate protocol handling"

**Incorrect.** gRPC is built on HTTP/2. Envoy sees gRPC requests as HTTP/2 requests with:
- `:method: POST`
- `:path: /package.Service/Method`
- `content-type: application/grpc`

All existing `request.*` Well-Known Attributes apply directly.

### "We need gRPC-specific Well-Known Attributes"

**Incorrect.** RFC 0002 explicitly supports HTTP/2. The same `request.url_path` attribute works for both:
- HTTP: `request.url_path == '/api/users'`
- gRPC: `request.url_path == '/UserService/GetUser'`

Optional future enhancement: `grpc.service` and `grpc.method` convenience attributes could parse the path for better UX, but are not required for GRPCRoute support.

### "Authorino/Limitador need changes for gRPC"

**Incorrect.** These components receive generated artifacts (AuthConfig CRDs, limit definitions) that are protocol-agnostic. The WASM filter evaluates CEL predicates against HTTP/2 attributes - this works identically for gRPC because gRPC IS HTTP/2.

---

## References

### Gateway API
- [GRPCRoute spec](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io/v1.GRPCRoute)
- [GRPCRouteRule spec](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io/v1.GRPCRouteRule) — sorting precedence rules
- [GRPCRouteMatch spec](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io/v1.GRPCRouteMatch)
- [GRPCMethodMatch spec](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io/v1.GRPCMethodMatch)
- [Policy Attachment (GEP-713)](https://gateway-api.sigs.k8s.io/geps/gep-713/)

### gRPC
- [gRPC over HTTP/2 protocol spec](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)

### Kuadrant
- [Well-Known Attributes (RFC 0002)](https://github.com/Kuadrant/architecture/blob/main/rfcs/0002-well-known-attributes.md)
- [policy-machinery PR #16 — GRPCRoute types](https://github.com/Kuadrant/policy-machinery/pull/16)

### Candidate gRPC Test Images
- [Istio echo](https://github.com/istio/istio/tree/master/pkg/test/echo) — multi-protocol server (HTTP, gRPC, TCP)
- [grpcbin](https://github.com/moul/grpcbin) — gRPC echo server with reflection
- [Fortio](https://fortio.org/) — load testing tool with built-in gRPC server
- [grpc-health-probe](https://github.com/grpc-ecosystem/grpc-health-probe)
