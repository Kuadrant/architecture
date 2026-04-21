# Standardize Release Branch Naming

- Feature Name: `release-branch-naming`
- Start Date: 2026-04-20
- RFC PR: [#170](https://github.com/Kuadrant/architecture/pull/170)
- Issue tracking: TBD
- Amends: [RFC 0008 — Kuadrant Release Process](0008-kuadrant-release-process.md)

# Summary
[summary]: #summary

Standardize release branch naming across all Kuadrant repositories to `release-X.Y` (minor-level, no `v` prefix). This aligns with OpenShift and Kubernetes conventions and replaces the current mix of `release-vX.Y`, `release-vX.Y.Z`, `release-X.Y`, and no branches at all, which complicates patch releases and prevents release automation from working uniformly across repos.

# Motivation
[motivation]: #motivation

RFC 0008 established that each component follows semantic versioning but did not prescribe a branch naming convention. Over time, each repo evolved its own pattern:

| Repo | Current pattern | Example |
|------|----------------|---------|
| kuadrant-operator | `release-vX.Y` | `release-v1.4` |
| console-plugin | `release-vX.Y` | `release-v0.3` |
| dns-operator | `release-X.Y` (mostly aligned) | `release-0.16`, `release-v0.17` |
| authorino-operator | `release-vX.Y.Z` | `release-v0.13.0` |
| limitador-operator | `release-vX.Y.Z` | `release-v0.13.0` |
| limitador | `release-vX.Y.Z` | `release-v1.5.0` |
| authorino | No release branches | Tags from `main` |
| wasm-shim | No release branches | Floating commit + tag |
| mcp-gateway | `release-X.Y.Z` | `release-0.6.0` |

This causes three problems:

1. **Patch releases require new branches under `release-vX.Y.Z`.** When cutting v0.13.1 for authorino-operator, a new `release-v0.13.1` branch must be created from the v0.13.0 tag. With a minor-level branch, v0.13.1 would just be a new tag on the existing `release-0.13` branch.

2. **No branch to backport to for authorino and wasm-shim.** Bug fixes land on `main` alongside features. A patch release either includes everything on `main` (violating semver) or requires retroactively creating a branch from the last tag and backporting individual fixes.

3. **Automation cannot assume a convention.** Release tooling must implement per-repo detection logic for branch names, `v` prefix presence, and version granularity. A uniform convention enables uniform automation.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Convention

All Kuadrant repositories that produce versioned release artifacts MUST use the following branch naming convention:

```text
release-X.Y
```

Where:
- `release-` is the literal prefix
- `X.Y` is the major.minor version (no patch component, no `v` prefix)

Branches use the bare version number (`release-1.4`), while tags retain the `v` prefix (`v1.4.0`). This matches the OpenShift and Kubernetes convention — branches identify the release line, tags identify the specific release.

## Branch lifecycle

1. **Creation**: A `release-X.Y` branch is created from `main` when the first release candidate for `vX.Y.0` is cut.

2. **Minor release**: The `vX.Y.0` tag is created on this branch after CI passes.

3. **Patch releases**: Bug fixes are backported from `main` to the `release-X.Y` branch via pull request (never direct push — CI must run on the PR). Each patch release (`vX.Y.1`, `vX.Y.2`, ...) is a new tag on the same branch. No new branch is created for patch releases. The release gate is automated: semver verification of changes + green CI. Regressions are fixed forward in the next patch.

4. **End of life**: The branch remains indefinitely for historical reference. It is not deleted.

## Tag format

Tags remain `vX.Y.Z` (unchanged from current practice). The exception is limitador, which uses dual tags `crate-vX.Y.Z` and `server-vX.Y.Z` — this is unaffected by the branch naming change.

## Examples

```text
main ──●──●──●──●──●──●──●──●──●──●──●──
        \                        \
         release-1.4              release-1.5
          │   │   │                │
         v1.4.0  v1.4.1          v1.5.0
               v1.4.2
```

### Patch release workflow

To release `v0.13.1` of authorino-operator:

```bash
# Create a backport branch from the release branch
git checkout release-0.13
git checkout -b backport-fix-xyz
git cherry-pick <bug-fix-commits>
git push origin backport-fix-xyz
# Open a PR targeting release-0.13 — CI runs on the PR
gh pr create --base release-0.13 --title "Backport: fix XYZ"
# After CI passes and PR merges, tag the release
git checkout release-0.13 && git pull
git tag -a v0.13.1 -m "v0.13.1" -s
git push origin v0.13.1
```

Compare with the current approach:
```bash
# Must create a new branch for every patch release
git checkout -b release-v0.13.1 v0.13.0
git cherry-pick <bug-fix-commits>
git push origin release-v0.13.1
# Same PR + tag flow, but with a new release branch each time
```

The key difference: with `release-X.Y`, the branch already exists and accumulates patches. With `release-X.Y.Z`, a new branch is needed for every patch.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Migration plan

No existing branches are renamed or deleted. The migration happens naturally: when each repo cuts its **next minor release**, it uses the new convention. Old branches remain for maintaining existing release lines.

| Repo | Current | Next minor action | Effort |
|------|---------|-------------------|--------|
| dns-operator | `release-X.Y` (already aligned, no `v`) | None | None |
| kuadrant-operator | `release-vX.Y` | Drop `v`: create `release-1.5` | Update RELEASE.md, workflows |
| console-plugin | `release-vX.Y` | Drop `v`: create `release-0.4` | Update RELEASE.md, workflows |
| authorino-operator | `release-vX.Y.Z` | Create `release-0.14` instead of `release-v0.14.0` | Update RELEASE.md, workflows |
| limitador-operator | `release-vX.Y.Z` | Create `release-0.14` instead of `release-v0.14.0` | Update RELEASE.md, workflows |
| limitador | `release-vX.Y.Z` | Create `release-1.6` instead of `release-v1.6.0` | Update RELEASE.md, workflows |
| authorino | No branches | Create `release-0.23` for next minor | Update RELEASE.md |
| wasm-shim | No branches | Create `release-0.14` for next minor | Update RELEASE.md |
| mcp-gateway | `release-X.Y.Z` | Create `release-0.7` instead of `release-0.7.0` | Update RELEASING.md, workflows |
| policy-machinery | No branches | On demand only — create `release-X.Y` when a backport is needed | None until needed |

### Per-repo changes

For repos migrating from `release-vX.Y.Z` or no branches:

1. Update `RELEASE.md` to document the new branch naming convention.
2. Update any GitHub Actions workflows that reference the branch name pattern (e.g., `release-v*.*.*` glob → `release-*.*`).
3. For repos with GitHub Actions that auto-create release branches: update the branch name template.

### Backward compatibility during transition

During the transition period (until all repos have cut at least one minor release with the new convention), release tooling should detect branches dynamically:

```bash
# Example: find the release branch for minor version 0.13
# Replace X.Y with actual version numbers
git branch -r | grep -E "origin/release-v?0\.13($|\.)" | head -1
```

This handles `release-0.13`, `release-v0.13`, and `release-v0.13.0` transparently.

## Scope

This RFC applies to all repositories listed in `kuadrant-operator/release.yaml` as dependencies, plus kuadrant-operator itself, the operand repos (authorino, limitador), and library dependencies that are pinned to specific minors per kuadrant release line.

**In scope:**
- All repos in `release.yaml`: authorino-operator, limitador-operator, dns-operator, wasm-shim, console-plugin, developer-portal-controller, mcp-gateway
- Orchestrator: kuadrant-operator
- Operands: authorino, limitador

**On demand only:**
- `policy-machinery` — a Go library compiled into kuadrant-operator, not a runtime component.
  - **Versioning**: Different kuadrant minors pin different policy-machinery minors (e.g., v0.6.x for kuadrant 1.3, v0.7.x for 1.4).
  - **Patch strategy**: Stay on the pinned minor by default. Prefer bumping to the latest patch of the same minor. If the fix only exists in a newer minor and backporting is too costly, a minor bump is acceptable if CI passes.
  - **Branches**: Create a release branch (e.g., `release-0.6`) only when a backport to an older minor is actually needed. Do not create branches preemptively.
# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why `release-X.Y` over `release-X.Y.Z`

| Criterion | `release-X.Y` | `release-X.Y.Z` |
|-----------|:-:|:-:|
| Patch release: new branch needed? | No | Yes |
| Backport target is obvious? | Yes — one branch per minor | Must find or create the right patch branch |
| Branch count growth | Linear with minors | Linear with every release |
| Matches OpenShift/Kubernetes | Yes | No |

## Why no `v` prefix on branches

Tags use `v` (`v1.4.0`) because that's the Go module convention and the semver convention for identifying a specific release. Branches identify a release *line*, not a specific release — the `v` adds no information and diverges from OpenShift (`release-4.20`) and Kubernetes (`release-1.30`). Aligning with these conventions reduces friction for contributors who work across projects.

dns-operator already uses this convention (`release-0.16`). kuadrant-operator's earliest release branch (`release-0.9`) also used it before later branches added `v`.

## Why not `release/X.Y` (slash separator)

Some projects (e.g., Envoy Gateway) use `release/X.Y` with a directory-style separator. While valid, it differs from OpenShift/Kubernetes convention and every existing Kuadrant branch.

## Why not tag-only (no branches)

authorino and wasm-shim currently use this approach. It works for minor releases but makes patch releases difficult — you need somewhere to backport bug fixes to. Creating the branch retroactively is possible but error-prone and loses the history of what was intentionally included.

# Prior art
[prior-art]: #prior-art

- **OpenShift**: Uses `release-X.Y` branches (e.g., `release-4.20`, `release-5.1`). Documented in [openshift/enhancements CONVENTIONS.md](https://github.com/openshift/enhancements/blob/master/CONVENTIONS.md): "release-4.# for maintenance branches". Branch maintenance is automated; breaking convention makes repos harder to support.
- **Kubernetes**: Uses `release-X.Y` branches (e.g., `release-1.30`). Patch releases are tags on the same branch.
- **Istio**: Uses `release-X.Y` branches. Same pattern.
- **Envoy Gateway**: Uses `release/vX.Y` branches (slash separator, `v` prefix).
- **Operator SDK**: Uses `vX.Y.x` branches.

The `release-X.Y` pattern directly matches OpenShift and Kubernetes — the two ecosystems Kuadrant operates within.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Should GitHub Actions release workflows be updated as part of this RFC, or tracked as follow-up issues per repo?
- How many minor versions should be maintained simultaneously? (e.g., N-1 like OpenShift)
- Should backport automation (Kubernetes-style `/cherry-pick release-X.Y` bot that creates backport PRs) be adopted to manage the cross-repo cascade when supporting multiple minors?

# Future possibilities
[future-possibilities]: #future-possibilities

With a uniform branch convention in place:
- **Automated patch releases**: A tool can verify changes are patch-safe (semver verification of actual code diffs), check CI is green, and tag releases across all repos without manual intervention. The release gate is automated verification + CI, not manual QE sign-off. Regressions are fixed forward.
- **Multi-minor support**: Multiple `release-X.Y` branches can be maintained simultaneously. Each kuadrant release branch's `release.yaml` maps to specific dependency minors, enabling coordinated patch releases across supported versions (e.g., patching kuadrant 1.3 and 1.4 in one pass).
- **Backport automation**: Kubernetes-style tooling (`/cherry-pick release-X.Y` comment on merged PRs) can automate backport PR creation across the cross-repo cascade when a bug fix needs to land on multiple release branches.
- **Uniform branch protection**: Rules can be applied using a single `release-*` glob pattern across all repos.
