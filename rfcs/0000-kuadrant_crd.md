# RFC Template

- Feature Name: Control Policy management at the Kuadrant CR level
- Start Date: 2022-11-15
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/issues/0000)

# Summary
[summary]: #summary

Not every use cases requires, or even would want to permit usage of all of Kuadrant's capabilities to
be enabled on the target cluster. This proposal aims at giving users control, through the `kuadrant`
custom resource, of what gets enabled on the cluster when deploying Kuadrant.

# Motivation
[motivation]: #motivation

The rational behind this proposal is that not every deployment requires all of Kuadrant features to be
supported; currently `RateLimitPolicy` and `AuthPolicy`. There is no reason to impose the workload to
run on the cluster if it isn't need, but there might be more reasons why explicitly disabling one
feature might be desirable: disabling the users to actually apply one or the other policy to their
traffic. Authorization for instance could always be implemented at the application level as per the
organization desire, while rate limiting is achieved through `RateLimitPolicy` attached to the network
resources. Just as the opposite situation might be applicable to an organization that wants to do
authn/z by leveraging Kuadrant, but have no use cases for rate limiting.

This proposal aims not only at defining a way to disable one of these features at deploy time, but
also clarify what the behavior would be when such a state transitions while some policy are already
existent; what the user experience would be when trying to apply a policy that's been explicitly
disabled by the cluster administrator when Kuadrant was installed.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When applying the `Kuadrant` resource to the cluster for the first time, you have the option to
disable the support for a policy by specifying the following in the `spec` of the resource:

```yaml
authentication:
  policy: disabled
rateLimiting:
  policy: disabled
```

The `policy` key of either of the service, `authentication` and `rateLimiting`, defaults to `enabled`.
That value can be specified as `disabled`. The behavior on both services is consistent: it disables
the support for the policy backed by the service, i.e. `AuthPolicy` and `RateLimitPolicy`
respectively.

When disabling `RateLimitPolicy`, Kuadrant will not create the `Limitador` object used by the
`limitador-operator`. Disabling the `AuthPolicy` yields the same result for the `Authorino` object.
Additionally it will avoid registering the [external
authorizer](https://istio.io/latest/docs/tasks/security/authorization/authz-custom/) with Istio. The
both policies API will be exposed and the reconcilers will still be registered. Creating a policy that
is disabled would be denied.

A disabled policy can be enabled by changing the value of the `policy` key. Flipping the value can
only happen when no policy object exist, e.g. in order to disable the `authentication`, no
`AuthPolicy` can exist or this would fail.

While no default `Limitador` or `Authorino` object would be created, a user can absolutely create one
or the other and use their APIs directly. In the case of authentication, the user would need to
register the external authorizer with Istio themselves.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Changing the definition of the `kuadrant` custom resource is relatively straight forward. It is
intended that the way to enable or disable the support for either policy is not a boolean value. Using
an enumeration, i.e. `disabled` or `enabled`, enables growing the set of values later.

## Installation

On initial installation, honoring one or the other setting would be fairly straight forward as well:

The both CRDs, as well as their reconcilers all get registered whether the `policy` support is
`enabled` or `disabled`.

- `enabled` The Kuadrant controller makes sure the external authorizer gets registered with Istio and
  makes sure there is an `Authorino` and/or `Limitador` object without any changes as it does today.
- `disabled` None of the above would happen. Most importantly, any missing part required for the above
  shouldn't have the reconciliation fail and prevent further work it requires to do (as reconciling
  another policy that's being `enabled`).

Additionally a `ValidatingWebhookConfiguration` gets registered for each CR's `CREATE` operation for
the entire cluster. The webhook will only allow admissions of resources for which the policy is
currently enabled. `UPDATE` and `DELETE` should remain unhindered, as in order to transition the
policy support requires no policies to be present.

If any of the following steps fails, all previously applied changes need to be unrolled and the
`status` block of the CR would clearly reflected what changed and that Kuadrant is currently _not_
operational. This is to be considered an atomic operation, either all gets successfully deployed or
nothing is.

If a policy is disabled but a policy is being added, when disallowing the admission, the webhook would
use a response code `501 Not Implemented` with the additional `message` indicating the policy is
disabled and as such can't be honored.

### In an OSSM cluster

The deployment of `Kuadrant` should fail if `authentication` isn't explicitly disabled.

## State transitions of policy support

If the `Kuadrant` object gets updated to change the support of a policy, the change would only be
considered if no such policy exists in the cluster. No state transitions with regards to policy
support is possible if policies of the given type are present.

While it would be good to be able to enable the support for a policy while such policies already
exist, there is much to consider as to what this would entail - here are a few things to consider:

- can we apply all these policies?
- what ordering should be considered to apply them?
- what would a conflict mean?
- what would disabling `authentication` for a cluster, while existing routes are currently protected by these?

Requiring to first remove all policies and then only safely disable or enable the support for that
kind of policy seems reasonable at this stage and can always be added later, which clearly specified
behavior.

### Disabling the support for a policy

When disabling the support for a policy, merely the admission webhook's behavior changes. The
`Authorino` resource or `Limitador` ones remain untouched.

# Drawbacks
[drawbacks]: #drawbacks

The flipside of the flexibility this provides users with, this introduces quite some complexity. Hence
the desire to keep this very rigid as to what are the preconditions to change the support for a
policy.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Possibly the gain of not running unnecessary services on the cluster isn't big enough for us to
justify adding this complexity. Yet, as of today, we need a way to disable `authentication` in the
case of running within an OSSM backed cluster.

We could disable `authentication` on a heuristic base approach: running within OSSM? Disable
`authentication`. The user experience seems less than desirable and confusing as best, as in future
version this would probably change. That being said we're still pre-v1. We'd also need to find a way
to know very early on what environment we are being deployed in.

# Prior art
[prior-art]: #prior-art

While I couldn't find, probably because of lacking knowledge about similar use-cases, anything that
contradicts what this approach proposes, I couldn't see how what this would entail goes against any
Kubernetes best practices for Operators.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Is this really what we want?
- Can we get authentication supported in OSSM and possibly pun on this a little longer?
- Should we have a deployment that skips authentication entirely when in an OSSM cluster?

# Future possibilities
[future-possibilities]: #future-possibilities

- relaxing the requirements for state transitions on policy support
- support some kind of lazy support for policy, where limitador and/or authorino are only lazily
  deployed to the cluster.
