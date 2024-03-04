# Distributed DNS Load Balancing

- Feature Name: Distributed DNS Load Balancing
- Start Date: 2024-01-17
- RFC PR: [Kuadrant/architecture#0008](https://github.com/Kuadrant/architecture/pull/55)
- Issue tracking: [Kuadrant/architecture#0008](https://github.com/Kuadrant/architecture/issues/56)

# Terminology

- OCM: [Open Cluster Management](https://open-cluster-management.io/)
- Dead End Records: Records that target a kuadrant defined CNAME that no longer exists

# Summary
[summary]: #summary

Enable the DNS Operator to manage DNS for one or more hostnames across one or more clusters.  Remove the requirement for a central OCM based "hub" cluster or multi-cluster control plane in order to enable multi-cluster DNS.

# Motivation
[motivation]: #motivation

The DNS operator has limited functionality for a single cluster, and none for multiple clusters without leveraging OCM. Having the DNS operator capable of supporting the full feature set of DNSPolicy in these situations significantly eases on-boarding users to leveraging `DNSPolicy` across both single and multi-cluster deployments.

# Diagrams
[diagrams]: #diagrams


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Single Cluster and Multiple Clusters
Every cluster requires a DNS Policy configured with a provider secret. Every zone will be treated as a distributed DNS zone. In each cluster, when a matching listener host defined in the gateway is found that belongs to one of the available zones in the provider, the DNS controller will add addresses from that gateway to the zone file based on the chosen strategy (simple | loadbalanced) in the DNSPolicy. Additionally the controller will be responsible for regularly validating what is in the zone ensuring its records are correctly reflected, that their are no dead end records and updating a local reflection of the zone records. Assuming no misconfiguration of DNSPolicy (more on this later), all of the controllers will be attempting to reach a **combined consolidated state** that reflects the full record set for a given DNS name. 
But as each controller is responsible for part of the overall DNS record value set for a given DNS name in a shared zone and as neither the zone nor records are lockable, there can be scenarios where one controller overwrites some part of a zone/record with a stale view. This stale view however will eventually become consistent (see below). It is important to note that this design favours availability and partition tolerance (AP). Perfect data consistency at all times is not the goal but rather it is expected that the system can tolerate eventual consistency. 

### Why are we ok with a zone falling out of sync for a short period of time?

DNS records (managed by Kuadrant or not) for a given listener host cannot tolerate a constantly changing system and provide a good user experience. Rather it is intended that the record set remain relatively static. It is built into DNS to allow clients to cache results for long periods of time and so DNS itself is an eventually consistent system (write to the zone, secondary servers poll for changes, clients cache for TTL etc).  The source of truth in this design is stored across many clusters; the remote DNS Zone is not the ultimate source of truth for DNS controllers but rather a reflection of the truth held by many clusters. It is this slow rate of change, consolidatable state and the fact the zone (from an individual controller's perspective) is not to be trusted that allows us to use a distributed approach. 
As pointed out, the number of writes to the DNS zone should be few (although potentially "bursty") and done by relatively few clients. Constant individual and broad changes to any DNS system are likely to result in a poor experience as clients will be out of sync and potentially send traffic to endpoints that are no longer present (example changing the address of the gateway once every 30 seconds). 
What about weight and GEO routing? Clusters do not change GEO often. Once set up, GEO should not be something that constantly changes. Weighting is also something that should not change constantly. Rather it will be set and left in most cases. There is no use case for a constantly changing DNS Zone.

### How can a zone fall out of sync?

The scenarios outlined below describe how the record set for a given DNS name in a zone can temporarily fall out of sync. While they exist and will happen, it is reasonable to assume that these scenarios are relatively uncommon in a stable environment due to the low rate of change and defensive pattern for varying the time between validation loops (more below). Also if they were to occur in a stable environment, they should only occur if a big change is happening across the entire fleet or, more likely, affect a small subset of the fleet such as clusters being added or decommissioned or the geo/weighting being tweaked / set on a subset of the fleet. Finally every scenario needs multiple clusters to be attempting to update the DNS zone at exactly the same time.  

Below are the known triggers for an actual change to records in the DNSZone. If two or more clusters attempt one of these changes at the same time a potential conflict can occur:

- New Gateways added to the fleet with a (new DNSPolicy added) with an existing shared listener host.

If two or more clusters are attempting to add / update addresses at the same time, it is possible that for a short period the last controller to write will have a stale view and overwrite a value from another cluster. Example (A record with multiple values). There will always be a valid value set, but it may not be the full set of addresses for a short period. 

- Gateways being decommissioned from the fleet or clusters being decommissioned (DNSPolicy being deleted)

Again if multiple clusters where being removed / added and the controllers attempted to update the same record at the same time, they may overwrite each others values for a short period of time.

- Geo or Weighting being changed 

As adding GEO or Weighting attributes can change the record set, if you were to change these on several clusters and some of those clusters attempted to update the record set at the same time, there may be some inconsistency for a short period.


### How do we resolve a stale state?

(note this may change subtly and be optimized in implementation but the fundamental principles apply)

Each gateway listener has its own record set reflected on cluster as a DNSRecord that the DNS controller understands and will add into the specified zone (also reflected on cluster in the status of the DNSRecord). When the controller needs to perform a write to the zone, it first pulls the existing zone, writes this to the status and then merges its own DNS record spec with what is in the zone. It then updates the remote zone with this new reflection and writes the resulting record set to the status of the DNS record. It then immediately re-queues the record to validate against the zone (example 5 seconds + or - a random jitter) later. When the validation loop executes, it will check the zone still contains the changes it expects to be there (IE its records) and update the status to reflect the status of the validation check. If this check is successful, it will update its status and then enter a quiet phase where it re-queues itself for 15 (example) minutes later to re-validate (this allows for an unexpected overwrite). 
If a new change is made to the local DNS record, it starts the write process again. If the validation is not successful (records are not present), the controller will assume there is a conflict and update the zone again and enter a re-try state for that DNS record where the validation interval is (example 5 seconds) plus or minus a random jitter amount of time within a set range (example 1-5 seconds). This jitter reduces the chances of a reoccurring clash with the same controller in the future.
In a worst case scenario, this should lead to a linear amount of time (re-try * num clashing controller) before the system becomes consistent where which ever cluster is the last to write will be the first to see a valid state. 

example status but will likely change in implementation:

```
{
  Type: UnresolvedConflictDiscovered
  Status: True
  Reason: RecordsMissing|ConflictingStrategyDiscovered
  Message: "current atttempt is y in z minutes"
}
```

Each DNS change results in a minimum of 3 requests to the cloud provider 2 lists (1 before the write and one post the write) and 1 update within the re-try interval (IE 5 seconds + a max 5 second random jitter) so total request is (3 request per ~5-10 seconds * the number of conflicting controllers). E.G. if you had a situation where ten clusters were clashing this would mean worst case 30 requests with the first 5-10 seconds and falling off linearly (IE 27 within the next 5-10 seconds 24 and so on until the correct state was converged on). Note with the random jitter, it is likely that this would fall off at a faster rate I am describing absolute worst case for reaching a converged state.


### How do we spot misconfiguration

Misconfiguration is focused around the DNSPolicy (as this is our API). However other forms of misconfig may also be spotted with the proposed method. What the controller will observe is an unresolvable state:

Misconfiguration of a DNSPolicy for a shared listener host, is something that would mean the DNS zone could not end up in a consolidated state. Rather it is a long term conflicting state. Examples of this would be if you had one cluster with a DNSPolicy whose strategy was set to LoadBalanced and second cluster with a DNSPolicy that had a strategy of simple (assuming this DNSPolicy was targeting a listener with the same hostname).

In situations like this, all of the DNS operators would reach the error state outlined above where they were unable to validate their changes were present and the re-tries would continue indefinitely (this would be visible in the status as shown above). Effectively you would observe all the controllers for given host continually updating the zone and never entering a quite period.


## Migrating from single/multiple clusters to OCM

This is not covered.

## Migrating from single to multiple cluster (distributed)

The steps for this are quite straight forward:
This assumes you already have 1 cluster with a gateway and DNSPolicy setup.
- Create a new gateway that has a listener defined with a hostname you want to use across clusters
- Create a DNSPolicy that uses the same strategy as the DNSPolicy already configured
- Ensure you create and link to the same DNS Provider (credential) as your first cluster

## Cleanup

Cleanup of a cluster's zone file entries is triggered by deleting the DNS Policy that caused the records to be created.

The DNS policy will have a finalizer, when it is deleted the kuadrant operator will mark the created DNS record for deletion.

The DNS record will have a finalizer from the DNS Operator which will clean the records from the zone before removing the finalizer.

## Switching strategy
The DNSPolicy offers 2 strategies 
1) Simple (one A record multiple values)
2) LoadBalanced (leverages CNAMES and A records to provide advanced DNS based routing to be used (example GEO or Weighted) )

The strategy field will be immutable once set. To migrate from one strategy to the other you must first delete the existing policy and recreate it with a new strategy. In a multi-cluster environment, this can mean you have a period of time where 2 different strategies are set (simple on one and load balanced on another). The controller will mitigate against this in the following way:
When attempting to update the remote zone, it first does a read. If it spots existing records in that zone that correlate with a different strategy, it will not write its own records but will flag this as an error state in the status. So the records will remain in the original strategy until all policies for that listener match. This is true also if you delete all policies 


## Orphan Mitigation

It is possible for an orphan set of records to be created. This is not something we currently mitigate against directly in the controller. However, you can define health checks tht would remove these orphans from the DNS response. Additionally, there is future work that leverages a "heart beat" that should allow each alive controller to see and remove a dead controllers records.

### Health Checks
The health checks provided by the various DNS Providers will be used. However we may choose to cover this area in more detail in a subsequent RFC
[AWS](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/DNS-failover.html)
[Google DNS](https://cloud.google.com/DNS/docs/zones/manage-routing-policies#before_you_begin)
[Azure](https://learn.microsoft.com/en-us/azure/traffic-manager/traffic-manager-monitoring)

## OCM / ACM
This is very similar, with the caveat that the "cluster" is the hub cluster for this OCM and that the gateway will be a Kuadrant gatewayClass. You can think about this setup as "single cluster" with a gateway that has multiple addresses. 



# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## General Logic Overview

The general flow in the Kuadrant operator follows a single path, where it will always output a DNSRecord, which specifies everything needed for the listener host of a given gateway on the cluster, and is unaware of the concept of distributed DNS.

This DNSRecord is reconciled by the DNS Operator. In the event of a change to the endpoints in the DNSRecord resource the DNS Operator will first pull down the relevant records for that DNS name from the provider and store them in the DNSRecord status, then it will update its own endpoints in that record set and validate the record set has no "dead ends" before writing the changes back to the DNS Provider's zone. Once the remote write is successful, it will then re-queue itself to do a validation. The validation will be the same validation as previously done before the write. It will ensure that the endpoints it owns are present or removed in case of deletion. If the validation is successful, the controller will mark this in the status and will re-queue validation for ~15 minutes later (example). If validation is unsuccessful, it will re-queue the DNSRecord for validation rapidly (5 seconds for example). With each re-queue after an unsuccessful validation, it will add a random "jitter" to increase the chance it moves out of sync with other actors. Each time it re-queues, it will mark this in the status (see above). At any point, if there is a change to the on-cluster DNSRecord, this back off and validation will be reset and started again.  

### Configuration Updates
There will be a new configuration option that can be applied as runtime argument (to allow us emergency configuration rescue in case of an unexpected issue) to the kuadrant-operator:
- `clusterID`  (see below for how this is used)
  - When absent, this is generated in code (more information below). A unique string to identify this cluster apart from all other clusters.

### Changes to DNS Policy reconciler (kuadrant operator)

This will primarily need to be made aware that it will need to propagate the DNS Policy health check block into any DNS Records that it creates or updates using the provider specific fields.

It will also need to update the format of the records in the DNSRecord it creates, to make use of the new GUID and clusterID when formatting the record names and targets.

### DNS Operator

This component takes the DNSRecord output by the kuadrant-operator and ensure they are accurately reflected in the DNS Provider zone and currently assumes the DNS Record is the source of truth and always ensures that the zone in the DNS Provider matches the local DNS Record CR.

There are several changes required in this component:

#### Generate a ClusterID

The cluster will need a deterministic manner to generate an ID that is unique enough to not clash with other clusterIDs, that can be regenerated if ever required.

This clusterID is used as a prefix to identify which A/CNAME records were created by the local cluster

The suggestion [here](https://groups.google.com/g/kubernetes-sig-architecture/c/mVGobfD4TpY/m/nkdbkX1iBwAJ) is to use the UID of the `kube-system` namespace, as this [cannot be deleted easily](https://groups.google.com/g/kubernetes-sig-architecture/c/mVGobfD4TpY/m/lR4SnaqpAAAJ) - so is practically speaking indelible.

We can take the UID of that namespace and apply a hashing algorithm to result in 6-7 character code which can be used as the clusterID for the local cluster.

#### Update GUID Logic

Currently the DNS Operator builds the LB CNAME with a guid based on the name of the gateway, as this could potentially be different on different clusters, this will need to be updated to generate based on the zone's root hostname, to ensure it is consistently generated on all clusters that interact with this zone.

#### Ensure local records are accurate

When the local DNSRecord is updated, the DNS Operator will ensure those values are present in the zone by interacting directly with the DNS Provider - however it will now only remove existing records if the name contains the local clusters clusterID, it will also remove CNAME values, if the value contains the local clusters clusterID. The DNS Operator will store the record set for a given DNS name in the status of that DNSRecord.

Whenever the DNS Operator does a write (see above) it will re-queue the DNSRecord to verify the results are present. This verification only reconcile is identified when the observedGeneration and Generation are equal, if validation fails a back-off will be used plus jitter (additional random time). 

If the spec of the DNSRecord is changed locally, this will reset the validation.

#### Cleanup

When a DNS Policy is marked for deletion the kuadrant operator will delete all relevant kuadrantRecords.

Whenever a deleted kuadrantRecord is reconciled by the DNS Operator, it will remove any relevant records from the DNS Provider (i.e. any record with the clusterID prefix in the name. and any target with the clusterID in the target) for the deleted host.

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

#### Routing Strategies

There are currently 2 routing strategies "simple" and "loadbalanced".

##### Simple
The simple strategy is also compatible with ExternalDNS which makes it useful in some use-cases. This strategy outputs a very simplistic zone file and will only be functional with a single cluster.

##### Loadbalanced
This strategy outputs a DNS record tree with various levels of CNAMEs to allow for geo loadbalancing as well as loadbalancing within a geo and is compatible with multiple clusters.

This can be seen here: ![Load Balanced DNS Records](0008-distributed-DNS-assets%2F0008-loadbalanced-DNS-records.jpg)


#### Rate Limiting

TODO how we handle the response (requeue with back off)

API usage limits on DNS providers:
- [Google](https://cloud.google.com/service-usage/docs/quotas) - 240 read requests per minute, 60 writes per minute
- [Azure](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/request-limits-and-throttling) - 60 lists and 1000 gets per minute, 40 writes per minute
- [Route53](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/DNSLimitations.html) - 5 API requests per second read or write
- No health checks for private clusters

#### Metrics

Some useful metrics have been determined:
- Emit a metric whenever the DNS Operator writes to the zone.
- Emit a metric whenever the kuadrant operator updates the kuadrantZone spec.
- Emit a metric whenever the DNS Operator deletes an entry from the zone.
- Emit a metric whenever the DNS Operator prunes a shared record from the zone.

# Drawbacks
[drawbacks]: #drawbacks

clean up in disaster scenario for multi-cluster:
- any cluster nuked without time to cleanup will never be cleaned from the zone.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- It is the most robust, and can handle the failure of any cluster
- It provides the most valuable and robust DNS health checking network
- Allows MGC without OCM/ACM
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

## Strategy Check

It would be possible to have the controller write the strategy to txt record for the root host. `a.b.com:strategy:simple` . We could then implement a pre-write check. If this txt record was set to a conflicting strategy, other controllers would not write their changes but instead would report this in their status. It would be up the end user then to decide to either change the conflicting policies or delete the simple policy and allow a load balanced policy to take over. 

## Heartbeat Check

It may turn out that disappearing clusters create more headaches than currently anticipated, and in this scenario we could respond with having each cluster regularly update a heartbeat TXT field (say every 5 minutes), and on this timer also check all other clusters heart beats, if some are falling old then all the records related to that heartbeat's clusterID can be removed.

## Migration from single / multi cluster to OCM/ACM

Once this work has completed, it's possible that there is a migration path apparent from non-ocm to ocm without having to start from scratch, e.g. convert back to a single cluster > install OCM hub to that cluster > convert other clusters into spokes.

## Integration with local DNS servers (e.g. CoreDNS)

To further improve the simple demo of the MGC product, making the clusters work as nameservers removes the requirement for any DNS providers, and further simplifies the local development environment setup, so that the features of MGC can be explored without requiring excessive steps from the user.
