# RFC: OLM Dependency Model Transition

- Feature Name: `olm_dependency_model_transition`
- Start Date: 2026-03-30
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/issues/0000)

# Summary
[summary]: #summary

Replace the OLMv0 dependency model (which automatically installs Authorino, Limitador, and DNS operators) with a runtime approach where the Kuadrant operator creates and manages OLMv1 ClusterExtension CRs for its dependencies. This preserves the single-action install experience while aligning with OLMv1's design, which explicitly removes automatic dependency installation.

# Motivation
[motivation]: #motivation

The OLM (Operator Lifecycle Manager) dependency model is being deprecated as part of the transition to OLMv1 (ClusterExtensions). OLMv1 explicitly will not automatically install missing dependencies when a user requests an operator. The [OLMv1 design decisions](https://github.com/operator-framework/operator-controller/blob/main/docs/project/olmv1_design_decisions.md) state that it "will err on the side of predictability and cluster-administrator awareness."

Today, Kuadrant ships as a single OLM catalog that bundles four operators:

- **kuadrant-operator** (the main operator)
- **authorino-operator** (authentication/authorization)
- **limitador-operator** (rate limiting)
- **dns-operator** (DNS management)

Users add one CatalogSource, create one Subscription, and OLM's dependency resolution handles installing all four operators automatically. When the OLMv1 dependency model is removed, this single-action install experience breaks.

The target timeline for this transition is **end of 2026**.

### Requirements

1. **Preserve the single-action install experience** - users should not need to manually install four plus separate operators.
2. **Kuadrant owns the full lifecycle of its dependencies** - the Kuadrant operator manages versions, installation, upgrades, and removal of Authorino, Limitador, and DNS operators. Cluster admins do not independently upgrade these components.
3. **Targets OLMv1** - the solution needs to work with OLMv1 (ClusterExtensions API), which can be installed on both OpenShift and vanilla Kubernetes.
4. **Dependency operators remain Kuadrant-exclusive** - Authorino, Limitador, and DNS operators are already exclusively owned by Kuadrant. This requirement is maintained.
5. **Allows for selective dependency installation** - The number of dependencies is likely to grow as more policies are added and the extension model matures. Users at some point are likely to want/need to be able to choose which dependencies to deploy based on the policies they want to use.
6. **No independent installation of Kuadrant components** - When using OLMv1 to install Kuadrant, it is expected that there is no independent installation of Kuadrant components (e.g., Authorino) on the cluster. Users who currently install Authorino independently for standalone use should transition to using Kuadrant with selective profiles to get just the capabilities they need (e.g., AuthPolicy only). Note: OLMv1 allows two ClusterExtensions to share the same CRD **if the CRD content is identical**, but relying on this for coexistence introduces version alignment complexity that is best avoided.


### OLMv1 Context

With OLMv1, installing an operator requires the user to explicitly create several resources ([getting started guide](https://operator-framework.github.io/operator-controller/getting-started/olmv1_getting_started/)):

1. **ClusterCatalog** — points to an image registry containing operator bundles
2. **Namespace** — where the operator will run
3. **ServiceAccount** — OLMv1 impersonates this SA to install bundle contents
4. **ClusterRoles/Roles + Bindings** — grant the SA permissions for everything the operator needs to create
5. **ClusterExtension** — the install intent, referencing the SA and catalog

Without dependency management, a user installing the full Kuadrant stack would need to repeat steps 2-5 for each of the four operators (~20+ resources manually). The goal is to reduce this to a single ClusterExtension install for Kuadrant, with the operator managing the rest.

### OLMv1 Design Decisions Relevant to Kuadrant

Reference: [OLMv1 Design Decisions](https://github.com/operator-framework/operator-controller/blob/main/docs/project/olmv1_design_decisions.md)

| Decision | Impact on Kuadrant |
|---|---|
| **No automatic dependency installation** | The core reason we need this work — OLMv1 won't install Authorino/Limitador/DNS for us |
| **No cluster-admin permissions** | OLMv1 uses user-provided ServiceAccounts to install bundles. We need to provision these SAs with correct RBAC |
| **Single extension ownership** | Each managed object is owned by exactly one ClusterExtension. Our ClusterRoles are owned by Kuadrant but *referenced* by dependency ClusterExtensions via operator-created bindings — this should be fine |
| **Flexible bundle contents** | Bundles can contain arbitrary resources (ClusterRoles, etc.), confirming we can ship dependency installer ClusterRoles in the Kuadrant bundle |
| **Fine-grained version control** | Admins can pin versions per operator. Our operator can set specific versions in the ClusterExtension CRs it creates |
| **Constraint validation, not resolution** | OLMv1 will check if constraints are met but won't act on them. Reinforces that we must handle installation ourselves |
| **GitOps-friendly APIs** | ClusterExtension is declarative and eventually-consistent, which fits well with the operator creating/reconciling these CRs |
| **Per-operator upgrade policies** | Each dependency ClusterExtension can have its own upgrade policy, giving us fine-grained control |
| **Shared CRDs allowed if identical** | Two ClusterExtensions can share the same CRD if the content is identical. We avoid relying on this by requiring no independent installation of Kuadrant components alongside Kuadrant |

### Out of Scope

- Non-OLM environments (e.g., Helm installs on vanilla Kubernetes) are out of scope for this proposal. Existing Helm tooling will continue to handle dependency installation in those environments.

- Implementing any selective dependency deployment at this stage.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### User Install Workflow

1. **Create a ClusterCatalog** pointing to the Kuadrant catalog image (contains all operators).
2. **Create the Kuadrant ClusterExtension** with its namespace, ServiceAccount, and RBAC (this is the only ClusterExtension the user creates).
3. **Create the Kuadrant CR** — the operator detects OLMv1, creates ServiceAccounts, ClusterRoleBindings, and ClusterExtension CRs for each required dependency in the Kuadrant CR's namespace.
4. **Dependencies are installed automatically** — OLMv1 processes the dependency ClusterExtensions using the operator-created ServiceAccounts and the pre-provisioned ClusterRoles.
5. **Operator reports readiness** — once all dependencies are running, the Kuadrant CR status reflects a healthy state.

To uninstall, the user deletes the Kuadrant CR (operator cleans up dependency ClusterExtensions, ServiceAccounts, and ClusterRoleBindings) and then deletes the Kuadrant ClusterExtension.

### Upgrade Experience

When the user upgrades Kuadrant (by updating the version on the Kuadrant ClusterExtension), the new operator version automatically upgrades its dependencies to compatible versions. The user does not need to manually upgrade dependency operators. The Kuadrant CR status reports progress and surfaces any failures during the upgrade.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### OLMv1 Detection

The operator must auto-detect whether OLMv1 is present on the cluster before attempting to manage ClusterExtension CRs. This follows the existing pattern used for other dependencies — the operator already detects CRD availability at startup (e.g., Istio, Envoy Gateway, cert-manager CRDs in `internal/controller/state_of_the_world.go`). The same approach would be used to check for the `clusterextensions.olm.operatorframework.io` CRD. If OLMv1 is not installed, the dependency management reconciler is not registered and the operator falls back to the current behaviour of reporting missing dependencies via status conditions.

### RBAC and ServiceAccount Provisioning

The provisioning is split between build time and runtime:

- **Build time (bundle):** ClusterRoles defining the permissions each dependency operator needs are shipped as static resources in the Kuadrant bundle. These are cluster-scoped and namespace-independent.
- **Runtime (operator):** ServiceAccounts, ClusterRoleBindings, and ClusterExtension CRs are created by the Kuadrant operator in the same namespace as the Kuadrant CR.

Example resources the operator would create at runtime:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: authorino-installer
  namespace: kuadrant-system  # same namespace as the Kuadrant CR
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: authorino-installer-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: authorino-installer-role  # pre-provisioned in bundle
subjects:
- kind: ServiceAccount
  name: authorino-installer
  namespace: kuadrant-system
---
apiVersion: olm.operatorframework.io/v1
kind: ClusterExtension
metadata:
  name: authorino-operator
spec:
  namespace: kuadrant-system
  serviceAccount:
    name: authorino-installer
  source:
    sourceType: Catalog
    catalog:
      packageName: authorino-operator
      version: 0.5.0
```

### Selective Installation via Profiles

A `spec.profiles` field on the Kuadrant CR could drive selective installation — the operator would only create ClusterExtensions for dependencies required by the selected profiles. The exact profile taxonomy (policy-based, capability-based, etc.). This is purely an example to provoke thought and give wider context. Implementation of this is out of scope for this RFC.

### Upgrade Flow

When Kuadrant itself is upgraded (via its own ClusterExtension), the new version of the operator may require updated versions of its dependencies. The upgrade flow would be:

1. **Kuadrant's ClusterExtension is upgraded** — OLMv1 deploys the new Kuadrant operator image. The new operator binary contains the updated dependency version mappings (e.g., Authorino 0.5.0 → 0.6.0).
2. **Operator reconciles the Kuadrant CR** — detects that the dependency ClusterExtension CRs specify outdated versions.
3. **Operator updates dependency ClusterExtension CRs** — patches the `spec.source.catalog.version` field on each dependency ClusterExtension to the required version.
4. **OLMv1 processes the updated ClusterExtensions** — upgrades each dependency operator using its pre-provisioned ServiceAccount and RBAC.
5. **Operator reports status** — once dependencies are running at the expected versions, the Kuadrant CR status reflects readiness.

### Upgrade Ordering

Dependencies must be upgraded before the Kuadrant operator relies on any new dependency features. Since the Kuadrant operator drives the upgrade by updating the ClusterExtension CRs, it can enforce ordering:

- Update dependency ClusterExtension CRs first
- Wait for each dependency ClusterExtension to report a healthy/installed status
- Only then proceed with its own reconciliation that depends on new dependency features

The new Kuadrant operator **must** be backwards-compatible with the previous dependency versions. There is an unavoidable window between the Kuadrant operator starting (step 1) and dependencies finishing their upgrade (step 4) where the old dependency versions are still running. If the operator is not backwards-compatible with the old versions during this window, it will fail. This backwards-compatibility requirement should be enforced as part of the release process and testing.

### Failure Scenarios

- **Dependency upgrade fails** — the ClusterExtension for a dependency reports a failed state. The Kuadrant operator should surface this in the Kuadrant CR status conditions and not proceed with reconciliation that depends on the failed component.
- **Catalog unavailable** — if the catalog containing dependency bundles is unreachable, OLMv1 cannot process the ClusterExtension update. The operator should report this as a degraded condition.
- **Version not found in catalog** — the requested dependency version does not exist in the catalog. The operator should report this clearly in status.
- **Rollback** — OLMv1 does not provide automatic rollback. If a dependency upgrade fails, the cluster admin would need to manually intervene by either fixing the issue or reverting the Kuadrant ClusterExtension to the previous version, which would cause the operator to reconcile the dependency versions back to their previous values.

### Version Pinning

Each Kuadrant operator release would embed a compatibility matrix mapping its own version to the required dependency versions. This could be a build-time constant or configuration baked into the operator image, ensuring that a given Kuadrant version always deploys a known-good set of dependencies.

### Security: RBAC Escalation Prevention

Kubernetes prevents a ServiceAccount from creating ClusterRoleBindings that reference ClusterRoles with broader permissions than the SA itself holds, unless the SA has the `bind` verb on those specific ClusterRoles.

The Kuadrant operator creates ServiceAccounts and ClusterRoleBindings at runtime for each dependency. The ClusterRoles themselves are pre-provisioned as static resources in the Kuadrant bundle at build time with known, fixed names. This allows the Kuadrant operator's own RBAC to be scoped narrowly using `resourceNames`:

```yaml
- apiGroups: [rbac.authorization.k8s.io]
  resources: [clusterroles]
  verbs: [bind]
  resourceNames:
  - authorino-installer-role
  - limitador-installer-role
  - dns-installer-role
```

This means the Kuadrant operator:
- **Can** bind ServiceAccounts to the pre-defined dependency installer ClusterRoles
- **Cannot** bind arbitrary ClusterRoles or escalate beyond the known dependency roles
- **Does not** need the actual permissions of the dependency operators on its own ServiceAccount

# Drawbacks
[drawbacks]: #drawbacks

- Adds runtime complexity to the Kuadrant operator — a new reconciler for ClusterExtension lifecycle management, with associated failure modes around catalog availability and upgrade orchestration.
- The Kuadrant operator takes on responsibility for managing OLMv1-specific resources, coupling it more tightly to the OLM ecosystem.
- Requires the Kuadrant operator's ServiceAccount to have `bind` permissions on dependency ClusterRoles, which is a sensitive (though narrowly scoped) privilege.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Why this design

- Aligns with OLMv1's model where each operator is a separate ClusterExtension
- Preserves the single-action install experience that users have today
- Supports selective dependency installation as the number of dependencies grows
- The Kuadrant operator already has the pattern for CRD detection and conditional reconciler registration, making this a natural extension
- RBAC is narrowly scoped using pre-provisioned ClusterRoles with `bind` by `resourceNames`

### Alternatives considered

**Embedding dependency operator deployments in the Kuadrant bundle** — all four operator Deployments (and their CRDs) would be bundled into a single OLMv1 bundle. One ClusterExtension installs everything.

Pros:
- Single ClusterExtension on the cluster — no additional ClusterExtensions created at runtime
- No additional RBAC beyond what the operators already need
- No dependency on external catalogs for dependency operator images
- All components are versioned and tested together as a single unit

Cons:
- Larger bundle size
- Tighter coupling between Kuadrant and its dependency operators at build time
- More complex build pipeline — must pull in and package multiple operator binaries/images
- Upgrading a single dependency requires rebuilding and releasing the entire bundle
- May conflict with OLMv1's ownership model if dependency CRDs are also published independently
- All-or-nothing — cannot support selective dependency installation as the number of policies and dependencies grows
- No operator-level visibility into individual dependency failures — if an embedded dependency deployment fails, the Kuadrant operator has no ability to detect, report on, or recover from the failure since OLMv1 manages all deployments as a single unit

**Reason not chosen:** The selective dependency installation requirement rules out this approach. As the number of Kuadrant policies and dependencies grows, users need the ability to install only what they need. This approach bundles everything together with no way to opt out of individual components.

**Embedding ClusterExtension CRs in the Kuadrant bundle** — OLMv1's ownership model requires each managed object to be owned by exactly one extension. Nested ClusterExtension installation is not a supported pattern, and the ServiceAccount scoping would require cluster-admin level privileges.

**Embedding Subscription CRs in the bundle** — Subscriptions are an OLMv0 concept that does not exist in OLMv1. In OLMv0, Subscriptions are not a supported bundle resource type. This approach has no path forward.

### Impact of not doing this

Without this change, Kuadrant users on OLMv1 would need to manually create ~20+ resources to install the full stack. This significantly degrades the install experience and increases the barrier to adoption.

# Prior art
[prior-art]: #prior-art

- **OLMv0 dependency resolution** — the current model being replaced. OLMv0 resolves dependencies automatically via CRD-based dependency declarations in `bundle/metadata/dependencies.yaml`. This worked well but is being removed in OLMv1. [OLM Dependency Resolution](https://olm.operatorframework.io/docs/concepts/olm-architecture/dependency-resolution/)
- **ArgoCD OLMv1 sample** — the [operator-controller ArgoCD sample](https://github.com/operator-framework/operator-controller/blob/main/config/samples/olm_v1_clusterextension.yaml) demonstrates the full set of resources required to install an operator via OLMv1, including ServiceAccount, RBAC, and ClusterExtension.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What is the exact profile taxonomy? Should profiles map to policy names (`AuthPolicy`, `DNSPolicy`), capability names (`auth`, `dns`, `ratelimiting`), or something else?
- What are the exact ClusterRole definitions needed for each dependency installer? These need to be derived from each dependency operator's actual RBAC requirements.
- How should the operator handle the case where a user changes profiles on an existing Kuadrant CR (e.g., removing a profile that has an active dependency)?
- What is the exact target OCP version for OLMv1 GA?
- If a cluster already has an independent Authorino installation, what is the migration path for users transitioning to Kuadrant-managed Authorino?

# Future possibilities
[future-possibilities]: #future-possibilities

- **Selective profiles** — once the profiles mechanism is implemented, it could be extended to cover Kuadrant extensions (OIDCPolicy, PlanPolicy, TelemetryPolicy) as they mature, allowing users to opt in to specific extension operators.
- **Health monitoring** — the dependency management reconciler could provide richer health monitoring by watching ClusterExtension status conditions and surfacing detailed diagnostics in the Kuadrant CR status.
- **Cross-cluster dependency management** — in multi-cluster scenarios, the dependency management pattern could be extended to ensure consistent dependency versions across clusters.
