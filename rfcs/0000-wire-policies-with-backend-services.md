# RFC 0000

- Feature Name: `wire_policies_with_backend`
- Start Date: 2022-11-04
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/issues/0000)

# Summary
[summary]: #summary

This RFC proposes a mechanism to wire Kuadrant policies with rate limiting and authN/authZ services.

# Motivation
[motivation]: #motivation

After the PR [#48](https://github.com/Kuadrant/kuadrant-operator/pull/48) was merged,
Kuadrant suffered an unwanted side effect: Kuadrant's policies only worked when kuadrant was installed in the `kuadrant-system` namespace.
This issue comes from the fact that the policy controllers are no longer deployed as components of a particular Kuadrant instance.
Instead, the policy controllers live at the Kuadrant's operator pod and they are up&running even if there is no Kuadrant instance running in the cluster.
The very source issue of this "side effect" is the design about how backend services were wired with the Ratelimit/Auth policies.
The design allowed one kuadrant instance to be installed in any namespace, however, the design only allowed one kuadrant instance to be running in the cluster.

### How it worked before the merge #48
When an instance of Kuadrant, represented by a Kuadrant custom Resource (RC), was created, the following workflow was run by the Kuadrant operator:
* Read the Kuadrant custom resource, paying attention to the namespace. Let's call `K` the namespace where the Kuadrant CR is created.
* Deploy one Limitador instance in the `K` namespace
* Deploy one Authorino instance in the `K` namespace
* Register Authorino instance living in `K` in the Istio system as an external authorization service.
* Deploy the RateLimitPolicy controller passing as env var the address of the limitador instance in the `K` namespace.
* Deploy the AuthPolicy controller

When the user created a rate limit policy, the controller already knew about the Limitador's location (name and namespace) to configure it accordingly with the spec of the policy.

Authorino is a k8s controller and the Kuadrant's operator deploys it in cluster-wide mode
without any [sharding](https://github.com/Kuadrant/authorino/blob/main/docs/architecture.md#sharding) defined.
When the user created an auth policy, the controller does not need to know where authorino lives because a) it assumes that there is only one Authorino instance (which might be wrong as well) and b) the controller assumes that Authorino is watching the entire cluster without filtering.
Thus, the controller manages an AuthConfig object in the hard-coded `kuadrant-system` namespace (which btw it is also wrong).

### How it works after the merge #48
When the policy controllers were moved to the operator's pod, one of the design's requirements was unmet: the controllers know at deploy time Limitador's location. Thus, causing the issue.
The design assumed one policy controller instance per limitador instance. The policy controller got limitador's location at boot time via an environment variable.
After the policies merge into the operator's pod, the policies controllers will be a singleton instance (one pod, one container) at the cluster level.
Regardless of the kuadrant's support for multiple or single limitador/authorino instances, the policies controllers will be running in a single pod in the entire cluster.
Even if kuadrant only supports a single limitador/authorino instance,
the policies controllers still need to know the location of the limitador/authorino instances.

Therefore, a new design is needed that wires the user created policies with an existing limitador/authorino instance. Even though, currently, this wiring up works for the AuthPolicy, it is done under the assumption of a single Authorino watching all the cluster for the AuthConfig objects, which is a sub-optimal design.

### Potential Scenarios

* **Kuadrant supports only one instance deployed in a hard-coded namespace**

No need to wire, as authorino and limitador location is well known. All policies will be linked to the same instance of Limitador/Authorino.

* **Kuadrant supports only one instance deployed in a configurable namespace**

Wiring is needed. The policy controllers do not know where Limitador/Authorino live. All policies will be linked to the same instance of Limitador/Authorino.

* **Kuadrant supports multiple instances in a cluster**

Wiring is needed. There should be a way to know the Limitador/Authorino instance only by reading the policy.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The proposal is to make Kuadrant to support multiple instances in a cluster.

This proposal is based on a simple design decision: One gateway runs one and only one kuadrant's external auth service (Authorino) and only one kuadrant's external rate limit service (Limitador).

Multiple rate limit or auth stages involving multiple instances of Authorino and Limitador can be implemented.
Until there is a real use case and it is strictly necessary, this scenario is discarded for implementing kuadrant's protection.
The main reason is about complexity. It is already complex enough to reason about rate limiting and auth services having a single stage. Adding multiple rate limiting stages, or hitting multiple Limitador instances in a single stage (doable with the WASM module) makes it too complex to reason about observed behavior. Currently there is no use case to require that complex scenario.

A kuadrant instance includes:

* One Limitador deployment instance
* One Authorino deployment instance
* A list of dedicated gateways. Those gateways cannot be shared between multiple kuadrant instances.

A diagram to ilustrate some concepts:

![](https://i.imgur.com/y7gQfRa.png)

Highlights:
* The Kuadrant instance is not enclosed by k8s namespaces.
* One gateway can belong to (be managed by) one and only one kuadrant instance.
* One kuadrant instance does not own rate limit policies or auth policies.
* Each kuadrant instance owns one instance (possibly multiple replicas, though) of Limitador and one instance of Authorino. Those instances are shared among all gateways managed by the kuadrant instance.
* The traffic routed by HTTPRoute 1 through the gateway A will be protected by RLP 1 and KAP 1, using Limitador and Authorino instances located at the namespace K1.
* The traffic routed by HTTPRoute 2 through the gateway B will be protected by RLP 2 and KAP 2, using Limitador and Authorino instances located at the namespace K1.
* The traffic routed by HTTPRoute 2 through the gateway C will be protected by RLP 2 and KAP 2, using Limitador and Authorino instances located at the namespace K2.
* The HTTPRoute 2 example shows that when the traffic for the same service is routed through multiple gateways, at least for rate limiting, Kuadrant cannot keep consistent counters. The user would expect X rps and actually it would be X rps per gateway.
* The traffic matching HTTPRoute 3 will be protected by RLP 3 and KAP 3. These policies are both gateway targeting policies. Gateway targeted policies will be applied only to traffic matching at least one HTTPRoute.
* The traffic hitting the Gateway E3 will __not__ be protected by any policies even though RLP 4 and KAP 4 target the Gateway E. Gateway targeted policies will be applied only to traffic matching at least one HTTPRoute. Since no HTTPRoute is targeting Gateway E, policies have no effect.

### The Kuadrant CRD

Currently, the Kuadrant CRD has an empty spec.

```yaml
apiVersion: kuadrant.io/v1beta1
kind: Kuadrant
metadata:
  name: kuadrant-sample
spec: {}
```

According to the definition above of the kuadrant instance,
the proposed new Kuadrant CRD would add a label __selector__ to specify which gateways that instance would manage.
Additionally, for dev testing purposes, the Kuadrant CRD would have image URL fields for the kuadrant controller, Limitador and Authorino components. A Kuadrant CR example:

```yaml
apiVersion: kuadrant.io/v1beta1
kind: Kuadrant
metadata:
  name: kuadrant-sample
spec:
  controlPlane:
    image: quay.io/kuadrant/kuadrant-operator:mytag
  limitador:
    image: quay.io/kuadrant/limitador:mytag
  authorino:
    image: quay.io/kuadrant/authorino:mytag
  gatewaysSelector:
    matchLabels:
      app: kuadrant
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

- `Deploy the RateLimitPolicy controller passing as env var the address of the limitador instance in the K namespace.`
  * "This is meant for the WASM plugin and the istio `envoy_filter` I reckon, no?"
- `Deploy the AuthPolicy controller`
  * "Same for this one ^^"
- `The traffic routed by HTTPRoute 2 through the gateway B will be protected by RLP 2 and KAP 2, using Limitador and Authorino instances located at the namespace K1`
  * "It's a bit tricky to think of a use case of an underlying service, rate limited by 2 instances of limitador (not sure if applies to authorino too) when it comes to the same HTTPRoute with 2 parentRefs of GW managed by different Kuadrant instances"

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would be and how it would affect the platform and project as a whole. Try to use this section as a tool to further consider all possible interactions with the project and its components in your proposal. Also consider how this all fits into the roadmap for the project and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but otherwise related.

Note that having something written down in the future-possibilities section is not a reason to accept the current or a future RFC; such notes should be in the section on motivation or rationale in this or subsequent RFCs. The section merely provides additional information.
