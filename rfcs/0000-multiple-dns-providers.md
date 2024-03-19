# RFC Template

- Feature Name: Multiple DNS Provider Support
- Start Date: 2023-05-25
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000) #TODO update once PR submitted
- Issue tracking: [Kuadrant/multicluster-gateway-controller](https://github.com/Kuadrant/multicluster-gateway-controller/issues/189)

# Summary
[summary]: #summary

The purpose of this proposal is to add support for additional DNS providers, namely [Google Cloud DNS](https://cloud.google.com/dns), [Azure DNS](https://learn.microsoft.com/en-us/azure/dns/dns-overview) and [CoreDNS](https://coredns.io/) (for on-prem support and/or local development).

# Motivation
[motivation]: #motivation

Currently the [multi-cluster gateway controller](https://github.com/Kuadrant/multicluster-gateway-controller) (MGC) supports AWS Route53 as its only DNS provider. Having support for additional DNS providers will give end users greater flexibility as they will have the option to choose from a range of providers to best suit their use case while also avoiding vendor lock-in to AWS.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Overview

The [multi-cluster gateway controller](https://github.com/Kuadrant/multicluster-gateway-controller) (MGC) currently has three core features that it requires of any new DNS provider in order to offer full support:
- DNSRecord management
- Zone management
- DNS Health checks

However, we do not want to limit to providers that only offer these features, and so to add support for a provider the minimum the provider should offer is API access to managed DNS records. MGC will continue to provide Zone management and DNS Health checks support on a per-provider basis where appropriate.

## Provider support overview

| Provider         | DNS Records | DNS Zones | DNS Health |
|------------------|-------------|-----------|------------|
| AWS Route53      | X           | X         | X          |
| Google Cloud DNS | X           | X         | -          |
| AzureDNS         | X           | X         | -          |
| CoreDNS          | X           | -         | -          |

## Proposed Changes
In order to achieve the above, the following changes are required:
- Add DNSProvider as an API for MGC which contains all the required config for that particular provider including the credentials. This can be thought of in a similar way to a cert manager [Issuer](https://cert-manager.io/docs/concepts/issuer/).
- Update ManagedZone to add a reference to a DNSProvider. This will be a required field on the ManagedZone and a DNSProvider must exist before a ManagedZone can be created.
- Update all controllers load the DNSProvider directly from the ManagedZone during reconciliation loops and remove the single controller wide instance.
- Add new provider implementations for [Google DNS](assets/multiple-dns-provider-support/google/google.md), [Azure DNS](assets/multiple-dns-provider-support/azure/azure.md) and CoreDNS.
    - All providers constructors should accept a single struct containing all required config for that particular provider.
    - Providers must be configured from credentials passed in the config and not rely on environment variables.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

#TODO

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- How error would be reported to the users.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Adding support for multiple providers will increase the test matrix required to validate that all is working as expected, athough this could be said for the majority of new features added to a project/component and therefore does not seem enough of a deterant to not proceed with this proposal.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Investigation was carried out into the suitability of [ExternalDNS](https://github.com/kubernetes-sigs/external-dns) as the sole means of managing dns resources.
Unfortunately, while ExternalDNS does offer support for basic dns record management with a wide range of providers, there were too many features missing making it unsuitable at this time for integration.

### ExternalDNS as a separate controller

Run ExternalDNS, as intended, as a separate controller alongside mgc, and pass all responsibility for reconciling DNSRecord resources to it. All DNSRecord reconciliation is removed from MGC.

Issues:

- A single instance of ExternalDNS will only work with a single provider and a single set of credentials. As it is, in order to support more than a single provider, more than one ExternalDNS instance would need to be created, one for each provider/account pair.
- Geo and Weighted routing policies are not implemented for any provider other than AWS Route53.
- Only supports basic dns record management (A,CNAME, NS records etc ..), with no support for managed zones or health checks.

### ExternalDNS as a module dependency

Add ExternalDNS as a module dependency in order to make use of their DNS Providers, but continue to reconcile DNSRecords in MGC.

Issues:

- ExternalDNS Providers all create clients using the current environment. Would require extensive refactoring in order to modify each provider to optionally be constructed using static credentials.
- Clients were all internal making it impossible, without modification, to use the upstream code to extend the provider behaviour to support additional functionality such as managed zone creation.

# Prior art
[prior-art]: #prior-art

#TODO

# Unresolved questions
[unresolved-questions]: #unresolved-questions

#TODO

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

#TODO

Think about what the natural extension and evolution of your proposal would be and how it would affect the platform and project as a whole. Try to use this section as a tool to further consider all possible interactions with the project and its components in your proposal. Also consider how this all fits into the roadmap for the project and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but otherwise related.

Note that having something written down in the future-possibilities section is not a reason to accept the current or a future RFC; such notes should be in the section on motivation or rationale in this or subsequent RFCs. The section merely provides additional information.