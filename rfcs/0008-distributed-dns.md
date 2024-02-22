# Distributed DNS Load Balancing

- Feature Name: Distributed DNS Load Balancing
- Start Date: 2024-01-17
- RFC PR: [Kuadrant/architecture#0008](https://github.com/Kuadrant/architecture/pull/55)
- Issue tracking: [Kuadrant/architecture#0008](https://github.com/Kuadrant/architecture/issues/56)

# Summary
[summary]: #summary

Enable the DNS Operator to manage DNS for one or more hostnames shared across one or more clusters.  Remove the requirement for a central "hub" cluster or multi-cluster control plane.

# Motivation
[motivation]: #motivation

Currently, the DNS operator has limited functionality for a single cluster, and none for multiple clusters without OCM. Having the DNS operator enabled to manage DNS in these situations significantly eases onboarding users to leveraging DNSPolicy across both single and multi-cluster deployments.

# Diagrams
[diagrams]: #diagrams

- ![Distributed DNS](0008-distributed-dns-assets%2F0008-distributed-dns.jpg)
- ![Load Balanced DNS Records](0008-distributed-dns-assets%2F0008-loadbalanced-dns-records.jpg)

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Single Cluster and Multiple Cluster
Every cluster requires a DNS Policy configured with a provider secret. Every zone will be treated as a distributed DNS zone. In each cluster, when a matching listener host defined in the gateway is found that matches one of the available zones in the provider the addresses from that gateway will be appended to the zone file - in a non-destructive manner - allowing multiple clusters to all add their gateways addresses to the same zone file, without destroying each other's values.

### Health Checks
The health checks provided by the various DNS Providers will be used.

## OCM / ACM
This is very similar, with the caveat that the "cluster" is the hub cluster for this OCM and that the gateway will be a Kuadrant gatewayClass.

## Migrating from single/multiple clusters to OCM

This is not covered.

## Cleanup

Cleanup of a cluster's zone file entries is triggered by deleting the DNS Policy that caused the kuadrantRecords to be created.

The DNS policy will have a finalizer, when it is deleted the controller will mark the created kuadrantRecords for deletion.

The kuadrantRecords will have a finalizer from the DNS Operator which will clean the records from the zone before removing the finalizer.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Terminology
 - zone / DNS Provider zone: The set of records listed in the DNS Provider (the live list of DNS records)
 - KuadrantRecord: The DNSRecord created on the local cluster by Kuadrant - only contains the DNS requirements for the local cluster
 - ZoneRecord: The DNSRecord created on the local cluster by the DNS Operator, this reflects the DNSRecord as it appears in the zone.
 - Root Host: The listener on a gateway that caused the Kuadrant Operator to create a KuadrantRecord (i.e the host that the users will be interacting with)

## General Logic Overview

The general flow in the Kuadrant operator follows a single path, where it will always output a kuadrantRecord, which specifies everything needed for this workload on this cluster, and is unaware of the concept of distributed DNS.

This kuadrantRecord is reconciled by the DNS Operator, the DNS Operator will act with the DNS Provider's zone and ensure that the records in the zone relevant to the hosts defined within gateways on it's cluster are present. When cleaning up a DNS Record the operator will ensure all records owned by the policy for the gateway on the local cluster are removed, then the DNS Operator will also prune unresolvable records (an unresolvable record is a record that has no logical value such as an IP Address or a CNAME to a different root hostname) related to the hosts defined in the kuadrantRecords.

### Configuration Updates
There will be a new configuration option that can be applied as runtime argument (to allow us emergency configuration rescue in case of an unexpected issue) to the kuadrant-operator:
- `clusterID`  (see below for how this is used)
  - When absent, this is generated in code (more information below). A unique string to identify this cluster apart from all other clusters.

### Changes to DNS Policy reconciler (kuadrant operator)

This will primarily need to be made aware that it will need to propagate the DNS Policy health check block into any DNS Records that it creates or updates using the provider specific fields.

It will also need to update the format of the records in the KuadrantRecord, to make use of the new GUID and clusterID when formatting the record names and targets.

### DNS Operator

This component takes the kuadrantRecords output by the kuadrant-operator and ensure they are accurately reflected in the DNS Provider zone and currently assumes the DNS Record is the source of truth and always ensures that the zone in the DNS Provider matches the local DNS Record CR.

There are several changes required in this component:

#### Generate a ClusterID

The cluster will need a deterministic manner to generate an ID that is unique enough to not clash with other clusterIDs, that can be regenerated if ever required.

This clusterID is used as a prefix to identify which A/CNAME records were created by the local cluster

The suggestion [here](https://groups.google.com/g/kubernetes-sig-architecture/c/mVGobfD4TpY/m/nkdbkX1iBwAJ) is to use the UID of the `kube-system` namespace, as this [cannot be deleted easily](https://groups.google.com/g/kubernetes-sig-architecture/c/mVGobfD4TpY/m/lR4SnaqpAAAJ) - so is practically speaking indelible.

We can take the UID of that namespace and apply a hashing algorithm to result in 6-7 character code which can be used as the clusterID for the local cluster.

#### Update GUID Logic

Currently the DNS Operator builds the LB CNAME with a guid based on the name of the gateway, as this could potentially be different on different clusters, this will need to be updated to generate based on the zone's root hostname, to ensure it is consistently generated on all clusters that interact with this zone.

#### Ensure local records are accurate

When the local KuadrantRecord is updated, the DNS Operator will ensure those values are present in the zone by interacting directly with the DNS Provider - however it will now only remove existing records if the name contains the local clusters clusterID, it will also remove CNAME values, if the value contains the local clusters clusterID.

Whenever the DNS Operator does a create or update on the zone (but not a delete), it will requeue the KuadrantRecord for TTL / 2, to verify the results are present. This is identified when the observedGeneration and Generation are equal, we should list the zone and ensure all expected records are present. If they are absent, add them and requeue to verify again.

#### Cleanup

When a DNS Policy is marked for deletion the kuadrant operator will delete all relevant kuadrantRecords.

Whenever a deleted kuadrantRecord is reconciled by the DNS Operator, it will remove any relevant records from the DNS Provider (i.e. any record with the clusterID prefix in the name. and any target with the clusterID in the target) for the deleted kuadrantZone's root host.

#### Prune dead branches from DNS Records

If a KuadrantRecord is deleting, there is the potential that we are leaving behind a dead branch, which will need to be recursively cleaned until none remain.

What is a dead branch? If a CNAME exists whose value is a record that does not exist, this CNAME will not resolve, as our DNS Records are structured similarly to a tree, this will be referenced as a "dead branch".

The controller will need to work through the DNS Record and ensure that from hostname to A (or external CNAME) records is structurally sound and that any dead branches are removed, to do this, the DNS Operator will need to first read what is published in the provider zone:

#### Build ZoneRecord from the provider zone

When the DNS Operator has finished reconciling a deleting kuadrantRecord it will pull the most recent records from the DNS Provider and construct a separate zoneRecord for the same root host as the deleting kuadrantRecord.

#### Prune the zoneRecord

It will recursively analyse the zoneRecord and ensure that it is structurally sound and that all records can be resolved to at least one external CNAME or A record.

If during this pruning period the zoneRecord is modified, then these modifications will need to be sent to the DNS Provider to clean the zone.

Once completed, the zoneRecord can be deleted (this may be possible to only reflect in memory completely).

#### Removing the finalizer from kuadrantRecords

When a kuadrantRecord is reconciled, the DNS Operator should apply a finalizer to it.

When a kuadrantRecord is being removed, the following must be successfully completed before the finalizer can be removed:
- Remove the local clusters records and targets from the relevant zone
- Perform a prune on the relevant root host
- Apply the results of the prune to the DNS Provider

Only once these actions have all resolved should the finalizer be removed, and the kuadrantRecord allowed to be deleted.

Should a deleted DNS Record be recreated, or a deleted DNS Policy be restored, all of the deleted records will be restored by the DNS Operator, at that point.

#### Aggregating Status

When the DNS Operator performs actions based on the state of a kuadrantRecord, it should update the kuadrantRecord status with results of these actions, similar to how it is currently implemented.

When the Kuadrant operator sees these statuses on the kuadrantRecords related to a DNS Policy, it should aggregate them into a consolidated status on the DNS Policy status.

It will be possible from this DNS Policy status to determine that all the records for this cluster, related to this DNS Policy, have been accepted by the DNS Provider.

#### Routing Strategies

There are currently 2 routing strategies "simple" and "loadbalanced".

##### Simple
The simple strategy is also compatible with ExternalDNS which makes it useful in some use-cases. This strategy outputs a very simplistic zone file and will only be functional with a single cluster.

##### Loadbalanced
This strategy outputs a DNS record tree with various levels of CNAMEs to allow for geo loadbalancing as well as loadbalancing within a geo and is compatible with multiple clusters.

This can be seen here: ![Load Balanced DNS Records](0008-distributed-dns-assets%2F0008-loadbalanced-dns-records.jpg)

##### Migration

These 2 strategies are not compatible, and as such the RoutingStrategy field will be set to immutable, requiring the deletion and recreation of the DNS Policy in order to update this value. This will result in the related records being removed from the zone before being created in the new format.

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

API usage limits on DNS providers:
- [Google](https://cloud.google.com/service-usage/docs/quotas) - 240 read requests per minute, 60 writes per minute
- [Azure](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/request-limits-and-throttling) - 60 lists and 1000 gets per minute, 40 writes per minute
- [Route53](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/DNSLimitations.html) - 5 API requests per second read or write
- No health checks for private clusters

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

## Heartbeat Check

It may turn out that disappearing clusters create more headaches than currently anticipated, and in this scenario we could respond with having each cluster regularly update a heartbeat TXT field (say every 5 minutes), and on this timer also check all other clusters heart beats, if some are falling old then all the records related to that heartbeat's clusterID can be removed.

## Migration from single / multi cluster to OCM/ACM

Once this work has completed, it's possible that there is a migration path apparent from non-ocm to ocm without having to start from scratch, e.g. convert back to a single cluster > install OCM hub to that cluster > convert other clusters into spokes.

## Integration with local DNS servers (e.g. CoreDNS)

To further improve the simple demo of the MGC product, making the clusters work as nameservers removes the requirement for any DNS providers, and further simplifies the local development environment setup, so that the features of MGC can be explored without requiring excessive steps from the user.
