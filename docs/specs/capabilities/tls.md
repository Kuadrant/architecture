# TLS

## Components

| Component | Repo | Language | Role |
|-----------|------|----------|------|
| TLSPolicy CRD | kuadrant-operator | Go | User-facing policy (v1) attached to Gateway |
| cert-manager | (external) | Go | Issues and renews TLS certificates |

TLS is simpler than other capabilities — kuadrant-operator translates TLSPolicy directly into cert-manager Certificate CRs. No additional Kuadrant components involved.

## Data Flow

```
User creates TLSPolicy (kuadrant.io/v1)
  targets a Gateway (optionally a specific Listener via sectionName)
         |
         v
kuadrant-operator reconciles
         |
         +---> TLSPoliciesValidator
         |        validates target, checks for conflicts
         |        verifies cert-manager Issuer/ClusterIssuer exists
         |
         +---> EffectiveTLSPoliciesReconciler
         |        for each targeted listener with TLS termination:
         |          validates listener has hostname, TLS config, certificateRefs
         |          creates cert-manager Certificate CR
         |          Certificate DNSNames = listener hostname
         |          Certificate secretName = listener certificateRef
         |
         +---> TLSPolicyStatusUpdater
                  checks Issuer readiness + Certificate readiness
                  sets Accepted + Enforced conditions

cert-manager reconciles Certificate
         |
         v
Issues certificate from Issuer/ClusterIssuer
Stores TLS cert+key in Kubernetes Secret
Gateway controller picks up Secret for TLS termination
```

## CRD Spec Structure

### TLSPolicy (`kuadrant.io/v1`)

```yaml
apiVersion: kuadrant.io/v1
kind: TLSPolicy
metadata:
  name: my-tls
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway              # Gateway only (not HTTPRoute)
    name: prod-gateway
    sectionName: https         # optional: target specific listener

  # Issuer reference (required)
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer        # "Issuer" (namespaced) or "ClusterIssuer"
    # If kind omitted or "Issuer", must be in same namespace as TLSPolicy

  # Certificate options (all optional)
  commonName: "api.example.com"          # X.509 CN (max 64 chars)
  duration: 2160h                        # certificate lifetime (default 90 days)
  renewBefore: 720h                      # renew this long before expiry (default 2/3 of duration, min 5m)
  revisionHistoryLimit: 3                # max CertificateRequest revisions kept (≥1 or nil)

  usages:                                # X.509 key usages (default: digital signature + key encipherment)
    - digital signature
    - key encipherment

  privateKey:                            # private key configuration
    algorithm: RSA                       # RSA, ECDSA, or Ed25519
    encoding: PKCS1                      # PKCS1 or PKCS8
    size: 2048                           # key size in bits
    rotationPolicy: Always               # Never or Always
```

**Note**: TLSPolicy does NOT support defaults/overrides — it attaches directly to Gateway listeners. No policy merging hierarchy.

## Listener Requirements

TLSPolicy only creates Certificates for listeners that meet ALL conditions:

| Requirement | Field | Value |
|-------------|-------|-------|
| TLS configured | `listener.tls` | Must be non-nil |
| TLS mode | `listener.tls.mode` | Must be `Terminate` |
| Certificate refs | `listener.tls.certificateRefs` | At least one ref |
| Ref group | `certificateRef.group` | `""` or `"core"` (K8s Secret) |
| Ref kind | `certificateRef.kind` | `""` or `"Secret"` |
| Ref namespace | `certificateRef.namespace` | Same as Gateway (no cross-namespace) |
| Hostname | `listener.hostname` | Should be set (falls back to `*`) |

**Shared secrets not supported**: Multiple listeners referencing the same TLS Secret will cause an `IncorrectCertificate` error. Each listener needs its own Secret.

## Cross-Repo Contract

### kuadrant-operator -> cert-manager (via Certificate CR)

kuadrant-operator creates cert-manager Certificate CRs:

- **Name**: `<gateway-name>-<listener-name>` (e.g., `prod-gateway-https`)
- **Namespace**: Same as the certificateRef Secret namespace
- **Owner**: TLSPolicy (via OwnerReference)
- **Labels**: Kuadrant operator common labels

Generated Certificate spec:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: prod-gateway-https
  ownerReferences:
    - kind: TLSPolicy
spec:
  dnsNames:
    - api.example.com              # from listener.hostname
  secretName: api-example-tls      # from listener.tls.certificateRefs[0].name
  issuerRef:
    name: letsencrypt-prod         # from TLSPolicy
    kind: ClusterIssuer
  commonName: api.example.com      # from TLSPolicy (optional)
  duration: 2160h                  # from TLSPolicy (optional)
  renewBefore: 720h                # from TLSPolicy (optional)
  usages:                          # from TLSPolicy (default: digital signature + key encipherment)
    - digital signature
    - key encipherment
  privateKey:                      # from TLSPolicy (optional)
    algorithm: RSA
    encoding: PKCS1
    size: 2048
    rotationPolicy: Always
  revisionHistoryLimit: 3          # from TLSPolicy (optional)
```

## Validation

### Preconditions

1. **Gateway API installed** — CRD must be present
2. **cert-manager installed** — CRD must be present
3. **Target found** — Gateway (and optional sectionName) must exist in topology
4. **No conflicts** — Only one TLSPolicy per targetRef (oldest wins)
5. **Issuer exists** — Referenced Issuer/ClusterIssuer must exist

### Conflict resolution

When multiple TLSPolicies target the same Gateway (or same listener), the **oldest policy** (by creation timestamp) wins. Newer policies get `Accepted=False` with reason `Conflicting`.

## TLSPolicy Status

### Conditions

| Condition | True | False |
|-----------|------|-------|
| `Accepted` | Policy validated, target found, no conflicts | Target not found, issuer missing, dependency missing, conflict |
| `Enforced` | Issuer ready AND all Certificates ready | Issuer not ready, certificate missing or not ready |

`Enforced` is only evaluated when `Accepted` is true.

### Enforcement checks

1. **Issuer readiness**: Looks up Issuer/ClusterIssuer, checks for `Ready=True` condition
2. **Certificate readiness**: For each listener, looks up expected Certificate, checks for `Ready=True` condition

## Workflow Position

TLS runs as a parallel task alongside DNS and data-plane (auth/ratelimit) workflows:

```
Main reconciliation loop
  ├── DNS workflow
  ├── TLS workflow          ← runs in parallel
  ├── Data plane workflow   (auth, rate limiting)
  ├── Observability
  └── Developer portal
```

## Key Source Files

### kuadrant-operator
- CRD types: [`api/v1/tlspolicy_types.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/api/v1/tlspolicy_types.go)
- TLS workflow: [`internal/controller/tls_workflow.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/internal/controller/tls_workflow.go)
- Validator: [`internal/controller/tlspolicies_validator.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/internal/controller/tlspolicies_validator.go)
- Certificate reconciler: [`internal/controller/effective_tls_policies_reconciler.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/internal/controller/effective_tls_policies_reconciler.go)
- Status updater: [`internal/controller/tlspolicy_status_updater.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/internal/controller/tlspolicy_status_updater.go)
- State of the world: [`internal/controller/state_of_the_world.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/internal/controller/state_of_the_world.go)
