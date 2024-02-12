# Distributed DNS Load Balancing

- Feature Name: Distributed DNS Load Balancing
- Start Date: 2024-01-17
- RFC PR: [Kuadrant/architecture#0008](https://github.com/Kuadrant/architecture/pull/55)
- Issue tracking: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/issues/0000)

# Summary
[summary]: #summary

Enable the DNS Operator to manage DNS for one or more hostnames shared across one or more clusters, where a central "hub" cluster is not required or may not be available, and increase the robustness of the DNS Health checks by executing them from all the available clusters.

# Motivation
[motivation]: #motivation

Currently, the DNS record controller has limited functionality for a single cluster, and none for multiple clusters without OCM. Having the DNSRecord controller enabled to manage DNS in these situations significantly eases onboarding users to the advantages of the DNSRecord controller.

# Diagrams
[diagrams]: #diagrams

- [Overview](./0008-distributed-dns-assets/distributed-dns-usecases.jpg)
- [P2P Health Checks](./0008-distributed-dns-assets/distributed-dns-healthchecks.jpg)
- [DNS Pruning](./0008-distributed-dns-assets/distributed-dns-pruning.jpg)
- [DNS Controller](./0008-distributed-dns-assets/distributed-dns-dns-operator.jpg)

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Single Cluster
In a single cluster scenario, create a DNS Policy that references a DNS provider. When a listener host defined in the gateway is found that matches one of the available zones in the provider, the addresses of that gateway will be created as A or CNAME records for that host in the appropriate zone file following the configuration outlined in the DNS Policy.

### Single Cluster Health Checks
If health checks are defined in the DNS Policy, these will also be created and used to remove unhealthy IPs from the Zone file, should all health checks fail, then all A records will be published to avoid an NXDOMAIN response.

## Multiple Cluster
In a multiple cluster scenario, every cluster requires a DNS Policy configured with a provider secret that specifies a unique `multiClusterID`, they should all reference the same zone(s) in the same DNS Provider for the same host. In each cluster, when a matching listener host defined in the gateway is found that matches one of the available zones in the provider the addresses from that gateway will be appended to the zone file - in a non-destructive manner - allowing multiple clusters to all add their gateways addresses to the same zone file, without destroying each other's values.

### Multiple Cluster Health Checks
If health checks are defined in the DNS Policy, these will be created on all clusters, there are then 2 types of health checks:
1) Local Health Checks - used to check the local gateway's IP addresses are responding for the expected host.
2) Remote Health Checks - used to check that the other IP addresses in the zone file are responding for that zones host.

When a local health check is failing, this IP will not be published to the zone. If it has already been published it will be removed only if there is more than one available address

When a remote health check is failing a TXT record is added/amended in the zone to note the failure - should the percentage of failing clusters exceed the `MultiClusterFailureThreshold` (configured in the DNS Policy), that unhealthy IP is removed from the zone by any cluster with an unhealthy check on that IP.

If all local and remote health checks for a particular host are marked as unhealthy, all known IPs are published, to avoid an NXDOMAIN response.

## OCM / ACM
This is very similar to a single cluster, with the caveat that the "single" cluster is the hub cluster for this OCM. The primary difference is that the gateway will be a Kuadrant gatewayClass.

### OCM / ACM Health Checks
This is also the same as single cluster.

## Migrating from Single cluster to Multiple cluster without OCM

In the single cluster: 
- Edit all relevant provider secrets and set:
  - `multiClusterID` to some unique value

In the additional clusters:
- Copy the DNS Policies and provider secrets from the single cluster to any additional clusters, and update the provider secrets with unique `multiClusterID` values.
- (optional) Create a gateway and workload which uses the same hosts as the single cluster.

## Migrating from single/multiple clusters to OCM

This is not covered.

## Cleanup

Cleanup of a cluster's zone file entries is triggered by deleting the DNS Policy that caused the DNS Records to be created.

The DNS policy will have a finalizer, when it is deleted the controller will mark the created DNS Records for deletion.

The DNS Records will have a finalizer from the DNS Operator which will clean the records from the zone before removing the finalizer.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## General Logic Overview

The general flow for in the Kuadrant operator follows a single path, where it will always output a DNS Record which specifies everything needed for this workload on this cluster, and is unaware of the concept of distributed DNS, or phrased more simply:
Build a DNS Record (let's refer to this as a kuadrantRecord) based on the local gateways and DNS Policy.

This kuadrantRecord is reconciled by the DNS Operator, where there are 2 possible routes:

Where a multiClusterID is specified in the relevant ProviderSecret the DNS Operator will act with the DNS Provider's zone as the authority and ensure that the records in the zone relevant to the hosts defined within gateways on it's cluster are accurate. When cleaning up a DNS Record in this scenario the operator will ensure all records owned by the policy for the gateway on the local cluster are removed and then prune unresolvable records related to the hosts defined in the gateway from the DNS Provider zone.

Where a multiClusterID is not specified the DNS Operator will act with the DNS Record as the authority and ensure the entire host is updated in the zone to exactly match the specified DNS Record when cleaning up in this scenario, the host is fully removed from the DNS Provider zone.

### API Updates
New configuration options in the providerSecret:
- `multiClusterID`
  - When absent the DNS Operator will consider itself the owner of the zone and the local DNS Records as the authority.
  - When present the DNS Operator will consider the zone the authority and only insert/update any missing records from the local DNS Records.
  - This value will be hashed whenever it is used in the zone file i.e. in the A/CNAME records and also in the health check TXT record)
  - Used when generating an A record, e.g. `<hash>-<multiClusterID>.<lb-id>.<host...>`
- `RefreshInterval`
  - When absent it defaults to the shortest TTL of created records. This is how often in seconds to reconcile the DNS Policy in order to pull fresh zone values from the DNS Provider.

New fields added to the DNS Policy CRD: 
- `HealthCheck.MultiClusterFailureThreshold`
  - When absent defaults to `100`, what percentage of clusters need to be reporting an endpoint unhealthy on a remote IP for it to be removed from the zone by a cluster that does not own that IP.

New fields added to the DNS Record CRD:
- `HealthCheck`
  - Propagated from the DNS Policy, to facilitate reconciling health checks in the DNS controller.
- `ProviderSecret`
  - Propagated from the DNS Policy to facilitate pulling the zone in the DNS controller.

### Changes to DNS Policy reconciler (kuadrant operator)

This will primarily need to be made aware of the new fields on the DNS Policy (Healthcheck block and providerSecret field) that it will need to propagate into any DNS Records that it creates or updates. Otherwise the logic of this controller remains largely the same.

### DNS Operator

A new component that will deal with taking the DNS Records output by the DNSPolicy reconciler and ensure they are accurately reflected in the DNS Provider zone.

This component should already exist, but in a fashion that assumes the DNS Record as the source of truth and always ensures that the zone matches the local DNS Record CR.

There are several changes required in this component:

#### Build DNS Records from the provider zone

When the DNS Operator reconciles a DNS Record (kuadrantRecord) it will first look to the referenced providerSecret to see if a multiClusterID is set, if so the operator will first pull the most recent records from the DNS Provider and construct a separate DNS Record which accurately reflects the state of the zone in the provider (zoneRecord).

As this could result in many requests to the DNS Provider API a mechanism is required to reduce this to only request when required:

#### Update GUID Logic

Currently the DNS Operator builds the LB CNAME with a guid based on the name of the gateway, as this could potentially be different on different clusters, this will need to be updated to generate based on the zone's root hostname, to ensure it is consistently generated on all clusters that interact with this zone.

#### Testing if a DNS Provider pull is Required

When the DNS Operator publishes any changes to a zone, it will also add/update a (`kuadrant.last_modified_at`) TXT record with the current timestamp.

If the DNS Operator has no local zoneRecords for a zone (in a providerSecret with a multiClusterID), it will list and build the zoneRecords.

If the DNS Operator already has zoneRecords it will first execute a GET request for the `kuadrant.last_modified_at` TXT record, if the value of that field is more recent than the zoneRecords lastModified time, then the DNS Operator will list and rebuild it's zoneRecords.

#### Ensure local records are accurate

Once the zoneRecord is created / updated on cluster, the operator will then scan through this DNS Record and remove any records that contain the local multiClusterID stored in the status of the DNS Record (more info on this later). Then inject the values from the kuadrantRecord into the zoneRecord. After this the DNS Operator will sanity check the result and prune any bad records (we will expand on that later). Once the zoneRecord is updated and sanity checked, if it has changed then the result is sent back to the DNS Provider so that the zone can be updated and made accurate (with an added update to the (`kuadrant.last_modified_at`) TXT record.

#### Update response to failing health check 

When a health check fails, there are 2 scenarios that need to be considered:

##### Local Health Check Failing

In this scenario, the cluster will immediately remove the IP from the DNS Record, as it is a local workload that is not functional.

##### Remote Health Check Failing

After ensuring its own hashed multiClusterID is present in the relevant TXT record, the controller will count the totalClusters which is the sum of unique multiClusterIDs in that hosts A records. Then it will count the reportingClusters which is the number of multiClusterIDs in the relevant TXT record, if the reportingClusters/totalClusters >= HealthCheck.MultiClusterFailureThreshold then this IP will be removed for this host.

example TXT record:
```
myapp.pb.apps.com/24.34.45.56: ghye7d34; lhoyg391
```

As this TXT record can be quite long (when many clusters are present), if this fields exceeds a DNS Record limit (255 characters), it will run into a second TXT record with the same key.

Example TXT record:
```
myapp.pb.apps.com/24.34.45.56: ghye7d34; lhoyg391; ... <more cluster ids>
myapp.pb.apps.com/24.34.45.56: hir01fcd; aoe8912;
```

##### Handling no IPs published

If after processing the health checks the result is a DNS Record with no A records, then all of the A records will be persisted (this is to avoid an NXDOMAIN response).

#### Regularly reconcile DNS Records

As changes to the zone file can happen without triggering an event in the DNS Operator, the operator will need to poll the DNS Provider in order to ensure that it has an accurate set of records to ensure that changes in the zone are not missed. e.g. a cluster would not have health checks for any clusters that were added since it last read the zone.

This interval is defined in the ProviderSecret `RefreshInterval` and defaults to the shortest TTL it is using in the DNS records. This is modifiable to handle a scenario where the DNS Provider is rate-limiting the requests due to being polled too frequently.

This interval will also have a slight random shudder applied to it at every requeue event (i.e. between -5% and 5% delay), to avoid multiple clusters managed through a config manager like ArgoCD all hitting the API within milliseconds of each other.

examples:
- An operator with a 60 second refresh interval, every requeue will be scheduled for a value between 57 and 63 seconds.
- An operator with a 120 second refresh interval every requeue will be scheduled for a value between 114 and 126 seconds.

#### Prune dead branches from DNS Records

What is a dead branch? If a CNAME exists whose value is a record that does not exist, this CNAME will not resolve, as our DNS Records are structured similarly to a tree, this will be referenced as a "dead branch".

Once the DNS Record has been reconciled, there is a possibility that dead branches might exist due to health checks or clusters being removed, in this case the controller will need to work through the DNS Record and ensure that from hostname to A records is structurally sound and that any dead branches are removed.

An exception, as usual, is that if all records would be removed as the result of pruning then the DNS Record should be left unsaved, unless the DNS Record is trying to delete, in which case the records can be removed from the provider entirely.

#### Handle Adding/Changing multiClusterID

When a DNS Record is created, the cluster ID from the provider secret is set in the status of the DNS Record. If a current value exists and is different, then the records from the zone that used the existing value in the status are removed, the records created with a new multiClusterID are set, and the value in the status is updated.

# Drawbacks
[drawbacks]: #drawbacks

clean up in disaster scenario for multi-cluster:
- all clusters nuked
- more than `MultiClusterFailureThreshold` are nuked

API usage limits on DNS providers:
- [Google](https://cloud.google.com/service-usage/docs/quotas) - 240 read requests per minute
- [Azure](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/request-limits-and-throttling) - 60 lists and 1000 gets per minute
- [Route53](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/DNSLimitations.html) - 5 API requests per second

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- It is the most robust, and can handle the failure of any cluster
- It provides the most valuable and robust DNS health checking network
- Without this solution MGC cannot be used without OCM/ACM
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

Once this work has completed, it's possible that there is a migration path apparent from non-ocm to ocm without having to start from scratch, e.g. convert back to a single cluster, install OCM hub to that cluster and convert other clusters into spokes.

## Integration with local DNS servers (e.g. CoreDNS)

To further improve the simple demo of the MGC product, making the clusters work as nameservers removes the requirement for any DNS providers, and further simplifies the local development environment setup, so that the features of MGC can be explored without requiring excessive steps from the user.