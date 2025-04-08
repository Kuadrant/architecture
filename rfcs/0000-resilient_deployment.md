# RFC Template

- Feature Name: Resilient Deployment
- Start Date: 2025-04-07
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [Kuadrant/architecture#117](https://github.com/Kuadrant/architecture/issues/117)

# Summary
[summary]: #summary

<!-- TODO: One paragraph explanation of the feature. -->

# Motivation
[motivation]: #motivation

<!-- Why are we doing this? What use cases does it support? What is the expected outcome? -->
As our user move to deploying kuadrant into production environments there is a want to create more resilient deployments.
This includes deploying multiply replicas of the data plane components, Authorino & Limitador.
In the case of Limitador, this also includes being able to persist counters.

As Kuadrant we want to provide a user experience that makes deploying a more resilient product possible.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

<!-- Explain the proposal as if it was implemented and you were teaching it to Kuadrant user. That generally means: -->
<!---->
<!-- - Introducing new named concepts. -->
<!-- - Explaining the feature largely in terms of examples. -->
<!-- - Explaining how a user should *think* about the feature, and how it would impact the way they already use Kuadrant. It should explain the impact as concretely as possible. -->
<!-- - If applicable, provide sample error messages, deprecation warnings, or migration guidance. -->
<!-- - If applicable, describe the differences between teaching this to existing and new Kuadrant users. -->

At the core, there are two different areas of focus required here.
There is the resilience of the deployments, Authorino & Limitador, and there is the resilience of the counters used within Limitador.
These two areas address the resiliency as whole.

For the deployments there are a number of configurations required.
- TopologyConstraints
- PodDisruptionBudget
- Resource Limits
- Replicas
With Authoring and Limitador there are different ways of configuring some of the features.

The counters in Limitador can be persisted with external storage.
- [Counter storages](https://github.com/Kuadrant/limitador/blob/main/doc/server/configuration.md#counter-storages)
- [Limitador API](https://github.com/Kuadrant/limitador-operator/blob/main/api/v1alpha1/limitador_types.go#L203-L211)

Configuration of the feature will be done in the Kuadrant CR.
This is to make it simpler for the user.
**API Design (WIP)**
```yaml
apiVersion: kuadrant.io/v1beta1
kind: Kuadrant
spec:
  resilience:
    authorization: True
    rateLimiting: True
    counterStorage: {} # lifts storage struct from the limitador CR.
```

Equally important, if not more important will be relaying information back to the user on the status of the configuration.
What the status block looks like is unknown currently.
<!-- TODO: more understanding is required here -->

## Behavior
It is important that we understand what the behavior of this feature should be, and convey that behavior to the user clearly (documentation).
There are a number of uses that needs to be covered in the first iteration of this feature.
- New installations
- Existing installations with zero custom configuration
- Existing installations with custom configuration
- Removal of configuration
- SRE reaction events
- The user wants more

### New installations
This is by far the simplest configuration.
By default, the all resilience configuration will be blank.
Thus, the feature will be disabled.
One reason for this choice is area the counter storage requiring configuration.

### Existing installations with zero custom configuration
This act very much like a new installation. 
The resilience configuration will be blank, thus, the feature is disabled.

Once the feature is enabled the all the relevant resources will be created, with the kuadrant-operator having ownership where possible.

### Existing installations with custom configuration
In the case the installation has custom configuration nothing changes till the users defines the resilience feature in the Kuadrant CR.

Once the user configures the resilience feature the kuadrant-operator will create any missing configurations, and take ownership of those configurations.
The kuadrant-operator will not modify, extend, update, or take ownership of any existing configuration[^1].
However, the status will reflect there is configuration that the kuadrant-operator doesn't owner, or there is configuration outside what the kuadrant-operator expects.

[^1]: This true for any resource that is not already managed by the kuadrant-operator. 

### Removal of configuration.
In this case the user has configured the resilience features in the past, and is turning them off.
Where the features are turned off, the kuadrant-operator will clean resources it owns.
Even if the user has modified that configuration.

This needs to be called out very clearly in the documentation.
Why we do this is to simplify the reconciliation of configuration.

### SRE reaction events
From time to time SRE teams would need to change configurations for other to address issues.
We should allow the user configure the deployment to a state that is we regard as not resilient.
The one example that comes to mind is scaling replicas to zero for address some issue.
The user should be allowed to that, but the Kuadrant CR should state the configuration is below expected spec.
The below spec would be regarded as an error.

### The user wants more
The user knows more about their deployment than we ever can.
We need to allow the user to modify the configuration to suit their needs.
When a user modifies the configuration, the kuadrant-operator should not reconcile back the changes.

If the user changes are below the minimum spec that we state, this should be reflected in the status.
If the user changes the configuration at all it should be reflected in the status. 
However, when the spec is more than what we regard as minimum the status should be a highlight not warning or error.
How this is reflected in the status of the Kuadrant CR is unknown currently.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

<!-- This is the technical portion of the RFC. Explain the design in sufficient detail that: -->
<!---->
<!-- - Its interaction with other features is clear. -->
<!-- - It is reasonably clear how the feature would be implemented. -->
<!-- - How error would be reported to the users. -->
<!-- - Corner cases are dissected by example. -->
<!---->
<!-- The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work. -->

# Drawbacks
[drawbacks]: #drawbacks

<!-- Why should we *not* do this? -->

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

<!-- - Why is this design the best in the space of possible designs? -->
<!-- - What other designs have been considered and what is the rationale for not choosing them? -->
<!-- - What is the impact of not doing this? -->

# Prior art
[prior-art]: #prior-art

<!-- Discuss prior art, both the good and the bad, in relation to this proposal. -->
<!-- A few examples of what this can include are: -->
<!---->
<!-- - Does another project have a similar feature? -->
<!-- - What can be learned from it? What's good? What's less optimal? -->
<!-- - Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background. -->
<!---->
<!-- This section is intended to encourage you as an author to think about the lessons from other tentatives - successful or not, provide readers of your RFC with a fuller picture. -->
<!---->
<!-- Note that while precedent set by other projects is some motivation, it does not on its own motivate an RFC. -->

# Unresolved questions
[unresolved-questions]: #unresolved-questions

<!-- - What parts of the design do you expect to resolve through the RFC process before this gets merged? -->
<!-- - What parts of the design do you expect to resolve through the implementation of this feature before stabilization? -->
<!-- - What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC? -->

Should resilience be enabled by default? 
While the simple answer would be yes, there is the issue of a configuration being required for counter persistence's.

While the kuadrant-operator will not take ownership of existing configuration.
What the sub operators would do is unknown at this stage if the feature is enabled via their CRs.

# Future possibilities
[future-possibilities]: #future-possibilities

<!-- Think about what the natural extension and evolution of your proposal would be and how it would affect the platform and project as a whole. Try to use this section as a tool to further consider all possible interactions with the project and its components in your proposal. Also consider how this all fits into the roadmap for the project and of the relevant sub-team. -->
<!---->
<!-- This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but otherwise related. -->
<!---->
<!-- Note that having something written down in the future-possibilities section is not a reason to accept the current or a future RFC; such notes should be in the section on motivation or rationale in this or subsequent RFCs. The section merely provides additional information. -->
