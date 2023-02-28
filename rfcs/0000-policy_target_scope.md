# RFC A single Policy scoped to a HTTPRouteRule

- Feature Name: Policy targeting HTTPRoute and HTTPRouteRules
- Start Date: 2022-11-09
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/issues/0000)

# Summary
[summary]: #summary

Currently both `RateLimitPolicy` and `AuthPolicy` allow for a policy to only affect a _subset_ of the
traffic on a targeted `HTTPRoute`. This proposal aims at simplifying the scope of a policy to target
_all_ the traffic of a route or the traffic specified by a specific `HTTPRouteRule`'s `HTTPRouteMatch`.

# Motivation
[motivation]: #motivation

As per the specification, a given resource will always need to support multiple policies targeting it.
Even if we only support a single e.g. `RateLimitPolicy` per resource, there will alway be a possibility
for another policy being defined else where in that resource's hierarchy: e.g. _policy1_ targeting the
`Gateway`, while _policy2_ targeting the `HTTPRoute` being attached to the `Gateway`. Until there is an
`HTTPRoute` tho, _policy1_ will have no semantic, as there is no traffic routed, so no policy for
Kuadrant to enforce, as it operates on the data plane. When an `HTTPRoute` is attached to the `Gateway`
is the point when that _policy1_ becomes applicable. If that `HTTPRoute` also defines a `RateLimitPolicy`,
only one will be enforced, according to the `default` & `override` rules of the specification. 

The idea is to keep things simple (if only for now), so that when traffic is routed through an
`HTTPRoute` only one policy of a `kind` becomes applicable and applies to the whole traffic of that route
(or a subset thereof, see below).

The user might want to qualify traffic to only a specific `HTTPRouteRule` of a given `HTTPRoute`. That would be
permitted for as long as the `Policy` reflects the exact same set of `HTTPRouteMatch` as the `HTTPRouteRule` defines.
If two policies target the same `HTTPRouteRule`, one would take precedence over the other according to
the Gateway API's merging rules.

With these rules, Kuadrant can "easily" identify what policies apply to an `HTTPRoute`, possibly
reflecting the `status` of the policy on a per `HTTPRouteRule` level, for when one get "shadowed".

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Traffic to your service is ultimately routed when an `HTTPRoute` binds at least one  `Gateway`, thru
the
[`parentRefs`](https://gateway-api.sigs.k8s.io/references/spec/#gateway.networking.k8s.io/v1beta1.ParentReference),
and wires traffic to the
[`backendRef`](https://gateway-api.sigs.k8s.io/references/spec/#gateway.networking.k8s.io/v1beta1.HTTPBackendRef),
i.e. your service, according to the
[`HTTPRouteMatch`](https://gateway-api.sigs.k8s.io/references/spec/#gateway.networking.k8s.io/v1beta1.HTTPRouteMatch)
of a
[`HTTPRouteRule`](https://gateway-api.sigs.k8s.io/references/spec/#gateway.networking.k8s.io/v1beta1.HTTPRouteRule)
define on that `HTTPRoute`. All the traffic matched by _any_ `HTTPRouteRule` of a given `HTTPRoute`
will be affected by which ever `RateLimitPolicy` or `AuthPolicy` you attach to it, unless the _Policy_ references a
specific `HTTPRouteRule` by duplicating all of its `HTTPRouteMatch`es.

Any Kuadrant _Policy_ will apply to the traffic that is ultimately routed through because of a
`HTTPRouteRule`, which binds incoming traffic to a backend service and as such defines a route. Each
_Policy_ has to be a complete specification to be accepted by Kuadrant. For any given route, Kuadrant
resolves the _Policy_ to enforce based on the `default` & `override` specification of the individual
_Policies_. 

```yaml
kind: PaintPolicy
spec:
  override:
    color: blue
  targetRef:
    kind: Gateway
    name: my-gw
```

This `PaintPolicy` defines `color` as `blue`. Since there is no other `PaintPolicy` attached to the
`my-gw` `Gateway`, its namespace and so forth, see [the specification for the
hierarchy](https://gateway-api.sigs.k8s.io/references/policy-attachment/#hierarchy), the `override` value
for `color` precedes and as such `blue` will be enforced for _any_ `HTTPRoute` that ends up being wired
to this `Gateway`.

That policy tho has no traffic to police yet. Unless an `HTTPRoute` binds that `Gateway` to a
`Backend`, no traffic can exist and as such can't be applied to by the policy. As such Kuadrant will
_not_ create any further of its custom resources (`CR`) until that happens.

Let's change our `PaintPolicy` to use a `default` instead and show how this can then now the behavior can
be changed by other another `PaintPolicy` attached further down the [hierarchy of network
resources](https://gateway-api.sigs.k8s.io/references/policy-attachment/#hierarchy):

```yaml
kind: PaintPolicy
spec:
  default:
    color: blue
  targetRef:
    kind: Gateway
    name: my-gw
```

Say, we now have a `Backend` (e.g. `foo`) wired to the `my-gw` using some `HTTPRoute` (e.g.
`foo-route`), none of which have yet any `PaintPolicy` attached. Yet, since the `Gateway` used as a
policy defined, Kuadrant will create a `PaintLimit` `CR` and eventually configure its service to apply
the policy for traffic routed to that `foo` `Backend`. The `PaintPolicy`
to eventually apply will be resolved using the network resource hierarchy. The
traffic "painted" will be the one matching the union of all the  `HTTPRouteMatch` of each
`HTTPRouteRule` as defined in that `HTTPRoute`. 

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

With `foo-route` declaring a single hostname (`example.com`) and a single `HTTPRouteMatch` (anything
starting with `/foo`), the policy will apply to the same traffic: `example.com/foo*`, so that `foo` 's
traffic to port `8080` will be painted blue, as specified by the `default` section of our `PaintPolicy`.

In order to change that color, we'd need to define another `PaintPolicy` that defines a `default` and
targets our `foo-route`:

```yaml
kind: PaintPolicy
spec:
  default:
    color: green
  targetRef:
    kind: HTTPRoute
    name: foo-route
```

Attaching the `PaintPolicy` above to our `HTTPRoute` will _shadow_ the `PaintPolicy` attached to the
`Gateway`. It is important to note that there is no way for the policy to discriminate the traffic it
applies to at this stage. All traffic to `foo-route` is now painted `green`, while any further routes
attached to `my-gw` would still be painted `blue`.

>**Note:** The status block of a policy should reflect whether it is shadowing or being shadowed by other
>policies at that level of the hierarchy.

Which would effectively disable all rate limiting traffic to the `foo-route` and ultimately the `foo`
`Backend`.

Using another example, let's consider a single `HTTPRoute` with two `HTTPRouteRule`s:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: bar-route
spec:
  parentRefs:
  - name: my-gw
  hostnames:
  - "bar.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /login
    backendRefs:
    - name: bar-svc
      port: 8081
  - matches:
    - headers:
      - type: Exact
        name: env
        value: production
    backendRefs:
    - name: bar-svc
      port: 8080
```

It is important to consider that only _one_ rule will ever be match for any given requests. It makes for
a simpler model to have only one _Policy_ to also only ever being matched, aligning the both behaviors.
To apply a policy on the first rule only, the user needs to duplicate that match on the policy targeting
`bar-route`:

```yaml
kind: PaintPolicy
spec:
  default:
    color: red
  targetRef:
    kind: HTTPRoute
    name: foo-route
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /login
```

Hereby we are coloring the traffic to our `/login` on port `8081` to be `red`. That coloring _will not_
apply to other matches. So that traffic to `bar-svc` on port `8080` would be colored `green` as per the
`PaintPolicy` attached to `my-gw`.

The resolving of _Policies_ to enforce occurs at the `HTTPRoute` or within individual `HTTPRouteRule`
within a route. The merging strategy will resolve the individual `HTTPRouteRule` all the policies boil
down to, stack them according to the `default` and `override` and appy the most specific one according to
the Gateway API Policy Attachment hierarchy. Targeting a complete `HTTPRoute` is the equivalent of
targeting each indidividual `HTTPRouteRule`s that composes the route.

## What actual use-cases does this address?

 - An admin restricts all traffic using a `default` at the gateway to force application developers to
   "think" about how to authorize access to their services and have them come up with sensible `default`
   on their `HTTPRoute`s
 - An application developer protects its service by some rate limit, while an cluster admin could come
   and use an `override` to mitigate a DDoS.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

So there are a couple of things to consider here:

- status reporting on "not yet" existing `HTTPRouteRule`, e.g. a Policy declares a set of `matches` that
  don't (yet?) exist. The policy must _not_ be enforced... yet. But could eventually. This needs proper
  reporting. 

# Drawbacks
[drawbacks]: #drawbacks

- The use cases this approach doesn't address… 
- Not very DRY... the need to keep `HTTPRouteRule`s in sync between the Policy and the `HTTPRoute` is
  suboptimal.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- [`HTTPRouteFilter`](https://gateway-api.sigs.k8s.io/references/spec/#gateway.networking.k8s.io/v1beta1.HTTPRouteFilter)
  would be an alternative and provide finer grained control over how a policy applies, i.e. at the
  `HTTPRouteRule` level, or even on a per `HTTPBackendRef` within an `HTTPRouteRule`. But today there is
  no way (that I know of) for us to add a `HTTPRouteFilterType` to the list of supported
  `HTTPRouteFilter` by a Gateway API implementor. 

# Prior art
[prior-art]: #prior-art

I don't know of prior art per se, _but_ the envoy gateway project, if only for rate limiting, seems to
[avoid policy attachment](https://github.com/envoyproxy/gateway/issues/670#issuecomment-1299114982), but
they are an implementor and as such can extend the
[`HTTPRouteFilter`](https://gateway-api.sigs.k8s.io/references/spec/#gateway.networking.k8s.io/v1beta1.HTTPRouteFilter)
types. This proposed approach if actually trying to enable achieving the same control `HTTPRouteFilter`
gives, while using _Policy Attachement_ and targeting a specific `HTTPRouteRule` within a `HTTPRoute`.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What use cases are we making "overly" complex, because of the duplication of `HTTPRoute`'s
  `HTTPRouteMatch` ?
- Is it fine for the `matches` section to be at the same level as `default`, `override` and `targetRef`?
  That bit is unclear to me [from the
  specification](https://gateway-api.sigs.k8s.io/references/policy-attachment/#policy-boilerplate).

# Future possibilities
[future-possibilities]: #future-possibilities

 - Selectors for `HTTPRoute` on which a policy should be "pushed down"
 - Introduce partial _policies_ and merging of them, rather than having them being complete specs.
