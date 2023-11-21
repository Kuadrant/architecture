# Kuadrant CI/CD release process

- Feature Name: `kuadrant-ci-cd`
- Start Date: 2023-11-21
- RFC PR: [Kuadrant/architecture#0038](https://github.com/Kuadrant/architecture/pull/38)
- Issue tracking: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/issues/0000)

# Summary
[summary]: #summary

This RFC not only proposes a new release process for Kuadrant components, but also a new and more agile way of getting
the desired "artifacts" (images, manifests, etc.) into the hands of the users, devs, QE team and any other process that
needs them.

# Motivation
[motivation]: #motivation

The current process is not only slow, but also involves manual steps that are prone to human error. Its implementation
is also not very flexible, and it's not easy to add new steps or change the existing ones without replicating the same
code in different repositories. It also involves a convolution of different tools and services that are not easy to
maintain and that are not very well integrated with each other.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Explain the proposal as if it was implemented and you were teaching it to Kuadrant user. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how a user should *think* about the feature, and how it would impact the way they already use Kuadrant. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing and new Kuadrant users.

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