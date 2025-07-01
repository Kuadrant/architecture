# DNS Cluster Aware Delegation

- Feature Name: `dns_cluster_aware_delegation`
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/issues/0000)

# Summary
[summary]: #summary

Proposal to add functionality to delegate the processing of a DNSRecord to a designated cluster or clusters in a multi cluster environment. 

# Motivation
[motivation]: #motivation

Multi cluster DNS is currently achieved by using the eventual provider DNS service (AWS Route etc ..) as a store for ownership metadata using specially created TXT records, and as a central API service that all clusters can communicate with.
Each cluster processes its own DNSRecords, becoming aware of other DNSRecords contributing to the same set of endpoints via this centrally stored data, in turn allowing it to correctly translate the DNSRecord endpoints into an appropriate API operation (Create/Update/Delete) and get to the desired state. 

In some cases, such as our current CoreDNS solution, there is no central off cluster API service that can be configured on all clusters and as such a different approach to multiple cluster discovery and reconciliation is required.

* Add a provider agnostic alternative for multi cluster dns support while still maintaining current behaviour for existing providers.
* Reduce the number of required CoreDNS instances to one, but still allow multiple if required, redundancy etc...

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Delegating Authority to Publish

Delegating authority is a concept whereby you can optionally specify that you do not want each DNSPolicy to publish its endpoints directly to a provider, but instead have a central authoritative DNSRecord be created that will take care of publishing all endpoints for a particular set of records.

This feature is controlled by the `authorityDelegation` field in the DNSPolicy spec, which has a single required field `role` which must be one of `primary` or `remote`.

- `spec.authorityDelegation` when set signals that records produced by this policy will have their endpoints published via an authoritative record
- `spec.authorityDelegation.role` signals what role this particular policy is taking, it is expected there should be at least one `primary`

### Primary Role

There should always be at least one primary, and the primary is in charge of ensuring that the authoritative DNSRecord that all publishing requests are delegated to is created.
The authoritative DNSRecord is created in the same namespace as the original DNSRecord created by the DNSPolicy and at a minimum will contain the original DNSRecords endpoints.  

### Remote Role

A remote will only have its endpoints added to an authoritative DNSRecord that already exists in the current namespace and does not initiate the creation of the authoritative DNSRecord.
If an authoritative DNSRecord does not exist, an error will be reported until such time as one does exist.

**Example 1.** DNSPolicy using authority delegation with `primary` role
```yaml
apiVersion: kuadrant.io/v1
kind: DNSPolicy
metadata:
  name: prod-web-coredns
spec:
  authorityDelegation:
    role: primary
  targetRef:
    name: prod-web-istio-coredns
    group: gateway.networking.k8s.io
    kind: Gateway
  providerRefs:
    - name: dns-provider-credentials-coredns
```

**Example 2.** DNSPolicy using authority delegation with `remote` role
```yaml
apiVersion: kuadrant.io/v1
kind: DNSPolicy
metadata:
  name: prod-web-coredns
spec:
  authorityDelegation:
    role: remote
  targetRef:
    name: prod-web-istio-coredns
    group: gateway.networking.k8s.io
    kind: Gateway
  providerRefs:
    - name: dns-provider-credentials-coredns
```

If the above example policies were created, three DNSRecord resources would exist in the target namespace, with a single DNSRecord resource (the authoritative) appearing as the owner of all endpoints in the target dns provider.

Note: While this concept is intended to work with the multi cluster concept described below, it is also possible to use this on a single cluster to combine several records together. 
This might be desirable if you have many records and want to reduce the number of API calls to a provider and also the number of owner TXT records that need to be managed.

[//]: # (mnairn: Add a diagram or something to visualise the records being produced  )

## Multi Cluster

The DNS reconciler can be made cluster aware allowing it to process DNSRecord resources on other clusters as well as its own.

Making the controller aware of other clusters is done via specially labeled secret resources containing kubeconfig data that exist in the `kuadrant-system` namespace.

The kubeconfig data should be generated from an appropriately scoped service account that exists on the target cluster.

When a cluster secret is created, the DNS controller on that cluster will automatically detect it and start watching for DNSRecord resources on that external cluster as well as its own.

**Example** Secret containing kubeconfig data
```yaml
apiVersion: v1
kind: Secret
data:
  kubeconfig: <kubeconfig data>
metadata:
  labels:
    kuadrant.io/multicluster-kubeconfig: "true"
  name: kind-kuadrant-local-2
  namespace: kuadrant-system
type: Opaque
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## CRD Provider

A new provider implementation that allows a DNSRecord resource to act as a zone record will be created. 
Endpoints from the source are injected into the target "zone" record using the same plan logic as all other providers. 
A zone record could be getting updated by many DNSRecords just like any other providers zone.

## Authority Delegation

A new API on the DNSRecord that allows a DNSRecord (and DNSPolicy) to signal that it is delegating authority of publishing to another DNSRecord. 
Internally uses the CRD provider described above to do this. 
Assigns a role to the delegation, one of "primary" or "remote" to signal what this particular records role is. 
Primary records will update/create the authoritative record on the same cluster and supply the provider credentials, remote records only update existing zone records.

## Multicluster

Updates the dns operator to allow watches for DNSRecords on different clusters, ultimately allowing a single controller to reconcile DNSRecords from multiple clusters. 
Clusters are configured via Secrets containing kubeconfig data and labelled as cluster secrets.

# Drawbacks
[drawbacks]: #drawbacks

* Adds another path through the code that isn't strictly necessary for the majority of providers.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The initial idea here was to look at a way to "push" records from multiple "remote" clusters to a central "primary" cluster that would be in charge of combining those records into a single "authoritative" record and ultimately in charge of publishing that record to the provider (CoreDNS). 
This would require all clusters to have credentials capable of writing to the "primary" cluster.

However, it appears to be a more common pattern in kubernetes projects that deal with multi cluster communication to have the "primary" clusters read from multiple "remote" clusters. 
This would also only require the "primary" to have read access, or very limited write (status only) access to each cluster.

# Prior art
[prior-art]: #prior-art

* https://istio.io/latest/docs/setup/install/multicluster/
* https://github.com/kubernetes-sigs/multicluster-runtime

# Unresolved questions
[unresolved-questions]: #unresolved-questions

* We ideally do not want to need access to secrets on the `remote` clusters from the `primary`, but we currently rely on determining the provider via the type of secret. During merging of remote records on remote clusters we might need to know what provider a particular policy is targeting in order to properly match it up to an authoritative record on the primary.

# Future possibilities
[future-possibilities]: #future-possibilities

* While this is being proposed for the purposes of improving our current CoreDNS solution, there is no reason that this can't be used by any provider and might become a better alternative for multi cluster generally.  
