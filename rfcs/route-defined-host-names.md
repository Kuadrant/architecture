# RFC Template

- HTTPRoute defined Host Names
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [GH Issue](https://github.com/Kuadrant/kuadrant-operator/issues/1386)

# Summary
[summary]: #summary

When attaching a DNSPolicy or TLSPolicy to a Gateway with a listener that has a wildcard hostname e.g. *.app.com defined, we default creating a wildcard DNS record to match and a wildcard TLS certficiate to secure communication. A wildcard DNS name and TLS certificate may not always be desirable espcially if you want to take advantage of health checks. Some users might want to more tightly control which hostnames are served and defined in DNS and have certificates. While this could be achieved by adding specific listeners to a gateway, this could become quite cumbersome for platform engineers that want to enable development teams to self serve or if a large number of unique hostnames are required. Additonally health checks do not operate against wildcard hosts as they do not provide any real value given there could be many workloads behind a single wildcard.  

# Motivation
[motivation]: #motivation

- Add better control over what happens with a wildcard hostname defined at the gateway
- Enable platform engineers to delegate hostname definition to the development teams while still retaining control of the root host and which namespaces can define HTTPRoutes via the gateway listener definition
- Enable health checks to work when using a wildcard listener.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When using DNSPolicy or TLSPolicy you will now be able to define whether you want a wildcard certificate and DNSRecord or if you want to delegate the hostname definition to the attached HTTPRoutes. To delegate you will need to set `AllowWildcardHosts` to `false`. When delegated, the DNSPolicy will setup a new DNSRecord for each unique hostname defined under a listener with a wildcard hostname and TLSPolicy will setup a new multi-hostname certificate . 

We will achieve this by adding an `AllowWildcardHosts` to the DNSPolicy and TLSPolicy. if this is set to false, no wildcard records will be created and no wildcard TLS certificates will be created regardless of the hostnames specified. An empty listener hostname is considered a wildcard.  If `AllowWildcardHosts` is false and a wildcard is defined in a HTTPRoute, the DNS and TLS policy will report that no record or tls certificate has been created for that listener / httproute via the status of the policy. So effectively the route will not be exposed via DNS.



# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A new property `AllowWildcardHosts (bool)`  will be added to both TLS and DNS policy.  It will default to true, as this is the current behaviour. When set to false, it will take the following actions:

- Defne new DNSRecords and certificates for any wildcard listeners based on the hosts defined in attached HTTPRoutes
- Flag any wildcard HTTPRoutes as having no DNSRecord or certificate created due to the policy applied. 
- Any existing wildcard DNSRecords and TLS certificates will be removed once valid replacements are in place.
- Delegated records and certifciates will be only removed once all attached HTTPRoutes with matching hostnames are removed as it is possible to have multiple HTTPRoutes with the same hostnames.

The change here will be isolated to the Kuadrant Operator and how it defines the DNSRecord and TLS certificate based on the policy. There should be no change to the DNS operator. This change will be backwards compatible and not impact existing DNSRecords. Duplicate hostnames will not result in multiple records. Deletion of these records and certificates will only happen once all HTTPRoutes referencing the hostname attached to the gateway listener are also removed.


# Drawbacks
[drawbacks]: #drawbacks

Delegating DNS name creation based on hostname in the HTTPRoute, means developers can add and remove DNSRecords from the zone. So delegation of this responsibility needs to be done carefully. That said, being able to add HTTPRoutes is also highly privelleged and impactful. Gateway admins should consider carefully which user can do this and which namespaces are allowed to attach HTTPRoutes to a particular listener.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- It allows careful control over the DNS records and allows better usage of features like health checks

# Prior art
N/A

# Unresolved questions

N/A

# Future possibilities

N/A