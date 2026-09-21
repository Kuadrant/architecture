# Runtime-Managed Network Policies for Kuadrant

- Feature Name: `network-policy`
- Status: Draft
- Start Date: 2026-08-10
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [CONNLINK-1205](https://issues.redhat.com/browse/CONNLINK-1205)

# Summary
[summary]: #summary

The kuadrant-operator will dynamically create and manage standard Kubernetes NetworkPolicy objects (`networking.k8s.io/v1`) at runtime to provide ingress-only network isolation for all components across the operator and operand namespaces.
This is an always-on feature requiring no user configuration.

Policies are created in two scopes:

- **Operator namespace**: A label-scoped deny policy targeting pods with `kuadrant.io/managed: "true"`, plus per-operator allow policies.
- **Operand namespace** (the namespace where the Kuadrant CR is created): A label-scoped deny policy targeting pods with `kuadrant.io/managed: "true"`, plus per-operand allow policies.

Both namespaces use label-scoped deny to avoid disrupting other workloads that may share the namespace.
When the operator and operand namespaces are the same, a single deny policy and the combined set of allow policies are created.

# Motivation
[motivation]: #motivation

Today, all pods in the Kuadrant deployment accept ingress traffic from any source in the cluster.
Any compromised pod or misconfigured workload can reach internal gRPC, wasm-serving, metrics, and OIDC endpoints without restriction.
There is no defense-in-depth at the network layer.

This is a security gap that needs to be closed.
The feature establishes a deny-by-default ingress posture with explicit allow rules for known traffic patterns.

Several constraints shape the approach:

- **OLM compatibility**: OLM bundle policies only cover the operator — operand policies must be managed at runtime regardless.
Additionally, not all OLM versions support NetworkPolicy in bundles.
- **Operand placement**: Operands (Authorino, Limitador) are deployed in the Kuadrant CR's namespace, which may differ from the operator namespace.
Policies must follow.
- **Egress complexity**: Locking down egress would require tracking Redis endpoints (Limitador), OIDC providers (Authorino), tracing endpoints, DNS resolution, and API server IPs — all dynamic and fragile.
Ingress-only deny provides meaningful isolation without this complexity.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When the kuadrant-operator starts, it automatically creates NetworkPolicy resources that lock down ingress traffic to all Kuadrant components.
No user action is required — network isolation is always on.

From a user's perspective:

- **Nothing changes operationally.** All existing traffic patterns (metrics scraping, Gateway-to-Authorino gRPC, Gateway-to-Limitador gRPC, wasm-shim loading, console plugin access) continue to work.
The operator creates allow rules for all known traffic flows.
- **Unauthorized traffic is blocked.** Pods outside the expected namespaces cannot reach Kuadrant's internal ports.
A random pod in a user namespace cannot call Authorino's gRPC endpoint.
Metrics ports are open to all to support any monitoring stack, but gRPC and wasm ports are restricted to known sources.
- **Gateway namespace changes are handled automatically.** When a new Gateway is created in a namespace, the operator updates its NetworkPolicies to allow traffic from that namespace.
When a Gateway is removed, the rules are pruned.
- **Kuadrant CR namespace is respected.** If the Kuadrant CR (and therefore Authorino and Limitador) is in a different namespace from the operator, the operator creates appropriately scoped policies in both namespaces.
Policies only target pods labeled `kuadrant.io/managed: "true"`, so other workloads in either namespace are unaffected.

The only visible artifact of this feature is the set of `NetworkPolicy` resources in the operator and operand namespaces, all labeled with `kuadrant.io/managed: "true"`.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Namespace Model

The Kuadrant CR's namespace determines where operands (Authorino, Limitador) are deployed.
This may or may not be the same as the operator namespace.

- **Operator namespace**: Contains kuadrant-operator, authorino-operator, limitador-operator, dns-operator, and console-plugin.
This may be a dedicated namespace or a shared namespace with other workloads.
- **Operand namespace**: Contains Authorino and Limitador deployments.
Determined by where the user creates the Kuadrant CR.

Either namespace may contain workloads not managed by Kuadrant.
To avoid disrupting those workloads, both the operator and operand namespaces use a label-scoped deny model targeting only pods with `kuadrant.io/managed: "true"`.

| Scope | Deny Policy | Rationale |
|---|---|---|
| Operator namespace | `podSelector: { kuadrant.io/managed: "true" }` (label-scoped) | Other workloads may share this namespace. |
| Operand namespace (if different) | `podSelector: { kuadrant.io/managed: "true" }` (label-scoped) | Other workloads may share this namespace. |

### Labeling Prerequisite

This design requires that all Kuadrant component pods carry the `kuadrant.io/managed: "true"` label.
Not all components currently have this label applied.
As a prerequisite to this work, the operator and sub-operator deployments must be updated to ensure every pod managed by Kuadrant is labeled.
This includes pods created directly by the kuadrant-operator and pods created by the sub-operators (authorino-operator, limitador-operator, dns-operator).

### Upgrade Considerations

On upgrade from a version without network policies to a version with them, the labeling must be in place before the deny policies take effect.
The upgrade path must ensure:

1. **Labels are applied before deny policies are created.** The reconciler must label all existing component pods (or ensure the sub-operators roll out updated pod templates with the label) before creating deny NetworkPolicies.
If a deny policy is created before a pod is labeled, that pod will not be selected by the deny policy and will remain open — but more critically, it will also not be selected by any allow policy, so if the pod is later labeled it will immediately fall under deny-all with no allow rules if the allow policy uses a different selector.
2. **Sub-operator pod template updates trigger rolling restarts.** Adding a label to a Deployment's pod template causes a rollout.
The operator should account for the temporary disruption during this rollout on upgrade.
3. **Ordering within the reconciler.** The network policy workflow should run after the sub-operator reconciliation workflows that ensure labels are present, so that by the time deny policies are created, all pods are guaranteed to be labeled and matched by both deny and allow policies.

## Ingress-Only Deny Model

All deny and allow policies use `policyTypes: [Ingress]` only.
Egress is left unrestricted.

Restricting egress would require dynamically tracking outbound destinations (Redis, OIDC providers, tracing collectors, DNS, API server) across all components.
The API server is particularly problematic as NetworkPolicy cannot select host-network endpoints.
By restricting ingress only, the operator avoids this complexity and eliminates the chicken-and-egg problem.
The kuadrant-operator can start and communicate outbound freely, then create its own ingress allow rules.

## Policy Specifications

Every NetworkPolicy created by the operator carries the label `kuadrant.io/managed: "true"` for controller-runtime cache filtering.

### Operator Namespace Policies

**`kuadrant-deny-managed`** — Deny all ingress to Kuadrant-managed pods in the operator namespace.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: kuadrant-deny-managed
  namespace: <operator-namespace>
  labels:
    kuadrant.io/managed: "true"
spec:
  podSelector:
    matchLabels:
      kuadrant.io/managed: "true"
  policyTypes:
  - Ingress
```

**`kuadrant-np-kuadrant-operator`** — Allow ingress to the kuadrant-operator.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: kuadrant-np-kuadrant-operator
  namespace: <operator-namespace>
  labels:
    kuadrant.io/managed: "true"
spec:
  podSelector:
    matchLabels:
      app: kuadrant-operator
  ingress:
  # Metrics — open to all
  - ports:
    - protocol: TCP
      port: 8080
  # gRPC extensions — within operator namespace only
  - from:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 50051
  # Wasm-shim serving — from gateway namespaces (dynamically populated)
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: gateway-system
    ports:
    - protocol: TCP
      port: 8082
  policyTypes:
  - Ingress
```

The `from` entries for port 8082 are dynamically generated based on the namespaces where Gateway resources exist.
The example above shows a single gateway namespace; in practice the operator adds one `namespaceSelector` entry per discovered gateway namespace.

**`kuadrant-np-authorino-operator`** — Allow metrics ingress only.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: kuadrant-np-authorino-operator
  namespace: <operator-namespace>
  labels:
    kuadrant.io/managed: "true"
spec:
  podSelector:
    matchLabels:
      app: authorino-operator
  ingress:
  - ports:
    - protocol: TCP
      port: 8080
  policyTypes:
  - Ingress
```

**`kuadrant-np-dns-operator`** — Allow metrics ingress only.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: kuadrant-np-dns-operator
  namespace: <operator-namespace>
  labels:
    kuadrant.io/managed: "true"
spec:
  podSelector:
    matchLabels:
      app: dns-operator
  ingress:
  - ports:
    - protocol: TCP
      port: 8080
  policyTypes:
  - Ingress
```

**`kuadrant-np-limitador-operator`** — Allow metrics ingress only.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: kuadrant-np-limitador-operator
  namespace: <operator-namespace>
  labels:
    kuadrant.io/managed: "true"
spec:
  podSelector:
    matchLabels:
      app: limitador-operator
  ingress:
  - ports:
    - protocol: TCP
      port: 8080
  policyTypes:
  - Ingress
```

**`kuadrant-np-console-plugin`** (only when console-plugin is installed) — Allow ingress from the console.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: kuadrant-np-console-plugin
  namespace: <operator-namespace>
  labels:
    kuadrant.io/managed: "true"
spec:
  podSelector:
    matchLabels:
      app: console-plugin
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: openshift-console
    ports:
    - protocol: TCP
      port: 9443
  policyTypes:
  - Ingress
```

The console-plugin is not installed in all configurations.
The reconciler creates this policy only when it detects the console-plugin deployment in the operator namespace.
If the console-plugin is removed, this policy is removed.

### Operand Policies

When the operand namespace differs from the operator namespace, all three policies below are created in the operand namespace.
When they are the same namespace, only the two allow policies (`kuadrant-np-authorino`, `kuadrant-np-limitador`) are created — the operator namespace `kuadrant-deny-managed` already covers the deny function.

**`kuadrant-deny-managed`** (only when operand namespace differs from operator namespace) — Deny ingress to Kuadrant-managed pods.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: kuadrant-deny-managed
  namespace: <kuadrant-cr-namespace>
  labels:
    kuadrant.io/managed: "true"
spec:
  podSelector:
    matchLabels:
      kuadrant.io/managed: "true"
  policyTypes:
  - Ingress
```

**`kuadrant-np-authorino`** — Allow ingress to Authorino.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: kuadrant-np-authorino
  namespace: <kuadrant-cr-namespace>
  labels:
    kuadrant.io/managed: "true"
spec:
  podSelector:
    matchLabels:
      app: authorino
  ingress:
  # Metrics — open to all
  - ports:
    - protocol: TCP
      port: 8080
  # gRPC — from gateway namespaces
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: gateway-system
    ports:
    - protocol: TCP
      port: 50051
  # HTTP auth — from gateway namespaces
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: gateway-system
    ports:
    - protocol: TCP
      port: 5001
  # OIDC discovery — open to all
  - ports:
    - protocol: TCP
      port: 8083
  policyTypes:
  - Ingress
```

**`kuadrant-np-limitador`** — Allow ingress to Limitador.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: kuadrant-np-limitador
  namespace: <kuadrant-cr-namespace>
  labels:
    kuadrant.io/managed: "true"
spec:
  podSelector:
    matchLabels:
      app: limitador
  ingress:
  # Metrics — open to all
  - ports:
    - protocol: TCP
      port: 8080
  # gRPC — from gateway namespaces
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: gateway-system
    ports:
    - protocol: TCP
      port: 8081
  policyTypes:
  - Ingress
```

## Port Matrix

| Component | Namespace | Port | Protocol | Purpose | Ingress Source |
|---|---|---|---|---|---|
| kuadrant-operator | Operator | 8080 | TCP | Metrics | Any |
| kuadrant-operator | Operator | 50051 | TCP | gRPC extensions | Within operator namespace |
| kuadrant-operator | Operator | 8082 | TCP | Wasm-shim serving | Gateway namespaces |
| kuadrant-operator | Operator | 8081 | TCP | Readiness probe | Not blocked (kubelet uses host network) |
| authorino-operator | Operator | 8080 | TCP | Metrics | Any |
| authorino-operator | Operator | 8081 | TCP | Readiness probe | Not blocked |
| dns-operator | Operator | 8080 | TCP | Metrics | Any |
| dns-operator | Operator | 8081 | TCP | Readiness probe | Not blocked |
| limitador-operator | Operator | 8080 | TCP | Metrics | Any |
| limitador-operator | Operator | 8081 | TCP | Readiness probe | Not blocked |
| console-plugin | Operator | 9443 | TCP | Console UI | `openshift-console` namespace |
| authorino | Operand | 8080 | TCP | Metrics | Any |
| authorino | Operand | 50051 | TCP | gRPC auth | Gateway namespaces |
| authorino | Operand | 5001 | TCP | HTTP auth | Gateway namespaces |
| authorino | Operand | 8083 | TCP | OIDC discovery | Any |
| limitador | Operand | 8080 | TCP | Metrics | Any |
| limitador | Operand | 8081 | TCP | gRPC rate limiting | Gateway namespaces |

## RBAC

The existing `manager-role` ClusterRole requires one addition:

```yaml
- apiGroups:
  - networking.k8s.io
  resources:
  - networkpolicies
  verbs:
  - create
  - delete
  - get
  - list
  - patch
  - update
  - watch
```

Since `manager-role` is already a ClusterRole, this single addition covers both the operator namespace and any operand namespace.
No new Roles or RoleBindings are needed.

## Controller-Runtime Cache Filtering

All managed NetworkPolicies carry the label `kuadrant.io/managed: "true"`.
The controller-runtime cache for `networking.k8s.io/v1/NetworkPolicy` is configured with a label selector matching this label.
This ensures the operator only watches and caches NetworkPolicies it owns, avoiding unnecessary load from other NetworkPolicies in the cluster.

## Reconciler Design

NetworkPolicy management is implemented as new workflows in the existing state-of-the-world reconciler.

**Trigger:** The reconciler runs on every state-of-the-world sync.
NetworkPolicy workflows are evaluated alongside existing workflows.

**Workflow steps:**

1.
**Determine namespaces** — Read the Kuadrant CR to identify the operand namespace.
Compare with the operator's own namespace.

2.
**Build desired state** — Construct the full set of desired NetworkPolicy objects based on the policy specifications above.
Gateway namespaces for wasm-shim and gRPC ingress rules are derived from the Gateway resources already present in the reconciler's topology.

3.
**Reconcile** — For each desired NetworkPolicy, create or update to match the desired state.
Remove any NetworkPolicy with `kuadrant.io/managed: "true"` that is no longer in the desired set (e.g., a Gateway namespace was removed, or the operand namespace changed).

**Dynamic gateway namespace handling:** When a new Gateway is created in a previously unseen namespace, the next reconciliation adds that namespace to the ingress `from` rules on the kuadrant-operator (port 8082), authorino (ports 50051, 5001), and limitador (port 8081) policies.
When a Gateway is removed from a namespace (and no other Gateways remain there), those namespace entries are pruned from the ingress rules.

**Operand namespace changes:** If the Kuadrant CR is deleted from one namespace and created in another, the reconciler removes the operand policies (`kuadrant-deny-managed`, `kuadrant-np-authorino`, `kuadrant-np-limitador`) from the old namespace and creates them in the new one.

## Lifecycle and Cleanup

NetworkPolicies use Kubernetes owner references for lifecycle management:

- **Operator namespace policies**: Owned by the kuadrant-operator Deployment.
When the Deployment is deleted (uninstall), Kubernetes garbage collection automatically removes all operator-namespace NetworkPolicies.
- **Operand namespace policies**: Owned by the Kuadrant CR.
When the Kuadrant CR is deleted, Kubernetes garbage collection automatically removes the operand-namespace NetworkPolicies along with the operands themselves.

Owner references only function within a single namespace.
This is compatible with the design because operator-namespace policies and their owner (the Deployment) are co-located, and operand-namespace policies and their owner (the Kuadrant CR) are co-located.

This follows the same ownership model used by the rest of the operator (see RFC 0019) and avoids the need for finalizers or manual cleanup logic.

# Drawbacks
[drawbacks]: #drawbacks

- **Labeling prerequisite**: All Kuadrant component pods must carry the `kuadrant.io/managed: "true"` label for the deny policy to cover them.
Any component pod missing this label will not be selected by the deny policy and will remain open to all ingress.
This labeling work must be completed across all sub-operators before network policies are effective.
- **No egress restrictions** means a compromised pod can still make outbound connections to arbitrary destinations.
This is an accepted trade-off for the significant reduction in implementation complexity.
- **Pod selector labels** (`app: kuadrant-operator`, `app: authorino`, etc.) must match the actual labels on component deployments.
If upstream operators change their pod labels, the NetworkPolicies will stop matching and those components will lose their allow rules (falling back to deny-all).
- **Gateway namespace ingress is namespace-scoped, not pod-scoped.** Ingress rules for gateway-facing ports (wasm-shim 8082, authorino gRPC 50051/HTTP 5001, limitador gRPC 8081) allow traffic from any pod in the gateway namespace, not just the gateway proxy pods.
This means a non-gateway workload co-located in that namespace could reach these ports.
Three alternatives were considered:
  - *Add a pod selector using gateway provider labels* (e.g., `gateway.networking.k8s.io/gateway-name`): Tighter access, but couples us to labeling conventions that vary between providers (Istio, Envoy Gateway) and may change without notice.
A label change would silently break the allow rule, causing the component to fall back to deny-all.
  - *Label gateway pods ourselves* (e.g., `kuadrant.io/gateway-proxy: "true"`): We control the label, but requires RBAC to patch pods in other namespaces, and the gateway provider's reconciler may strip labels it does not recognise.
  - *Namespace selector only* (chosen): When referring to pods managed by a different operator, using just the namespace selector allows the remote operator to restructure without breaking the policy.
Simplest to manage, decoupled from provider internals, but wider access than strictly necessary.
The risk is accepted because gateway namespaces are typically infrastructure namespaces with controlled workload placement.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why this design

- **Runtime management** gives a single code path for both operator and operand policies, avoids OLM compatibility issues, and can react to dynamic state (Gateway namespaces, Kuadrant CR placement).
- **Standard NetworkPolicy** is GA, universally supported by all CNI plugins, and does not require elevated cluster-scoped RBAC beyond what the operator already has.
- **Ingress-only deny** provides meaningful security improvement while avoiding the complexity explosion of egress restrictions.
- **One policy per component** is easy to debug, uses descriptive naming, and allows independent evolution of each component's rules.

## Alternatives considered

### Static NetworkPolicies in OLM Bundles

Rejected.
Not all OLM versions support NetworkPolicy in bundles.
Additionally, OLM bundle policies only cover the operator — operand policies must be managed by the operator at runtime regardless.
This would create split ownership with no benefit.

### AdminNetworkPolicy / BaselineAdminNetworkPolicy

Explored in [PR #2122](https://github.com/Kuadrant/kuadrant-operator/pull/2122) as a reference implementation.
Rejected because:
- Requires OVN-Kubernetes CNI — not portable across CNI plugins
- Cluster-scoped resources require elevated RBAC
- `policy.networking.k8s.io/v1alpha1` is still alpha
- Standard NetworkPolicy (`networking.k8s.io/v1`) is GA and universally supported

### Egress + Ingress Deny Model

Rejected.
Egress restrictions would require tracking Redis endpoints (Limitador), OIDC providers (Authorino), tracing endpoints, DNS resolution, and API server IPs — all dynamic and fragile.
The API server is particularly problematic: NetworkPolicy cannot select host-network endpoints, so egress to the API server requires either allowing all egress or dynamically discovering the API server's IP.
Ingress-only deny provides meaningful isolation without this complexity.

### Opt-in via Kuadrant CR Configuration

Rejected.
Network isolation is a security baseline.
Making it optional means deployments that don't explicitly enable it remain fully open.
Always-on ensures every deployment gets protection by default.

### Helm Chart Templates

Not chosen as the primary mechanism.
Runtime management by the operator handles Helm, OLM, and direct installations uniformly, and can react to dynamic state (Gateway namespaces appearing, Kuadrant CR placement changes) that static templates cannot.

## Impact of not doing this

All Kuadrant components remain fully open to ingress traffic from any pod in the cluster, with no defense-in-depth at the network layer.

# Prior art
[prior-art]: #prior-art

- **PR #2122 — AdminNetworkPolicy reference** — A draft PR by Boomatang implementing deny-by-default using ANP/BANP (`policy.networking.k8s.io/v1alpha1`).
Served as a learning exercise and identified issues (incorrect BANP naming, missing rules, overly broad selectors) that informed this RFC's design.
- **PR #2121 — CRC development setup** — A draft PR providing a CRC-based development environment with network policy manifests, make targets, and an observability stack for iterating on network policy designs.
- **Kubernetes NetworkPolicy** — The standard `networking.k8s.io/v1` API used by this RFC.
GA since Kubernetes 1.7, supported by all CNI plugins.
Key behaviors: "No Policy == Allow all", "Any Policy == Default deny" for the selected direction, policies are additive (can only add allow rules), health checks from kubelet are not blocked (host network).
- **Network Policy API project** — The upstream Kubernetes SIG-Network project developing AdminNetworkPolicy and BaselineAdminNetworkPolicy as cluster-scoped alternatives to namespace-scoped NetworkPolicy.
Still alpha (`v1alpha1`), with a newer `v1alpha2` introducing ClusterNetworkPolicy with explicit tiers.
Not used in this RFC due to OVN-Kubernetes requirement and alpha status.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- **Pod selector labels**: The YAML examples use labels like `app: kuadrant-operator` and `app: authorino`.
The actual labels on component deployments need to be verified during implementation and may differ.
- **Label coverage audit**: A full audit of which component pods currently carry the `kuadrant.io/managed: "true"` label is needed.
Any gaps must be addressed as a prerequisite to this work.

# Future possibilities
[future-possibilities]: #future-possibilities

- **Egress restrictions**: A future RFC could add egress rules for components with well-known, static egress targets.
The ingress-only model in this RFC is designed to be extended with egress rules additively — no existing policies would need to change.
- **AdminNetworkPolicy integration**: As ANP matures and gains broader CNI support, the operator could optionally create ANP resources alongside or instead of standard NetworkPolicies for clusters that support them.
The deny-all baseline could be implemented as a BANP for cluster-wide enforcement.
- **User-configurable allow rules**: A future Kuadrant CR field could allow users to add custom ingress sources (e.g., additional monitoring namespaces, debugging tools) without modifying the operator's managed policies.
- **Network policy status reporting**: The operator could report the status of managed NetworkPolicies in the Kuadrant CR status, giving users visibility into the network isolation posture.
- **Multi-cluster support**: When Kuadrant supports multi-cluster deployments, network policies may need to account for cross-cluster traffic patterns.

# References

## Research and Prior Work
- [PR #2121 — CRC development setup with network policies](https://github.com/Kuadrant/kuadrant-operator/pull/2121)
- [PR #2122 — AdminNetworkPolicy reference implementation](https://github.com/Kuadrant/kuadrant-operator/pull/2122)

## Kubernetes
- [Kubernetes NetworkPolicy documentation](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Network Policy API](https://network-policy-api.sigs.k8s.io/)
