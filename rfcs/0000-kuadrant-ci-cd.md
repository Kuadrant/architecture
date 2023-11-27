# Kuadrant CI/CD release process

- Feature Name: `kuadrant-ci-cd`
- Start Date: 2023-11-21
- RFC PR: [Kuadrant/architecture#0038](https://github.com/Kuadrant/architecture/pull/38)
- Issue tracking: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/issues/0000)

# Summary
[summary]: #summary

This RFC not only proposes a new release process for Kuadrant components, but also a new and more agile way of getting
the desired "artifacts" (images, manifests, etc.) into the hands of the users, devs, QE team and any other process that
needs them.

# Motivation
[motivation]: #motivation

The current process is not only slow, but also involves manual steps that are prone to human error. Its implementation
is also not very flexible, and it's not easy to add new steps or change the existing ones without replicating the same
code in different repositories. It also involves a convolution of different tools and services that are not easy to
maintain and that are not very well integrated with each other.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Current process

Before we dive into the new process, let's take a look at the current one. The current process is based on the
independent build and release of each service (Authorino and Limitador), their respective Operators and the WasmShim
for then building and releasing the Kuadrant Operator. It's important that we follow this order because the
Kuadrant Operator depends on the WasmShim, and the Authorino and Limitador Operators. The process is as follows:

### Authorino
A particularity of Authorino is that it follows the [Controller Pattern](https://kubernetes.io/docs/concepts/architecture/controller/#controller-pattern),
it's implemented as a Kubernetes controller thanks to the [Operator SDK](https://sdk.operatorframework.io/) and
[Kustomize](https://kustomize.io/).

#### Artifacts
The deliverable artifacts are the [Authorino image](https://quay.io/repository/kuadrant/authorino)
and its [manifests](https://github.com/Kuadrant/authorino/blob/main/install/kustomization.yaml), the AuthConfig CRD and
role definitions.

#### Build / Release
The build process is triggered by a particular GitHub Action that runs on demand (workflow_dispatch) named "Build and push image",
and to build an image version “v0.X.Y” of Authorino, the following steps are needed:

1. Pick a <git-ref> (SHA-1) as source.
Create a new tag and named release “v0.X.Y”. Make sure to write change notes highlighting all the new features, bug fixes,
enhancements, etc ([example](https://github.com/Kuadrant/authorino/releases/tag/v0.9.0)).

2. Run the GHA ‘Build and push images’ for the “v0.X.Y” tag. This will cause a new image to be built and pushed to quay.io/kuadrant/authorino.

3. Create the release and release notes on [Github](https://github.com/Kuadrant/authorino/releases/new) using the tag from above, named: `Authorino vM.m.d`

Notes:
PRs merged to the main branch of Authorino cause a new image to be built (GH Action) and pushed automatically to
quay.io/kuadrant/authorino:<git-ref> – the quay.io/kuadrant/authorino:latest tag is also moved to match the latest <git-ref>.

### Limitador
In the case of Limitador, it's a completely different story. It's a Rust project, that it's split into a library (crate)
and a server. The server is a binary that uses the library, and it's built using the [Cargo](https://doc.rust-lang.org/cargo/)
build system. The library is published to [crates.io](https://crates.io/crates/limitador) and the server is published to
quay.io/kuadrant/limitador.

#### Artifacts
The deliverable artifacts are the [Limitador service image](https://quay.io/repository/kuadrant/limitador) and the Rust
crate published to [crates.io](https://crates.io/crates/limitador).

#### Build / Release

##### `limitador` crate to crates.io

1. A branch needs to be created, e.g. `release-0.5.0`
```sh
git checkout -b release-0.5.0
```

2. Remove the `-dev` suffix from both `Cargo.toml`
 ```diff
diff --git a/limitador-server/Cargo.toml b/limitador-server/Cargo.toml
index dd2f311..b555df8 100644
--- a/limitador-server/Cargo.toml
+++ b/limitador-server/Cargo.toml
@@ -1,6 +1,6 @@
 [package]
 name = "limitador-server"
-version = "1.3.0-dev"
+version = "1.3.0"
diff --git a/limitador/Cargo.toml b/limitador/Cargo.toml
index 3aebf9d..d17b92b 100644
--- a/limitador/Cargo.toml
+++ b/limitador/Cargo.toml
@@ -1,6 +1,6 @@
 [package]
 name = "limitador"
-version = "0.5.0-dev"
+version = "0.5.0"
 ```

3. Commit the changes
```sh
git commit -am "[release] Releasing Crate 0.5.0 and Server 1.3.0"
```

4. Create a tag named after the version, with the `crate-v` prefix and push it to remote
```sh
git tag -a crate-v0.5.0 -m "[tag] Limitador crate v0.5.0"
git push origin crate-v0.5.0
```

5. Manually run the `Release crate` workflow action on [Github](https://github.com/Kuadrant/limitador/actions/workflows/release.yaml) providing the version to release in the input box,
e.g. `0.5.0`, if all is correct, this should push the release to [crates.io](https://crates.io/crates/limitador/versions)

6. Create the release and release notes on [Github](https://github.com/Kuadrant/limitador/releases/new) using the tag from above, named: `Limitador crate vM.m.d`


##### `limitador-server` container image to quay.io

1. Create a branch for your version with the `v` prefix, e.g. `v1.3.0`
2. Make sure your `Cargo.toml` is reflecting the proper version, see above
3. Push the branch to remote, which should create a matching release to quay.io with the tag name based of your branch, i.e. in this case `v1.3.0`
4. Create a tag with the `server-v` prefix, e.g. `server-v1.3.0`
5. Push the tag to Github
6. Create the release and release notes on [Github](https://github.com/Kuadrant/limitador/releases/new) using the tag from above, named: `vM.m.d`
7. Delete the branch, only keep the tag used for the release

##### After the release

1. Create a `next` branch off `main`
2. Update the _both_ Cargo.toml to point to the next `-dev` release
3. Create PR
4. Merge to `main`

 ```diff
diff --git a/limitador-server/Cargo.toml b/limitador-server/Cargo.toml
index dd2f311..011a2cd 100644
--- a/limitador-server/Cargo.toml
+++ b/limitador-server/Cargo.toml
@@ -1,6 +1,6 @@
 [package]
 name = "limitador-server"
-version = "1.3.0-dev"
+version = "1.4.0-dev"
 ```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- How error would be reported to the users.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- Does another project have a similar feature?
- What can be learned from it? What's good? What's less optimal?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other tentatives - successful or not, provide readers of your RFC with a fuller picture.

Note that while precedent set by other projects is some motivation, it does not on its own motivate an RFC.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would be and how it would affect the platform and project as a whole. Try to use this section as a tool to further consider all possible interactions with the project and its components in your proposal. Also consider how this all fits into the roadmap for the project and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but otherwise related.

Note that having something written down in the future-possibilities section is not a reason to accept the current or a future RFC; such notes should be in the section on motivation or rationale in this or subsequent RFCs. The section merely provides additional information.