# RFC: Kuadrant Operator Consolidation

- Feature Name: `kuadrant_operator_consolidation`
- Start Date: 2026-06-11
- RFC PR: [Kuadrant/architecture#189](https://github.com/Kuadrant/architecture/pull/189)
- Issue tracking: [Kuadrant/architecture#164](https://github.com/Kuadrant/architecture/issues/164)

# Summary
[summary]: #summary

Consolidate the Kuadrant ecosystem into a single operator. The kuadrant-operator manages all component controllers (authorino-operator, limitador-operator, dns-operator, mcp-gateway), removing the need for separate OLM packages per component. Users install one operator via Helm or OLM and get a fully functional Kuadrant deployment.

# Goals

1. Prepare for OLMv1 by removing OLM operator dependency declarations (`dependencies.yaml`). All changes are implemented and tested on OLMv0
2. Deploy all component controllers (authorino-operator, limitador-operator, dns-operator, mcp-gateway) from the kuadrant-operator at runtime
3. Support two installation methods from a single operator: OLM (for OpenShift) and Helm (for any Kubernetes distribution). Maintain a valid Helm chart for kuadrant-operator as a first-class installation method
4. Bring mcp-gateway into the Kuadrant ecosystem as a managed component controller
5. Reduce the number of OLM packages from 4 to 1, simplifying the release and catalog pipeline
6. Maintain a single-action install experience for users
7. No deprecations: all existing CRDs, Deployments, and resources remain. The same controllers and workloads run on the cluster as before

# Non-goals

1. **Removing wrapper CRs (Authorino CR, Limitador CR)**: these are preserved. Deferred to a follow-on RFC
2. **Removing child operator controllers**: authorino-operator and limitador-operator continue to run as Deployments. Deferred to a follow-on RFC
3. **Migrating to OLMv1**: all changes run on OLMv0. OLMv1 compatibility is a readiness outcome, not a migration step
4. **Refactoring component Helm charts**: upstream charts are consumed as-is. Improved templating is future work
5. **Conditional component deployment**: all components are always deployed. Optional component installation is future work

# Motivation
[motivation]: #motivation

## OLM dependency resolution is being removed

OLMv1 [explicitly removes automatic dependency installation](https://github.com/operator-framework/operator-controller/blob/main/docs/project/olmv1_design_decisions.md). The [design decisions document](https://github.com/operator-framework/operator-controller/blob/main/docs/project/olmv1_design_decisions.md) states:

> OLM v1 will not automatically install a missing dependency when a user requests an operator. The primary reasoning is that OLM v1 will err on the side of predictability and cluster-administrator awareness.

Bundles declaring `olm.package.required`, `olm.gvk.required`, or `olm.constraint` will be rejected under OLMv1. Kuadrant's current model relies entirely on these declarations. This has been verified on OpenShift CRC: attempting to install the current kuadrant-operator via OLMv1 `ClusterExtension` fails with `bundle has a dependency declared via property "olm.package.required" which is currently not supported`.

The exact process for transitioning a cluster (fully) from OLMv0 to OLMv1 is not yet documented. Currently on OpenShift OLMv1 and OLMv0 run alongside each other. However, regardless of when or how that transition happens, the dependency resolution mechanism Kuadrant relies on will not exist in the future. We should eliminate our dependency declarations now, while still on OLMv0, so that Kuadrant is already compatible when the transition arrives.

## Per-component OLM packaging adds overhead within a Kuadrant deployment

Authorino, Limitador, and mcp-gateway are independent projects with standalone use cases outside Kuadrant. Their operators were created in part to support independent installation and lifecycle management. However, within a Kuadrant deployment, they function as internal components whose lifecycle is tied to the Kuadrant CR. The per-component OLM packaging (separate CSV, Subscription, bundle image, catalog entry, release pipeline) adds overhead that is only justified when the components are deployed independently.

This RFC consolidates the Kuadrant-as-a-product installation path. The individual component repos and their Helm charts remain available for standalone installation. Nothing prevents a user from installing just Limitador or just Authorino directly from their own repos.

## Kuadrant needs to run beyond OpenShift

Kuadrant's deployment targets are not limited to OpenShift. In the future, we want to support any Kubernetes distribution (EKS, GKE, AKS, and others). An architecture that depends on OLM for installation and lifecycle management ties Kuadrant to an OpenShift-specific mechanism that doesn't exist on these platforms. Helm is the common denominator. Making Helm the primary installation method ensures Kuadrant is a first-class citizen on any Kubernetes distribution.

## Reduced release and maintenance overhead

Each independent OLM operator requires its own bundle image, catalog entry, release pipeline, and version coordination. Consolidating into a single OLM package eliminates per-component release overhead while keeping the same controllers and workloads running on the cluster. No CRDs, Deployments, or other resources are removed or deprecated as part of this work.

## Established pattern for OLMv1 readiness

The meta-operator pattern (a parent operator deploying child components at runtime) is the approach highlighted by the OLM team for operators that previously relied on `olm.package.required` dependencies. It is already used in production by Red Hat OpenShift AI ([opendatahub-operator/rhods-operator](https://github.com/opendatahub-io/opendatahub-operator)) for 18+ components, and by the [cluster-olm-operator](https://github.com/openshift/cluster-olm-operator) which manages OLMv1's own sub-components using the same Helm SDK client-only rendering. The approach described in this RFC follows the same pattern, with the only difference being that charts are embedded in the operator image at build time rather than extracted from component images via init containers at runtime.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The kuadrant-operator becomes an umbrella operator that deploys component controllers at runtime. Wrapper CRs (Authorino CR, Limitador CR) are preserved. No breaking changes for users.

## Before

4 OLM operators (kuadrant, authorino, limitador, dns), each with its own CSV, bundle image, and catalog entry. OLM manages all lifecycles independently.

## After

1 OLM operator (kuadrant-operator) with a single CSV, bundle, and catalog entry. Component controllers (authorino-operator, limitador-operator, dns-operator, mcp-gateway) are no longer OLM packages. They become internal controller deployments whose lifecycle is managed by the kuadrant-operator at runtime via Helm chart rendering. No component controller bundles, catalogs, or CSVs need to be maintained.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

For visual diagrams covering the build-time chart sync, runtime reconciliation chain, RBAC model, and resource ownership, see [Architecture Diagrams](0019-olmv1-operator-consolidation/architecture-diagrams.md).

## Chart sync process

Component Helm charts are sourced from upstream repos (authorino-operator, limitador-operator, dns-operator, mcp-gateway). A sync tool downloads each chart as-is and places it in the operator repo, driven by a config file (`component-charts/sync.yaml`) that declares which upstream repos and branches to track:

```text
component-charts/
├── sync.yaml               # Component tracking config
├── authorino-operator/     # Complete upstream chart
├── limitador-operator/     # Complete upstream chart
├── dns-operator/           # Complete upstream chart
└── mcp-gateway/            # Complete upstream chart
```

Charts are copied as-is from upstream with no rendering, splitting, or modification. The complete chart (Chart.yaml, values.yaml, templates/, crds/) is preserved so the operator can render it at runtime exactly as it would be used in a standalone Helm installation. The sync tool resolves the tracked branch to a commit SHA and pins it in `sync.yaml` for reproducibility.

Synced charts are committed to the repo. The sync is run manually via `make sync-component-charts` or triggered automatically via `repository_dispatch` from component repos. Charts are also packaged into the operator container image for runtime access.

## Resource ownership

All child operator resources are managed by the kuadrant-operator at runtime. The only resources managed by the installer (OLM or Helm) are the kuadrant-operator's own resources:

**Managed by the installer (Helm chart or OLM bundle):**

| Resource | Notes |
|---|---|
| kuadrant-operator CRDs | Kuadrant, AuthPolicy, RateLimitPolicy, DNSPolicy, TLSPolicy, TokenRateLimitPolicy |
| kuadrant-operator Deployment | The operator itself |
| kuadrant-operator ServiceAccount, ClusterRole, ClusterRoleBinding | Operator's own RBAC |

**Managed by the kuadrant-operator at runtime (on startup):**

| Resource | Notes |
|---|---|
| Component CRDs | AuthConfig, Authorino, Limitador, DNSRecord, DNSHealthCheckProbe, MCPGatewayExtension, MCPServerRegistration, MCPVirtualServer |
| Component ClusterRoles | All child operator ClusterRoles |
| Component controller Deployments | authorino-operator, limitador-operator, dns-operator, mcp-gateway |
| Component ServiceAccounts | Per-component identity |
| Component ClusterRoleBindings | Binds component SAs to component ClusterRoles |
| Component Roles, RoleBindings | Namespace-scoped RBAC (e.g. leader election) |
| Component Services, ConfigMaps | Metrics, configuration |

**Managed by the kuadrant-operator at runtime (triggered by Kuadrant CR creation):**

| Resource | Notes |
|---|---|
| Wrapper CRs | Authorino CR, Limitador CR (reconciled by child operators into workloads) |

Child operator CRDs are not included in the OLM bundle. This is a deliberate choice that eliminates GVK conflicts during OLM upgrades from the current multi-operator installation (see [Migration](#migration-from-multi-operator-olm-installation)).

## Runtime rendering

On startup, before the controller manager begins watching resources, the kuadrant-operator renders each component controller's Helm chart from `/charts/<name>/` in the container image using the Helm Go SDK (`ClientOnly=true`, `DryRun=true`). This is the same pattern used by OpenShift's [cluster-olm-operator](https://github.com/openshift/cluster-olm-operator) and [opendatahub-operator](https://github.com/opendatahub-io/opendatahub-operator) in production.

Charts are rendered and applied following Helm's install ordering (CRDs first, then dependent resources). All child operator resources must be established before the controller manager starts watching for CRDs.

Component controllers are deployed into the kuadrant-operator's namespace (e.g. `kuadrant-system`). They are control plane singletons, one instance per cluster, not per Kuadrant CR. This matches the current model where OLM deploys all child operators into the same namespace.

Component controller Deployments are not owned by the Kuadrant CR. They run independently of whether a Kuadrant CR exists. The Kuadrant CR only triggers data plane resources (wrapper CRs, policies).

## RBAC

The kuadrant-operator requires permissions to create and manage all child operator resources at runtime:

- **`apiextensions.k8s.io` CRDs**: unscoped `create` (required because the resource may not exist yet), plus `get`, `list`, `watch`, `update`, `patch` scoped to specific child CRD names via `resourceNames`. This ensures the operator can only modify CRDs it manages while limiting the broad permission to `create` only
- **`rbac.authorization.k8s.io` ClusterRoles**: unscoped `create` (same reason as CRDs), plus `get`, `list`, `watch`, `update`, `patch`, `bind`, `escalate` scoped to specific child ClusterRole names via `resourceNames`
- **`rbac.authorization.k8s.io` ClusterRoleBindings, Roles, RoleBindings**: standard CRUD to bind component SAs to their ClusterRoles
- **`apps` Deployments, core resources**: standard CRUD for component controllers

The `escalate` verb on ClusterRoles is required because child operator ClusterRoles grant permissions (e.g. access to AuthConfig resources) that the kuadrant-operator itself does not hold. Without `escalate`, Kubernetes RBAC prevents creating a ClusterRole with rules the creator doesn't already have. The `bind` verb on ClusterRoles is also required because the charts render ClusterRoleBindings that reference those ClusterRoles. Without `bind`, Kubernetes RBAC prevents creating a binding to a ClusterRole whose permissions you don't hold.

## Two supported installation methods

- **Helm**: Single `helm install` deploys the kuadrant-operator and its own CRDs. The operator starts and deploys all child operator resources (CRDs, ClusterRoles, controllers) at runtime from embedded charts.
- **OLM**: Single operator in the catalog. The bundle contains only the kuadrant-operator's own CRDs and Deployment. No child operator CRDs or ClusterRoles in the bundle. The operator deploys them at runtime, same as the Helm path.

Both paths produce the same cluster state: kuadrant-operator running, child operator CRDs established, all component controllers deployed and running. Child operator resources are managed by the operator at runtime identically in both paths.

## Image references

OLM convention requires all container images to be declared in the CSV's `relatedImages` section and as `RELATED_IMAGE_*` environment variables on the operator Deployment. This is used for image mirroring in disconnected environments, version pinning at release time, and downstream build system automation. The consolidated kuadrant-operator CSV must include all component images that were previously declared in individual operator CSVs:

| Env var | Default image | Purpose |
|---|---|---|
| `RELATED_IMAGE_AUTHORINO_OPERATOR` | `quay.io/kuadrant/authorino-operator:latest` | Authorino operator controller |
| `RELATED_IMAGE_AUTHORINO` | `quay.io/kuadrant/authorino:latest` | Authorino workload |
| `RELATED_IMAGE_LIMITADOR_OPERATOR` | `quay.io/kuadrant/limitador-operator:latest` | Limitador operator controller |
| `RELATED_IMAGE_LIMITADOR` | `quay.io/kuadrant/limitador:latest` | Limitador workload |
| `RELATED_IMAGE_DNS_OPERATOR` | `quay.io/kuadrant/dns-operator:latest` | DNS operator controller |
| `RELATED_IMAGE_MCP_GATEWAY` | `ghcr.io/kuadrant/mcp-controller:latest` | MCP Gateway controller |

These are declared as environment variables on the kuadrant-operator Deployment (`config/manager/manager.yaml`). The downstream build system overrides them with version-pinned or mirrored image references. The Helm rendering reads these env vars and applies the correct images when rendering child operator charts, either by patching the rendered Deployment image directly (for simple charts that don't support values-based overrides) or by passing Helm values (for charts with configurable image fields).

## Wrapper CRs preserved

Authorino CR and Limitador CR are still created by kuadrant-operator when a Kuadrant CR is created. Component controllers reconcile these wrapper CRs to create workloads. No change to this flow. Users see no difference in behaviour.

## OLM bundle and catalog

The bundle/catalog pipeline deals with a single package (kuadrant-operator). Component controller bundle images, catalog channel entries, and dependency injection are removed. The bundle contains only the kuadrant-operator's own CRDs and resources. Child operator CRDs are not included in the bundle; they are applied at runtime by the operator. This eliminates OLM GVK conflicts during upgrades from the current multi-operator installation.

## Component repos

No changes are required in component repos for the consolidation to work. The sync tool downloads charts as-is.

However, some upstream chart changes would be beneficial:

- **Consistent labels**: applying consistent labels across all component resources (e.g. `kuadrant.io/component`) would improve topology tracking and filtering. Currently, the Authorino Deployment lacks distinguishing labels, preventing it from being tracked in the kuadrant-operator topology
- **Unique Deployment selectors**: the limitador-operator chart uses a generic `control-plane: controller-manager` selector that collides with the kuadrant-operator in the same namespace. This is a pre-existing bug that needs fixing upstream
- **Fixed ClusterRole names**: charts using Helm's `fullname` helper for ClusterRole names create a fragile dependency on the release name. ClusterRoles are cluster-scoped permission definitions and should use fixed names
- **Graceful CRD handling**: component controllers should handle missing CRDs gracefully (e.g. mcp-gateway currently crashes if Istio EnvoyFilter CRD is not installed)

Since component repos no longer need to produce their own OLM bundles or catalogs, OLM-specific artefacts can also be removed to reduce maintenance burden: `bundle/`, `catalog/`, `config/manifests/`, `config/deploy/olm/`, `config/scorecard/`, `bundle.Dockerfile`, `operator-sdk` and `opm` tool dependencies.

## Automated component sync

A Go CLI tool (`hack/sync-components/`) manages chart syncing, driven by a config file (`component-charts/sync.yaml`) that declares each component's upstream repo, chart path, tracked branch, and pinned commit SHA (`ref`). The tool resolves the tracked branch to its latest commit SHA, downloads the chart at that SHA, and updates only the `ref` field in the config file. The `tracked-branch` field remains unchanged and continues to identify which upstream branch to follow.

When a component repo pushes to its tracked branch, a `repository_dispatch` event notifies the kuadrant-operator. A GitHub Actions workflow receives the event, determines which kuadrant-operator branches track that component (currently the default branch only), runs the sync tool, and creates or updates a PR per target branch and component (e.g. `sync/main/dns-operator`).

Release branch sync support is deferred until the branch mapping strategy for releases is defined. Changes to the release process and version pinning strategy are out of scope for this RFC and will be addressed separately.

## Migration from multi-operator OLM installation

### How it works

Since child operator CRDs are not included in the OLM bundle, there are no GVK conflicts between the consolidated kuadrant-operator and the existing child operator CSVs. The OLM upgrade proceeds without stalling.

On startup, the upgraded kuadrant-operator:

1. Applies child operator CRDs (which already exist from the previous installation)
2. Deploys child operator controllers from embedded Helm charts
3. Detects and removes orphaned child operator Subscriptions and CSVs programmatically

The data plane (Authorino server, Limitador server) is unaffected throughout. Verified on OpenShift CRC: deleting child operator CSVs removes their controller Deployments but does not cascade-delete CRDs, ClusterRoles, or data plane workloads. The consolidated operator's child controllers take over seamlessly.

### OLMv1 compatibility

The consolidated bundle has been tested with OLMv1 `ClusterExtension` on OpenShift CRC and installs successfully. No dependency declarations, no GVK conflicts. This is in contrast to the current multi-operator bundle which is explicitly rejected by OLMv1: `bundle has a dependency declared via property "olm.package.required" which is currently not supported`.

### Upgrade testing

Automated upgrade tests must be established to validate the migration path from the current multi-operator OLM installation to the consolidated operator. These tests should:

- Start with the previous release installed via OLM (all 4 operators with their CSVs)
- Create a Kuadrant CR with active policies and ingress traffic
- Upgrade to the consolidated release via OLM catalog channel
- Verify child operator Subscriptions and CSVs are cleaned up by the operator
- Verify all child CRDs survived the migration
- Verify all component controllers are running (now managed by kuadrant-operator)
- Verify zero data plane disruption throughout (policies continue to be enforced, no traffic loss)

These tests should run in CI for every release and serve as ongoing regression testing for the OLM upgrade path.

# Future possibilities
[future-possibilities]: #future-possibilities

This RFC deliberately preserves wrapper CRs and child operator controllers. A follow-on RFC should address further architectural simplification:

- **Removing intermediate operator layers and wrapper CRs**: deploying Authorino and Limitador workloads directly from the kuadrant-operator, eliminating authorino-operator and limitador-operator as running controllers. This removes the `Authorino` and `Limitador` CRDs from the user-facing API surface. Each operator removed is a separate container image with its own Go dependency tree. Every transitive dependency is a potential CVE that needs scanning, patching, and releasing. Removing these controllers reduces the security surface area and maintenance burden.
- **Control plane status CR**: a new cluster-scoped CR (e.g. `KuadrantControlPlane` or `KuadrantPlatform`, separate from the Kuadrant CR) for reporting child operator health and eventually enabling conditional component deployment. The operator would auto-create a default on startup. This separates control plane management (which components are deployed, their health) from data plane management (the Kuadrant CR, policies, workloads). This is the same pattern used by RHOAI/RHODS, where `DSCInitialization` handles platform setup and `DataScienceCluster` controls component lifecycle.
- **Conditional component deployment**: using the control plane CR to allow operators to be selectively enabled or disabled, avoiding deploying components that are not needed (e.g. mcp-gateway on clusters without Istio).
- **Extracting a dedicated policy controller**: separating policy reconciliation (AuthPolicy, RateLimitPolicy, DNSPolicy, TLSPolicy) from the kuadrant-operator into a `kuadrant-policy-controller` deployed as a component, making the kuadrant-operator purely an orchestration layer.
- **Kuadrant CR evolution**: exposing day 2 configuration (replicas, resources, storage backend) through the Kuadrant CR.

# Drawbacks
[drawbacks]: #drawbacks

- **Broader RBAC for kuadrant-operator**: the kuadrant-operator needs an unscoped `create` on `apiextensions.k8s.io` CRDs (scoped `create` is not possible in Kubernetes RBAC since the resource does not exist yet) and `escalate` on ClusterRoles. All other CRD verbs (`get`, `update`, `patch`) are scoped to specific child CRD names. These permissions follow the same pattern used by the OpenShift [opendatahub-operator](https://github.com/opendatahub-io/opendatahub-operator) and [cluster-olm-operator](https://github.com/openshift/cluster-olm-operator) in production.
- **Chart sync maintenance**: component controller charts must be kept in sync. Automated dependency sync (via GitHub dispatch) mitigates this but adds CI complexity.
- **No OLM CRD protection**: OLM does not know about child operator CRDs, so it cannot prevent another OLM-managed operator from claiming the same GVKs. In practice, this is a narrow risk: it requires installing a standalone child operator via OLM alongside kuadrant, which is a user error. It is worth noting that OLM's GVK protection only prevents conflicts between OLM-managed packages. It does nothing to prevent a Helm chart or raw manifest from altering or replacing a CRD on the cluster.
- **Startup failure is fatal**: if a child operator chart is malformed or the cluster rejects a resource, the kuadrant-operator will not start. This is a feature, not a bug: it prevents partially-upgraded states where some components are on the new version and others are not. The previous operator pod continues running until the new one is healthy (Deployment rollout strategy).

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why this design

- **No new operator**: the kuadrant-operator itself becomes the umbrella operator, avoiding a new codebase, image, CI pipeline, and release process.
- **No breaking changes**: wrapper CRs are preserved. Wrapper CR removal is deferred to a follow-on RFC when it can be properly investigated and tested.
- **Helm as first-class citizen**: both installation paths (direct Helm and OLM) use the same charts. The kuadrant-operator Helm chart is fully visible to Helm for its own resources. Child operator resources are rendered from the same upstream charts in both paths.
- **No OLM migration friction**: child operator CRDs are not in the bundle, so OLM GVK conflicts do not occur. The upgrade proceeds automatically and the operator handles cleanup of orphaned child Subscriptions/CSVs.
- **OLMv1 ready**: verified working with OLMv1 `ClusterExtension` on OpenShift CRC. No dependency declarations, no GVK conflicts.
- **Reduced operator footprint**: the OLM catalog goes from 4 packages to 1. Component repos can remove all OLM-specific artefacts.
- **Proven pattern**: OpenShift's [cluster-olm-operator](https://github.com/openshift/cluster-olm-operator) uses identical Helm SDK client-only rendering (`ClientOnly=true`, `DryRun=true`, `action.NewInstall`) to deploy sub-components in production. The [opendatahub-operator](https://github.com/opendatahub-io/opendatahub-operator) uses a similar manifest-based approach for 18+ components.
- **Path to further simplification**: the runtime rendering infrastructure enables future conditional component deployment and will be reused when child operators are flattened (deploying Authorino/Limitador workloads directly from their Helm charts).

## Alternatives considered

#### Installer-managed CRDs and ClusterRoles (split chart approach)

The chart sync tool renders each upstream chart, classifies resources by kind, and splits them into separate directories: `crds/` and `rbac/` for installer-managed resources, `charts/` for runtime rendering. CRDs and ClusterRoles are included in the OLM bundle and Helm chart at build time. The operator only renders namespaced resources (Deployments, SAs, etc.) at runtime, skipping CRDs and ClusterRoles.

**Rejected because:**
- Including child CRDs in the OLM bundle causes GVK conflicts during upgrades from the current multi-operator installation. `opm render` bakes `olm.gvk` properties from any CRDs in the bundle image, and the resolver blocks the upgrade when those GVKs are already provided by existing child operator CSVs. OLM provides no visible error when the upgrade is blocked; the subscription silently stays at `AtLatestKnown` with no indication that an upgrade is available but stalled.
- Splits the chart into installer-managed and runtime-managed resources, diverging from how the chart would be used in a standalone Helm installation. This creates two different deployment models for the same component.
- For the Helm install path, CRDs and ClusterRoles must be extracted into the kuadrant-operator Helm chart at build time while the operator renders the rest at runtime. Helm has no visibility of the runtime-managed resources.
- Tightly couples the sync tool to chart internals. The tool must render and classify resources by kind, and changes to chart structure can silently produce incorrect classification.
- No path to conditional component deployment. CRDs and ClusterRoles are installed at install time regardless of whether the component is needed.

#### Consolidated CSV (large CSV approach)

All child operator resources (CRDs, ClusterRoles, Deployments, SAs, RBAC) included directly in the kuadrant-operator CSV as static manifests. No runtime rendering. OLM and Helm both deploy the complete set of resources at install time. The existing `config/dependencies/` kustomize references are wired into the bundle and Helm chart build paths.

**Rejected because:**
- Same OLM GVK conflict as the split chart approach: child CRDs in the bundle cause `opm render` to bake GVK properties that conflict with existing child operator CSVs, blocking upgrades.
- For the Helm install path, child operator resources need to be duplicated or the Helm chart needs to pull from the same kustomize source as the bundle, coupling the two paths and producing the same cluster state through different mechanisms.
- All components always deployed with no path to conditional installation. Components that depend on cluster prerequisites (e.g. mcp-gateway requires Istio) cannot be excluded.
- Does not establish the runtime rendering infrastructure needed for future work (flattening child operators, deploying data plane workloads directly from their Helm charts).

#### New umbrella operator (separate from kuadrant-operator)

A separate operator that coordinates deployment while kuadrant-operator handles only policy reconciliation.

**Rejected because:**
- Introduces a new codebase, image, CI pipeline, and release process.
- Two operators to debug (deployment issues vs policy issues).
- The kuadrant-operator already has the Kuadrant CR watch, the topology, and the workflow infrastructure. Adding Helm rendering to it is simpler than building a new operator that needs all the same context.

#### Operator-managed Subscriptions

The kuadrant-operator creates OLM Subscription resources for each child operator at runtime, either programmatically or via resources included in the bundle. OLM then installs the child operators from the catalog as it does today, but triggered by the operator rather than by `dependencies.yaml`.

**Rejected because:**
- `Subscription` is an OLMv0 resource that is replaced by `ClusterExtension` in OLMv1. The operator would need to handle both APIs or be rewritten when OLMv1 becomes the default.
- Only works on clusters with OLM installed. For Helm installs on vanilla Kubernetes (EKS, GKE, AKS), child operators are installed via Helm chart dependencies. This means two completely separate code paths for deploying the same components, and the operator would need to detect which mechanism is in use.
- Child operator bundles, catalogs, and release pipelines all remain. The only thing removed is `dependencies.yaml`, so the per-component release overhead is unchanged.
- Still results in multiple CSVs and Subscriptions on the cluster, with no single view of product health.
- While simpler as a first step, it does not contribute to the longer-term goal of consolidating operators. The Subscription management code would be discarded when child operators are eventually absorbed into kuadrant-operator.

#### Manual installation of dependencies

Each component published as its own independent OLM package with no dependency declarations. Users install each one separately.

**Rejected because:**
- Significant UX regression. Users must create multiple Subscriptions or ClusterExtensions.
- Users are responsible for version compatibility and coordinated upgrades.
- Does not scale. Adding new components means more manual steps.

# Prior art
[prior-art]: #prior-art

- **[cluster-olm-operator](https://github.com/openshift/cluster-olm-operator)**: manages OLMv1 sub-components using Helm charts rendered client-only. Uses init containers to extract charts from component images at pod startup, renders with the same Helm SDK pattern (`ClientOnly=true`, `DryRun=true`, `action.NewInstall`). Our approach embeds charts at build time instead, but the runtime rendering is identical. Deployed in production OpenShift.
- **[opendatahub-operator](https://github.com/opendatahub-io/opendatahub-operator)** (downstream: [rhods-operator](https://github.com/red-hat-data-services/rhods-operator), Red Hat OpenShift AI): manages 18+ component controllers via manifests fetched from upstream repos at build time and applied at runtime. Uses `DataScienceCluster` CR with `managementState` per component for conditional deployment. Highlighted by the OLM team as a reference implementation for operators that need other operators present on the cluster without using OLM dependency declarations.
- **[OpenShift Ingress Operator](https://github.com/openshift/cluster-ingress-operator)**: deploys Istio control plane (including CRDs like EnvoyFilter) at runtime triggered by a GatewayClass CR. Another example of a core OpenShift operator managing CRDs and controllers at runtime without OLM involvement.
- **[OLMv0 dependency resolution](https://olm.operatorframework.io/docs/concepts/olm-architecture/dependency-resolution/)**: the current model being replaced.
- **[OLMv1 design decisions](https://github.com/operator-framework/operator-controller/blob/main/docs/project/olmv1_design_decisions.md)**: documents why dependency resolution was removed.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- **OLMv0 to OLMv1 cluster transition**: the exact process for migrating a cluster from OLMv0 to OLMv1 is not documented. The umbrella operator approach ensures Kuadrant is ready regardless of how this transition works.

- **CRD ownership conflicts with standalone installations**: if standalone Authorino is already installed on a cluster, the kuadrant-operator will apply the same CRDs via SSA with `Force: true`, potentially overwriting the standalone version. This is a user error scenario (installing two things that manage the same CRDs) but could cause unexpected behaviour if the versions differ.

- **Wrapper CR removal timeline**: this RFC preserves wrapper CRs. The timeline and approach for removing them depends on how much the Authorino and Limitador CRs are used for direct user customisation vs being purely internal. This will be addressed in a follow-on RFC.

- **Control plane status and configuration CR**: the RFC identifies the need for a new CR to report child operator health and eventually enable conditional deployment, but the design of this CR is deferred to implementation.
