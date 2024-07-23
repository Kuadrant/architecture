# RFC Template

- Feature Name: Observability API
- Start Date: 2024-07-22
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/issues/0000)

# Summary
[summary]: #summary

This is a proposal for a new Kuadrant observability API. Providing a way for users to enable or disable different pieces of the observability stack.
# Motivation
[motivation]: #motivation

Users of Kuadrant (Platform engineers and or Site reliability engineers) will want a way to be able to opt in or opt out of different aspects of observability. Providing users this ability allows them to integrate Kuadrant observability pieces with their own observability stack or Openshift user workload monitoring. Resulting in a fully comprehensive observability view all configured in one location.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The observability api, will allow a user to choose different parts of the offered Kuadrant observability. This provides more freedom to the user rather then the all or nothing current approach for certain aspects or the manual entry in multiple locations approach. With the new way the user will only have to fill in their configuration in one location.

### Potential configurations

The different aspects a user might want to modify could be the following:

| Observability piece   | Kuadrant component                                        | Options                                                                                                                       |
|-----------------------|-----------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Logging               | Kuadrant operator, Authorino operator, Limitador Operator | **component**  **logLevel**:  **logMode**:                                                           |
| Tracing               | Authorino operator, Limitador operator                    | **component** , **endpoint** , **tags**  , **insecure**, **strageyRules**  |
| Metrics               | Kuadrant operator, Authorino, Limitador, DNS Operator     | **component** , **enableService** , **port** , **deep**                                            |
| Alerts *              | Kuadrant operator, Authorino, Limitador                   | **namespace** , **component**, **enable**                                                                 |
| Dashboards *          | Kuadrant operator, Authorino, Limitador                   | **namespace** , **component**, **enable**                                                                 |
| Other 3rd Party *     | e.g Kiali                                                 | **enable** bool                                                                                                               |

###### **Note**: Observability pieces with a * denotes these are post v1 milestone

### Example: Every component has every option available
The below example is a scenario where every option available is being used all configurable by the same type of API. 

Although below is showing a CR with every value filled in, the changes will come into full affect through a phased approach due to the nature of some aspects not being available yet. The phased approach can follow the versioning syntax that k8s like v1beta1 or v1alpha1 and be portrayed in the CRD. 
 
```yaml 

apiVersion: Kuadrant.io/v1alpha1
kind: observability
spec:
      logging:
        component:
          authorino:
            logLevel: debug
            logMode: production
          limitador: 
            logLevel: info
            logMode: production
          authorino: 
            logLevel: error
            logMode: development
          limitador: 
            logLevel: debug
            logMode: development  
    
    tracing:
      component:  
        limitador:
          endpoint: rpc://tempo.tempo.svc.cluster.local:4317
          insecure: true
          tags: tag1, tag2
          strategyRules: #Note: this is a mock up of a potential new feature that is currently being discussed in the following [RFC](https://github.com/Kuadrant/architecture/pull/96)
            rule1:
              best-rule-in-the-world-1 
            rule2:
              best-rule-in-the-world-2  
        authorino:
          endpont: rpc://tempo.tempo.svc.cluster.local:4317
          insecure: true
            tags: tag1, tag2
          strategyRules: 
            rule1:
              best-rule-in-the-world-1
            rule2:
              best-rule-in-the-world-2     

    metrics:
      component: 
        authorino: 
          enableService: true
          port: 8080
          deep: true
        authorinoOperator:
          enableService: true
          port: 8080
        limitador:
          enableService: true
          port: 8080
          deep: false
        limitadorOperator:
          enableService: true
          port: 8080
        KuadrantOperator:
          enableService: true 
          port: 8080
          deep: true
        dnsOperator:
          enableService: true
          port: 8080   

    alerts:
      namespace: my-amazing-namespace
      component:
        authorino:
          operatorLevel: true
          componentLevel: true
        limitador:
          operatorLevel: true
          componentLevel: true  
        Kuadrant:
          operatorLevel: true
          componentLevel: true  

          
    dashboards:
      namespace: my-amazing-namespace
      component:
        authorino:
          operatorLevel: true
          componentLevel: true
        limitador:
          operatorLevel: true
          componentLevel: true  
        Kuadrant:
          operatorLevel: true

            
```
### Sample use case (With whats currently available)
A use case a user might have would be they desire setting up tracing in the Limitador operator implementing the required endpoints and optional tag. The user also wants metrics setup with custom ports and requires service and serviceMonitors to be created for Kuadrant-operator and Authorino-operator as well as have the Authorino have a log level of Debug.

```yaml 

apiVersion: Kuadrant.io/v1alpha1
kind: Observability
spec:
  tracing:  
    component:
      limitador:
          endpoint: rpc://tempo.tempo.svc.cluster.local:4317
          tags: tag1, tag2
  metrics: 
    component:
      authorino: 
        enableService: true
        port: 8084
        deep: true 
      kuadrant:   
        enableService: true
        port: 8084 
  logging:
    component:
      authorino: 
        logLevel: debug      
```
### Status
The status of the Observability CR will not be the observability stack is in a "healthy" state i.e Prometheus and Grafana is up and running. It should be the status of only the things that we contribute for example is new Logging and Tracing now in place. We will not be taking responsibility for aspects we don't have control over. 


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

In terms of how the information supplied in the Observability CR will get passed to the other Kuadrant components, multiple options have been brought up:

directly - like setting flags or configuration directly on the deployments of the different Kuadrant components.
indirectly -  Passing the information to Authorino & limitador via the Authorino & Limitador CRs
indirectly -  Passing the information to Authorino & limitador in the form of there own Observability CRs

The best approach would be the indirect approach, meaning once the Observability CR is updated the information is passed to the relevant component CR. For example the tracing section in the Authorino CR spec would be updated with the required endpoints and other configuration in the Observability CR, this would then be updated in the Authorino CR. This does mean that the spec exposed is whats in the Kuadrant component CR's but new changes can be requested and implemented in said components.

#### Adding, modifying and deleting values or no values

If a values is being configure either being added, modified or removed all changes will have to be made in the Observability CR. If the component level operators are updated to change these values without the Observability CR being changed they will be overridden back to the source of truth the Observability CR provides. 

If the value is removed from the Observability CR it will also be removed from the relevant component CRs. If the value is removed from the component CR but not the Observability CR it will be overridden and added back. 


# Unresolved questions

1. Should the configuration be Kuadrant scoped as in its changed for everything or should be individual component scoped?
2. Do we add the values to the Kuadrant CR instead?
3. Do we go with a observability policy instead of the proposed configuration style CR like whats being suggested? TLDR of what that would look like is quite similar to the observability CR in the fact that the observability piece is pulled out but instead of there just being one source of truth with the current method, and having the fine grained capabilities be in the one location. Instead you would have for most use cases a single observability policy that would be attached at the Kuadrant component level, in there you would have a Kuadrant level view i.e if i enable tracing here its enabled for all the components that can handle it. From there if a component needs some special configuration a separate policy can be created and attached to that specific component instead. 

In terms of Adding, modifying and deleting values or no values. Adding a value to the Kuadrant level view populates the values to the relevant component CR's. If finer configuration is needed then another policy is created for said component. If values are deleted from the Kuadrant level policy they are also then deleted from the relevant components CR's but if a component level policy is in place that will remain. If there is a observability policy in place in the Kuadrant level and a value is changes in the component CR then it will get overridden by the Kuadrant level policy. If there is no values in the Kuadrant level policy and a user changes the component level CR the value will not get overridden but the user will have to be made aware if something is added at the Kuadrant level in the future it will be overridden.

#### Work thats needed
The indirect approach allows for not much if any changes to the Authorino operator and the Limitador operator etc . The bulk of the work that would be needed would be in the Kuadrant operator.

In terms of if this piece of work would require its own observability controller the answer needs more discussion. Some of the work could be done by the Kuadrant CR but not everything for example the alerts or the dashboards don't make sense to have the Kuadrant operator implement them so a new "observability" controller would be needed. This then begs the question if we need a new controller for some parts of the CR it might make sense to have the new controller handle the full CR and not have the Kuadrant CR reconcile it.

The changes will come into full affect through a phased approach due to the nature of some aspects not being available yet. The phased approach can follow the versioning syntax that k8s like v1beta1 or v1alpha1 and be portrayed in the CRD. 
 

### Default configuration
By default if no observability CR or if values are left blank the current default values will still be used. The plan is to not change these and keep them as is. For new features like alerts and dashboard the default values will follow the same style and trend as the current approach with some features being disabled by default like the tracing or have certain default value, like info for logging for example.


# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The above approach allows for the following:
* User experience: The observability CR can be easily read to see what the current state of the observability configuration is. Also theres only one place to update rather then multiple.
* Abstraction: The [RFC 0006](https://github.com/Kuadrant/architecture/blob/main/rfcs/0006-Kuadrant_sub_components_configurations.md) suggests having observability in the Kuadrant CR with other non observability related variables. With new proposed ideas and aspects for observability coming down the line and the current quite extensive options users can choose from, the Kuadrant CR will become "muddied", hard to maintain and hard to read.  
* Future proof: Observability currently is Logging, metrics and tracing but theres plans for more configuration. Having it has a standalone API allows for engineering to easily add new features.
* Single source of truth: Rather then having multiple crs to check what the current configuration is theres a single source of truth. Preventing users from accidentally changing values by mistake

## Other options:
An other option that has been investigated which is very similar to above, is having observability configuration as a element of the Kuadrant CR spec. The majority of the work itself would be largely the same with operators having to move configuration to the Kuadrant CR and having new observability features use the Kuadrant CR as the source of truth. 

The reason why we should go with the above method is Abstraction. The Kuadrant CR quite quickly can get "muddied" with observability and be harder to read and maintain also losing its main purpose; being the install kick of and maintainer for the Kuadrant operators as well as the single source of truth aspect. From a user point of view having a users have to change configuration in 3+ separate crs in some cases, is tedious and slow.

If we don't decide to any of these options the user will have to manually in multiple places add there configuration of their desired observability stack which can result in poor user experience, mistakes being made and values not being tracked properly.

There was also a previous RFC [RFC 0006](https://github.com/Kuadrant/architecture/blob/main/rfcs/0006-Kuadrant_sub_components_configurations.md), that suggests adding everything to the Kuadrant CR, why this RFC should replace the observability aspect are for the reasons stated above:
  * Abstraction
  * User experience
  * Readability
  * Adding to much to the Kuadrant CR

# Prior art
[prior-art]: #prior-art

Manually adding configuration to Kuadrant operator crs.

# Future possibilities
[future-possibilities]: #future-possibilities

Currently we only have configuration for Logging, Tracing and Metrics. Post v1 the plan is to add alerts, dashboards and potentially other 3rd party like Kiali.

