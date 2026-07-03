# RFC: Evolving the Kuadrant Deployment

- Feature Name: `kuadrant_deployment_consolidation`
- Start Date: 2026-06-11
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [Kuadrant/architecture#164](https://github.com/Kuadrant/architecture/issues/164)

# Summary
[summary]: #summary

Maintain a single-action install experience with OLMv1. Reduce Kuadrant's operator footprint, release complexity, security surface area, and maintenance burden by eliminating authorino-operator and limitador-operator as separate components in a Kuadrant deployment. Make Helm the primary installation and configuration mechanism for Kuadrant across OpenShift, *.KS (AKS, GKS) and general Kubernetes. Introduce an "umbrella operator" to coordinate installation via programmatic Helm installation.

# Motivation
[motivation]: #motivation

### OLM dependency resolution is being removed

OLMv1 [explicitly removes automatic dependency installation](https://github.com/operator-framework/operator-controller/blob/main/docs/project/olmv1_design_decisions.md). The [design decisions document](https://github.com/operator-framework/operator-controller/blob/main/docs/project/olmv1_design_decisions.md) states:

> OLM v1 will not automatically install a missing dependency when a user requests an operator. The primary reasoning is that OLM v1 will err on the side of predictability and cluster-administrator awareness.

Bundles declaring `olm.package.required`, `olm.gvk.required`, or `olm.constraint` will be rejected under OLMv1. Kuadrant's current model relies entirely on these declarations.

The exact process for transitioning a cluster (fully) from OLMv0 to OLMv1 is not yet documented — this is an open question that affects all operators, not just Kuadrant. Currently on OpenShift OLMv1 and OLMv0 run alongside each other. However, regardless of when or how that transition happens, the dependency resolution mechanism Kuadrant relies on will not exist in the future. We should eliminate our dependency declarations now, while still on OLMv0, so that Kuadrant is already compatible when the transition arrives.

### Most of our dependency operators are simple wrappers

Operators are also acknowledged as a relatively complex solution for simple deployments. The [OLMv1 design decisions](https://github.com/operator-framework/operator-controller/blob/main/docs/project/olmv1_design_decisions.md#authors-of-simple-operators-ship-their-workload-without-an-operator) document identifies workloads that don't benefit from the operator pattern:

> A sizable portion of the OperatorHub catalog contain operators that are not actually taking advantage of the benefits of the operator pattern and are instead a simple wrapper around the workload. Using the Operator Capability Levels as a rubric, operators that fall into Level 1 and some that fall into Level 2 are not making full use of the operator pattern.
> If content authors had the choice to ship their content without also shipping an operator that performs simple installation and upgrades, many supporting these Level 1 and Level 2 operators might make that choice to decrease their overall maintenance and support burden while losing very little in terms of value to their customers.

These dependency operators exist because OLMv0 required each component to be packaged as its own operator with its own CSV, Subscription, and CRD. They were created to satisfy this packaging requirement, not because the workloads they manage have complex lifecycle needs. Some of our operators fall into this category:

**authorino-operator**: The reconciler creates a ServiceAccount, Services, RBAC, and a Deployment. It maps `Authorino` CR fields to Deployment args and volumes. The only non-trivial logic is toggling between cluster-wide and namespaced RBAC. A Helm chart with conditionals provides identical functionality.

**limitador-operator**: The reconciler creates a Service, Deployment, limits ConfigMap, optional PVC, and optional PodDisruptionBudget. The storage backend switch (in-memory, Redis, Redis-cached, disk) translates to different args, volumes, and deployment strategy. Still a spec-to-Deployment mapper replaceable by Helm.

The `Limitador` CR has an additional overlap: it also declares rate limits in `spec.limits`. Today, kuadrant-operator writes limits into this field, limitador-operator serializes them into a ConfigMap, and Limitador reads the mounted config file. In Kuadrant, the kuadrant-operator via policy is the only producer of limits kuadrant-operator can write the ConfigMap directly.

### Kuadrant needs to run beyond OpenShift

Kuadrant's deployment target(s) are not limited to OpenShift. In the future, we want to support any Kubernetes distribution — EKS, GKE, AKS, and others. An architecture that depends on OLM for installation and lifecycle management ties Kuadrant to an OpenShift-specific mechanism that doesn't exist on these platforms. Helm is the common denominator. Making Helm the primary installation method ensures Kuadrant is a first-class citizen on any Kubernetes distribution, with the umbrella operator serving as an OpenShift-specific layer.

### Fewer operators means a smaller security surface area and maintenance overhead

Each operator is a long-running privileged controller with its own ServiceAccount and cluster-scoped RBAC. authorino-operator holds permissions to create and modify ClusterRoles, ClusterRoleBindings, Deployments, Services, and Secrets across namespaces.

Each operator is also a separate container image with its own Go dependency tree. Every transitive dependency is a potential CVE that needs scanning, patching, and releasing. Eliminating two operators removes two privileged controllers from the cluster and two dependency trees from the security maintenance pipeline.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### Install on upstream Kubernetes

Users install Kuadrant directly via Helm:

```bash
helm install kuadrant kuadrant/kuadrant
```

The chart deploys all components — kuadrant-operator, Authorino, Limitador, and dns-operator — with configurable values for image references, replicas, resource limits, storage backend, and TLS.

### Install on OpenShift (OLM)

Users create a single Subscription (OLMv0) or ClusterExtension (OLMv1) for the umbrella operator. This is the only OLM resource the user creates. The umbrella operator renders the same Helm charts internally and deploys all components.

### Kuadrant CR

Users create a Kuadrant CR to configure operand behaviour — mTLS, observability, tracing, and developer portal. The umbrella operator now owns and reconciles this CR. kuadrant-operator watches only policy CRs (AuthPolicy, RateLimitPolicy, DNSPolicy, TLSPolicy, TokenRateLimitPolicy).

### Breaking changes

The `Authorino` and `Limitador` CRDs are part of Kuadrant's internal API surface — they are not user-facing. The public API (Kuadrant CR, AuthPolicy, RateLimitPolicy, DNSPolicy, TLSPolicy, TokenRateLimitPolicy) is unchanged.

- **`Authorino` CRD dropped** — internal deployment configuration moves to Helm values.
- **`Limitador` CRD dropped** — internal deployment configuration moves to Helm values. Rate limits written directly to ConfigMap by kuadrant-operator.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Umbrella Operator

A single operator published as an OLMv0 package with zero dependency declarations. It manages all Kuadrant components internally by rendering Helm charts at runtime using Helm's Go SDK with `ClientOnly=true` and `DryRun=true` — pure template rendering with no Helm release tracking. Rendered manifests are applied via controller-runtime. Charts are embedded in the umbrella operator image at build time.

The umbrella operator owns the component Deployments — not OLM. authorino-operator and limitador-operator are eliminated and their workloads are deployed directly. The umbrella operator provides a place to coordinate migration logic for cleaning up these operators on existing clusters.

On OLMv0, the umbrella operator is installed via a standard Subscription. If/when the cluster transitions to OLMv1, it migrates as a single ClusterExtension with no dependencies.

This follows the pattern used by OpenShift's [cluster-olm-operator](https://github.com/openshift/cluster-olm-operator).

### Helm Chart Refactoring

The current kuadrant-operator chart (`charts/kuadrant-operator/`) is a monolithic kustomize-generated blob — a single `manifests.yaml` of ~14,500 lines produced by `kustomize build config/helm`. This must be refactored to support per-component rendering, ordered deployment, and direct workload deployment.

**Required changes:**

1. **Replace kustomize generation with per-component Helm templates** — one template per component (Deployment, Service, RBAC, ServiceAccount). CRDs must be separated (via Helm's `crds/` directory or a dedicated template) since they must be established before resources that depend on them.
2. **Configurable values.yaml** — image references, namespaces, replica counts, and resource limits exposed as values. The umbrella operator injects version-pinned values at render time; direct Helm users override via `--set` or values files.
3. **Workload templates for Authorino and Limitador** — the chart deploys these workloads directly (Deployment, Service, RBAC) rather than creating operator CRs. Configuration previously exposed via the `Authorino` and `Limitador` CRs moves to Helm values.
4. **Ordered rendering support** — the umbrella operator renders and applies charts sequentially: CRDs first, then workloads and dependencies, then kuadrant-operator.

> The exact layout and requirements for these charts is something to be investigated and beyond the details of this doc.

### Managed Resources

Resource ownership is split between OLM and the umbrella operator.

**Managed by OLM (via the umbrella operator's bundle/CSV):**

| Resource | Purpose |
|---|---|
| CustomResourceDefinitions | All Kuadrant and DNS CRDs. Authorino and Limitador CRDs are dropped. OLM handles install, upgrade, and CRD safety checks. |
| ClusterRoles / ClusterRoleBindings | RBAC for the umbrella operator and kuadrant-operator. |
| Umbrella operator Deployment | The umbrella operator itself. |

**Managed by the umbrella operator (rendered from Helm charts):**

| Resource | Purpose |
|---|---|
| Deployments | kuadrant-operator, dns-operator, Authorino (workload), Limitador (workload) |
| ServiceAccounts | Per-component service identity |
| Roles / RoleBindings | Namespace-scoped RBAC  |
| Services | Metrics, auth, OIDC, rate limiting |
| ConfigMaps | Limitador limits (written by kuadrant-operator), operator configuration |

**Not managed by the umbrella operator or OLM:**

| Resource | Owner |
|---|---|
| Policy CRs (AuthPolicy, RateLimitPolicy, DNSPolicy, TLSPolicy, etc.) | User-created, reconciled by kuadrant-operator |
| AuthConfig CRs | Created by kuadrant-operator, reconciled by Authorino |

### Kuadrant CR Ownership

The Kuadrant CR moves from kuadrant-operator to the umbrella operator. Analysis of the [Kuadrant CR spec](https://github.com/Kuadrant/kuadrant-operator/blob/main/doc/reference/kuadrant.md) shows that its fields are concerned with operand management, operand configuration, and infrastructure wiring — not policy reconciliation.

| Spec field | Action | Category |
|---|---|---|
| (existence of CR) | Previously created Authorino and Limitador operand CRs — now triggers Helm-rendered workload configuration | Operand management |
| `spec.mtls.*` | Configures PeerAuthentication for mTLS between gateway and operands | Operand configuration |
| `spec.observability.enable` | Creates ServiceMonitor and PodMonitor CRs | Infrastructure wiring |
| `spec.observability.tracing.*` | Sets tracing endpoint on Authorino and Limitador | Operand configuration |
| `spec.observability.dataPlane.*` | Drives wasm-shim configuration via effective policy computation | Data plane configuration |
| `spec.components.developerPortal.enabled` | Creates/deletes Developer Portal Deployment | Operand management |

kuadrant-operator stops watching the Kuadrant CR entirely — it only watches policy CRs.

**Data plane observability configuration:** The one field that crosses the boundary is `spec.observability.dataPlane`. The umbrella operator projects this configuration to kuadrant-operator via a ConfigMap or environment variables on the kuadrant-operator Deployment. The user-facing API remains unchanged. 

### Migration (from existing OLMv0 installs)

> Note this is an outline only and it needs more thorough investigation and testing to validate

The umbrella operator is introduced **while the cluster is still running OLMv0**. On startup, it takes over deployment of all Kuadrant components. The migration sequence ensures zero data plane disruption.

#### Migration flow

1. **Scale down authorino-operator to 0 replicas** — stops reconciling. Authorino pods continue running. Data plane unaffected.
2. **Strip ownerReferences** from Authorino Deployment, Services, and RBAC pointing to the `Authorino` CR.
3. **Apply Helm-rendered Authorino resources** (same names) — takes ownership. No pod restart — same Deployment spec.
4. **Strip finalizer from `Authorino` CR, then delete it** — no cascading deletion since ownerReferences were removed.
5. **Repeat for limitador-operator** — scale down, strip ownerReferences from Limitador Deployment/Service/ConfigMap/PVC/PDB, apply Helm resources, strip finalizer, delete `Limitador` CR.
6. **Deploy updated kuadrant-operator from Helm** — this version writes limits ConfigMap directly and drops its watch on the Kuadrant CR.
7. **Umbrella operator takes ownership of the Kuadrant CR** — begins reconciling operand configuration, mTLS, observability.
8. **Clean up OLMv0 Subscriptions and CSVs** for the dependency operators — CSV deletion cascades to the operator Deployments (authorino-operator, limitador-operator), removing them. The operators are already scaled to 0 at this point so there is no impact. Helm-rendered workload resources are unaffected as they are owned by the umbrella operator, not the CSVs.

Each step is idempotent — if the umbrella operator restarts mid-migration, it re-checks state and resumes.

> Note: the first version of the umbrella operator will contain migration-specific logic that can be removed in subsequent versions.

#### What survives the migration

| Resource | Behaviour |
|---|---|
| CRDs | Persist — OLMv0 never deletes CRDs |
| Operand processes (Authorino, Limitador) | Continue running — pods are never deleted during migration |
| User-created CRs (policies, Kuadrant CR) | Persist |
| AuthConfig CRs | Persist — Authorino continues reconciling them |
| Data plane (auth, rate limiting) | No disruption — operands keep serving traffic |

#### Control plane gap

There is a brief window per component (between operator scale-down and Helm Deployment apply) where no controller is watching that component's configuration. During this window:

- Existing policies continue to be enforced by the running operands
- New policy changes or CR updates are not reconciled until the replacement is in place
- No user action is required — the umbrella operator handles the entire sequence

### Steady-state Upgrades

OLM deploys the new umbrella operator image (containing updated charts and image references); the umbrella operator handles everything else.

1. OLM upgrades the umbrella operator bundle — this updates CRDs, ClusterRoles, and ClusterRoleBindings.
2. The umbrella operator re-renders charts, detects drift, and applies changes in order:
   - **Workloads** (Authorino, Limitador) and **dns-operator** — updated first, waited on until healthy.
   - **kuadrant-operator** last — as the policy reconciler, it may depend on new capabilities from the updated workloads.

### Version Pinning

Each umbrella operator release pins component versions via Helm values embedded at build time. Image references for all components are set through the charts' `values.yaml`, ensuring a single mechanism for version control across both install paths.

### Extension Operators

Extension operators (e.g., mcp-gateway-operator) complement kuadrant-operator rather than being consumed by it. They are **not managed by the umbrella operator in the initial implementation**.

Extension operators must independently remove their OLMv0 dependency declarations. The umbrella operator's architecture supports absorbing them as subcharts in a future iteration.

### Upgrade Testing

The full upgrade lifecycle must be validated end-to-end.

**Test environment:** OpenShift with OLMv0, Kuadrant installed via OLMv0 Subscriptions, a Gateway with HTTPRoute/AuthPolicy/RateLimitPolicy applied, and active ingress traffic.

A background process sends requests through the gateway continuously across all phases, recording success/failure rates.

| Phase | Action | Validation |
|---|---|---|
| 1. Baseline | Record OLMv0 state (Subscriptions, CSVs, CRDs, pods) | All operators running, policies enforced |
| 2. Migration | Install umbrella operator; wait for migration to complete | Only umbrella operator Subscription/CSV remains; all components running via Helm; data plane uninterrupted |
| 3. Steady-state upgrade | Publish new umbrella operator version; OLM upgrades | All Deployments updated; no CRD regressions; data plane uninterrupted |

### Failure Scenarios

- **Deployment rollout failure** — umbrella operator reports degraded status and blocks further upgrades.
- **Image pull failure** — surfaced via Deployment's ImagePullBackOff condition.
- **CRD conflict** — CRDs are in the OLM bundle. If a CRD already exists (e.g., standalone Authorino), OLM surfaces the conflict via status.
- **Migration interrupted** — idempotent steps resume on restart.

### Day 2 Considerations

The dependency operators currently handle day 2 operations that must have a home in the new architecture.

**PodDisruptionBudgets:** limitador-operator optionally creates a PDB for Limitador when `spec.pdb` is set (not created by default). authorino-operator does not manage a PDB. In the new architecture, users create PDBs directly if needed — no operator involvement required.

**Scaling replicas:** Both operators support configuring replicas via their CRs. In the new architecture, initial replica counts are set via Helm values at install time. Users can scale workloads directly via `kubectl scale` — since the umbrella operator owns the Deployments (not OLM), user-applied scaling is not reverted. The umbrella operator must respect user-set replica counts and not overwrite them on reconciliation.

**Resource requirements:** Helm charts set sensible defaults for resource requests/limits. Users can adjust them directly on the Deployments. As with replicas, the umbrella operator must not overwrite user-applied resource configuration on reconciliation.

**Storage backend changes:** limitador-operator supports switching between in-memory, Redis, Redis-cached, and disk at runtime via CR updates. In the new architecture, storage backend is configured via Helm values. Changing it requires a Helm upgrade or a Kuadrant CR update that triggers the umbrella operator to re-render. The deployment strategy implications (e.g., `Recreate` for disk storage) must be handled in the chart templates.

**TLS certificate management:** authorino-operator validates that TLS cert Secrets exist before proceeding with deployment. This preflight check must move to the Helm chart (e.g., via `required` in templates) or to the umbrella operator's reconciliation logic.

### Migration Risks

- **Finalizer stuck on `Authorino` or `Limitador` CR** — the dependency operators set finalizers on their CRs. During migration, the operator is scaled to 0 and cannot run its finalizer logic. If the umbrella operator fails to strip the finalizer (RBAC issue, API error), the CR hangs in `Terminating` state indefinitely, blocking migration progress. **Mitigation:** user manually removes the finalizer via `kubectl edit` or `kubectl patch`.

- **CSV deletion cascades to owned resources** — deleting a dependency operator's CSV on OLMv0 causes OLM to delete the resources that CSV owns (operator Deployment, RBAC, ServiceAccount). The migration flow defers CSV cleanup to the final step (step 8), after Helm-rendered workload resources are already in place and owned by the umbrella operator. The CSV cascade only affects the operator Deployments (authorino-operator, limitador-operator), which are already scaled to 0. However, if the migration steps are executed out of order — for example, if a CSV is deleted before ownerReferences are stripped and Helm resources applied — the cascade could remove resources that the workloads depend on. **Mitigation:** the umbrella operator must verify Helm-rendered resources are in place and owned before triggering CSV cleanup.

# Drawbacks
[drawbacks]: #drawbacks

- **New operator to build and maintain** — the umbrella operator introduces a new codebase, image, CI pipeline, and release process but in doing so removes two other operators
- **Broad RBAC** — the umbrella operator needs permissions to manage Deployments, Services, RBAC, and ConfigMaps across namespaces.
- **Two operators to debug** — users must distinguish umbrella operator issues (deployment) from kuadrant-operator issues (policy reconciliation).

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Why this design

- **Separation of concerns** — kuadrant-operator is simplified to pure policy reconciliation; deployment orchestration is cleanly separated into the umbrella operator.
- **Helm as first-class citizen** — both installation paths (direct Helm and umbrella operator) consume the same charts, unifying upstream and downstream experiences and preparing for a *.KS future.
- **Reduced operator footprint** — eliminating two operators reduces security surface area, maintenance burden, and release complexity.
- **Migration automation** — the umbrella operator can coordinate the transition from existing OLMv0 installs without manual intervention.
- **Proven pattern** — OpenShift's [cluster-olm-operator](https://github.com/openshift/cluster-olm-operator) uses Helm client-only rendering at scale in production.

### Alternatives considered

#### Large CSV

A single OLM bundle containing all existing operators in one CSV with zero dependency declarations. OLM owns the operator Deployments; the operators continue to manage operand Deployments via their CRs.

**Rejected because:**
- Dependency operators are retained — their maintenance cost, security surface area, and release complexity remain. Does not advance the architectural simplification.
- Migration for existing clusters is problematic — the OLMv1 design decisions doc states "if OLMv0 installs a dependency for you, it does not uninstall it when it is no longer depended upon." If correct (assumption — needs verification), upgrading to the big CSV would leave existing dependency Subscriptions/CSVs in place, resulting in duplicate operators. Manual cleanup would likely be required.
- OLM owns all Deployments in the CSV and reconciles them back to the declared state — users cannot scale replicas, adjust resources, or change workload configuration without rebuilding the bundle.
- All-or-nothing deployment — no path to selective deployment.
- Does not put Helm at the centre of installation.

#### Manual Installation of Dependencies

Each component published as its own independent OLM package with no dependency declarations. Users install each one separately.

**Rejected because:**
- Significant UX regression — users must create multiple Subscriptions or ClusterExtensions, each with its own configuration.
- Users are responsible for version compatibility and coordinated upgrades.
- Does not scale — adding new components means more manual steps.
- Same migration problem — users must manage the transition from dependency-based installation themselves.

# Prior art
[prior-art]: #prior-art

- **[cluster-olm-operator](https://github.com/openshift/cluster-olm-operator)** — primary reference. Manages OLMv1 sub-components using Helm charts rendered client-only, with static resource controllers and deployment controllers for applying the rendered manifests.
- **[OLMv0 dependency resolution](https://olm.operatorframework.io/docs/concepts/olm-architecture/dependency-resolution/)** — the current model being replaced. Resolves dependencies automatically via CRD-based declarations.
- **[OLMv1 design decisions](https://github.com/operator-framework/operator-controller/blob/main/docs/project/olmv1_design_decisions.md)** — documents why dependency resolution was removed and endorses simpler packaging for Level 1/2 operators.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- **OLMv0 → OLMv1 cluster transition** — the exact process for migrating a cluster from OLMv0 to OLMv1 is not documented. This affects all operators, not just Kuadrant. The umbrella operator approach ensures Kuadrant is ready regardless of how this transition works — zero dependency declarations, single Subscription/ClusterExtension.

- **OLMv0 behaviour when a CSV drops dependency declarations** — when the kuadrant-operator CSV is upgraded to a version that removes its `olm.package.required` declarations, what happens to the dependency Subscriptions/CSVs that OLMv0 auto-installed? The OLMv1 design doc states they are not automatically removed, but this needs verification.

- **CRD ownership conflicts with standalone installations** — if standalone Authorino is already installed on a cluster, the umbrella operator's bundle includes Authorino CRDs. Under OLMv1's single-ownership model, two bundles cannot own the same CRDs.

# Future possibilities
[future-possibilities]: #future-possibilities

- **Selective component deployment** — the umbrella operator can conditionally render only the charts for components the user needs. Extension operators like mcp-gateway-operator would be natural candidates for opt-in deployment. Safety checks could prevent removing components with active policy CRs.

- **Absorbing extension operators** — mcp-gateway-operator and future extensions can be folded into the umbrella operator as subcharts, extending the single-action install to cover the full Kuadrant ecosystem.

- **Kuadrant CR evolution** — with the umbrella operator owning operand configuration, the Kuadrant CR could evolve to expose selective deployment toggles (e.g., `spec.components.rateLimiting.enabled`) without requiring kuadrant-operator changes.
