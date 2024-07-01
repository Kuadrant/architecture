# RFC Template

- Feature Name: `removal-managed-zone`
- Start Date: 2024-06-26
- RFC PR: [https://github.com/Kuadrant/architecture/pull/93](https://github.com/Kuadrant/architecture/pull/93)
- Issue tracking: [https://github.com/Kuadrant/dns-operator/issues/166](https://github.com/Kuadrant/dns-operator/issues/166)

# Summary
[summary]: #summary

The `ManagedZone` API was designed in the context of multiple tenants where the DNS operator was offered as a service and the owner of the service would own the credentials, the dns provider accounts and the zones being worked on. It had a requirement of adding and removing zones in the provider dynamically. This is no longer the case, but the `ManagedZone` API has remained in place and is mainly a reference to a credential for the underlying cloud DNS provider and some additional configuration such as the zone id and the root domain in that zone. We do not need this indirection and there is little value to the end user. So we will remove the `ManagedZone API` and replace it with an update to the DNSPolicy and DNSRecord API to add a label selector to secrets in the same namespace containing the needed credentials and the needed additional configuration. As part of this change we will also bump the version number of the DNSPolicy API to v1alpha2. 

# Motivation
[motivation]: #motivation

The `ManagedZone` API is required in order to create a DNSPolicy. Its original purpose and value is no longer present. It now creates an additional barrier for setting up a `DNSPolicy` that was never intended or really needed once we moved away from offering dns operator as a service. We want to improve this user experience while also simplifying our object model and controller code base.

The key user impact here is to avoid users having to create additional none policy `Kuadrant APIs` in order to be able to leverage the `DNSPolicy` API. 

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

You will no longer need or be able to create a `ManagedZone` resource. It will be expected that the zone already exists in your provider of choice. Now you will only create a credential via a Kubernetes secret. This secret will be where any additional configuration is also applied such as the zone id to target and the root domain in that zone to use. Removing the managed zone API will reduce API calls to the DNS provider and reduce events and load on the server and kuberentes API server.
You will no longer be limited to a single root domain or single provider per policy. The managed zone was a single reference, however we want to move to leverage a label selector which will mean multiple credentials can be used for a single policy. 

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The `ManagedZone` API will be fully removed. All tests will be updated/removed as needed. We will rename the `managedZoneRef` property to be `providerSelector` that will be a required field and will accept a label selector for secrets within the same namespace as the policy resource. We will expect the zone id and root domain to be specified in the provider secret, these will be required. These zone ids will limit what zones the controller will interact with and manage records within. 

## Secret structure

under the data field we will expect two new fields to be present

```

ZONE_ID: somezone
DOMAIN_NAME: *.a.b.com

```

If these fields are not present the status of the DNSPolicy will be updated to reflect the issue (more on status below). Only if these fields are present and listeners that match the `DOMAIN_NAME` are present will DNSRecords be created. If no record is created for a given secret, this will be considered a valid configuration (listeners may be added after a secret has been created and we shouldn't assume an error in this case).


## Secret label selector

The reason for using a label selector rather than a direct reference are as follows:

- It will allow a single DNSPolicy to be applied to multiple dns providers (specific configuration being now present in the secret)
- It will allow for multiple root domains (IE *.b.com and *.a.com can exist on a single gateway) to be used and controlled by DNSPolicy (currently not possible with managed zone)

Each secret discovered will result in a separate DNSRecord for any targeted listener in the gateway that matches or is a subdomain of the DOMAIN_NAME field specified in the secret using a longest prefix match strategy. Each DNSRecord will have a direct reference to the secret that resulted in its creation. 

## Secret deletion / modification

When a secret is deleted there will be no action taken by the DNS Controller. When a secret is updated the controller will trigger a reconcile of the policy. 

If the DNS controller fails to access a provider or fails to create the records due to the zone being absent for example, this failure will be reflected in the DNSRecord and piped back to the DNSPolicy status (below). 

## DNSPolicy Status updates

Each provider secret discovered will have a provider condition in the DNSPolicy reflecting each DNSRecords under the existing `recordConditions` section:

It will look something like:

```yaml
Type: ProviderAccessAvailable
Status: True
Message: Access to the dns provider (provider name) is active and available.
Reason: ProviderAccessAvailable
LastTransitionTime: time
```

This is also where missing or failed validation of a secrets fields will be reflected.

# Drawbacks
[drawbacks]: #drawbacks

We lose the managed zone API which currently allows you to create and delete zones via the resource. We don't see this as really being a bad thing as it is probably to easy to shoot yourself in the foot with this type of API. 

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- This design simplifies the object model, reduces look ups needed and the code base and improves the user experience.

# Prior art
[prior-art]: #prior-art

NA

# Unresolved questions
[unresolved-questions]: #unresolved-questions

There is an unknown in how we want to handle GEO codes. Not all providers use the same codes and so setting a default GEO in the policy could result in unexpected behaviour in the case of multiple secrets targeting multiple providers. 

# Future possibilities
[future-possibilities]: #future-possibilities
