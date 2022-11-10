# RFC Policy scoped to the whole targeted Route

- Feature Name: Policy scoped to the whole targeted Route
- Start Date: 2022-11-09
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/issues/0000)

# Summary
[summary]: #summary

Currently both `RateLimitPolicy` and `AuthPolicy` allow for a policy to only affect a _subset_ of the
traffic on a targeted `HTTPRoute`. This proposal aims at simplifying the scope of a policy to target
_all_ the traffic of a route.

# Motivation
[motivation]: #motivation

As per the specification, a given resource will always need to support multiple policies targeting it.
Even if we only support a single e.g. `RateLimitPolicy` per resource, there will alway be a
possibility for another policy being defined else where in that resource's hierarchy: e.g. _rlp1_
targeting the `Gateway`, while _rlp2_ targeting the `HTTPRoute` being attached to the `Gateway`. Until there is an `HTTPRoute` tho, _rlp1_ will have no semantic, as there is no traffic to rate limit. When an `HTTPRoute` is attached to the `Gateway` is the point when that _rlp1_ becomes applicable. If that `HTTPRoute` also defines a `RateLimitPolicy`, the two get merged, according to the `default` & `override` rules of the specification. 

The idea is to keep things simple (if only for now), so that when traffic is routed through an `HTTPRoute` all applicable policies of a `kind` have been merged into a single one _and_ that single policy then applies to the whole traffic of that route.

The current approach, which lets each policy define an arbitrary subset of the traffic to which it applies, makes it impossible to know how to merge the different policies into one. Each policy's set requires to know how to applicable it becomes to the union and/or the intersection of the said subsets. While a heuristic could be applied, there are use-cases where the other option will be desirable. The question then becomes which policy is the authority as to decide which it should be. In order to avoid having to answer these questions, this proposal looks at not letting a policy discriminate which part of traffic it applies to: it's the whole traffic, always.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Traffic to your service is ultimately routed when an `HTTPRoute` binds at least one  `Gateway`, thru the [`parentRefs`](https://gateway-api.sigs.k8s.io/references/spec/#gateway.networking.k8s.io/v1beta1.ParentReference), and wires traffic to the [`backendRef`](https://gateway-api.sigs.k8s.io/references/spec/#gateway.networking.k8s.io/v1beta1.HTTPBackendRef), i.e. your service, according to the [`HTTPRouteMatch`](https://gateway-api.sigs.k8s.io/references/spec/#gateway.networking.k8s.io/v1beta1.HTTPRouteMatch) of a [`HTTPRouteRule`](https://gateway-api.sigs.k8s.io/references/spec/#gateway.networking.k8s.io/v1beta1.HTTPRouteRule) define on that `HTTPRoute`. All the traffic matched by _any_ `HTTPRouteRule` of a given `HTTPRoute` will be affected by which ever `RateLimitPolicy` or `AuthPolicy` you attach to it.

Any Kuadrant `Policy` will apply to the entirety of the traffic that is routed through the `HTTPRoute` it is eventually attached to. A `Policy` can be attached directly to an `HTTPRoute`. It will apply its specification according to the `override` and `default` section of the `Policy`'s `spec`. Let's use a simple example:

```yaml
kind: FakeRLPolicy
spec:
  override:
    enabled: true
  default:
    enabled: false
    rps: 42
  targetRef:
    kind: Gateway
    name: my-gw
```

This `FakeRLPolicy` defines two values for the `enabled` configuration. Since there is no other `FakeRLPolicy` attached to the `my-gw` `Gateway`, its namespace and so forth, see [the specification for the hierarchy](https://gateway-api.sigs.k8s.io/references/policy-attachment/#hierarchy), the `override` value for `enabled` precedes and as such the `enabled` flag will be `true`. The `rps` value will be `42` from the `default` section, as it is omitted in the `override`s. The resulting policy (modulo the `targetRef`) to be applied would look like this:

```yaml
enabled: true
rps: 42
```

>**Note:** that the `default`'s `enabled` being specified here is useless. We're just using it as an example to introduce this concepts.

That policy tho has no traffic to police yet. Unless an `HTTPRoute` binds that `Gateway` to a `Backend`, no traffic can exist and as such can't be applied to by the policy. As such Kuadrant will _not_ create any further of its custom resources (`CR`) until that happens.

Say, we now have a `Backend` (e.g. `foo`) wired to the `my-gw` using some `HTTPRoute` (e.g. `foo-route`), none of which have yet any `FakeRLPolicy` attached. Yet, since the `Gateway` used as a policy defined, Kuadrant will create a `RateLimit` `CR` and eventually configure `Limitador` to apply a rate limit of 42 requests per second, for traffic routed to that `foo` `Backend`. The `RateLimit` will be derived from the resulting policy as described above: rate limit enabled at a 42 rps. Which traffic will be rate limited, will be matching the union of all the  `HTTPRouteMatch` of each `HTTPRouteRule` as defined in that `HTTPRoute`. 

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: foo-route
spec:
  parentRefs:
  - name: my-gw
  hostnames:
  - "example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /foo
    backendRefs:
    - name: foo
      port: 8080
```

With `foo-route` declaring a single hostname (`example.com`) and a single `HTTPRouteMatch` (anything starting with `/foo`), the rate limiting will apply to the same traffic: `example.com/foo*`, so that `foo` will not see more than 42 rps on port `8080`.

Given the `override` on the `my-gw` `FakeRLPolicy` there no way for our route to disable rate limiting on the traffic. `rps` on the other hand is a `default` and as such a `default` at the `HTTPRoute` level would take precedence in the hierarchy:

```yaml
kind: FakeRLPolicy
spec:
  override:
    enabled: false
  default:
    rps: 500
  targetRef:
    kind: HTTPRoute
    name: foo-route
```

Attaching the `FakeRLPolicy` above to our `HTTPRoute` would not disable rate limiting (as the `override` tries to set `enabled` to `false`, as this would still be overruled by the `my-gw` attached policy), but the `rps` would be increased to `500` as the `default` of the `HTTPRoute` overrules the ones from the `Gateway` (up in the hierarchy). It is important to not that there is no way for the policy to discriminate the traffic it applies to at this stage. All traffic, as described above, would be affected by the merge of the two policies:

```yaml
enabled: true
rps: 500
```

>**Note:** The status block of a policy should reflect the merged state of a policy as per other policies at that level of the hierarchy.

Any other `HTTPRoute` attached to the `my-gw` would still use the 42 rps limit. If we now delete the `FakeRLPolicy` attached to `my-gw`, the resulting policy would look like this:

```yaml
enabled: true
rps: 500
```

Which would effectively disable all rate limiting traffic to the `foo-route` and ultimately the `foo` `Backend`.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

So there are a couple of things to consider here:

- Being able to clearly identify and merge `HTTPRouteMatch`es from the `HTTPRoute` and apply accordingly to "lower down" resources
- What about two `Gateway`s having different policies attached to them? (see [Unresolved questions](#unresolved-questions) below)

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- How error would be reported to the users.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

- The use cases this approach doesn't address… 
- Proliferation of `HTTPRoute` rather than finer grained control at the policy level or at the `HTTPRouteRule` level

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- [`HTTPRouteFilter`](https://gateway-api.sigs.k8s.io/references/spec/#gateway.networking.k8s.io/v1beta1.HTTPRouteFilter) would be an alternative and provide finer grained control over how a policy applies, i.e. at the `HTTPRouteRule` level, or even on a per `HTTPBackendRef` within an `HTTPRouteRule`. But today there is no way (that I know of) for us to add a `HTTPRouteFilterType` to the list of supported `HTTPRouteFilter` by a Gateway API implementor. 

# Prior art
[prior-art]: #prior-art

I don't know of prior art per se, _but_ the envoy gateway project, if only for rate limiting, seems to [avoid policy attachment](https://github.com/envoyproxy/gateway/issues/670#issuecomment-1299114982), but they are an implementor and as such can extend the [`HTTPRouteFilter`](https://gateway-api.sigs.k8s.io/references/spec/#gateway.networking.k8s.io/v1beta1.HTTPRouteFilter) types.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- How do we deal with multiple `Gateway` having different set of policies attached to them (semantically or atomically?)?
  - Can we inject the proper config in each of them? Only if all is applicable? What if they are not? What when that changes over time?
  - Can these then be backed by a single e.g. `Limitador`? (in that case I think yes, as we could inject the "source" gw as a variable and add "synthetic" conditions accordingly)
- What use cases are we making "overly" complex, because of the `HTTPRoute` redundancy?

# Future possibilities
[future-possibilities]: #future-possibilities

 - Selectors for `HTTPRoute` on which a policy should be "pushed down"
 - Constraint language / bound definitions (always evaluatable to an actual value) for "lower" policies to operate in: e.g. `minValue: 30` and `maxValue: 100` as an `override`, with a `default` of `value: 50` or just `maxValue: 100`, so that `value` defaults to `100`.
