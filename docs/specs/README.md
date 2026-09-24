# Kuadrant Capability Specs

Cross-repository specification documents for Kuadrant. These specs capture how capabilities span multiple repositories, the contracts between components, and shared conventions.

Designed for use with AI coding assistants (Spec-Driven Development) but equally useful as human-readable architecture documentation.

## Files

| File | Description |
|------|-------------|
| [repo-map.md](repo-map.md) | Inter-repo dependencies, CRD ownership, and cross-repo contracts |
| [capabilities/gateway-policy.md](capabilities/gateway-policy.md) | Policy-machinery framework: topology DAG, policy attachment, controller system |
| [capabilities/rate-limiting.md](capabilities/rate-limiting.md) | Rate limiting end-to-end across kuadrant-operator, limitador, limitador-operator, wasm-shim |
| [capabilities/auth.md](capabilities/auth.md) | Auth/authz end-to-end across kuadrant-operator, authorino, authorino-operator, wasm-shim |
| [capabilities/dns.md](capabilities/dns.md) | DNS management across kuadrant-operator and dns-operator, including multi-cluster delegation |
| [capabilities/tls.md](capabilities/tls.md) | TLS certificate management via kuadrant-operator and cert-manager |
| [capabilities/cross-cutting.md](capabilities/cross-cutting.md) | Shared conventions: labels, conditions, errors, images, toolchain, testing |

## How to use with AI agents

Point your AI coding assistant at the relevant spec file(s) when working on cross-repo changes. The specs provide:

- **Data flow diagrams** showing how a user action propagates through components
- **CRD spec structures** with realistic YAML examples
- **Cross-repo contracts** specifying the exact data format at each integration boundary
- **Critical invariants** that must stay in sync across repos
- **Key source file links** for navigating directly to implementation
