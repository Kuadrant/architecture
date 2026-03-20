# Egress Gateway Support for Kuadrant

- Feature Name: `egress-gateway`
- Status: **Draft**
- Start Date: 2026-03-09
- Authors: Maksym Vavilov

# Summary

Enable Kuadrant's data plane policies (AuthPolicy, RateLimitPolicy) to operate on egress traffic flowing through an Istio egress gateway. The egress gateway infrastructure (connectivity, TLS, routing) is provided by Istio and configured by the user. Kuadrant's role is to ensure its policy set works in the egress context, provide documentation and examples, and lay the groundwork for future egress-specific capabilities.

# Motivation

Kuadrant currently attaches policies to ingress-oriented Gateway API resources, controlling inbound traffic from external clients to internal services. However, customers increasingly need application-layer egress controls:

- **Cost control**: Rate limiting outbound calls to paid external APIs (AI inference, cloud services)
- **Credential management**: Passively injecting API keys or OAuth tokens into outbound requests so application code doesn't manage secrets directly
- **Compliance and auditing**: Controlling and auditing which workloads can reach which external services
- **AI workloads**: External inference providers (OpenAI, Anthropic), remote MCP servers, and multi-cluster model serving all require egress gateway capabilities

The immediate ask comes from the RHOAI team, which needs rate limiting and authentication support for outbound traffic to external AI inference providers. Projects like MCP Gateway already handle egress connectivity (ServiceEntry, DestinationRule, HTTPRoute) but explicitly delegate auth and rate limiting to Kuadrant.

# Goals

1. Ensure the Kuadrant data plane policies (AuthPolicy, RateLimitPolicy) work with OSSM/Istio for egress
2. Support egress with the MCP Gateway (expected to work already — validate and document)
3. Provide examples and documentation for using AuthPolicy for token exchange and auth token management in the egress context (particularly for AI inference use cases)
4. Document the egress gateway setup including secure connectivity via ServiceEntry and DestinationRule (configured by the end user, not by Kuadrant)

# Non-Goals

1. **Egress traffic payload processing / body-based routing** — Request body inspection and routing (e.g., routing by model name in AI inference requests) is handled by external components (GIE BBR, ext-proc), not by Kuadrant
2. **API normalization** — Translating between API formats (e.g., OpenAI to Anthropic) is out of scope
3. **SDN-level automation** — Network-level egress controls (EgressIP, EgressFirewall, NetworkPolicy) are managed by the SDN layer (e.g., OVN-Kubernetes) and not automated by Kuadrant
4. **Envoy Gateway provider support** — Initial scope targets Istio/OSSM only

# Constraints

1. **No mesh requirement** — The solution must not require workloads to be in a service mesh. No sidecar injection. Workloads reach the egress gateway via its IP address or DNS
2. **Egress connectivity is user-configured** — Kuadrant does not create or manage ServiceEntry, DestinationRule, or egress gateway deployments. These are configured by the user via Istio APIs. Kuadrant operates on traffic already flowing through the gateway. **Risk:** Some Istio-based Gateway API implementations (e.g., Red Hat OpenShift Cluster Ingress Controller) may reserve Istio APIs (ServiceEntry, DestinationRule) for infrastructure use only, making them unavailable to end users. On such platforms, the egress gateway setup described here may not be supported. This constraint limits initial egress support to environments where Istio APIs are fully accessible (e.g., OSSM deployed via the Sail Operator)
3. **Traffic routing to the egress gateway is assumed** — For Stage 1, we assume traffic is already flowing through the egress gateway (via explicit gateway IP, DNS configuration, or network policy). How workloads discover and reach the egress gateway is the user's responsibility. Stage 2 explores using DNSPolicy with CoreDNS to automate this (see [Future Work](#dns-for-egress))

# Background

## How Kuadrant Works Today (Ingress)

Kuadrant operates as an orchestration layer that translates high-level policy intent into provider-specific resources. For a typical ingress setup:

```
User creates AuthPolicy/RateLimitPolicy attached to Gateway or HTTPRoute
    -> Kuadrant builds a topology DAG: GatewayClass -> Gateway -> HTTPRoute -> Backend
    -> Kuadrant computes effective policies for each path through the topology
    -> Kuadrant creates:
        - AuthConfig CRs (consumed by Authorino for ext_authz decisions)
        - Limitador Limits (consumed by Limitador for rate limit decisions)
        - Istio WasmPlugin (loads the wasm-shim into Envoy's filter chain)
        - Istio EnvoyFilter (adds ext_authz and ratelimit clusters pointing to Authorino and Limitador)
    -> Istio reads these CRs and configures Envoy via xDS
```

### How Kuadrant Targets Gateways

This is relevant for understanding whether egress works without changes:

1. **Topology builder** watches all Gateway resources in the cluster regardless of GatewayClass or labels. Any Gateway is included in the topology DAG.

2. **IstioExtensionReconciler** iterates over all gateways in the topology, computes the effective policies for each, and creates a **WasmPlugin** per gateway. The WasmPlugin uses `targetRefs` to reference the Gateway by name:

```yaml
# WasmPlugin created by Kuadrant (simplified)
apiVersion: extensions.istio.io/v1alpha1
kind: WasmPlugin
metadata:
  name: kuadrant-<gateway-name>
  namespace: <gateway-namespace>
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: <gateway-name>
  url: <wasm-shim-image>
  phase: STATS
  pluginConfig: <action-sets-derived-from-effective-policies>
```

3. **IstioAuthClusterReconciler** creates an **EnvoyFilter** per gateway that adds the ext_authz cluster (pointing to Authorino) to Envoy's configuration. Similarly uses `targetRefs` by gateway name.

4. **IstioRateLimitClusterReconciler** creates an **EnvoyFilter** per gateway that adds the ratelimit cluster (pointing to Limitador).

Since the reconcilers target gateways by name via `targetRefs` and the topology builder watches all gateways, there is **no inherent ingress-only filtering**. An egress gateway should be picked up by the same reconciliation loop.

### Data Plane Request Flow (Ingress)

```
Client request arrives at Gateway (Envoy)
    -> WasmPlugin (wasm-shim) evaluates action sets:
        - Matches request against CEL predicates derived from HTTPRoute rules
        - For matching rules, calls ext_authz (Authorino) for auth decisions
        - For matching rules, calls ratelimit service (Limitador) for rate limit decisions
    -> If authorized and within limits, Envoy routes to backend via HTTPRoute
```

The wasm-shim operates on standard HTTP request attributes (`request.url_path`, `request.headers`, `request.method`, etc.) — it does not inspect the traffic direction.

## Key Difference: Egress Topology

For ingress:
```
External Client -> Gateway -> HTTPRoute -> Backend Service (internal)
```

For egress:
```
Internal Workload -> Egress Gateway -> HTTPRoute -> External Service (via ServiceEntry)
```

From Envoy's perspective, the filter chain is the same — listeners, filters, clusters. The proxy doesn't inherently distinguish between ingress and egress. The same WasmPlugin and EnvoyFilter resources should apply to an egress gateway.

### Data Plane Request Flow (Egress)

```
Internal Workload
  |  Request to external service (e.g., api.openai.com)
  v
Egress Gateway (Envoy)
  |
  +- WasmPlugin (wasm-shim) evaluates action sets:
  |     - Matches request against CEL predicates from HTTPRoute rules
  |     - Calls ext_authz (Authorino): authorize workload, optionally inject credentials
  |     - Calls ratelimit (Limitador): enforce outbound rate limits
  |
  +- If authorized and within limits:
  |     Envoy routes via HTTPRoute -> ServiceEntry + DestinationRule
  |     TLS origination to external service
  |
  v
External Service (api.openai.com)
```

## Egress Gateway Setup (Istio)

An Istio egress gateway consists of the following resources, all configured by the user:

**1. Gateway** — Defines the egress gateway listeners:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: egress-gateway
spec:
  gatewayClassName: istio
  listeners:
    - name: http
      protocol: HTTP
      port: 80
```

**2. ServiceEntry** — Registers the external service in Istio's service registry, making the external FQDN routable:
```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-api
spec:
  hosts:
    - api.example.com
  ports:
    - number: 443
      name: https
      protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
```

**3. DestinationRule** — Configures TLS origination (the gateway establishes TLS to the external service):
```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: external-api
spec:
  host: api.example.com
  trafficPolicy:
    tls:
      mode: SIMPLE          # TLS origination
      sni: api.example.com
```

For mTLS to external services (where the gateway presents a client certificate), use `mode: MUTUAL` with `clientCertificate`, `privateKey`, and `caCertificates` fields.

**4. HTTPRoute** — Routes traffic from the egress gateway to the external service:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: external-api-route
spec:
  parentRefs:
    - name: egress-gateway
  hostnames:
    - api.example.com
  rules:
    - backendRefs:
        - name: api.example.com
          kind: Hostname
          group: networking.istio.io
```

This is the same pattern used by MCP Gateway for [external MCP server registration](https://github.com/Kuadrant/mcp-gateway/blob/main/docs/guides/external-mcp-server.md).

**Kuadrant policies attach to the Gateway or HTTPRoute** in the same way as for ingress — the only difference is what sits on either side of the gateway.

## Why ServiceEntry Over Upstream Alternatives

ServiceEntry is chosen over the upstream wg-ai-gateway XBackendDestination CRD because XBackendDestination is still in early alpha — the API is not finalized and there is no production track record. ServiceEntry is battle-tested and widely deployed.

## OSSM

OpenShift Service Mesh (OSSM) is Red Hat's supported distribution of Istio on OpenShift, deployed via the Sail Operator. "Support OSSM/Istio" means ensuring Kuadrant's policies work on this specific distribution. In practice OSSM is Istio with Red Hat packaging, so the implementation work is the same — but validation must happen on OSSM specifically.

# Design

## Stage 1: Validation, Documentation, and Policy Support

The primary deliverable is ensuring the existing Kuadrant data plane policies work for egress on OSSM/Istio and providing comprehensive documentation.

### Hypothesis

Based on code analysis (see [How Kuadrant Targets Gateways](#how-kuadrant-targets-gateways)), existing policies should work on egress without code changes:

- The topology builder watches all Gateways regardless of class or labels
- WasmPlugin and EnvoyFilter use `targetRefs` by gateway name, not direction-specific selectors
- The wasm-shim operates on standard HTTP request attributes, which are the same for egress traffic

This hypothesis needs runtime validation. See Tasks 1-4 in the [Execution](#execution) section for the specific validation plan.

### AuthPolicy on Egress

AuthPolicy in the egress context shifts from "authenticate the external client" to "authorize the internal workload to access this external service."

Use cases:
- Only workloads in specific namespaces can reach `api.openai.com`
- Only service accounts with specific labels can access production external APIs
- Deny access to unapproved external services

**Credential injection** — Authorino's `response` section supports header injection, which may be usable for injecting external API credentials (API keys, bearer tokens) into outbound requests. This is the primary mechanism for the AI inference token exchange use case. Whether this is sufficient or a dedicated AuthTokenManagementPolicy is needed is an investigation item (Task 0b) validated in Task 2.

### RateLimitPolicy on Egress

Rate limiting outbound requests uses the same mechanism as ingress: descriptors extracted by the Wasm filter, limits enforced by Limitador.

Use cases:
- Limit outbound calls to OpenAI to 100 requests/minute per namespace
- Per-workload rate limits on external API consumption
- Global rate limits on expensive external services to control costs

### MCP Gateway Integration

The full stack — MCP Gateway for routing, Kuadrant for policy enforcement — needs end-to-end validation on egress (Task 4).

## Stage 2: Future Work

These items are out of scope for the initial delivery but inform the long-term direction. They are prioritized by a combination of value and effort.

### P1: Validate Remaining Data Plane Policies

If Stage 1 confirms that AuthPolicy and RateLimitPolicy work on egress, these policies should work too — they use the same wasm-shim and extension mechanism. Validation effort is low.

- **TokenRateLimitPolicy** — Token-based rate limiting for outbound AI inference calls. Direct value for the RHOAI use case (cost control by token count, not just request count)
- **PlanPolicy** (extension) — Subscription-tier rate limiting on egress. Key for multi-tenant AI platforms (e.g., "free tier gets 10 req/min to OpenAI, paid tier gets 1000 req/min")
- **TelemetryPolicy** (extension) — Custom metric labels for egress traffic observability. Enables labelling outbound requests by destination, workload, or cost center

### P2: Policies Requiring New Capabilities

These require design work or new reconciliation modes beyond what Stage 1 validates.

- **OIDCPolicy** (extension) — OIDC-specific authentication on egress. Useful for environments where internal workloads authenticate via OIDC before accessing external services. AuthPolicy already covers the general case, so this is incremental
- **AuthTokenManagementPolicy** — A dedicated policy for passive credential injection that goes beyond what AuthPolicy's `response` section can provide: declarative credential-to-service binding, N-to-1 credential mapping, credential rotation, body-phase injection. May be a new CRD or an extension to AuthPolicy, depending on learnings from Stage 1. High value, high effort
- **TLSPolicy (egress mode)** — Provision client certificates (via cert-manager) that the egress gateway presents to external services requiring mTLS. Today this is manually configured via DestinationRule (`mode: MUTUAL` with cert paths). TLSPolicy could automate cert provisioning and rotation. Different from ingress TLSPolicy (which provisions server certs for listeners). Medium effort — cert-manager integration exists but the policy would need to target DestinationRule or ServiceEntry rather than Gateway listeners

### P3: Infrastructure Policies

These require significant design work and warrant their own RFCs.

- **DNSPolicy (egress mode)** — Leverage the dns-operator's existing CoreDNS provider to route external hostnames through the egress gateway. The dns-operator already supports CoreDNS — a Kuadrant CoreDNS plugin watches `DNSRecord` CRs via label selectors and serves them as DNS responses, entirely in-cluster. The approach:
  1. Kuadrant deploys a CoreDNS instance with the Kuadrant plugin
  2. DNSPolicy creates DNSRecords mapping external hostnames to the egress gateway's IP, using the CoreDNS provider
  3. The cluster's default CoreDNS is configured to forward specific subdomains to the Kuadrant CoreDNS instance
  4. Workloads resolve `api.openai.com` → egress gateway IP → traffic flows through the gateway

  **What already works:** The dns-operator CoreDNS provider and plugin are functional. **What needs work:** DNSPolicy needs a new egress reconciliation mode ("redirect external hostnames to gateway internally" vs "advertise gateway externally"). This warrants its own RFC.

### Future Considerations

Lower-priority items that may become relevant as the egress story matures.

- **Envoy Gateway support** — Extend egress policy support to Envoy Gateway once Istio implementation is stable
- **Gateway API egress abstractions** — Gateway API-level wrappers for what ServiceEntry and DestinationRule provide today, removing the Istio-specific dependency
- **Service discovery** — Discovery based on capabilities (e.g., "AI inference," "database," "MCP tools") allowing workloads to find external services without prior knowledge
- **SDN integrations** — EgressIP for source NAT, routing through designated firewalls/proxies, compliance and sovereignty governance, VPN tunnel handling

# Component Changes (Stage 1)

| Component | Change Type |
|-----------|-------------|
| kuadrant-operator | Validation — verify policies work on egress gateway. Fixes if gaps found |
| wasm-shim | Validation — verify correct operation on egress gateway |
| docs | End-to-end egress gateway documentation and examples |

**No changes expected:**

| Component | Why |
|-----------|-----|
| Authorino / Authorino Operator | AuthConfig CRDs are direction-agnostic |
| Limitador / Limitador Operator | Rate limiting is protocol-agnostic |

# Alternatives Considered

## Credential Injection: AuthPolicy vs Dedicated Policy

The main design question for egress is how to handle credential injection — injecting external API keys/tokens into outbound requests.

**Option A: Use AuthPolicy's `response` section (chosen for Stage 1)**

Authorino's `response` section supports header injection, which can inject credentials into outbound requests.

- Pros: No new CRD, reuses existing infrastructure, available today
- Cons: AuthPolicy semantics are about authentication/authorization, not credential injection. May confuse users. Credential injection may need to happen at a different phase in the filter chain than ext_authz

**Option B: Dedicated AuthTokenManagementPolicy (deferred to Stage 2)**

A new CRD with declarative credential-to-service bindings.

- Pros: Clear semantics, purpose-built for the use case, supports advanced patterns (N-to-1 mapping, body-phase injection, credential rotation)
- Cons: New CRD to design, implement, and maintain. Requires understanding the use cases first

**Option C: Extension-based approach**

Implement credential injection as a Kuadrant OOP extension (gRPC over Unix socket).

- Pros: Keeps the core operator lean, extensible
- Cons: Extensions are still maturing, adds operational complexity

**Option D: BBR plugin for credential injection**

Use the GIE BBR ext-proc framework for credential injection, as proposed in the MaaS egress document.

- Pros: Handles body-phase timing requirements for AI inference (credential injection after body-based routing)
- Cons: Adds dependency on GIE, only applicable when BBR is deployed, not general-purpose

**Decision:** Start with Option A in Stage 1 to gain experience. Task 0b investigates its viability. Learnings will inform whether Option B is needed in Stage 2.

# Security Considerations

1. **Credential exposure**: When using AuthPolicy for credential injection, external API credentials in Kubernetes Secrets must be carefully managed via RBAC
2. **Egress authorization**: Without AuthPolicy, any workload that can reach the egress gateway could access external services. Default-deny policies should be recommended
3. **Trust boundaries**: The egress gateway becomes a trust boundary — it may hold credentials that workloads don't have

# Execution

Implementation is tracked in the parent GitHub issue: https://github.com/Kuadrant/architecture/issues/145 

---

## References

- [OpenShift Application Egress Gateway Proposal](https://docs.google.com/document/d/1Z6IYPft3804LXkuk1dp1jLS0p4stZaYTvp2vXECRnB0/) — Shane Utt, Craig Brookes, Daniel Grimm
- [Egress Inference Support for MaaS](../Proposal_%20Egress%20Inference%20Support%20for%20MaaS.md) — Daniele Zonca
- [wg-ai-gateway Egress Proposal](https://github.com/kubernetes-sigs/wg-ai-gateway/blob/main/proposals/10-egress-gateways.md)
- [MCP Gateway External Server Guide](https://github.com/Kuadrant/mcp-gateway/blob/main/docs/guides/external-mcp-server.md)
- [Kuadrant AuthPolicy Documentation](https://docs.kuadrant.io/latest/kuadrant-operator/doc/overviews/auth/)
- [Kuadrant RateLimitPolicy Documentation](https://docs.kuadrant.io/latest/kuadrant-operator/doc/overviews/rate-limiting/)
- [Istio Egress Gateway Documentation](https://istio.io/latest/docs/tasks/traffic-management/egress/egress-gateway/)
