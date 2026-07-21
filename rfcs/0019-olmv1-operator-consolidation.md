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

## Per-component OLM packaging adds overhead within a Kuadrant deployment

Authorino, Limitador, and mcp-gateway are independent projects with standalone use cases outside Kuadrant. Their operators were created in part to support independent installation and lifecycle management. However, within a Kuadrant deployment, they function as internal components whose lifecycle is tied to the Kuadrant CR. The per-component OLM packaging (separate CSV, Subscription, bundle image, catalog entry, release pipeline) adds overhead that is only justified when the components are deployed independently.

This RFC consolidates the Kuadrant-as-a-product installation path. The individual component repos and their Helm charts remain available for standalone installation. Nothing prevents a user from installing just Limitador or just Authorino directly from their own repos.

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
| Component ClusterRoleBindings | Binds component SAs to pre-installed ClusterRoles using `bind` |
| Component Roles, RoleBindings | Namespace-scoped RBAC (e.g. leader election) |
| Component Services, ConfigMaps | Metrics, configuration |
| Wrapper CRs | Authorino CR, Limitador CR (reconciled by child operators into workloads) |

The operator does not create or modify cluster-scoped resources (CRDs, ClusterRoles) at runtime. If a rendered chart produces a ClusterRole or CRD, the operator skips it with a log warning.

## Runtime rendering

When a user creates a Kuadrant CR, the kuadrant-operator renders each component controller's Helm chart from `/charts/<name>/` in the container image using the Helm Go SDK (`ClientOnly=true`, `DryRun=true`). The rendered namespaced resources are applied via Server-Side Apply with the kuadrant-operator as the field manager.

All component controller Deployments are owned by the Kuadrant CR via ownerReference. This includes dns-operator, which previously ran independently and did not require a Kuadrant CR.

## RBAC

The kuadrant-operator does not need to duplicate component controller permissions. The Kubernetes `bind` verb allows creating ClusterRoleBindings to named ClusterRoles without holding all their permissions. The operator's ClusterRole includes `bind` on each component's ClusterRole `resourceNames`. No `escalate` is needed since the operator does not modify ClusterRole rules at runtime; ClusterRoles are pre-installed by the installer (Helm or OLM bundle).

Only ClusterRole names need tracking, not their contents. When a component controller changes its RBAC rules, no kuadrant-operator change is needed unless a ClusterRole is added or renamed.

Component ClusterRoles must use fixed, predictable names. The `bind` permissions use `resourceNames`, so the operator must know the exact names at build time. Charts that derive ClusterRole names from the Helm release name (e.g. via the `fullname` helper) create a fragile dependency that can silently break the RBAC contract. The sync tool validates rendered ClusterRole names at sync time, catching any mismatches before they reach a cluster.

## Two supported installation methods

- **Helm**: Single `helm install` deploys everything. The kuadrant-operator Helm chart includes all CRDs (kuadrant and component) in its `crds/` directory following Helm best practices, and component ClusterRoles in `templates/`. CRDs and ClusterRoles are generated at `make helm-build` time from `config/child-operators/crds/` and `config/child-operators/rbac/` via kustomize.
- **OLM**: Single operator in the catalog. The bundle includes all component CRDs and ClusterRoles (from `config/child-operators/` via `config/manifests/kustomization.yaml`). No component controller bundles in the catalog.

Both paths produce the same cluster state: CRDs and ClusterRoles installed, kuadrant-operator running, no component controllers until a Kuadrant CR is created.

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

These are declared as environment variables on the kuadrant-operator Deployment (`config/manager/manager.yaml`). The downstream build system overrides them with version-pinned or mirrored image references. The Helm reconcilers read these env vars and apply the correct images when rendering child operator charts, either by patching the rendered Deployment image directly (for simple charts that don't support values-based overrides) or by passing Helm values (for charts with configurable image fields).

## Wrapper CRs preserved

Authorino CR and Limitador CR are still created by kuadrant-operator. Component controllers reconcile these wrapper CRs to create workloads. No change to this flow. Users see no difference in behaviour.

## Kuadrant CR deletion must manage component lifecycle

Component controller Deployments are owned by the Kuadrant CR via ownerReference. Several CRDs use finalizers that require their controller to be running:

- Authorino CR has finalizer `authorino.kuadrant.io/finalizer`
- DNSRecord CRs have finalizers for cleaning up external DNS provider records

If the Kuadrant CR is deleted and the cascade deletes the controllers simultaneously, CRs with finalizers will be stuck in `Terminating` state. The kuadrant-operator must have a finalizer on the Kuadrant CR that enforces correct deletion order: ensure all controller-dependent resources with finalizers (wrapper CRs, DNSRecords, and any others) have completed their finalizer logic before allowing the cascade to remove component controller Deployments.

## OLM bundle and catalog

The bundle/catalog pipeline deals with a single package (kuadrant-operator). Component controller bundle images, catalog channel entries, and dependency injection are removed. Component CRDs and ClusterRoles are included in the kuadrant-operator bundle directly from `config/child-operators/`. All component CRDs are declared as `owned` in the CSV, making kuadrant-operator the sole OLM owner. This requires a pre-upgrade cleanup step for existing multi-operator installations (see [Migration](#migration-from-multi-operator-olm-installation)).

## Component repos

No changes are required in component repos for the consolidation to work. The sync tool will be designed to handle any chart structure.

However, some upstream chart changes would be beneficial:

- **CRDs directory**: charts that keep CRDs in `templates/` rather than the native Helm `crds/` directory could benefit from separating them, making the sync tool's job cleaner
- **Consistent labels**: applying consistent labels across all component resources (e.g. `kuadrant.io/component`) would improve topology tracking and filtering. Currently, the Authorino Deployment lacks distinguishing labels, preventing it from being tracked in the kuadrant-operator topology
- **Unique Deployment selectors**: the limitador-operator chart uses a generic `control-plane: controller-manager` selector that collides with the kuadrant-operator in the same namespace. This is a pre-existing bug that needs fixing upstream
- **Fixed ClusterRole names**: charts using Helm's `fullname` helper for ClusterRole names create a fragile dependency on the release name. ClusterRoles are cluster-scoped permission definitions and should use fixed names

Since component repos no longer need to produce their own OLM bundles or catalogs, OLM-specific artefacts can also be removed to reduce maintenance burden: `bundle/`, `catalog/`, `config/manifests/`, `config/deploy/olm/`, `config/scorecard/`, `bundle.Dockerfile`, `operator-sdk` and `opm` tool dependencies.

## Automated dependency sync

With the kuadrant-operator consuming charts from multiple child repos, automation is needed to keep them in sync. A pattern for this already exists between authorino and authorino-operator: push to main triggers a `repository_dispatch` to the downstream repo, which runs the sync tool and creates a PR.

The same pattern should be adopted for the kuadrant-operator. When a child repo's CI completes successfully on a tracked branch, it dispatches to the kuadrant-operator. The sync tool runs, and if there are changes, a PR is created or updated. The automation should manage a single open PR per target branch and child component (e.g. `sync/main/authorino-operator`, `sync/release-v1.5/authorino-operator`), force-updating it on each trigger rather than accumulating stale PRs.

This automation must support not just `main` but also `release-*` branches. Multiple kuadrant-operator release branches may track different branches of the same child component. For example, `kuadrant-operator/main` may track `authorino-operator/main` while `kuadrant-operator/release-v1.5` tracks `authorino-operator/release-v1.5`.

Changes to the release process and version pinning strategy are out of scope for this RFC and will be addressed separately.

## Migration from multi-operator OLM installation

### The problem: OLM GVK conflict

OLM's dependency resolver prevents two CSVs from providing the same GVK (Group-Version-Kind) within a namespace. When the consolidated kuadrant-operator bundle declares child CRDs (AuthConfig, Limitador, DNSRecord, etc.) as `owned`, the resolver sees a conflict with the still-installed child operator CSVs that also own those GVKs.

This conflict manifests as a `ConstraintsNotSatisfiable` condition on the Subscription:

```
constraints not satisfiable: kuadrant-operator.v0.0.1 and dns-operator.v0.0.0
provide DNSHealthCheckProbe (kuadrant.io/v1alpha1)
```

The GVK properties are baked into the bundle by `opm render` from any CRDs present in the bundle image. They cannot be removed from the CSV `owned` list without also removing the CRDs from the bundle, and they cannot be removed from the `olm.gvk` catalog properties at all since `opm` generates them from the bundle content.

The upgrade stalls safely: the existing deployment continues running, no data is lost, and the user has time to resolve the conflict.

### Chosen approach: documented migration cleanup

The consolidated bundle declares all child CRDs as `owned`. The OLM upgrade stalls until the user removes the existing child operator Subscriptions and CSVs that conflict. This is a one-time migration step.

In practice, the catalog update happens first (either pushed by an administrator or picked up automatically by OLM's catalog polling). The upgrade stalls safely, and the user then performs the cleanup to unblock it. For users who want full control over timing, setting `installPlanApproval: Manual` on the Subscription before the catalog update is recommended.

**Migration steps:**

```bash
# 1. (Recommended) Set manual approval to control upgrade timing
kubectl patch subscription kuadrant -n kuadrant-system \
  --type=merge -p '{"spec":{"installPlanApproval":"Manual"}}'

# 2. Remove child operator subscriptions (and mcp-gateway if installed standalone)
kubectl delete subscription -n kuadrant-system \
  authorino-operator-preview-kuadrant-operator-catalog-kuadrant-system \
  dns-operator-preview-kuadrant-operator-catalog-kuadrant-system \
  limitador-operator-preview-kuadrant-operator-catalog-kuadrant-system
# If mcp-gateway was installed standalone:
# kubectl delete subscription -n <namespace> <mcp-gateway-subscription-name>

# 3. Remove child operator CSVs (and mcp-gateway if installed standalone)
kubectl delete csv -n kuadrant-system \
  authorino-operator.v0.0.0 \
  dns-operator.v0.0.0 \
  limitador-operator.v0.0.0
# If mcp-gateway was installed standalone:
# kubectl delete csv -n <namespace> <mcp-gateway-csv-name>

# 4. If using manual approval, approve the install plan
kubectl get installplan -n kuadrant-system
kubectl patch installplan <name> -n kuadrant-system \
  --type=merge -p '{"spec":{"approved":true}}'
```

**Tested behaviour** (verified on OpenShift CRC):

1. Deleting child operator CSVs does **not** cascade-delete their CRDs. OLM tracks CRD ownership via labels (`operators.coreos.com/<operator>.<namespace>`) and annotations, not via Kubernetes ownerReferences. CRDs are cluster-scoped and CSVs are namespace-scoped, so Kubernetes cross-scope garbage collection does not apply.
2. Deleting child CSVs **does** remove their managed Deployments (the child operator controllers stop).
3. After cleanup, OLM resolves the upgrade and installs the consolidated bundle. The bundle re-installs all CRDs (updating them if schemas changed) and the kuadrant-operator deploys child controllers via Helm rendering.
4. With automatic upgrades, the upgrade stalls with `ResolutionFailed` until the cleanup is performed. The existing deployment continues running with no data loss. Once the user performs the cleanup, the next resolver cycle (every few seconds) succeeds.

The `ResolutionFailed` condition message clearly identifies the GVK conflict, making it straightforward to diagnose and point users at the migration documentation.

### Alternatives considered for migration

#### Transition release with runtime CRD management

A stepping-stone release where the bundle contains no child CRDs. The operator applies CRDs at runtime, removes child subscriptions/CSVs programmatically, and sets OLM ownership labels. The next release then includes child CRDs as `owned` in the bundle.

**Rejected because:**
- Requires granting the operator `apiextensions.k8s.io` permissions to create/update CRDs. The `create` verb cannot be scoped by `resourceNames` (the resource doesn't exist yet), so it grants broad CRD creation ability.
- Creates a window where child CRDs are unmanaged by OLM. No CSV declares them as `owned`, so OLM provides no protection against another operator claiming those GVKs.
- Adds a release that exists solely for migration mechanics, increasing complexity.
- The two-phase approach (transition + final) doubles the migration testing surface.

#### Stripping child CRDs from CSV owned list

Keep child CRDs in the bundle but remove them from the CSV's `spec.customresourcedefinitions.owned` list in a post-generation step. This avoids the GVK conflict at the CSV level.

**Rejected because:**
- `opm render` generates `olm.gvk` properties from CRDs present in the bundle image, regardless of what the CSV declares. The resolver uses these properties, not the CSV's `owned` list. Even with child CRDs stripped from `owned`, the resolver still sees the GVK conflict.
- `operator-sdk bundle validate` rejects bundles where CRDs are present but not declared in the CSV. While validation can be skipped, it indicates the bundle is in an unsupported state.

#### New API versions for child CRDs

Introduce new API versions (e.g. `v1alpha2`) for child CRDs to avoid GVK overlap with existing child operator CSVs.

**Rejected because:**
- CRDs must still serve existing versions for backwards compatibility. A CRD serving both `v1alpha1` and `v1alpha2` still provides the `v1alpha1` GVK, which conflicts with the child CSV.
- Creating new API versions purely for migration mechanics adds permanent API surface area for a transient problem.
- Does not address the child subscription constraint (subscriptions pin child CSVs regardless of GVK overlap).

### Other migration considerations

- **Data plane continuity**: wrapper CRs (Authorino CR, Limitador CR) and their workloads are preserved throughout migration. Deleting child CSVs removes operator controllers but does not affect running workloads (Authorino server, Limitador server).
- **Control plane gap**: between deleting child CSVs (which removes child controllers) and the consolidated operator starting (which redeploys them via Helm), there is a brief window where child controllers are not running. During this window, changes to wrapper CRs or child CRDs are not reconciled. Existing workloads continue running.
- **Resource naming consistency**: resource names produced by the chart rendering should match names currently used by OLM-installed operators to avoid orphaned resources.
- **Consistent labelling**: the umbrella operator can apply consistent labels to all managed resources, addressing existing gaps (e.g. Authorino Deployment lacks distinguishing labels for topology tracking).

### Upgrade testing

Automated upgrade tests must be established to validate the migration path from the current multi-operator OLM installation to the consolidated operator. These tests should:

- Start with the previous release installed via OLM (all 4 operators with their CSVs)
- Create a Kuadrant CR with active policies and ingress traffic
- Perform pre-upgrade cleanup (remove child subscriptions and CSVs)
- Upgrade to the consolidated release via OLM catalog channel
- Verify all child CRDs survived the migration
- Verify kuadrant-operator CSV declares ownership of all CRDs
- Verify all component controllers are running (now managed by kuadrant-operator)
- Verify zero data plane disruption throughout (policies continue to be enforced, no traffic loss)
- Verify CRD schema changes are applied correctly when present

These tests should run in CI for every release and serve as ongoing regression testing for the OLM upgrade path.

# Future possibilities
[future-possibilities]: #future-possibilities

This RFC deliberately preserves wrapper CRs and child operator controllers. A follow-on RFC should address further architectural simplification:

- **Removing intermediate operator layers and wrapper CRs**: deploying Authorino and Limitador workloads directly from the kuadrant-operator, eliminating authorino-operator and limitador-operator as running controllers. This removes the `Authorino` and `Limitador` CRDs from the user-facing API surface. Each operator removed is a separate container image with its own Go dependency tree. Every transitive dependency is a potential CVE that needs scanning, patching, and releasing. Removing these controllers reduces the security surface area and maintenance burden.
- **Extracting a dedicated policy controller**: separating policy reconciliation (AuthPolicy, RateLimitPolicy, DNSPolicy, TLSPolicy) from the kuadrant-operator into a `kuadrant-policy-controller` deployed as a component, making the kuadrant-operator purely an orchestration layer.
- **Selective component installation**: adding `spec.components` to the Kuadrant CRD for user-facing control over which components are deployed.
- **Kuadrant CR evolution**: exposing day 2 configuration (replicas, resources, storage backend) through the Kuadrant CR.

# Drawbacks
[drawbacks]: #drawbacks

- **Broader RBAC for kuadrant-operator**: the kuadrant-operator now needs permissions to manage Deployments, Services, RBAC, and ConfigMaps for all component controllers, plus `bind` on their ClusterRoles.
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

- **CRD ownership conflicts with standalone installations**: if standalone Authorino is already installed on a cluster, the kuadrant-operator's bundle includes Authorino CRDs. Under OLMv1's single-ownership model, two bundles cannot own the same CRDs. This is the same GVK conflict pattern as the migration scenario, and would require the standalone operator to be removed first.

- **Wrapper CR removal timeline**: this RFC preserves wrapper CRs. The timeline and approach for removing them depends on how much the Authorino and Limitador CRs are used for direct user customisation vs being purely internal. This will be addressed in a follow-on RFC.
