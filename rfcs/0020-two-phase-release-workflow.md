# Two-Phase Release Workflow

- Feature Name: `two_phase_release_workflow`
- Start Date: 2026-06-04
- RFC PR: [Kuadrant/architecture#178](https://github.com/Kuadrant/architecture/pull/178)
- Issue tracking: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/issues/0000)

# Summary
[summary]: #summary

This RFC introduces a two-phase release workflow model for all Kuadrant component repositories.
Every release is split into two distinct GitHub Actions workflows: a **pre-release** workflow that performs code changes and opens a pull request to the release branch, and a **release** workflow that tests, builds artifacts, and creates the GitHub Release as its final step.
A `release.yaml` file is added to every component repository to serve as the machine-readable source of truth for version and dependency information.
This model replaces the single-workflow release approach and evolves the process described in [RFC 0008](0008-kuadrant-release-process.md).

# Motivation
[motivation]: #motivation

The Kuadrant v1.5.0 release exposed systemic failures in the single-workflow release model:

- **False success reporting.** 
Release workflows reported success before downstream workflows completed. 
In one case, a release workflow succeeded, images were built and published, but tests that ran afterward failed - leaving published artifacts that did not pass validation.

- **Hidden workflow chains.** 
A single release could trigger cascading workflows across multiple repositories in a fire-and-forget pattern.
Downstream failures were invisible to the person who triggered the release.

- **Silent downstream failures.**
Helm chart synchronization failed for every component because branch protection rules on the target repository required PR-based changes, but workflows attempted direct pushes.
This class of failure was invisible to the release trigger.

- **Code changes during release violate policy.**
Red Hat security policy (Segregation of Duties) requires all changes to protected branches to go through pull requests with an approver who is not the author.
Single-workflow releases that commit directly to release branches violate this constraint.

- **No dependency gating.**
There was no mechanism to prevent an operator from releasing before its operand releases existed. 
This led to operators shipping with references to images that did not yet exist.

These failures share a root cause: the single-workflow model conflates code changes with artifact publishing, provides no human review gate, and has no visibility into downstream outcomes.
The two-phase model addresses all of these by separating concerns, introducing a PR-based approval gate between phases, and requiring the GitHub Release to be the final artifact — meaning a release either fully succeeds or does not exist.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## The two workflows

A compliant repository has exactly two release-related workflows and one CI validation check:

### Pre-release workflow

The pre-release workflow is triggered manually.
It takes a target version as input, makes all necessary code changes (version bumps, manifest regeneration, dependency updates), and opens a pull request against the release branch.

The workflow does **not** create tags or publish tagged (versioned) artifacts.
It may build images based on the commit SHA for testing purposes.
Its only durable outputs are a release branch (created if it does not already exist) and a pull request containing all version-related code changes.

After the workflow completes, a human reviews and approves the pull request.
CI checks on the pull request validate that the code is correct, and a version gate check confirms that all declared dependencies have been released.
Only after the pull request is merged does the release proceed to the second phase.

### Release workflow

The release workflow is triggered manually after the pre-release pull request has been merged.
It takes the release branch as input and reads the target version from the `release.yaml` file on that branch.

The release workflow follows a strict ordering:

1. **Test** - Run smoke tests, linting, and validation checks.
If any test fails, the workflow stops. No artifacts are created.
2. **Tag** - Create and push a git tag from the release branch HEAD.
3. **Build artifacts** - Build and publish all release artifacts for the component. 
Where artifacts have internal dependencies, they must be built in the correct order.
Artifacts that are independent of each other may be built in parallel.
4. **Create GitHub Release** - This is the **last** step.
The GitHub Release is created only after all preceding jobs succeed. 
If any artifact build fails, no GitHub Release is created, and the release is considered failed.

The release workflow must **not** make code changes or create commits on the release branch.
It must not directly trigger workflows in other repositories, though it may create pull requests to external repositories (e.g., for helm chart synchronization).
The only mutation to the git repository is the tag.

### Version gate (CI check)

A CI check runs on pull requests to release branches when `release.yaml` is modified.
It enforces the following rules:

1. On release branches, the version in `release.yaml` must not be `0.0.0`.
2. On release branches, dependency versions must not be `0.0.0` - they must target a concrete release version.
3. For each dependency, the corresponding GitHub Release with that version must exist in the organization.

This check prevents operators from releasing before their operand releases are available.
It fails fast on the pre-release pull request rather than discovering missing dependencies during the release workflow.

## The release.yaml file

Every component repository contains a `release.yaml` file at the repository root.
This file declares the component's version and, where applicable, the versions of its dependencies.

On the `main` branch, `release.yaml` looks like:

```yaml
component-name:
  version: "0.0.0"

dependencies:
  operand-a: "0.0.0"
  operand-b: "0.0.0"
```

The version `0.0.0` is a sentinel value meaning "targets latest / under active development." This is the permanent state on `main`.

On a release branch, after the pre-release workflow runs, `release.yaml` is updated to concrete versions:

```yaml
component-name:
  version: "1.5.0"

dependencies:
  operand-a: "2.4.0"
  operand-b: "0.8.0"
```

The `dependencies` section is optional. 
Leaf components (operands with no Kuadrant dependencies) omit it.
Operators declare their operand dependencies, so the version gate can verify availability before the release proceeds.

## Release branch naming

Release branches follow the convention `release-X.Y` where `X.Y` is the minor version, without a `v` prefix.
For example, version `1.5.0` releases from branch `release-1.5`, and a subsequent patch `1.5.1` releases from the same branch.
This convention is defined in detail in [RFC 0018](https://github.com/Kuadrant/architecture/blob/main/rfcs/0018-rlease-branch-naming.md).

Using minor-level branches means patch releases do not require new branches - they reuse the existing release branch with an updated version in `release.yaml`.
This is critical for automation: a consistent branch naming scheme allows tooling to discover and operate on release branches without per-component configuration.

## What this model guarantees

- **No phantom releases.** 
If any step fails, no GitHub Release exists.
The release either fully succeeds or does not happen.
- **Tests before artifacts.**
Smoke tests must pass before any image is built or published.
No more published images that fail validation.
- **Human review gate.** 
All code changes are reviewed through a standard pull request before release proceeds.
This satisfies Red Hat's Segregation of Duties policy.
- **Dependency safety.**
The version gate prevents operators from releasing before their operands, eliminating references to non-existent images.
- **Contained releases.**
The release workflow does not directly trigger workflows in other repositories. 
It may create PRs to external repositories, but each component's release is self-contained and observable.
- **Branch protection compliance.**
All code changes go through PRs.
The release workflow only creates a tag - no commits to protected branches.

## Workflow for a release engineer

A typical minor release follows these steps:

1. Trigger the pre-release workflow with the target version (e.g., `1.5.0`).
2. The workflow creates the release branch `release-1.5` (if needed), updates `release.yaml`, runs component-specific preparation steps, and opens a PR.
3. Review the PR. CI runs tests and the version gate check. 
If the component is an operator, the version gate confirms operand releases exist.
4. Merge the PR.
5. Trigger the release workflow with the release branch (`release-1.5`).
6. The workflow reads the version from `release.yaml`, runs smoke tests, tags, builds all artifacts, and creates the GitHub Release.

For a patch release, the process is the same except the release branch already exists and `release.yaml` is updated to the patch version (e.g., `1.5.1`).

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Compliant repository specification

A repository is compliant with this RFC when it meets all the following criteria:

### Required files

| File | Location | Purpose |
|------|----------|---------|
| `release.yaml` | Repository root | Version and dependency declaration |
| `pre-release.yaml` | `.github/workflows/` | Pre-release workflow |
| `release.yaml` | `.github/workflows/` | Release workflow |
| `version-gate.yaml` | `.github/workflows/` | CI validation for release PRs |

### release.yaml specification

The `release.yaml` file at the repository root is a YAML document with the following structure:

```yaml
<component-name>:
  version: "<semver>"

dependencies:
  <dependency-name>: "<semver>"
  ...
```

**Fields:**

- `<component-name>` (required): The name of the component. Must match the repository name or the component's canonical identifier.
- `version` (required): A semantic version string (`X.Y.Z`), or `0.0.0` on the `main` branch to indicate active development.
- `dependencies` (optional): A map of dependency names to their required versions. Each dependency name must correspond to a repository in the Kuadrant GitHub organization.
The version `0.0.0` indicates "targets latest" and is not validated by the version gate.

**Invariants:**

- On `main`, the version is always `0.0.0` and all dependency versions are `0.0.0`.
- On release branches, the version must not be `0.0.0`.
- On release branches, dependency versions must not be `0.0.0` - they must be concrete released versions.
The version gate will verify that a GitHub Release exists for each dependency.
- Where the component's build system maintains its own version variables (e.g., Makefile variables), these must reflect the values in `release.yaml` on release branches.
The pre-release workflow is responsible for ensuring this consistency.

### Pre-release workflow specification

**Trigger:** `workflow_dispatch` with the following inputs:

| Input | Required | Description |
|-------|----------|-------------|
| `version` | Yes | Target semantic version (e.g., `1.5.0`) |
| `source-branch` | No | Branch to base the pre-release changes on. Defaults to `main`. For patch releases, this would be a branch containing cherry-picked fixes (e.g., a branch created from the release branch with backported changes). |
| `<operand-name>` | No | Version for a specific operand dependency (for operator repos). One input field per operand, improving the workflow dispatch form UX. For example, an operator depending on `limitador` and `authorino` would have separate `limitador-version` and `authorino-version` input fields. If left blank, the version currently set in `release.yaml` is used. |

**Permissions:** `contents: write`, `pull-requests: write`

**Jobs:**

1. **setup**
   - Validate the `version` input is valid semver.
   - Derive the release branch name from the version (e.g., `1.5.0` → `release-1.5`).
   - Create the release branch from `main` if it does not exist.

2. **prepare-release**
   - Create a working branch (e.g., `pre-release-v1.5.0`) from the source branch.
   - Update `release.yaml` with the target version and dependency versions.
   - Execute component-specific pre-release steps, deriving build-system version variables from `release.yaml` (e.g., generating override files, updating version constants in build files, regenerating manifests, updating lock files).
   - Commit and push all changes.

3. **open-pr**
   - Open a pull request from the working branch to the release branch.
   - The PR title follows the convention: `chore: prepare release vX.Y.Z`.

**Constraints:**

- The workflow must not create tags.
- The workflow may build images based on the commit SHA for testing purposes, but must not publish tagged (versioned) artifacts.
- The workflow must not trigger other workflows in other repositories.
- All code changes must be committed to the working branch, never directly to the release branch.

### Release workflow specification

**Trigger:** `workflow_dispatch` with the following inputs:

| Input | Required | Description |
|-------|----------|-------------|
| `release-branch` | Yes | The release branch to release from (e.g., `release-1.5`) |

**Permissions:** `contents: write`

**Job flow:**

```
read-version → smoke-tests → tag → build artifacts → create-release
```

The release workflow is a linear pipeline of five phases.
Each phase must complete successfully before the next begins.

**Phase specifications:**

1. **read-version** — Parse `release.yaml` to extract the version and any other version-related metadata needed by downstream jobs.
This phase must also verify that a GitHub Release for this version does not already exist.
If one does, the workflow fails immediately to prevent duplicate releases.

2. **smoke-tests** — Run the component's test suite: linting, formatting checks, unit tests, and any other checks that validate the release branch is in a releasable state.

3. **tag** — Create and push a git tag (`vX.Y.Z`) from the release branch HEAD.
The tag is the only git mutation the release workflow performs.

4. **build artifacts** — Build and publish all release artifacts for the component.
What artifacts are produced varies by component type — container images, helm charts, binaries, OLM bundles, catalog images, or any combination.
Where artifacts have internal dependencies (e.g., a bundle image depends on the operator image), they must be built in the correct order.
Artifacts that are independent of each other may be built in parallel.

5. **create-release** — Create the GitHub Release with auto-generated release notes and attach any built artifacts.
This must depend on **all** artifact build jobs.
It is always the final step in the workflow.

**Constraints:**

- The workflow must not make code changes or create commits on the release branch.
- The workflow must not directly trigger workflows in other repositories.
It may create pull requests to external repositories (e.g., for helm chart synchronization); if those PRs trigger CI workflows in the target repository, that is the responsibility of the target repository.
- Tests must pass before any artifacts are built.
- The GitHub Release must be the last artifact created.
If any preceding job fails, no GitHub Release is created.

### Version gate specification

**Trigger:** `pull_request` targeting `release-*` branches, filtered to changes in `release.yaml`.

**Permissions:** `contents: read`

**Validation rules:**

1. On a release branch, the version in `release.yaml` must not be `0.0.0`.
2. On a release branch, dependency versions must not be `0.0.0` — they must target a concrete release version.
3. For each dependency, verify that a GitHub Release with tag `v<version>` exists in the `<dependency-name>` repository within the Kuadrant organization.

**Failure behavior:** The check fails with a clear message indicating which dependency release is missing.
This blocks the pre-release PR from being merged until the dependency is released.


## Multi-component release orchestration

The two-phase model is designed to support orchestrated releases across the full Kuadrant component graph.
While the specifics of the orchestration tooling are outside the scope of this RFC, the model enables multi-component releases through the following properties:

- **Pre-release workflows can run for all components in parallel.**
Each opens a PR to its own release branch.
For operators, the version gate check on the PR will initially fail because operand releases do not yet exist.

- **Operand releases unblock operator PRs.**
When an operand's release workflow completes and creates a GitHub Release, the version gate check on dependent operator PRs begins to pass.
This creates a natural dependency-respecting release order without tight coupling between repositories.

- **Release workflows are triggered sequentially by dependency order.**
Operands (leaf nodes) are released first.
As their GitHub Releases appear, operator PRs become mergeable.
Operators are released after their operand dependencies are satisfied.

- **The `release.yaml` file is the integration point.**
Orchestration tooling reads `release.yaml` to discover the dependency graph and uses the version gate's pass/fail status to determine when each component is ready for its release phase.

## Adapting the model to a component

Each component repository implements the two workflows with component-specific build logic while preserving the workflow structure, job ordering, and constraints defined in this RFC.

Component-specific areas include:

- **Pre-release steps:** What code changes are needed to prepare a release (e.g., updating version constants in build files, regenerating manifests, updating lock files).
Where the build system maintains its own version variables, these should be derived from the `release.yaml` to preserve the single source of truth.
- **Smoke tests:** What validation is appropriate for the component (e.g., `go vet`, `cargo clippy`, `make bundle` validation, OLM bundle checks).
- **Artifact builds:** What images, charts, or binaries the component produces.

The reference implementation at [kuadrant-labs/workflow_layout](https://github.com/kuadrant-labs/workflow_layout) provides a complete template with placeholder markers for component-specific sections.
Components should use this as their starting point and replace the placeholder sections with their actual build logic.

# Drawbacks
[drawbacks]: #drawbacks

- **Two workflows instead of one increases operational surface area.**
Release engineers must trigger two workflows and wait for a PR review between them.
This is intentionally slower than a single-click release.

- **Human approval gate adds latency.**
The PR between pre-release and release requires human review and approval.
This is a tradeoff: the model prioritizes correctness and auditability over speed.

- **Duplicated workflow structure across repositories.**
Each repository carries its own copy of the workflow files.
While the structure is standardized, updates to the workflow model require changes across all component repositories.
A future centralization of shared actions could reduce this burden but is out of scope for this RFC.

- **Version gate relies on GitHub API availability.**
The version gate check queries the GitHub API to verify dependency releases exist.
If the API is unavailable or rate-limited, the check will fail, blocking the PR.
Using authenticated API calls mitigates rate limiting.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why two phases instead of one?

The single-workflow model conflates two fundamentally different operations: making code changes and publishing artifacts.
Separating them provides a natural point for human review, allows CI to validate the release branch before artifacts are built, and ensures that policy-required PR approvals are part of the release flow rather than bypassed by it.

## Why is the GitHub Release the last artifact?

The GitHub Release is the public-facing signal that a version exists.
If it is created before all artifacts are ready, consumers may attempt to use a version whose images, charts, or binaries are incomplete or missing.
By making it the final step, the model guarantees that when a release is visible, it is complete.
Conversely, if any build step fails, no release is visible — the failure state is clean.

## Why a file-based version declaration (release.yaml)?

Alternatives considered:

- **Git tags as the source of truth:**
Tags are created during the release workflow, not before.
The pre-release phase needs a version declaration before the tag exists.
Additionally, tags cannot express dependency relationships.

- **Version constants in source code only:**
Different languages declare versions differently (Cargo.toml, Makefile variables, package.json).
A unified `release.yaml` provides a single, language-agnostic location that tooling can read without understanding the component's build system.
Build-system-specific version declarations remain for developer-facing build targets, but on release branches they are derived from `release.yaml` by the pre-release workflow rather than maintained independently.

- **No dependency declaration:**
Without dependency information in a machine-readable format, there is no way to automatically gate operator releases on operand availability.
The version gate check would not be possible.

## Why manual workflow dispatch instead of event-driven triggers?

Event-driven triggers (e.g., on tag push, on release creation) create invisible chains of workflow executions across repositories.
When one fails, the failure is disconnected from the original trigger.
Manual dispatch keeps each release step explicit, observable, and attributable.
It also allows orchestration tooling to control the execution order rather than relying on event propagation timing.

## What is the impact of not doing this?

Without this change, releases will continue to produce false success signals, violate branch protection policies, and fail silently in downstream steps.
As the number of supported versions grows (requiring patch releases across multiple release lines), the fragility of the single-workflow model will compound.

# Prior art
[prior-art]: #prior-art

- **RFC 0008 — Kuadrant Release Process.**
The existing release process RFC defines the broader framework: versioning, cadence, QA handover, communication channels, and artifact registries.
This RFC evolves the workflow mechanics within that framework without replacing its other concerns.
RFC 0008's observation that "once the release process is accepted and battle-tested, we could aim to automate the process as much as possible" directly motivates the machine-readable `release.yaml` and standardized workflow structure introduced here.

- **Kubernetes release process.**
The Kubernetes project separates release preparation (branch creation, cherry-pick management) from the release cut itself.
The `krel` tool orchestrates this two-phase process, and release branches follow the `release-X.Y` convention.
Kuadrant's model draws on this separation of concerns and branch naming.

- **Operator Framework release practices.**
OLM-based operators commonly build images in a chain (operator → bundle → catalog).
This RFC formalizes that chain as explicit job dependencies in the release workflow rather than relying on separate, loosely-coupled workflows.

- **kuadrant-labs/workflow_layout.**
A [reference implementation](https://github.com/kuadrant-labs/workflow_layout) was built to validate the two-phase model.
It contains working pre-release, release, and version-gate workflows with placeholder markers for component-specific logic, along with example releases, pull requests, and workflow runs demonstrating the model in practice.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- **Smoke test baseline.**
What is the minimum set of smoke tests each component type must pass in the release workflow before artifacts are built?
This will be resolved during implementation as each component adopts the model.

- **Helm chart PR mechanics.**
The release workflow must open a PR to `kuadrant/helm-charts` instead of pushing directly.
The exact mechanism (cross-repo token, GitHub App, or other approach) needs to be determined during implementation.

- **Rollback procedure.**
If a release workflow partially succeeds (e.g., tag is pushed, but an image build fails), what is the recovery procedure?
The model ensures no GitHub Release is created, but the tag and any published images may need manual cleanup.

- **release.yaml schema evolution.**
As the model matures, additional fields may be needed in `release.yaml` (e.g., pre-release identifiers, build metadata).
The initial schema should be treated as a minimum viable specification.

# Future possibilities
[future-possibilities]: #future-possibilities

- **Automated patch release pipeline.**
The `release.yaml` file and version gate provide the foundation for automated patch releases.
A system could walk the component dependency graph, detect changes on release branches, and trigger pre-release and release workflows in dependency order — with human approval at the PR gate.
This is actively being designed but is independent of this RFC.

- **Centralized GitHub Actions.**
The duplicated workflow structure across repositories could be consolidated into shared composite actions and reusable workflows in a central repository.
This would reduce the maintenance burden of updating workflow logic across all components.
This is a separate initiative and does not affect the model defined here.

- **Release branch naming standardization.**
Full standardization of the `release-X.Y` branch naming convention across all repositories, including migration of existing branches that use different conventions, is tracked as a companion effort.
See [RFC 0018](https://github.com/Kuadrant/architecture/blob/main/rfcs/0018-release-branch-naming.md) for details.

- **Automated version gate expansion.**
The version gate currently checks for GitHub Release existence.
It could be extended to verify image availability in container registries, helm chart presence, or other artifact-specific checks.

- **Release status dashboard.**
With standardized `release.yaml` files across all components, a dashboard could aggregate release status across the entire Kuadrant suite — showing which components have pending pre-release PRs, which are ready for release, and which have completed releases for a given version.
