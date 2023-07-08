# RFC Template

- Feature Name: envoy-gw
- Start Date: 2023-07-07
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/issues/0000)

# Summary
[summary]: #summary

Provide (different?) deployment for Kuadrant, either with a default Gateway API provider or a pluggeable
backend. The proposal is to provide users a battery included experience, when getting started with
Kuadrant while not yet having a (supported) Gateway API provider installed on their cluster.

# Motivation
[motivation]: #motivation

Currently, Kuadrant comes with no batteries included, i.e. no Gateway API provider. This leaves it up to
the user to get it installed on their cluster (at least until the API becomes widely adopted and maybe is
present by default). This creates unnecessary friction for people wanting to reap the benefits provided
by the Kuadrant APIs. This proposal aims at addressing that unnecessary friction. The proposal is about
lowering the bar to entry for users to test drive and eventually deploy Kuadrant to their new or existing
clusters…

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

If a user, which is probably most users today, gets started with the Gateway API because of Kuadrant,
they would merely install a Kuadrant Operator that makes sure of installing all dependencies (either
because of the operator they install, the `Kuadrant` CR they deploy or sane defaults the operator would
apply), including the Gateway API and its provider. That provider would be _Envoy Gateway_, as it is
engineered to address this very usecase. 

For users that already may have a provider installed, e.g. Istio or OSSM, Kuadrant is able to plug itself
in seamlessly in such environments. Simply deploying the Operator as you'd do currently, would result in
the Kuadrant taking the necessary steps to wire its funcitonality within the existing deployments.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Doing so entails a few things: 

 - [ ] Get an _Envoy Gateway_ distribution ready to back Kuadrant use cases
   - including _howtos_ on how to "chain" existing ingress'es with gateway resources
 - [ ] Abstract the Gateway Provider behind an [Service Provider interface (SPI)](), to support at least Envoy GW, Istio & OSSM.
   - which makes for a clear contract any other Gateway API provider would be able to implement
 - [ ] Package different distributions (at the very least GW API included, or pluggeable) as different Operators/images?

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- How error would be reported to the users.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- Does another project have a similar feature?
- What can be learned from it? What's good? What's less optimal?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other tentatives - successful or not, provide readers of your RFC with a fuller picture.

Note that while precedent set by other projects is some motivation, it does not on its own motivate an RFC.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would be and how it would affect the platform and project as a whole. Try to use this section as a tool to further consider all possible interactions with the project and its components in your proposal. Also consider how this all fits into the roadmap for the project and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but otherwise related.

Note that having something written down in the future-possibilities section is not a reason to accept the current or a future RFC; such notes should be in the section on motivation or rationale in this or subsequent RFCs. The section merely provides additional information.