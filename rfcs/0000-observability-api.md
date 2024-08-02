# Observability configuration

- Feature Name: Observability configuration
- Start Date: 2024-07-22
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/issues/0000)

# Summary
[summary]: #summary

This is a proposal for adding the ability to configure aspects of observability for Kuadrant as a whole to the already existing Kuadrant custom resource.

# Motivation
[motivation]: #motivation

Users of Kuadrant (Platform engineers and or Site reliability engineers) will want a way to be able to configure different aspects of observability for Kuadrant without having to configure things at the component level themselves. Providing users this ability allows them to integrate Kuadrant observability pieces with their own observability stack or Openshift user workload monitoring without needing a deep knowledge of the inner workings of the components, resulting in a fully comprehensive observability view all configured in one location.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The observability configuration, will allow a user to configure different parts of the offered Kuadrant observability. This provides more freedom to the user rather than the current all or nothing approach for certain aspects or the manual entry in multiple locations approach. With the new way the user will only have to fill in their configuration in one location.

### Potential configurations

The different aspects a user might want to modify could be the following:

| Observability piece   | Options                                                   |
|-----------------------|-----------------------------------------------------------|
| Logging               | **logLevel**                                              |
| Tracing               | **endpoint** , **tags**  , **insecure**, **strageyRules** |
| Metrics               | **enableService**, **deep**                               |
| Alerts *              | **namespace**, **enable**                                 |
| Dashboards *          | **namespace**, **enable**                                 |
| Other 3rd Party *     | e.g Kiali  **enable** bool                                |

###### **Note**: Observability pieces with a * denotes these are post v1 milestone
            

### Sample use case (With whats currently available)
A use case a user might have could be the following:

1. Setup tracing for the Kuadrant components Authorino and Limitador with the required endpoint and optional tag. 
2. Disable the creation of the service and serviceMonitors created by Kuadrant for the components and their operators
3. Configure logging for Authorino and Limitador. Note: For the first phase this is not available for the operators as they don't use the component CRs to implement new log levels flags/env-vars are used instead.

```yaml 

apiVersion: Kuadrant.io/v1alpha1
kind: kuadrant
spec:
  observability:
    tracing: 
        endpoint: rpc://tempo.tempo.svc.cluster.local:4317
        tags: tag1, tag2
        strategyRules: rule1, rule2 # Note: This is not implemented yet and is a mock up of what the rules might potentially look like and relies on efforts from [RFC96[(https://github.com/Kuadrant/architecture/pull/96)] 
    metrics: 
        enable: false
    logging:
      logLevel: debug      
```

### Status
The status of the Kuadrant CR will not be whether the observability stack is in a "healthy" state, i.e., Prometheus and Grafana is up and running. It should be the status of only the things that we have control over. The status will show what components the user has configured, for example, the logging setup for Authorino and Limitador. We will not be taking responsibility for aspects we don't have control over. 

#### Example status:

```yaml 
logging:
  authorino: debug
  limitador: debug
tracing:
  authorino: endpoint, tags
  limitador: endpoint, tags
metrics:
  disabled
```


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

In terms of how the information supplied in the Kuadrant CR will get passed to the other Kuadrant components, multiple options have been brought up:

1. directly - like setting flags or configuration directly on the deployments of the different Kuadrant components.
2. indirectly -  Passing the information to Authorino & Limitador via the Authorino & Limitador CRs
3. indirectly -  Passing the information to Authorino & Limitador in the form of their own Observability CRs

The best approach would be the approach 2), an indirect approach, meaning once the Kuadrant CR is updated, the information is passed to the relevant component CR. For example the tracing section in the Authorino CR spec would be updated with the required endpoints and other configuration from the Kuadrant CR. This does mean that the spec exposed is whats in the Kuadrant component CR's, but new changes can be requested and implemented in said components.

#### Adding, modifying and deleting values or no values

If a value is being configured either by being added, modified or removed, all changes will have to be made in the Kuadrant CR. If the component level operators are updated to change these values without the Kuadrant CR being changed, they will be overridden back to the source of truth the Kuadrant CR provides. 

If the value is removed from the Kuadrant CR it will also be removed from the relevant component CRs. If the value is removed from the component CR but not the Kuadrant CR it will be overridden and added back. 

# Unresolved questions

1. Should users be able to also configure at the component level too? and what should happen, should the information travel back up the chain to the Kuadrant CR?

#### Answered q's
1. Should the configuration be Kuadrant scoped as in its changed for everything or should be individual component scoped?
**Yes, it should be Kuadrant level scoped, i.e., the logging level configured is configured for every component.The user can not pick and choose.**

2. Do we add the values to the Kuadrant CR instead?
**Yes the configuration should be in the Kuadrant CR, the original observability CR no longer make sense. We should avoid APIs that rely on another APIs being there.**

3. Do we go with a observability policy instead of the proposed configuration style CR like whats being suggested? TLDR of what that would look like is quite similar to the observability CR in the fact that the observability piece is pulled out. However,instead of there just being one source of truth with the current method, and having the fine grained capabilities be in the one location, you would instead have, for most use cases, a single observability policy that would be attached at the Kuadrant component level. This would define would a Kuadrant level view i.e., if I enable tracing here its enabled for all the components that can handle it. If a component needs some special configuration a separate policy can be created and attached to that specific component instead. 

In terms of adding, modifying and deleting values or no values; adding a value to the Kuadrant level view populates the values to the relevant component CR's. If finer configuration is needed then another policy is created for said component. If values are deleted from the Kuadrant level policy, they are also then deleted from the relevant components CR's but if a component level policy is in place that will remain. If there is a observability policy in place in the Kuadrant level and a value is changes in the component CR, then it will get overridden by the Kuadrant level policy. If there is no values in the Kuadrant level policy and a user changes the component level CR the value will not get overridden but the user will have to be made aware that if something is added at the Kuadrant level in the future, it will be overridden.

**As of the time of writing this, no the original suggestion solves the issues we have right now, we dont need the Observability policy.**

#### Work thats needed
The indirect approach allows for a few, if any, changes to the Authorino operator and the Limitador operator etc. The bulk of the work that would be needed would be in the Kuadrant operator.

The changes will come into full effect through a phased approach due to the nature of some aspects not being available yet. The phased approach can follow the versioning syntax that k8s like v1beta1 or v1alpha1 and be portrayed in the CRD. 
 

### Default configuration
By default if no observability configuration is given or if values are left blank the current default values will still be used. The plan is to not change these and keep them as is. For new features like alerts and dashboard the default values will follow the same style and trend as the current approach with some features being disabled by default like the tracing or have certain default value, like info for logging for example.


# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The above approach allows for the following:
* User experience: The observability configuration in the CR can be easily read to see what the current state of the observability configuration is. Also theres only one place to update rather then multiple.
* Single source of truth: Rather then having multiple CR's to check what the current configuration is theres a single source of truth. Preventing users from accidentally changing values by mistake

## Other options:
An other option that has been investigated which is very similar to above, is having observability configuration as its own API. The majority of the work itself would be largely the same with operators having to move configuration to the Kuadrant CR and having new observability features use the Kuadrant CR as the source of truth. 

The reason why we should not go with the above method is reliance. The Observability CR will rely on the Kuadrant CR being present and we moving away from that model. From a user point of view having a users have to change configuration in 3+ separate crs in some cases, is tedious and slow.

If we don't decide to go with any of these options, the user will have to manually add the configuration of their desired observability stack in multiple places, which can result in poor user experience, mistakes being made and values not being tracked properly.


# Prior art
[prior-art]: #prior-art

Manually adding configuration to Kuadrant operator crs.

# Future possibilities
[future-possibilities]: #future-possibilities

Currently we only have configuration for Logging, Tracing and Metrics. Post v1 the plan is to add alerts, dashboards and potentially other 3rd party like Kiali.

* For the logging piece this is currently only for Authorino and Limitador and not their operators or the Kuadrant operator as these require flags to be changed so in phase two we can look at what work needs to be done to make this cohesive. For example being able to change a config and that gets percolated down to all the components .

