# Distributed DNS Load Balancing

- Feature Name: Distributed DNS Load Balancing
- Start Date: 2024-01-17
- RFC PR: [Kuadrant/architecture#0008](https://github.com/Kuadrant/architecture/pull/55)
- Issue tracking: [Kuadrant/architecture#0008](https://github.com/Kuadrant/architecture/issues/56)

# Summary
[summary]: #summary

Enable the DNS Operator to manage DNS for one or more hostnames shared across one or more clusters.  Remove the requirement for a central "hub" cluster or multi-cluster control plane. Increase the robustness of the DNS Health checks by executing them from all the available clusters.

# Motivation
[motivation]: #motivation

Currently, the DNS operator has limited functionality for a single cluster, and none for multiple clusters without OCM. Having the DNS operator enabled to manage DNS in these situations significantly eases onboarding users to leveraging DNSPolicy across both single and multi-cluster deployments.

# Diagrams
[diagrams]: #diagrams

- ![Use Cases](0008-distributed-dns-assets%2Fdynamic-dns-use-cases.jpg)
- ![P2P Health Checks](0008-distributed-dns-assets%2Fdynamic-dns-health-checks.jpg)
- ![DNS Pruning](0008-distributed-dns-assets%2Fdynamic-dns-dns-pruning.jpg)
- ![DNS Controller](0008-distributed-dns-assets%2Fdynamic-dns-dns-controller.jpg)

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Single Cluster and Multiple Cluster
In a multiple cluster scenario, every cluster requires a DNS Policy configured with a provider secret. If a provider secret reference the same zone(s) in the same backing DNS Provider for the same host, it will be treated as a distributed DNS target. In each cluster, when a matching listener host defined in the gateway is found that matches one of the available zones in the provider the addresses from that gateway will be appended to the zone file - in a non-destructive manner - allowing multiple clusters to all add their gateways addresses to the same zone file, without destroying each other's values.

### Health Checks
If health checks are defined in the DNS Policy, these will be created and executed from the same cluster where the DNSPolicy is defined and target each cluster using that listener host name. When a health check is failing, a TXT record is added/amended in the zone to note the failure - should the percentage of failing clusters exceed the failure threshold (66%) by default but overridable via the `MultiClusterFailureThreshold`  flag on the controller  , that unhealthy IP is removed from the zone by any cluster with an unhealthy check on that IP. While 66% is a little arbitrary, it can be changed and gives a decent base line of 2 out of every 3 clusters failing before an IP is removed.

If all health checks for a particular host are marked as unhealthy, all known IPs are published, to avoid an NXDOMAIN response.

## OCM / ACM
This is very similar, with the caveat that the "cluster" is the hub cluster for this OCM and that the gateway will be a Kuadrant gatewayClass.

## Migrating from single/multiple clusters to OCM

This is not covered.

## Cleanup

Cleanup of a cluster's zone file entries is triggered by deleting the DNS Policy that caused the DNS Records to be created.

The DNS policy will have a finalizer, when it is deleted the controller will mark the created DNS Records for deletion.

The DNS Records will have a finalizer from the DNS Operator which will clean the records from the zone before removing the finalizer.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Terminology
 - zone / DNS Provider zone: The set of records listed in the DNS Provider (the live list of DNS records)
 - KuadrantRecord: The DNSRecord created on the local cluster by Kuadrant - only contains the DNS requirements for the local cluster
 - ZoneRecord: The DNSRecord created on the local cluster by the DNS Operator, this reflects the DNSRecord as it appears in the zone.

## General Logic Overview

The general flow in the Kuadrant operator follows a single path, where it will always output a local DNS Record (referred to as the kuadrantRecord) which specifies everything needed for this workload on this cluster, and is unaware of the concept of distributed DNS, or phrased more simply:
Build a DNS Record based on the local gateways and DNS Policy.

This kuadrantRecord is reconciled by the DNS Operator, the DNS Operator will act with the DNS Provider's zone as the authority and ensure that the records in the zone relevant to the hosts defined within gateways on it's cluster are accurate. When cleaning up a DNS Record the operator will ensure all records owned by the policy for the gateway on the local cluster are removed and then prune unresolvable records (an unresolvable record is a record that has no logical value such as an IP Address or a CNAME to a different root hostname) related to the hosts defined in the gateway from the DNS Provider zone.

### API Updates
New configuration options that can be applied as runtime arguments (to allow us emergency configuration rescue in case of an unexpected issue):
- `RefreshInterval`
  - When absent it defaults to the shortest TTL of created records (and can be backed off in case of encountering API rate-limits). This is how often in seconds to reconcile the DNS Records in order to pull fresh zone values from the DNS Provider.
- `MultiClusterFailureThreshold`
  - When absent defaults to `66`, what percentage of clusters need to be reporting an endpoint unhealthy on a remote IP for it to be removed from the zone by a cluster that does not own that IP.
- `clusterID`
  - When absent is generated in code (more information below). A unique string to identify this cluster apart from all other clusters.

New fields added to the DNS Record CRD:
- `HealthCheck`
  - Propagated from the DNS Policy, to facilitate reconciling health checks in the DNS controller.

### Changes to DNS Policy reconciler (kuadrant operator)

This will primarily need to be made aware of the new fields on the DNS Policy (Healthcheck block) that it will need to propagate into any DNS Records that it creates or updates. Otherwise the logic of this controller remains largely the same.

### DNS Operator

This component takes the DNS Records output by the kuadrant-operator and ensure they are accurately reflected in the DNS Provider zone and currently assumes the DNS Record is the source of truth and always ensures that the zone in the DNS Provider matches the local DNS Record CR.

There are several changes required in this component:

#### Generate a ClusterID

The cluster will need a deterministic manner to generate an ID that is unique enough to not clash with other clusterIDs, but determinstic enough that it can be regenerated if ever required.

This clusterID is used for multiple purposes:
- To identify which A/CNAME records were created by which cluster
- To see how many clusters have interacted with this zone (number of unique cluster IDs in all A/CNAME records)
- To report a failing health check
- To count how many clusters are reporting a failing health check.

The suggestion [here](https://groups.google.com/g/kubernetes-sig-architecture/c/mVGobfD4TpY/m/nkdbkX1iBwAJ) is to use the UID of the `kube-system` namespace, as this [cannot be deleted easily](https://groups.google.com/g/kubernetes-sig-architecture/c/mVGobfD4TpY/m/lR4SnaqpAAAJ) - so is practically speaking indelible.

We can take the UID of that namespace and apply a hashing algorithm to result in 6-7 character code which can be used as the clusterID for the local cluster.

#### Update GUID Logic

Currently the DNS Operator builds the LB CNAME with a guid based on the name of the gateway, as this could potentially be different on different clusters, this will need to be updated to generate based on the zone's root hostname, to ensure it is consistently generated on all clusters that interact with this zone.

#### Build DNS Records from the provider zone

When the DNS Operator reconciles a kuadrantRecord and will first pull the most recent records from the DNS Provider and construct a separate zoneRecord.

These zoneRecords will be created with the DNS Operator set as the owner, so they can be identified in the future. When reconciling a DNS Record is being reconciled by the DNS Operator, first check if it is owned by the DNS Operator and if so, do nothing - these DNS Records should never be reconciled.

As this could result in many requests to the DNS Provider API a mechanism is required to reduce this to only request when required:

#### Testing if a DNS Provider pull is Required

When the DNS Operator publishes any changes to a zone, it will also add/update a (`kuadrant.last_modified_at`) TXT record with the current timestamp.

If the DNS Operator has no local zoneRecords for a zone, it will list and build the zoneRecords.

If the DNS Operator already has zoneRecords (and their lastModified time is older than the refreshInterval, more on this later) it will first execute a GET request for the `kuadrant.last_modified_at` TXT record, if the value of that field is more recent than the zoneRecords lastModified time, then the DNS Operator will list and rebuild it's zoneRecords. The reason for this GET is some providers offer much higher limits on a GET of single record over a list of the zone (see the limits section).

If the zoneRecords lastModified time is less than the next refresh time, an API request should only be executed when the spec of the kuadrantRecord has changed (as this will likely result in a write to the API).

#### Ensure local records are accurate

Once the zoneRecord is created / updated on cluster, the operator will then scan through this zoneRecord and remove any records for the local cluster. Then inject the values from the kuadrantRecord into the zoneRecord. After this the DNS Operator will sanity check the result and prune any bad records (we will expand on that later). Once the zoneRecord is updated and sanity checked, if it has changed then the result is sent back to the DNS Provider so that the zone can be updated and made accurate (with an added update to the `kuadrant.last_modified_at`) TXT record.

#### Update to creation of health checks

Currently, health checks are created for all published IPs in a DNS Record, this is fine and should continue, however we will also want to ensure we have health checks for all health check TXT records - it is possible that a host/ip combination is both an A/CNAME and a TXT record, in this case care needs to be taken to avoid creating duplicates of the same healthcheck.

#### Update response to failing health check 

After ensuring its own ClusterID is present in the relevant TXT record, the controller will count the totalClusters which is the sum of unique clusterIDs in that hosts DNS records. Then it will count the reportingClusters which is the number of clusterIDs in the relevant TXT record, if `reportingClusters/totalClusters >= HealthCheck.MultiClusterFailureThreshold` and it is not the last IP for this host then this IP will be removed for this host.

example healthcheck TXT record:
```
myapp.pb.apps.com/24.34.45.56: ghye7d34; lhoyg391
```

As this TXT record can be quite long (when many clusters are present), if this fields exceeds a DNS Record limit (255 characters), it will run into a second TXT record with the same key.

Example multiple healthcheck TXT record:
```
myapp.pb.apps.com/24.34.45.56: ghye7d34; lhoyg391; ... <more clusterIDs>
myapp.pb.apps.com/24.34.45.56: hir01fcd; aoe8912;
```

If a failing health check results in the IP being removed, the TXT record should be persisted to ensure the existence of the health check while the IP is absent.

##### Handling no IPs published

If after processing the health checks the result is a DNS Record with no A records, then all of the A records will be persisted (this is to avoid an NXDOMAIN response). These values will be retrieved from the TXT records created for the health checks.

#### Regularly reconcile DNS Records
As changes to the zone file can happen without triggering an event in the DNS Operator, the operator will need to poll the DNS Provider in order to ensure that it has an accurate set of records. e.g. a cluster would not have health checks for any clusters that were added since it last read the zone.

This interval is defined in the ProviderSecret `RefreshInterval` and defaults to the shortest TTL it is using in the DNS records. This is modifiable to handle a scenario where the DNS Provider is rate-limiting the requests due to being polled too frequently.

If an attempted reconcile receives a rate limited response from the API then it will apply a shudder to it's requeue time to avoid this occurring again. This shudder is between -5% and 5% delay.

examples:
- An operator with a 60 second refresh interval sees a rate limit response, so requeues for a value between 57 and 63 seconds.
- An operator with a 120 second refresh interval sees a rate limit response, so requeues for a value between 114 and 126 seconds.

#### Prune dead branches from DNS Records

What is a dead branch? If a CNAME exists whose value is a record that does not exist, this CNAME will not resolve, as our DNS Records are structured similarly to a tree, this will be referenced as a "dead branch".

Once the DNS Record has been reconciled, there is a possibility that dead branches might exist due to health checks or clusters being removed, in this case the controller will need to work through the DNS Record and ensure that from hostname to A records is structurally sound and that any dead branches are removed.

An exception, as usual, is that if all records would be removed as the result of pruning then the DNS Record should be constructed from the health check TXT records, and all unhealthy endpoints published, unless the DNS Record is trying to delete, in which case the records can be removed from the provider entirely.

#### Cleanup

When a DNS Policy is marked for deletion the kuadrant operator will delete all relevant DNS Records.

Whenever a deleted DNS Record is reconciled by the DNS Operator, it will first check if a LIST is required and if so execute it as normal.

When merging the kuadrantRecord into the zoneRecord the DNS Operator will instead be ensuring that they are removed.

After removing all the local records, the DNS Operator will then perform a prune, and only in this scenario, if the zoneRecord is now empty these records are still deleted from the zone.

Once completed, the DNS Operator deletes the zoneRecord that matches the deleting kuadrantRecord, and then removes the finalizer from the kuadrantRecord.

#### Changing cluster IP/CNAME

If a cluster's IP address changes, we will need to handle removing the old value from the relevant zone. This needs to be removed from the ZoneRecord:
- The endpoints need to be removed
- Any TXT records for that IP/Host need to be removed.

#### Metrics

Some useful metrics have been determined:
- Emit a metric whenever the DNS Operator writes to the zone.
- Emit a metric whenever the kuadrant operator updates the kuadrantZone spec.
- Emit a metric whenever the DNS Operator removes an unhealthy IP.

# Drawbacks
[drawbacks]: #drawbacks

clean up in disaster scenario for multi-cluster:
- all clusters nuked
- more than `MultiClusterFailureThreshold` are nuked

API usage limits on DNS providers:
- [Google](https://cloud.google.com/service-usage/docs/quotas) - 240 read requests per minute, 60 writes per minute
- [Azure](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/request-limits-and-throttling) - 60 lists and 1000 gets per minute, 40 writes per minute
- [Route53](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/DNSLimitations.html) - 5 API requests per second read or write

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

## Migration from single / multi cluster to OCM/ACM

Once this work has completed, it's possible that there is a migration path apparent from non-ocm to ocm without having to start from scratch, e.g. convert back to a single cluster > install OCM hub to that cluster > convert other clusters into spokes.

## Integration with local DNS servers (e.g. CoreDNS)

To further improve the simple demo of the MGC product, making the clusters work as nameservers removes the requirement for any DNS providers, and further simplifies the local development environment setup, so that the features of MGC can be explored without requiring excessive steps from the user.
