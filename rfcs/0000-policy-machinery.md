# Policy Machinery for reconciliation

- Feature Name: `policy_machinery`
- Start Date: 2024-07-03
- RFC PR: [Kuadrant/architecture#95](https://github.com/Kuadrant/architecture/pull/95)
- Issue tracking: [Kuadrant/architecture#29](https://github.com/Kuadrant/architecture/issues/29)

# Summary
[summary]: #summary

Explain how Kuadrant's [Policy Machinery](https://github.com/Kuadrant/policy-machinery) can be used for reconciliation.

# Motivation
[motivation]: #motivation

The _Policy Machinery_ project ([repo](https://github.com/Kuadrant/policy-machinery), [pkg.go](https://pkg.go.dev/github.com/kuadrant/policy-machinery)) offers a set of types and functions for implementing Gateway API policies – i.e.
- highly flexible representation of topologies of targetable resources;
- calculating effective policies based on custom or default merge strategies;
- tooling to watch and reconcile resources based on cluster events.

These can be used for tailoring implemention of Kuadrant policies and Kuadrant instances. See [example](https://github.com/Kuadrant/policy-machinery/tree/main/examples/kuadrant) provided.

Leveraging the Policy Machinery can be key to:
  1. Improve flow control of concurrent reconciliation events
  2. Simplification of the calculation of effective policies respectively to the topological routing path of the requests
  3. Correct implementation of Defaults & Overrides' `merge` strategy ([RFC 0009](https://docs.kuadrant.io/0.8.0/architecture/rfcs/0009-defaults-and-overrides/))
  4. Supporting multiple policies targeting a same resource
  5. Supporting targeting sections of a resource (e.g. Gateway listeners, HTTPRouteRules)
  6. Supporting multiple targetRefs in a policy
  7. Extending policies to target other kinds of resources (e.g. GatewayClass, Service, Namespace) -– future

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Although essentially an implementation detail of the Kuadrant Operator, levering the Policy Machinery may introduce the following user-perceived features:
- Acknowledgement of the topological routing path of the request respectively to applicable _effective policies_
- New form of targeting sections of a resource with a policy
- Elevated meaning of uniquely identifiable concepts across policy resources (e.g. named policy rules)
- Possibility of multiple policies of a kind targeting a same resource
- (Window of opportunity for) introducing plural targetRefs

### User-acknowledgeable reference of the topological routing path of the request

One who specifies Kuadrant policies targeting resources at any levels of the hierarchy _Gateway → Listener → HTTPRoute → HTTPRouteRule_[^1] shall expect the effect of such policies to be reported always respectively to the lowest level of the hierarchy that the kind of policy allows targeting. E.g.:
- A DNSPolicy that allows targeting a Gateway hypothetically with or without specifying a `sectionName` actually targets gateway listeners; the user wants to reason about the state of DNSPolicies regarding their effect on each listeners specified in the Gateway. In a context with 2+ DNSPolicies, simultaneously targeting both a Gateway and specific listeners, DNS records for some listener hostnames may have been reconciled according to the specification from one policy or another, occasionally no policy at all.
- A RateLimitPolicy that allows targeting a Gateway, a HTTPRoute or specific HTTPRouteRule actually targets (directly or indirectly) HTTPRouteRule objects; the user wants to reason about the state of all RateLimitPolicies with respect to each HTTPRouteRule, where some HTTPRouteRules may be protected by one RateLimitPolicy, a combination of multiple RateLimitPolicies, or occasionally no RateLimitPolicy at all.

[^1]: Gateway listeners and HTTPRouteRules can be targeted by specifying in a policy their main Gateway and HTTPRoute objects as targets, in combination with either a `sectionName` (supported in the `targetRef` field of the policy) or via `routeSelectors` (in the policy spec proper), respectively. Not all kinds of policies support targeting all 4 kinds of targetables of the _Gateway → Listener → HTTPRoute → HTTPRouteRule_ hierarchy; some kinds of policies may support targeting only a few of those.

In the specific case of policy kinds that allow targeting HTTPRouteRules, due to complex network topologies supported by Gateway API, including in particular HTTPRoutes with multiple Gateway parents, a same HTTPRouteRule may or may not be affected by a policy, depending on which routing path in the network topology a request flows. Therefore, ultimately _users will reason about policies and effective policies in terms of the paths between at least gateways and the lowest levels targeted by the policies that a request can flow._ Possibly, in terms of all possible paths between Gateways and Services.

Kuadrant shall provide users with such visibility. Leveraging Policy Machinery is an implementation detail that nonetheless makes achieving this goal easier.

### User-facing changes to the policy APIs

#### Targeting sections of network resource

Leveraging Policy Machinery may also motivate some user-facing changes. In particular, replacing AuthPolicy's and RateLimitPolicy's `routeSelectors` for a `targetRef` with optional `sectionName` (made possible since [kubernetes-sigs/gateway-api#2895](https://github.com/kubernetes-sigs/gateway-api/pull/2985).)

This change would cause a policy of AuthPolicy or RateLimitPolicy kind to always be attached to its targets _entirely_, i.e. without having rules that attach to some sections and other rules to other sections or no section at all. This differs from current situation where a policy of those kinds can be attached to a HTTPRoute and some of the policy's rules more specifically attached to individual HTTPRouteRules only, including with multiple policy rules attached to different HTTPRouteRules of the same targeted HTTPRoute. Instead, attaching a policy must be a cohesive, unambiguous operation, that occasionally requires users to specify more fine-grained policy objects to be attached only to sections of a resource.

#### Identity of concepts across multiple policy objects

In some cases, splitting policy objects for the purpose of targeting sections of a network resource, without breaking the semantics of having a single set of policy objects cohesively defined, also implies that definitions about a same entity or concept within a policy (e.g. a limit definition), repeated at multiple policy objects, may have a way to represent to refer to the same thing (e.g. same set of counters).

This is the case, for example, of limit definitions in a RateLimitPolicy as well as cache configs in an AuthPolicy. To avoid creating multiple rate-limit counter namespaces (analogously, multiple authorization rule cache entries) for definitions that are effectively about the same entity, despite specified at multiple policy objects, the APIs must provide a way for users to convey one of the other intent: definitions refer to the same thing versus definitions refer to different things.[^2]

[^2]: In the context of rate-limit, this problem is also referred to as the problem of the _identity of a limit_.

#### Other user-facing changes

The following possible (non-required) other user-facing changes can be enabled leveraging Policy Machinery, without marginal implementation cost:

1. Plural `targetRefs`.
2. Multiple policies of a kind targeting a same network resource/resource section ("horizontal Defaults & Overrides.")
3. Aesthetical difference between [Direct](https://gateway-api.sigs.k8s.io/geps/gep-2648/) versus [Inherited](https://gateway-api.sigs.k8s.io/geps/gep-2649/) policies, implied by the merge strategies implemented by each kind of policy, rather than a necessary distinction.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Usage of the [Policy Machinery](https://github.com/Kuadrant/policy-machinery) consists of importing two packages:
- [`machinery`](https://github.com/Kuadrant/policy-machinery/tree/main/machinery): provides the types and abstractions to build Gateway API topologies of targetable network resources, policies and adjacent objects;
- [`controller`](https://github.com/Kuadrant/policy-machinery/tree/main/controller): offers tools for implementing topology-based custom controllers of reconciliation logic.

From that on, the following steps drive leveraging the Policy Machinery in the Kuadrant Operator:

1. Implement the [`machinery.Policy`](https://github.com/Kuadrant/policy-machinery/blob/e963418c9566a5e61acdd766c6109d3f0c147222/machinery/types.go#L47) interface for all kinds of policies.

    > [Example provided](https://github.com/Kuadrant/policy-machinery/tree/main/examples/kuadrant/apis) for the DNSPolicy, TLSPolicy, AuthPolicy and RateLimitPolicy kinds.

2. Define wrappers that implement the [`machinery.Object`](https://github.com/Kuadrant/policy-machinery/blob/e963418c9566a5e61acdd766c6109d3f0c147222/machinery/types.go#L13) interface for any kind of adjacent object whose unique identifier as a node in the topology graph cannot be based on the default [`controller.RuntimeObject`](https://github.com/Kuadrant/policy-machinery/blob/e963418c9566a5e61acdd766c6109d3f0c147222/controller/object.go#L17) type provided (if any.)

3. Implement the linking functions for all kinds of adjacent objects and corresponding parents and children in the topology graph, including types such as `Kuadrant`, Istio's `WasmPlugin`, `ConfigMap`, etc.

    > The `Kuadrant` custom resources shall be the roots of a directed acyclic graph (DAG) from which an entired topology of targetable network resources, adjacent objects and policies are connected.

4. Start a [`controller.Controller`](https://github.com/Kuadrant/policy-machinery/blob/e963418c9566a5e61acdd766c6109d3f0c147222/controller/controller.go#L130) that:

    1. Watches for all kinds of objects to be represented in the topology.
    2. Triggers a [`controller.Workflow`](https://github.com/Kuadrant/policy-machinery/blob/e963418c9566a5e61acdd766c6109d3f0c147222/controller/workflow.go#L12) on events related to any of the watched resources.

5. At every reconciliation event[^3]:
    1. Reconcile the internal objects for setting up the environment for a Kuadrant instance (deployments, gateway controller configs, etc).

    2. For each kind of policy and applicable path in the topology graph relevant for the policy kind:

       1. Compute an effective policy and give it a unique identifier.
       2. Perform (or delegate) the policy-specific configuration (DNS, TLS, Auth, RL) of the effective policy with the policy decision/enforcement point[^4].
       3. For policy kinds enforced at request time (data-plane policies), configure the gateway to call the policy decision point (PDP) on requests that match the attributes of the path[^5], passing in the payload to the PDP the identifier of the effective policy.
       4. Update the status stanzas of all targetables in the topology whose paths were configured for an effective policy (or lack of such), with a map that allows users to inspect, for a given path, what effective policy (if any) will be enforced.
           > **Note:** If unsuitable for the status stanza of the object, the details of the effective policies may require additional tooling to be inspected by the users and the mapping must be to the unique identifier of the effective policy.

    3. Store a DOT representation of the topology graph in a ConfigMap.

[^3]: Specific steps can be filter by type of event.

[^4]: While the DNS operator as well as the configuration performed by the Kuadrant Operator for a TLSPolicy are closer to the _enforcement_ of the specifications in the DNS and TLS policy objects, Authorino and Limitador are policy _decision_ points (PDP) rather. Indistinctively, control-plane operations that configure a service based on the specification of a policy, as well as the data-plane protection services that perform at request-time along with the gateways are all part of the policy enforcement.

[^5]: The attributes of a path in the topology from a Gateway to a HTTPRouteRule object typically include a hostname and the set of HTTPRouteMatches specified in the HTTPRouteRule.

# Drawbacks
[drawbacks]: #drawbacks

Part of the work consists on refactoring, without value perceived by the user.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

#### Annotations

Use of annnotations to track back references from targeted objects to policies. This approach has been slowly deprecated to favour the use of an in-memory directed acyclic graph (DAG) representing the relationship between network objects and policies[^6].

#### Bottom-up reconciliation

Bottom-up reconciliation by default, focusing on the policy resources first. This approach has been slowly refactored to using mappers ([event handlers](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/handler)) that often multiply a lower-level event into multiple top-down ones, occasionally with the occurrence of repetitive events.

#### DAG 1.0

Preliminary version of the topology DAG[^6] that:
 1. is bootstrapped at every reconciliation event (though leveraging [k8s.io/apimachinery](https://pkg.go.dev/k8s.io/apimachinery)'s and [sigs.k8s.io/controller-runtime](https://pkg.go.dev/sigs.k8s.io/controller-runtime)'s caches);
 2. does not include all kinds of targetables – missing object sections in particular;
 3. does not include internal configuration objects;
 4. was designed for one single kind of policy in each instance of the topology.

#### Effective policy-less reconciliation

Configuration of internal resources for implementing effective policy behavior:
  1. tailored for each specific kind of policy, without leveraging generic and resusable merge strategy functions;
  2. not organically integrated with the topology DAG;
  3. decoupled from the rather user-acknowledgeable reference of the topological routing path of the request.

[^6]: See [kuadrant-operator#530](https://github.com/Kuadrant/kuadrant-operator/issues/530).

# Prior art
[prior-art]: #prior-art

#### Envoy Gateway state-of-the-world reconciliation

Envoy Gateway has implemented a custom controller for Gateway API and provider-specific resources with the following characteristics similar to the Policy Machinery `controller` package's approach:
   1. Based on [controller-runtime](https://pkg.go.dev/sigs.k8s.io/controller-runtime)
   2. Single `Reconcile` function that:
      1. lists all watched resources from the cluster at every reconciliation event;
      2. rebuilds and updates a long-living [watchable](https://github.com/telepresenceio/watchable) map of all the resources;
      3. trigger reconciliation logic subscribed to changes to the map – `goroutines` decoupled from controller-runtime.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- The [fanout problem](https://gateway-api.sigs.k8s.io/geps/gep-713/#fanout-status-update-problems), especially of status reconciliation.
- ~~How to avoid changes performed by the reconciliation function to loop back in the form of multiple other (no-op) reconciliation events.~~ ⇨ issue state-modifying actions against the API server directly and safe-guard against create/update/delete events for states already reflected in the topology
- ~~How to compact multiple equifinal reconciliation events waiting in the queue into a single one, thus avoiding unnecessary loops.~~ ⇨ rely on controller-runtime event coalescing
- ~~What to do in case of reconciliation failures without retry~~ ⇨ always move the system to a final state, with proper status updating, and wait until state-of-the-world reconciliation kicks in again on the next event

# Future possibilities
[future-possibilities]: #future-possibilities

- Extending policies to target other kinds of resources (e.g. GatewayClass, Service, Namespace.)
