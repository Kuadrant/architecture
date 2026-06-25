# Kuadrant Data Plane Multi-Tenancy 

- Feature Name: `multi_tenancy`
- Start Date: 2026-06-24
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/issues/0000)

# Summary
[summary]: #summary

This RFC introduces data plane multi-tenancy to Kuadrant by allowing multiple Kuadrant custom resources to coexist in a single cluster, each managing a distinct set of Gateways with isolated data plane components.

The **control plane remains a singleton**: It is only the Authorino and Limitador instances that enforce auth and rate-limiting policy at the gateway level that change. Today, one Kuadrant resource implicitly manages every Gateway in the cluster through a single shared Authorino and Limitador deployment. This proposal replaces that model with scoped instances, where each Kuadrant resource:

- **Selects its Gateways via a label selector** (`spec.gatewaySelector`), giving platform operators explicit control over which Gateways belong to which Kuadrant instance.
- **Triggers deployment of its own Authorino and Limitador** (data plane components) in its namespace, providing full isolation of auth and rate-limiting backends between tenants.
- **Uses Gateway API ReferenceGrants** for cross-namespace Gateway management, following the established Gateway API pattern for authorizing cross-namespace references.
- **Enforces one Kuadrant instance per namespace**, keeping the component naming convention simple (each namespace has at most one `limitador` and one `authorino` resource).
- **Prevents Gateway overlap** — a Gateway may only have by a single Kuadrant data plane instance; conflicts are detected and reported via status conditions.

Cluster-wide concerns — DNS, TLS certificate provisioning, CRD management, and operator lifecycle — are unaffected. The kuadrant-operator, DNS operator, and cert-manager continue to operate as shared cluster services.

**Backwards compatibility:** when `gatewaySelector` is omitted, the Kuadrant instance defaults to managing all Gateways cluster-wide — preserving current behavior. A deprecation warning is emitted, and users should migrate to explicit selectors before a future release changes the default to "select nothing."

# Motivation
[motivation]: #motivation

Today, the Kuadrant operator enforces singleton install: if multiple Kuadrant CRs exist, the oldest one wins and the rest are silently ignored. Every Gateway in the cluster is managed by that single instance, sharing a single Authorino and a single Limitador.

- **Platform teams sharing a cluster** — different teams own different Gateways and want independent data plane stacks.
- **Vendor/tenant isolation** — a managed platform provisions a Kuadrant instance per customer tenant, each with its own Authorino and Limitador, scoped to their Gateway(s).

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Selecting Gateways

The Kuadrant CRD gains a new `spec.gatewaySelector` field — a standard Kubernetes label selector. Only Gateways whose labels match the selector are managed by that Kuadrant instance.

```yaml
apiVersion: kuadrant.io/v1beta1
kind: Kuadrant
metadata:
  name: team-a
  namespace: kuadrant-team-a
spec:
  gatewaySelector:
    matchLabels:
      kuadrant.io/managed-by: team-a
  observability:
    enable: true
```

A Gateway in another namespace opts in by carrying the matching label:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: api-gateway
  namespace: team-a-gateways
  labels:
    kuadrant.io/managed-by: team-a
spec:
  gatewayClassName: istio
  listeners:
    - name: http
      port: 80
      protocol: HTTP
```

### Key rules

1. **A Gateway may only be managed by one Kuadrant instance.** The controller performs continuous overlap detection. When a conflict is detected (e.g., two Kuadrant CRs select the same Gateway, or a Gateway label changes to match a second Kuadrant), the **existing/older Kuadrant instance retains ownership** and the newer one gets `Available: False` with reason `GatewayConflict`. The conflicting Gateway is not reconciled by the newer instance until the overlap is resolved.
2. **One Kuadrant instance per namespace.** A namespace may contain at most one Kuadrant CR. This ensures unambiguous ownership of the Limitador and Authorino resources in that namespace (both use fixed names `limitador` and `authorino`). This is enforced at admission time by a ValidatingAdmissionPolicy (VAP).
3. **Backwards compatibility: no selector = all Gateways.** When `gatewaySelector` is omitted, the Kuadrant instance manages all Gateways cluster-wide — preserving current singleton behavior. A `DeprecatedDefaultSelector` status condition and controller log warning are emitted to prompt migration. A future release will change this default to "select nothing." Note: a Kuadrant CR with no selector will conflict at runtime with any other Kuadrant CR, since it claims all Gateways.
4. **Policies attached to a Gateway are reconciled by the Kuadrant instance that manages that Gateway.** A policy targeting an unmanaged Gateway is not reconciled and receives an `Accepted: False` condition.

### Security model: Gateway labels as the tenant boundary

Label selectors follow the standard Kubernetes ownership pattern (Deployments→Pods, Services→Endpoints, etc.). In all of these, RBAC on the selected resource's labels is the security boundary — any principal who can modify labels can change set membership. Kuadrant inherits this property.

Because relabeling a Gateway changes which tenant's Authorino and Limitador handle its traffic, the blast radius is higher than typical label-based selection. **Operators should restrict `update` and `patch` on Gateway metadata to trusted principals.** Standard Kubernetes RBAC (e.g., a Role that grants `get`/`list` but not `update` on Gateways) is sufficient to prevent unauthorized reassignment.

If label-based reassignment proves to be an operational concern, the `targetRef` alternative (see [Rationale and alternatives](#why-label-selectors-over-targetref)) provides a stricter model where ownership is declared in the Kuadrant CR spec, requiring both Kuadrant CR write access *and* a ReferenceGrant for two-sided consent.

### Cross-namespace references via ReferenceGrant

When a Kuadrant CR has an **explicit `gatewaySelector`** and manages Gateways in a different namespace, a Gateway API `ReferenceGrant` must exist in the Gateway's namespace to authorize the cross-namespace reference:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-kuadrant-team-a
  namespace: team-a-gateways
spec:
  from:
    - group: kuadrant.io
      kind: Kuadrant
      namespace: kuadrant-team-a
  to:
    - group: gateway.networking.k8s.io
      kind: Gateway
```

Without this grant, the Kuadrant instance cannot manage Gateways in that namespace, even if the labels match. The operator sets `Available: False` with reason `ReferenceGrantRequired` on the Kuadrant status.

**ReferenceGrant is NOT required when:**
- The Kuadrant CR and Gateway are in the **same namespace** — co-tenancy is sufficient authorization (consistent with Gateway API conventions for same-namespace references).
- The Kuadrant CR has **no `gatewaySelector`** (legacy/deprecated mode) — cross-namespace management works without ReferenceGrants, preserving current singleton behavior. This ensures existing deployments where the Kuadrant CR in `kuadrant-system` manages Gateways cluster-wide continue to work without creating ReferenceGrants in every namespace.

### Per-instance data plane components

Each Kuadrant CR provisions its own **data plane components** — Limitador and Authorino — in its own namespace. These are the components that enforce policy at request time and need isolation between tenants. The current hardcoded name convention (`limitador`, `authorino`) is retained but scoped per namespace. The one-Kuadrant-per-namespace constraint ensures these names are unambiguous:

```
namespace: kuadrant-team-a
  ├── Kuadrant/team-a
  ├── Limitador/limitador        # data plane: rate limiting for team-a's Gateways
  └── Authorino/authorino        # data plane: auth for team-a's Gateways

namespace: kuadrant-team-b
  ├── Kuadrant/team-b
  ├── Limitador/limitador        # data plane: rate limiting for team-b's Gateways
  └── Authorino/authorino        # data plane: auth for team-b's Gateways
```

The Authorino Operator, Limitador Operator, DNS operator, and cert-manager are deployed as they are today — no changes required. They continue to instantiate data plane components (Authorino/Limitador pods) in whatever namespace CRs appear. It is the **kuadrant-operator** that changes to allow multiple Kuadrant CRs, each creating Authorino/Limitador CRs in its own namespace as it does now.

### Status reporting

Each Kuadrant instance reports status via the existing `Available` condition with new reasons (see [Status conditions](#status-conditions) for the full table). A new `status.managedGateways` field lists the Gateways currently managed by this instance.

### Example: two teams, one cluster

```
Cluster
│
├── namespace: kuadrant-system                          ── SHARED CONTROL PLANE ──
│   └── Deployment/kuadrant-operator-controller-manager  (single instance, reconciles all Kuadrant CRs)
│
├── namespace: kuadrant-team-a                          ── TEAM-A DATA PLANE ──
│   ├── Kuadrant/team-a  (gatewaySelector: kuadrant.io/managed-by=team-a)
│   ├── Limitador/limitador
│   └── Authorino/authorino
│
├── namespace: kuadrant-team-b                          ── TEAM-B DATA PLANE ──
│   ├── Kuadrant/team-b  (gatewaySelector: kuadrant.io/managed-by=team-b)
│   ├── Limitador/limitador
│   └── Authorino/authorino
│
├── namespace: team-a-gateways
│   ├── Gateway/api-gw     (labels: kuadrant.io/managed-by=team-a)
│   ├── ReferenceGrant/allow-kuadrant-team-a
│   ├── HTTPRoute/api-routes
│   ├── AuthPolicy/api-auth        → enforced via team-a's Authorino
│   └── RateLimitPolicy/api-rl     → enforced via team-a's Limitador
│
└── namespace: team-b-gateways
    ├── Gateway/internal-gw  (labels: kuadrant.io/managed-by=team-b)
    ├── ReferenceGrant/allow-kuadrant-team-b
    ├── HTTPRoute/internal-routes
    └── AuthPolicy/internal-auth   → enforced via team-b's Authorino
```

### Migration from singleton model

Existing users have a single Kuadrant CR with no `gatewaySelector`. The migration path:

1. **Phase 1 (this release):** An omitted `gatewaySelector` defaults to "select all Gateways" — identical to current behavior. The operator sets a `DeprecatedDefaultSelector` status condition and logs a warning: `"gatewaySelector is not set; defaulting to all Gateways. This default will change in a future release. Set an explicit gatewaySelector to suppress this warning."` Existing deployments continue to work unchanged.
2. **Phase 2 (future release):** The default changes to "select nothing." Users who have not added a `gatewaySelector` will see their Kuadrant instance stop managing Gateways. The operator logs an error and sets `Available: False` with a clear message.

For users who want to keep the current behavior permanently, they can add an empty `matchLabels: {}` selector (matches all) or a broad label selector. The deprecation only affects the *implicit* default.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## API changes

### KuadrantSpec

```go
type KuadrantSpec struct {
    // GatewaySelector selects which Gateways this Kuadrant instance manages.
    // Only Gateways whose labels match the selector are included in this
    // instance's topology. A Gateway may only be managed by one Kuadrant instance.
    //
    // When omitted, defaults to selecting all Gateways (backwards compatible).
    // This default is deprecated and will change to "select nothing" in a future release.
    // Set an explicit selector to opt out of the deprecated default.
    //
    // An empty selector (matchLabels: {}) explicitly selects all Gateways.
    // +optional
    GatewaySelector *metav1.LabelSelector `json:"gatewaySelector,omitempty"`

    Observability Observability `json:"observability,omitempty"`
    MTLS          *MTLS         `json:"mtls,omitempty"`
    Components    *Components   `json:"components,omitempty"`
}
```

### KuadrantStatus

```go
type KuadrantStatus struct {
    ObservedGeneration int64              `json:"observedGeneration,omitempty"`
    Conditions         []metav1.Condition `json:"conditions,omitempty" ...`
    MtlsAuthorino      *bool             `json:"mtlsAuthorino,omitempty"`
    MtlsLimitador      *bool             `json:"mtlsLimitador,omitempty"`

    // ManagedGateways lists the Gateways currently managed by this Kuadrant instance.
    // +optional
    ManagedGateways []GatewayReference `json:"managedGateways,omitempty"`
}

type GatewayReference struct {
    Namespace string `json:"namespace"`
    Name      string `json:"name"`
}
```

### Status conditions

The existing `Available` condition is used with new **reasons** to signal multi-tenancy issues, consistent with how mcp-gateway handles ReferenceGrant status:

| Condition | Status | Reason | Meaning |
|-----------|--------|--------|---------|
| `Available` | `False` | `GatewayConflict` | This Kuadrant instance's selector overlaps with an older instance that retains ownership of the contested Gateway(s). |
| `Available` | `False` | `ReferenceGrantRequired` | A matching cross-namespace Gateway exists but no valid ReferenceGrant authorizes the reference. |
| `Available` | `True` | `DeprecatedDefaultSelector` | No `gatewaySelector` specified; defaulting to "select all" (migration period only). |

### ValidatingAdmissionPolicy (one Kuadrant per namespace)

A Kubernetes ValidatingAdmissionPolicy (VAP) enforces the one-per-namespace constraint at admission time. VAPs are declarative, require no webhook infrastructure, and are evaluated by the API server directly:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: kuadrant-one-per-namespace
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: ["kuadrant.io"]
        apiVersions: ["v1beta1"]
        operations: ["CREATE"]
        resources: ["kuadrants"]
  paramKind:
    apiVersion: kuadrant.io/v1beta1
    kind: Kuadrant
  validations:
    - expression: "oldObject != null || request.namespace != params.metadata.namespace || request.name == params.metadata.name"
      messageExpression: "'only one Kuadrant resource is allowed per namespace ' + request.namespace"
      reason: Forbidden
```

Gateway overlap detection is **not** enforced at admission time — VAP CEL expressions cannot query arbitrary cluster resources (Gateways, other Kuadrant CRs across namespaces). Gateway conflicts are detected and reported at runtime via the `GatewayConflict` status reason (see [Runtime Gateway overlap detection](#runtime-gateway-overlap-detection)).

## Core change: per-Gateway data plane wiring

The central problem this RFC solves is: **when generating the Envoy configuration for a Gateway, which Authorino and Limitador should it point to?**

### How data plane wiring works today (singleton)

The operator configures each Gateway's Envoy proxy in two layers:

1. **Wasm shim installation** — an EnvoyFilter (Istio) or EnvoyExtensionPolicy (Envoy Gateway) installs the kuadrant wasm shim on the Gateway. The wasm config references Envoy cluster names, not hostnames:
   - `kuadrant-auth-service` — for auth checks
   - `kuadrant-ratelimit-service` — for rate limit checks

2. **Cluster definitions** — separate EnvoyFilter/EnvoyPatchPolicy resources define those Envoy clusters with the actual backend endpoints:
   - Auth cluster → `authorino-authorino-authorization.{kuadrant-ns}.svc.cluster.local:50051`
   - Rate limit cluster → derived from `limitador.Status.Service.Host`

Today, both the cluster names and the backend hostnames are **global constants**. The auth/rate-limit cluster reconcilers (`istio_auth_cluster_reconciler.go`, `istio_ratelimit_cluster_reconciler.go`, and their Envoy Gateway equivalents) call `GetKuadrantFromTopology()` to find the single Kuadrant CR, walk the topology to its Authorino/Limitador children, and derive the hostname from there. Every Gateway gets identical cluster definitions pointing to the same backends.

### How data plane wiring changes (multi-tenant)

With multiple Kuadrant instances, the cluster definitions for each Gateway must point to the **Authorino and Limitador in the namespace of the Kuadrant instance that owns that Gateway**:

```
Gateway api-gw (team-a):
  kuadrant-auth-service    → authorino-authorino-authorization.kuadrant-team-a.svc:50051
  kuadrant-ratelimit-service → limitador-limitador.kuadrant-team-a.svc:8081

Gateway internal-gw (team-b):
  kuadrant-auth-service    → authorino-authorino-authorization.kuadrant-team-b.svc:50051
  kuadrant-ratelimit-service → limitador-limitador.kuadrant-team-b.svc:8081
```

The Envoy cluster **names** (`kuadrant-auth-service`, `kuadrant-ratelimit-service`) can remain the same per-Gateway because each Gateway has its own EnvoyFilter/ExtensionPolicy scope. What changes is the **endpoint address** inside each cluster definition.

### What enables this: topology-aware Kuadrant resolution

The cluster reconcilers currently call `GetKuadrantFromTopology()` which returns the single oldest Kuadrant CR. This must change to a **Gateway-scoped lookup**: given a Gateway, walk the topology upward to find its owning Kuadrant, then walk downward from that Kuadrant to its Authorino/Limitador children.

The reconciler flow changes from:

```
// BEFORE (singleton)
kuadrant := GetKuadrantFromTopology(topology, state)         // global singleton
authorino := topology.Children(kuadrant, AuthorinoGroupKind)  // single Authorino
host := authorinoServiceInfoFromAuthorino(authorino)          // one hostname for all Gateways
// apply to ALL Gateways
```

to:

```
// AFTER (multi-tenant)
for _, gateway := range gateways {
    kuadrant := GetKuadrantForGateway(topology, gateway)          // Gateway's owner
    if kuadrant == nil { continue }                                // unmanaged Gateway
    authorino := topology.Children(kuadrant, AuthorinoGroupKind)   // this instance's Authorino
    host := authorinoServiceInfoFromAuthorino(authorino)            // per-instance hostname
    // apply to THIS Gateway only
}
```

This pattern applies to the following reconcilers (grouped by concern):

**Cluster definitions** (Envoy upstream endpoints to Authorino/Limitador — the core multi-tenancy change):
- `istio_auth_cluster_reconciler.go`
- `istio_ratelimit_cluster_reconciler.go`
- `envoy_gateway_auth_cluster_reconciler.go`
- `envoy_gateway_ratelimit_cluster_reconciler.go`

**Wasm config builders** (install wasm shim on Gateways):
- `istio_extension_reconciler.go`
- `envoy_gateway_extension_reconciler.go`

**Auth/rate-limit config generation** (must target the correct instance's namespace):
- `authconfigs_reconciler.go` — AuthConfig resources must be created in the namespace of the Kuadrant instance that owns the target Gateway, so the correct Authorino instance picks them up.
- `limitador_reconciler.go` — rate limit configs must target the Limitador in the owning Kuadrant's namespace.

**Tracing cluster definitions** (tracing endpoints may differ per instance):
- `istio_tracing_cluster_reconciler.go`
- `envoy_gateway_tracing_cluster_reconciler.go`

**Gateway provider integration** (mTLS, PeerAuthentication, Authorino integration):
- `authorino_istio_integration_reconciler.go`
- `limitador_istio_integration_reconciler.go`
- `istio_peerauthentication_reconciler.go`

**Effective policy reconcilers and helpers** (resolve which policies apply per Gateway):
- `effective_auth_policies_reconciler.go`
- `effective_ratelimit_policies_reconciler.go`
- `effective_tokenratelimit_policies_reconciler.go`
- `auth_workflow_helpers.go`
- `ratelimit_workflow_helpers.go`

**Status updaters** (must report status against the correct Kuadrant instance):
- `auth_policy_status_updater.go`
- `ratelimit_policy_status_updater.go`
- `tokenratelimitpolicy_status_updater.go`
- `kuadrant_status_updater.go`

**Other:**
- `observability_reconciler.go`
- `developerportal_reconciler.go`
- `authorino_reconciler.go`

In total, 25 call sites across 23 files reference `GetKuadrantFromTopology`.

## Topology changes

All the changes above are enabled by restructuring the topology DAG so that each Gateway has a specific Kuadrant parent.

### `LinkKuadrantToGatewayClasses` → `LinkKuadrantToGateways`

The current topology links Kuadrant to GatewayClasses unconditionally. This changes to a direct Kuadrant-to-Gateway link that applies the label selector:

```go
func LinkKuadrantToGateways(objs controller.Store) machinery.LinkFunc {
    kuadrants := lo.Map(objs.FilterByGroupKind(KuadrantGroupKind), controller.ObjectAs[*Kuadrant])
    referenceGrants := objs.FilterByGroupKind(ReferenceGrantGroupKind)

    return machinery.LinkFunc{
        From: KuadrantGroupKind,
        To:   schema.GroupKind{Group: gatewayapiv1.GroupVersion.Group, Kind: "Gateway"},
        Func: func(child machinery.Object) []machinery.Object {
            gateway := child
            var parents []machinery.Object
            for _, k := range kuadrants {
                if !selectorMatchesGateway(k.Spec.GatewaySelector, gateway) {
                    continue
                }
                // ReferenceGrant only required when gatewaySelector is explicitly set.
                // Legacy mode (nil selector) skips this check for backwards compatibility.
                if k.Spec.GatewaySelector != nil &&
                    k.GetNamespace() != gateway.GetNamespace() &&
                    !hasValidReferenceGrant(referenceGrants, k, gateway) {
                    continue
                }
                parents = append(parents, k)
            }
            return parents
        },
    }
}
```

The `selectorMatchesGateway` function handles the backwards-compatibility case:

```go
func selectorMatchesGateway(selector *metav1.LabelSelector, gw machinery.Object) bool {
    if selector == nil {
        return true // deprecated default: select all
    }
    sel, err := metav1.LabelSelectorAsSelector(selector)
    if err != nil {
        return false
    }
    return sel.Matches(labels.Set(gw.(metav1.Object).GetLabels()))
}
```

The topology DAG changes from:

```
Kuadrant → GatewayClass → Gateway → HTTPRoute
                                   → GRPCRoute
```

to:

```
Kuadrant ─┬→ Gateway → HTTPRoute
           │            → GRPCRoute
           ├→ Authorino
           └→ Limitador
```

GatewayClass remains in the topology (linked to Gateways by policy-machinery's standard `LinkGatewayClassToGatewayFunc`), but the **Kuadrant→GatewayClass link is removed**. The Kuadrant-to-Authorino and Kuadrant-to-Limitador links are unchanged (same namespace, hardcoded names).

**Impact on GatewayClass-dependent reconcilers:** Three effective policy reconcilers (`effective_auth_policies_reconciler.go`, `effective_ratelimit_policies_reconciler.go`, `effective_tokenratelimit_policies_reconciler.go`) currently call `targetables.Children(kuadrant)` expecting to receive GatewayClasses, then walk `GatewayClass→Gateway→Route→RouteRule` to compute effective policies. With the topology change, `targetables.Children(kuadrant)` returns Gateways directly. These reconcilers must be updated to start path enumeration from Gateways instead of GatewayClasses.

Reconcilers that need GatewayClass information for other purposes (e.g., checking `ControllerName` to distinguish Istio from Envoy Gateway) can still resolve a Gateway's GatewayClass via `topology.Targetables().Parents(gateway)` — this is the standard policy-machinery link and is unaffected by the Kuadrant→GatewayClass removal.

### `GetKuadrantFromTopology` → `GetKuadrantForGateway`

`GetKuadrantFromTopology` (returns the oldest singleton) is replaced by `GetKuadrantForGateway(topology, gateway)` which walks `topology.Parents(gateway)` to find the owning Kuadrant. See the before/after in the [topology-aware resolution](#what-enables-this-topology-aware-kuadrant-resolution) section. All 25 call sites across 23 files must be updated.

### Runtime Gateway overlap detection

A runtime precondition check detects Gateway overlap before each reconciliation cycle:

```go
func detectGatewayOverlaps(topology *machinery.Topology) map[string]overlapResult {
    // For each Gateway, collect all Kuadrant parents from the topology
    // If a Gateway has >1 Kuadrant parent, determine the winner by creationTimestamp
    // Return map of Gateway locator → {winner: *Kuadrant, losers: []*Kuadrant}
}
```

When a runtime conflict is detected:
1. **The oldest Kuadrant instance (by `creationTimestamp`) retains ownership.** It continues to reconcile the contested Gateway normally.
2. **The newer Kuadrant instance(s) get `Available: False` with reason `GatewayConflict`:** `"Gateway {ns}/{name} is already managed by Kuadrant {owner-ns}/{owner-name}. Remove the overlapping selector or relabel the Gateway to resolve."` The contested Gateway is excluded from the newer instance's topology — it does not reconcile policies for that Gateway.
3. **An event is emitted on the Gateway** identifying the conflict.
4. **Once the conflict is resolved** (labels changed, selector updated, or conflicting Kuadrant deleted), the condition is cleared automatically on the next reconciliation cycle.

### ReferenceGrant validation

The operator must watch `ReferenceGrant` resources in `state_of_the_world.go`. Validation is performed inside `LinkKuadrantToGateways` (see code above) — cross-namespace links without a valid ReferenceGrant are skipped when `gatewaySelector` is set. Legacy mode (nil selector) skips this check entirely.

### Extension impact

Extensions receive topology context via gRPC. The topology they receive will be scoped to the Gateways managed by their Kuadrant instance. Extensions using `kuadrant.Resolve()` will only see policies relevant to their instance's Gateways.

## Error reporting

| Scenario | Where | Outcome |
|----------|-------|---------|
| Second Kuadrant in same namespace | VAP (admission) | **Rejected:** `only one Kuadrant resource is allowed per namespace {ns}` |
| Two Kuadrant instances select the same Gateway | Controller (status) | **`Available: False`, reason `GatewayConflict`:** `Gateway {ns}/{name} is already managed by Kuadrant {owner-ns}/{owner-name}`. Older Kuadrant retains ownership. |
| Cross-namespace Gateway, no ReferenceGrant | Controller (status) | **`Available: False`, reason `ReferenceGrantRequired`:** `ReferenceGrant required in {gw-ns} to allow cross-namespace reference from {kuadrant-ns}` |
| Policy targets Gateway not managed by any Kuadrant | Controller (status) | **`Accepted: False` on policy:** `Target Gateway is not managed by any Kuadrant instance` |
| No Gateways match selector | Controller (status) | **`Available: True` with message:** `No Gateways match the configured gatewaySelector` |
| No gatewaySelector specified (migration period) | Controller (status) | **`Available: True`, reason `DeprecatedDefaultSelector`:** `gatewaySelector is not set; defaulting to all Gateways. Set an explicit gatewaySelector.` |

# Drawbacks
[drawbacks]: #drawbacks

1. **Breaking change on the horizon.** While the initial release is fully backwards compatible, the eventual default change (omitted selector = select nothing) will break users who don't migrate. The deprecation period must be long enough for adoption.

2. **Increased RBAC complexity.** ReferenceGrants must be created in each Gateway namespace for cross-namespace management. For clusters with many namespaces, this is additional operational overhead.

3. **Increased data plane resource consumption.** Each Kuadrant instance runs its own Limitador and Authorino. For n tenants, that's n Limitador + n Authorino deployments. The control plane (operator, DNS operator, cert-manager) is unaffected. Shared data plane backends would be more resource-efficient but harder to isolate.

4. **Data plane wiring refactor scope.** The cluster reconcilers, extension reconcilers, AuthConfig reconciler, and Limitador reconciler all assume a single global Kuadrant instance. Each must be refactored to resolve the correct Kuadrant per Gateway, derive the correct backend hostname, and scope generated resources accordingly. An error here means a Gateway could be wired to the wrong tenant's Authorino or Limitador — a data isolation failure. This is the highest-risk change in the proposal.

5. **GatewayClass link removal.** The current Kuadrant→GatewayClass→Gateway chain is replaced with a direct Kuadrant→Gateway link. Any logic that depends on the GatewayClass ancestry must be updated or must independently resolve the GatewayClass from the Gateway.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why gateway-based selection over GatewayClass-based

GatewayClass-based selection (one Kuadrant per GatewayClass) is simpler but too coarse. Multiple teams often share the same gateway implementation (e.g., Istio) but need separate policy stacks. Gateway-level selection supports this without requiring artificial GatewayClass proliferation.

## Why label selectors over targetRef

An alternative is explicit `targetRef` references to Gateways, similar to how MCPGatewayExtension targets a specific Gateway by name/namespace. This has precedent within the Kuadrant ecosystem and aligns with how Kuadrant policies already use `targetRef` to reference their target resources.

**Advantages of targetRef:**
- Explicit and unambiguous — no label coordination required between Kuadrant CR and Gateway owners
- Follows existing Kuadrant policy attachment patterns (`targetRef` on AuthPolicy, RateLimitPolicy, etc.)
- ReferenceGrant validation is well-understood for targetRef-style references (MCPGatewayExtension already implements this)
- Easier to reason about at admission time — the set of managed Gateways is static in the spec, not derived from labels

**Advantages of label selectors (proposed):**
- Dynamic set membership — Gateways can self-enroll by adding a label without modifying the Kuadrant CR
- Standard Kubernetes pattern (Deployments→Pods, Services→Endpoints, etc.)
- Supports set-based expressions (`matchExpressions`) for flexible grouping
- Better GitOps ergonomics — Gateway and Kuadrant CR can be managed by different teams/repos without coordination
- No need to update the Kuadrant CR when Gateways are added/removed

**Why this RFC proposes label selectors:** The primary use case is platform teams managing Gateways independently. Label selectors allow Gateway owners to opt in without requiring the Kuadrant admin to update the Kuadrant CR. However, targetRef is a viable alternative worth considering during RFC review — particularly if the coordination overhead of labels is a concern or if a single Kuadrant instance typically manages a small, stable set of Gateways.


## Why ReferenceGrants over RBAC

Gateway API's ReferenceGrant mechanism is the established pattern for cross-namespace authorization in the Gateway API ecosystem. Using standard RBAC would work but:
- Is less visible to Gateway API tooling
- Doesn't follow the precedent set by Gateway→Secret, HTTPRoute→Service references
- Would be non-standard for the ecosystem

## Impact of not doing this

Kuadrant's data plane remains single-tenant per cluster. Users needing auth/rate-limiting isolation must run separate clusters, duplicating not just data plane components but also the operator, DNS operator, cert-manager, and gateway infrastructure — all of which could otherwise be shared.

# Prior art
[prior-art]: #prior-art

- **Istio revisions** — `istio.io/rev` label on namespaces selects which `istiod` manages workloads. Namespace-level granularity has proven too coarse for some users, motivating Gateway-level selection here.
- **Envoy Gateway** — `--gateway-class-name` scopes each controller to one GatewayClass. Simple but requires GatewayClass proliferation when tenants share the same data plane.
- **cert-manager** — `Issuer` (namespaced) vs `ClusterIssuer` (cluster-scoped). Dual-scope CRDs add API surface; our label selector avoids that.
- **Gateway API ReferenceGrant** — established pattern for cross-namespace references (HTTPRoute→Service, etc.). We adopt the same pattern for Kuadrant→Gateway.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- **Migration timeline.** How many releases should the deprecation period for the implicit "select all" behavior span? One minor release? Two?

- **Extension isolation.** Do extensions need additional scoping beyond the topology filtering they inherit? Can an extension controller in namespace A affect Gateways managed by namespace B's Kuadrant? This may require per-Kuadrant-instance extension sockets.

- **Shared Limitador/Authorino.** This RFC mandates per-instance backends. Should a future iteration allow shared backends with logical isolation (e.g., Limitador namespaces, Authorino tenant labels)?

- **Label selector expressiveness.** Should `gatewaySelector` support `matchExpressions` in addition to `matchLabels`, or is `matchLabels` sufficient for the initial implementation? (Using `metav1.LabelSelector` supports both, but should the docs/examples emphasize one?)

- **Same-namespace ReferenceGrant exemption.** This RFC proposes that same-namespace Gateways don't need a ReferenceGrant. Should this be configurable for security-sensitive deployments that want explicit grants even within a namespace?

# Future possibilities
[future-possibilities]: #future-possibilities

- **Quota/resource limits per Kuadrant instance.** Platform admins could set limits on the number of Gateways, policies, or rate limit configurations a single Kuadrant instance can manage.

- **Shared backend pools.** Allow multiple Kuadrant instances to share a single Limitador or Authorino deployment with logical isolation (e.g., Limitador namespace prefixes, Authorino label-based filtering).

- **Hierarchical tenancy.** A "super-admin" Kuadrant instance could define override policies that apply across all tenant instances, similar to the existing defaults/overrides mechanism but across tenancy boundaries.

- **Automatic ReferenceGrant provisioning.** A higher-level controller or webhook could auto-create ReferenceGrants based on namespace ownership labels, reducing operational friction.

- **Gateway self-enrollment webhook.** A mutating webhook could automatically add the `kuadrant.io/managed-by` label when a Gateway is created in a namespace with a specific annotation, enabling zero-touch onboarding.

- **Per-instance extension sockets.** Extensions could be deployed per Kuadrant instance, each with its own Unix socket, enabling per-tenant extension behavior (e.g., different OIDC providers per tenant).
