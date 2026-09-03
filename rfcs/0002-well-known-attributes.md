# Well-known Attributes

- Feature Name: `well-known-attributes`
- Start Date: 2023-06-13
- RFC PR: [Kuadrant/architecture#17](https://github.com/Kuadrant/architecture/pull/17)
- Issue tracking: [Kuadrant/architecture#53](https://github.com/Kuadrant/architecture/issues/53)

# Summary
[summary]: #summary

Define a well-known structure for users to declare request data selectors in their RateLimitPolicies and AuthPolicies. This structure is referred to as the Kuadrant _Well-known Attributes_.

# Motivation
[motivation]: #motivation

The well-known attributes let users write policy rules – conditions and, in general, dynamic values that refer to attributes in the data plane - in a concise and seamless way.

Decoupled from the policy CRDs, the well-known attributes:
1. define a common language for referring to values of the data plane in the Kuadrant policies;
2. allow dynamically evolving the policy APIs regarding how they admit references to data plane attributes;
3. encompass all common and component-specific selectors for data plane attributes;
4. have a single and unified specification, although this specification may occasionally link to additional, component-specific, external docs.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

One who writes a Kuadrant policy and wants to build policy constructs such as conditions, qualifiers, variables, etc, based on dynamic values of the data plane, must refer the attributes that carry those values, using the declarative language of Kuadrant's _Well-known Attributes_.

A dynamic data plane value is typically a value of an attribute of the request or an [Envoy Dynamic Metadata](https://www.envoyproxy.io/docs/envoy/latest/configuration/advanced/well_known_dynamic_metadata) entry. It can be a value of the outer request being handled by the API gateway or proxy that is managed by Kuadrant ("context request") or an attribute of the direct request to the Kuadrant component that delivers the functionality in the data plane (rate-limiting or external auth).

A _Well-known Selector_ is a construct of a policy API whose value contains a direct reference to a well-known attribute. The language of the well-known attributes and therefore what one would declare within a well-known selector resembles a JSON path for navigating a possibly complex JSON object.

**Example 1.** Well-known selector used in a condition

```yaml
apiGroup: examples.kuadrant.io
kind: PaintPolicy
spec:
  rules:
  - when:
    - selector: auth.identity.group
      operator: eq
      value: admin
    color: red
```

In the example, `auth.identity.group` is a well-known selector of an attribute `group`, known to be injected by the external authorization service (`auth`) to describe the group the user (`identity`) belongs to. In the data plane, whenever this value is equal to `admin`, the abstract `PaintPolicy` policy states that the traffic must be painted `red`.

**Example 2.** Well-known selector used in a variable

```yaml
apiGroup: examples.kuadrant.io
kind: PaintPolicy
spec:
  rules:
  - color: red
    alpha:
      dynamic: request.headers.x-color-alpha
```

In the example, `request.headers.x-color-alpha` is a selector of a well-known attribute `request.headers` that gives access to the headers of the context HTTP request. The selector retrieves the value of the `x-color-alpha` request header to dynamically fill the `alpha` property of the abstract `PaintPolicy` policy at each request.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The _Well-known Attributes_ are a compilation inspired by some of the [**Envoy attributes**](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/advanced/attributes) and Authorino's [**Authorization JSON**](https://github.com/Kuadrant/authorino/blob/main/docs/architecture.md#the-authorization-json) and its related [JSON paths](https://github.com/Kuadrant/authorino/blob/main/docs/features.md#common-feature-json-paths-valuefromauthjson).

From the Envoy attributes, only attributes that are available before establishing connection with the upstream server qualify as a Kuadrant well-known attribute. This excludes attributes such as the response attributes and the upstream attributes.

As for the attributes inherited from Authorino, these are either based on Envoy's [`AttributeContext`](https://pkg.go.dev/github.com/envoyproxy/go-control-plane/envoy/service/auth/v3?utm_source=gopls#AttributeContext) type of the external auth request API or from internal types defined by Authorino to fulfill the [Auth Pipeline](https://github.com/Kuadrant/authorino/blob/main/docs/architecture.md#the-auth-pipeline-aka-enforcing-protection-in-request-time).

These two subsets of attributes are unified into a single set of well-known attributes. For each attribute that exists in both subsets, the name of the attribute as specified in the Envoy attributes subset prevails. Example of such is `request.id` (to refer to the ID of the request) superseding `context.request.http.id` (as the same attribute is referred in an Authorino [`AuthConfig`](https://github.com/Kuadrant/authorino/blob/main/docs/architecture.md#the-authorino-authconfig-custom-resource-definition-crd)).

<br/>

The next sections specify the well-known attributes organized in the following groups:
- [Request attributes](#request-attributes)
- [Connection attributes](#connection-attributes)
- [Metadata attribute](#metadata-and-filter-state-attributes)
- [Auth attributes](#auth-attributes)
- [Rate-limit attributes](#rate-limit-attributes)

## Request attributes

The following attributes are related to the context HTTP request that is handled by the API gateway or proxy managed by Kuadrant.

<table>
  <thead>
    <tr>
      <th><p>Attribute</p></th>
      <th><p>Type</p></th>
      <th><p>Description</p></th>
      <th><p>Auth</p></th>
      <th><p>RL</p></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><p>request.id</p></td>
      <td><p>String</p></td>
      <td><p>Request ID corresponding to <code><span>x-request-id</span></code> header value</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p>✓</p></td>
    </tr>
    <tr>
      <td><p>request.time</p></td>
      <td><p>Timestamp</p></td>
      <td><p>Time of the first byte received</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p>✓</p></td>
    </tr>
    <tr>
      <td><p>request.protocol</p></td>
      <td><p>String</p></td>
      <td><p>Request protocol (“HTTP/1.0”, “HTTP/1.1”, “HTTP/2”, or “HTTP/3”)</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p>✓</p></td>
    </tr>
    <tr>
      <td><p>request.scheme</p></td>
      <td><p>String</p></td>
      <td><p>The scheme portion of the URL e.g. “http”</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p>✓</p></td>
    </tr>
    <tr>
      <td><p>request.host</p></td>
      <td><p>String</p></td>
      <td><p>The host portion of the URL</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p>✓</p></td>
    </tr>
    <tr>
      <td><p>request.method</p></td>
      <td><p>String</p></td>
      <td><p>Request method e.g. “GET”</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p>✓</p></td>
    </tr>
    <tr>
      <td><p>request.path</p></td>
      <td><p>String</p></td>
      <td><p>The path portion of the URL</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p>✓</p></td>
    </tr>
    <tr>
      <td><p>request.url_path</p></td>
      <td><p>String</p></td>
      <td><p>The path portion of the URL without the query string</p></td>
      <td align="center"><p></p></td>
      <td align="center"><p>✓</p></td>
    </tr>
    <tr>
      <td><p>request.query</p></td>
      <td><p>String</p></td>
      <td><p>The query portion of the URL in the format of “name1=value1&amp;name2=value2”</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p>✓</p></td>
    </tr>
    <tr>
      <td><p>request.headers</p></td>
      <td><p>Map&lt;String, String&gt;</p></td>
      <td><p>All request headers indexed by the lower-cased header name</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p>✓</p></td>
    </tr>
    <tr>
      <td><p>request.referer</p></td>
      <td><p>String</p></td>
      <td><p>Referer request header</p></td>
      <td align="center"><p></p></td>
      <td align="center"><p>✓</p></td>
    </tr>
    <tr>
      <td><p>request.useragent</p></td>
      <td><p>String</p></td>
      <td><p>User agent request header</p></td>
      <td align="center"><p></p></td>
      <td align="center"><p>✓</p></td>
    </tr>
    <tr>
      <td><p>request.size</p></td>
      <td><p>Number</p></td>
      <td><p>The HTTP request size in bytes. If unknown, it must be -1</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p></p></td>
    </tr>
    <tr>
      <td><p>request.body</p></td>
      <td><p>String</p></td>
      <td><p>The HTTP request body. (Disabled by default. Requires additional proxy configuration to enabled it.)</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p></p></td>
    </tr>
    <tr>
      <td><p>request.raw_body</p></td>
      <td><p>Array&lt;Number&gt;</p></td>
      <td><p>The HTTP request body in bytes. This is sometimes used instead of <code>body</code> depending on the proxy configuration.</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p></p></td>
    </tr>
    <tr>
      <td><p>request.context_extensions</p></td>
      <td><p>Map&lt;String, String&gt;</p></td>
      <td><p>This is analogous to <code>request.headers</code>, however these contents are not sent to the upstream server. It provides an extension mechanism for sending additional information to the auth service without modifying the proto definition. It maps to the internal opaque context in the proxy filter chain. (Requires additional configuration in the proxy.)</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p></p></td>
    </tr>
  </tbody>
</table>

## Connection attributes

The following attributes are available once the downstream connection with the API gateway or proxy managed by Kuadrant is established. They apply to HTTP requests (L7) as well, but also to proxied connections limited at L3/L4.

<table>
  <thead>
    <tr>
      <th><p>Attribute</p></th>
      <th><p>Type</p></th>
      <th><p>Description</p></th>
      <th><p>Auth</p></th>
      <th><p>RL</p></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><p>source.address</p></td>
      <td><p>String</p></td>
      <td><p>Downstream connection remote address</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p>✓</p></td>
    </tr>
    <tr>
      <td><p>source.port</p></td>
      <td><p>Number</p></td>
      <td><p>Downstream connection remote port</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p>✓</p></td>
    </tr>
    <tr>
      <td><p>source.service</p></td>
      <td><p>String</p></td>
      <td><p>The canonical service name of the peer</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p></p></td>
    </tr>
    <tr>
      <td><p>source.labels</p></td>
      <td><p>Map&lt;String, String&gt;</p></td>
      <td><p>The labels associated with the peer. These could be pod labels for Kubernetes or tags for VMs. The source of the labels could be an X.509 certificate or other configuration.</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p></p></td>
    </tr>
    <tr>
      <td><p>source.principal</p></td>
      <td><p>String</p></td>
      <td><p>The authenticated identity of this peer. If an X.509 certificate is used to assert the identity in the proxy, this field is sourced from “URI Subject Alternative Names“, “DNS Subject Alternate Names“ or “Subject“ in that order. The format is issuer specific – e.g. SPIFFE format is <code>spiffe://trust-domain/path</code>, Google account format is <code>https://accounts.google.com/{userid}</code>.</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p></p></td>
    </tr>
    <tr>
      <td><p>source.certificate</p></td>
      <td><p>String</p></td>
      <td><p>The X.509 certificate used to authenticate the identify of this peer. When present, the certificate contents are encoded in URL and PEM format.</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p></p></td>
    </tr>
    <tr>
      <td><p>destination.address</p></td>
      <td><p>String</p></td>
      <td><p>Downstream connection local address</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p>✓</p></td>
    </tr>
    <tr>
      <td><p>destination.port</p></td>
      <td><p>Number</p></td>
      <td><p>Downstream connection local port</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p>✓</p></td>
    </tr>
    <tr>
      <td><p>destination.service</p></td>
      <td><p>String</p></td>
      <td><p>The canonical service name of the peer</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p></p></td>
    </tr>
    <tr>
      <td><p>destination.labels</p></td>
      <td><p>Map&lt;String, String&gt;</p></td>
      <td><p>The labels associated with the peer. These could be pod labels for Kubernetes or tags for VMs. The source of the labels could be an X.509 certificate or other configuration.</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p></p></td>
    </tr>
    <tr>
      <td><p>destination.principal</p></td>
      <td><p>String</p></td>
      <td><p>The authenticated identity of this peer. If an X.509 certificate is used to assert the identity in the proxy, this field is sourced from “URI Subject Alternative Names“, “DNS Subject Alternate Names“ or “Subject“ in that order. The format is issuer specific – e.g. SPIFFE format is <code>spiffe://trust-domain/path</code>, Google account format is <code>https://accounts.google.com/{userid}</code>.</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p></p></td>
    </tr>
    <tr>
      <td><p>destination.certificate</p></td>
      <td><p>String</p></td>
      <td><p>The X.509 certificate used to authenticate the identify of this peer. When present, the certificate contents are encoded in URL and PEM format.</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p></p></td>
    </tr>
    <tr>
      <td><p>connection.id</p></td>
      <td><p>Number</p></td>
      <td><p>Downstream connection ID</p></td>
      <td align="center"><p></p></td>
      <td align="center"><p>✓</p></td>
    </tr>
    <tr>
      <td><p>connection.mtls</p></td>
      <td><p>Boolean</p></td>
      <td><p>Indicates whether TLS is applied to the downstream connection and the peer ceritificate is presented</p></td>
      <td align="center"><p></p></td>
      <td align="center"><p>✓</p></td>
    </tr>
    <tr>
      <td><p>connection.requested_server_name</p></td>
      <td><p>String</p></td>
      <td><p>Requested server name in the downstream TLS connection</p></td>
      <td align="center"><p></p></td>
      <td align="center"><p>✓</p></td>
    </tr>
    <tr>
      <td><p>connection.tls_session.sni</p></td>
      <td><p>String</p></td>
      <td><p>SNI used for TLS session</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p></p></td>
    </tr>
    <tr>
      <td><p>connection.tls_version</p></td>
      <td><p>String</p></td>
      <td><p>TLS version of the downstream TLS connection</p></td>
      <td align="center"><p></p></td>
      <td align="center"><p>✓</p></td>
    </tr>
    <tr>
      <td><p>connection.subject_local_certificate</p></td>
      <td><p>String</p></td>
      <td><p>The subject field of the local certificate in the downstream TLS connection</p></td>
      <td align="center"><p></p></td>
      <td align="center"><p>✓</p></td>
    </tr>
    <tr>
      <td><p>connection.subject_peer_certificate</p></td>
      <td><p>String</p></td>
      <td><p>The subject field of the peer certificate in the downstream TLS connection</p></td>
      <td align="center"><p></p></td>
      <td align="center"><p>✓</p></td>
    </tr>
    <tr>
      <td><p>connection.dns_san_local_certificate</p></td>
      <td><p>String</p></td>
      <td><p>The first DNS entry in the SAN field of the local certificate in the downstream TLS connection</p></td>
      <td align="center"><p></p></td>
      <td align="center"><p>✓</p></td>
    </tr>
    <tr>
      <td><p>connection.dns_san_peer_certificate</p></td>
      <td><p>String</p></td>
      <td><p>The first DNS entry in the SAN field of the peer certificate in the downstream TLS connection</p></td>
      <td align="center"><p></p></td>
      <td align="center"><p>✓</p></td>
    </tr>
    <tr>
      <td><p>connection.uri_san_local_certificate</p></td>
      <td><p>String</p></td>
      <td><p>The first URI entry in the SAN field of the local certificate in the downstream TLS connection</p></td>
      <td align="center"><p></p></td>
      <td align="center"><p>✓</p></td>
    </tr>
    <tr>
      <td><p>connection.uri_san_peer_certificate</p></td>
      <td><p>String</p></td>
      <td><p>The first URI entry in the SAN field of the peer certificate in the downstream TLS connection</p></td>
      <td align="center"><p></p></td>
      <td align="center"><p>✓</p></td>
    </tr>
    <tr>
      <td><p>connection.sha256_peer_certificate_digest</p></td>
      <td>String</td>
      <td><p>SHA256 digest of the peer certificate in the downstream TLS connection if present</p></td>
      <td align="center"><p></p></td>
      <td align="center"><p>✓</p></td>
    </tr>
  </tbody>
</table>

## Metadata attribute

The following attribute is related to the Envoy proxy filter chain. They include metadata exported by the proxy throughout the filters.

<table>
  <thead>
    <tr>
      <th><p>Attribute</p></th>
      <th><p>Type</p></th>
      <th><p>Description</p></th>
      <th><p>Auth</p></th>
      <th><p>RL</p></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><p>metadata</p></td>
      <td><p><a href="https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/base.proto#envoy-v3-api-msg-config-core-v3-metadata">Metadata</a></p></td>
      <td><p>Dynamic request metadata</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p>✓</p></td>
    </tr>
  </tbody>
</table>

## Auth attributes

The following attributes are exclusive of the external auth service (Authorino).

<table>
  <thead>
    <tr>
      <th><p>Attribute</p></th>
      <th><p>Type</p></th>
      <th><p>Description</p></th>
      <th><p>Auth</p></th>
      <th><p>RL</p></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><p>auth.identity</p></td>
      <td><p>Any</p></td>
      <td><p>Single resolved identity object, post-identity verification</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p></p></td>
    </tr>
    <tr>
      <td><p>auth.metadata</p></td>
      <td><p>Map&lt;String, Any&gt;</p></td>
      <td><p>External metadata fetched</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p></p></td>
    </tr>
    <tr>
      <td><p>auth.authorization</p></td>
      <td><p>Map&lt;String, Any&gt;</p></td>
      <td><p>Authorization results resolved by each authorization rule, access granted only</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p></p></td>
    </tr>
    <tr>
      <td><p>auth.response</p></td>
      <td><p>Map&lt;String, Any&gt;</p></td>
      <td><p>Response objects exported by the auth service post-access granted</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p></p></td>
    </tr>
    <tr>
      <td><p>auth.callbacks</p></td>
      <td><p>Map&lt;String, Any&gt;</p></td>
      <td><p>Response objects returned by the callback requests issued by the auth service</p></td>
      <td align="center"><p>✓</p></td>
      <td align="center"><p></p></td>
    </tr>
  </tbody>
</table>

The auth service also supports modifying selected values by chaining [modifiers](https://github.com/Kuadrant/authorino/blob/main/docs/features.md#string-modifiers) in the path.

## Rate-limit attributes

The following attributes are exclusive of the rate-limiting service (Limitador).

<table>
  <thead>
    <tr>
      <th><p>Attribute</p></th>
      <th><p>Type</p></th>
      <th><p>Description</p></th>
      <th><p>Auth</p></th>
      <th><p>RL</p></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><p>ratelimit.domain</p></td>
      <td><p>String</p></td>
      <td><p>The rate limit domain. This enables the configuration to be namespaced per application (multi-tenancy).</p></td>
      <td align="center"><p></p></td>
      <td align="center"><p>✓</p></td>
    </tr>
    <tr>
      <td><p>ratelimit.hits_addend</p></td>
      <td><p>Number</p></td>
      <td><p>Specifies the number of hits a request adds to the matched limit. Fixed value: `1`. Reserved for future usage.</p></td>
      <td align="center"><p></p></td>
      <td align="center"><p>✓</p></td>
    </tr>
  </tbody>
</table>

# Drawbacks
[drawbacks]: #drawbacks

The decoupling of the well-known attributes and the language of well-known attributes and selectors from the individual policy CRDs is what makes it somewhat flexible and common across the components (rate-limiting and auth). However, it's less structured and it introduces another syntax for users to get familiar with.

This additional language competes with the language of the route selectors ([RFC 0001](https://github.com/Kuadrant/architecture/blob/main/rfcs/0001-rlp-v2.md)), based on Gateway API's [`HTTPRouteMatch`](https://gateway-api.sigs.k8s.io/v1alpha2/references/spec/#gateway.networking.k8s.io/v1beta1.HTTPRouteMatch) type.

Being "soft-coded" in the policy specs (as opposed to a hard-coded sub-structure inside of each policy type) does not mean it's completely decoupled from implementation in the control plane and/or intermediary data plane components. Although many attributes can be supported almost as a pass-through, from being used in a selector in a policy, to a corresponding value requested by the wasm-shim to its host, that is not always the case. Some translation may be required for components not integrated via wasm-shim (e.g. Authorino), as well as for components integrated via wasm-shim (e.g. Limitador) in special cases of composite or abstraction well-known attributes (i.e. attributes not available as-is via ABI, e.g. `auth.identity` in a RLP). Either way, some validation of the values introduced by users in the selectors may be needed at some point in the control plane, thus requiring arguably a level of awaresness and coupling between the well-known selectors specification and the control plane (policy controllers) or intermediary data plane (wasm-shim) components.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

As an alternative to JSON path-like selectors based on a well-known structure that induces the proposed language of well-known attributes, these same attributes could be defined as sub-types of each policy CRD. The Golang packages defining the common attributes across CRDs could be shared by the policy type definitions to reduce repetition. However, that approach would possibly involve a staggering number of new type definitions to cover all the cases for all the groups of attributes to be supported. These are constructs that not only need to be understood by the policy controllers, but also known by the user who writes a policy.

Additionally, all attributes, including new attributes occasionally introduced by Envoy and made available to the wasm-shim via ABI, would always require translation from the user-level abstraction how it's represented in a policy, to the actual form how it's used in the wasm-shim configuration and Authorino AuthConfigs.

Not implementing this proposal and keeping the current state of things mean little consistency between these common constructs for rules and conditions on how they are represented in each type of policy. This lack of consistency has a direct impact on the overhead faced by users to learn how to interact with Kuadrant and write different kinds of policies, as well as for the maintainers on tasks of coding for policy validation and reconciliation of data plane configurations.

# Prior art
[prior-art]: #prior-art

Authorino's dynamic [JSON paths](https://github.com/Kuadrant/authorino/blob/main/docs/features.md#common-feature-json-paths-valuefromauthjson), related to Authorino's [Authorization JSON](https://github.com/Kuadrant/authorino/blob/main/docs/architecture.md#the-authorization-json) and used in `when` conditions and inside of multiple other constructs of the AuthConfig, are an example of feature of very similar approach to the one proposed here.

Arguably, Authorino's perceived flexibility would not have been possible with the Authorization JSON selectors. Users can write quite sophisticated policy rules (conditions, variable references, etc) by leveraging the those dynamic selectors. Because they are backed by JSON-based machinery in the code, Authorino's selectors have very little to, in some cases, none at all variation compared [Open Policy Agent's Rego policy language](https://www.openpolicyagent.org/docs/latest/policy-reference/), which is often used side by side in the same AuthConfigs.

Authorino's Authorization JSON selectors are, in one hand, more restrict to the structure of the [`CheckRequest`](https://pkg.go.dev/github.com/envoyproxy/go-control-plane/envoy/service/auth/v3?utm_source=gopls#CheckRequest) payload (`context.*` attributes). At the same time, they are very open in the part associated with the internal attributes built along the Auth Pipeline (i.e. `auth.*` attributes). That makes Authorino's Authorization JSON selectors more limited, compared to the [Envoy attributes](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/advanced/attributes) made available to the wasm-shim via ABI, but also harder to validate. In some cases, such as of deep references to inside objects fetched from external sources of metadata, resolved OPA objects, JWT claims, etc, it is impossible to validate for correct references.

Another experience learned from Authorino's Authorization JSON selectors is that they depend substantially on the so-called "[modifiers](https://github.com/Kuadrant/authorino/blob/main/docs/features.md#string-modifiers)". Many use cases involving parsing and breaking down attributes that are originally available in a more complex form would not be possible without the modifiers. Examples of such cases are: extracting portions of the path and/or query string parameters (e.g. collection and resource identifiers), applying translations on HTTP verbs into corresponding operations, base64-decoding values from the context HTTP request, amongst several others.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

1. How to deal with the differences regarding the availability and data types of the attributes across clients/hosts?

2. Can we make more attributes that are currently available to only one of the components common to both?

3. Will we need some kind of global support for modifiers (functions) in the well-known selectors or those can continue to be an Authorino-only feature?

4. Does Authorino, which is more strict regarding the data structure that induces the selectors, need to implement this specification or could/should it keep its current selectors and a translation be performed by the AuthPolicy controller?

# Future possibilities
[future-possibilities]: #future-possibilities

1. Extend with more well-known attributes that abstract common patterns and/or for rather opinioned use cases. Examples:
   - `auth.*` attributes supported in the rate limit service
   - `request.authenticated`
   - `request.operation.(read|write)`
   - `request.param.my-param`
   - `connection.secure`

2. Other Envoy attributes

<details>
  <summary><a href="https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/advanced/attributes#wasm-attributes">Wasm attributes</a></summary>

  <table>
    <thead>
      <tr>
        <th><p>Attribute</p></th>
        <th><p>Type</p></th>
        <th><p>Description</p></th>
        <th><p>Auth</p></th>
        <th><p>RL</p></th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td><p>wasm.plugin_name</p></td>
        <td><p>String</p></td>
        <td><p>Plugin name</p></td>
        <td align="center"><p></p></td>
        <td align="center"><p>✓</p></td>
      </tr>
      <tr>
        <td><p>wasm.plugin_root_id</p></td>
        <td><p>String</p></td>
        <td><p>Plugin root ID</p></td>
        <td align="center"><p></p></td>
        <td align="center"><p>✓</p></td>
      </tr>
      <tr>
        <td><p>wasm.plugin_vm_id</p></td>
        <td><p>String</p></td>
        <td><p>Plugin VM ID</p></td>
        <td align="center"><p></p></td>
        <td align="center"><p>✓</p></td>
      </tr>
      <tr>
        <td><p>wasm.node</p></td>
        <td><p><a href="https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/base.proto#envoy-v3-api-msg-config-core-v3-node"><span>Node</span></a></p></td>
        <td><p>Local node description</p></td>
        <td align="center"><p></p></td>
        <td align="center"><p>✓</p></td>
      </tr>
      <tr>
        <td><p>wasm.cluster_name</p></td>
        <td><p>String</p></td>
        <td><p>Upstream cluster name</p></td>
        <td align="center"><p></p></td>
        <td align="center"><p>✓</p></td>
      </tr>
      <tr>
        <td><p>wasm.cluster_metadata</p></td>
        <td><p><a href="https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/base.proto#envoy-v3-api-msg-config-core-v3-metadata">Metadata</a></p></td>
        <td><p>Upstream cluster metadata</p></td>
        <td align="center"><p></p></td>
        <td align="center"><p>✓</p></td>
      </tr>
      <tr>
        <td><p>wasm.listener_direction</p></td>
        <td><p>Number</p></td>
        <td><p>Enumeration value of the <a href="https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/listener/v3/listener.proto#envoy-v3-api-field-config-listener-v3-listener-traffic-direction"><span>listener traffic direction</span></a></p></td>
        <td align="center"><p></p></td>
        <td align="center"><p>✓</p></td>
      </tr>
      <tr>
        <td><p>wasm.listener_metadata</p></td>
        <td><p><a href="https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/base.proto#envoy-v3-api-msg-config-core-v3-metadata">Metadata</a></p></td>
        <td><p>Listener metadata</p></td>
        <td align="center"><p></p></td>
        <td align="center"><p>✓</p></td>
      </tr>
      <tr>
        <td><p>wasm.route_name</p></td>
        <td><p>String</p></td>
        <td><p>Route name</p></td>
        <td align="center"><p></p></td>
        <td align="center"><p>✓</p></td>
      </tr>
      <tr>
        <td><p>wasm.route_metadata</p></td>
        <td><p><a href="https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/base.proto#envoy-v3-api-msg-config-core-v3-metadata">Metadata</a></p></td>
        <td><p>Route metadata</p></td>
        <td align="center"><p></p></td>
        <td align="center"><p>✓</p></td>
      </tr>
      <tr>
        <td><p>wasm.upstream_host_metadata</p></td>
        <td><p><a href="https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/base.proto#envoy-v3-api-msg-config-core-v3-metadata">Metadata</a></p></td>
        <td><p>Upstream host metadata</p></td>
        <td align="center"><p></p></td>
        <td align="center"><p>✓</p></td>
      </tr>
    </tbody>
  </table>
</details>

<details>
  <summary><a href="https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/advanced/attributes#configuration-attributes">Proxy configuration attributes</a></summary>

  <table>
    <thead>
      <tr>
        <th><p>Attribute</p></th>
        <th><p>Type</p></th>
        <th><p>Description</p></th>
        <th><p>Auth</p></th>
        <th><p>RL</p></th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td><p>xds.cluster_name</p></td>
        <td><p>String</p></td>
        <td><p>Upstream cluster name</p></td>
        <td align="center"><p></p></td>
        <td align="center"><p>✓</p></td>
      </tr>
      <tr>
        <td><p>xds.cluster_metadata</p></td>
        <td><p><a href="https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/base.proto#envoy-v3-api-msg-config-core-v3-metadata">Metadata</a></p></td>
        <td><p>Upstream cluster metadata</p></td>
        <td align="center"><p></p></td>
        <td align="center"><p>✓</p></td>
      </tr>
      <tr>
        <td><p>xds.route_name</p></td>
        <td><p>String</p></td>
        <td><p>Route name</p></td>
        <td align="center"><p></p></td>
        <td align="center"><p>✓</p></td>
      </tr>
      <tr>
        <td><p>xds.route_metadata</p></td>
        <td><p><a href="https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/base.proto#envoy-v3-api-msg-config-core-v3-metadata">Metadata</a></p></td>
        <td><p>Route metadata</p></td>
        <td align="center"><p></p></td>
        <td align="center"><p>✓</p></td>
      </tr>
      <tr>
        <td><p>xds.upstream_host_metadata</p></td>
        <td><p><a href="https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/base.proto#envoy-v3-api-msg-config-core-v3-metadata">Metadata</a></p></td>
        <td><p>Upstream host metadata</p></td>
        <td align="center"><p></p></td>
        <td align="center"><p>✓</p></td>
      </tr>
      <tr>
        <td><p>xds.filter_chain_name</p></td>
        <td><p>String</p></td>
        <td><p>Listener filter chain name</p></td>
        <td align="center"><p></p></td>
        <td align="center"><p>✓</p></td>
      </tr>
    </tbody>
  </table>
</details>

3. Add some support for value modifiers (functions), along the lines of Authorino's [JSON path modifiers](https://github.com/Kuadrant/authorino/blob/main/docs/features.md#string-modifiers) and/or Envoy attributes' [path expressions](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/advanced/attributes#path-expressions).
