# An Open-Architecture for Kuadrant 

- Feature Name: open-arch
- Start Date: 2025-02-26
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/issues/0000)

# Summary
[summary]: #summary

Kuadrant has been so far mostly focused on specific areas: AuthN/Z, DNS, TLS and Rate Limiting. Most of these areas
evolved in silos, independently from each other. There were some efforts to standardize certain aspects and get some
of these features to work somewhat in tandem, in the form of authenticated rate limiting, which integrates two of
the areas Kuadrant concerns itself with.

The intent of this proposal is to formalize Kuadrant as a _platform_ to extend [Gateway API][1] functionality through the use
of [Policy Attachment][2]. 

# Motivation
[motivation]: #motivation

A few components emerged overtime that play a central role in _how_ functionality is exposed in Kuadrant:

- Policies themselves;
- [CEL][3] & Well-Known Attributes;
- The [DAG][4] representing the current state of the cluster regarding Gateway API objects, as well as the policies
  attached to them;
- The [wasm-shim][https://github.com/Kuadrant/wasm-shim/?tab=readme-ov-file#sample-configuration] through its
  configuration, enabling Kuadrant's own "filter chain" equivalent within the Gateway.

The idea would be to expand on these components and formalizing their interfaces to their respective consumers. While
these are solid building blocks, as proven by our current experience with those in building existing features, we would
need to open Kuadrant up to make use of these to support a modular model for policies.

Initially, users would only be exposed to these changes through _metapolicies_, domain specific policies that expose a
higher-level abstraction to our existing policies.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## What's a *Metapolicy*?

A *Metapolicy* is a policy just like any other [Gateway API Policy][2], other than it only produces one or more "core
Kuadrant policies". A *Metapolicy* is managed by a `MetapolicyController` that will interact with the
`KuadrantController` to integrate seamlessly in the ecosystem.

Think of a `MetapolicyController` being a pure function that takes the custom *Metapolicy* and the Gateway API network
objects it targets as input, and outputs one or more Kuadrant Policies as a result.

## Implementing a custom *Metapolicy*

Fundamentally a *Metapolicy* isn't much different from a regular Kubernetes [Custom
Resource](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/), and as such will be
managed by some controller. But unlike a regular [Kubernetes
Controller](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#custom-controllers),
it also needs to reconcile when changes to properties of the Gateway API it uses are observed. For that reason, the
controller for a custom *Metapolicy* registers itself with the `KuadrantController`.

> [!NOTE]
> Below is the proposed inital integration point for `MetapolicyController`s. The idea is to keep the deployment model
> fairly open moving forward. As of now, this would be the deployment model for _our own *Metapolicies*


> [!TODO]
> Some initial SDK usage example


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Plugin architecture

### gRPC to controller

- [ ] Unix socket for "in-pod/embedded" plugins
- [ ] Declarative way to load plugins
- [ ] Provide a decorated [Controller Runtime](https://github.com/kubernetes-sigs/controller-runtime) Golang that
  provides the changes triggered by the DAG

### gRPC streams for eventing on the DAG

- [ ] Subscriptions infered from CEL and the current CR being reconcilied

### Typed DSL in CEL

> [!IMPORTANT]
> Provide both the `MetapolicyCR` & the `GwAPIContext` instances relative to a reconciliation
> The `GwAPIContext` would not only be responsible for resolving the [CEL expressions][3] but also deriving the
> necessary subscriptions needed to inform the controller when a reconciliation is needed because of changes to the
> Gateway API objects queried

- [ ] Support existing `context`s in the CEL interpreter's `Env`
- [ ] Find a way to express the `context`s present/set for each CEL


# Drawbacks
[drawbacks]: #drawbacks


# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

# Prior art
[prior-art]: #prior-art


# Unresolved questions
[unresolved-questions]: #unresolved-questions


# Future possibilities
[future-possibilities]: #future-possibilities


[1]: https://gateway-api.sigs.k8s.io/
[2]: https://gateway-api.sigs.k8s.io/geps/gep-2649/
[3]: https://cel.dev
[4]: https://github.com/Kuadrant/policy-machinery
