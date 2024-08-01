# Gateway API Metrics Exporter

- Feature Name: gateway-api-metrics-exporter
- Start Date: 2024-07-26
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [kuadrant-operator/issues/675](https://github.com/Kuadrant/kuadrant-operator/issues/675)

# Summary
[summary]: #summary

A new sub-project for a [prometheus exporter](https://prometheus.io/docs/instrumenting/writing_exporters/) that exports metrics about the state of [Gateway API resources](https://gateway-api.sigs.k8s.io/api-types/gateway/) in a kubernetes cluster.


# Motivation
[motivation]: #motivation

Allow additional stateful information about Gateway API resources to be made available via metrics.
Currently a set of metrics are made available via the [gateway-api-state-metrics](https://github.com/Kuadrant/gateway-api-state-metrics/blob/main/METRICS.md) project.
However, there are [limitations](https://github.com/Kuadrant/gateway-api-state-metrics/issues/1) with what resource information can be exposed using the underlying [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) project.
Additional stateful information would include:

- Individual listener status within a Gateway
- HTTPRoute status

For example, the individual status listener information from a Gateway:

```yaml
  status:
    listeners:
    - attachedRoutes: 1
      conditions:
      - lastTransitionTime: "2023-08-15T13:22:06Z"
        message: No errors found
        observedGeneration: 1
        reason: Ready
        status: "True"
        type: Ready
      - lastTransitionTime: "2023-08-15T13:22:06Z"
        message: No errors found
        observedGeneration: 1
        reason: ResolvedRefs
        status: "True"
        type: ResolvedRefs
      name: api
```

and HTTPRoute parents status conditions:

```yaml
  status:
    parents:
    - conditions:
      - lastTransitionTime: "2024-05-16T16:17:38Z"
        message: Object affected by AuthPolicy default/toystore
        observedGeneration: 1
        reason: Accepted
        status: "True"
        type: kuadrant.io/AuthPolicyAffected
      - lastTransitionTime: "2024-05-16T16:18:51Z"
        message: Object affected by RateLimitPolicy default/toystore
        observedGeneration: 1
        reason: Accepted
        status: "True"
        type: kuadrant.io/RateLimitPolicyAffected
      controllerName: kuadrant.io/policy-controller
      parentRef:
        group: gateway.networking.k8s.io
        kind: Gateway
        name: api-gateway
        namespace: kuadrant-system
    - conditions:
      - lastTransitionTime: "2024-05-16T16:17:38Z"
        message: Object affected by AuthPolicy default/toystore
        observedGeneration: 1
        reason: Accepted
        status: "True"
        type: kuadrant.io/AuthPolicyAffected
      - lastTransitionTime: "2024-05-16T16:18:51Z"
        message: Object affected by RateLimitPolicy default/toystore
        observedGeneration: 1
        reason: Accepted
        status: "True"
        type: kuadrant.io/RateLimitPolicyAffected
      - lastTransitionTime: "2024-05-20T11:45:33Z"
        message: Route was valid
        observedGeneration: 1
        reason: Accepted
        status: "True"
        type: Accepted
      - lastTransitionTime: "2024-05-20T11:45:33Z"
        message: All references resolved
        observedGeneration: 1
        reason: ResolvedRefs
        status: "True"
        type: ResolvedRefs
      controllerName: istio.io/gateway-controller
      parentRef:
        group: gateway.networking.k8s.io
        kind: Gateway
        name: api-gateway
        namespace: kuadrant-system
```

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

First, implement the existing Gateway API metrics that are part of the [gateway-api-state-metrics](https://github.com/Kuadrant/gateway-api-state-metrics/blob/main/METRICS.md) project.
This will allow the new prometheus exporter to replace that project as a whole.
The metrics will be backwards compatible with the gateway-api-state-metrics project, with the addition of new metrics.

Second, implement new metrics, as per the examples below, to capture the additional status information:

```promql
gatewayapi_gateway_status_listeners_conditions{namespace="<NAMESPACE>",name="<GATEWAY>",listener_name="<LISTENER_NAME>",type="<ResolvedRefs|Ready|Other>} 1
```
This metric captures the status condition types of individual listeners in a Gateway.
It will allow the status of individual named listeners to be queried, graphed and alerted on via metrics.
The health status of listeners can then be visualised in a stat panel in Grafana, showing healthy and unhealthy listeners.
This expands the debugging path beyond just the overall health of a Gateway.

```promql
gatewayapi_httproute_status_parents_conditions{namespace="<NAMESPACE>",name="<GATEWAY>",controller_name="<CONTROLLER_NAME>,"parent_group="<PARENT_GROUP>",parent_kind="<PARENT_KIND>",parent_name="<PARENT_NAME>",parent_namespace="<PARENT_NAMESPACE>",type="<ResolvedRefs|Accepted|Other>} 1
```
This metric captures the condition types for each parent of a HTTPRoute.
The `type` field would also record any custom types set by a controller. For example, `kuadrant.io/AuthPolicyAffected` and `kuadrant.io/RateLimitPolicyAffected`.
This will allow the health of HTTPRoutes to be reported via metrics. A HTTPRoute that has an `type` of Accepted and value of 1 means the HTTPRoute is accepted by the Gateway and can be considered healthy.
It will also allow policy specific information about a HTTPRoute to be represented in metrics.
For example, alerting on any HTTPRoutes that don't have the `kuadrant.io/AuthPolicyAffected` type with a value of 1 i.e. HTTPRoutes without an AuthPolicy.

Tests will be added directly to the project in a similar manner to the [redis-exporter](https://github.com/oliver006/redis_exporter/blob/master/exporter/http_test.go).
The test environment will bring up a kind cluster, create the Gateway API CRDs, example Gateway & HTTPRoute resources, then test the scrape endpoint.
This will be the same as how [metrics are tested](https://github.com/Kuadrant/gateway-api-state-metrics/blob/main/tests/e2e.sh) for the gateway-api-state-metrics project.
There is a [separate test function](https://github.com/Kuadrant/gateway-api-state-metrics/blob/main/tests/e2e/main_test.go) for each resource.

Existing [example dashboards](https://github.com/Kuadrant/gateway-api-state-metrics/tree/main/src/dashboards) in the gateway-api-state-metrics project will be copied over to the exporter project and continue to work as before.
However, initially it will just be the Gateway, GatewayClass and HTTPRoute dashboards as those will be the metrics that are implemented first.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The exporter will be written in golang and follow the guidelines from https://prometheus.io/docs/instrumenting/writing_exporters/.
Other exporters like the https://github.com/prometheus/node_exporter/tree/master and https://github.com/oliver006/redis_exporter will be referenced for patterns and library usage.
Metrics will only be pulled from the kubernetes API when Prometheus scrapes them.
That is, the exporter will not perform scrapes based on its own timers.
All scrapes will be synchronous.

The [client-go](https://github.com/kubernetes/client-go) library will be used for all kubernetes API calls.
As the number of Gateways and HTTPRoutes could vary greatly, there is a performance consideration with these API calls if there are a lot of resources.
To allow for this, a single list of all resources of a kind will used rather than 1 by 1.
If in future there are issues with performance, there is an option to cache responses to expensive queries.

A 'gateway_metrics_up` metric will be included, as per https://prometheus.io/docs/instrumenting/writing_exporters/#failed-scrapes
such that the exporter can continue to respond in a standard way if there are issues with some aspects of scraping.
The scrape response should include all metrics that have information available at that time of scraping.

# Drawbacks
[drawbacks]: #drawbacks

This is an additional library to maintain.
However, it will supercede the gateway-api-state-metrics project and dependency on kube-state-metrics.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

In theory it should be possible to get the desired functionality from the kube-state-metrics project if the proposed change in https://github.com/kubernetes/kube-state-metrics/pull/2059 is accepted and subsequently implemented.
However, that proposal has been open since May 2023.
I have ruled out the possibility of helping with the implementation of this change in that project due to:

- lack of detailed knowledge of the current implementation of selectors in kube-state-metrics
- potential complexity of implementing generic cel support in that project
- it not being core to the Kuadrant project goals, combined with the ongoing maintenance commitment after implementation. It wouldn't be fair to land the change we need without following up on maintenance after

The proposed design is a more focused solution on the needs of the Kuadrant project from Gateway API resources in the form of metrics.
There are plenty of examples of exporters out there that we can reference and follow established patterns.

If we don't make this change, we are limited to having just the overall Gateway status available via metrics,
and no HTTPRoute status information on which we can visualise and alert.

# Prior art
[prior-art]: #prior-art

The primary prior art are the kube-state-metrics project, and the gateway-api-state-metrics projects.
The gateway-api-state-metrics project uses the [CustomResourceStateMetrics](https://github.com/Kuadrant/gateway-api-state-metrics/blob/main/config/default/custom-resource-state.yaml) configuration feature of kube-state-metrics to configure what fields in which resources should be made available via metrics.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

n/a

# Future possibilities
[future-possibilities]: #future-possibilities

Although this exporter is intended to replace the gateway-api-state-metrics project,
it will likely take a phased approach to get to that point.
The initial goal is to get 'like for like' functionality from a Kuadrant project point of view (Gateways, GatewayClasses and HTTPRoutes),
followed by the new status functionality as detailed in this RFC.
Other resources, such as TLSRoute, UDPRoute etc.. can be added later.
