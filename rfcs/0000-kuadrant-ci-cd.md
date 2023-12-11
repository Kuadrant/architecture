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

<details>
<summary>Summary of the current process</summary>

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

### Authorino Operator
The Authorino operator is similar to its operand, it fully follows the [Operator Pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
and to build its manifests relies on [Kustomize](https://kustomize.io/).

#### Artifacts
Its artifacts are the [Authorino Operator image](https://quay.io/repository/kuadrant/authorino-operator),
[Authorino Operator Bundle](https://quay.io/repository/kuadrant/authorino-operato-bundle),
[Authorino Operator catalog](https://quay.io/repository/kuadrant/authorino-operator-catalog) and regarding its [manifests](https://github.com/Kuadrant/authorino-operator/blob/main/config/crd/bases/operator.authorino.kuadrant.io_authorinos.yaml)
the Authorino CRD and role definitions.

#### Build / Release
To release a version “v0.W.Z” of Authorino Operator, these are the steps needed:

1. Pick a stable (released) version “v0.X.Y” of the operand known to be compatible with operator’s image for “v0.W.Z”; if needed, make a release of the operand first.
2. Run the [Github Action Release operator](https://github.com/Kuadrant/authorino-operator/actions/workflows/release.yaml);
make sure to fill all the fields:
  * Branch containing the release workflow file – default: ‘main’
  * Commit SHA or branch name of the operator to release – usually: ‘main’
  * Operator version to release (without prefix) – i.e. ‘0.W.Z’
  * Authorino version the operator enables installations of (without prefix) – i.e. ‘0.X.Y’
  * If the release is a prerelease
  * Bundle and catalog channels (comma-separated) – usually: 'stable'
3. Run the GHA ‘Build and push images’ for the “v0.W.Z” tag, specifying ‘Authorino version’ equals to “0.X.Y” (without the leading “v”). This will cause the new images (bundle and catalog included) to be built and pushed to the corresponding repos in quay.io/kuadrant.

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

### Limitador Operator
This operator is also implemented using the [Operator SDK](https://sdk.operatorframework.io/) and
[Kustomize](https://kustomize.io/).

#### Artifacts
The deliverable artifacts are the [Limitador Operator image](https://quay.io/repository/kuadrant/limitador-operator),
[Limitador Operator Bundle](https://quay.io/repository/kuadrant/limitador-operato-bundle),
[Limitador Operator image](https://quay.io/repository/kuadrant/limitador-operator) and regarding its
[manifests](https://github.com/Kuadrant/limitador-operator/tree/main/bundle/manifests) the Limitador CRD and role
definitions.

#### Build / Release
This build process is similar to the Authorino Operator one, but it's triggered by a different GitHub Action
(Build Images) which is a `workflow_dispatch` one, and it's triggered manually. The steps are as follows:

1. Pick a stable (released) version “v0.X.Y” of the operand (Limitador) known to be compatible with operator’s image for
“v0.W.Z”; if needed, make a release of the operand first.
2. Create a branch named after the version, e.g. `release-0.7.0`
```sh
git checkout -b release-0.7.0
```
3. Tag the branch with the `v` prefix, e.g. `v0.7.0`, push the tag to remote
```sh
git tag -a v0.7.0 -m "[tag] Limitador Operator v0.7.0"
git push origin v0.7.0
```
4. Run the GHA 'Build Images' with the following parameters:
  * Branch containing the release workflow file – example: `release-0.7.0`, default: `main`
  * Operator bundle version (without prefix) - example: `0.7.0`, default: `0.0.0`
  * Operator tag (with prefix) - example: `v0.7.0`, default: `latest`
  * Limitador version (without prefix) – example: ‘1.3.0’, default: `0.0.0`
  * Limitador operator replaced version (without prefix) - example: `0.6.0`, default: `0.0.0-alpha`
  * Bundle and catalog channels (comma-separated) – example: `stable`, default: `preview`

### WasmShim
A Proxy-Wasm module written in Rust, acting as a shim between Envoy and Limitador. It's built using an automated
GitHub Action and published to quay.io/kuadrant/wasm-shim.

#### Artifacts
The deliverable artifact is the [WasmShim image](https://quay.io/repository/kuadrant/wasm-shim).

#### Build / Release
The before mentioned GitHub Action is triggered by a push to any branch of the repository. It builds the image and
pushes it to quay.io/kuadrant/wasm-shim:<git-ref> and quay.io/kuadrant/wasm-shim:latest.

In order to obtain a new release of the WasmShim, the following steps need to be followed:

1. A branch needs to be created, e.g. `release-0.3.0`

```sh
git checkout -b release-0.3.0
```

2. A new tag needs to be created, e.g. `v0.3.0` and pushed to remote

```sh
git tag -a v0.3.0 -m "[tag] WasmShim v0.3.0"
git push origin v0.3.0
```

3. The GitHub Action will be triggered and will build and push the image to quay.io/kuadrant/wasm-shim:v0.3.0

### Kuadrant Operator
The final step of the process is to build and release the Kuadrant Operator. This is a Go project that follows the
[Operator Pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) following the [Operator SDK](https://sdk.operatorframework.io/)
and to build its manifests relies on [Kustomize](https://kustomize.io/) as the previous operators mentioned.

#### Artifacts
The deliverable artifacts are the [Kuadrant Operator image](https://quay.io/repository/kuadrant/kuadrant-operator),
[Kuadrant Operator Bundle](https://quay.io/repository/kuadrant/kuadrant-operato-bundle),
[Kuadrant Operator image](https://quay.io/repository/kuadrant/kuadrant-operator) and regarding its
[manifests](https://github.com/Kuadrant/kuadrant-operator/tree/main/bundle) we can find the Kuadrant CRD, RateLimitPolicy
CRD, AuthPolicy CRD and role definitions.

#### Build / Release
The build and release process of this operator is similar to the Limitador Operator one, but its dependencies are more,
including the WasmShim, the Authorino Operator and the Limitador Operator. The steps are as follows:

1. A stable released version of Limitador Operator, Authorino Operator and WasmShim images are needed.

2. Create a branch named after the version, e.g. `release-0.5.0`
```sh
git checkout -b release-0.5.0
```

3. Tag the branch with the `v` prefix, e.g. `v0.5.0`, push the tag to remote
```sh
git tag -a v0.5.0 -m "[tag] Kuadrant Operator v0.5.0"
git push origin v0.5.0
```

4. Run the GHA 'Build Images' with the following parameters:
* Branch containing the release workflow file – example: `release-0.5.0`, default: `main`
* Operator bundle version (without prefix) - example: `0.5.0`, default: `0.0.0`
* Operator tag (with prefix) - example: `v0.5.0`, default: `latest`
* Authorino Operator bundle version (without prefix) - example: `0.10.0`, default: `latest`
* Limitador Operator bundle version (without prefix) - example: `0.7.0`, default: `latest`
* WASM Shim version (without prefix) – example: `0.3.0`, default: `latest`
* Kuadrant operator replaced version (without prefix) - example: `0.4.1`, default: `0.0.0-alpha`
* Bundle and catalog channels (comma-separated) – example: `stable`, default: `preview`


### OperatorHub
The current 3 operators (Authorino, Limitador and Kuadrant) are published to [OperatorHub](https://operatorhub.io/).
This is done manually, copying the released bundle from each operator and opening a PR to the [community-operators](https://github.com/k8s-operatorhub/community-operators/)
and [community-operators-prod](https://github.com/redhat-openshift-ecosystem/community-operators-prod/) repositories.
The documentation for this process can be found in the [contribution section of OperatorHub](https://operatorhub.io/contribute).

A brief summary of the process is as follows:

1. In each community-operators repository, create a branch named after the name of the operator and its version, following
the template `[operator-name]-[semantic-version]`, e.g.`kuadrant-operator-v0.5.0` and create a directory within the
`operators/[operator-name]/[semantic-version]` path.
```sh
git checkout -b kuadrant-operator-v0.5.0
mkdir -p operators/kuadrant-operator/0.5.0
```

2. Copy the bundle and bundle.Dockerfile from the released operator branch to the `community-operators` repository directory just created.
```sh
cp -r [path-to]/kuadrant-operator/bundle/* [path-to]/community-operators/operators/kuadrant-operator/0.5.0/
cp [path-to]/kuadrant-operator/bundle.Dockerfile [path-to]/community-operators/operators/kuadrant-operator/0.5.0/
```

3. Create a PR to the `community-operators` (same for `community-operators-prod`) repository and follow the guidelines
exposed in the auto generated description of the PR.

</details>

## New process
The new process ideally should be as simple as possible, and should be able to be triggered by any person, process and
set of events (particular commits, tags, merge, etc) that needs the artifacts. It also should be able to be repeated
in a consistent way and without the need of any human setting variables or parameters at the time of triggering it
(at least for the most common cases).

Given our current process, we can identify some similarities and particularities between the different components:

* All of them are built using a CI/CD tool (GitHub Actions) and published to a container registry (quay.io).
* Not all of them are built using the same build system (Cargo, Go, Operator SDK, etc.).
* 3 of them follow the Operator Pattern and are built using the Operator SDK (Authorino, Limitador and Kuadrant operators).
* 2 of them are Rust projects (Limitador and WASMShim).
* There's a dependency between the different components, Kuadrant Operator depends on Authorino and Limitador Operators,
and the WasmShim; while the before mentioned operators depend on their operands (Authorino and Limitador).
* There's no clear steps defined regarding the approval of the deliverables (images, manifests, etc.) and their
publication to our website, Github releases, OperatorHub, etc.

Given the before mentioned points, we can identify some common steps that can be extracted and reused for all the
components, and some particularities that need to be addressed in a different way, but the overall
process should be the same for all of them in terms of triggering it and the steps that need to be followed.

### Option 1: Release File
The first option is to create a file in the root of each repository that contains the information needed to build and
release the component. This file could be a YAML, TOML or simply a key-value file. The file should contain the following
information:

* Component/s name/s and version/s (In the case of Limitador, it should contain the name and version of the crate and the name of the server).
* Dependencies name/s and version/s.
* Replace version (in the case of the Operators, this version is needed for the OLM upgrade process).

This file should be updated every time a new release is made, and it should be committed to the repository to the release
branch. This file will be used by the CI/CD tool to build and release the component. In the case of the main branch, the
file will include the next version to be released, following the floating suffix `-dev` for its version and its dependencies.

The CI/CD workflow will be triggered by a specific event (tag, commit, etc.) and will read the file and use the
information to build and release the component. The workflow will be the same for all the components, and it will
contain the following steps:

Manual steps:
1. Create a new branch from the main branch or cherry-picking the commits that need to be released.
2. Update the release file with the new version/s and dependencies.
3. Commit the changes to the release branch.
4. Create a new tag with the new version following the semantic versioning vX.Y.Z.
5. Push the tag to the repository.
6. After the release workflow is done:
   * approve the Github draft release and publish it.
   * change the draft PR on OperatorHub to ready to review.

Automatic steps on the CI/CD workflow:
1. Checkout the repository.
2. Read the file and extract the information.
3. Builds the manifests and images.
4. Create a draft release on Github with the information extracted from the file.
5. Build the component after running verification and tests.
6. Publish the component to the container registry.


### Option 2: Release Script
The second option involves creating a script that will be used with the same purpose of releasing the component's deliverables.
The script will be a bash script that will prompt the user for the specific information needed, such as the component's
name and version, its dependencies, etc. It will also fail if there's missing information or if the information provided
is not valid. The script will be the same for all the components, and it will contain the following steps:

Manual steps:
1. Create a new branch from the main branch or cherry-picking the commits that need to be released.
2. Run the script and provide the information needed.
3. Commit the changes to the release branch.
4. Create a new tag with the new version following the semantic versioning vX.Y.Z.
5. Push the tag to the repository.
6. After the release workflow is done:
   * approve the Github draft release and publish it.
   * change the draft PR on OperatorHub to ready to review.

Automatic steps on the CI/CD workflow:
1. Checkout the repository.
2. Run tests and verify manifests, bundle, etc.
3. Build the component/s.
4. Publish the component/s to the container registry.
5. Create a draft release on Github with the information provided by the user.

### Option 3: Release Workflow
The third option is to create a Github Action that will be manually triggered by the user. The action will be the same
for all the components, but will prompt the user for the specific information needed, such as the component's name and
version, its dependencies, etc. It will also fail if there's missing information or if the information provided is not
valid. The action will contain the following steps:

Manual steps:
1. From the Actions tab, select the Release action.
2. Introduce the information needed. I.e.:
   * Component/s name/s and version/s.
   * Dependencies name/s and version/s.
   * Replace version (in the case of the Operators, this version is needed for the OLM upgrade process).
3. Run the action.
4. After the release workflow is done:
   * approve the Github draft release and publish it.
   * change the draft PR on OperatorHub to ready to review.

Automatic steps on the CI/CD workflow:
1. Checkout the repository.
2. Build the component manifests and bundle.
3. Run tests and verify manifests, bundle, etc.
4. Build the component/s image/s.
5. Publish the component/s to the container registry.
6. Create a draft release on Github with the information provided by the user.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

For the different options proposed, we'll be using [Limitador](https://github.com/Kuadrant/limitador) and the
[Limitador Operator](https://github/com/Kuadrant/limitador-operator) as examples since the former is the most different
from the rest of the components, and the latter is the most common one.

## Release File
For this option, will be using a YAML file named `release.yaml` that will be placed in the root of the repository.

### Limitador
The file will contain the following information for Limitador component in the main branch pointing to the
`[next-version]-dev`:

```yaml
server_version: 1.4.0-dev
crate_version: 0.6.0-dev
```

In order to prepare the release, the following steps need to be followed:
1. Create a new branch with the selection of commits that need to be released.
```sh
git checkout -b release-1.4.0
```

2. Update the release file with the new version/s.
```yaml
server_version: 1.4.0
crate_version: 0.6.0
```

3. Execute the release script/makefile target.
    TBD. This will use the information from the release file to update the Cargo.toml files and commit the changes to the
    release branch with message "[release] Releasing Limitador crate 0.6.0 and Server 1.4.0".

You will see the following changes:

 ```diff
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
 ```diff
diff --git a/limitador-server/Cargo.toml b/limitador-server/Cargo.toml
index dd2f311..011a2cd 100644
--- a/limitador-server/Cargo.toml
+++ b/limitador-server/Cargo.toml
@@ -1,6 +1,6 @@
 [package]
 name = "limitador-server"
-version = "1.4.0-dev"
+version = "1.4.0"
 ```

4. Create a new tag with the new version following the semantic versioning vX.Y.Z.
```sh
git tag -a v1.4.0 -m "[tag] Limitador v1.4.0"
git push origin v1.4.0
```

At this stage, the release workflow will be triggered and will do the before mentioned steps in the [Guide-level explanation](#guide-level-explanation).

5. After the release workflow is done:
   * approve the Github draft release and publish it.
   * change the draft PR on OperatorHub to ready to review.

### Limitador Operator
For the Limitador Operator, the release file will contain the following information in the main branch pointing to the
`[next-version]-dev`:

```yaml
operator_version: 0.8.0-dev
limitador_version: 1.4.0-dev
replaces_version: 0.7.0
```

In order to prepare the release, the following steps need to be followed:
1. Create a new branch with the selection of commits that need to be released.
```sh
git checkout -b release-0.8.0
```

2. Update the release file with the new version/s.
```yaml
operator_version: 0.8.0
limitador_version: 1.4.0
replaces_version: 0.7.0
```

3. Execute the release script/makefile target.
   TBD. This will use the information from the release file to update the manifests and bundle files and commit the changes to the
   release branch with message "[release] Releasing Limitador Operator 0.8.0".

Between others, you will see the following changes:
```diff
diff --git a/bundle.Dockerfile b/bundle.Dockerfile
index 3aebf9d..d17b92b 100644
--- a/bundle.Dockerfile
+++ b/bundle.Dockerfile
@@ -1,6 +1,6 @@
 LABEL operators.operatorframework.io.bundle.channels.v1=alpha
 LABEL operators.operatorframework.io.bundle.channels.v1=stable
```
```diff
diff --git a/bundle/manifests/limitador-operator.clusterserviceversion.yaml b/bundle/manifests/limitador-operator.clusterserviceversion.yaml
index 3aebf9d..d17b92b 100644
--- a/bundle/manifests/limitador-operator.clusterserviceversion.yaml
+++ b/bundle/manifests/limitador-operator.clusterserviceversion.yaml
@@ -1,6 +1,6 @@
 apiVersion: operators.coreos.com/v1alpha1
 kind: ClusterServiceVersion
-metadata:
-  name: limitador-operator.v0.8.0-dev
+metadata:
+  name: limitador-operator.v0.8.0
 spec:
   apiservicedefinitions:
     owned:
@@ -9,7 +9,7 @@ spec:
    customresourcedefinitions:
      owned:
      - description: Limitador is the main resource that describes a rate limiting policy.
-       displayName: Limitador v0.8.0-dev
+       displayName: Limitador v0.8.0
        kind: Limitador
        name: limitadors.operator.kuadrant.io
        version: v1alpha1
@@ -17,7 +17,7 @@ spec:
        - description: RateLimitPolicy is the resource that describes a rate limiting policy.
            displayName: RateLimitPolicy
            kind: RateLimitPolicy
-           name: ratelimitpolicies.operator.kuadrant.io.v0.8.0-dev
+           name: ratelimitpolicies.operator.kuadrant.io.v0.8.0
            version: v1alpha1
          - description: AuthPolicy is the resource that describes an authentication policy.
            displayName: AuthPolicy
@@ -26,7 +26,7 @@ spec:
replaces: limitador-operator.v0.7.0
-  version: 0.8.0-dev
+  version: 0.8.0
```

4. Create a new tag with the new version following the semantic versioning vX.Y.Z.
```sh
git tag -a v0.8.0 -m "[tag] Limitador Operator v0.8.0"
git push origin v0.8.0
```

At this stage, the release workflow will be triggered and will do the before mentioned steps in the [Guide-level explanation](#guide-level-explanation).

5. After the release workflow is done:
   * approve the Github draft release and publish it.
   * change the draft PR on OperatorHub to ready to review.

6. Create a `next` branch off `main`
7. Update the _both_ release files to point to the next `-dev` release
8. Create PR
9. Merge to `main`

### Notes
* For any other specific version that is not meant for the actual release, the release file could contain the following information:
```yaml
server_version: 1.4.0-rc1
crate_version: 0.6.0-rc1
```
or

```yaml
operator_version: 0.8.0-qe-test
limitador_version: 1.4.0-qe-test
replaces_version: 0.7.0
```

## Release Script
The release script process will be similar to the release file one, but instead of updating a file that will be used
as input for the needed script or workflow, the user will be prompted for the information needed to build and release
the component.

### Limitador
In order to prepare the release, the following steps need to be followed:

1. Create a new branch with the selection of commits that need to be released.
```sh
git checkout -b release-1.4.0
```

2. Run the release script/makefile target.
   This will communicate the user the parameters needed when ran without any, or it could prompt the user for them.
   When successfully ends applying the changes, it will commit them to the release branch with message
   "[release] Releasing Limitador crate 0.6.0 and Server 1.4.0".

```sh
./release.sh limitador-server=1.4.0 limitador-crate=0.6.0
```
You will see the following changes:

 ```diff
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
 ```diff
diff --git a/limitador-server/Cargo.toml b/limitador-server/Cargo.toml
index dd2f311..011a2cd 100644
--- a/limitador-server/Cargo.toml
+++ b/limitador-server/Cargo.toml
@@ -1,6 +1,6 @@
 [package]
 name = "limitador-server"
-version = "1.4.0-dev"
+version = "1.4.0"
 ```

3. Create a new tag with the new version following the semantic versioning vX.Y.Z.
```sh
git tag -a v1.4.0 -m "[tag] Limitador v1.4.0"
git push origin v1.4.0
```
At this stage, the release workflow will be triggered and will do the before mentioned steps in the [Guide-level explanation](#guide-level-explanation).

4. After the release workflow is done:
   * approve the Github draft release and publish it.
   * change the draft PR on OperatorHub to ready to review.

5. Create a `next` branch off `main`
6. Update the _both_ release files to point to the next `-dev` release
7. Create PR
8. Merge to `main`

### Limitador Operator
In order to prepare the release, the following steps need to be followed:

1. Create a new branch with the selection of commits that need to be released.
```sh
git checkout -b release-0.8.0
```

2. Run the release script/makefile target.
   This will communicate the user the parameters needed when ran without any, or it could prompt the user for them.
   When successfully ends applying the changes, it will commit them to the release branch with message
   "[release] Releasing Limitador Operator 0.8.0".

```sh
./release.sh limitador-operator=0.8.0 limitador=1.4.0 replaces=0.7.0
```

Between others, you will see the following changes:
```diff
diff --git a/bundle/manifests/limitador-operator.clusterserviceversion.yaml b/bundle/manifests/limitador-operator.clusterserviceversion.yaml
index 3aebf9d..d17b92b 100644
--- a/bundle/manifests/limitador-operator.clusterserviceversion.yaml
+++ b/bundle/manifests/limitador-operator.clusterserviceversion.yaml
@@ -1,6 +1,6 @@
 apiVersion: operators.coreos.com/v1alpha1
 kind: ClusterServiceVersion
-metadata:
-  name: limitador-operator.v0.8.0-dev
+metadata:
+  name: limitador-operator.v0.8.0
 spec:
   apiservicedefinitions:
     owned:
@@ -9,7 +9,7 @@ spec:
    customresourcedefinitions:
      owned:
      - description: Limitador is the main resource that describes a rate limiting policy.
-       displayName: Limitador v0.8.0-dev
+       displayName: Limitador v0.8.0
        kind: Limitador
        name: limitadors.operator.kuadrant.io
        version: v1alpha1
@@ -17,7 +17,7 @@ spec:
        - description: RateLimitPolicy is the resource that describes a rate limiting policy.
            displayName: RateLimitPolicy
            kind: RateLimitPolicy
-           name: ratelimitpolicies.operator.kuadrant.io.v0.8.0-dev
+           name: ratelimitpolicies.operator.kuadrant.io.v0.8.0
            version: v1alpha1
          - description: AuthPolicy is the resource that describes an authentication policy.
            displayName: AuthPolicy
@@ -26,7 +26,7 @@ spec:
replaces: limitador-operator.v0.7.0
-  version: 0.8.0-dev
+  version: 0.8.0
```

3. Create a new tag with the new version following the semantic versioning vX.Y.Z.
```sh
git tag -a v0.8.0 -m "[tag] Limitador Operator v0.8.0"
git push origin v0.8.0
```

At this stage, the release workflow will be triggered and will do the before mentioned steps in the [Guide-level explanation](#guide-level-explanation).

4. After the release workflow is done:
   * approve the Github draft release and publish it.
   * change the draft PR on OperatorHub to ready to review.

5. Create a `next` branch off `main`
6. Update the _both_ release files to point to the next `-dev` release
7. Create PR
8. Merge to `main`

### Notes
* For any other specific version that is not meant for the actual release, the release script could be used with the following parameters:
```sh
./release.sh limitador-server=1.4.0-rc1 limitador-crate=0.6.0-rc1
```
or

```sh
./release.sh limitador-operator=0.8.0-qe-test limitador=1.4.0-qe-test replaces=0.7.0
```

* This option could be combined with the release file one, so the script could read the information from the file or prompt the user for it.

## Release Workflow
The release workflow process, besides bringing the same benefits as the other ones, will also allow to trigger the
entire process from a Github Action on demand, which it also could be seen as a complement to the other options.

### Limitador
In order to prepare the release, the following steps need to be followed:

1. From the Actions tab, select the Release action. It will prompt the user the following parameters:
   * Server version. I.e.: `1.4.0`
   * Crate version. I.e.: `0.6.0`
   * Release branch/commit/tag. I.e.: `release-1.4.0`
   * Release tag. If not provided, it will be generated from the version. I.e.: `v1.4.0`

2. Run the action.
This will commit the changes to the release branch with message "[release] Releasing Limitador crate 0.6.0 and Server 1.4.0".
It will verify the manifests, build the images and publish them to the container registry. It also will draft a release
on Github with the information provided by the user.

3. After the release workflow is done:
   * approve the Github draft release and publish it.
   * change the draft PR on OperatorHub to ready to review.

The following steps can be automated by the release workflow, but they could be done manually as well:
4. Create a `next` branch off `main`
5. Update the _both_ release files to point to the next `-dev` release
6. Create PR
7. Merge to `main`

### Limitador Operator
In order to prepare the release, the following steps need to be followed:

1. From the Actions tab, select the Release action. It will prompt the user the following parameters:
   * Operator version. I.e.: `0.8.0`
   * Limitador version. I.e.: `1.4.0`
   * Replaces version. I.e.: `0.7.0`
   * Release branch/commit/tag. I.e.: `release-0.8.0`
   * Release tag. If not provided, it will be generated from the version. I.e.: `v0.8.0`

2. Run the action.
This will commit the changes to the release branch with message "[release] Releasing Limitador Operator 0.8.0".
It will verify the manifests, build the images and publish them to the container registry. It also will draft a release
on Github with the information provided by the user.

3. After the release workflow is done:
   * approve the Github draft release and publish it.
   * change the draft PR on OperatorHub to ready to review.

The following steps can be automated by the release workflow, but they could be done manually as well:
4. Create a `next` branch off `main`
5. Update the _both_ release files to point to the next `-dev` release
6. Create PR
7. Merge to `main`

### Notes
* For any other specific version that is not meant for the actual release, the release workflow could be executed with
custom paramaters.
* This option could be an enhancement to the previous ones, and work as a complement to them.

# Drawbacks
[drawbacks]: #drawbacks

* It will require some effort to implement the new process.
* There's still some other-than-code steps that need to be done manually, such as the approval of the Github draft release,
the approval of the OperatorHub PR, communication (blogpost, slack, etc).
* It will need a thorough verification and testing to ensure it works as expected.

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