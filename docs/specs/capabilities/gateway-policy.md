# Gateway Policy Framework

## Components

| Component | Repo | Language | Role |
|-----------|------|----------|------|
| policy-machinery | policy-machinery | Go | Shared library: topology DAG, policy attachment, merge strategies, controller framework |
| kuadrant-operator | kuadrant-operator | Go | Consumes policy-machinery to build topology, compute effective policies, run reconciliation workflows |

## What policy-machinery provides

policy-machinery implements the [Gateway API Policy Attachment](https://gateway-api.sigs.k8s.io/geps/gep-713/) (GEP-713) pattern as a reusable library. It provides:

1. **Topology** — a DAG of Gateway API resources with policies attached
2. **Policy merging** — defaults/overrides with pluggable merge strategies
3. **Effective policy computation** — walk topology paths to produce merged policies per route rule
4. **Controller framework** — event-driven reconciliation with subscriptions and workflows

## Core Interfaces

All defined in `machinery/types.go`:

```go
// Base interface for all topology objects
type Object interface {
    schema.ObjectKind
    GetNamespace() string
    GetName() string
    GetLocator() string    // unique ID in topology, e.g. "Gateway:default/my-gw"
}

// Resources that can have policies attached (GEP-713)
type Targetable interface {
    Object
    SetPolicies([]Policy)
    Policies() []Policy
}

// Policy objects with merge logic
type Policy interface {
    Object
    GetTargetRefs() []PolicyTargetReference
    GetMergeStrategy() MergeStrategy
    Merge(Policy) Policy
}

// Function type for custom merge logic
type MergeStrategy func(source, target Policy) Policy
```

## Topology DAG

### Structure

The topology is a directed acyclic graph where nodes are Kubernetes objects and edges represent relationships:

```
Kuadrant
    |
GatewayClass
    |
Gateway ──────────────────── [Policies: DNSPolicy, TLSPolicy]
    |
Listener ─────────────────── [Policies: DNSPolicy (sectionName), TLSPolicy (sectionName)]
    |
HTTPRoute ────────────────── [Policies: AuthPolicy, RateLimitPolicy]
    |
HTTPRouteRule ────────────── [Policies: AuthPolicy (sectionName), RateLimitPolicy (sectionName)]
    |
Service / ServicePort
```

Side links to generated resources:
```
Listener ──→ DNSRecord, Certificate
HTTPRouteRule ──→ AuthConfig
Gateway ──→ WasmPlugin (Istio) / EnvoyExtensionPolicy (EG)
Kuadrant ──→ Limitador, Authorino
```

### Built-in Gateway API types

Wrapper types in `machinery/gateway_api_types.go` implement `Targetable`:

| Type | Wraps | Locator format | Parent field |
|------|-------|----------------|-------------|
| `GatewayClass` | `*gwapiv1.GatewayClass` | `GatewayClass:name` | — |
| `Gateway` | `*gwapiv1.Gateway` | `Gateway:ns/name` | — |
| `Listener` | `*gwapiv1.Listener` | `Gateway:ns/name#listener` | `Gateway` |
| `HTTPRoute` | `*gwapiv1.HTTPRoute` | `HTTPRoute:ns/name` | — |
| `HTTPRouteRule` | `*gwapiv1.HTTPRouteRule` | `HTTPRoute:ns/name#rule-idx` | `HTTPRoute` |
| `GRPCRoute` | `*gwapiv1.GRPCRoute` | `GRPCRoute:ns/name` | — |
| `Service` | `*core.Service` | `Service:ns/name` | — |
| `ServicePort` | `*core.ServicePort` | `Service:ns/name#port` | `Service` |

### LinkFunc — defining edges

Each edge in the DAG is defined by a `LinkFunc`:

```go
type LinkFunc struct {
    From schema.GroupKind    // parent kind
    To   schema.GroupKind    // child kind
    Func func(child Object) []Object   // given child, return its parents
}
```

**Built-in links** (in `machinery/gateway_api_topology.go`):
- `LinkGatewayClassToGatewayFunc` — Gateway references GatewayClass by `spec.gatewayClassName`
- `LinkGatewayToListenerFunc` — Gateway contains Listeners (expansion)
- `LinkListenerToHTTPRouteFunc` — HTTPRoute `spec.parentRefs` with sectionName
- `LinkGatewayToHTTPRouteFunc` — HTTPRoute `spec.parentRefs` (no listener expansion)
- `LinkHTTPRouteToHTTPRouteRuleFunc` — HTTPRoute contains rules (expansion)
- `LinkHTTPRouteRuleToServiceFunc` — BackendRefs in rules
- Similar for GRPCRoute, TCPRoute, TLSRoute, UDPRoute

### Policy attachment

When the topology is built, policies are attached to targetables by matching locators:

```go
// In NewTopology():
policiesByTargetRef[targetRef.GetLocator()] = append(..., policy)

// Then:
targetable.SetPolicies(policiesByTargetRef[targetable.GetLocator()])
```

A policy's `GetTargetRefs()` returns locators like `Gateway:default/my-gw` or `Gateway:default/my-gw#https` (with sectionName).

### Topology options

```go
type GatewayAPITopologyOptions struct {
    GatewayClasses    []*GatewayClass
    Gateways          []*Gateway
    HTTPRoutes        []*HTTPRoute
    Services          []*Service
    Policies          []Policy
    Objects           []Object
    Links             []LinkFunc

    // Expansion flags — create sub-nodes for sections
    ExpandGatewayListeners  bool   // Gateway → Listener nodes
    ExpandHTTPRouteRules    bool   // HTTPRoute → HTTPRouteRule nodes
    ExpandServicePorts      bool   // Service → ServicePort nodes
}
```

kuadrant-operator enables all expansion flags to get the most granular topology.

### DAG validation

The topology builder validates the graph is acyclic using Kahn's algorithm (topological sort). If a cycle is detected, topology creation fails (unless `AllowLoops` is set).

## Effective Policy Computation

### Algorithm

`EffectivePolicyForPath[T Policy](path []Targetable, predicate func(Policy) bool) *T`:

1. **Gather** — collect all policies of type `T` from every targetable in the path, filtered by predicate
2. **Sort** — order from least specific (GatewayClass) to most specific (HTTPRouteRule)
3. **Merge** — `lo.ReduceRight()` from most-specific to least-specific, calling `policy.Merge(other)` at each step
4. **Return** — the single merged effective policy

### Path computation

Paths are computed via depth-first search:

```go
// All paths from any GatewayClass to a specific HTTPRouteRule
paths := topology.Targetables().Paths(gatewayClass, httpRouteRule)
```

Each path is a slice: `[GatewayClass, Gateway, Listener, HTTPRoute, HTTPRouteRule]`

### How kuadrant-operator uses it

For each policy type, a dedicated reconciler computes effective policies:

```go
// EffectiveAuthPolicyReconciler
for _, gatewayClass := range gatewayClasses {
    for _, httpRouteRule := range httpRouteRules {
        paths := targetables.Paths(gatewayClass, httpRouteRule)
        for i := range paths {
            effectivePolicy := EffectivePolicyForPath[*AuthPolicy](
                paths[i],
                isAuthPolicyAcceptedAndNotDeletedFunc(state),
            )
            if effectivePolicy != nil {
                effectivePolicies[pathID] = EffectiveAuthPolicy{
                    Path:           paths[i],
                    Spec:           *effectivePolicy,
                    SourcePolicies: SourcePoliciesFromEffectivePolicy(*effectivePolicy),
                }
            }
        }
    }
}
state.Store(StateEffectiveAuthPolicies, effectivePolicies)
```

Effective policy reconcilers exist for:
- `EffectiveAuthPolicyReconciler` → `StateEffectiveAuthPolicies`
- `EffectiveRateLimitPolicyReconciler` → `StateEffectiveRateLimitPolicies`
- `EffectiveTokenRateLimitPolicyReconciler` → `StateEffectiveTokenRateLimitPolicies`
- `EffectiveDNSPoliciesReconciler` (uses listener-level, not route-rule-level)
- `EffectiveTLSPoliciesReconciler` (uses listener-level)

### Merge strategies (implemented in kuadrant-operator)

| Strategy | Defaults | Overrides |
|----------|----------|-----------|
| `atomic` | Target wins if non-empty, else source | Source always wins |
| `merge` | Target rules + source rules not in target | Source rules + target rules not in source |

Each rule carries a `Source` field (`kind/namespace/name`) for tracing which policy contributed it.

## Controller Framework

### Architecture

```go
type Controller struct {
    client     *dynamic.DynamicClient
    manager    ctrlruntime.Manager
    cache      *CacheStore           // watchable object store
    topology   *gatewayAPITopologyBuilder
    runnables  map[string]Runnable   // resource watchers
    reconcile  ReconcileFunc         // main reconciliation function
}
```

### Event flow

```
Resource watcher detects change (via SharedInformer)
         |
         v
CacheStore updated (watchable.Map)
         |
         v
Controller detects diff (old vs new snapshot)
  - CreateEvent: UID not in old state
  - UpdateEvent: UID in both, objects differ
  - DeleteEvent: UID in old, not in new
         |
         v
Topology rebuilt from current cache
         |
         v
ReconcileFunc called with (events, topology, state)
         |
         v
Subscription matchers filter events per reconciler
         |
         v
Matching reconcilers execute
```

### Subscription

Each reconciler declares which events it cares about:

```go
type Subscription struct {
    ReconcileFunc ReconcileFunc
    Events        []ResourceEventMatcher
}

type ResourceEventMatcher struct {
    Kind            *schema.GroupKind   // filter by resource kind
    EventType       *EventType         // Create, Update, Delete (nil = any)
    ObjectNamespace string             // filter by namespace (empty = any)
    ObjectName      string             // filter by name (empty = any)
}
```

### Workflow

Workflows compose reconcilers into phases:

```go
type Workflow struct {
    Precondition  ReconcileFunc   // runs first (validation)
    Tasks         []ReconcileFunc // run concurrently
    Postcondition ReconcileFunc   // runs last (status updates)
}
```

### ReconcileFunc signature

```go
type ReconcileFunc func(
    ctx context.Context,
    resourceEvents []ResourceEvent,
    topology *machinery.Topology,
    err error,
    state *sync.Map,           // shared state between reconcilers
) error
```

## kuadrant-operator Topology Wiring

### Object and policy registration

In `state_of_the_world.go`:

```go
// Policy kinds (attached to targetables in topology)
controller.WithPolicyKinds(
    DNSPolicyGroupKind,
    TLSPolicyGroupKind,
    AuthPolicyGroupKind,
    RateLimitPolicyGroupKind,
    TokenRateLimitPolicyGroupKind,
)

// Non-policy, non-targetable objects tracked in topology
controller.WithObjectKinds(
    KuadrantGroupKind,
    ConfigMapGroupKind,
    DeploymentGroupKind,
)
```

### Custom links (kuadrant-operator specific)

| Link | From | To | Purpose |
|------|------|----|---------|
| `LinkKuadrantToGatewayClasses` | Kuadrant | GatewayClass | Root of topology |
| `LinkKuadrantToLimitador` | Kuadrant | Limitador | Component dependency |
| `LinkKuadrantToAuthorino` | Kuadrant | Authorino | Component dependency |
| `LinkLimitadorToDeployment` | Limitador | Deployment | Readiness tracking |
| `LinkListenerToDNSRecord` | Listener | DNSRecord | DNS capability |
| `LinkDNSPolicyToDNSRecord` | DNSPolicy | DNSRecord | Ownership |
| `LinkListenerToCertificate` | Listener | Certificate | TLS capability |
| `LinkTLSPolicyToIssuer` | TLSPolicy | Issuer | Issuer validation |
| `LinkTLSPolicyToClusterIssuer` | TLSPolicy | ClusterIssuer | Issuer validation |
| `LinkHTTPRouteRuleToAuthConfig` | HTTPRouteRule | AuthConfig | Auth capability |
| `LinkGatewayToWasmPlugin` | Gateway | WasmPlugin | Istio WASM injection |
| `LinkGatewayToEnvoyFilter` | Gateway | EnvoyFilter | Istio cluster config |
| `LinkGatewayToEnvoyPatchPolicy` | Gateway | EnvoyPatchPolicy | EG cluster config |
| `LinkGatewayToEnvoyExtensionPolicy` | Gateway | EnvoyExtensionPolicy | EG WASM injection |
| `LinkKuadrantToPeerAuthentication` | Kuadrant | PeerAuthentication | Istio mTLS |
| `LinkKuadrantToServiceMonitor` | Kuadrant | ServiceMonitor | Observability |
| `LinkKuadrantToPodMonitor` | Kuadrant | PodMonitor | Observability |

### Conditional registration

Links and watchers are registered conditionally based on installed CRDs:

```go
// state_of_the_world.go — BootOptionsBuilder methods
getGatewayAPIOptions()         // always required
getIstioOptions()              // if Istio CRDs present
getEnvoyGatewayOptions()       // if EnvoyGateway CRDs present
getCertManagerOptions()        // if cert-manager CRDs present
getDNSOperatorOptions()        // if dns-operator CRDs present
getLimitadorOperatorOptions()  // if limitador-operator CRDs present
getAuthorinoOperatorOptions()  // if authorino CRDs present
```

This allows the single operator binary to adapt to different cluster configurations.

## Main Reconciliation Workflow

```
Event arrives
    |
    v
[Init Workflow] (Precondition)
├── EventLogger
└── TopologyReconciler (writes topology to ConfigMap)
    |
    v
[Parallel Tasks]
├── DNS Workflow
│   ├── Precondition: DNSPoliciesValidator
│   ├── Task: EffectiveDNSPoliciesReconciler
│   └── Postcondition: DNSPolicyStatusUpdater
│
├── TLS Workflow
│   ├── Precondition: TLSPoliciesValidator
│   ├── Task: EffectiveTLSPoliciesReconciler
│   └── Postcondition: TLSPolicyStatusUpdater
│
├── Data Plane Workflow
│   ├── Precondition: Validators (Auth, RateLimit, TokenRateLimit)
│   ├── Tasks:
│   │   ├── Precondition: Effective policy computation (Auth, RL, TokenRL)
│   │   ├── AuthConfigsReconciler
│   │   ├── LimitadorLimitsReconciler
│   │   ├── Provider-specific cluster + extension reconcilers
│   │   └── Istio integration reconcilers
│   └── Postcondition: Status updaters (Auth, RL, TokenRL)
│
├── Observability Workflow
├── Developer Portal Workflow
└── Console Plugin Workflow
    |
    v
[Finalize Workflow] (Postcondition)
├── KuadrantStatusUpdater
├── PolicyMetricsReconciler
├── GatewayPolicyDiscoverabilityReconciler
└── ExtensionReconcilers
```

## State Passing Between Reconcilers

Reconcilers communicate via `state *sync.Map`:

| Key | Type | Producer | Consumer |
|-----|------|----------|----------|
| `StateEffectiveAuthPolicies` | `map[string]EffectiveAuthPolicy` | EffectiveAuthPolicyReconciler | AuthConfigsReconciler, ExtensionReconcilers |
| `StateEffectiveRateLimitPolicies` | `map[string]EffectiveRateLimitPolicy` | EffectiveRateLimitPolicyReconciler | LimitadorLimitsReconciler, ExtensionReconcilers |
| `StateEffectiveTokenRateLimitPolicies` | `map[string]EffectiveTokenRateLimitPolicy` | EffectiveTokenRateLimitPolicyReconciler | LimitadorLimitsReconciler, ExtensionReconcilers |
| `StateDNSPolicyAcceptedKey` | `map[string]error` | DNSPoliciesValidator | EffectiveDNSPoliciesReconciler |
| `StateTLSPolicyAcceptedKey` | `map[string]error` | TLSPoliciesValidator | EffectiveTLSPoliciesReconciler |
| `StateAuthPolicyValid` | `map[string]error` | AuthPolicyValidator | EffectiveAuthPolicyReconciler |
| `StateRateLimitPolicyValid` | `map[string]error` | RateLimitPolicyValidator | EffectiveRateLimitPolicyReconciler |
| `StateLimitadorLimitsModified` | `bool` | LimitadorLimitsReconciler | (status reporting) |

## Key Source Files

### policy-machinery
- Core interfaces: [`machinery/types.go`](https://github.com/Kuadrant/policy-machinery/blob/main/machinery/types.go)
- Topology builder: [`machinery/topology.go`](https://github.com/Kuadrant/policy-machinery/blob/main/machinery/topology.go)
- Gateway API topology: [`machinery/gateway_api_topology.go`](https://github.com/Kuadrant/policy-machinery/blob/main/machinery/gateway_api_topology.go)
- Gateway API types: [`machinery/gateway_api_types.go`](https://github.com/Kuadrant/policy-machinery/blob/main/machinery/gateway_api_types.go)
- Core K8s types: [`machinery/core_types.go`](https://github.com/Kuadrant/policy-machinery/blob/main/machinery/core_types.go)
- Controller: [`controller/controller.go`](https://github.com/Kuadrant/policy-machinery/blob/main/controller/controller.go)
- Workflow: [`controller/workflow.go`](https://github.com/Kuadrant/policy-machinery/blob/main/controller/workflow.go)
- Subscription: [`controller/subscriber.go`](https://github.com/Kuadrant/policy-machinery/blob/main/controller/subscriber.go)
- Events: [`controller/events.go`](https://github.com/Kuadrant/policy-machinery/blob/main/controller/events.go)
- Cache: [`controller/cache.go`](https://github.com/Kuadrant/policy-machinery/blob/main/controller/cache.go)
- Topology builder (controller): [`controller/topology_builder.go`](https://github.com/Kuadrant/policy-machinery/blob/main/controller/topology_builder.go)
- Runnable/watcher: [`controller/runnable.go`](https://github.com/Kuadrant/policy-machinery/blob/main/controller/runnable.go)

### kuadrant-operator
- State of the world (wiring): [`internal/controller/state_of_the_world.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/internal/controller/state_of_the_world.go)
- Merge strategies: [`api/v1/merge_strategies.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/api/v1/merge_strategies.go)
- Topology links (Kuadrant root): [`api/v1beta1/topology.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/api/v1beta1/topology.go)
- Data plane workflow: [`internal/controller/data_plane_policies_workflow.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/internal/controller/data_plane_policies_workflow.go)
- DNS workflow: [`internal/controller/dns_workflow.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/internal/controller/dns_workflow.go)
- TLS workflow: [`internal/controller/tls_workflow.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/internal/controller/tls_workflow.go)
- Effective auth policies: [`internal/controller/effective_auth_policies_reconciler.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/internal/controller/effective_auth_policies_reconciler.go)
- Effective RL policies: [`internal/controller/effective_ratelimit_policies_reconciler.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/internal/controller/effective_ratelimit_policies_reconciler.go)
- Effective token RL policies: [`internal/controller/effective_tokenratelimit_policies_reconciler.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/internal/controller/effective_tokenratelimit_policies_reconciler.go)
- Effective DNS policies: [`internal/controller/effective_dnspolicies_reconciler.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/internal/controller/effective_dnspolicies_reconciler.go)
- Effective TLS policies: [`internal/controller/effective_tls_policies_reconciler.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/internal/controller/effective_tls_policies_reconciler.go)
- Istio utils (links): [`internal/istio/utils.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/internal/istio/utils.go)
- EnvoyGateway utils (links): [`internal/envoygateway/utils.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/internal/envoygateway/utils.go)
- Authorino utils (links): [`internal/authorino/utils.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/internal/authorino/utils.go)
