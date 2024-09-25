# Kuadrant API Conventions

This document outlines a none exhaustive set of conventions that should be used when defining Kuadrant APIs. It is initially focused around the concept of a policy API as these make up the bulk of our public facing APIs. It is expected that this set of conventions will grow as needed overtime.


## General rule of thumb

If in doubt, first check if a similar API or field exists in the [Gateway API spec](https://gateway-api.sigs.k8s.io/reference/spec/) and re-use that definition to specify what you need in your API. If you need to differ, ensure you explain why in any RFC or PR. 

Below are some common examples of these fields and specifications:

## Policy Attachment APIs

Policy APIs must follow the specification laid out by the Gateway API policy attachment designs. First step is to understand if you are going to design or are working on a direct or inherited policy. The linked docs will help with this but if unsure reach out on github or slack. 

Within the linked docs are set of requirements for each type of policy:

[Direct Policy Requirements](https://gateway-api.sigs.k8s.io/geps/gep-2648/#direct-policy-attachment-requirements-in-brief)

[Inherited Policy Requirements](https://gateway-api.sigs.k8s.io/geps/gep-2649/#inherited-policy-attachment-requirements-in-brief)

[Existing direct policy API](https://docs.kuadrant.io/0.10.0/kuadrant-operator/doc/reference/dnspolicy/)

[Existing inherited policy API](https://docs.kuadrant.io/0.10.0/kuadrant-operator/doc/reference/ratelimitpolicy/)

Beyond the set of requirement outlined in these specifications, Kuadrant has an additional set of conventions that are used in order to try and ensure our APIs remain consistent and familiar for users of the Gateway API. Most if not all of these come directly from looking at Gateway API.


### API Fields and Types

**Durations**

A duration is a field expressing something that configures behaviour that should happen according to a specified duration of time. Examples of this include but are not limited to:

- RateLimit counter expiry
- HealthCheck interval (how often to do the check)

Any API field that expresses this type of time bound behaviour should use a type of `time.Duration` allowing the duration to be specified in a single field and must match `^([0-9]{1,5}(h|m|s|ms)){1,4}$`: 

Examples:

`20ms,20s,20m,20h` 

[more detail and information](https://gateway-api.sigs.k8s.io/geps/gep-2257/#gateway-api-duration-format)


field names for this type of configuration are left up to the designer of the API as they are often specific to the task at hand: IE with a health check it is called an `interval` within a limit it is called a `duration` for the timeout filter in HTTPRoute it is called `request` and `backendRequest`

**hostname | hostnames**

When asking for a hostname the field in the API should be named `hostname` rather than other options such as `host, domain` etc. The type used for this should be a `string` or an alias to a `string`. The plural of this for an array should be `hostnames`. This also aligns with how the Gateway API specifies these fields. where possible re-use the Gateway API types:

[HostnameType](https://github.com/kubernetes-sigs/gateway-api/blob/main/apis/v1/shared_types.go#L523):

[Gateway Listener host](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io/v1.Listener)

[HTTPRoute hostnames](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io/v1.HTTPRouteSpec)



**protocol**

If there is a need to specify a protocol in the API the field for asking for this should be named: `protocol` and should accept a `string` or an alias to a string where possible use [ProtocolType](https://github.com/kubernetes-sigs/gateway-api/blob/main/apis/v1/gateway_types.go#L363) provided by Gateway API 

**port**

If there is a need to specify a port in the API, the field should be named `port` and should accept an `int32` with a min of 1 and a max of 65535. Where possible use [PortNumberType](https://github.com/kubernetes-sigs/gateway-api/blob/main/apis/v1/shared_types.go#L233) provided by Gateway API 


**status**

For more information on status please refer to 

[Policy Status](../rfcs/0004-policy-status.md)