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

> [!TODO]
> Some initial SDK usage example


The example below is a modified example from the 
[controller-runtime's own example](https://github.com/kubernetes-sigs/controller-runtime/blob/main/examples/builtins/controller.go#L39).

```go
func (r *reconcileMetapolicy) Reconcile(ctx context.Context, request reconcile.Request, kuadrant *KuadrantContext) (reconcile.Result, error) {
  // Fetch the MetaPolicy from the cache
  rs := &user.MetaPolicy{}
  err := r.client.Get(ctx, request.NamespacedName, rs)
  if errors.IsNotFound(err) {
    log.Error(nil, "Could not find MetaPolicy")
    return reconcile.Result{}, nil
  }

  if err != nil {
    return reconcile.Result{}, fmt.Errorf("could not fetch MetaPolicy: %+v", err)
  }

  // Set the label if it is missing
  if rs.Labels == nil {
    rs.Labels = map[string]string{}
  }
	
  // resolve the CEL expression using the `KuadrantContext`
  label := kuadrant.evaluateExpression("kuadrant.gatewaysFor(self)[0].metadata.name")

  if rs.Labels["gateway"] == label {
    return reconcile.Result{}, nil
  }

  // Update the MetaPolicy
  rs.Labels["gateway"] = label
  err = r.client.Update(ctx, rs)
  if err != nil {
    return reconcile.Result{}, fmt.Errorf("could not write MetaPolicy: %+v", err)
  }

  return reconcile.Result{}, nil
}
```
It on differs in three ways from the original example:

1. The `Reconcile` signature takes an additional parameter, the `kuadrant *KuadrantContext`;
1. It acts upon an hypothetical `user.MetaPolicy`;
1. It uses the `evaluateExpression` to resolve the CEL expression `kuadrant.gatewaysFor(self)[0].metadata.name` instead
   of using a fixed string for the label's value.

Because that expression is evaluated upon creation, the `KuadrantRuntime` will make sure to reconcile whenever that cel
expression's evaluated result would change; in this particular example whenever the name of the first `Gateway` changes.

> [!NOTE]
> Below is the proposed inital integration point for `MetapolicyController`s. The idea is to keep the deployment model
> fairly open moving forward. As of now, this would be the deployment model for _our own_ *Metapolicies*

To deploy our custom reconciler, we need to let the Kuadrant operator know about it, by adding it to the `Kuadrant` CR:

```yaml
apiVersion: kuadrant.io/v1beta2
kind: Kuadrant
metadata:
  name: kuadrant-sample
spec:
  plugins:
    userMetaPolicy:
      deployment: embedded
```

The `UserMetaPolicy` will need to be packaged with a custom Docker image containing the plugin. It'll be automatically
registered with the Kuadrant Controller from its own config file.

> [!TODO]
> Support out-of-image deployments of plugins, i.e. ones running as their own container. Which then also means we need
> to deal with failures et al, so probably adding the necessity for more configuration options for these plugins

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


### Deployment

> [!IMPORTANT]
> This would initially be deployed as their own processes running alongside the kuadrant controller. They'd communicate
> though unix sockets, over gRPC & gRPC streams (for the eventing).
>
> Later, we could support different deployment models, as all that'd be needed is being able to use gRPC essentially.

- [ ] Known location for plugins to live in; one per directory
- [ ] Unix socket opened within the same location?

# Drawbacks
[drawbacks]: #drawbacks

> [!todo]
> Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

> [!todo]
> - Why is this design the best in the space of possible designs?
> - What other designs have been considered and what is the rationale for not choosing them?
> - What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

> [!todo]
> Discuss prior art, both the good and the bad, in relation to this proposal.
> A few examples of what this can include are:
>
> - Does another project have a similar feature?
> - What can be learned from it? What's good? What's less optimal?
> - Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.
>
> This section is intended to encourage you as an author to think about the lessons from other tentatives - successful or not, provide readers of your RFC with a fuller picture.
>
> Note that while precedent set by other projects is some motivation, it does not on its own motivate an RFC.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

> [!todo]
> - What parts of the design do you expect to resolve through the RFC process before this gets merged?
> - What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
> - What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

> [!todo]
> Think about what the natural extension and evolution of your proposal would be and how it would affect the platform and project as a whole. Try to use this section as a tool to further consider all possible interactions with the project and its components in your proposal. Also consider how this all fits into the roadmap for the project and of the relevant sub-team.
>
> This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but otherwise related.
>
> Note that having something written down in the future-possibilities section is not a reason to accept the current or a future RFC; such notes should be in the section on motivation or rationale in this or subsequent RFCs. The section merely provides additional information.

[1]: https://gateway-api.sigs.k8s.io/
[2]: https://gateway-api.sigs.k8s.io/geps/gep-2649/
[3]: https://cel.dev
[4]: https://github.com/Kuadrant/policy-machinery
