# An Open-Architecture for Kuadrant 

- Feature Name: open-arch
- Start Date: 2025-02-26
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/issues/0000)

# Summary
[summary]: #summary

Kuadrant has been so far mostly focus on specific areas: AuthN/Z, DNS, TLS and Rate Limiting. Most of these areas
evolved in silos, independently from each other. There were some effort to either standardize certain aspects and some
of these features work somewhat in tandem, in the form of authenticated rate limiting, which somewhat integrates two of
the areas Kuadrant concerns itself with.

The intent of this proposal is to formalize Kuadrant as a _platform_ to extend [Gateway API][1] functionality through the use
of [Policy Attachment][2]. 

# Motivation
[motivation]: #motivation

A few components emerged overtime that play a central role in _how_ functionality is exposed in Kuadrant:

- Policies themselves;
- [CEL][3] & Well-Known Attributes;
- The [DAG][4] representing the current state of the cluster regarding Gateway API objects, as well as the policies
  attached to them;
- The [wasm-shim][https://github.com/Kuadrant/wasm-shim/?tab=readme-ov-file#sample-configuration] through its
  configuration, enabling Kuadrant's own "filter chain".

The idea would be to expand on these components and formalizing their interfaces to their respective consumers. While
these are solid building blocks, as proven by our current experience with those to build existing features, we would
need to open Kuadrant up to make use of these to support a modular model for policies.

Initially, users would only be exposed to these changes through _metapolicies_, domain specific policies that expose a
higher-level abstraction to our existing policies.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Implementing a custom metapolicy


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Plugin architecture

### gRPC to controller

- Unix socket for "in-pod/embedded" plugins

### gRPC streams for eventing on the DAG

- subscriptions infered from CEL


# Drawbacks
[drawbacks]: #drawbacks


# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

# Prior art
[prior-art]: #prior-art


# Unresolved questions
[unresolved-questions]: #unresolved-questions


# Future possibilities
[future-possibilities]: #future-possibilities


[1]: https://gateway-api.sigs.k8s.io/
[2]: https://gateway-api.sigs.k8s.io/geps/gep-2649/
[3]: https://cel.dev
[4]: https://github.com/Kuadrant/policy-machinery
