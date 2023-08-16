# RFC for TLSPolicy API

- Feature Name: `tls_policy_api`
- Start Date: 2023-08-16
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/issues/0000)

## Summary

Introduce a new TLSPolicy API that allows Gateway Administrators comprehensive management of automated, cert-manager based TLS for Gateways.

## Motivation

Gateway Administrators require flexibility and configurability in managing automated TLS certificates for their particular use-cases. The current implementation lacks user hooks and configurable options.

## Guide-level explanation

### TLSPolicy API: 

Key features include:
- **Selecting a TLS Provider**: Enables admins to select a specific ACME provider.
- **Setting Certificate Renewal Window**: Define renewal periods and lead times for renewals.
- **Strategies**: Choose between multiple host certificate strategies or per host certificate.
- **Security Standards**: Enforce certain standards on certificates generated.

If no TLSPolicy is defined, control remains with the user, where they are free to manage certificates in a way that suits them.

## Reference-level explanation

### Implementation:

The TLSPolicy API provides mechanisms to choose a TLS provider, define renewal windows, and select TLS strategies, based similar cert-manager APIs. It also brings in additional considerations like enforcing HTTPS-only listeners and wildcard host support.

The API will interface with Gateway operations, managing TLS configurations dynamically for each Gateway and its listeners. Existing methods in the Gateway will be utilized.

**Example Policy**:

```yaml
apiVersion: kuadrant.io/v1alpha1
kind: TLSPolicy
metadata:
  name: prod-web
  namespace: multi-cluster-gateways
spec:
  targetRef:
    name: prod-web
    group: gateway.networking.k8s.io
    kind: Gateway
  certManager:
    issuerRef:
      group: cert-manager.io
      kind: ClusterIssuer
      name: glbc-ca
```


## Drawbacks

The new API introduces complexity and could potentially lead to confusion if users are not well-informed or if adequate documentation is not provided. The API does introduce, in some ways, a thin layer of abstraction over cert-manager, but we've tried to keep our API as similar as possible.

## Rationale and alternatives

This will align our TLS support with DNSPolicy and other Kuadrant policies. 

## Prior art

While `cert-manager` has been used for issuing and managing certificates, its configurations remain opaque to the end-user. The TLSPolicy API aims to provide transparency and flexibility to address this limitation.

## Unresolved questions



## Future possibilities

The TLSPolicy API might evolve to support more advanced TLS configurations, integrations with newer ACME or other providers, or even expand into broader security configurations.

