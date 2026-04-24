# Feature: GRPCRoute Support for Kuadrant Policies

- Feature Name: `grpcroute-support`
- Status: **Draft**
- Start Date: 2026-03-13
- Issue tracking: [Kuadrant/architecture#156](https://github.com/Kuadrant/architecture/issues/156)

## Summary

This design document outlines the implementation of GRPCRoute support in the Kuadrant ecosystem. GRPCRoute is a Gateway API resource (GA since v1.1.0) that provides gRPC-native routing semantics. This work enables Kuadrant policies to work with GRPCRoute resources:

- **Data plane policies** (AuthPolicy, RateLimitPolicy) can target GRPCRoute resources via the standard `targetRef` field
- **Extension policies** (PlanPolicy) can target GRPCRoute resources via the standard `targetRef` field
- **Infrastructure policies** (DNSPolicy, TLSPolicy, TelemetryPolicy) continue to work when applied to Gateways with attached GRPCRoutes

This proposal also adds gRPC well-known attributes (a `grpc` Optional binding with `service` and `method` fields) to improve user experience when writing policy conditions and rate limit counters for gRPC traffic.

TokenRateLimitPolicy is explicitly excluded as it requires protobuf response body parsing for token counting, which is beyond the scope of this proposal.

**Note on TelemetryPolicy:** TelemetryPolicy is categorized as an infrastructure policy because it currently only supports Gateway-level targeting (XValidation: `self.kind == 'Gateway'`), not route-level targeting. It works identically for Gateways with HTTPRoute or GRPCRoute attachments, requiring no GRPCRoute-specific changes.

## Goals

1. Enable **data plane policies** (AuthPolicy, RateLimitPolicy) to target GRPCRoute resources via the standard `targetRef` field
2. Enable **extension policies** (PlanPolicy) to target GRPCRoute resources via the standard `targetRef` field
3. Verify **infrastructure policies** (DNSPolicy, TLSPolicy, TelemetryPolicy) continue to work correctly when applied to Gateways with attached GRPCRoutes
4. Generate correct CEL predicates from GRPCRouteMatch (service/method matching)
5. Integrate GRPCRoutes into the existing topology graph
6. Provide example applications and documentation for gRPC use cases
7. Maintain full backward compatibility with existing HTTPRoute-based workflows
8. Add gRPC well-known attributes (a `grpc` Optional binding with `service` and `method` fields) for improved UX in `when` clauses and rate limit counters

## Non-Goals

1. **TokenRateLimitPolicy support** - TokenRateLimitPolicy is explicitly excluded from GRPCRoute support. It requires response body parsing to extract token counts from LLM/AI responses. For gRPC, responses are protobuf-encoded rather than JSON, requiring WASM shim changes beyond this proposal's scope. Supporting TokenRateLimitPolicy on GRPCRoute would require:
   - Protobuf schema registration and parsing in the WASM shim
   - Response deserialization logic for arbitrary gRPC services
   - Token counting from protobuf message fields

   This is a significant effort that warrants its own RFC if there is future demand.

2. **OIDCPolicy support** - OIDCPolicy is explicitly excluded from GRPCRoute support. The OAuth2 authorization code flow implemented by OIDCPolicy is designed for browser-based clients and relies on HTTP 302 redirects and cookie storage. These behaviors are unverified with gRPC-Web clients and incompatible with native gRPC clients. Specific concerns:
   - **Redirect handling**: gRPC-Web libraries may not follow HTTP 302 redirects or may treat them as protocol errors
   - **Cookie support**: gRPC-Web implementations may not store/send cookies required for session management
   - **Content-Type mismatch**: gRPC-Web expects `application/grpc-web` responses but receives standard HTTP redirect headers
   - **Native gRPC incompatibility**: Native gRPC clients (non-browser) cannot participate in browser-based OAuth flows

   Users needing OIDC authentication for gRPC services should use AuthPolicy directly with JWT validation, which already supports GRPCRoute targets. OIDCPolicy support could be reconsidered in the future if there is validated demand from gRPC-Web use cases and after testing confirms browser redirect/cookie behavior works correctly.

3. **Changes to backend services** - Authorino and Limitador require no modifications for GRPCRoute support (WASM shim requires changes for gRPC well-known attributes - see Goal #8)
4. **APIProduct CRD** - Developer portal feature, separate domain from traffic policy

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
| `request.protocol` | `HTTP/1.1` or `HTTP/2` | `HTTP/2` |
| `grpc.hasValue()` | `false` | `true` |
| `grpc.service` | Error (optional.none()) | `UserService` (from `/UserService/GetUser`) |
| `grpc.method` | Error (optional.none()) | `GetUser` (from `/UserService/GetUser`) |

This means:
- **No changes needed to WASM shim for basic GRPCRoute support** - CEL predicates against HTTP attributes work for gRPC (WASM shim changes are needed only for gRPC well-known attributes - see Task 10)
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

Route-targeting policies (AuthPolicy, RateLimitPolicy, PlanPolicy) will accept `GRPCRoute` as a valid `targetRef` kind. The policy structure is identical to HTTPRoute — only the `targetRef` changes.

**Policies that can directly target GRPCRoute:**
- **Data plane policies**: AuthPolicy, RateLimitPolicy
- **Extension policies**: PlanPolicy

**Policies that work with GRPCRoute indirectly (via Gateway targeting):**
- **Infrastructure policies**: DNSPolicy, TLSPolicy, TelemetryPolicy
  - These policies target Gateways, not routes
  - They work identically whether the Gateway has HTTPRoute or GRPCRoute attachments
  - No GRPCRoute-specific changes needed (verification only)

**Policies excluded:**
- **TokenRateLimitPolicy**: Requires protobuf response body parsing (see Non-Goal #1)
- **OIDCPolicy**: OAuth2 flow incompatible with gRPC clients (see Non-Goal #2)

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

#### CRD Validation Updates

Each policy's CRD requires an XValidation marker update to accept GRPCRoute as a valid targetRef kind. These updates are implemented as part of each policy's GRPCRoute support task:

**AuthPolicy, RateLimitPolicy, PlanPolicy** (Tasks 4, 5, 6):
```go
// Update from:
// +kubebuilder:validation:XValidation:rule="self.kind == 'HTTPRoute' || self.kind == 'Gateway'"

// To:
// +kubebuilder:validation:XValidation:rule="self.kind == 'HTTPRoute' || self.kind == 'GRPCRoute' || self.kind == 'Gateway'"
```

**Policies excluded (TokenRateLimitPolicy, OIDCPolicy):** No XValidation changes needed. These policies' existing validation rules only allow HTTPRoute and Gateway, so GRPCRoute targetRef will be rejected automatically.

After updating markers, run `make generate manifests` to regenerate CRDs.

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
| kuadrant-operator | `kuadrant/kuadrant-operator` | CRD validation, RBAC, watchers, topology, predicates, reconcilers |
| wasm-shim | `kuadrant/wasm-shim` | gRPC request detection, path parsing, new well-known attributes for CEL evaluation |
| authorino | `kuadrant/authorino` | Extract `grpc.service` and `grpc.method` well-known attributes for authorization evaluators |
| architecture (RFC 0002) | `kuadrant/architecture` | Add `grpc.service` and `grpc.method` to Well-Known Attributes specification |
| kuadrant-console-plugin | `kuadrant/kuadrant-console-plugin` | UI updates: resource registry, topology visualization, console tabs, GRPCRoute policies page |
| testsuite | `kuadrant/testsuite` | GRPCRoute class, gRPC backend, E2E tests |

#### Affected Policies (kuadrant-operator)

Route-targeting policies (AuthPolicy, RateLimitPolicy, PlanPolicy) require implementation changes to support GRPCRoute. Infrastructure policies work without changes. Implementation requirements differ based on policy architecture:

| Policy Type | Policies | Implementation Changes |
|-------------|----------|----------------------|
| **Data Plane Policies** | AuthPolicy, RateLimitPolicy | targetRef validation + reconciler updates for GRPCRouteRule iteration + AuthConfig/Limitador generation |
| **Extension Policies** | PlanPolicy | targetRef validation + event subscriptions (no reconciler logic changes needed) |
| **Infrastructure Policies** | DNSPolicy, TLSPolicy, TelemetryPolicy | No changes: target Gateways not routes, verification only |
| **Excluded Policies** | TokenRateLimitPolicy, OIDCPolicy | Not supported: TokenRateLimitPolicy requires protobuf response parsing (see Non-Goal #1), OIDCPolicy incompatible with gRPC clients (see Non-Goal #2) |

**Extension policies** use the same WASM shim and event-driven reconciliation as data plane policies, so GRPCRoute support works automatically once the operator infrastructure is in place. These policies only need targetRef validation and event subscription updates — no custom reconciliation logic changes are required.

**Infrastructure policies** are Gateway-only and work identically regardless of attached route types (HTTPRoute or GRPCRoute). TelemetryPolicy is categorized here because its XValidation restricts targeting to Gateway-only (`self.kind == 'Gateway'`).

#### RBAC Updates Required

**Main Operator ClusterRole** (`config/rbac/role.yaml`):

The operator requires permissions to watch and access GRPCRoute resources:

```yaml
- apiGroups: ["gateway.networking.k8s.io"]
  resources: ["grpcroutes", "grpcroutes/status"]
  verbs: ["get", "list", "watch"]
```

**Extension Policy ClusterRoles** (`cmd/extensions/*/config/rbac/role.yaml`):

Extension policies (PlanPolicy) require GRPCRoute permissions for their reconcilers to watch and react to GRPCRoute changes:

```yaml
- apiGroups: ["gateway.networking.k8s.io"]
  resources: ["grpcroutes"]
  verbs: ["get", "list", "watch"]
```

Without these RBAC permissions, watchers and reconcilers will fail with permission denied errors at runtime, making the feature non-functional.

#### Repositories NOT Affected

| Component | Why No Changes |
|-----------|----------------|
| Authorino Operator | Manages AuthConfig CRDs - no protocol-specific logic |
| Limitador / Limitador Operator | Receives limit definitions - protocol agnostic |
| DNS Operator | Operates on Gateway listeners - route type irrelevant |

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

The current `ObjectsInRequestPath()` returns hard-coded HTTPRoute types. GRPCRoute support uses **separate functions** returning concrete types:

- `ObjectsInHTTPRequestPath()` — returns `[]machinery.HTTPRoute`, `[]machinery.HTTPRouteRule`, etc.
- `ObjectsInGRPCRequestPath()` — returns `[]machinery.GRPCRoute`, `[]machinery.GRPCRouteRule`, etc.

This approach is chosen because policy reconcilers use separate iteration loops for HTTPRouteRules and GRPCRouteRules (see below), so callers already know the route type at call time. Separate functions are simpler to implement and debug than a unified struct with type assertion helpers.

**Alternative considered:** Unified struct with `machinery.Targetable` interface fields and type-assertion helpers (`IsHTTPRoute()`, `AsGRPCRoute()`). Rejected as over-engineering for current needs — no callers require route-type-agnostic access.

#### Reconciler Iteration Strategy

Effective policy reconcilers should use **separate iteration loops** for HTTPRouteRules and GRPCRouteRules, rather than combined iteration with type switching. This provides clearer separation and easier debugging.

#### Event Subscription Approach

Only `GRPCRouteGroupKind` should be added to event matchers — not `GRPCRouteRuleGroupKind`. This follows the existing HTTPRoute pattern where only route-level kinds are used in event subscriptions; rule-level kinds are only used in link functions and topology filtering.

Adding `GRPCRouteGroupKind` to the shared `dataPlaneEffectivePoliciesEventMatchers` list will also cause TokenRateLimitPolicy reconcilers to receive GRPCRoute events. These will be no-ops since TokenRateLimitPolicy does not support GRPCRoute targets (see Non-Goal #1).

#### Well-Known Attributes Behavior with gRPC

**Decision:** Well-Known Attributes remain protocol-agnostic and reflect HTTP/2 reality regardless of route type.

For gRPC requests, `request.method` will always be `POST` because that is the underlying HTTP/2 method that gRPC uses. To match on the gRPC method itself (e.g., "Echo", "GetUser"), users must use `request.url_path` predicates that match the `/Service/Method` pattern.

**Rationale:** Well-Known Attributes are provided by Envoy via the Proxy-Wasm API and represent what Envoy sees on the wire (HTTP/2). The route type (HTTPRoute vs GRPCRoute) is a Gateway API concept for routing configuration, not a data plane protocol distinction. Mixing these concerns would create inconsistent semantics where the same attribute has different meanings based on the Kubernetes resource type rather than the actual protocol.

**Alternative considered:** Making `request.method` context-aware, returning the gRPC method name (parsed from the path) when used with GRPCRoute. This was rejected because:
1. It would require WASM shim changes to detect gRPC requests (via `content-type: application/grpc`) and parse paths
2. It creates inconsistent attribute semantics where `request.method` means "HTTP method" for HTTPRoute but "gRPC method name" for GRPCRoute
3. It violates the principle that Well-Known Attributes reflect the wire protocol, not the Kubernetes resource configuration

**gRPC-specific well-known attributes:** The attributes `grpc.service` and `grpc.method` will be implemented as separate well-known attributes without breaking existing behavior or requiring changes to `request.method` semantics. This approach preserves protocol-accurate WKAs while providing gRPC-specific attributes for policy authors.

#### Hostname Conflict Resolution Between Route Types

**Decision:** HTTPRoute and GRPCRoute resources with overlapping hostnames cannot be attached to the same Gateway Listener. Gateway API implementations must reject subsequent routes during reconciliation by setting their `Accepted` condition to `False` in the route's status.

**Behaviour per Gateway API specification:**
- When routes of different types (HTTPRoute and GRPCRoute) are attached to the same Listener with matching or overlapping hostnames, the Gateway implementation enforces hostname uniqueness
- The first route successfully attached (based on controller reconciliation order) is accepted
- Subsequent conflicting routes receive `Accepted=False` status with a reason explaining the conflict
- This prevents ambiguous routing where the same hostname could resolve to both HTTP and gRPC backends

**Recommended practice:**
- Use **separate hostnames** for HTTP and gRPC traffic (e.g., `api.example.com` for HTTPRoute, `grpc.example.com` for GRPCRoute)
- This approach provides clear routing semantics and avoids rejection scenarios

**Workaround for shared hostname:**
- If the same hostname must serve both HTTP and gRPC traffic, use HTTPRoute for both types of traffic
- HTTPRoute can match gRPC requests using path patterns (`/Service/Method`) and the `content-type: application/grpc` header
- This loses the UX benefits of GRPCRoute's native service/method matching but allows hostname sharing

**Kuadrant policy implications:**
- Policies attached to **rejected routes** (those with `Accepted=False`) will not be enforced
- The topology graph will include rejected routes, but their policy attachments are inactive
- Users should monitor route status conditions to identify conflicts
- Consider adding validation or warnings in the console plugin when detecting hostname overlaps between route types

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

### gRPC Well-Known Attributes

The following well-known attributes improve UX for `when` clauses and rate limit counters:

- `grpc` — CEL Optional binding always present in the evaluation context; equals `optional.none()` for non-gRPC (HTTP) requests and contains gRPC request metadata for gRPC requests
  - `grpc.service` — extracted from `request.url_path` (e.g., `UserService` from `/UserService/GetUser`)
  - `grpc.method` — extracted from `request.url_path` (e.g., `GetUser` from `/UserService/GetUser`)

**Implementation approach:**
- WASM shim detects gRPC requests via `content-type: application/grpc` header
- Path parsing extracts service and method components from `/Service/Method` format
- `grpc` binding is implemented as a CEL Optional type
- For gRPC requests: `grpc` contains `optional.of(metadata)` where metadata includes `service` and `method` fields
- For non-gRPC requests: `grpc` contains `optional.none()`
- Internal predicate generation continues using `request.url_path` to avoid coupling

**Note:** These attributes are **optional** for policy authors. All existing predicates using `request.url_path` continue to work. These well-known attributes provide convenience for policy conditions like:
```yaml
when:
  - predicate: grpc.hasValue() && grpc.value().service == 'UserService'
  - predicate: grpc.hasValue() && grpc.value().method == 'GetUser'
```

#### CEL Optional Semantics

The `grpc` binding uses CEL's Optional type to provide type-safe access to gRPC metadata:

**Checking for gRPC requests:**
```yaml
grpc.hasValue()  # Returns true for gRPC requests, false for HTTP
```

**Safe attribute access:**
```yaml
# Correct: Check hasValue() before accessing fields using .value()
when:
  - predicate: grpc.hasValue() && grpc.value().service == 'UserService'

# Also valid: Combine with boolean operators
when:
  - predicate: grpc.hasValue() && grpc.value().service == 'UserService' && grpc.value().method == 'GetUser'

# For negation: Always check hasValue() first
when:
  - predicate: grpc.hasValue() && grpc.value().service != 'AdminService'
```

**Benefits of Optional type:**
- **Type safety**: Accessing `grpc.value().service` without checking `hasValue()` is a CEL error
- **No empty string bugs**: Cannot accidentally match HTTP traffic with negation logic
- **Clear semantics**: `hasValue()` explicitly indicates whether the request is gRPC
- **Consistent with CEL idioms**: Optional types are a standard CEL pattern for nullable values ([CEL Optional Types](https://github.com/google/cel-go/blob/v0.23.2/cel/library.go#L360-L425))
- **Fail-closed security**: Missing `grpc.hasValue()` checks cause CEL evaluation errors at request time, preventing accidental policy bypasses. Unlike empty-string approaches where negation logic (e.g., `grpc.value().service != 'AdminService'`) could silently match all HTTP traffic, the Optional type ensures policies fail explicitly rather than allowing unintended access.

**Best practice for all policies:** Always include `grpc.hasValue()` checks even when targeting a specific GRPCRoute. While only gRPC traffic flows through GRPCRoute resources, including the check provides defensive coding and prevents errors if policies are later copied to Gateway targets or if routing behavior changes. Consistent use of `hasValue()` makes policy intent explicit and improves maintainability.

### Security Considerations

**Policy Enforcement:**

GRPCRoute support uses the same policy enforcement mechanisms as HTTPRoute. No new attack surface is introduced:
- AuthConfig CRDs for authentication/authorization (Authorino)
- Limitador limit definitions for rate limiting
- WASM filter CEL evaluation for predicate matching

gRPC requests are processed as HTTP/2 requests using existing well-known attributes (`request.url_path`, `request.method`, `request.headers`). All policies enforce identically regardless of route type.

**Implementation Correctness:**

Incomplete implementation could cause silent policy bypass. All components documented in the "Component Changes" section (CRD validation, RBAC permissions, link functions, reconcilers) must be implemented together. Missing any component could result in policies appearing to be applied but not enforcing correctly.

## Implementation Plan

Work is organized into 12 tasks across 7 repositories. Each task delivers independently testable, working functionality and can be reviewed and merged separately while respecting dependencies.

Tasks are structured so that each policy-specific task includes full end-to-end functionality (predicate generation, reconciler updates, gateway provider wiring, and integration tests), ensuring that each completed task delivers working features rather than partial implementations. Tasks 10 and 11 (gRPC well-known attributes in WASM shim and Authorino) can be developed in parallel with operator work (Tasks 3-7) since these implementations are independent and the operator continues using `request.url_path` patterns internally.

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
- PlanPolicy targeting GRPCRoute

**Cross-policy scenarios:**
- Mixed HTTPRoute and GRPCRoute on same Gateway
- Policy inheritance (Gateway → GRPCRoute)
- DNSPolicy on Gateway with attached GRPCRoutes resolves hostnames correctly
- TLSPolicy on Gateway with attached GRPCRoutes provisions certificates correctly
- TelemetryPolicy on Gateway with attached GRPCRoutes works correctly

**Providers:**
- Both Istio and Envoy Gateway providers

### E2E Tests (testsuite)

The `kuadrant/testsuite` repository provides the broader E2E test coverage following the existing HTTPRoute test patterns. This includes a `GRPCRoute` class implementing the `GatewayRoute` interface, a gRPC client wrapper, and test cases mirroring the existing HTTPRoute suite (auth enforcement, rate limiting, section targeting, deletion reconciliation, policy inheritance). See Task 9 for details.

## Execution

### Todo

- [ ] **Task 5: kuadrant-operator - RateLimitPolicy Support** (depends on Task 4)
  Enable RateLimitPolicy to target GRPCRoute including ratelimit cluster reconcilers and integration tests with real gRPC traffic. Update RateLimitPolicy CRD XValidation to accept GRPCRoute targetRef.

- [ ] **Task 6: kuadrant-operator - Extension Policies Support** (depends on Task 4)
  Enable PlanPolicy to target GRPCRoute with integration tests. Update PlanPolicy CRD XValidation to accept GRPCRoute targetRef. Add ClusterRole permissions for grpcroutes in PlanPolicy's RBAC.

- [ ] **Task 7: kuadrant-operator - GRPCRoute Policy Discoverability Reconciler** (depends on Tasks 5, 6)
  Create GRPCRoutePolicyDiscoverabilityReconciler to add status conditions to GRPCRoutes showing which policies affect them.

- [ ] **Task 8: kuadrant-operator - Examples & Documentation** (depends on Tasks 2, 7, optionally 10)
  Create example manifests and user guide documentation for gRPC with all supported policies, including grpcurl verification commands. Examples should demonstrate both `request.url_path` patterns and gRPC well-known attributes (if Task 10 is complete).

- [ ] **Task 9: testsuite - GRPCRoute E2E Test Coverage** (depends on Tasks 2, 7, optionally 10)
  Add GRPCRoute support to the testsuite framework and E2E tests mirroring existing HTTPRoute test patterns. Test coverage for gRPC well-known attributes is optional but recommended.

- [ ] **Task 10: wasm-shim & RFC 0002 - gRPC Well-Known Attributes** (no blockers - can be developed in parallel)
  Implement `grpc` as a CEL Optional binding containing `service` and `method` fields in the WASM shim for improved UX in policy conditions and rate limit counters. For gRPC requests, `grpc` contains `optional.of(metadata)` where metadata includes service and method; for HTTP requests, `grpc` contains `optional.none()`. Update RFC 0002 specification with gRPC well-known attributes documentation.

- [ ] **Task 11: authorino - gRPC Well-Known Attributes** (no blockers - can be developed in parallel)
  Extract `grpc.service` and `grpc.method` from gRPC requests and add to authorization JSON for use in OPA policies, CEL authorization rules, and other evaluators. For gRPC requests, include a `grpc` object with `service` and `method` fields. For non-gRPC requests, the `grpc` field must be absent (not present in the JSON) to align with OPA's undefined value semantics when checking field existence (e.g., `input.grpc.service` returns undefined for HTTP requests, allowing Rego policies to distinguish between HTTP and gRPC). Note: OPA/Rego does not have a `has()` built-in function; field presence is checked via undefined value semantics or using `object.get(input, "grpc", {})` with defaults.

- [ ] **Task 12: kuadrant-console-plugin - GRPCRoute UI Support** (depends on Task 7)
  Add GRPCRoute support to the OpenShift Console plugin: update resource registry, add GRPCRoute to topology visualization with icon and context menu, add "Policies" tab to GRPCRoute detail pages, create GRPCRoutePoliciesPage component, update RESOURCE_POLICY_MAP to show which policies can target GRPCRoute (excluding OIDCPolicy and TokenRateLimitPolicy).

### Completed

- [x] **Task 1: policy-machinery - GRPCRoute Controller Wiring** — [Kuadrant/policy-machinery#65](https://github.com/Kuadrant/policy-machinery/issues/65)
  Controller-layer wiring for GRPCRoute resources. The machinery layer already has full support via [PR #16](https://github.com/Kuadrant/policy-machinery/pull/16).

- [x] **Task 2: gRPC Backend Image for Testing & Examples** — [Kuadrant/kuadrant-operator#1823](https://github.com/Kuadrant/kuadrant-operator/issues/1823)
  Select and validate a publicly available gRPC-capable image for use in integration tests, E2E tests, and examples.

- [x] **Task 3: kuadrant-operator - Core GRPCRoute Infrastructure** — [Kuadrant/kuadrant-operator#1820](https://github.com/Kuadrant/kuadrant-operator/issues/1820)
  Register GRPCRoute watcher, update topology building, and abstract path extraction to support both HTTPRoute and GRPCRoute. Add ClusterRole permissions for grpcroutes and grpcroutes/status.

- [x] **Task 4: kuadrant-operator - AuthPolicy Support + Data Plane Wiring** — [Kuadrant/kuadrant-operator#1821](https://github.com/Kuadrant/kuadrant-operator/issues/1821)
  Enable AuthPolicy to target GRPCRoute with full end-to-end functionality including predicate generation, gateway provider reconcilers (WasmPlugin, auth cluster configs), and integration tests with real gRPC traffic. Update AuthPolicy CRD XValidation to accept GRPCRoute targetRef.

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

## Appendix: Frequently Asked Questions

### "Does gRPC need separate protocol handling?"

**No, gRPC uses HTTP/2.** From Envoy's perspective, gRPC requests are HTTP/2 requests with specific characteristics:
- `:method: POST` (gRPC always uses POST)
- `:path: /package.Service/Method` (contains service and method)
- `content-type: application/grpc` (identifies the request as gRPC)

All existing `request.*` Well-Known Attributes apply directly to gRPC traffic. This is why the WASM shim, Authorino, and Limitador require no changes — they already process HTTP/2 attributes correctly.

### "Do we need gRPC-specific Well-Known Attributes?"

**Not required for basic GRPCRoute support, but included for improved UX.** RFC 0002 explicitly supports HTTP/2, and the same `request.*` attributes work for both HTTP and gRPC:
- HTTP: `request.url_path == '/api/users'`
- gRPC: `request.url_path == '/UserService/GetUser'`

Note that `request.method` will always be `POST` for gRPC requests, as gRPC uses HTTP/2 POST for all calls. To match on the gRPC method itself, use `request.url_path` which contains the full `/Service/Method` path.

**Additional well-known attributes:** This proposal adds `grpc.service` and `grpc.method` attributes to provide better UX when writing policy conditions and rate limit counters, but these are optional—all GRPCRoute support works using `request.url_path`.

### "Do Authorino and Limitador need changes for gRPC?"

**No changes required.** These components receive generated artifacts (AuthConfig CRDs, limit definitions) that are protocol-agnostic. The WASM filter evaluates CEL predicates against HTTP/2 attributes, which works identically for gRPC because gRPC runs over HTTP/2.

From Authorino's perspective, a gRPC request looks like an HTTP/2 POST with specific path and header patterns. From Limitador's perspective, rate limit descriptors are built from CEL expressions regardless of the underlying protocol. Both components process gRPC traffic correctly without any code changes.

### "What happens to gRPC well-known attributes for HTTP requests?"

**The `grpc` binding is an Optional with no value.** When policies are applied to Gateways with mixed HTTP and gRPC traffic:

- For gRPC requests: `grpc.hasValue()` returns `true`, attributes are accessible
- For HTTP requests: `grpc.hasValue()` returns `false`, accessing attributes is a CEL error

**Type safety prevents bugs:** Unlike empty string approaches, you cannot accidentally match HTTP traffic:
```yaml
# This is safe - only matches gRPC traffic
when:
  - predicate: grpc.hasValue() && grpc.value().service != 'AdminService'

# This would be a CEL evaluation error on HTTP traffic (caught at request time)
when:
  - predicate: grpc.value().service != 'AdminService'  # Missing hasValue() check!
```

**Best practice for mixed traffic:** Always use `grpc.hasValue()` to check for gRPC requests first:
```yaml
when:
  - predicate: grpc.hasValue() && grpc.value().service == 'UserService'
```

**For route-level policies:** If your policy targets a specific GRPCRoute (not a Gateway), you can omit the `hasValue()` check since only gRPC traffic flows through GRPCRoutes, but including it is recommended for clarity.

---

## References

### Gateway API
- [GRPCRoute spec](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io/v1.GRPCRoute)
- [GRPCRouteRule spec](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io/v1.GRPCRouteRule) — sorting precedence rules
- [GRPCRouteMatch spec](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io/v1.GRPCRouteMatch)
- [GRPCMethodMatch spec](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io/v1.GRPCMethodMatch)
- [Policy Attachment (GEP-713)](https://gateway-api.sigs.k8s.io/geps/gep-713/)
- [GEP-1016: GRPCRoute Enhancement](https://gateway-api.sigs.k8s.io/geps/gep-1016/) — hostname conflict resolution between route types
- [Gateway API Hostname Concepts](https://gateway-api.sigs.k8s.io/concepts/api-overview/#hostname-matching) — hostname matching and uniqueness requirements

### gRPC
- [gRPC over HTTP/2 protocol spec](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)

### Kuadrant
- [Well-Known Attributes (RFC 0002)](https://github.com/Kuadrant/architecture/blob/main/rfcs/0002-well-known-attributes.md)
- [policy-machinery PR #16 — GRPCRoute types](https://github.com/Kuadrant/policy-machinery/pull/16)
- [wasm-shim repository](https://github.com/Kuadrant/wasm-shim) — Proxy-Wasm module for Envoy integration

### CEL & Proxy-Wasm
- [Common Expression Language (CEL) specification](https://github.com/google/cel-spec)
- [CEL-go Optional Types Library](https://github.com/google/cel-go/blob/v0.23.2/cel/library.go#L360-L425) — `hasValue()`, `value()`, `optional.of()`, `optional.none()` documentation
- [CEL-go Optional Type Implementation](https://github.com/google/cel-go/blob/v0.23.2/common/types/optional.go) — Go implementation of Optional wrapper type
- [CEL-go Variable Declaration](https://github.com/google/cel-go/blob/v0.23.2/cel/decls.go#L144) — `cel.Variable()` function for declaring variables in CEL environment
- [CEL-go Activation Context](https://github.com/google/cel-go/blob/v0.23.2/interpreter/activation.go#L22-L31) — mechanism for supplying input values to CEL programs
- [Proxy-Wasm specification](https://github.com/proxy-wasm/spec) — WebAssembly for Proxies (ABI specification)

### OPA & Authorino
- [Authorino OPA Authorization](https://docs.kuadrant.io/latest/authorino/docs/features/#open-policy-agent-opa-rego-policies-authorizationopa) — using OPA policies in AuthPolicy
- [OPA Policy Reference](https://www.openpolicyagent.org/docs/latest/policy-reference/#built-in-functions) — Rego built-in functions (note: `has()` is not included)
- [OPA Understanding Undefined](https://www.openpolicyagent.org/docs/latest/policy-language/#understanding-undefined) — how Rego handles undefined values and field presence checking

### Candidate gRPC Test Images
- [Istio echo](https://github.com/istio/istio/tree/master/pkg/test/echo) — multi-protocol server (HTTP, gRPC, TCP)
- [grpcbin](https://github.com/moul/grpcbin) — gRPC echo server with reflection
- [Fortio](https://fortio.org/) — load testing tool with built-in gRPC server
- [grpc-health-probe](https://github.com/grpc-ecosystem/grpc-health-probe)
