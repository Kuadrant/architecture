# CoreDNS Integration

With DNSPolicy, you can have your DNS managed automatically based on the gateways you define and the provider you configure. This DNS integration has traditionally been focused on cloud DNS providers (AWS Route53, Google Cloud DNS, Azure DNS).

If you do not use cloud DNS providers or want to limit your exposure to cloud providers, you can opt for something fully self-managed or a hybrid model where the zone is delegated to DNS servers running within your own infrastructure.

## Overview

To provide integration with CoreDNS, Kuadrant extends the DNS Operator and provides a Kuadrant [CoreDNS plugin](https://coredns.io/manual/plugins/) that sources records from the `kuadrant.io/v1alpha1/DNSRecord` resource and applies GEO and weighted response capabilities equivalent to what is provided by the various cloud DNS providers we support. There are no changes to the DNSPolicy API and no needed changes to the Kuadrant policy controllers. This integration is isolated to the DNS Operator and the CoreDNS plugin.

## Architecture


The CoreDNS integration supports both single-cluster and multi-cluster deployments:

### Single Cluster Model

A single cluster runs both DNS Operator and CoreDNS with the Kuadrant plugin. DNSRecords are created with a provider reference pointing to a CoreDNS provider secret. The CoreDNS plugin watches for DNSRecords labeled with the appropriate zone name and serves them directly.

This model is suitable for organizations that want to self-host their DNS infrastructure without the complexity of multi-cluster coordination.

### Multi-Cluster Delegation Model

Multiple clusters can participate in serving a single DNS zone through a delegation model that enables geographic distribution and high availability:

- **Primary clusters**: Run both DNS Operator and CoreDNS. They reconcile delegating DNSRecords from all clusters into authoritative DNSRecords served by their CoreDNS instances.
- **Secondary clusters**: Run DNS Operator only. They create delegating DNSRecords but do not interact with DNS providers directly.

This model enables workloads across multiple clusters to contribute DNS endpoints to a unified zone, with primary clusters maintaining the authoritative view. Each primary cluster can independently serve the complete record set, providing both high availability and geographic distribution of DNS services.

## Key Design Decisions

### Zone Labeling Mechanism

Rather than requiring CoreDNS to watch all DNSRecords in the cluster, the integration uses a label-based filtering mechanism. The DNS Operator applies a zone-specific label (`kuadrant.io/coredns-zone-name`) to DNSRecords when they are reconciled. The CoreDNS plugin watches only for DNSRecords with labels matching its configured zones, reducing resource overhead and providing clear zone boundaries.

### GEO and Weighted Routing

The CoreDNS Kuadrant plugin implements GEO and weighted routing using the same algorithmic approach as cloud providers:

- **GEO routing**: Uses geographical-database integration (with the CoreDNS `geoip` plugin) to return region-specific endpoints
- **Weighted routing**: Applies probabilistic selection based on endpoint weights
- **Combined routing**: First applies GEO filtering, then weighted selection within the matched region

This approach provides parity with cloud DNS provider capabilities while maintaining full control over the DNS infrastructure.

## Deployment Scenarios

### Scenario 1: Single Cluster with CoreDNS

A single Kubernetes cluster runs both DNS Operator and CoreDNS. Users create DNSRecords pointing to the CoreDNS provider secret. CoreDNS serves the records directly.

```
┌─────────────────────────────┐
│   Kubernetes Cluster        │
│  ┌──────────────────────┐   │
│  │   DNS Operator       │   │
│  └──────────────────────┘   │
│  ┌──────────────────────┐   │
│  │   CoreDNS            │   │
│  │   + Kuadrant Plugin  │   │
│  └──────────────────────┘   │
│  ┌──────────────────────┐   │
│  │   DNSRecords         │   │
│  │   (labeled)          │   │
│  └──────────────────────┘   │
└─────────────────────────────┘
          │
          ▼
   External DNS Queries
```

### Scenario 2: Multi-Cluster with 2 Primary + 1 Secondary

Two primary clusters run CoreDNS and reconcile delegating DNSRecords from all three clusters. Each primary cluster independently serves the complete authoritative record set. The secondary cluster creates delegating DNSRecords but does not run CoreDNS.

```
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│ Primary Cluster 1│    │ Primary Cluster 2│    │Secondary Cluster │
│                  │    │                  │    │                  │
│ DNS Operator     │◄───┤ DNS Operator     │◄───┤ DNS Operator     │
│ (primary)        │───►│ (primary)        │───►│ (secondary)      │
│                  │    │                  │    │                  │
│ CoreDNS          │    │ CoreDNS          │    │ (no CoreDNS)     │
│                  │    │                  │    │                  │
│ DNSRecords       │    │ DNSRecords       │    │ DNSRecords       │
│ - Delegating     │    │ - Delegating     │    │ - Delegating     │
│ - Authoritative  │    │ - Authoritative  │    │                  │
└──────────────────┘    └──────────────────┘    └──────────────────┘
         │                       │
         ▼                       ▼
    External DNS Queries    External DNS Queries
```

## Integration Points

### DNS Operator

The DNS Operator is extended to support CoreDNS as a provider type (`kuadrant.io/coredns`). When reconciling DNSRecords with a CoreDNS provider:

In both single-cluster and multi-cluster scenarios, the DNS Operator creates an authoritative DNSRecord with zone labels for the CoreDNS plugin to watch and serve. The key difference lies in how this authoritative record is populated:

- **Single cluster**: The authoritative DNSRecord contains endpoints from the single cluster
- **Multi-cluster delegation**: The authoritative DNSRecord is created by reading and merging delegating DNSRecords from all connected clusters

The DNS Operator distinguishes between primary and secondary roles to determine whether it should reconcile delegating records into authoritative records.

### CoreDNS Plugin

The CoreDNS plugin integrates with Kubernetes to serve DNSRecords as DNS responses:

- Watches DNSRecords with zone-specific labels
- Extracts endpoints and routing metadata (GEO codes, weights)
- Applies routing logic to DNS queries based on client location and endpoint weights
- Leverages existing CoreDNS plugins (geoip, metadata) for supporting functionality

### Cluster Interconnection

Multi-cluster delegation requires primary clusters to read DNSRecords from other clusters. This is achieved through kubeconfig-based cluster interconnection secrets that grant read access to DNSRecord resources across clusters. This approach reuses Kubernetes RBAC rather than introducing a separate authentication mechanism.

## Implementation References

For detailed implementation guidance, configuration examples, and operational procedures, see:

- **Integration Guide**: [CoreDNS Integration Guide](https://github.com/Kuadrant/kuadrant-operator/blob/main/doc/overviews/coredns-integration.md) - Production deployment and multi-cluster setup
- **Local Development Guide**: [Local testing with Kind clusters](https://github.com/Kuadrant/dns-operator/blob/main/docs/coredns/README.md)
- **Provider Configuration**: [CoreDNS provider secret setup](https://github.com/Kuadrant/dns-operator/blob/main/docs/provider.md)
- **CoreDNS Plugin**: [Plugin source code and syntax reference](https://github.com/Kuadrant/dns-operator/blob/main/coredns/plugin/README.md)
