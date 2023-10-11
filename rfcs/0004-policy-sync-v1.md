# RFC - Policy Sync

- Feature Name: `policy_sync_v1`
- Start Date: 2023-10-10
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [https://github.com/Kuadrant/architecture/issues/26](https://github.com/Kuadrant/architecture/issues/26)

# Summary
[summary]: #summary

Enable policies defined in the control plane (hub) to be synced to the data plane (spoke clusters).

# Motivation
[motivation]: #motivation

Currently any policies targeting gateways in the spokes need to be defined in the spokes, and it can be cumbersome, time-consuming and error prone to require these to be duplicated across multiple spoke clusters.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Explain the proposal as if it was implemented and you were teaching it to Kuadrant user. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how a user should *think* about the feature, and how it would impact the way they already use Kuadrant. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing and new Kuadrant users.

The policy sync system will allow a gateway-admin to create or modify a gateway class at the hub level and specify a series of GVKs for that gatewayClass (for example the AuthPolicy GVK).

When a gateway is created in the hub that uses this gatewayClass, any AuthPolicies that target that gateway will be watched by the MGC.

When these gateways are placed on a spoke, any AuthPolicies targeting that gateway will also be placed on that same spoke.

When the AuthPolicies are placed on the relevant spokes, they will be manipulated to target the new gateway in the spoke.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

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

#### We will not be using the policy framework to complete this objective:
The policy framework is an improvement on the system currently used to place resources on spoke clusters from the hub, essentially removing the need to read the placement decision and manually create manifestworks.

As such, we might want to eventually look at migrating to using the system for all hub > spoke placements.

However, the policy framework currently has a significant limitation preventing our usecase, which is that we cannot read the status back from the resources created in the spoke.

The current proposal, regarding this work, is to proceed with our current manifestwork solution, and migrate all hub > spoke placement logic if the policy framework reaches a point that it is more feasible.

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