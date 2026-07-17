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

# Motivation
[motivation]: #motivation

## OLM dependency resolution is being removed

OLMv1 [explicitly removes automatic dependency installation](https://github.com/operator-framework/operator-controller/blob/main/docs/project/olmv1_design_decisions.md). The [design decisions document](https://github.com/operator-framework/operator-controller/blob/main/docs/project/olmv1_design_decisions.md) states:

> OLM v1 will not automatically install a missing dependency when a user requests an operator. The primary reasoning is that OLM v1 will err on the side of predictability and cluster-administrator awareness.

Bundles declaring `olm.package.required`, `olm.gvk.required`, or `olm.constraint` will be rejected under OLMv1. Kuadrant's current model relies entirely on these declarations.

The exact process for transitioning a cluster (fully) from OLMv0 to OLMv1 is not yet documented — this is an open question that affects all operators, not just Kuadrant. Currently on OpenShift OLMv1 and OLMv0 run alongside each other. However, regardless of when or how that transition happens, the dependency resolution mechanism Kuadrant relies on will not exist in the future. We should eliminate our dependency declarations now, while still on OLMv0, so that Kuadrant is already compatible when the transition arrives.

## Dependency operators exist to satisfy OLM packaging requirements

These dependency operators (authorino-operator, limitador-operator) exist because OLMv0 required each component to be packaged as its own operator with its own CSV, Subscription, and CRD. They were created to satisfy this packaging requirement, not because the workloads they manage have complex lifecycle needs. Each maintains its own release pipeline, bundle images, and catalog entries. Consolidating them under the kuadrant-operator eliminates this per-component overhead.

## Kuadrant needs to run beyond OpenShift

Kuadrant's deployment targets are not limited to OpenShift. In the future, we want to support any Kubernetes distribution (EKS, GKE, AKS, and others). An architecture that depends on OLM for installation and lifecycle management ties Kuadrant to an OpenShift-specific mechanism that doesn't exist on these platforms. Helm is the common denominator. Making Helm the primary installation method ensures Kuadrant is a first-class citizen on any Kubernetes distribution.

## Reduced release and maintenance overhead

Each independent OLM operator requires its own bundle image, catalog entry, release pipeline, and version coordination. Consolidating into a single OLM package eliminates per-component release overhead while keeping the same controllers and workloads running on the cluster. No CRDs, Deployments, or other resources are removed or deprecated as part of this work.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The kuadrant-operator becomes an umbrella operator that deploys component controllers at runtime. Wrapper CRs (Authorino CR, Limitador CR) are preserved. No breaking changes for users.

## Before

4 OLM operators (kuadrant, authorino, limitador, dns), each with its own CSV, bundle image, and catalog entry. OLM manages all lifecycles independently.

## After

1 OLM operator (kuadrant-operator) with a single CSV, bundle, and catalog entry. Component controllers (authorino-operator, limitador-operator, dns-operator, mcp-gateway) are no longer OLM packages. They become internal controller deployments whose lifecycle is managed by the kuadrant-operator. No component controller bundles, catalogs, or CSVs need to be maintained.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

For visual diagrams covering the build-time chart sync, runtime reconciliation chain, RBAC model, and resource ownership, see [Architecture Diagrams](0019-olmv1-operator-consolidation/architecture-diagrams.md).

## Chart sync process

Component Helm charts are sourced from upstream repos (authorino-operator, limitador-operator, dns-operator, mcp-gateway). A Go sync tool (`hack/sync-child-charts/`) downloads each chart, renders it using the Helm SDK, and classifies the rendered resources by kind. The output is split into three directories:

```text
config/child-operators/
├── crds/                         # Rendered CRDs (one file per component)
│   ├── authorino-operator.yaml
│   ├── limitador-operator.yaml
│   ├── dns-operator.yaml
│   └── mcp-gateway.yaml
├── rbac/                         # Rendered ClusterRoles (one file per component)
│   ├── authorino-operator.yaml
│   ├── limitador-operator.yaml
│   ├── dns-operator.yaml
│   └── mcp-gateway.yaml
└── charts/                       # Raw chart templates for runtime rendering
    ├── authorino-operator/
    ├── limitador-operator/
    ├── dns-operator/
    └── mcp-gateway/
```

The separation serves a specific purpose:

- **`crds/` and `rbac/`** contain rendered output (real resource names, no Helm template expressions). These are included in the kuadrant-operator's OLM bundle and Helm chart via kustomize at build time. They are cluster-scoped resources managed by the installation method.
- **`charts/`** contains the raw Helm chart templates (Chart.yaml, values.yaml, templates/) copied as-is from upstream. These are packaged into the operator container image and rendered at runtime when a Kuadrant CR is created.

The sync tool handles both simple charts (single `manifests.yaml` generated by kustomize) and mature charts with full Helm templating (helpers, conditionals, configurable values). It renders to classify resources but preserves the raw templates for runtime use.

Synced charts are committed to the repo. The sync is run manually via `make sync-child-operator-charts` when updating component versions.

## Kuadrant-operator Helm chart

The kuadrant-operator's own Helm chart (`charts/kuadrant-operator/`) should be updated to follow Helm best practices. Currently, it's a monolithic `manifests.yaml` generated by kustomize with all resources (including CRDs) inline in `templates/`. As part of this work, the chart should be restructured to use Helm's native `crds/` directory for all CRDs (kuadrant and component). This ensures correct install ordering (CRDs installed before templates) and aligns with Helm conventions.

## Resource ownership

Resource ownership is clearly split between the installation method and the operator:

**Managed by the installer (Helm chart or OLM bundle):**

| Resource | Source | Notes |
|---|---|---|
| All CRDs (kuadrant + component) | `config/child-operators/crds/` and `config/crd/` | Installed before the operator starts |
| Component ClusterRoles | `config/child-operators/rbac/` | Installed before the operator starts |
| kuadrant-operator Deployment | `config/manager/` | The operator itself |
| kuadrant-operator ServiceAccount, ClusterRole, ClusterRoleBinding | `config/rbac/` | Operator's own RBAC |

**Managed by the kuadrant-operator at runtime (triggered by Kuadrant CR creation):**

| Resource | Notes |
|---|---|
| Component controller Deployments | authorino-operator, limitador-operator, dns-operator, mcp-gateway |
| Component ServiceAccounts | Per-component identity |
| Component ClusterRoleBindings | Binds component SAs to pre-installed ClusterRoles using `bind`/`escalate` |
| Component Roles, RoleBindings | Namespace-scoped RBAC (e.g. leader election) |
| Component Services, ConfigMaps | Metrics, configuration |
| Wrapper CRs | Authorino CR, Limitador CR (reconciled by child operators into workloads) |

The operator does not create or modify cluster-scoped resources (CRDs, ClusterRoles) at runtime. If a rendered chart produces a ClusterRole or CRD, the operator skips it with a log warning.

## Runtime rendering

When a user creates a Kuadrant CR, the kuadrant-operator renders each component controller's Helm chart from `/charts/<name>/` in the container image using the Helm Go SDK (`ClientOnly=true`, `DryRun=true`). The rendered namespaced resources are applied via Server-Side Apply with the kuadrant-operator as the field manager.

All component controller Deployments are owned by the Kuadrant CR via ownerReference. This includes dns-operator, which previously ran independently and did not require a Kuadrant CR.

## RBAC

The kuadrant-operator does not need to duplicate component controller permissions. The Kubernetes `bind` verb allows creating ClusterRoleBindings to named ClusterRoles without holding all their permissions. The operator's ClusterRole includes `bind`/`escalate` on each component's ClusterRole `resourceNames`.

`bind` is sufficient for steady-state runtime operation (creating ClusterRoleBindings). `escalate` is retained for migration scenarios where the operator may need to update existing ClusterRole rules to align with the consolidated model. Whether rule changes are needed depends on the migration implementation. If migration only requires metadata updates (annotations, labels), `escalate` can be removed.

Only ClusterRole names need tracking, not their contents. When a component controller changes its RBAC rules, no kuadrant-operator change is needed unless a ClusterRole is added or renamed.

Component ClusterRoles must use fixed, predictable names. The `bind`/`escalate` permissions use `resourceNames`, so the operator must know the exact names at build time. Charts that derive ClusterRole names from the Helm release name (e.g. via the `fullname` helper) create a fragile dependency that can silently break the RBAC contract. The sync tool validates rendered ClusterRole names at sync time, catching any mismatches before they reach a cluster.

## Two supported installation methods

- **Helm**: Single `helm install` deploys everything. The kuadrant-operator Helm chart includes all CRDs (kuadrant and component) in its `crds/` directory following Helm best practices, and component ClusterRoles in `templates/`. CRDs and ClusterRoles are generated at `make helm-build` time from `config/child-operators/crds/` and `config/child-operators/rbac/` via kustomize.
- **OLM**: Single operator in the catalog. The bundle includes all component CRDs and ClusterRoles (from `config/child-operators/` via `config/manifests/kustomization.yaml`). No component controller bundles in the catalog.

Both paths produce the same cluster state: CRDs and ClusterRoles installed, kuadrant-operator running, no component controllers until a Kuadrant CR is created.

## Wrapper CRs preserved

Authorino CR and Limitador CR are still created by kuadrant-operator. Component controllers reconcile these wrapper CRs to create workloads. No change to this flow. Users see no difference in behaviour.

## Kuadrant CR deletion must manage component lifecycle

Component controller Deployments are owned by the Kuadrant CR via ownerReference. Several CRDs use finalizers that require their controller to be running:

- Authorino CR has finalizer `authorino.kuadrant.io/finalizer`
- DNSRecord CRs have finalizers for cleaning up external DNS provider records

If the Kuadrant CR is deleted and the cascade deletes the controllers simultaneously, CRs with finalizers will be stuck in `Terminating` state. The kuadrant-operator must have a finalizer on the Kuadrant CR that enforces correct deletion order: ensure all controller-dependent resources with finalizers (wrapper CRs, DNSRecords, and any others) have completed their finalizer logic before allowing the cascade to remove component controller Deployments.

## OLM bundle and catalog

The bundle/catalog pipeline deals with a single package (kuadrant-operator). Component controller bundle images, catalog channel entries, and dependency injection are removed. Component CRDs and ClusterRoles are included in the kuadrant-operator bundle directly from `config/child-operators/`.

## Component repos

No changes are required in component repos for the consolidation to work. The sync tool will be designed to handle any chart structure.

However, some upstream chart changes would be beneficial:

- **CRDs directory**: charts that keep CRDs in `templates/` rather than the native Helm `crds/` directory could benefit from separating them, making the sync tool's job cleaner
- **Consistent labels**: applying consistent labels across all component resources (e.g. `kuadrant.io/component`) would improve topology tracking and filtering. Currently, the Authorino Deployment lacks distinguishing labels, preventing it from being tracked in the kuadrant-operator topology
- **Unique Deployment selectors**: the limitador-operator chart uses a generic `control-plane: controller-manager` selector that collides with the kuadrant-operator in the same namespace. This is a pre-existing bug that needs fixing upstream
- **Fixed ClusterRole names**: charts using Helm's `fullname` helper for ClusterRole names create a fragile dependency on the release name. ClusterRoles are cluster-scoped permission definitions and should use fixed names

Since component repos no longer need to produce their own OLM bundles or catalogs, OLM-specific artefacts can also be removed to reduce maintenance burden: `bundle/`, `catalog/`, `config/manifests/`, `config/deploy/olm/`, `config/scorecard/`, `bundle.Dockerfile`, `operator-sdk` and `opm` tool dependencies.

## Automated dependency sync

Component repos need automation to notify the kuadrant-operator when their charts change. A pattern already exists between authorino and authorino-operator: push to main triggers a `repository_dispatch` to the downstream repo, which runs the sync tool and creates a PR.

## Migration considerations

- **CRD ownership transfer**: OLM tracks CRD ownership via annotations. These need updating during the transition.
- **Orphaned OLM resources**: OLM will not automatically uninstall previously resolved child operators. Subscriptions, CSVs, and associated resources must be explicitly cleaned up.
- **Control plane resource conflicts**: existing Deployments may have different field managers (OLM vs SSA). The safest approach is to delete old control plane resources before upgrade. This does not affect the data plane.
- **Data plane continuity**: wrapper CRs must be preserved throughout migration. Their workloads continue running uninterrupted.
- **Resource naming consistency**: resource names produced by the chart rendering should match names currently used by OLM-installed operators to avoid orphaned resources.
- **Consistent labelling**: the umbrella operator can apply consistent labels to all managed resources, addressing existing gaps (e.g. Authorino Deployment lacks distinguishing labels for topology tracking).

# Future possibilities
[future-possibilities]: #future-possibilities

This RFC deliberately preserves wrapper CRs and child operator controllers. A follow-on RFC should address further architectural simplification:

- **Removing intermediate operator layers and wrapper CRs**: deploying Authorino and Limitador workloads directly from the kuadrant-operator, eliminating authorino-operator and limitador-operator as running controllers. This removes the `Authorino` and `Limitador` CRDs from the user-facing API surface. Each operator removed is a separate container image with its own Go dependency tree. Every transitive dependency is a potential CVE that needs scanning, patching, and releasing. Removing these controllers reduces the security surface area and maintenance burden.
- **Extracting a dedicated policy controller**: separating policy reconciliation (AuthPolicy, RateLimitPolicy, DNSPolicy, TLSPolicy) from the kuadrant-operator into a `kuadrant-policy-controller` deployed as a component, making the kuadrant-operator purely an orchestration layer.
- **Selective component installation**: adding `spec.components` to the Kuadrant CRD for user-facing control over which components are deployed.
- **Kuadrant CR evolution**: exposing day 2 configuration (replicas, resources, storage backend) through the Kuadrant CR.

# Drawbacks
[drawbacks]: #drawbacks

- **Broader RBAC for kuadrant-operator**: the kuadrant-operator now needs permissions to manage Deployments, Services, RBAC, and ConfigMaps for all component controllers, plus `bind`/`escalate` on their ClusterRoles.
- **Chart sync maintenance**: component controller charts must be kept in sync. Automated dependency sync (via GitHub dispatch) mitigates this but adds CI complexity.
- **Finalizer ordering**: the Kuadrant CR must manage deletion order to prevent wrapper CR finalizers from being orphaned. This adds reconciliation complexity.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why this design

- **No new operator**: the kuadrant-operator itself becomes the umbrella operator, avoiding a new codebase, image, CI pipeline, and release process.
- **No breaking changes**: wrapper CRs are preserved. Wrapper CR removal is deferred to a follow-on RFC when it can be properly investigated and tested.
- **Helm as first-class citizen**: both installation paths (direct Helm and OLM) consume the same charts, unifying upstream and downstream experiences.
- **Reduced operator footprint**: the OLM catalog goes from 4 packages to 1. Component repos can remove all OLM-specific artefacts.
- **Proven pattern**: OpenShift's [cluster-olm-operator](https://github.com/openshift/cluster-olm-operator) uses Helm client-only rendering in production.

## Alternatives considered

#### New umbrella operator (separate from kuadrant-operator)

A separate operator that coordinates deployment while kuadrant-operator handles only policy reconciliation.

**Rejected because:**
- Introduces a new codebase, image, CI pipeline, and release process.
- Two operators to debug (deployment issues vs policy issues).
- The kuadrant-operator already has the Kuadrant CR watch, the topology, and the workflow infrastructure. Adding Helm rendering to it is simpler than building a new operator that needs all the same context.

#### New umbrella operator with kuadrant-operator as policy controller

Introduce a new umbrella operator for deployment orchestration. The existing kuadrant-operator becomes a pure policy controller deployed as a component, and the new operator takes ownership of the Kuadrant CR.

**Rejected because:**
- Introduces a new codebase, image, CI pipeline, and release process.
- The kuadrant-operator already has the Kuadrant CR watch, the topology, and the workflow infrastructure. Adding Helm rendering to it is simpler than building a new operator that needs all the same context.
- Two operators to debug (deployment issues vs policy issues).
- Splitting policy reconciliation out of the kuadrant-operator can still be done as a future phase if needed, without requiring a new operator to be built first.

#### Large CSV

A single OLM bundle containing all existing operators in one CSV with zero dependency declarations.

**Rejected because:**
- Dependency operators are retained. Their maintenance cost, security surface area, and release complexity remain.
- OLM owns all Deployments in the CSV and reconciles them back to the declared state. Users cannot scale replicas or adjust resources without rebuilding the bundle.
- Does not put Helm at the centre of installation.

#### Manual installation of dependencies

Each component published as its own independent OLM package with no dependency declarations. Users install each one separately.

**Rejected because:**
- Significant UX regression. Users must create multiple Subscriptions or ClusterExtensions.
- Users are responsible for version compatibility and coordinated upgrades.
- Does not scale. Adding new components means more manual steps.

# Prior art
[prior-art]: #prior-art

- **[cluster-olm-operator](https://github.com/openshift/cluster-olm-operator)**: manages OLMv1 sub-components using Helm charts rendered client-only.
- **[OLMv0 dependency resolution](https://olm.operatorframework.io/docs/concepts/olm-architecture/dependency-resolution/)**: the current model being replaced.
- **[OLMv1 design decisions](https://github.com/operator-framework/operator-controller/blob/main/docs/project/olmv1_design_decisions.md)**: documents why dependency resolution was removed.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- **OLMv0 to OLMv1 cluster transition**: the exact process for migrating a cluster from OLMv0 to OLMv1 is not documented. The umbrella operator approach ensures Kuadrant is ready regardless of how this transition works.

- **OLMv0 behaviour when a CSV drops dependency declarations**: when the kuadrant-operator CSV is upgraded to a version that removes its `olm.package.required` declarations, what happens to the dependency Subscriptions/CSVs that OLMv0 auto-installed? The OLMv1 design doc states they are not automatically removed, but this needs verification.

- **CRD ownership conflicts with standalone installations**: if standalone Authorino is already installed on a cluster, the kuadrant-operator's bundle includes Authorino CRDs. Under OLMv1's single-ownership model, two bundles cannot own the same CRDs.

- **Wrapper CR removal timeline**: this RFC preserves wrapper CRs. The timeline and approach for removing them depends on how much the Authorino and Limitador CRs are used for direct user customisation vs being purely internal. This will be addressed in a follow-on RFC.
