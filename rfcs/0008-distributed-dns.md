# Distributed DNS Load Balancing

- Feature Name: Distributed DNS Load Balancing
- Start Date: 2024-01-17
- RFC PR: [Kuadrant/architecture#0008](https://github.com/Kuadrant/architecture/pull/55)
- Issue tracking: [Kuadrant/architecture#0008](https://github.com/Kuadrant/architecture/issues/56)

# Terminology

- OCM: [Open Cluster Management](https://open-cluster-management.io/)
- Dead End Records: Records that target a Kuadrant defined CNAME that no longer exists

# Summary
[summary]: #summary

Enable the DNS Operator to manage DNS for one or more hostnames across one or more clusters.  Remove the requirement for a central OCM based "hub" cluster or multi-cluster control plane in order to enable multi-cluster DNS.

# Motivation
[motivation]: #motivation

The DNS operator has limited functionality for a single cluster, and none for multiple clusters without leveraging OCM. Having the DNS operator capable of supporting the full feature set of DNSPolicy in these situations significantly eases on-boarding users to leveraging `DNSPolicy` across both single and multi-cluster deployments and gives a single layer to work against rather than defining resources in hubs and then understanding how to distribute them across spoke clusters.

# Diagrams
[diagrams]: #diagrams


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Single Cluster and Multiple Clusters
Every cluster requires a gateway and DNSPolicy configured with a provider secret. In each cluster, when a matching listener host defined in the gateway is found that belongs to one of the available zones in the DNS provider, the Kuadrant Operator, based on the chosen DNSPolicy strategy (simple | loadbalanced), will reflect the gateway address into a DNSRecord and the DNSRecord controller (part of the DNS Operator) will reconcile endpoints from that DNSRecord into the zone. The DNS controller will be responsible for regularly validating what is in the zone, reflecting that zone into the DNSRecord status, ensuring its own records are correctly reflected and that their are no dead end records present. So the combination of each of the controllers records within or across clusters will bring the zone records  to a **cohesive and consolidated state** that reflects the full record set for a given DNS name. 


## Migrating from distributed to centeralised (OCM Hub)

This is not covered.

## Migrating from single to multiple cluster (distributed)

The steps for this are quite straight forward:
This assumes you already have 1 cluster with a gateway and DNSPolicy setup.
- Create a new gateway in a second cluster that has a listener defined with a hostname you want to use across clusters
- Create a DNSPolicy that uses the same strategy as the DNSPolicy already configured that targets this gateway (very likely a mirror copy)
- Ensure you create and link to the same DNS Provider (via credential) with the existing zone

## Cleanup

Cleanup of a cluster's zone file entries is triggered by deleting the DNS Policy that caused the records to be created. The DNSPolicy will have a finalizer, when it is deleted the Kuadrant operator will mark the created DNSRecord for deletion. The DNS record will have a finalizer from the DNS Operator which will clean the records from the zone before removing the finalizer.

## Switching strategy (IE from LoadBalanced to Simple or the other way)
The DNSPolicy offers 2 strategies 
1) Simple (one A record multiple values)
2) LoadBalanced (leverages CNAMES and A records to provide advanced DNS based routing to be used (example GEO and Weighted) )

The strategy field will be immutable once set. To migrate from one strategy to the other you must first delete the existing policy and recreate it with a new strategy. In a multi-cluster environment, this can mean you have a period of time where 2 different strategies are set (simple on one and load balanced on another). The controller will mitigate against this in the following way. When attempting to update the remote zone, it first does a read. If it spots existing records in that zone that correlate with a different strategy, it will not write its own records but will flag this as an error state in the status. So the records will remain in the original strategy until all policies for that listener host match (more in this below).


## Orphan Mitigation

It is possible for an orphan set of records to be created if for example a cluster is removed and the controller on that cluster is not given time to clean up. This is not something we currently mitigate against directly in the controller. However, you can define health checks that would remove these orphans from the DNS response. Additionally, there is future work that leverages a "heart beat" that should allow each alive controller to see and remove a dead controllers records within a given criteria.

### Health Checks
The health checks provided by the various DNS Providers will be used AWS will be the only one implemented initially as part of this RFC as we have done it before. For the others we may choose to cover this area in more detail in a subsequent RFC and they may be limited to having infra within that provider.

[AWS](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/DNS-failover.html) (well understood)
[Google DNS](https://cloud.google.com/DNS/docs/zones/manage-routing-policies#before_you_begin)
[Azure](https://learn.microsoft.com/en-us/azure/traffic-manager/traffic-manager-monitoring)

## OCM / ACM
This is very similar, with the caveat that the "cluster" is the hub cluster for this OCM and that the gateway will be a Kuadrant gatewayClass. You can think about this setup as "single cluster" with a gateway that has multiple addresses. It will continue to work with these changes. 


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation


## General Logic Overview

### Kuadrant Operator

The general flow in the Kuadrant operator follows a single path, where it will always output a valid DNSRecord, which specifies everything needed for the listener host of a given gateway on the cluster, and is unaware of the concept of distributed DNS. The core logic for doing this already exists.


### DNS Operator / controller

We will update the DNS Operator to leverage the external DNS provider logic (IE effectively use it as a library) and layer our multi-cluster changes ontop of this logic. Our hope is to keep what we are doing as compatible with external DNS as possible so that we can contribute / propose changes to external-dsn
The DNSRecord created by the Kuadrant operator is reconciled by the DNSRecord controller as part of the DNS Operator component (the operator and controller terms can be used interchangeably). In the event of a change to the endpoints in the DNSRecord resource (including deletion and creation of the resource), the DNSRecord controller will first pull down the relevant records for that DNS name from the provider and store them in the DNSRecord status. Next it will update the affected endpoints in that record set and validate the record set has no "dead ends" before writing the changes back to the DNS Provider's zone. Once the remote write is successful, it will then re-queue the DNSRecord for validation. The validation will be the same validation as previously done before the write. It will pull the zone and ensure that the endpoints from its own local DSNRecord are reflected correctly. If the validation is successful, the controller will mark this in the status of the DNSRecord and will re-queue the validation for ~15 (example) minutes later (a stable state verification). If validation is unsuccessful, it will re-queue the DNSRecord for validation rapidly (5 seconds for example). With each re-queue after an unsuccessful validation, it will add a random amount of "jitter" time to increase the chance it moves out of sync with any other actors. Each time it re-queues, it will mark this in the status (see below). At any point, if there is a change to the on-cluster DNSRecord, this back off and validation will be reset and started again.  

As each controller is responsible for part of the overall DNS record set for a given DNS name in a shared zones and potentially as values in a shared record and as neither the zone nor records are lockable, there will be scenarios where one controller overwrites some part of a zone/record with a stale view. This can happen if two or more controllers attempt to update the remote zone at the same time as each controller may have read the zone before another clusters has executed it's write allowing the zone to be updated without the knowledge of other actors that have also done a read. During this type of timing clash, effectively the last controller to write will have its changes stored. When it does validation, its changes will still be present and it will revert to a stable cycle. The other controllers involved, will see their changes missing in the remote zone during their validation check and so will pull and update the zone again (setting a new validation with a random jitter applied to increase the chances they don't clash again). Again the last controller to write will see its changes. So with this we can predict a worst case scenario of (num clashing controllers * (validation_loop + jitter)). However with adding in the jitter, it is likely that this will be a shorter period of time as multiple clusters will fall out of sync and should resolve their state within the min-max requeue interval rather than only ever one at a time. 

### Why are we ok with a zone falling out of sync for a short period of time?

As mentioned each of the controllers is, other than in error scenarios (covered later), attempting to bring the zone into a converged and consistent state and so while some stale state can be written, the controllers will not fight (IE go endlessly back and forth) over what should actually be in the zone. Each controller is only interested in its own records and removing dead ends. 
As mentioned, the zone is not lockable, stale data can be written. But over a "relatively short time" the zone should come into a consistent shape. As a general rule, DNS records (managed by Kuadrant or not) for a given listener host cannot tolerate a constantly changing system and provide a good user experience. Rather it is intended that the record set remain relatively static. It is built into DNS to allow clients to cache results for long periods of time and so DNS itself is an eventually consistent system (write to the zone, secondary servers poll for changes, clients cache for TTL etc).  Additionally the temporary impact of a clash should be localized to only the values being changed by the subset of clusters.
As pointed out, the number of writes to the DNS zone should be few (although potentially "bursty") and done by relatively few concurrent clients. Constant individual and broad changes to any DNS system are likely to result in a poor experience as clients (browsers etc) will be out of sync and potentially send traffic to endpoints that are no longer present (example changing the address of the gateway once every 30 seconds). 
What about weight and GEO routing? Clusters do not change GEO often. Once set up, GEO should not be something that constantly changes. Weighting is also something that should not change constantly. Rather it will be set and left in most cases. There is no use case for a constantly changing DNS Zone. 
So the triggers for a change and so potential conflict would be if across 2 or more clusters you had

- concurrent Address, Geo or Weighting changes
- Misconfiguration (IE conflicting DNS strategies across 2 or more clusters coverd below)
- Concurrent health check changes (AWS only initially)

example status but will likely change in implementation:

```
{
  Type: UnresolvedConflict
  Status: False
  Reason: FirstValidation
  Message: "listener: a.b.com validated successfully. re-queue validation for <time>"
}
{
  Type: UnresolvedConflict
  Status: True
  Reason: RecordsMissing|ConflictingStrategy|ConflictingValue
  Message: "listener: a.b.com strategy : simple conflicts. Current validation atttempt is y in z minutes. Next is <time>"
}
{
  Type: UnresolvedConflict
  Status: True
  Reason: RecordsMissing|ConflictingStrategy|ConflictingValue
  Message: "listener: a.b.com health check changed : Current validation atttempt is y in z minutes. Next is <time>"
}

```

### How do we spot misconfiguration / on-going conflicts

Misconfiguration is focused around the DNSPolicy (as this is our Kuadrant API). However other forms of misconfig/conflict may also be spotted with the proposed method. 
With this type of conflict, the controller will observe an unresolvable state. This will present itself as there having been no change to the spec of the local DNSRecord (observed by the generation sequence number matching the observedGeneration in the status) but the remote values in the zone have changed. When this is spotted we will update the status as above and increment a metric to count encountered conflicts, we may also decide to further increase the requeue time to reduce load. Issue has been reported and so back off until issue resolved.
An example of this would be if you had one cluster with a DNSPolicy whose strategy was set to LoadBalanced and second cluster with a DNSPolicy that had a strategy of Simple (assuming this DNSPolicy was targeting a listener with the same hostname). This type of configuration would mean the structure of the DNS zone would constantly change. In a strategy conflict, the controller will be able to observe this due to the structure of the remote record set. In this case, the controller will not overwrite but instead will report the issue and requeue.


### Configuration Updates
There will be a new configuration option that can be applied as runtime argument (to allow us emergency configuration rescue in case of an unexpected issue) to the kuadrant-operator:
- `clusterID`  (see below for how this is used)
  - When absent, this is generated in code (more information below). A unique string to identify this cluster apart from all other clusters.

### Changes to DNS Policy reconciler (kuadrant operator)

This will primarily need to be made aware that it will need to propagate the DNS Policy health check block into any DNS Records that it creates or updates using the provider specific fields.

It will also need to update the format of the records in the DNSRecord it creates. We will drop the use of a guid from the DNS name and instead simple use a prefix (klb) for example. This will mean multiple gateways with the same or different names can now update the record set if their gateway shares a listener with a common host name. 

### DNS Operator

This component takes the DNSRecord output by the kuadrant-operator and ensure they are accurately reflected in the DNS Provider zone and currently assumes the DNS Record is the source of truth and always ensures that the zone in the DNS Provider matches the local DNS Record CR.

There are several changes required in this component:

#### Generate a ClusterID

The cluster will need a deterministic manner to generate an ID that is unique enough to not clash with other clusterIDs, that can be regenerated if ever required.

This clusterID is used to identify which A/CNAME records were created by which cluster/controller. This is done via the registry implementation in external-dns. Initially we will use the TXT registry, but in the future we will also want to offer a different form of storage for the registry .

The suggestion [here](https://groups.google.com/g/kubernetes-sig-architecture/c/mVGobfD4TpY/m/nkdbkX1iBwAJ) is to use the UID of the `kube-system` namespace, as this [cannot be deleted easily](https://groups.google.com/g/kubernetes-sig-architecture/c/mVGobfD4TpY/m/lR4SnaqpAAAJ) - so is practically speaking indelible.

We can take the UID of that namespace and apply a hashing algorithm to result in 6-7 character code which can be used as the clusterID for the local cluster.

#### Ensure local records are accurate

When the local DNSRecord is updated, the DNS Operator will ensure those values are present in the zone by interacting directly with the DNS Provider. As mentioned above it will pull the zone, merge in its changes and write back the result before requeuing the resource for validation. With each requeuing it will add a random amount of time to attempt to fall out sync with any conflicting controllers.

#### Cleanup

When a DNS Policy is marked for deletion the kuadrant operator will delete all relevant kuadrantRecords.

Whenever a deleted kuadrantRecord is reconciled by the DNS Operator, it will remove any relevant records from the DNS Provider (i.e. any record identified by the clusterID for the deleted host.

#### Prune dead ends from DNS Records

If a DNSRecord is deleting, there is the potential that we are leaving behind a dead end, which will need to be recursively cleaned until none remain.

What is a dead end? If a CNAME exists whose value is a record that does not exist, this CNAME will not resolve.

The controller will need to work through the DNS Record and ensure that from hostname to A (or external CNAME) records is structurally sound and that any dead ends are removed, to do this, the DNS Operator will need to first read what is published in the provider zone.

#### Removing the finalizer from DNSRecord

When a DNSRecord is reconciled, the DNS Operator should apply a finalizer to it.

When a DNSRecord is being removed, the following must be successfully completed before the finalizer can be removed:
- Remove the local clusters records and targets from the relevant zone
- Perform a prune on the relevant root host
- Apply the results of the prune to the DNS Provider
- re-queue for validation

Only once these actions have all resolved should the finalizer be removed, and the kuadrantRecord allowed to be deleted.

Should a deleted DNS Record be recreated, or a deleted DNS Policy be restored, all of the deleted records will be restored by the DNS Operator, at that point.

#### Aggregating Status

When the DNS Operator performs actions based on the state of a kuadrantRecord, it should update the kuadrantRecord status with results of these actions, similar to how it is currently implemented.

When the Kuadrant operator sees these statuses on the DNSRecord related to a DNS Policy, it should aggregate them into a consolidated status on the DNS Policy status.

It will be possible from this DNS Policy status to determine that all the records for this cluster, related to this DNS Policy, have been accepted by the DNS Provider.


#### Cloud Provider Rate Limiting

It is possible particularly if there were a large number of hosts that we would see rate limited responses from the cloud provider. These rate limited responses are most likely to happen in a burtsy scenerio. A change to all members of the fleet. An entire new fleet being brought on line all at the same time. The controller will return a specific error type to the reconcile loop if it recieves a rate limit response. This will be logged in the status of the DNSPolicy and the controller will attempt to mitigate this by increasing its requeue time (IE slowing down the number of requests ) and recording this in the status.

We don't currently have a theoretical maximum number of hosts / clusters but our recommendation is not to go beyond a small number of clusters initially. We intend to put together a more comprehensive scaling impact in the near future. It is worth mentioning with the expected stable state of DNS and a relatively low change rate, that even if we got rate limited for a short period it would resolve itself and that the number of clusters that could be supported would be quite large.

API usage limits on DNS providers:
- [Google](https://cloud.google.com/service-usage/docs/quotas) - 240 read requests per minute, 60 writes per minute
- [Azure](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/request-limits-and-throttling) - 60 lists and 1000 gets per minute, 40 writes per minute
- [Route53](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/DNSLimitations.html) - 5 API requests per second read or write
- No health checks for private clusters

#### Metrics

Some useful metrics have been determined:
- Emit a metric whenever the DNS Operator writes to the zone.
- Emit a metric whenever the kuadrant operator updates reads the zone.
- Emit a metric whenever the DNS Operator deletes an entry from the zone.
- Emit a metric whenever the DNS Operator prunes a shared record from the zone.
- Emit a metric for counting discovered conflicts

# Drawbacks
[drawbacks]: #drawbacks

clean up in disaster scenario for multi-cluster:
- any cluster nuked without time to cleanup will never be cleaned from the zone.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Allows multi-cluster DNS without OCM/ACM as a dependency
- It provides the most simple migration procedure from single to multiple clusters
- It provides a simple and useful demo without excessive setup

# Prior art
[prior-art]: #prior-art

No prior art.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Dealing with significant portions of the cluster groups going offline unexpectedly.

# Future possibilities
[future-possibilities]: #future-possibilities

## Heartbeat Check

It may turn out that disappearing clusters create more headaches than currently anticipated, and in this scenario we could respond with having each cluster regularly update a heartbeat TXT field (say every 5 minutes), and on this timer also check all other clusters heart beats, if some are falling old then all the records related to that heartbeat's clusterID can be removed.

## Migration from single / multi cluster to OCM/ACM

Once this work has completed, it's possible that there is a migration path apparent from non-ocm to ocm without having to start from scratch, e.g. convert back to a single cluster > install OCM hub to that cluster > convert other clusters into spokes.

## Integration with local DNS servers (e.g. CoreDNS)

To further improve the simple demo of the MGC product, making the clusters work as nameservers removes the requirement for any DNS providers, and further simplifies the local development environment setup, so that the features of MGC can be explored without requiring excessive steps from the user.

## Accurate scaling predictions.

Currently we don't have a strong grip on how far we can scale, but are confident we can scale to a small number of clusters initially (3-5). Beyond this number it may be worth looking at the ACM/OCM integration until we have more accurate information.

## Additional registry implementations

Adding different forms of storage based on the registry API 

