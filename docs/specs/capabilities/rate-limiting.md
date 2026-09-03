# Rate Limiting

## Components

| Component | Repo | Language | Role |
|-----------|------|----------|------|
| RateLimitPolicy CRD | kuadrant-operator | Go | User-facing policy (v1) attached to Gateway/HTTPRoute |
| TokenRateLimitPolicy CRD | kuadrant-operator | Go | Token-based variant (v1alpha1) for AI/LLM workloads |
| Limitador | limitador | Rust | Rate-limiting service with counter storage |
| Limitador Operator | limitador-operator | Go | Deploys/configures Limitador instances |
| wasm-shim | wasm-shim | Rust | Envoy WASM filter that extracts descriptors and calls Limitador |

## Data Flow

```
User creates RateLimitPolicy (kuadrant.io/v1)
         |
         v
kuadrant-operator reconciles
         |
         +---> LimitadorLimitsReconciler
         |        writes limit definitions to Limitador CR spec.limits
         |        limitador-operator syncs to ConfigMap -> mounted into Limitador pods
         |
         +---> IstioExtensionReconciler / EnvoyGatewayExtensionReconciler
                  builds ActionSets with CEL predicates + descriptor keys
                  writes to WasmPlugin CR (Istio) or EnvoyExtensionPolicy (EG)
                  wasm-shim parses config at filter init

Request arrives at Envoy
         |
         v
wasm-shim evaluates ActionSet predicates (hostname, path, method, headers)
         |
         v
wasm-shim evaluates Action predicates (limit-level When conditions)
         |
         v
wasm-shim builds gRPC RateLimitRequest (scope + descriptors + hits_addend)
         |
         v
Limitador receives request
  - matches descriptors to limits by namespace + condition predicates
  - resolves counter variables to partition counters (e.g. per-user)
  - checks counter against max_value/seconds window
         |
         v
Response: OK (200) or OverLimit (429)
```

### TokenRateLimitPolicy (dual-phase)

```
Request phase:
  wasm-shim -> Limitador CheckRateLimit (hits_addend=0, check only)
  If over limit -> reject immediately (429)

Response phase:
  wasm-shim -> Limitador Report (hits_addend = responseBodyJSON("/usage/total_tokens"))
  Increments counter with actual token consumption from LLM response body
```

## CRD Spec Structure

### RateLimitPolicy (`kuadrant.io/v1`)

```yaml
apiVersion: kuadrant.io/v1
kind: RateLimitPolicy
metadata:
  name: my-rlp
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute          # or Gateway
    name: my-route

  # Top-level conditions (CEL, all must be true)
  when:
    - predicate: "request.method == 'POST'"

  # Limit definitions (map keyed by unique name)
  limits:
    per-user-limit:
      rates:
        - limit: 100         # max requests
          window: 1m         # time window (Gateway API duration: h/m/s/ms)
      counters:
        - expression: "auth.identity.username"   # CEL: partition counter per user
      when:
        - predicate: "request.path.startsWith('/api')"   # limit-level condition

    global-limit:
      rates:
        - limit: 1000
          window: 1h

  # OR use defaults/overrides for policy hierarchy
  # (mutually exclusive with bare limits)
  defaults:
    strategy: atomic    # "atomic" (default) or "merge"
    limits: { ... }
  overrides:
    strategy: atomic
    limits: { ... }
```

### TokenRateLimitPolicy (`kuadrant.io/v1alpha1`)

Same structure as RateLimitPolicy but:
- Limits use token-based counting (dual-phase check/report)
- `hits_addend` derived from response body (`responseBodyJSON("/usage/total_tokens")`)

## Policy Merging

Policies attach at Gateway or HTTPRoute level. When multiple policies apply to the same request path, they are merged from most specific to least specific.

### Strategies

| Strategy | Defaults behavior | Overrides behavior |
|----------|-------------------|-------------------|
| `atomic` | Target wins if non-empty, else source | Source always wins |
| `merge` | Target limits + source limits not in target | Source limits + target limits not in source |

### Hierarchy

```
Gateway-level RateLimitPolicy (least specific)
    |
    v  merge/override
HTTPRoute-level RateLimitPolicy (most specific)
    |
    v
Effective policy (what gets reconciled)
```

Each limit rule carries a `source` field tracking which policy contributed it (format: `kind/namespace/name`).

## Cross-Repo Contracts

### kuadrant-operator -> Limitador (via limitador-operator)

kuadrant-operator writes to `Limitador.Spec.Limits`:

```yaml
# Limitador CR spec.limits entry
- namespace: "default/my-route"              # <route-namespace>/<route-name>
  name: per-user-limit                       # limit name from RateLimitPolicy
  maxValue: 100
  seconds: 60
  conditions:
    - "descriptors[0]['limit.per_user_limit__a1b2c3d4'] == '1'"
  variables:
    - "descriptors[0]['auth.identity.username']"
```

limitador-operator syncs these to a ConfigMap mounted at `/home/limitador/etc/limitador-config.yaml`.

### kuadrant-operator -> wasm-shim (via WasmPlugin CR)

kuadrant-operator writes ActionSets into WasmPlugin `spec.pluginConfig`:

```json
{
  "actionSets": [{
    "name": "route-rule-1",
    "routeRuleConditions": {
      "hostnames": ["api.example.com"],
      "predicates": ["request.url_path.startsWith('/api')"]
    },
    "actions": [{
      "service": "ratelimit-service",
      "scope": "default/my-route",
      "conditionalData": [{
        "predicates": ["request.method == 'POST'"],
        "data": [
          { "key": "limit.per_user_limit__a1b2c3d4", "value": "1" },
          { "key": "auth.identity.username", "value": "auth.identity.username" }
        ]
      }]
    }]
  }]
}
```

### wasm-shim -> Limitador (gRPC)

wasm-shim sends `RateLimitRequest`:

```protobuf
RateLimitRequest {
  domain: "default/my-route"       // matches Limitador limit namespace
  descriptors: [{
    entries: [
      { key: "limit.per_user_limit__a1b2c3d4", value: "1" },
      { key: "auth.identity.username", value: "alice" }
    ]
  }]
  hits_addend: 1
}
```

### Critical invariant

The **descriptor key** in wasm-shim config (`limit.per_user_limit__a1b2c3d4`) MUST match the **condition** in Limitador limits (`descriptors[0]['limit.per_user_limit__a1b2c3d4'] == '1'`). Both are generated by kuadrant-operator using the same `LimitNameToLimitadorIdentifier()` function.

**Identifier format**: `limit.<sanitized_name>__<4-byte-SHA256-hex>`
- Sanitized: only alphanumeric and `_`
- Hash input: `<policy-namespace>/<policy-name>/<limit-key-name>`
- TokenRateLimitPolicy uses prefix `tokenlimit.` instead of `limit.`

## Limitador Internals

### Limit matching algorithm

1. Filter limits by namespace (= `scope` from gRPC request)
2. For each limit, evaluate all `conditions` as CEL predicates against the `descriptors` context
3. For each limit, resolve all `variables` — if any variable expression returns null, the limit does not apply
4. Matching limits produce Counters keyed by: `(limit_id, resolved_variable_values)`
5. First counter exceeding `max_value` in its `seconds` window -> `OverLimit`

### Counter identity

A counter is uniquely identified by:
- The limit it belongs to (namespace + conditions + seconds + variables)
- The resolved variable values (e.g., `user_id=alice`)

### Storage backends

| Backend | Use case | Flag |
|---------|----------|------|
| `memory` | Dev/test, single replica | (default) |
| `disk` | Persistent, single replica only | `disk` |
| `redis` | Production, multi-replica | `redis <URL>` |
| `redis_cached` | Production, high throughput | `redis_cached <URL>` |

### gRPC services

| Service | Method | Purpose |
|---------|--------|---------|
| Kuadrant RLS | `CheckRateLimit` | Check + increment counter |
| Kuadrant RLS | `Report` | Increment counter only (fire-and-forget) |
| Envoy RLS v3 | `ShouldRateLimit` | Standard Envoy compatibility |

Proto: `limitador-server/proto/kuadrantrls.proto`

## WASM Service Names

| Service name | Used by | Calls |
|--------------|---------|-------|
| `ratelimit-service` | RateLimitPolicy | Limitador `CheckRateLimit` |
| `ratelimit-check-service` | TokenRateLimitPolicy (request phase) | Limitador `CheckRateLimit` |
| `ratelimit-report-service` | TokenRateLimitPolicy (response phase) | Limitador `Report` |

## Key Source Files

### kuadrant-operator
- CRD types: [`api/v1/ratelimitpolicy_types.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/api/v1/ratelimitpolicy_types.go)
- Token CRD types: [`api/v1alpha1/tokenratelimitpolicy_types.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/api/v1alpha1/tokenratelimitpolicy_types.go)
- Merge strategies: [`api/v1/merge_strategies.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/api/v1/merge_strategies.go)
- Effective policy calc: [`internal/controller/effective_ratelimit_policies_reconciler.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/internal/controller/effective_ratelimit_policies_reconciler.go)
- Limitador limits reconciler: [`internal/controller/limitador_limits_reconciler.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/internal/controller/limitador_limits_reconciler.go)
- WASM action building: [`internal/controller/ratelimit_workflow_helpers.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/internal/controller/ratelimit_workflow_helpers.go)
- WASM types: [`internal/wasm/types.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/internal/wasm/types.go)
- Rate limit index: [`internal/ratelimit/index.go`](https://github.com/Kuadrant/kuadrant-operator/blob/main/internal/ratelimit/index.go)

### limitador
- Limit struct: [`limitador/src/limit.rs`](https://github.com/Kuadrant/limitador/blob/main/limitador/src/limit.rs)
- CEL evaluation: [`limitador/src/limit/cel.rs`](https://github.com/Kuadrant/limitador/blob/main/limitador/src/limit/cel.rs)
- Counter struct: [`limitador/src/counter.rs`](https://github.com/Kuadrant/limitador/blob/main/limitador/src/counter.rs)
- Storage keys: [`limitador/src/storage/keys.rs`](https://github.com/Kuadrant/limitador/blob/main/limitador/src/storage/keys.rs)
- Core rate limiter: [`limitador/src/lib.rs`](https://github.com/Kuadrant/limitador/blob/main/limitador/src/lib.rs)
- Kuadrant gRPC service: [`limitador-server/src/envoy_rls/kuadrant_service.rs`](https://github.com/Kuadrant/limitador/blob/main/limitador-server/src/envoy_rls/kuadrant_service.rs)
- Proto definition: [`limitador-server/proto/kuadrantrls.proto`](https://github.com/Kuadrant/limitador/blob/main/limitador-server/proto/kuadrantrls.proto)
- Example limits: [`limitador-server/examples/limits.yaml`](https://github.com/Kuadrant/limitador/blob/main/limitador-server/examples/limits.yaml)

### limitador-operator
- CRD types: [`api/v1alpha1/limitador_types.go`](https://github.com/Kuadrant/limitador-operator/blob/main/api/v1alpha1/limitador_types.go)
- Deployment options: [`pkg/limitador/deployment_options.go`](https://github.com/Kuadrant/limitador-operator/blob/main/pkg/limitador/deployment_options.go)
- K8s object builders: [`pkg/limitador/k8s_objects.go`](https://github.com/Kuadrant/limitador-operator/blob/main/pkg/limitador/k8s_objects.go)
- Controller: [`controllers/limitador_controller.go`](https://github.com/Kuadrant/limitador-operator/blob/main/controllers/limitador_controller.go)

### wasm-shim
- Rate limit task: [`src/kuadrant/pipeline/tasks/ratelimit.rs`](https://github.com/Kuadrant/wasm-shim/blob/main/src/kuadrant/pipeline/tasks/ratelimit.rs)
- Rate limit service client: [`src/services/rate_limit.rs`](https://github.com/Kuadrant/wasm-shim/blob/main/src/services/rate_limit.rs)
- Config parsing: [`src/configuration.rs`](https://github.com/Kuadrant/wasm-shim/blob/main/src/configuration.rs)
