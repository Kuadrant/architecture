# Policy Machinery for reconciliation

- Feature Name: `policy_machinery`
- Start Date: 2024-07-03
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [Kuadrant/architecture#29](https://github.com/Kuadrant/architecture/issues/29)

# Summary
[summary]: #summary

Explain how Kuadrant's [Policy Machinery](https://github.com/Kuadrant/policy-machinery) can be used for reconciliation.

# Motivation
[motivation]: #motivation

**Policy Machinery** ([repo](https://github.com/Kuadrant/policy-machinery), [pkg.go](https://pkg.go.dev/github.com/kuadrant/policy-machinery)) offers a set of interfaces and structs for implementing Gateway API policies – calculating effective policies based on custom or default merge strategies, on top of highly flexible topologies of targetable resources, that can be tailored specifically for the context of reconciling resources that implement Kuadrant's kinds of policies. (Example provided [here](https://github.com/Kuadrant/policy-machinery/blob/main/examples/kuadrant/integration_test.go).)

Using such tool can be key to:
- correctly implementing Defaults & Overrides ([RFC 0009](https://docs.kuadrant.io/0.8.0/architecture/rfcs/0009-defaults-and-overrides/)) beyond the level of atomic merges;
- supporting multiple policies targeting a same resource;
- supporting targeting sections of a resource (e.g. Gateway listeners, HTTPRouteRules);
- supporting multiple targetRefs in a policy;
- futurely extending policies to target other kinds of resources (e.g. GatewayClass, Service, Namespace.)

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### User-acknowledgeable reference of the topological routing path of the request

One who specifies Kuadrant policies targeting resources at any levels of the hierarchy _Gateway → Listener → HTTPRoute → HTTPRouteRule_[^1] shall expect the effect of such policies to be reported always respectively to the lowest level of the hierarchy that the kind of policy allows targeting. E.g.:
- A DNSPolicy that allows targeting a Gateway hypothetically with or without specifying a `sectionName` actually targets gateway listeners; the user wants to reason about the state of DNSPolicies regarding their effect on each listeners specified in the Gateway. In a context with 2+ DNSPolicies, simultaneously targeting both a Gateway and specific listeners, DNS records for some listener hostnames may have been reconciled according to the specification from one policy or another, ocasionally no policy at all.
- A RateLimitPolicy that allows targeting a Gateway, a HTTPRoute or specific HTTPRouteRule actually targets (directly or indirectly) HTTPRouteRule objects; the user wants to reason about the state of all RateLimitPolicies with respect to each HTTPRouteRule, where some HTTPRouteRules may be protected by one RateLimitPolicy, a combination of multiple RateLimitPolicies, or ocasionally no RateLimitPolicy at all.

[^1]: Gateway listeners and HTTPRouteRules can be targeted by specifying in a policy their main Gateway and HTTPRoute objects as targets, in combination with either a `sectionName` (supported in the `targetRef` field of the policy) or via `routeSelectors` (in the policy spec proper), respectively. Not all kinds of policies support targeting all 4 kinds of targetables of the _Gateway → Listener → HTTPRoute → HTTPRouteRule_ hierarchy; some kinds of policies may support targeting only a few of those.

In the specific case of policy kinds that allow targeting HTTPRouteRules, due to complex network topologies supported by Gateway API, including in particular HTTPRoutes with multiple Gateway parents, a same HTTPRouteRule may or may not be affected by a policy, depending on which routing path in the network topology a request flows. Therefore, ultimately _users will reason about policies and effective policies in terms of the paths between at least gateways and the lowest levels targeted by the policies that a request can flow._ Possibly, in terms of all possible paths between Gateways and Services.

Kuadrant shall provide users with such visibility. Leveraging Policy Machinery is an implementation detail that nonetheless makes achieving this goal easier.

### User-facing changes to the policy APIs

#### Targeting sections of network resource

Leveraging Policy Machinery may also motivate some user-facing changes. In particular, replacing AuthPolicy's and RateLimitPolicy's `routeSelectors` for a `targetRef` with optional `sectionName` (requires [kubernetes-sigs/gateway-api#2895](https://github.com/kubernetes-sigs/gateway-api/pull/2985)) or moving the `routeSelectors` field to the top-level of the policy only (along or as part of `targetRef`, until [kubernetes-sigs/gateway-api#2895](https://github.com/kubernetes-sigs/gateway-api/pull/2985) is merged.)

This change would cause a policy of AuthPolicy or RateLimitPolicy kind to always be attached to its targets _entirely_, i.e. without having rules that attach to some sections and other rules to other sections or no section at all. This differs from current situation where a policy of those kinds can be attached to a HTTPRoute and some of the policy's rules more specifically attached to individual HTTPRouteRules only, including with multiple policy rules attached to different HTTPRouteRules of the same targeted HTTPRoute. Instead, attaching a policy must be a cohesive, unambiguous operation, that ocasionally requires users to specify more fine-grained policy objects to be attached only to sections of a resource.

#### Identity of concepts across multiple policy objects

In some cases, splitting policy objects for the purpose of targeting sections of a network resource, without breaking the semantics of a having a single set of policy objects cohesively defined, also implies that definitions about a same entity or concept within a policy (e.g. a limit definition), repeated at multiple policy objects, may have a way to represent to refer to the same thing (e.g. same set of counters).

This is the case, for example, of limit definitions in a RateLimitPolicy as well as cache configs in an AuthPolicy. To avoid creating multiple rate-limit counter namespaces (analogously, multiple authorization rule cache entries) for definitions that are effectively about the same entity, despite specified at multiple policy objects, the APIs must provide a way for users to convey one of the other intent: definitions refer to the same thing versus definitions refer to different things.[^2]

[^2]: In the context of rate-limit, this problem is also refered to as the problem of the _identity of a limit_.

#### Other user-facing changes

The following possible (non-required) other user-facing changes can be enabled leveraging Policy Machinery, without marginal implementation cost:

1. Plural `targetRefs`.
2. Multiple policies of a kind targeting a same network resource/resource section ("horizontal Defaults & Overrides.")
3. Aesthetical difference between [Direct](https://gateway-api.sigs.k8s.io/geps/gep-2648/) versus [Inherited](https://gateway-api.sigs.k8s.io/geps/gep-2649/) policies, implied by the merge strategies implemented by each kind of policy, rather than a necessary distinction.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

[WIP]

1. Implement the [`Policy`](https://github.com/Kuadrant/policy-machinery/blob/66032b181c3f7a100fe9e9ecd57f7847af3433b0/machinery/types.go#L43) interface for all kinds of policies.

    > **Note:** Until [kubernetes-sigs/gateway-api#2895](https://github.com/kubernetes-sigs/gateway-api/pull/2985) is merged, AuthPolicy and RateLimitPolicy may need to use the `routeSelectors` in the implementation of `GetTargetRefs`. A wrapper for the HTTPRouteRule as a [`Targetable`](https://github.com/Kuadrant/policy-machinery/blob/main/machinery/types.go#L31) may have to be defined with its `GetURL` method matching the identifiers returned by the policy's `GetTargetRefs`.

2. Define wrappers that implement the [`Object`](https://github.com/Kuadrant/policy-machinery/blob/main/machinery/types.go#L13) interface for all kinds of internal objects relevant to the configuration of policy enforcement, for whose instances we want to leverage the topology as an intermediary structure to control the reconciliation.

3. At runtime, build a long living instance of a topology with all preexisting Gateway API objects (Gateways, Listeners, HTTPRoutes and HTTPRouteRules), policies (all kinds), and controlled internal configuration objects (e.g. WasmPlugins).

4. At every reconciliation event:
    1. Make a copy of the topology and update it to its new state based on the event that triggered reconciliation.[^3]
    2. For each kind of policy, get each path in the topology between each Gateway and each object of the lowest kind that the policy can target.

        > **Note:** The kinds of policies in this step can be filtered to the kinds that are knwon to be affected by the type of event that triggered the reconciliation.

    3. For each path, compute an effective policy and give it a unique identifier.
    4. Perform (or delegate) the policy-specific configuration (DNS, TLS, Auth, RL) for the decision/enforcement[^4] of the effective policy.
    5. For policy kinds enforced at request time (data-plane policies), configure the gateway to call the policy decision point (PDP) for requests that match the attributes of the path[^5], passing in the payload to the PDP the identifier of the effective policy.
    6. Update the status stanzas of all targetables in the topology whose paths were configured for an effective policy (or lack of such), with a map that allows users to inspect, for a given path, what effective policy (if any) will be enforced.
        > **Note:** If unsuitable for the status stanza of the object, the details of the effective policies may require additional tooling to be inspected by the users and the mapping must be to the unique identifier of the effective policy.

[^3]: At a much higher cost, this steps could be replaced by rebuilding the entire topology from scratch by reading again all resources from the cluster.

[^4]: While the DNS operator as well as the configuration performed by the Kuadrant Operator for a TLSPolicy are closer to the _enforcement_ of the specifications in the DNS and TLS policy objects, Authorino and Limitador are policy _decision_ points (PDP) rather. Indistinctively, control-plane operations that configure a service based on the specification of a policy, as well as the data-plane protection services that perform at request-time along with the gateways are all part of the policy enforcement.

[^5]: The attributes of a path in the topology from a Gateway to a HTTPRouteRule object typically include a hostname and the set of HTTPRouteMatches specified in the HTTPRouteRule.

All steps in 4 that involve changing the state of the cluster (create, update, delete objects), whose object kind exists in the topology, must be performed against the copy of the topology itself first. Then, a diff between known state of the topology and the updated copy shall be computed and all required operations to update the cluster sent to the API server.

For auth, as of today, step 4.v requires giving up on Istio's [AuthorizationPolicy](https://istio.io/latest/docs/reference/config/security/authorization-policy/#AuthorizationPolicy) and Envoy Gateway's [SecurityPolicy](https://gateway.envoyproxy.io/v0.6.0/design/security-policy/) for controlling the call to the external authorization service, as both these APIs won't support supplying a data such as the unique identifier of the effective policy to enforce. Alternatively, the [wasm-shim](https://github.com/Kuadrant/wasm-shim) can be modified to send the external authorization request.

# Drawbacks
[drawbacks]: #drawbacks

Part of the work consists on refactoring, without value percieved by the user.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

[TODO]

# Prior art
[prior-art]: #prior-art

[WIP]

- Use of annnotations to track back references from targeted objects to policies – slowly deprecated to favour the use of an in-memory directed acyclic graph (DAG) representing the relationship between network objects and policies[^6].
- Bottom-up reconciliation by default – slowly refactored to using mappers ([event handlers](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/handler)) that often multiply a lower-level event into multiple top-down ones, ocasionally with lots of repetition.
- Rudimentary version of the topology directed acyclic graph (DAG)[^6] that:
    1. is bootstrapped at every reconciliation event (though leveraging [k8s.io/apimachinery](https://pkg.go.dev/k8s.io/apimachinery)'s and [sigs.k8s.io/controller-runtime](https://pkg.go.dev/sigs.k8s.io/controller-runtime)'s caches);
    2. does not include all kinds of targetables – missing object sections in particular;
    3. does not include internal configuration objects;
    4. was designed for one single kind of policy in each instance of the topology.
- Computation of effective policies:
    1. tailored for each specific kind of policy, without leveraging much of more generic, resusable merge strategy functions;
    2. not organically integrated with the topology DAG;
    3. decoupled from the rather user-acknowledgeable reference of the topological routing path of the request.

[^6]: See [kuadrant-operator#530](https://github.com/Kuadrant/kuadrant-operator/issues/530).

# Unresolved questions
[unresolved-questions]: #unresolved-questions

[WIP]

- The [fanout problem](https://gateway-api.sigs.k8s.io/geps/gep-713/#fanout-status-update-problems), especially of status reconciliation.
- How to avoid changes performed by the reconciliation function to loop back in the form of multiple other (no-op) reconciliation events.
- How to compact multiple equifinal reconciliation events waiting in the queue into a single one, thus avoiding unnecessary loops.

# Future possibilities
[future-possibilities]: #future-possibilities

- Extending policies to target other kinds of resources (e.g. GatewayClass, Service, Namespace.)
