# `OpenIDConnectPolicy` - a metapolicy

- Feature Name: oidc-policy
- Start Date: 2025-02-05
- RFC PR: [Kuadrant/architecture#0114](https://github.com/Kuadrant/architecture/pull/114)
- Issue tracking: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/issues/0000)

# Summary
[summary]: #summary

The `OpenIDConnectPolicy` aims at helping users to easily & quickly get started with the Kuadrant stack to setup an
[OpenID Connect flow](https://openid.net/developers/how-connect-works/) without needing to initially understand how to
use Kuadrant's `AuthPolicy` to achieve their goals. 

# Motivation
[motivation]: #motivation

There are a few goals this implementation hopes to achieve and set a precedent for introducing more such APIs in
Kuadrant:

 - users looking at an example policy would be able to make mostly sense of it without needing to lookup documentation;
 - the policy would apply sensible defaults in most cases and only require the very minimal input from the user;
 - the idea is to implement it as _metapolicy_, which would only be consumed by a controller to produce a `AuthPolicy`
   that would then enforce the flow;
 - it is _not_ the goal to expose all the power of the `AuthPolicy` to the user through the new policy, but rather to let
   users achieve most of their goals by slowly exploring defaults of the policy and eventually dig down the stack, into
   the `AuthPolicy` to expand their use cases when they need it;
 - explore how to, over time, grow the initial minimal API of the policy into a larger one to cover more use cases than
   the one currently set as the goal for this RFC

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The `OpenIDConnectPolicy` is a direct policy attachment that attaches to one or multiple
[`HTTPRoute`](https://gateway-api.sigs.k8s.io/api-types/httproute/) objects. Targeted routes will require requests to
carry an access token to be authorized through. Should the token be missing or be invalid, the requesting entity gets
redirected to the `provider`'s `authorizationEndpoint` specified in the policy for them to identify.

It is important to note here that the access token is to be negotiated stored by the client. While this may change in
the future, currently the token is to either be provided by the appropriate HTTP header (`Authorization: Bearer`, which 
is the default source), or by a Cookie.

## Minimal example

Using a very minimal example, we'll use
[gitlab.com](https://docs.gitlab.com/ee/integration/openid_connect_provider.html) as our provider for an OpenID Connect
flow to protect all requests routed to our application by the `HTTPRoute`.

```yaml
apiVersion: kuadrant.io/v1alpha1
kind: OpenIDConnectPolicy
metadata:
  name: gitlab
spec:
  targetRef:
    name: toystore
    group: gateway.networking.k8s.io
    kind: HTTPRoute
  provider:
    issuer: https://gitlab.com
    credentials:
      clientID: 35001bfef37042bf2fb125e9e8f99f0c719231632ab62a18cbf5220c3d1f8f10
```

First we need to provide the `issuer` of our `provider`, in this case `https://gitlab.com`. Kuadrant will default 
to [querying the provider for the additional metadata](https://datatracker.ietf.org/doc/html/rfc8414) it requires
from [gitlab's endpoint](https://gitlab.com/.well-known/openid-configuration), initially the `authorizationEndpoint` to which 
to redirect unauthorized requests to.

If the request for the additional metadata fails, all access to the protected resources will be `Unauthorized`. All the
information from the endpoint can be manually specified under `provider` to mitigate these failures. These values would
also override any values successfully obtained from querying the provider's endpoint.

Next, we'll have to identify our request with the provider using the `clientID` that were
provided to us by, in this example, gitlab. You'd probably rather store these in a `credentialsRef` instead as we did
here. A `credentialsRef` would point to a Kubernetes'
[`Secret`](https://kubernetes.io/docs/concepts/configuration/secret/) with the required information. And that's all you
should need to get started with to use an OpenID Connect provider to authorize access to your application. 

## The complete `OpenIDConnectPolicy`

Here is an example of the complete proposal for configuring an authorization flow using OpenID Connect:

```yaml
apiVersion: kuadrant.io/v1alpha1
kind: OpenIDConnectPolicy
metadata:
  name: gitlab
spec:
  targetRef:
    name: toystore
    group: gateway.networking.k8s.io
    kind: HTTPRoute
  tokenSource: authorizationHeader
  provider:
    issuer: https://gitlab.com
    discoveryEndpoint: https://gitlab.com/.well-known/openid-configuration
    authorizationEndpoint: https://gitlab.com/oauth/authorize
    authorizationEndpointPayload:
      query:
        client_id: provider.credentials.clientID
        redirect_uri: route.gateway.listeners[0].hostname + "/oauth/callback"
        scope: |- 
          "openid"
        response_type: |-
          "code"
    tokenEndpoint: https://gitlab.com/oauth/token
    introspectionEndpoint: https://gitlab.com/oauth/introspect
    jwksUri: https://gitlab.com/oauth/discovery/keys
    credentials:
      clientID: 35001bfef37042bf2fb125e9e8f99f0c719231632ab62a18cbf5220c3d1f8f10
      clientSecret: gloas-35001bfef37042bf2fb125e9e8f99f0c719231632ab62a18cbf5220c3d1f8f10
    redirectUri: 
      host: example.org
      path: /oauth/callback
```

Some of these were inferred in the simple example above, while others are implied and some not even necessary for the
basic case. `tokenEndpoint` and `introspectionEndpoint` are both unnecessary for this flow to succeed. That's because we
use the `authorizationHeader` as the source for our access token. In this flow, we let the client deal with getting the
token from the identity provider, i.e. gitlab, as well as storing it and sending it along each request to our
application.

While we do need the `authorizationEndpoint` to redirect unauthorized requests to the identity provider, as well as the
`jwksUri` to validate the token when one is present, these can be inferred from the `issuer` by querying the
`discoveryEndpoint` from the provider, which itself is inferred from the `issuer`.

Kuadrant will also automatically append a query string to the `authorizationEndpoint` when forwarding a user. That will
contain data inferred from the configuration itself, as well as some defaults:

 - `client_id`: from the `provider`'s `credential` section
 - `redirect_uri`: composed of `host`, inferred if there is a single listener for the `Gateway` the targeted `HTTPRoute`
   is pointed to; and the `path`, which defaults to `/oauth/callback`
 - `scope`: defaults to `openid`
 - `response_type`: defaulting to `code`

 All of which can be overridden by explicitly configuring the `authorizationEndpointPayload` explicitly. Any entry can
 be added, the value needs to be a valid CEL expression and will be encoded depending on the how the payload is to be
 sent. Here `query`, each key/value pair will be URL encoded and appended to the `authorizationEndpoint` when
 redirecting. 

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### WIP 

Some of this I think I have sorted out in a proof of concept... But I'm not completely done yet.
I'm trying to see if I can get away with: 
 - Having the client completely deal with `401 Unauthorized` and have it initiate the flow;
 - Having the client intercept the redirect when being redirected from the Identity Provider, rather than needing to
   have the `AuthConfig` deal with it so that the flow is completely handled by the client.

I'm doing this by using the information provided here and using it inside the `AuthPolicy` directly for now. All that
then remains is writing the mapper controller for "translating" the `OpenIDConnectPolicy` into an `AuthPolicy`.

# Drawbacks
[drawbacks]: #drawbacks


# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

 - This iteration is fairly low risk. It tries to make this all work with multiple OIDC providers, starting with Gitlab.
 - Today, no changes are required to `AuthorizationPolicy` or `Authorino` nor any other component in Kuadrant, this only 
   does some to the heavy lifting for the users. 
 - The API leaves some space to grow it more complex use cases as we evolve this to support possibly more advanced
   use cases.

# Prior art
[prior-art]: #prior-art

 - I'm looking into some OIDC client & server side libraries to see how to best shape the API.

# Unresolved questions
[unresolved-questions]: #unresolved-questions


# Future possibilities
[future-possibilities]: #future-possibilities

 - Add support for letting the upstream deal with the token exchange?
 - Add support for letting Kuadrant (i.e. Authorino?) exchange the token with the Identity Provider and store it?
 - Investigate how to expand on the authorization aspect: either by composing with another resource, expanding that one,
   leveraging defaults & overrides, have the user fix it a the `AuthPolicy` layer themselves, ... ?

