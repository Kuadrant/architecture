# Tracing Sampling Strategy Rules

- Feature Name: tracing_sampling_strategy_rules
- Start Date: 2024-07-09
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/issues/0000)

# Summary
[summary]: #summary

Extend the tracing configuration in Kuadrant components to allow more complex sampling strategy rules.
These rules will be able to use [well-known attributes](./0002-well-known-attributes.md) to make decisions on what traffic should be sampled.
Additionally, users will be able to add well-known attributes as fields to spans emitted by Kuadrant components.
At the time of writing, the Kuadrant components of concern are those that request traffic can pass through. That is, Authorino and Limitador.
That being said, the feature being proposed here should not be limited to them only and may be relevant to some future components as well.

# Motivation
[motivation]: #motivation

Allow more complex sampling strategy rules that let users target the traffic they are most interested in getting tracing information about.
For example, only trace 0.1% of regular user traffic, and 50% of admin user traffic, based on user group membership.
Allowing users to specify these kinds of rules will also give them control of the amount of data being captured, which can have cost implications.
Allowing users to add well-known attributes as fields to spans will assist them in debugging request related problems.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Currently, to [enable tracing in Kuadrant components](https://docs.kuadrant.io/0.8.0/kuadrant-operator/doc/observability/tracing/), tracing configuration must be added in the following places:

- The gateway provider e.g. Istio, via a Telemetry & Istio resource. This is where a `randomSamplingPercentage` can be set for all traffic.
- The Authorino CR. The collector endpoint to send spans to is configured in `tracing.endpoint`. Whether or not secure grpc is used is configured in `tracing.insecure`
- The Limitador CR. The collector endpoint to send spans to is configured in `tracing.endpoint`.

The ability to set configuration options for sampling strategy rules will be made available to users.
However, to make it easier for users to configure tracing across Kuadrant components, all the existing tracing configuration, as well as any new configuration, will be abstracted back to a central Kuadrant API.
A new `ObservabilityPolicy` custom resource will be introduced to expose this new Kuadrant API.
Users will be able to configure the tracing endpoint, sampling stratgey rules, and any additional fields to include in spans in this resource.
The kuadrant operator will configure all the components accordingly.
The sampling strategy rules will allow users to specify if traffic should be sampled based on values of [well-known attributes](./0002-well-known-attributes.md).

NOTE: Before example ObservabilityPolicy resources can be given here, there are unresolved questions to be resolved further below.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

As the new `ObservabilityPolicy` custom resource will be a Kuadrant API, the entry point of changes will be in the kuadrant-operator.
How the configuration in this resource propegates down to the Kuadrant components is TBD.
The status of the configuration in the individual components should propegate back to the status block of the `ObservabilityPolicy` resource.
Changes will be required in both Limitador and Authorino to support the new sampling strategy & span field rules.
Depending on the integration point between the Kuadrant operator and those components, changes may also be required in the Limitador and Authorino operators.
What those rules look like, and how this configuration is passed to those resources is TBD.

NOTE: Before a solution can be further detailed, there are unresolved questions to be resolved further below.

# Drawbacks
[drawbacks]: #drawbacks

The proposed changes add extra complexity to multiple Kuadrant components.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

If this is work is not done, users will be restricted in how they can specify what traffic passing through Kuadrant should be sampled for tracing.
Those restrictions will depend on what configuration the gateway provider exposes, like in the [Telemetry resource](https://istio.io/latest/docs/reference/config/telemetry/#Tracing) in Istio.
They will also not be able to add extra attributes as fields to spans, potentially limiting the ability to effectively troublshoot.

# Prior art
[prior-art]: #prior-art

TBD

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Should the existing tracing configuration in the Authorino & Limitador CRs be [abstracted back to the Kuadrant CR](https://github.com/Kuadrant/kuadrant-operator/issues/731) as an iterative improvement prior to introducing the `ObservabilityPolicy`?
- Would an `ObservabilityPolicy` be a kuadrant operator API that takes care of configuring both limitador and authorino:
  - directly - like setting flags or configuration directly on the deployments of authorino & limitador?
  - indirectly - and Authornio & limitador have their own APIs in the Authorino & LImitador CRs?
  - indirectly - and Authornio & limitador have their own APIs in the form of `ObservabilityPolicy` CRs too?
- How should sampling strategy rules be specified? Some possibilities are:
  - [WhenConditions](https://github.com/Kuadrant/kuadrant-operator/blob/bed695f7ba75a1d4576c5f1205c745e0910f0e81/api/v1beta2/ratelimitpolicy_types.go#L79-L90), as used in RateLimitPolicy resources
  - [PatternExpressions](https://github.com/Kuadrant/authorino/blob/27066876c239e848e3a07b8774bf3f1b6a963954/api/v1beta2/auth_config_types.go#L153-L164), as used in Authorino AuthConfig resources.
  - [Common Expression Language](https://github.com/google/cel-spec) (CEL)

# Future possibilities
[future-possibilities]: #future-possibilities

- Allow for defaults and overrides for configuration in different regions. This can be useful for compliance and regulations reasons like keeping PII in tracing data in a specific geographical region.
- Alerting on tracing configuration if it's in violation of regulations or not in compliance.
