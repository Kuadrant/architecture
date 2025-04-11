# RFC Template

- Feature Name: Resilient Deployment
- Start Date: 2025-04-07
- RFC PR: [Kuadrant/architecture#119](https://github.com/Kuadrant/architecture/pull/119)
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
Give the spec as below these are the steps that would be required.
```yaml
apiVersion: kuadrant.io/v1beta1
kind: Kuadrant
spec:
  resilience:
    authorization: True
    rateLimiting: True
    counterStorage: {} # lifts storage struct from the limitador CR.
```

## spec.resilience.authorization
For `spec.resilience.authorization` we can make use of some features within the authorino CR, but will require updating to the authorino deployment and creation of PodDiruptionBudget CR.

The authorino CR allows the setting of replicas `spec.replicas`.
This should be set to 2. 
The user should have the power to modify this number to what they want.
If the number is less than the minimum we recommend (2), the kuadrant CR should report an error in the status.
If the user uses a number greater than what we recommend, the kuadrant CR should report an information status.
```yaml
kind: Authorino
metadata:
  name: authorino
spec:
  replicas: 2
```

The deployment for authorino needs to be modified in the following was.
`resources.requests` and `topologySpreadConstraints` need to be added.
The kuadrant status is easier to manage with the `resources.requests` as it can be less than what is recommended.
Values for the `resources.requests` need to be discovered. 
The `topologySpreadConstraints` on the other hand is not as easy to state if it is out of spec in a good way (higher spec) or a bad way (lesser spec).
Still the status in the kuadrant CR needs to reflect these resources being out of spec.
```yaml
kind: Deployment
metadata:
  name: authorino
spec:
  template:
    spec:
      containers:
        - name: authorino
          resources:
            requests:
              cpu: 10m
              memory: 10Mi
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              authorino-resource: authorino
        - maxSkew: 1
          topologyKey: kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              authorino-resource: authorino
```

The PodDisruptionBudget is the next resource that needs to be created.
As this is a resource that is none existent in a normal deployment of kuadrant.
The kuadrant-operator will be required to take ownership of the resource.
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: authorino
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      authorino-resource: authorino
```


## spec.resilience.rateLimiting
For `spec.resilience.rateLimiting` we can make use of most features within the limitador CR, but will be requiring modifying the limitador deployment for the `topologySpreadConstraints`.

The limitador CR allows the setting of the `pdb`(PodDisruptionBudget), `resourceRequirements.requests`, and `replicas`.
These resources follow the same user updating feature.
When the feature is active the status of each section should be reported to when out of spec.
```yaml
apiVersion: limitador.kuadrant.io/v1alpha1
kind: Limitador
metadata:
  name: limitador
spec:
  pdb:
    maxUnavailable: 1
  replicas: 2
  resourceRequirements:
    requests:
      cpu: 10m
      memory: 10Mi
```

The `topologySpreadCondtraints` needs to be configured with in the limitador deployment CR.
```yaml
# patches (merge) the limitador-operator owned Deployment with this partial resource
apiVersion: apps/v1
kind: Deployment
metadata:
  name: limitador-limitador
spec:
  template:
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              limitador-resource: limitador
        - maxSkew: 1
          topologyKey: kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              limitador-resource: limitador
```

Limitador does require persisted storage.
This can be configured in the limitador CR under `spec.storage`.
If this section is missing from the limitador CR, and the `spec.resilience.rateLimiting`, the kuadrant CR needs to raise a warning in the status.
The status is only a warning as it could be possible the user wants in memory counters.

## spec.resilience.counterStorage
For `spec.resilience.counterStorage` the configuration will be added to the limitador CR under the `spec.storage` section.

In the kuadrant CR this will be an object that matches the structure of the limitador [spec.storage](https://github.com/Kuadrant/limitador-operator/blob/626341d2aff5f6b8028317dc0e7d1bb27eb8d3d4/api/v1alpha1/limitador_types.go#L203-L212).
Unlike the other resilience features, once configured the kuadrant-operator need to maintain the configuration within the limitador CR. 

This will introduce an upgrade issue.
If the user has configured the storage option with in the limitador CR prior to this feature added, their configuration will be removed.

## Kuadrant CR Status
The kuadrant CR Status block will be used to tell the user the state of the different features that have being enabled.
Each feature can raise warnings and errors independent of each other.

## doc.kuadrant.io guides
On [docs.kuadrant.io](https://docs.kuadrant.io) there will be a section that describes the features of `spec.resilience`.
This will show examples of the configuration that each feature will configure, explain how to modify the configuration.

What happens when the feature is disabled will be outlined, so the user can make an informed decision when disabling the feature.

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

There has being some exploring work done, and guides to show a user how to set this up.
- [github.com/kuadrant/deployment](https://github.com/kuadrant/deployment)
- [Resilient Deployment of data plane compnents (docs guide)](https://docs.kuadrant.io/dev/install-olm/#resilient-deployment-of-data-plane-components)

There are a number of other RFCs releating to the intruduction of features into the kuadrant CR.
- [RFC: Observability API](https://github.com/Kuadrant/architecture/pull/97)
- [RFC for mTLS](https://github.com/Kuadrant/architecture/pull/110)
- [RFC: Standardize the Kuadrant Spec](https://github.com/Kuadrant/architecture/pull/112)

# Unresolved questions
[unresolved-questions]: #unresolved-questions

<!-- - What parts of the design do you expect to resolve through the RFC process before this gets merged? -->
<!-- - What parts of the design do you expect to resolve through the implementation of this feature before stabilization? -->
<!-- - What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC? -->

Should resilience be enabled by default? 
While the simple answer would be yes, there is the issue of a configuration being required for counter persistence's.

While the kuadrant-operator will not take ownership of existing configuration.
What the sub operators would do is unknown at this stage if the feature is enabled via their CRs.

How to many counterStorage configuration on upgrades? More so how to many existing storage configurations when this feature is introduced?

# Future possibilities
[future-possibilities]: #future-possibilities

<!-- Think about what the natural extension and evolution of your proposal would be and how it would affect the platform and project as a whole. Try to use this section as a tool to further consider all possible interactions with the project and its components in your proposal. Also consider how this all fits into the roadmap for the project and of the relevant sub-team. -->
<!---->
<!-- This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but otherwise related. -->
<!---->
<!-- Note that having something written down in the future-possibilities section is not a reason to accept the current or a future RFC; such notes should be in the section on motivation or rationale in this or subsequent RFCs. The section merely provides additional information. -->
