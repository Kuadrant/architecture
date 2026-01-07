# DNS Failover Proposal

author: Phil Brookes <pbrookes@redhat.com>

Github Issue: https://github.com/Kuadrant/kuadrant-operator/issues/1638

## Proposal Summary

DNS failover in Kuadrant from an existing active site(s) to an existing passive site(s). The process is manually triggered, but automatically enacted by the DNS Operator. In the case of a catastrophic failure in the primary site, a sysadmin can do the equivalent of flipping a switch to have the DNS Operator write the passive sites records to the zone and remove all of the primary sites records from the zone, with status and metrics to monitor the progress.

## Use Cases

- GEO failover: Site 1 in US, Site 2 inactive in EU: US site fails and DNS Failover is used to send traffic to the EU site
- Provider failover: Site 1 in multi-geo AWS, Site 2 in multi-geo GCP: AWS fails and DNS Failover is used to send traffic to the GCP site
- Maintenance: Site 1 in a maintenance window, Site 2 in a non-overlapping maintenance window: DNS Failover to Site 2 while Site 1 performs maintenance.
- Migration: Assign existing infrastructure to Site 1, and new site to Site 2, use failover to migrate traffic to Site 2 before retiring Site 1

## User Stories

- As an infrastructure provider I want to be able to quickly start a DNS Failover procedure to an existing passive site as part of a comprehensive failover strategy so that I can recover from a disaster in an orderly fashion.
- As an infrastructure provider I want to be able to remove all references to a non-existent site in the zone so that I can ensure no traffic is flowing to a failed site.
- As an infrastructure provider I want the trigger for the failover process to be manual and external to the site, so that I can always trigger failover when needed and ensure the recovery site is ready to receive traffic.
- As a user of the workload I want to be able to continue (post-recovery) to use the workload without having to change my workflow so that my work is uninterrupted by a disaster.

## User Experience

An infrastructure provider creates some clusters in Site A, and configures a workload on them using Kuadrant, as they intend to setup a failover site in Site B, they set the Kuadrant configuration to use the DNS Group name “Site A”, via the Kuadrant CR. They use the Kuadrant DNS CLI to set the active group to “Site A” (kuadrant dns set-active-group --provider-ref=<ref> zoneRootHost “Site A”) and the DNS records publish and traffic starts to flow to Site A. They then create clusters in Site B and configure the same workload and set the DNS Group Name to: “Site B” in the Kuadrant CR. The DNS records for this site are automatically left unpublished due to the active group being set to Site A.

The clusters in Site A function normally for a time, and eventually a disaster occurs and Site A is taken entirely offline, seeing this an infrastructure provider uses the provided Kuadrant DNS CLI to change the active group from “Site A” to “Site B” (kuadrant dns set-active-group --provider-ref=<ref> zoneRootHost “Site B”), after a few moments for the operators to process the change, all the DNS Records for Site A have been deleted from the zone and replaced with DNS Records for Site B, once propagated all traffic will be directed to Site B.

Eventually, Site A returns to functionality, and the infrastructure provider wants to revert this change so that traffic flows to Site A only again. The infrastructure provider uses the provided Kuadrant DNS CLI to change the active group from “Site B” to “Site A” (kuadrant dns set-active-group --provider-ref=<ref> zoneRootHost “Site A”) and as above the DNS zone will be updated to only have records pointing to site A.

## Implementation Overview

The DNS Operator will gain the concept of a group which will augment the concept of record ownership and be assigned to each operator at runtime. There will be an optional registry entry available in any zone which will list the active groups for that zone, if this registry entry is absent or empty all groups are considered active. If present only the groups contained in that registry entry will be considered active: The DNS Operator will also be reprogrammed with the ability to delete records and registry entries created by inactive group members when it is in an active group.

The active groups can be set and managed manually, and in some cases this may be preferred, however we will also be providing some new commands in the DNS CLI tool to facilitate the management of this registry entry.

This change will involve some changes to the DNS CLI and the DNS-Operator:

### DNS CLI Changes

The DNS CLI will need a few new commands for interacting with the active group record in a zone directly:
get-active-groups: Read out the current value of active groups
add-active-group: Ensure the presence of a group in the set of groups
remove-active-group: Ensure a group is not present in the set of existing groups

All of these commands will expect some common arguments (when not using CoreDNS, more on this shortly):
zoneRootHost: The root host of the zone(s), in order to identify which zone(s) to interact with
provider-ref: The secret with the authentication required to edit this zone(s)

Some commands will also require either the set of groups to add or remove.

The DNS CLI will need to be added to the productised builds and we will need to figure out how that can be distributed.

#### CoreDNS Considerations

If these commands are run with a provider of Route53, GCP, or Azure they will be given a provider-ref with credentials to read and write to the zone. However more consideration is required when the provider is CoreDNS.

In CoreDNS, there is a “zone” DNS Record on each primary cluster which is used to answer DNS queries, and this is where the active groups will be stored (so that the dns operator logic is the same in all scenarios). However the value in this zone will be populated by the CoreDNS plugin which will read it from the same external zone that the NAMESERVER records are set in which is not managed by the DNS Operator. As this is manually configured, the active group record in the external zone  will need to also be manually configured. 

However, if this external zone is using Route53, GCP or Azure, it is still possible to have the DNS CLI command manage this field, by creating a provider secret which grants access to that zone, and using that in combination with the DNS CLI.

### DNS Operator Changes

The DNS Operator will be updating the data that it adds to the registry to enable this feature, although this will be largely unnoticeable to our users. We will be extending the concept of ownership to include the concept of group ownership. We will also be including a reference to the targets in an endpoint that any particular owner is the owner of (i.e. if multiple owners have added multiple values to a record, the registry will note which owners added which values).

The majority of this feature will be implemented in the DNS Operator, in particular:
- Add groupID to DNS Record
- Add groupID to registry entries
- Add relevantTargets to registry entries (i.e. which of the targets for this record did this owner expect to exist)
- Reading the value of the active groups in the zone pre-reconcile
- Not publishing records when not in an active group (but still check your group is noted accurately in the zone)
- Unpublishing inactive groups records from the zone when in an active group
- Unpublishing inactive groups targets from records in the zone when in an active group
- Unpublishing inactive groups registry entries from the zone when in an active group

#### Add GroupID to DNS Records

When a DNS Operator is reconciling a DNS Record, it will set or update its GroupID in the status of the DNS Record, this has 2 purposes: 
Exposing the groupID to the user for validation and debugging
In CoreDNS the primary clusters can compare this value to the active groups when building/maintaining the zone record

#### Add GroupID to registry entries

When a DNS Operator is reconciling a DNS Record, any registry entries it creates as part of this process will now include its groupID (where one is assigned) alongside its ownerID.

#### Add Relevant Targets to registry entries

Whether or not the DNS Operator has a groupID assigned, it will now include in the registry for an endpoint the targets it wanted to ensure were present. This enables us to identify which targets are expected by: clusters in active groups, clusters in inactive groups and ungrouped clusters.

#### Reading the value of the active groups in the zone pre-reconcile

When an operator with a groupID assigned to it is reconciling a DNS Record, its first action will be to read in the active groups value from the zone in order to ascertain whether or not its own groupID is active or not.

#### Not publishing records when not in an active group

If an Operator has a groupID assigned and that group is not active, it will not interact with the zone. 

The exception here is it will check the zone for its own registry entries to ensure the groupID is accurate - doing nothing if it has no existing registry entries. This will allow us to handle the changing of a groupID from an active group to an inactive group.

#### Unpublishing Inactive Records, Targets and Registry entries

Once an Operator with a groupID has finished reconciling its own records into the zone, it will then proceed to remove all inactive entries from the zone:

For each endpoint found in the zone the operator will execute the following:
- If all owners are in an inactive group, delete the record
- If any target is only owned by inactive groups, delete the target

It will also delete all registry entries from inactive groups.

This will require some rewiring of the internals of the DNS Operator, as it currently does not have the capacity to delete records or registry entries created by a different owner.

##### CoreDNS Considerations for Unpublishing Inactive Records, Targets and Registry entries

With CoreDNS the majority of the clusters are secondary and are not taking any actions when reconciling DNS Records, other than setting the finalizer and now also adding the GroupID to the status.

The Primary clusters when reconciling a remote DNS Record will need to look at the GroupID in the DNS Record to determine whether or not it should be included in the “zone” DNS Record based on the current Active groups value set in the “zone” DNS Record by the CoreDNS plugin. 

The primary will also need to write the group ID (from the DNS Record and ignore its own runtime value - if present) and also the targets to the relevant registry entry when it is creating the record in the “zone” DNS Record.

During these reconciles is also when the primary DNS Operators will execute the above cleanup logic to clean inactive data out of the “zone” DNS Record.

## Not Included in the Proposal

It is assumed that the DNS Provider where the zone is hosted will be available, and this solution is not proposing to Failover to an alternate DNS Provider should it fail.

## Potential Future work

We should investigate exposing metrics about the various groups records in the zone, for use in a dashboard.
