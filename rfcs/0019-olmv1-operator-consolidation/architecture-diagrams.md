# Kuadrant Operator Consolidation: Architecture Diagrams

## Key

| Term | Meaning |
|------|---------|
| **SSA** | Server-Side Apply, the Kubernetes apply method where the server tracks field ownership per manager |
| **CRB** | ClusterRoleBinding |
| **SA** | ServiceAccount |
| **CRD** | CustomResourceDefinition |
| **escalate** | RBAC verb that allows creating ClusterRoles with permissions the creator does not hold |
| **bind** | RBAC verb that allows creating ClusterRoleBindings to ClusterRoles whose permissions the creator does not hold |
| **GITREF** | Git reference (branch, tag, or SHA) used to pull charts from upstream repos |
| **Wrapper CR** | Authorino/Limitador custom resources created by kuadrant-operator and reconciled by child operators |

## Build-Time: Chart Sync and Packaging

```mermaid
graph LR
    subgraph upstream["Upstream Repos"]
        AO["Kuadrant/authorino-operator"]
        LO["Kuadrant/limitador-operator"]
        DO["Kuadrant/dns-operator"]
        MG["Kuadrant/mcp-gateway"]
    end

    SYNC["make sync-child-operator-charts"]

    AO -->|GITREF| SYNC
    LO -->|GITREF| SYNC
    DO -->|GITREF| SYNC
    MG -->|GITREF| SYNC

    subgraph local["config/child-operators/charts/"]
        ao_t["authorino-operator/"]
        lo_t["limitador-operator/"]
        do_t["dns-operator/"]
        mg_t["mcp-gateway/"]
    end

    SYNC -->|"copies chart as-is"| local
```

Charts are copied unmodified from upstream repos. No rendering, splitting, or classification at sync time. Each chart directory contains the complete upstream chart (Chart.yaml, values.yaml, templates/, crds/).

```mermaid
graph LR
    subgraph local["config/child-operators/charts/"]
        CHARTS["Complete upstream charts"]
    end

    CHARTS -->|"COPY in Dockerfile"| IMAGE["Operator container image<br/>/charts/"]
```

The kuadrant-operator OLM bundle and Helm chart contain only the kuadrant-operator's own resources (CRDs, Deployment, RBAC). Child operator resources are not included in the bundle or Helm chart.

## Cluster State After Installation (before Kuadrant CR)

```mermaid
graph TB
    subgraph operator-ns["Operator namespace (e.g. kuadrant-system)"]
        KOP["kuadrant-operator"]
        AO["authorino-operator"]
        LO["limitador-operator"]
        DO["dns-operator"]
        MG["mcp-gateway controller"]
    end

    subgraph crds["CRDs"]
        K_CRD["Kuadrant, AuthPolicy<br/>RateLimitPolicy, DNSPolicy<br/>TLSPolicy"]
        A_CRD["Authorino, AuthConfig"]
        L_CRD["Limitador"]
        D_CRD["DNSRecord<br/>DNSHealthCheckProbe"]
        M_CRD["MCPGatewayExtension<br/>MCPServerRegistration<br/>MCPVirtualServer"]
    end

    subgraph installed-by["Installed by"]
        INSTALLER["Helm or OLM"] -.-> K_CRD
        INSTALLER -.-> KOP
        KOP -->|"renders charts<br/>at startup"| AO
        KOP -->|"renders charts<br/>at startup"| LO
        KOP -->|"renders charts<br/>at startup"| DO
        KOP -->|"renders charts<br/>at startup"| MG
        KOP -->|"applies CRDs<br/>at startup"| A_CRD
        KOP -->|"applies CRDs<br/>at startup"| L_CRD
        KOP -->|"applies CRDs<br/>at startup"| D_CRD
        KOP -->|"applies CRDs<br/>at startup"| M_CRD
    end

    KOP -.->|"waiting for<br/>Kuadrant CR"| IDLE["No data plane workloads yet<br/>Child controllers already running"]
```

All child operator controllers, CRDs, and ClusterRoles are deployed at operator startup before the controller manager begins watching resources. No Kuadrant CR is needed for the control plane to be running.

## Runtime: Reconciliation Chain

```mermaid
graph TB
    USER["User"] -->|"creates"| KCR["Kuadrant CR"]
    USER -->|"creates"| MGCR["MCPGatewayExtension CR"]

    subgraph operator-ns["Operator namespace (e.g. kuadrant-system)"]
        KOP["kuadrant-operator"]
        AO["authorino-operator"]
        LO["limitador-operator"]
        DO["dns-operator"]
        MG["mcp-gateway controller"]
    end

    KCR -->|"triggers"| KOP

    subgraph kuadrant-cr-ns["Kuadrant CR namespace"]
        ACR["Authorino CR"]
        LCR["Limitador CR"]
        AW["Authorino Deployment"]
        LW["Limitador Deployment"]
        MGW["MCP broker + router"]
    end

    KOP -->|"creates wrapper CR"| ACR
    KOP -->|"creates wrapper CR"| LCR

    AO -->|"reconciles"| ACR
    LO -->|"reconciles"| LCR
    MG -->|"reconciles"| MGCR

    ACR --> AW
    LCR --> LW
    MGCR --> MGW
```

Child operator controllers are already running in the operator namespace (deployed at startup). When a user creates a Kuadrant CR, kuadrant-operator creates wrapper CRs (Authorino CR, Limitador CR) in the Kuadrant CR's namespace. The child operators reconcile these into data plane workloads. MCPGatewayExtension is created directly by the user.

## RBAC Model

```mermaid
graph LR
    subgraph sa["Service Accounts"]
        KSA["kuadrant-operator SA"]
        ASA["authorino-operator SA"]
        LSA["limitador-operator SA"]
        DSA["dns-operator SA"]
        MSA["mcp-gateway SA"]
    end

    subgraph roles["ClusterRoles"]
        KR["kuadrant-operator-manager<br/>infrastructure perms<br/>+ escalate/bind on child roles<br/>+ CRD create"]
        AR["authorino-operator-manager<br/>authorino-manager-role<br/>authorino-manager-k8s-auth-role"]
        LR_["limitador-operator-manager-role"]
        DR["dns-operator-manager-role<br/>dns-operator-remote-cluster-role"]
        MR["mcp-gateway-controller"]
    end

    subgraph who["Created by"]
        OLM_H["Helm or OLM"]
        KUADRANT["kuadrant-operator<br/>at startup"]
    end

    OLM_H -->|"installs"| KSA
    OLM_H -->|"installs"| KR
    KSA --> KR

    KUADRANT -->|"creates ClusterRole<br/>using escalate"| AR
    KUADRANT -->|"creates ClusterRole<br/>using escalate"| LR_
    KUADRANT -->|"creates ClusterRole<br/>using escalate"| DR
    KUADRANT -->|"creates ClusterRole<br/>using escalate"| MR
    KUADRANT -->|"creates CRB<br/>using bind"| ASA
    KUADRANT -->|"creates CRB<br/>using bind"| LSA
    KUADRANT -->|"creates CRB<br/>using bind"| DSA
    KUADRANT -->|"creates CRB<br/>using bind"| MSA
    ASA --> AR
    LSA --> LR_
    DSA --> DR
    MSA --> MR
```

The installer (Helm or OLM) only installs the kuadrant-operator's own SA, ClusterRole, and CRB. All child operator ClusterRoles, SAs, and CRBs are created by the kuadrant-operator at startup using `escalate` (to create ClusterRoles with permissions it does not hold) and `bind` (to create CRBs referencing those ClusterRoles).

## Resource Ownership

```mermaid
graph TB
    USER["User"] -->|"creates"| KCR["Kuadrant CR"]
    USER -->|"creates"| POLICIES["AuthPolicy, RateLimitPolicy<br/>DNSPolicy, TLSPolicy"]
    USER -->|"creates"| MGCR["MCPGatewayExtension CR"]

    subgraph installer["Installed by Helm or OLM"]
        KOP_CRDs["kuadrant-operator CRDs"]
        KOP_DEP["kuadrant-operator Deployment"]
        KOP_RBAC["kuadrant-operator SA, ClusterRole, CRB"]
    end

    subgraph startup["Deployed by kuadrant-operator at startup"]
        CHILD_CRDs["Child operator CRDs"]
        CHILD_CR["Child operator ClusterRoles"]
        AUTH_OP["authorino-operator Deployment + SA + CRB"]
        LIM_OP["limitador-operator Deployment + SA + CRB"]
        DNS_OP["dns-operator Deployment + SA + CRB"]
        MCP_OP["mcp-gateway Deployment + SA + CRB"]
    end

    subgraph kuadrant-cr["Triggered by Kuadrant CR"]
        AUTH_CR["Authorino CR"]
        LIM_CR["Limitador CR"]
    end

    KCR --> AUTH_CR
    KCR --> LIM_CR

    AUTH_CR -->|"reconciled by<br/>authorino-operator"| AUTH_WL["Authorino Deployment"]
    LIM_CR -->|"reconciled by<br/>limitador-operator"| LIM_WL["Limitador Deployment"]
    MGCR -->|"reconciled by<br/>mcp-gateway"| MCP_WL["MCP broker + router"]
```

Three layers of resource ownership:
1. **Installer** (Helm/OLM): kuadrant-operator's own resources only
2. **kuadrant-operator at startup**: child operator CRDs, ClusterRoles, controllers (not tied to any CR)
3. **kuadrant-operator via Kuadrant CR**: wrapper CRs that trigger data plane workloads
