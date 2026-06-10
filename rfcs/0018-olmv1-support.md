# RFC: OLM Dependency Model Transition

- Feature Name: `olm_dependency_model_transition`
- Start Date: 2026-03-30
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [Kuadrant/architecture#164](https://github.com/Kuadrant/architecture/issues/164)

# Summary
[summary]: #summary

Replace OLMv0's automatic dependency installation with a separate umbrella operator that deploys and manages Kuadrant's core components (kuadrant-operator, authorino-operator, limitador-operator, dns-operator) and their operands. Helm charts become the single manifest source — consumed directly via `helm install` and by the umbrella operator via client-only rendering. The kuadrant-operator is simplified to focus exclusively on policy reconciliation; operator installation, configuration and operand creation move to the umbrella operator. Extension operators such as [mcp-gateway-operator](https://github.com/Kuadrant/mcp-gateway) independently remove their OLMv0 dependency declarations to become OLMv1-ready; folding them into the umbrella operator is expected but is a future step. A goal of this pattern is to also enable enhancements such as optional installation of specific components based on the use case.

- OpenShift (OLM): Umbrella Operator main domain
- Kubernetes: Helm domain

This follows the pattern used by OpenShift's [cluster-olm-operator](https://github.com/openshift/cluster-olm-operator).

# Motivation
[motivation]: #motivation

OLMv1 (ClusterExtensions) [explicitly removes automatic dependency installation](https://github.com/operator-framework/operator-controller/blob/main/docs/project/olmv1_design_decisions.md), breaking Kuadrant's current single-action install experience. Kuadrant operator responsibilities are currently dual purpose: Installation and policy reconciliation. Finally with the advent of extensions and a growing remit of possible policies, we should prepare for a future where not all components are installed with every installation.

Target timeline: **end of 2026**.

### Requirements

1. **Single-action install** — users create one ClusterExtension, not many.
2. **Full lifecycle management** — the umbrella operator manages versions, upgrades, and removal of all components.
3. **OLMv1 compatible** — works with the ClusterExtensions API on both OpenShift and vanilla Kubernetes.
4. **Separation of concerns** — the umbrella operator handles deployment; kuadrant-operator (becomes kuadrant-controller) handles policy reconciliation and has no OLMv1 awareness.
5. **Helm as single manifest source** — both installation paths (direct Helm (upstream plain kubernetes) and umbrella operator (OpenShift)) consume the same charts.
6. **Umbrella operator owns operands** — installs each dependency operator and creates their operands (Authorino, Limitador etc). The kuadrant-operator no longer performs these tasks.
7. **Extension operator compatibility** — extension operators that depend on kuadrant-operator (e.g., mcp-gateway-operator) independently remove their OLMv0 dependency declarations. The architecture supports folding them into the umbrella operator in a future iteration.
8. **Extensible** — supports future selective deployment of components based on which policies are enabled.

### OLMv1 Context

OLMv1 requires explicit creation of several resources per operator ([getting started guide](https://operator-framework.github.io/operator-controller/getting-started/olmv1_getting_started/)). Key design decisions: no automatic dependency installation, no cluster-admin permissions (user-provided ServiceAccounts), flexible bundle contents, and per-operator version control.

### Out of Scope

- Selective component deployment. The architecture supports it and has this in mind for a future iteration, but the initial implementation deploys all components.
- Folding extension operators (e.g., mcp-gateway-operator) into the umbrella operator initially. Extension operators independently remove their OLMv0 dependency declarations to become OLMv1-ready. Integrating them as umbrella operator subcharts is a future step.
- Not covered here is the exact layout of the charts and how they are made available to the umbrella operator (spike needed).

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### Prerequisites

- **OLMv0 to OLMv1 cluster upgrade path** — the cluster must support upgrading from OLMv0 to OLMv1. The migration strategy is designed so that the umbrella operator is introduced while the cluster is still running OLMv0, making the eventual OLMv1 upgrade a no-op from Kuadrant's perspective — the only remaining OLMv0-managed operator (the umbrella operator itself) has zero dependency declarations and migrates cleanly to a single ClusterExtension.
- **OLMv1 behaviour with existing dependency declarations is undefined** — OLMv1 [does not support](https://operator-framework.github.io/operator-controller/project/olmv1_limitations/) `olm.package.required`, `olm.gvk.required`, or `olm.constraint` declarations, but does not document what happens to operators already installed via OLMv0 that carry these declarations when a cluster is upgraded to OLMv1. The **assumption** is that existing operators and their operands remain in place, but dependency resolution and updates stop.

### Greenfield Install (OLMv1)

1. Create a ClusterCatalog pointing to the Kuadrant catalog image.
2. Create the umbrella operator's ClusterExtension (with namespace, ServiceAccount, RBAC). This is the only ClusterExtension the user creates.
3. OLMv1 deploys the umbrella operator and **all** CRDs (included in the umbrella operator bundle).
4. The umbrella operator renders the Helm charts and deploys all components: RBAC, operator Deployments, and operand instances.
5. Create the Kuadrant CR (owned by umbrella operator now see migration) (to configure) — kuadrant-operator begins reconciling policies.

Uninstall: delete the Kuadrant CR, then delete the ClusterExtension.

### Migration (from OLMv0 to umbrella operator)

The umbrella operator is introduced **while the cluster is still running OLMv0**. It is published as a standard OLMv0 operator (its own Subscription/CSV) with no `olm.package.required` or `olm.gvk.required` dependency declarations. On startup, it takes over deployment of all Kuadrant components by replacing the existing OLMv0-managed operators with Helm-rendered Deployments and takes over ownership of existing Kuadrant CR.

This eliminates OLMv0 dependency declarations before the OLMv1 transition, since [OLMv1 explicitly does not support automatic dependency installation](https://operator-framework.github.io/operator-controller/project/olmv1_design_decisions/). When OLMv1 arrives, the umbrella operator is the only operator to migrate — a single ClusterExtension with no dependencies.

#### Migration flow

The umbrella operator requires RBAC permissions to manage OLMv0 Subscriptions and ClusterServiceVersions. On startup it executes the following sequence:

1. **Delete the kuadrant-operator Subscription** — this is done first because the kuadrant-operator CSV declares `olm.package.required` dependencies. If dependency operator Subscriptions were deleted first, OLMv0's dependency resolver could recreate them while the kuadrant-operator CSV still exists.
2. **Delete the kuadrant-operator CSV** — OLMv0 removes the operator Deployment, RBAC, and ServiceAccount that the CSV owns. CRDs and user-created CRs (policies, Kuadrant CR) are not deleted.
3. **Deploy the new kuadrant-operator from Helm** — this version no longer reconciles operands (Authorino, Limitador instances) and **drops its watch on the Kuadrant CR**. The new controller starts up and resumes policy reconciliation only.
4. **Umbrella operator takes ownership of the Kuadrant CR** — now that kuadrant-operator is no longer watching the Kuadrant CR, the umbrella operator begins reconciling it (operand management, mTLS, observability). This must happen after step 3 to avoid two controllers competing over the same resources. See [Kuadrant CR Ownership Change](#kuadrant-cr-ownership-change).
5. **For each dependency operator** (authorino-operator, limitador-operator, dns-operator), sequentially:
   - Delete the OLMv0 Subscription
   - Delete the CSV — OLMv0 removes the operator Deployment
   - Deploy the new operator from Helm

Each step is idempotent — if the umbrella operator restarts mid-migration, it re-checks which Subscriptions and CSVs still exist and resumes from where it left off.

> Note while there maybe a need for some custom logic for the first version of the umbrella operator, ideally this would be removed in subsequent versions.

#### What survives the migration

| Resource | Behaviour |
|---|---|
| CRDs | Persist — OLMv0 never deletes CRDs |
| Operand CRs (Authorino, Limitador instances) | Persist — not owned by Subscriptions or CSVs |
| Operand processes | Continue running — deleting an operator's Deployment does not affect the operands it created |
| User-created CRs (policies, Kuadrant CR) | Persist |
| Data plane (auth, rate limiting) | No disruption — operands keep serving traffic |

#### Control plane gap

There is a brief window per operator (between CSV deletion and Helm Deployment rollout) where no controller is watching that operator's CRs. During this window:

- Existing policies continue to be enforced by the running operands
- New policy changes or CR updates are not reconciled until the replacement controller starts
- No user action is required — the umbrella operator handles the entire sequence

#### Post-migration state

After migration completes, the only OLMv0-managed resource on the cluster is the umbrella operator's own Subscription/CSV. All Kuadrant component operators are deployed and managed by the umbrella operator via Helm charts. The cluster is OLMv1-ready — the umbrella operator has zero dependency declarations and can be migrated to a single ClusterExtension.

### Steady-state upgrades

Steady-state upgrades follow the same pattern regardless of whether the cluster is running OLMv0 or OLMv1. OLM deploys the new umbrella operator image (containing updated charts and image references); the umbrella operator handles everything else.

1. OLM upgrades the umbrella operator bundle — this updates CRDs, ClusterRoles, and ClusterRoleBindings (OLM handles their lifecycle and safety checks).
2. The umbrella operator re-renders charts, detects drift, and applies changes in order:
   - **Dependency operators** (Authorino, Limitador, DNS) — updated and waited on until healthy.
   - **kuadrant-operator** last — as the policy reconciler, it may depend on new operand capabilities introduced by the dependency updates.

### Upgrade Testing

The full upgrade lifecycle must be validated end-to-end. The test scenario covers migration from OLMv0, transition to OLMv1, and data plane continuity throughout.

**Test environment:** Kind cluster and OpenShift with OLMv0, Kuadrant installed via OLMv0 Subscriptions, a Gateway with HTTPRoute/AuthPolicy/RateLimitPolicy applied, and active ingress traffic.

A background process sends requests through the gateway continuously across all phases, recording success/failure rates.

| Phase | Action | Validation |
|---|---|---|
| 1. Baseline | Record OLMv0 state (Subscriptions, CSVs, CRDs, pods, operand CRs) | All operators running, policies enforced (auth rejects unauthenticated, rate limits applied) |
| 2. Migration | Install umbrella operator via OLMv0 Subscription; wait for migration to complete | Only umbrella operator Subscription/CSV remains; all operators running via Helm; CRDs, operand CRs, Kuadrant CR persist; data plane uninterrupted |
| 3. OLMv1 transition | Upgrade cluster to OLMv1; create umbrella operator ClusterExtension | ClusterExtension healthy; all component operators and operands still running; data plane uninterrupted |
| 4. Steady-state upgrade | Publish new umbrella operator version; OLMv1 upgrades via ClusterExtension | All Deployments updated; no CRD regressions; data plane uninterrupted |

**Acceptance criteria:** zero data plane downtime, no policy enforcement gaps, full resource continuity (CRDs, CRs, and operand instances are never deleted and recreated).

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Managed Resources

Resource ownership is split between OLM and the umbrella operator. OLM manages the umbrella operator itself and all CRDs; the umbrella operator manages everything else.

#### Managed by OLM (via the umbrella operator's bundle/CSV)

| Resource | Purpose |
|---|---|
| CustomResourceDefinitions | All Kuadrant, Authorino, Limitador, and DNS CRDs. OLM handles install, upgrade, and CRD safety checks. |
| ClusterRoles | RBAC for each component operator (defined in the CSV's `clusterPermissions`). OLM also auto-generates aggregate roles (admin/edit/view/crdview) from CRDs. |
| ClusterRoleBindings | Binds ClusterRoles to component ServiceAccounts. Created by OLM from the CSV. |
| umbrella operator Deployment | The umbrella operator itself |

#### Managed by the umbrella operator

Resources are rendered from Helm charts and applied via controller-runtime. The umbrella operator is the sole owner — no other controller should manage these resources after migration.

| Resource | Purpose |
|---|---|
| Deployments | Operator controllers: kuadrant-operator, authorino-operator, limitador-operator, dns-operator |
| ServiceAccounts | Per-operator service identity |
| Roles / RoleBindings | Namespace-scoped RBAC (e.g., leader election) |
| Services | Metrics, webhooks |
| ConfigMaps | Operator configuration where applicable |

**Operand and configuration CRs:**

| Resource | Notes |
|---|---|
| Authorino CR (`authorinos.operator.authorino.kuadrant.io`) | Created by umbrella operator (previously kuadrant-operator) |
| Limitador CR (`limitadors.limitador.kuadrant.io`) | Created by umbrella operator (previously kuadrant-operator) |
| Kuadrant CR (`kuadrants.kuadrant.io`) | User-created. Reconciled by umbrella operator — see [Kuadrant CR Ownership Change](#kuadrant-cr-ownership-change) |

#### Not managed by the umbrella operator or OLM

| Resource | Owner |
|---|---|
| Policy CRs (AuthPolicy, RateLimitPolicy, DNSPolicy, TLSPolicy, etc.) | User-created, reconciled by kuadrant-operator |
| AuthConfig CRs | Created by kuadrant-operator, reconciled by Authorino |
| Operand Deployments / Pods (e.g., authorino, limitador-limitador) | Created by their respective operators (authorino-operator, limitador-operator) |
| Gateway, HTTPRoute | User-created, reconciled by gateway controller |

### Helm as Unified Manifest Source

The umbrella operator renders Helm charts at runtime using Helm's Go SDK with `ClientOnly=true` and `DryRun=true` — pure template rendering with no Helm release tracking. Rendered manifests are applied via controller-runtime. Charts are embedded in the umbrella operator image at build time.

This is the same pattern used by [cluster-olm-operator](https://github.com/openshift/cluster-olm-operator), which uses Helm as a templating engine rather than a package manager.

### Helm Chart Refactoring

The current kuadrant-operator chart (`charts/kuadrant-operator/`) is a monolithic kustomize-generated blob — a single `manifests.yaml` of ~14,500 lines produced by `kustomize build config/helm`. This must be refactored to support per-component rendering, ordered deployment, and operand creation.

#### Current state

- One chart with a single `templates/manifests.yaml` containing all operators, CRDs, and RBAC
- `Chart.yaml` declares subcharts for authorino-operator, limitador-operator, and dns-operator (pulled from `https://kuadrant.io/helm-charts/`)
- `values.yaml` is nearly empty — no configurable values exposed
- Dependency operator manifests pulled via kustomize remote refs (e.g., `github.com/Kuadrant/authorino-operator/config/deploy?ref=main`)
- No operand templates — operand creation is handled by kuadrant-operator at runtime

#### Required changes

1. **Replace kustomize generation with per-component Helm templates** — the kuadrant-operator chart needs templates instead of a kustomize dump. One template per component (containing its Deployment, Service, RBAC, ServiceAccount). CRDs are the exception: they must be separated (via Helm's `crds/` directory or a dedicated template) since operand CRs cannot be applied before their CRD is established.
2. **Configurable values.yaml** — image references, namespaces, replica counts, and resource limits exposed as values. The umbrella operator injects version-pinned values at render time; direct Helm users override via `--set` or values files.
3. **Operand templates** — the chart includes templates for operand CRs (Authorino, Limitador instances). These are created at install time on both paths, replacing the current runtime creation by kuadrant-operator.
4. **Dependency operator subcharts** — these already exist in their respective repos. The parent chart's `Chart.yaml` dependencies remain.
5. **Ordered rendering support** — the umbrella operator renders and applies subcharts sequentially: CRDs first, then dependency operators, then kuadrant-operator.

### Extension Operators

Extension operators complement kuadrant-operator rather than being consumed by it. They are **not managed by the umbrella operator in the initial implementation** — folding them in is a future step (see Out of Scope).

**mcp-gateway-operator** — currently declares `olm.package.required` on `kuadrant-operator >=1.4.3` and is installed from its own separate catalog (`mcp-gateway-catalog`). To become OLMv1-ready, mcp-gateway-operator must independently remove this dependency declaration from its CSV. It then migrates to OLMv1 as its own ClusterExtension with no dependencies — the same pattern as the umbrella operator.

The umbrella operator's architecture supports absorbing extension operators as subcharts in a future iteration. When that happens, the umbrella operator's migration flow would extend to clean up the extension operator's Subscription, CSV, and CatalogSource — the same pattern used for dependency operators.

### OLM Manifest Removal

With the umbrella operator owning deployment of all components via Helm charts, individual operators no longer need their own OLM packaging artifacts (CSV, bundle manifests, catalog entries, `bundle.Dockerfile`). These can optionally be removed from each operator repo, leaving only the Helm chart as the manifest source.

This is **optional per operator**. Operators such as authorino-operator and limitador-operator have standalone use cases outside of a Kuadrant installation and may choose to retain their own OLM bundles for independent installation via OLMv1. Operators that are only used as part of a Kuadrant deployment (e.g., kuadrant-operator, dns-operator) are stronger candidates for removal.

### Kuadrant CR Ownership Change

The Kuadrant CR moves from kuadrant-operator to the umbrella operator. Analysis of the [Kuadrant CR spec](https://github.com/Kuadrant/kuadrant-operator/blob/main/doc/reference/kuadrant.md) and reconciliation logic shows that none of its fields drive policy reconciliation — they are entirely concerned with operand management, operand configuration, and infrastructure wiring.

#### What the Kuadrant CR reconciler does today

| Spec field | Action | Category |
|---|---|---|
| (existence of CR) | Creates Authorino and Limitador operand CRs with owner references | Operand management |
| `spec.mtls.enable`, `spec.mtls.authorino`, `spec.mtls.limitador` | Configures PeerAuthentication for mTLS between gateway and operands | Operand configuration |
| `spec.observability.enable` | Creates ServiceMonitor and PodMonitor CRs for all components | Infrastructure wiring |
| `spec.observability.tracing.defaultEndpoint`, `spec.observability.tracing.insecure` | Sets tracing endpoint on Authorino and Limitador operand specs | Operand configuration |
| `spec.observability.dataPlane.defaultLevels`, `spec.observability.dataPlane.httpHeaderIdentifier` | Drives wasm-shim configuration via effective policy computation | Data plane configuration (see below) |
| `spec.components.developerPortal.enabled` | Creates/deletes Developer Portal Deployment | Operand management |

All policy reconciliation (AuthPolicy → AuthConfig, RateLimitPolicy → Limitador limits, etc.) is triggered by changes to policy CRs, not the Kuadrant CR.

#### What moves to the umbrella operator

The umbrella operator takes over reconciliation of the Kuadrant CR and all actions categorised as operand management, operand configuration, and infrastructure wiring. kuadrant-operator stops watching the Kuadrant CR entirely — it only watches policy CRs (AuthPolicy, RateLimitPolicy, DNSPolicy, TLSPolicy, TokenRateLimitPolicy).

#### Ownership handoff ordering

During migration, the ordering in the [migration flow](#migration-flow) is critical for the Kuadrant CR handoff:

1. The new kuadrant-operator is deployed first (migration step 3) — this version **drops** its watch on the Kuadrant CR and no longer reconciles operands.
2. Only after kuadrant-operator is running and healthy does the umbrella operator begin reconciling the Kuadrant CR itself — creating/adopting operand CRs, applying mTLS configuration, etc.

If this ordering is reversed (umbrella operator reconciles the Kuadrant CR while the old kuadrant-operator is still running), both controllers would compete over operand CRs and infrastructure resources. The migration flow prevents this by ensuring kuadrant-operator is replaced before the umbrella operator takes ownership.

> Note this is likely the first piece of work. Defining the umbrella operator and taking ownership of the kuadrant CR

#### Data plane observability configuration

The one field that crosses the boundary is `spec.observability.dataPlane`. Today this is read from the Kuadrant CR and fed into the policy reconciliation pipeline to configure wasm-shim trace levels and request correlation headers. Since kuadrant-operator will no longer watch the Kuadrant CR, this configuration must be projected to it by the umbrella operator.

**Approach:** When the umbrella operator reconciles the Kuadrant CR, it extracts the `spec.observability.dataPlane` configuration and projects it to kuadrant-operator via a ConfigMap (or environment variables on the kuadrant-operator Deployment). kuadrant-operator reads this configuration at startup or watches the projected ConfigMap for changes. This keeps the user-facing API unchanged — users still set `spec.observability.dataPlane` on the Kuadrant CR — while cleanly separating the reconciliation responsibilities.

#### API compatibility

No breaking API change is required — the Kuadrant CR spec remains the same. The behavioural change is:

- Operands exist before the Kuadrant CR is created (deployed by umbrella operator/Helm), rather than being created in response to it
- The umbrella operator reconciles the Kuadrant CR instead of kuadrant-operator
- kuadrant-operator becomes a pure policy reconciler with no operand or infrastructure concerns

For existing clusters, the Kuadrant CR persists through migration unchanged. The umbrella operator adopts it and begins reconciling it. kuadrant-operator's new version simply stops watching it.

### Version Pinning

Each umbrella operator release pins component versions via Helm values embedded at build time. Image references for all operators and operands are set through the charts' `values.yaml`, ensuring a single mechanism for version control across both install paths. OLM manages the sequencing of Umbrella Operator deployments to ensure an ordered upgrade.

### RBAC

The umbrella operator's ServiceAccount requires broad permissions. These are granted via the ClusterExtension's ServiceAccount. Each component operator's RBAC is part of its rendered chart.

### Failure Scenarios

- **Deployment rollout failure** — reports degraded status and blocks further upgrades (e.g., kuadrant-operator is not upgraded if a dependency failed).
- **Image pull failure** — surfaced via the Deployment's ImagePullBackOff condition.
- **CRD conflict** — CRDs are in the OLM bundle. If a CRD already exists (e.g., standalone Authorino), OLM surfaces the conflict via the ClusterExtension status. The umbrella operator does not apply CRDs directly.
- **Rollback** — (not currently an option with OLM and so out of scope currently)

# Drawbacks
[drawbacks]: #drawbacks

- **New operator to maintain** — new codebase, image, build pipeline, and release process.
- **Broad RBAC** — creating CRDs, ClusterRoles, and Deployments across namespaces is inherent to the pattern but is a wide permission set.
- **Version coordination** — all operators must be released and tested together; compatibility matrix must be maintained.
- **Two operators to debug** — users must distinguish umbrella operator problems (deployment) from kuadrant-operator problems (policy reconciliation).

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Why this design

- **Separation of concerns** — kuadrant-operator is simplified; deployment orchestration is cleanly separated.
- **No runtime OLMv1 dependency** — manages Deployments directly, reducing blast radius of OLMv1 issues.
- **Helm reuse** — existing charts from dependency operator repos are consumed as-is. No manifest duplication. Helm becomes more of a first class citizen
- **Proven pattern** — cluster-olm-operator uses Helm client-only rendering at scale in production.

### Alternatives considered

#### Single OLMv1 bundle containing all operators ("big CSV")

Instead of building a new umbrella operator, all component operators (kuadrant-operator, authorino-operator, limitador-operator, dns-operator) are packaged into a single OLM bundle. The CSV's `install.spec.deployments` section contains all four Deployments; `clusterPermissions` and `permissions` contain the RBAC for each operator's ServiceAccount; and all CRDs ship in the bundle's `manifests/` directory. OLM creates and owns everything — ServiceAccounts, RBAC, Deployments, CRDs.

**User experience (OLMv1):**

1. Create a ClusterCatalog pointing to the Kuadrant catalog image.
2. Create one ClusterExtension (with namespace, ServiceAccount, RBAC). The user-provided ServiceAccount needs permissions broad enough to cover all four operators.
3. OLMv1 deploys the single CSV — all operators and CRDs are installed as a unit.
4. Create the Kuadrant CR — kuadrant-operator creates operands and begins reconciling policies as it does today.

**Migration (OLMv0 → big CSV):** Publish a new version of the kuadrant-operator package that contains the big CSV. OLMv0 upgrades the existing Subscription, replacing the old CSV (which declared `olm.package.required` dependencies) with the new one that embeds all operators directly and has no dependency declarations. Dependency operator Subscriptions and CSVs become orphaned and can be cleaned up manually or via a migration job. When OLMv1 arrives, only one ClusterExtension is needed.

**Advantages:**

- **No new operator to build or maintain** — no umbrella operator codebase, image, or CI pipeline. Significantly less engineering effort.
- **Simpler mental model** — one CSV, one bundle, OLM manages everything.
- **OLM handles upgrades, health monitoring, and CRD safety checks** — no custom upgrade orchestration needed.
- **kuadrant-operator stays as-is** — no need to split out operand creation or change Kuadrant CR ownership. The existing reconciliation model is unchanged.
- **Identical install UX to the umbrella operator** — one ClusterExtension, one action.

**Drawbacks:**

- **All-or-nothing deployment** — every component is always installed. As Kuadrant grows in capabilities (more policy types, extension operators like mcp-gateway-operator), users cannot opt out of components they don't need. This becomes increasingly wasteful and may push toward the umbrella operator pattern later.
- **No deployment ordering** — OLM applies the bundle as a unit with no guaranteed startup order. kuadrant-operator must tolerate dependency operators not being ready yet (retry loops, degraded status). This is likely already the case but becomes a hard requirement.
- **No Helm as first-class install path** — Helm charts are not the manifest source; the CSV is. Upstream Kubernetes users without OLM get a different install mechanism, and chart maintenance diverges from the OLM bundle.
- **Coordinated releases remain** — all operators still ship together at the same version. Independent versioning is not possible.
- **Extension operator integration is rigid** — folding in new operators (e.g., mcp-gateway-operator) means rebuilding the CSV bundle rather than adding a subchart.
- **Same CRD ownership conflict** — standalone Authorino + big CSV hits the same OLMv1 single-ownership problem as the umbrella operator approach.

**Assessment:** The big CSV is a viable near-term approach that delivers the same single-action install UX with significantly less engineering effort. It is the pragmatic choice if selective deployment and Helm-first upstream installs are not immediate priorities. However, as Kuadrant's scope grows — more policy types, optional extensions, varied deployment profiles — the all-or-nothing nature of the big CSV is likely to become a constraint, and a transition to the umbrella operator pattern may be warranted.

# Future possibilities
[future-possibilities]: #future-possibilities

- **Selective component deployment** — a CR API (e.g., `KuadrantInstall`) mapping enabled policies to required components. Extension operators like mcp-gateway-operator would be natural candidates for opt-in deployment. Safety checks prevent removing components with active policy CRs.

# Prior art
[prior-art]: #prior-art

- **[cluster-olm-operator](https://github.com/openshift/cluster-olm-operator)** — primary reference. Manages OLMv1 sub-components using Helm charts rendered client-only, with static resource controllers and deployment controllers for applying the rendered manifests.
- **[OLMv0 dependency resolution](https://olm.operatorframework.io/docs/concepts/olm-architecture/dependency-resolution/)** — the current model being replaced. Resolves dependencies automatically via CRD-based declarations.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- **CRD ownership conflicts with standalone installations** — on OLMv0, if standalone Authorino is already installed, OLMv0's dependency resolver recognises the existing installation as satisfying `olm.package.required` and does not deploy a second instance. There is no CRD conflict. With the umbrella operator, the situation changes: the umbrella operator's bundle includes all Authorino CRDs. If a standalone Authorino ClusterExtension also exists, OLMv1's single-ownership model prevents two bundles from owning the same CRDs. 
