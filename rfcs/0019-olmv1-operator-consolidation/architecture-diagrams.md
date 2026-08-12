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
| **Wrapper CR** | Authorino/Limitador custom resources created by kuadrant-operator and reconciled by component controllers |

## Build-Time: Chart Sync and Packaging

```mermaid
graph LR
    subgraph upstream["Upstream Repos"]
        AO["Kuadrant/authorino-operator"]
        LO["Kuadrant/limitador-operator"]
        DO["Kuadrant/dns-operator"]
        MG["Kuadrant/mcp-gateway"]
    end

    SYNC["make sync-component-charts"]

    AO -->|"tracked-branch → SHA"| SYNC
    LO -->|"tracked-branch → SHA"| SYNC
    DO -->|"tracked-branch → SHA"| SYNC
    MG -->|"tracked-branch → SHA"| SYNC

    subgraph local["component-charts/"]
        cfg["sync.yaml<br/>(tracking config)"]
        ao_t["authorino-operator/"]
        lo_t["limitador-operator/"]
        do_t["dns-operator/"]
        mg_t["mcp-gateway/"]
    end

    SYNC -->|"copies chart as-is,<br/>pins commit SHA"| local
```

Charts are copied unmodified from upstream repos. No rendering, splitting, or classification at sync time. Each chart directory contains the complete upstream chart (Chart.yaml, values.yaml, templates/, crds/). The sync tool resolves each component's tracked branch to a commit SHA and pins it in `sync.yaml`.

```mermaid
graph LR
    subgraph local["component-charts/"]
        CHARTS["Complete upstream charts"]
    end

    CHARTS -->|"COPY in Dockerfile"| IMAGE["Operator container image<br/>/charts/"]
```

The kuadrant-operator OLM bundle and Helm chart contain only the kuadrant-operator's own resources (CRDs, Deployment, RBAC). Component resources are not included in the bundle or Helm chart.

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
        K_CRD["Kuadrant, AuthPolicy<br/>RateLimitPolicy, DNSPolicy<br/>TLSPolicy, KuadrantControlPlane"]
        A_CRD["Authorino, AuthConfig"]
        L_CRD["Limitador"]
        D_CRD["DNSRecord<br/>DNSHealthCheckProbe"]
        M_CRD["MCPGatewayExtension<br/>MCPServerRegistration<br/>MCPVirtualServer"]
    end

    KCP["KuadrantControlPlane CR<br/>(auto-created singleton)"]

    subgraph installed-by["Installed by"]
        INSTALLER["Helm or OLM"] -.-> K_CRD
        INSTALLER -.-> KOP
    end

    subgraph bootstrap["Pre-manager CRD bootstrap"]
        KOP -->|"applies CRDs<br/>before manager starts"| A_CRD
        KOP -->|"applies CRDs<br/>before manager starts"| L_CRD
        KOP -->|"applies CRDs<br/>before manager starts"| D_CRD
        KOP -->|"applies CRDs<br/>before manager starts"| M_CRD
    end

    KOP -->|"auto-creates"| KCP
    KCP -->|"triggers<br/>reconciliation"| AO
    KCP -->|"triggers<br/>reconciliation"| LO
    KCP -->|"triggers<br/>reconciliation"| DO
    KCP -->|"triggers<br/>reconciliation"| MG

    KOP -.->|"waiting for<br/>Kuadrant CR"| IDLE["No Kuadrant-managed data plane workloads yet<br/>Component controllers already running"]
```

The startup sequence has two phases:
1. **Pre-manager**: CRDs are applied and established before the controller manager starts (required for PolicyMachineryController boot detection)
2. **Post-manager**: the KuadrantControlPlane controller reconciles the auto-created CR, deploying all component controllers, RBAC, and services

No Kuadrant CR is needed for the control plane to be running. MCPGatewayExtension is an exception, it is created directly by the user and reconciled by the mcp-gateway controller independently of the Kuadrant CR.

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

Component controllers are already running in the operator namespace (deployed at startup). When a user creates a Kuadrant CR, kuadrant-operator creates wrapper CRs (Authorino CR, Limitador CR) in the Kuadrant CR's namespace. The component controllers reconcile these into data plane workloads. MCPGatewayExtension is created directly by the user.

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
        KR["kuadrant-operator-manager<br/>infrastructure perms<br/>+ escalate/bind on component roles<br/>+ CRD create/list/watch"]
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

The installer (Helm or OLM) only installs the kuadrant-operator's own SA, ClusterRole, and CRB. All component ClusterRoles, SAs, and CRBs are created by the kuadrant-operator via the KuadrantControlPlane controller using `escalate` (to create ClusterRoles with permissions it does not hold) and `bind` (to create CRBs referencing those ClusterRoles).

## Resource Ownership

```mermaid
graph TB
    USER["User"] -->|"creates"| KCR["Kuadrant CR"]
    USER -->|"creates"| POLICIES["AuthPolicy, RateLimitPolicy<br/>DNSPolicy, TLSPolicy"]
    USER -->|"creates"| MGCR["MCPGatewayExtension CR"]

    subgraph installer["Installed by Helm or OLM"]
        KOP_CRDs["kuadrant-operator CRDs<br/>(incl. KuadrantControlPlane)"]
        KOP_DEP["kuadrant-operator Deployment"]
        KOP_RBAC["kuadrant-operator SA, ClusterRole, CRB"]
    end

    KCP["KuadrantControlPlane CR<br/>(auto-created)"]

    subgraph controlplane["Deployed by KuadrantControlPlane controller"]
        COMP_CRDs["Component CRDs"]
        COMP_CR["Component ClusterRoles"]
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
1. **Installer** (Helm/OLM): kuadrant-operator's own resources only (including KuadrantControlPlane CRD)
2. **KuadrantControlPlane controller**: component CRDs, ClusterRoles, controllers, driven by the auto-created KuadrantControlPlane CR, independent of Kuadrant CR. All resources labelled `app.kubernetes.io/managed-by: kuadrant-operator`
3. **kuadrant-operator via Kuadrant CR**: wrapper CRs that trigger data plane workloads
