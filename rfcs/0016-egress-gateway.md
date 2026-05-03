# Egress Gateway Support for Kuadrant

- Feature Name: `egress-gateway`
- Status: **Draft**
- Start Date: 2026-03-09
- Authors: Maksym Vavilov

# Summary

Enable Kuadrant's policy set to operate on egress traffic flowing through an Istio egress gateway. Initially, validate that existing data plane policies (AuthPolicy, RateLimitPolicy) work in the egress context, investigate DNS-based routing and credential injection patterns, and provide documentation and examples. Longer term, extend DNSPolicy to automate egress routing, develop credential management capabilities (AuthTokenManagementPolicy), and align with upstream EgressGateway APIs.

> **Terminology:** In this RFC, "egress gateway" refers to the Istio pattern of using a standard Gateway API Gateway resource configured for outbound traffic to external services, with routing defined via Istio's ServiceEntry and DestinationRule CRDs. This is not a separate Gateway type or Kuadrant-managed resource — it is user-configured infrastructure that Kuadrant policies attach to.

# Motivation

Kuadrant currently attaches policies to ingress-oriented Gateway API resources, controlling inbound traffic from external clients to internal services. However, customers increasingly need application-layer egress controls:

- **Cost control**: Rate limiting outbound calls to paid external APIs (AI inference, cloud services)
- **Credential management**: Passively injecting API keys or OAuth tokens into outbound requests so application code doesn't manage secrets directly
- **Compliance and auditing**: Controlling and auditing which workloads can reach which external services
- **AI workloads**: External inference providers (OpenAI, Anthropic), remote MCP servers, and multi-cluster model serving all require egress gateway capabilities

Projects like MCP Gateway already handle egress connectivity (ServiceEntry, DestinationRule, HTTPRoute) but explicitly delegate auth and rate limiting to Kuadrant.

# Goals

1. Ensure the Kuadrant data plane policies (AuthPolicy, RateLimitPolicy) work with Istio for egress
2. Investigate DNS-based routing for directing pod traffic through the egress gateway to external services
3. Investigate AuthPolicy credential injection viability for the egress context (token exchange, auth token management for AI inference use cases)
4. Support egress with the MCP Gateway (expected to work already — validate and document)
5. Document the egress gateway setup including secure connectivity via ServiceEntry and DestinationRule (configured by the end user, not by Kuadrant)

# Non-Goals

1. **Egress traffic payload processing / body-based routing** — Request body inspection and routing (e.g., routing by model name in AI inference requests) is handled by external components, not by Kuadrant
2. **API normalization** — Translating between API formats (e.g., OpenAI to Anthropic) is out of scope
3. **SDN-level automation** — Network-level egress controls (EgressIP, EgressFirewall, NetworkPolicy) are managed by the SDN layer (e.g., OVN-Kubernetes) and not automated by Kuadrant
4. **Envoy Gateway provider support** — Initial scope targets Istio only

# Constraints

1. **No mesh requirement** — The solution must not require workloads to be in a service mesh. No sidecar injection. Workloads reach the egress gateway via its IP address or DNS
2. **Egress connectivity is user-configured** — Kuadrant does not create or manage ServiceEntry, DestinationRule, or egress gateway deployments. These are configured by the user via Istio APIs. Kuadrant operates on traffic already flowing through the gateway. **Risk:** Some Istio-based Gateway API implementations may reserve Istio APIs (ServiceEntry, DestinationRule) for infrastructure use only, making them unavailable to end users. On such platforms, the egress gateway setup described here may not be supported. This constraint limits initial egress support to environments where Istio APIs are fully accessible
3. **Traffic routing to the egress gateway is assumed** — We assume traffic is already flowing through the egress gateway (via explicit gateway IP, DNS configuration, or network policy). How workloads discover and reach the egress gateway is the user's responsibility. DNS-based routing is an investigation item in this RFC (see [DNS-Based Routing](#dns-based-routing)). Automating this via DNSPolicy is future work (see [Future Work](#future-work))

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

> **Note:** For egress use cases, a ClusterIP service type may be preferred to avoid provisioning an external load balancer. Deployment variations will be covered in the documentation deliverable.

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

For mTLS to external services (where the gateway presents a client certificate), users configure `mode: MUTUAL` with `clientCertificate`, `privateKey`, and `caCertificates` fields. This is a manual configuration today — see [TLSPolicy (egress mode)](#future-work) for how this could be automated in the future.

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

# Design

The primary deliverable is ensuring the existing Kuadrant data plane policies work for egress on Istio, investigating DNS routing and credential injection patterns, and providing comprehensive documentation.

## Hypothesis

Based on code analysis (see [How Kuadrant Targets Gateways](#how-kuadrant-targets-gateways)), existing policies should work on egress without code changes:

- The topology builder watches all Gateways regardless of class or labels
- WasmPlugin and EnvoyFilter use `targetRefs` by gateway name, not direction-specific selectors
- The wasm-shim operates on standard HTTP request attributes, which are the same for egress traffic

This hypothesis needs runtime validation.

## AuthPolicy on Egress

AuthPolicy in the egress context shifts from "authenticate the external client" to "authorize the internal workload to access this external service."

Use cases:
- Only workloads in specific namespaces can reach `api.openai.com`
- Only service accounts with specific labels can access production external APIs
- Deny access to unapproved external services

**Credential injection** — Authorino's `response` section supports header injection, which may be usable for injecting external API credentials (API keys, bearer tokens) into outbound requests. This is the primary mechanism for the AI inference token exchange use case. Whether this is sufficient or a dedicated AuthTokenManagementPolicy is needed is an investigation item.

## RateLimitPolicy on Egress

Rate limiting outbound requests uses the same mechanism as ingress: descriptors extracted by the Wasm filter, limits enforced by Limitador.

Use cases:
- Limit outbound calls to OpenAI to 100 requests/minute per namespace
- Per-workload rate limits on external API consumption
- Global rate limits on expensive external services to control costs

## DNS-Based Routing

Applications need to route traffic through the egress gateway to reach external services. There are two approaches to consider, each with different trade-offs:

**Approach 1: Internal hostname** — Application uses a local hostname (e.g., `my.api.local`) that resolves to the egress gateway. The gateway rewrites the request and routes to the real external service (`my.api.com`). TLS is simpler — the application sends HTTP to the gateway, which handles TLS origination to the external service. However, the application must be aware of the internal hostname.

**Approach 2: External hostname** — Application uses the real hostname (`my.api.com`) transparently. DNS is configured so the hostname resolves to the egress gateway IP instead of the real service IP. This is transparent to the application but requires DNS interception (e.g., cluster CoreDNS forwarding rules). TLS requires careful handling — if the application initiates HTTPS, certificate validation will fail unless the gateway terminates and re-originates TLS.

```
Approach 1 (internal hostname):
Application: curl http://my.api.local
    -> resolves to egress gateway IP
    -> Egress Gateway rewrites and routes to my.api.com (TLS origination)

Approach 2 (external hostname):
Application: curl http://api.openai.com
    -> DNS intercepted, resolves to egress gateway IP
    -> Egress Gateway routes to api.openai.com (TLS origination)
```

The investigation will cover:
- Feasibility of both approaches, including TLS considerations
- DNS configuration options for the external hostname approach (e.g., cluster CoreDNS forwarding rules, platform-specific variations)
- Validation that pods successfully resolve hostnames to the egress gateway IP
- End-to-end flow: application uses hostname → resolves to gateway IP → HTTPRoute matches → policies apply → traffic reaches external service
- Documentation of the configuration steps, trade-offs, and any limitations

Automating DNS routing via DNSPolicy is out of scope for this iteration — see [Future Work](#future-work).

## MCP Gateway Integration

The full stack — MCP Gateway for routing, Kuadrant for policy enforcement — needs end-to-end validation on egress.

# Component Changes

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

**Option A: Use AuthPolicy's `response` section (chosen for initial validation)**

Authorino's `response` section supports header injection, which can inject credentials into outbound requests.

- Pros: No new CRD, reuses existing infrastructure, available today
- Cons: AuthPolicy semantics are about authentication/authorization, not credential injection. May confuse users. Credential injection may need to happen at a different phase in the filter chain than ext_authz

**Option B: Dedicated AuthTokenManagementPolicy (deferred to Future Work)**

A new CRD with declarative credential-to-service bindings.

- Pros: Clear semantics, purpose-built for the use case, supports advanced patterns (N-to-1 mapping, body-phase injection, credential rotation)
- Cons: New CRD to design, implement, and maintain. Requires understanding the use cases first

**Option C: Extension-based approach**

Implement credential injection as a Kuadrant OOP extension (gRPC over Unix socket).

- Pros: Keeps the core operator lean, extensible
- Cons: Extensions are still maturing, adds operational complexity

**Option D: BBR plugin for credential injection**

Use the BBR ext-proc framework for credential injection.

- Pros: Handles body-phase timing requirements for AI inference (credential injection after body-based routing)
- Cons: Adds external dependency, only applicable when BBR is deployed, not general-purpose

**Decision:** Start with Option A to gain experience. Investigation will determine its viability. Learnings will inform whether Option B is needed.

# Security Considerations

1. **Credential exposure**: When using AuthPolicy for credential injection, external API credentials in Kubernetes Secrets must be carefully managed via RBAC
2. **Egress authorization**: Without AuthPolicy, any workload that can reach the egress gateway could access external services. Default-deny policies should be recommended
3. **Trust boundaries**: The egress gateway becomes a trust boundary — it may hold credentials that workloads don't have

# Future Work

These items are out of scope for the initial delivery but inform the long-term direction. They are prioritized by a combination of value and effort.

## P1: Validate Remaining Data Plane Policies

If the initial validation confirms that AuthPolicy and RateLimitPolicy work on egress, these policies should work too — they use the same wasm-shim and extension mechanism. Validation effort is low.

- **TokenRateLimitPolicy** — Token-based rate limiting for outbound AI inference calls. Direct value for cost control by token count, not just request count
- **PlanPolicy** (extension) — Subscription-tier rate limiting on egress. Key for multi-tenant AI platforms (e.g., "free tier gets 10 req/min to OpenAI, paid tier gets 1000 req/min")
- **TelemetryPolicy** (extension) — Custom metric labels for egress traffic observability. Enables labelling outbound requests by destination, workload, or cost center

## P2: Policies Requiring New Capabilities

These require design work or new reconciliation modes beyond what the initial validation covers.

- **AuthTokenManagementPolicy** — A dedicated policy for passive credential injection that goes beyond what AuthPolicy's `response` section can provide: declarative credential-to-service binding, N-to-1 credential mapping, credential rotation, body-phase injection. May be a new CRD or an extension to AuthPolicy, depending on learnings from the initial investigation. High value, high effort
- **DNSPolicy (egress mode)** — Automate DNS-based routing to the egress gateway. Today, cluster CoreDNS forwarding is configured manually. DNSPolicy could automate this by creating DNSRecords mapping external hostnames to the egress gateway's IP and configuring forwarding rules. This requires a new egress reconciliation mode ("redirect external hostnames to gateway internally" vs the current "advertise gateway externally") and warrants its own RFC
- **OIDCPolicy** (extension) — OIDC-specific authentication on egress. Useful for environments where internal workloads authenticate via OIDC before accessing external services. AuthPolicy already covers the general case, so this is incremental
- **TLSPolicy (egress mode)** — Provision client certificates (via cert-manager) that the egress gateway presents to external services requiring mTLS. Today this is manually configured via DestinationRule (`mode: MUTUAL` with cert paths). TLSPolicy could automate cert provisioning and rotation. Different from ingress TLSPolicy (which provisions server certs for listeners). Medium effort — cert-manager integration exists but the policy would need to target DestinationRule or ServiceEntry rather than Gateway listeners

## P3: Long-Term Direction

These require significant design work and warrant their own RFCs.

- **Upstream egress API alignment** — The upstream Kubernetes AI Gateway WG is exploring egress gateway support ([proposal](https://github.com/kubernetes-sigs/wg-ai-gateway/blob/main/proposals/10-egress-gateways.md)). As upstream egress APIs mature, Kuadrant should align with them, reducing reliance on Istio-specific ServiceEntry/DestinationRule
- **Envoy Gateway support** — Extend egress policy support to Envoy Gateway once Istio implementation is stable
- **Gateway API egress abstractions** — Gateway API-level wrappers for what ServiceEntry and DestinationRule provide today, removing the Istio-specific dependency
- **Service discovery** — Discovery based on capabilities (e.g., "AI inference," "database," "MCP tools") allowing workloads to find external services without prior knowledge
- **SDN integrations** — EgressIP for source NAT, routing through designated firewalls/proxies, compliance and sovereignty governance, VPN tunnel handling

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
