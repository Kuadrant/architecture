# Graduate Components to v1.0

- Feature Name: `semver-v1-graduation`
- Start Date: 2026-04-21
- RFC PR: TBD
- Issue tracking: TBD

# Summary
[summary]: #summary

Graduate all Kuadrant components still on 0.x to v1.0.0, signaling API stability and making semantic versioning meaningful. These components are production-ready and depended on by kuadrant-operator v1.x — the 0.x versions are inertia, not a reflection of stability.

# Motivation
[motivation]: #motivation

The Kuadrant suite ships as v1.4, yet most of its components are still versioned at 0.x:

| Component | Current | Stable? |
|-----------|:-------:|:-------:|
| kuadrant-operator | v1.4.3 | Already v1.x |
| limitador | v2.3.0 | Already v2.x |
| authorino | v0.25.0 | 25 minor releases, production use |
| authorino-operator | v0.24.0 | 24 minor releases, production use |
| limitador-operator | v0.17.1 | 17 minor releases, production use |
| dns-operator | v0.16.1 | 16 minor releases, production use |
| wasm-shim | v0.12.3 | 12 minor releases, production use |
| console-plugin | v0.3.6 | Production use |
| policy-machinery | v0.8.0 | Compiled into kuadrant-operator |

Under [semver](https://semver.org/), 0.x means "initial development — anything may change at any time." A minor bump (0.16→0.17) carries no backward-compatibility promise. This creates three problems:

1. **Semver is meaningless for automation.** Tooling that classifies a minor bump as "additive feature" and a major bump as "breaking change" cannot trust those signals at 0.x. Patch release verification has to special-case every 0.x component.

2. **The version contradicts reality.** These components have stable APIs and are depended on by kuadrant-operator v1.x in production. Calling them 0.x understates their maturity and may discourage adoption.

3. **Go module versioning.** For Go libraries at v2+, the module path must include a `/v2` suffix per Go module conventions. authorino (v0.25) and the operators (v0.17–v0.24) are approaching version numbers where a 1.0 graduation is cleaner than eventually hitting v2.0 directly from 0.x.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## What changes

Each 0.x component bumps its next minor release to v1.0.0 instead of v0.N+1.0. From that point, semver rules are enforced:
- PATCH (1.0.x): bug fixes only
- MINOR (1.x.0): new features, backward compatible
- MAJOR (2.0.0): breaking changes

## What does NOT change

- No API changes, no code changes, no behavioral changes. The v1.0.0 release of each component is the same code that would have been the next 0.x release.
- Tag format remains `vX.Y.Z`.
- Release branch convention follows [RFC 0015](0015-release-branch-naming.md): `release-1.0` for each component.
- kuadrant-operator (already v1.x) and limitador (already v2.x) are unaffected.

## Graduation list

| Component | Current | Graduates to | Notes |
|-----------|:-------:|:------------:|-------|
| authorino | v0.25.0 | v1.0.0 | |
| authorino-operator | v0.24.0 | v1.0.0 | |
| limitador-operator | v0.17.1 | v1.0.0 | |
| dns-operator | v0.16.1 | v1.0.0 | |
| wasm-shim | v0.12.3 | v1.0.0 | |
| console-plugin | v0.3.6 | v1.0.0 | |
| policy-machinery | v0.8.0 | v1.0.0 | Go module path stays `github.com/kuadrant/policy-machinery` (no `/v2` needed since graduating from 0.x to 1.x) |

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Per-component changes

For each graduating component:

1. **Release branch**: Create `release-1.0` (per RFC 0015 convention).
2. **Version files**: Update version to `1.0.0` (Cargo.toml, package.json, Makefile, etc.).
3. **RELEASE.md**: Update to reflect new versioning.
4. **Go modules** (authorino, authorino-operator, limitador-operator, dns-operator, policy-machinery): No module path change needed — the Go spec only requires a `/vN` suffix for v2+.
5. **Consumers**: Update `go.mod` / `Cargo.toml` / `release.yaml` references in kuadrant-operator to use v1.0.0.

## Timing

All components should graduate in the **same kuadrant release cycle** to avoid a mixed state where some dependencies are 0.x and others are 1.x within the same suite release. The kuadrant-operator `release.yaml` for that cycle would reference v1.0.0 for all graduating components.

## What constitutes a "stable API"

Each component's public API surface should be reviewed before graduation. The v1.0.0 tag is a commitment that:
- CRD schemas (for operators): existing fields will not be removed or have their semantics changed without a major bump
- Go package exports (for libraries): exported types, functions, and interfaces are stable
- CLI flags and env vars: existing flags will not be removed without a major bump
- Rust public API (for limitador crate, wasm-shim): `pub` items are stable

This review does not need to be exhaustive — these APIs are already stable in practice. The review confirms there are no known planned breaking changes in the near term.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why not stay at 0.x

Staying at 0.x is the easiest path but perpetuates the mismatch between version numbers and actual stability. It also means semver-aware tooling (patch release verification, dependency update bots, security scanners) cannot trust version signals.

## Why not graduate incrementally (one component at a time)

Graduating one component at a time creates a transitional period where kuadrant-operator depends on a mix of 0.x and 1.x components. This is technically fine but adds confusion. Graduating all at once in a single kuadrant release is cleaner.

## Why v1.0.0 and not v1.N.0 (matching current minor)

The graduation is about signaling stability, not preserving version history. v1.0.0 is the conventional "this is now stable" marker. Starting at v1.25.0 (for authorino) would be unusual and confusing.

# Prior art
[prior-art]: #prior-art

- **Kubernetes**: Graduated from 0.x to 1.0 in 2015 after being in production at Google for years.
- **Istio**: Graduated to 1.0 in 2018 after extensive production usage.
- **cert-manager**: Graduated to 1.0 in 2020 after being stable for years at 0.x.
- **Envoy Gateway**: Still at 0.x (as of 2025), despite production usage — a cautionary example of inertia.

In all cases, the v1.0 release was primarily a signaling event, not a technical one.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Should all components graduate simultaneously, or is a staggered approach (e.g., operators first, then operands) acceptable?
- Does wasm-shim's API (Proxy-Wasm ABI) warrant the same stability commitment, or is it more of an internal implementation detail?
- Should policy-machinery graduate at the same time, given it's a library with a smaller user base outside kuadrant-operator?

# Future possibilities
[future-possibilities]: #future-possibilities

With all components at v1.x, semver becomes trustworthy across the suite:
- Patch release automation can apply the same rules uniformly — no special cases for 0.x components.
- Dependency update tools (Dependabot, Renovate) can make meaningful decisions about whether a bump is safe.
- The version numbers communicate production-readiness to potential adopters.
