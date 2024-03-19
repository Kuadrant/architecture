# RFC 0000

- Feature Name: `multiple kuadrant instances`
- Start Date: 2023-01-12
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/issues/0000)

# Summary
[summary]: #summary

This RFC proposes a new kuadrant architecture design to enable **multiple kuadrant instances** to be running in a single cluster.

![](https://i.imgur.com/ZsPibfO.png)

# Motivation
[motivation]: #motivation

The main benefit of multiple Kuadrant instances in a single cluster is that it allows dedicated Kuadrant's services for tenants.

Dedicated Kuadrant deployment brings lots of benefits. Just to name a few:
* Protection against external traffic load spikes. Other tenant's traffic spikes does not affect Authorino/Limitador throughput and delay as it would when shared.
* No need to have cluster administrator role to deploy a kuadrant instance. One tenant administrator can manage gateways, Limitador and Authorino instances (including deployment modes).
* The cluster administrator gets control and visibility across all the Kuadrant instances, while the tenant administrator only gets control over their specific gateway(s), Limitador and Authorino instances.
* (looking for ideas for more benefits)...

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### Kuadrant instance definition
![](https://i.imgur.com/BfOXfnB.png)

A kuadrant is composed of:
* One Limitador deployment instance
* One Authorino deployment instance
* A list of dedicated gateways.

Some properties to highlight:

* The policies are not included as part of the kuadrant instances.
* The Kuadrant instance is not enclosed by k8s namespaces.
* Gateways are not shared between kuadrant instances. Each gateway is managed by a single kuadrant instance.
* The control plane has cluster scope and will be shared between instances. In other words, it is only in the data plane that each Kuadrant instance has dedicated services and resources.
* Each kuadrant instance owns one instance (possibly multiple replicas, though) of Limitador and one instance of Authorino. Those instances are shared among all gateways included in the kuadrant instance.

In the following diagram policies RLP 1 and KAP 1 are applied in the instance *A* and the policies RLP 2 and KAP 2 are applied in the instance *B*.

![](https://i.imgur.com/yChVsT6.png)

### All the gateways referenced by a single policy must belong to the same kuadrant instance

The Gateway API allows, in its latest version  [v1beta1](https://gateway-api.sigs.k8s.io/v1alpha2/references/spec/#gateway.networking.k8s.io/v1beta1.CommonRouteSpec), an HTTPRoute to have multiple gateway parents. Thus, a kuadrant policy might technically target multiple gateways managed by multiple kuadrant instances. Kuadrant does **not** support this use case.

![](https://i.imgur.com/ZpsBf4i.png)

The main reason is related to the rate limiting capability. The limits specified in the RateLimit Policy would be enforced per kuadrant instance basis (provided by Limitador instance). Thus, traffic hitting one gateway would see different rate limiting counters than traffic hitting the other gateway. The user would expect X rps and actually it would be X rps per gateway. For consistency reasons, when this configuration happens, the control plane will reject the policy.

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

```yaml
apiVersion: kuadrant.io/v1beta1
kind: Kuadrant
metadata:
  name: kuadrant-a
spec:
  gatewaysSelector:
    matchLabels:
      app: kuadrant-a
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Wiring Kuadrant policies with Kuadrant instances
Technically, the Kuadrant policies do not belong to any Kuadrant instance. At any moment of time, one policy can switch the targeted network resource specified in the `spec` from one gateway to another. Directly or indirectly via the HTTPRoute. The target references are dynamic by nature, so is the list of gateways to which kuadrant policies should apply.
Thus, the Kuadrant's control plane needs a procedure to associate a policy with **one** kuadrant instance at any time. When the control plane knows which kuadrant instance is affected, the policy rules can be used to configure the Limitador and Authorino instances belonging to that kuadrant's instance. Since the associated kuadrant instance of a policy is dynamic by nature, this procedure must be executed on  every event related to the policy.

When the policy's `targetRef` targets a Gateway, there is a direct reference to the gateway.

When the policy's `targetRef` targets an HTTPRoute, Kuadrant will follow the [`parentRef`](https://gateway-api.sigs.k8s.io/v1alpha2/references/spec/#gateway.networking.k8s.io%2fv1beta1.CommonRouteSpec) attribute which should be a direct reference to the gateway or gateways.

Given a gateway, Kuadrant needs to find out which Kuadrant's instance is managing that specific gateway. By design, Kuadrant knows it is only one. There are at least two options to implement that mapping:
* Read all Kuadrant CR objects and the first one that matches label selector.
  * This approach works as long as the control plane ensures that each gateway is matched by only one kuadrant gateway selector. The control plane must reject any new kuadrant instance matching a gateway already "taken" by other kuadrant instance.
* Add annotation in the gateway with a value of the Name/Namespace of the Kuadrant CR.
  * This approach is commonly used. Requires annotation management.


### Just one external auth stage and one rate limiting stage
Kuadrant configures the gateway with a single external authorization stage (backed by Authorino) and a single external rate limiting stage (backed by Limitador).
Multiple rate limit or authN/AuthZ stages involving multiple instances of Authorino and Limitador can be implemented, technically speaking.
Until there is a real use case and it is strictly necessary, this scenario is discarded. The main reason is about complexity. It is already complex enough to reason about rate limiting and auth services having a single stage. Adding multiple rate limiting stages, or hitting multiple Limitador instances in a single stage (doable with the WASM module) makes it too complex to reason about observed behavior. Currently there is no use case to require that complex scenario.

# Drawbacks
[drawbacks]: #drawbacks

Multitenancy is not a requested capability from users. Usually ingress gateways are shared resources managed by cluster administrators and a cluster may have only few of them. It is also a cluster admin task to route traffic to the ingress gateway. Cluster users usually do not control the life cycle of the ingress gateways in order to have their own Kuadrant instance.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?

The gateway is the top class entity in the design, not the policy. The API protection happens at the gateway and the configuration needs to be done at the gateway. This kuadrant instance design protects the gateway isolating them from other instances (mis)configurations or traffic spikes.

- What other designs have been considered and what is the rationale for not choosing them?

TODO

- What is the impact of not doing this?

This design is a step forward in a consistent API to protect other service APIs. It makes easier to protect any API, no matter traffic nature. Either north-south or east-west. It makes easier to have the scenario where cluster users deploy their own (not ingress) gateways and enable API protection declaratively.

# Prior art
[prior-art]: #prior-art

TODO

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?

Validate the main points of the design:
a) Single auth/ratelimit stage in the processing pipeline of the gateway

b) Gateways are not shared among kuadrant instances

- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?

The wiring mechanism.

- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

Supporting multiple gateway providers #7

# Future possibilities
[future-possibilities]: #future-possibilities

TODO
