# RFC 0000 Custom underlying services

- Feature Name: `Custom underlying services`
- Start Date: 2023-04-26
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [Kuadrant/kuadrant-operator#164](https://github.com/Kuadrant/kuadrant-operator/issues/164)

# Summary
[summary]: #summary

Enable externally managed services to be leveraged by the kuadrant control plane to support some traffic policies, currently Authorization Policy and Rate Limit Policy.

# Motivation
[motivation]: #motivation

Decouple Kuadrant from any deployment settings of the underlying services backing kuadrant. To name a few, deployment settings are about replicas, (auto) scaling settings, logging, metrics, backend storage options, listener ports and transport protocols.

Support for multicluster rate limiting use cases where shared counters are required.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

One kuadrant control plane instance is represented by one Kuadrant custom resource. The main idea is to wire the underlaying services backing kuadrant with the kuadrant instance within the Kuadrant custom resource spec.

```yaml
apiVersion: kuadrant.io/v1beta1
kind: Kuadrant
metadata:
  name: <kuadrant-name>
  namespace: <kuadrant-namespace>
spec:
  authorinoRef:
      name: <authorino-name>
      namespace: <authorino-namespace>
  limitadorRef:
      name: <limitador-name>
      namespace: <limitador-namespace>
```

**References targeting custom resources**

The **target references** are themselves **custom resources** representing an instance of the service. Specifically:

* `authorinoRef` contains a reference to [Authorino CR](https://github.com/Kuadrant/authorino-operator/blob/6e7b0005a35107aafc01f2cfb7d7e235ea237bbf/config/crd/bases/operator.authorino.kuadrant.io_authorinos.yaml) provided by the [Authorino Operator](https://github.com/Kuadrant/authorino-operator).
* `limitadorRef` contains a reference to one [Limitador CR](https://github.com/Kuadrant/limitador-operator/blob/d68a13c12a5ab76cc93547a8433a49cfef92f18c/api/v1alpha1/limitador_types.go) provided by the [Limtiador Operator](https://github.com/Kuadrant/limitador-operator)

Referencing a custom resource provides good level of abstraction and decoupling. The kuadrant control plane must obtain all required information from the targeted custom resources to implement the integration of the services in the data plane. For instance, the Limitador CR provides useful information in the `status` section. First and foremost, if the service is available (condition type `Ready`).

```yaml=
apiVersion: limitador.kuadrant.io/v1alpha1
kind: Limitador
metadata:
  name: limitador
  namespace: kuadrant-system
spec: {}
status:
  conditions:
  - lastTransitionTime: "2023-05-02T13:23:45Z"
    message: Limitador is ready
    reason: Ready
    status: "True"
    type: Ready
  observedGeneration: 1
  service:
    host: limitador-limitador.kuadrant-system.svc.cluster.local
    ports:
      grpc: 8081
      http: 8080
```

**Only one reference**

From a kuadrant CR you can only **reference one**, and only one, limitador and authorino instance. This is a design decision taken for the current needs and powered by its simplicity. In the future, if there are use cases for that, one kuadrant instance could manage multiple, let's say Limitador, instances and assign those instances to the managed gateways based on some rules. Something like, this gateway A gets assigned Limitador A and this gateway B gets assigned Limitador B for whatever reason. Currently, this scenario is out of scope.

**References are optional**

The references are **optional**. In the absence of a reference, the Kuadrant operator will deploy and operate a default deployment (yet to be defined) to support the kuadrant policies.

**References are namespaced**

The references can target custom resources in namespaces other than the Kuadrant's namespace.

**Watch external services**

The Kuadrant's control plane should be watching referenced resources for changes. and react upon any change applying any update to make sure the integration is successful.

---
Explain the proposal as if it was implemented and you were teaching it to Kuadrant user. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how a user should *think* about the feature, and how it would impact the way they already use Kuadrant. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing and new Kuadrant users.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

TODO

---
This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- How error would be reported to the users.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

TODO

---

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?

  * Expose Limitador CRD and Authorino CRD in the Kuadrant CRD
  * Split Limitador CRD in Limitador CRD and Limits CRD. The limitador CRD would take care of deployment settings and Limits CRD would be used to configure limits. The kuadrant control plane would only need to create Limit CRs.

- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

TODO

---

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- Does another project have a similar feature?
- What can be learned from it? What's good? What's less optimal?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other tentatives - successful or not, provide readers of your RFC with a fuller picture.

Note that while precedent set by other projects is some motivation, it does not on its own motivate an RFC.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

TODO

---

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

TODO

---

Think about what the natural extension and evolution of your proposal would be and how it would affect the platform and project as a whole. Try to use this section as a tool to further consider all possible interactions with the project and its components in your proposal. Also consider how this all fits into the roadmap for the project and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but otherwise related.

Note that having something written down in the future-possibilities section is not a reason to accept the current or a future RFC; such notes should be in the section on motivation or rationale in this or subsequent RFCs. The section merely provides additional information.
