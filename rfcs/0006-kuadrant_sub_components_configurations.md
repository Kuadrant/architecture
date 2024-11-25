# Configuration of Kuadrant Sub Components

- Feature Name: `sub-components-config`
- Start Date: 2023-09-11
- RFC PR: [Kuadrant/architecture#25](https://github.com/Kuadrant/architecture/pull/25)
- Issue tracking: [Kuadrant/kuadrant-operator#163](https://github.com/Kuadrant/kuadrant-operator/issues/163)
- Status: Withdrawn

# Summary
[summary]: #summary

Enable configuration of sub components of Kuadrant from a centralized location, namely the Kuadrant CR. 

# Motivation
[motivation]: #motivation

The initial request comes from MGC to configure Redis for Limitador by the following issue [#163](https://github.com/Kuadrant/kuadrant-operator/issues/163).
MGC's current work around is to update the Limitador CR after the deployment with the configuration setting for Redis Instance.
This change would allow for the configuration of sub components before the Kuadrant is deployed. 

This reduces the number of CRs that users of Kuadrant are required to modify to get the installation they require.
The sub components CRs (Authorino, Limitador) never have to be modified by a Kuadrant user (and should never be modified by a Kuadrant User).

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

As the Kuadrant operator would be responsible for reconciling these configurations into the requested components, restrictions and limitations can be placed on the components which maybe allowed in a standalone installation.
An example in this space is the disk storage for Limitador which is a new feature and the Kuadrant installation may not want to support it till there is a proven track record for the feature.

For existing Kuadrant Users this may be a possible breaking changes if those users manually configure the Kuadrant sub components via their CRs.
A guide can be created to help migrate the users configurations to the Kuadrant CR.
This guide can be part of the release notes and/or possibly released before the release of Kuadrant.

The deployment configuration for each component can be placed in the Kuadrant CR.
These configurations are then reconciled into the CRs for each component. 
Only the options below are exposed in the Kuadrant CR. 
All fields in the spec are optional.

``` yaml
apiVersion: kuadrant.io/v1beta1
kind: Kuadrant
metadata:
    name: kuadrant-sample
spec:
    limitador:
        afffinity: ...
        listener: ...
        pdb: ...
        replicas: ...
        resourceRequirements: ...
        storage: ...
    authorino:
        evaluatorCacheSize: ...
        healthz: ...
        listener: ...
        logLevel: ...
        metrics: ...
        oidcServer: ...
        replicas: ...
        tracing: ...
        volumes: ...
status:
    ...
```

The Kuadrant operator will watch for changes in the Authorino and Limitador CRs, reconciling back any changes that a user may do to these configurations.
How ever Kuadrant operator will not reconcile fields that are given above. 
An example of this is the `image` field on the Authorino CR.
This field allows a user to set the image that Authorino is deployed with.
The feature is meant for dev and testing purposes. 
If a user wishes to use a different image, they can.
Kuadrant assumes they know what they are doing but requires the user to set the change on the component directly.

Only the sub component operator will be responsible for actioning the configurations pasted from the Kuadrant CR to the sub components CR.
This ensure no extra changes will be required in the sub operators to meet the needs of Kuadrant.

Status errors related to the configuration of the sub components should be reported back to the Kuadrant CR.
The errors messages in Kuadrant state what components are currently having issue and which resource to review for more details.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

All the fields in the Authorino and Limitador CRs that are configurable in the Kuadrant CR are optional and have sound defaults.
Kuadrant needs to remain installable with out having to set any spec in the Kuadrant CR.

The Kuadrant operator should only reconcile the spec that is given. 
This would mean if the user states the number of replicas to be used in one of the components only the replica field for that component should be reconciled.
As the other fields would be blank at this stage, blank fields would not be reconciled to the component CR.
By this behaviour a few things are being achieved.
Component controllers define the defaults to be used in the components.
Optional fields in the component CRs never get set with blank values.
Blank values in the component CR could override the defaults of the components causing unexpected behaviour.
Existing Kuadrant users may already have custom fields set in the component CRs.
By only reconciling the fields set in the kuadrant CR this allows time for a user to migrate their custom configuration from the component CR to the Kuadrant CR.

## Fields to reconcile
Fields being reconcile can be classified into different groups.
These classifications are based around the tasks a user is achieve.

- Kubernetes native, setting that affect how Kubernetes handles the resource.
- Observability, configuration settings that allow insights into how the applications are operation.
This can be Kubernetes native or external tooling.
- Application Settings, setting targeting the application and how it connects to external services.

### Authorino Spec
#### Kubernetes native
- replicas

#### Observability
- healthz
- logLevel
- metrics
- tracing

#### Application Settings
- evaluatorCacheSize
- listener
- oidcServer
- volumes

### Limitador Spec
#### Kubernetes native
- afffinity
- pdb
- replicas
- resourceRequirements

#### Application Settings
- listener
- storage

## Fields not reconciled
There are a number of fields in both Authorino and Limitador that are not reconciled.
Reasons for doing this are:

It is better to start with a sub set of features and expand to include more at a later date.
Removing feature support is far harder than adding it.

There are four classifications the unreconciled fields fail into.
- Deprecated, fields that are deprecated and/or have plans to be removed from the spec in the future.
- Unsupported, the features would have hard coded or expected defaults in the Kuadrant operator. 
Work would be required to all the custom configurations options.
- Dev/Testing focused, features that should only be used during development & testing and not recommended for production.
The defaults would for the fields are the production recommendations.
- Reconciled by others, this mostly affects Limitador as the deployment configuration and runtime configuration are in the same CR.
In the case of Kuadrant the runtime configuration for Limitador is added via the RateLimitingPolicy CR.

### Authorino Spec

#### Unsupported
- clusterWide
- authConfigLabelSelectors
- secretLabelSelectors

#### Dev/Testing focused
- image
- imagePullPolicy
- logMode

### Limitador Spec

#### Unsupported
- RateLimitHeaders

#### Reconciled by others
- Limits

#### Deprecated
- version

# Drawbacks
[drawbacks]: #drawbacks

As the Kuadrant CR spec will be a sub set of the features that can be configured in the sub components spec, extra maintenances will be required to ensure specs are in sync.

New features of a component will not be accessible in Kuadrant initially.
This is both a pro and a con.

Documentation becomes harder, as the sub component should be documenting their own features but in Kuadrant the user does not configure the feature in sub component.
This has the risk of confusing new users.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

One alternative that was being looked at was allowing the user to bring their own Limitador instances by stating which Limitador CR Kuadrant should use. 
A major point of issue with this approach was knowing what limits the user had configured and what limits Kuadrant configured. 
Sharing global counters is a valid reason to want to share Limitador instances. 
How ever it this case Limitador would not be using one replica and therefore would have a back-end storage configured.
It is the back-end storage that needs to be shared across instances.
This can be done with adding the configuration in the Kuadrant CR.

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

-----
- Is there a need to add validation on the configuration?
- If a valid configuration is add to the Kuadrant CR and this configuration is pass to the sub components CR but there is a error trying to setting up the configuration.
How is this error reported back to the user? 
An example of this is configuring Redis as the back-end in Limitador, this requires stating the name and namespace of a configmap.
The Limitador CR will have an error if the configmap does not exist and as the user only configures the Kuadrant CR this error may go unnoticed.
This is only one example but there is a need for good error reporting back to the user, where they would expect to see the error.


# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would be and how it would affect the platform and project as a whole. Try to use this section as a tool to further consider all possible interactions with the project and its components in your proposal. Also consider how this all fits into the roadmap for the project and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but otherwise related.

Note that having something written down in the future-possibilities section is not a reason to accept the current or a future RFC; such notes should be in the section on motivation or rationale in this or subsequent RFCs. The section merely provides additional information.

-----
The implementation stated here allows the user to state spec fields in the component CRs or the Kuadrant CR (Kuadrant CR overrides the component CRs).
A future possibility would be to warn the user if they add configuration to the components CRs that would get overridden if the same spec fields are configured in the Kuadrant CR.

