# RFC Multiple gateway providers in the same cluster

- Feature Name: `gateway_providers`
- Start Date: 2022-12-13
- RFC PR: [Kuadrant/architecture#7](https://github.com/Kuadrant/architecture/pull/7)
- Issue tracking: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/issues/0000)

# Summary
[summary]: #summary

Option to reference 1+ specific control plane/gateway providers when requesting a Kuadrant instance.

# Motivation
[motivation]: #motivation

Because [Red Hat® OpenShift® Service Mesh (OSSM)](https://docs.openshift.com/container-platform/4.11/service_mesh/v2x/ossm-about.html) supports multiple service mesh control planes running in the same cluster, as well as [Istio](https://istio.io/) can be installed in a same cluster where OSSM is running<sup>*</sup>, Kuadrant users must have a way to select, when requesting a Kuadrant instance, the specific control plane provider that implements the gateway functionality (_"gateway provider"_) that is targeted by the Kuadrant policies managed by that instance (_"Kuadrant instance"_).

Without being able to indicate which provided control plane instance to target, among multiple possibly existing in the cluster, the Kuadrant control plane (Kuadrant Operator) needs to select one based upon convention or heuristics, which may result in the wrong gateway provider selected, as well as ocasional runtime errors when configuring the functional components.

Examples of selection of gateway provider based on convention or heuristic include the current approach by which the Kuadrant Operator tries to default to Istio as the primary provider by checking for the presence of the `IstioOperator.install.istio.io` CRD in the cluster, and only fall back to `ServiceMeshControlPlane.maistra.io` (OSSM) if Istio is not installed (see [kuadrant-operator#111](https://github.com/kuadrant/kuadrant-operator/pull/111)).

One way this approach can lead to a runtime error is when the Istio CRD is installed but OSSM is the actual control plane providing the gateway functionality and the one intended by the user when requesting the Kuadrant instance.

Moreover, with the current implementation, the name of the control plane resource that the Kuadrant Operator will try to patch to inject the required configs for the functional components to work – Authorino in particular – is read from the values of the `ISTIOOPERATOR_NAMESPACE` and `ISTIOOPERATOR_NAME` environment variables or, in the absence of such, their default values – i.e. `"istio-system"` and `"istiocontrolplane"`, respectivelly. When more than one gateway provider is available in the cluster, the Kuadrant control plane can only be aware of one of them, making effectively impossible to support multiple instances of Kuadrant working each with a different provider in the same cluster.

Having one instance of Kuadrant working with multiple gateway providers at the same time is also not an option today.

<br/>

<sub><sup>*</sup>TBC</sub>

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When requesting an instance of Kuadrant to the Kuadrant Operator, users can specify in which control plane/gateway providers that one instance of Kuadrant must configure the functional components (Authorino and Limitador) by listing the resource references in the `Kuadrant` CR. E.g.:

```yaml
apiVersion: kuadrant.io/v1beta1
kind: Kuadrant
metadata:
  name: kuadrant-sample
  namespace: kuadrant-system
spec:
  gatewayProviders: # Alternatively: `controlPlaneProviders`, `controlPlaneRefs`, `gatewayProviderRefs`, `providers`
  - group: maistra.io/v2
    kind: ServiceMeshControlPlane
    name: basic
    namespace: istio-system
  - group: maistra.io/v2
    kind: ServiceMeshControlPlane
    name: enterprise
    namespace: istio-system
```

In the example above, the functional components (Authorino and Limitador) deployed as part of the `kuadrant-sample` Kuadrant instance will be configured in all gateways managed by the `basic` or `enterprise` OSSM control planes, thus causing `AuthPolicy` and `RateLimitPolicy` custom resources that target network objects within the hierarchy of those gateways to be handled by those Authorino and Limitador instances deployed with `kuadrant-sample`.

**Constraints**

At least one provider must be supplied in the `gatewayProviders` field.

The following kinds of resources are initially supported:
- `IstioOperator` (`install.istio.io/v1alpha1`)
- `ServiceMeshControlPlane` (`maistra.io/v2`)

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

When reconciling a `Kuadrant` custom resource, for every Istio-based gateway provider listed in `spec.gatewayProviders`, the Kuadrant Operator will:
1. patch the control plane configuration resource ("mesh config") for the provider, injecting the proper Istio [extension provider](https://pkg.go.dev/istio.io/api/mesh/v1alpha1?utm_source=gopls#MeshConfig_ExtensionProvider) setting, pointing it to the Authorino external authorization service;
2. (only when OSSM) add the namespace of the `Kuadrant` CR to the mesh – see [Adding or removing projects from the service mesh](https://docs.openshift.com/container-platform/4.11/service_mesh/v2x/ossm-create-mesh.html#ossm-member-roll-modify_ossm-create-mesh) for details. This step is needed, otherwise the Authorino service will not be reachable to the gateways that are members of the mesh.

For non Istio-based gateway providers (none currently supported), the Kuadrant Operator may need to register the external authorization service – and possibly the rate limit service as well – as instructions and requirements from each provider.

<br/>

**Example 1.** Kuadrant instance for gateways provided by 2 OSSM service meshes.

In the example provided in the previous section with 2 OSSM service meshes listed in the `gatewayProviders` field of the `Kuadrant` CR, the Operator would look up for 2 `ServiceMeshControlPlane` resources, named `basic` and `enterprise`, defined in the `istio-system`. In each of these resources, the Operator would edit the `spec` field so the following entry is present:

```yaml
apiGroup: maistra.io/v2
kind: ServiceMeshControlPlane
metadata:
  name: <basic|enterprise>
  namespace: istio-system
spec:
  techPreview:
    meshConfig:
      extensionProviders:
      - name: <unique-name-of-the-kuadrant-authorization-service-for-this-kuadrant-instance>
        provider:
          envoyExtAuthzGrpc:
            service: <qualified-name-of-the-authorino-service-for-this-kuadrant-instance>
            port: <port-number-of-the-grpc-authorino-service-for-this-kuadrant-instance>
  ...
```

Moreover, the Operator would as well make sure a [`ServiceMeshMemberRoll`](https://docs.openshift.com/container-platform/4.11/service_mesh/v2x/ossm-create-mesh.html#ossm-member-roll-modify_ossm-create-mesh) resource named `default` exists and includes the namespace of the `Kuadrant` CR in its `spec.members` field. I.e.

```yaml
apiGroup: maistra.io/v1
kind: ServiceMeshControlPlane
metadata:
  name: default
  namespace: istio-system
spec:
  members:
  - kuadrant-system
  - ...
```

<br/>

**Example 2.** Kuadrant instance for gateways provided by 1 OSSM service mesh and 1 Istio service mesh.

```yaml
apiVersion: kuadrant.io/v1beta1
kind: Kuadrant
metadata:
  name: kuadrant-sample
  namespace: kuadrant-system
spec:
  gatewayProviders:
  - group: maistra.io/v2
    kind: ServiceMeshControlPlane
    name: basic
    namespace: istio-system
  - group: install.istio.io/v1alpha1
    kind: IstioOperator
    name: istiocontrolplane
    namespace: istio-system
```

The Kuadrant Operator will configure the external authorization extension provider for the `basic` OSSM `ServiceMeshControlPlane` resource as described in the previous example. Additionally, the Operator will configure the extension provider in the `IstioOperator` resource named `istiocontrolplane`:

```yaml
apiGroup: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istiocontrolplane
  namespace: istio-system
spec:
  meshConfig:
    extensionProviders:
    - name: <unique-name-of-the-kuadrant-authorization-service-for-this-kuadrant-instance>
      provider:
        envoyExtAuthzGrpc:
          service: <qualified-name-of-the-authorino-service-for-this-kuadrant-instance>
          port: <port-number-of-the-grpc-authorino-service-for-this-kuadrant-instance>
  ...
```

# Drawbacks
[drawbacks]: #drawbacks

Reconciliation becomes complex with multiple control plane providers per Kuadrant instance.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

A less costly approach could consist upon allowing a Kuadrant instance to explicitly point to one and only one gateway provider. Multiple could still be present in the cluster, however, for each Kuadrant instance, configuration would be injected into only one of the available providers. To have Kuadrant injecting configuration in more than one control plane, multiple Kuadrant instances would have to be deployed.

This alternative would simplify reconciliation and the API. The latter could look like the following, based on a example with 2 providers and 2 Kuadrant instances:

```yaml
apiVersion: kuadrant.io/v1beta1
kind: Kuadrant
metadata:
  name: kuadrant-sample-1
  namespace: kuadrant-system
spec:
  gatewayProvider:
    group: maistra.io/v2
    kind: ServiceMeshControlPlane
    name: basic
    namespace: istio-system
---
apiVersion: kuadrant.io/v1beta1
kind: Kuadrant
metadata:
  name: kuadrant-sample-2
  namespace: kuadrant-system
spec:
  gatewayProvider:
    group: install.istio.io/v1alpha1
    kind: IstioOperator
    name: istiocontrolplane
    namespace: istio-system
```

The issue with this simpler approach depends on the definition of "Kuadrant instance". While one instance implies one deployment of Authorino and (more importantly) one deployment of Limitador, sharing a functional component of Kuadrant across services meshes would not be possible. For rate limiting with no support for external storage (nor any kind synchronisation between instances), this limitation translates to no shared counters across different control planes, i.e. each service mesh sees a different count.

Perhaps the described issue is a minor issue or even no issue at all, given the closed aspect of service meshes. I.e., one could argue that a service mesh is supposed to self-contain all the management services it requires, in a way that instances of Authorino and Limitador (and therefore instances of Kuadrant – according to current definition) are not to be shared across different meshes anyway; ergo pointing one and only one gateway provider for each Kuadrant instance would suffice.

If, on the other hand, the premise above is found to be false, then either multiple gateways providers per instance of Kuadrant has to be supported or the definition of "Kuadrant instance" needs to be revised.

# Prior art
[prior-art]: #prior-art

N/A.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

1. Is having multiple service mesh control planes (i.e. multiple gateway providers) within the same cluster a real use case or rather just a technical possibility that we choose not to support?
2. Does a Kuadrant instance need to be able to target more than one control plane/gateway provider? (See [Rationale and alternatives](#rationale-and-alternative).)
3. Should targeting a control plane/gateway provider be required to be explicit (e.g. in a `Kuadrant` CR) or it can based on convention/heuristic (e.g. default to Istio)? _Note:_ I am personally inclined to it needs to be explicit, hence the proposal in this RFC.
4. What gateway providers should we initially support? _A:_ Istio and OpenShift Service Mesh (OSSM).
5. What other gateway providers could we aim to support in the future? _Possible answers:_ [Envoy Gateway](https://gateway.envoyproxy.io/v0.2.0/), [Contour](https://projectcontour.io), other API gateways not based on Envoy, ...
6. Is "gateway provider" an adequate term? What about supporting injecting Kuadrant into sidecar proxies in the future?

# Future possibilities
[future-possibilities]: #future-possibilities

Given the current support for Istio and OpenShift Service Mesh (OSSM), detected automatically by the Kuadrant Operator, it was not mentioned as an official [alternative](#rationale-and-alternative), however another option for approaching the problem of supporting multiple control plane/gateway providers in the same cluster that goes in the opposite direction of what's reasoned in this RFC so far – yet with its own comparative advantages – would be maintaining so-called "Kuadrant distributions". A distribution of Kuadrant would be made specifically for a particular gateway provider. E.g., there would be a version of Kuadrant dedicated exclusively to Istio (`kuadrant-istio`), another one for OSSM (`kuadrant-ossm`), and so forth.

Although I don't believe this is a route we want to pursue<sup>*</sup>, it's worth mentioning that such approach has the advantage of yielding simpler codebases that are easier to read, to maintain and ocasionally to drop. On the flip side, it also means having more codebases and versions to maintain and to sync across, and challenges to avoid too much repetition between the different "distributions".

<br/>

Regardless of the exact approach eventually adopted to support multiple gateway providers in the same cluster, future possibilities opened by this discussion involve assessing different providers we want to be compatible with. Istio-based service mesh control planes are obvious choices, but there are also [Envoy Gateway](https://gateway.envoyproxy.io/v0.2.0/), [Contour](https://projectcontour.io), other API gateways not based on Envoy, [Red Hat 3scale API Management](https://www.redhat.com/en/technologies/jboss-middleware/3scale), among multiple other potential integrations.

<br/>

<sub><sup>*</sup>TBC</sub>
