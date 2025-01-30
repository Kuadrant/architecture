# Remove Managed Zone

- Feature Name: `remove-managed-zone`
- Start Date: 2024-06-26
- RFC PR: [https://github.com/Kuadrant/architecture/pull/93](https://github.com/Kuadrant/architecture/pull/93)
- Issue tracking: [https://github.com/Kuadrant/dns-operator/issues/166](https://github.com/Kuadrant/dns-operator/issues/166)

# Summary
[summary]: #summary

ManagedZone was designed in the context of multiple tenants where the DNS operator was offered "as a service" and the owner of the service would own the credentials, the dns provider accounts and the zones being worked on. It had a requirement of adding and removing zones in the provider on demand. This is no longer the case, but the `ManagedZone` API has remained in place. It is now mainly a reference to a credential for the underlying cloud DNS provider. We do not need this additional layer of indirection. We will remove the ManagedZone API and change the DNSPolicy and DNSRecord to a localReference that defaults to a credential secret but is flexible enough to be expanded upon in the future.

# Motivation
[motivation]: #motivation

The motivation for these changes is to simplify the user experience of setting up a DNSPolicy by removing the need for to create a  `ManagedZone` resource before being able to do anything with a DNSPolicy.

The key user impact here is to avoid users having to create additional none policy `Kuadrant APIs` in order to be able to leverage the `DNSPolicy` API. 

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

There will no longer be a `ManagedZone` resource that needs to be created as a pre-requisite to setting up a DNSPolicy. Instead it will be expected that the zone already exists in the provider of choice. If it does not exist, you will see this reflected as an error state in the DNSPolicy status. Credentials will continue to be stored in secrets by default, but these secrets will also be where any future additional zone configuration is applied.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

change the DNSPolicy to have local object reference array for provider secrets in the same namespace.

```
spec:
  providerRefs
    - name: myawsroute53

```
Initially this array will have a limit of 1. The reason for an object array is we expect to want to allow multiple providers secrets to be specified in the future. This allows us to control the number of secrets and allow for possible expansion in the future.

Where we use the existing credential in the kuadrant operator we will now load a list of credentials and iterate through each of them and create a specific DNSRecord instance for the target DNS provider (again initially only 1) populating that DNSRecord with a reference to its required credential. DNSRecord will only have a single reference. The DNS Operator will pick up on these records in the same way it already does except it will load the secret directly rather than via the ManagedZone. The provider type is already specified as part of our existing secret definition:

```
type: kuadrant.io/aws
```

All of the tests will be updated/removed as needed. We will rename the `managedZoneRef` property to be `providerRefs` that will be a required field. This will be an array type with the `maxItems` set to 1 initially. 


Any issues updating/contacting a target DNS provider will be reflected in the existing status conditions.


## Secret deletion / modification

All secret events will be filtered down to secrets that have:

```
type: kuadrant.io/aws
```

When a secret is deleted the DNSRecord controller will trigger a reconcile of record resources in the same namespace. This will result in a status update on those records that reference this deleted secret informing of a missing secret, this will then automatically be reflected back into the DNSPolicy. Deleting a secret will not result in deleting the associated DNSRecords. 
Kuadrant operator does not need to do anything when a secret is deleted as it will be watching the associated DNSRecords and reflecting status back to the DNSPolicy when it changes. The Kuadrant operator's responsibility is to ensure a correct DNSRecord is constructed and then pipe that through to the DNSOperator. It does not interact with the DNSProvider.

When a secret is updated or added the DNSRecord controller will trigger a reconcile of records in the same namespace. Kuadrant operator will re-validate the domain and continue to watch for status changes to the DNSRecords. 

If the DNSRecord controller fails to access a provider or fails to create the records due to a bad credential or a specified zone being absent for example, this failure will be reflected in the DNSRecord and piped back to the DNSPolicy. 

## DNSPolicy Status updates

DNSPolicy already reflects the status conditions of each DNSRecords under the existing `recordConditions` section. So no major update is required here.


# Drawbacks
[drawbacks]: #drawbacks

We lose a definitive API for modeling a zone.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- This design simplifies the object model, reduces look ups needed and the code base and improves the user experience.

# Prior art
[prior-art]: #prior-art

NA

# Unresolved questions
[unresolved-questions]: #unresolved-questions

None

# Future possibilities
[future-possibilities]: #future-possibilities

- Expand the list of credentials to allow populating multiple DNS providers with the record set. 
