# Kuadrant Proposal - API

Kuadrant aims to expose a simple API, yet flexible enough to accomodate API Management requirements.

## Problem Statement

Currently, the [latest](https://github.com/Kuadrant/kuadrant-controller/releases/tag/v0.2.0) release of 
kuadrant API is based on [apiproducts.v1beta1.networking.kuadrant.io](https://github.com/Kuadrant/kuadrant-controller/blob/v0.2.0/doc/architecture.md#api-product-crd)
and [apis.v1beta1.networking.kuadrant.io](https://github.com/Kuadrant/kuadrant-controller/blob/v0.2.0/doc/architecture.md#api-crd) custom kubernetes resource APIs.
Together with these CRDs, there is a bunch of kuadrant service annotations and labels available in order to
provide the kuadrant [service discovery](https://github.com/Kuadrant/kuadrant-controller/blob/v0.2.0/doc/service-discovery.md) feature. 
All together, kuadrant provides routing, basic and global rate limiting and, finally, global authN (not authZ) features
to protect customer owned APIs. While, good enough for a wide set of use cases, there are several 
limitations:

* Rate limit is enforced to all services and endpoints defined in a APIProduct. It cannot be specified route/path wise.
* Only high level rate limit directives are defined (`global`, `perRemoteIP` and `authenticated`). There is no option to specify with a fine-grained rate limit API.
* Authentication is limited to `API key` and `OpenID Connect`.
* Authentication is enforced to all services and endpoints defined in a APIProduct. It cannot be specified route/path wise.
* Authorization is not available.

## High Level Goals

## Proposed Solution
