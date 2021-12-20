# Kuadrant Proposal - API

Kuadrant aims to expose a simple API, yet flexible enough to accomodate API Management requirements.

## Problem Statement

Currently, the [latest](https://github.com/Kuadrant/kuadrant-controller/releases/tag/v0.2.0) release of
kuadrant API is based on [apiproducts.v1beta1.networking.kuadrant.io](https://github.com/Kuadrant/kuadrant-controller/blob/v0.2.0/doc/architecture.md#api-product-crd)
and [apis.v1beta1.networking.kuadrant.io](https://github.com/Kuadrant/kuadrant-controller/blob/v0.2.0/doc/architecture.md#api-crd)
custom kubernetes resource APIs. Together with these CRDs, there is a bunch of kuadrant service
annotations and labels available in order to provide the kuadrant
[service discovery](https://github.com/Kuadrant/kuadrant-controller/blob/v0.2.0/doc/service-discovery.md)
feature. All together, kuadrant provides routing, basic and global rate limiting and, finally,
global authN (not authZ) features to protect customer owned APIs. While, good enough for a wide set
of use cases, there are several limitations:

* Rate limit is enforced to all services and endpoints defined in a APIProduct. It cannot be specified route/path wise.
* Only high level rate limit directives are defined (`global`, `perRemoteIP` and `authenticated`). There is no option to specify with a fine-grained rate limit API.
* Authentication is enforced to all services and endpoints defined in a APIProduct. It cannot be specified route/path wise.
* Authentication is limited to `API key` and `OpenID Connect`.
* Authorization is not available.

## High Level Goals

* Expose simple API, API management oriented, to protect customer owned API's.
* Simple API can be extended with a fine-grained API to leverage full featured Authorino API and Limitador API.
* Path/Operation based protection (i.e. authentication and rate limit)

## Proposed Solution

The kuadrant API will be oriented to meet API management requirements. In that context, the kuadrant
API will be designed around the OpenAPI standard.
Specifically, [OpenAPI 3.X specification](https://github.com/OAI/OpenAPI-Specification/blob/main/versions/3.0.2.md).
Kuadrant will define the *APIProduct* object class (a CRD in terms of kubernetes) to wrap a set of
API management configurations on top of a list of customer owened API definitions.

### API Product

**Custom Resource Definition GVK**

Currently, as of kuadrant v0.2.0, APIProduct [Group Version Kind](https://book.kubebuilder.io/cronjob-tutorial/gvks.html)
is: [apiproducts.v1beta1.networking.kuadrant.io](https://github.com/Kuadrant/kuadrant-controller/blob/v0.2.0/doc/architecture.md#api-product-crd)

Taking into account kuadrant is not providing networking level API, but **API Management** level API,
the proposed GVK is: **apiproducts.v1alpha1.apim.kuadrant.io**.

**Domains**

APIProduct resource will be configured with a list of destination domains or hostnames to which
traffic is being sent.

It is the responsability of cluster operators and API owners to forward *îngress* traffic for those
specific domains to the kuadrant gateway. A kuadrant gateway would be a gateway configured by the
kuadrant control plane, having as input the API Product resource.

For example, cluster operators can deploy required (cloud provider level) networking resources to
forward traffic to the cluster ingress gateways. Then, the API owners can create (cluster level)
networking resources, like kubernetes Ingress or Gateway API HttpRoutes, to route that traffic to
the kuadrant gateways. In this example only *North/South* traffic was referenced. By design,ç
it could also be *East/west* traffic. By design, there is nothing preventing from having *East/West*
traffic being protected by kuadrant. This is mainly because kuadrant is not a networking API.

**Protected API specification**

Kuadrant will support the open standard
[OpenAPI 3.X specification](https://github.com/OAI/OpenAPI-Specification/blob/main/versions/3.0.2.md)
for the API spec.

Each *API Product* can hold multiple OAS docs references. The *API Product* `spec` field includes references
in the form of URLs and config map references. The actual content may be included in the `status` field.

Kuadrant will extend the OAS doc leveraging
[OAS custom extensions](https://github.com/OAI/OpenAPI-Specification/blob/main/versions/3.0.2.md#specificationExtensions)
`x-kuadrant-*`, initially for rate limit purposes. In the future, OAS extensions could be leveraged
for other API management features, for instance usage plans. More about the kuadrant custom
extensions in the spec reference section.

In case of lacking one OpenAPI doc, kuadrant will provide tooling for easy generation of OpenAPI docs.

Any kuadrant configuration at API path or operation level will be defined as OpenAPI kuadrant extension.

**Backends - Upstreams**

The protected API source, owned by the kuadrant customer, can be specified in two forms:
* A (base) URL
* A kubernetes service reference (name and namespace).

When the source is a kubernetes service, kuadrant provides annotations and labels that can be added
to the service. These annotations and labels can specify, for example, the OAS reference.
These annotations and labels aim to make the `spec` of the *API Product* resource simpler to specify.

**AuthConfig Templates**

The configuration of the API authentication requires *three* steps. Only the first
two steps involve kuadrant, but all three of them are required to have the system working.
For completeness, all of them are listed here.

1. Operation based authentication config

The customer needs to specify which API operation (method + path) requires authentication and
which type of authentication (API key, oauth2, oidc, http basic, etc). OpenAPI spec fully covers
this use case and kuadrant can read this configuration from OAS doc. For any authN type not covered
by OAS 3.X, kuadrant could provide extensions for added authentication types. OpenAPI 3.x does
not support authentication requirements at path level, only global or operation level.

1. Configuration for each type of authentication

The customer needs to specify the authentication provider configuration. Kuadrant provides Authorino
`AuthConfig` custom resource to specify authentication provider configuration. The *API Product*
resource will include a template object in the `.spec` to specify `AuthConfig` objects.

The *API Product* resource allows to define multiple AuthConfig templates.

To illustrate this with examples, the API Key auth type would require `AuthConfig` template with
`spec.identity[].apiKey` object. The OIDC auth type would require `AuthConfig` template with
`spec.identity[].oidc` object.

Kuadrant can add some validations to prevent inconcistencies. For instance, if the OAS doc specifies
that API key is being required for some API operations, then kuadrant can make sure that the
`AuthConfig` template includes `apiKey` identity type.

Actually, Authorino's `AuthConfig` covers far more than authentication. The customer can do more
than just authentication, the customer can apply authorization policies as well.

1. Authentication data provisioning

The customer needs to provision some authentication data. For the API key use case, the customer
needs to generate secrets with the actual tokens. For the OIDC use case, the customer needs to
provision OAUTH2 clients and users in the Authentication provider.

**RateLimit Template**

The configuration of the API rate limit requires *two* steps.

1. Path/Operation based rate limit config

The customer needs to define which API paths/operations require rate limit enforcing.
Furthermore, the customer needs to define what type of rate limit needs to be enforced at
path/operation level. Global rate limit is a particular case where all API operations require
enforcing rate limit of a particular rate limit type.

At each API path/operation level, the customer may define
[envoy rate limit actions](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#envoy-v3-api-msg-config-route-v3-ratelimit)
which populates a vector of descriptor entries sent to the rate limit service (Limitador in kuadrant).

The rate limit actions will be defined as OpenAPI 3.X kuadrant extensions at global, path or
operation level.

1. Rate limit configuration

The customer needs to specify the rate limit service provider configuration. Kuadrant provides
Limitador's `RateLimit` custom resource to specify rate limit configuration. The *API Product*
resource will include a template object in the `.spec` to specify `RateLimit` objects.

The *API Product* resource allows to define multiple `RateLimit` templates.

### API Product Reference

| Field | Type | Description | Required |
| --- | --- | --- | --- |
| domains | `string[]` | The destination authority to which traffic is being sent | yes |

**Example**




## Next steps

* Usage plans: design kuadrant plans to define rate limits based on kuadrant metrics. Kuadrant metrics will be associated to API operations.
* Developer accounts and portal.
* Gateway level policies for multiple use cases, for example:
  * Header manipulation
  * URL rewritting
  * Websockets
  * HTTP2 endpoint
  * Upstream TLS client certificate
  * ...
