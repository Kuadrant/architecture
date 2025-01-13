# Standardize the Kuadrant Spec

- Feature Name: Standardize the Kuadrant Spec
- Start Date: 10/01/2025
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/issues/0000)

# Summary
[summary]: #summary

This RFC is trying to describe a common API design that can be used when adding feature to Kuadrant that are configurable thought the Kuadrant CR.

# Motivation
[motivation]: #motivation

As we are move forward with the development of Kuadrant, we are wanting to add new features to the product.
Some of these features will be configurable by the user, the Kuadrant CR is an expected place for this to happen. 
Before starting to add these configurable features, a standardize way of defining features in the Kuadrant CR should be agreed upon. 
This RFC lays the groundwork for an API design which the new features will follow.
By having a standardized API design, future development can move faster as the API design will have the groundwork already done.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The core of the design is made up of four elements. 
`spec.feature_a` is the name of the feature that is being configured in that section.
`spec.feature_a.configure` is a boolean field which tells the kuadrant operator to use the configuration defined in the Kuadrant CR for the feature. 
This is done to allow us the flexibility to have features active by default.
`spec.feature_a.enable` is a list of strings that represent the components the configuration will be applied too.
Having it a list of strings allows us to extend or reduce the number of accepted values without needing to modify the CRD. 
With the use of strings as values means keywords can also be defined, for example `"default"` could be shorthand for a sub set of components affected by the feature.
`spec.feature_a.disable` is a list of strings that repenting the components that the configuration will *not* be applied too.
Again shorthand keywords can be used here.
```yaml
spec:
  feature_a:
    configure: true # optional, boolean, defaults to false
    enable: [] # optional, list of strings
    disable: [] # optional, list of strings
```
This may not be the prefect design, and in the following use case of possible future kuadrant feature will show how far the design and concept can be pushed.

## mTLS

This feature has an [RFC](https://github.com/Kuadrant/architecture/pull/110/files) which describes how the feature works.
The example here shows how that configuration would work in the standard being outlined in this RFC.
At time of writing there is a discussion on wheather the feature should be active by default. 
In this example it is assumed the feature will be active by default.

There are only two components that are outlined in the related RFC as being affected by the feature, `authorino` and `limitador`.
This means that by default the feature would be enabled for both elements.
If the user only wants the feature enable for one of the two there is two approaches the user can take.
The first is to be explicit about which component they want the feature enabled for.
```yaml
spec:
  mtls:
    configure: true
    enable: ["authorino"]
```
Or they could be explicit about which components they do **not** want the feature enabled for.
```yaml
spec:
  mtls:
    configure: true
    disable: ["limitador"]
```
Both of the above configurations achieve the same effect.
If in the future this feature applies to a third component, the behaviour would not be the same.
In the first example the new component would not get the configured, while in the second example it would be configured.

If the user was wanting to fully disable the feature, as this example assumes active by default, they would use the following configuration.
```yaml
spec:
  mtls:
    configure: true
    disable: ["authorino", "limitador"]
```
Again if in the future if a third component was added, in the above configuration it may not be disabled by default.
This can be addressed by using keywords, and `"defaults"` in this case.
```yaml
spec:
  mtls:
    configure: true
    disable: ["defaults"]
```

With this configuration if any component is added in the future that we decide should be enabled by default, would be disabled by the user's configuration.

## Service Monitors
Service monitors currently does not have an RFC related to it, but the feature is being discussed.
Before going on a brief over view of this feature may be helpful.
The service monitors feature defines what component a service monitor should be configured for, allowing metrics to be scraped from.
This feature would affect many components including the Kuadrant-operator.

For this example the `"defaults"` keyword will reference authorino, dns-operator and limitador.
This leaves authorino-operator, limitador-operator and kuadrant-operator which would be optional.
Assuming this is the case once the feature is implemented is out of scope for this RFC.

Given that the lists are explicit the following configuration would only create service monitors for the kuadrant-operator.

```yaml
spec:
  service_monitors:
    configure: true
    enable: ["kuadrant-operator"]
```
If the user was wanting to extend which components had the feature applied to the configuration would look like:
```yaml
spec:
  service_monitors:
    configure: true
    enable: ["defaults", "kuadrant-operator"]
```
This can be expanded more to where the user does not make use of the dns-operator and therefore does not want service monitors to be created for the feature.
```yaml
spec:
  service_monitors:
    configure: true
    enable: ["defaults", "kuadrant-operator"]
    disable: ["dns-operator"]
```

## Log Level adjustment.
Being able to config the log level of the components from the global level has been a feature that has being talked about many times.
There is currently [https://github.com/Kuadrant/architecture/pull/97](https://github.com/Kuadrant/architecture/pull/97) would cover the finer details of the feature implementation.
However, it does show an issue with this API design.
The issue can be resolved.

To describe the issue, there are many components and many log levels.
How does the API design allow some components to be at one log level while others at a different level.
There are two approaches that can be taken.
Both have their pros and cons.

The first method is to have sub-fields under the feature field which would describe the configuration for that field.
```yaml
spec:
  logging_level:
    info:
      configure: true
      enable: []
      disable: []
    debug:
      configure: true
      enable: []
      disable: []
```
The second method is to use a list based approach.
In this method an extra field is added to the object.
```yaml
spec:
  logging_level:
    - level: "info"
      configure: true
      enable: []
      disable: []
    - level: "debug"
      configure: true
      enable: []
      disable: []
```
The list based method seems to have more flexibility than the field based method.
As the level is defined as a string and not a key, if there is a new component with different log levels added, updating to support those levels does not require updating the CRD.
However, the first approach is more user readable. 

One thing that would be left up to the feature design, and is out of scope of this RFC is how the elements should be prioritized. 
For example if a component is listed as enable in two sets of configures, which configuration is added.
In the case of logging I believe it should be the more verbose configuration.
```yaml
spec:
  logging_level:
    - level: "error"
      configure: true
      enable: ["kuadrant-operator"]
      disable: []
    - level: "debug"
      configure: true
      enable: ["kuadrant-operator"]
      disable: []
```

## Other features
The previous examples give a good example of how the API design is layout, and can be extended. 
However, there are a number of other features that could be supported by this API design.
These are mention here, but their structure is not shown.

- Multiply gateway support and configuration. [https://github.com/Kuadrant/architecture/pull/7](https://github.com/Kuadrant/architecture/pull/7)
- Custom installation of Kuadrant. [https://github.com/Kuadrant/kuadrant-operator/issues/71](https://github.com/Kuadrant/kuadrant-operator/issues/71)
- Installation of dashboards. [https://github.com/Kuadrant/kuadrant-operator/issues/1067](https://github.com/Kuadrant/kuadrant-operator/issues/1067)
- Installation of alerts.
- Tracing [https://github.com/Kuadrant/architecture/pull/96](https://github.com/Kuadrant/architecture/pull/96)
- High availability [https://github.com/Kuadrant/kuadrant-operator/issues/798](https://github.com/Kuadrant/kuadrant-operator/issues/798)
- Observability [https://github.com/Kuadrant/architecture/pull/97](https://github.com/Kuadrant/architecture/pull/97)

# Drawbacks
[drawbacks]: #drawbacks

The API design will not cover every use that is needed, and once the case is hit the UX of the API will become harder.
If this design is not followed and every feature defines its own design the UX for the user will always be hard, that in itself makes it normal.

This also reduces the level of design needed to add new features to the kuadrant CR, which could lead to over adding of features.
The over adding of features brings a maintenances cost on the operator that we may not be willing to support.

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
