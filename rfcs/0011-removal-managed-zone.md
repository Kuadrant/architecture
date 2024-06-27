# RFC Template

- Feature Name: `removal-managed-zone`
- Start Date: 2024-06-26
- RFC PR: [https://github.com/Kuadrant/architecture/pull/93](https://github.com/Kuadrant/architecture/pull/93)
- Issue tracking: [https://github.com/Kuadrant/dns-operator/issues/166](https://github.com/Kuadrant/dns-operator/issues/166)

# Summary
[summary]: #summary

Originally the `ManagedZone` API was designed in the context of multiple tenants where the DNS operator was offered as a service and the owner of the service would own the credentials, the dns provider accounts and the zones being worked on. This is no longer the case, however `ManagedZone` has remained in place. It has now become mainly just a reference to a credential to the underlying DNS provider. We do not need this indirection and it provides little to no value to the end user. So we will remove the `ManagedZone API` and replace it with an update to the DNSPolicy and DNSRecord API to add a local reference to a secret containing the needed credentials and any additional configuration. As part of this change we will also bump the version number of the DNSPolicy API to v1alpha2. 

# Motivation
[motivation]: #motivation

The `ManagedZone` API is required in order to create a DNSPolicy. Its original purpose and value is no longer present. It now creates a barrier and additional learning curve for users. We want to improve this user experience while also simplifying our object model and controller code base.

The use case here is to avoid having to create additional none policy Kuadrant APIs in order to be able to create an actual policy API. 

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

You will no longer need or be able to create a `ManagedZone` resource. Instead you will create a credential in a Kubernetes secret, but will reference that secret directly from the DNS policy. Any additional provider specific configuration such as the zone ids to work with will be added to the provider credential.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The `ManagedZone` API will be removed. All tests will be updated/removed as needed. We will rename the `managedZoneRef` property to be `providerRef` that will be a required field and will accept the name of a secret in the same namespace. We will expect zone ids to be specified in the provider secret (this is the main piece of configuration in the existing zone resource ). These zone ids will limit what zones the controller will interact with and manage records within. 

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

None 

# Future possibilities
[future-possibilities]: #future-possibilities

We may extend the providerRef property in the future to allow it to use a selector so that you can have multiple provider secrets and have the records populated into multiple providers.