# RFC 0022: Emit parsed token-usage fields for external metering

- Feature Name: `usage_metadata_emission`
- Start Date: 2026-09-15
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [Kuadrant/architecture#193](https://github.com/Kuadrant/architecture/issues/193)

# Summary
[summary]: #summary

The wasm-shim's `StoreTask` pipeline already parses model-response bodies (both non-streaming JSON and SSE streams) to extract token counts for `TokenRateLimitPolicy` accounting. Today the parsed data has exactly one consumer: fields extracted via `responseBodyJSON()` CEL expressions are evaluated and folded into the Limitador `hits_addend`. Everything else the parser sees is discarded.

This RFC proposes emitting the full parsed usage fields, plus request terminal status, into Envoy dynamic metadata so that external metering and billing pipelines can consume per-request usage records without deploying a second body-parsing filter.

# Motivation
[motivation]: #motivation

`TokenRateLimitPolicy` proves the gateway can account tokens accurately, including for SSE streams (the parser extracts from the penultimate event before `[DONE]`). But metering and billing pipelines external to Kuadrant cannot consume a counter increment inside Limitador. They need per-request records with:

- All usage fields the model reported: `prompt_tokens`, `completion_tokens`, `total_tokens`, and the detail objects (`prompt_tokens_details.cached_tokens`, `completion_tokens_details.reasoning_tokens`) that pricing commonly differentiates on.
- A terminal status (`ok` | `client_disconnect` | `upstream_error`), indispensable for fair billing of interrupted streams and observable only at the data plane.
- Correlation with the request identity that AuthPolicy/Authorino already resolved into filter metadata.

The RHOAI Models-as-a-Service pattern is a concrete consumer. Today operators building commercial AI inference services must deploy a second body-parsing filter alongside the wasm-shim, buffering and parsing the same bytes twice on every AI request. The llm-d project's Inference Payload Processor (external-metering plugin via ext_proc) is another independent implementation of the same parse. Both exist because the wasm-shim parses the data and does not expose it.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A platform operator who already uses `TokenRateLimitPolicy` to enforce per-tenant token limits also wants to produce billing records. Today they must deploy a separate metering filter to capture usage from the same response bodies the wasm-shim already parses.

With this feature, the operator adds `usageReporting` to their policy configuration:

```yaml
apiVersion: kuadrant.io/v1
kind: TokenRateLimitPolicy
metadata:
  name: my-model-limits
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: inference-route
  usageReporting:
    enabled: true
    includeDetails: true
```

When enabled, the wasm-shim writes the parsed usage fields into Envoy dynamic metadata under the key `kuadrant.usage` after completing its normal extraction. The operator then configures Envoy access logging to serialize these fields:

```
%DYNAMIC_METADATA(kuadrant.usage:prompt_tokens)%
%DYNAMIC_METADATA(kuadrant.usage:completion_tokens)%
%DYNAMIC_METADATA(kuadrant.usage:total_tokens)%
%DYNAMIC_METADATA(kuadrant.usage:status)%
%DYNAMIC_METADATA(kuadrant.usage:stream)%
```

Alternatively, operators can consume the metadata via OpenTelemetry export or an ext_proc filter that reads it. The choice of sink is theirs, using existing Envoy and Gateway API telemetry configuration.

The feature is opt-in and default-off. Policies that do not set `usageReporting` behave exactly as before.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Current extraction path (wasm-shim v0.14.2)

The `StoreTask` (formerly `TokenUsageTask`, renamed on main) handles response body extraction:

1. CEL expressions containing `responseBodyJSON('/path')` calls trigger body buffering.
2. For non-streaming responses, the JSON body is parsed and fields extracted by JSON Pointer path.
3. For SSE responses, the `SseBodyParser` tracks the last two events and on `finalize()` extracts from the penultimate event (skipping the `[DONE]` sentinel).
4. Extracted values are converted to CEL values. Non-scalar values (Objects, Arrays) are stringified via `parse_json_scalar` rather than preserved as structured data.
5. The `export_to_host` flag on `StoreTask` controls whether extracted values are written to Envoy attributes via `set_attribute` (which wraps the proxy-wasm `set_property` ABI).
6. `is_end_of_stream()` gates finalization: the parser accumulates across body chunks and only finalizes when the stream ends.

## Proposed changes

### 1. Structured metadata emission

After `StoreTask` extraction, when `usageReporting.enabled` is true, write a namespaced dynamic metadata object under `kuadrant.usage` containing:

| Field | Type | Source |
|-------|------|--------|
| `prompt_tokens` | integer | `responseBodyJSON('/usage/prompt_tokens')` |
| `completion_tokens` | integer | `responseBodyJSON('/usage/completion_tokens')` |
| `total_tokens` | integer | `responseBodyJSON('/usage/total_tokens')` |
| `status` | string | Derived from end-of-stream handling (see below) |
| `stream` | boolean | Whether the response was SSE |

When `includeDetails` is true, also emit:

| Field | Type | Source |
|-------|------|--------|
| `prompt_tokens_details` | object | `responseBodyJSON('/usage/prompt_tokens_details')` |
| `completion_tokens_details` | object | `responseBodyJSON('/usage/completion_tokens_details')` |

This requires preserving JSON Object values as structured metadata rather than stringifying them through `parse_json_scalar`. Today Objects pass through `parse_json_scalar` which converts them to strings. The change is to emit them as structured metadata maps when the target is dynamic metadata (not CEL evaluation).

### 2. Terminal status derivation

The `is_end_of_stream()` check already distinguishes normal completion. The addition is to capture two additional terminal states:

- `ok`: `is_end_of_stream()` returns true with a complete response.
- `client_disconnect`: stream reset received before end-of-stream.
- `upstream_error`: upstream returns an error status or the connection is reset by the upstream.

Today resets produce no output. The change is to finalize with a status field indicating the terminal condition.

### 3. Configuration plumbing

A new field `usageReporting` on `TokenRateLimitPolicy` (or a gateway-level config object) is compiled into the WASM plugin configuration the same way existing policy fields are. The kuadrant-operator translates the CRD field into the wasm-shim configuration format.

## Optional companion (separable)

Inject `stream_options: {"include_usage": true}` on streaming requests when usage reporting is enabled. Today, extraction yields zero for streams where the client did not request usage, making the counter and any emitted record read zero. This could be a separate issue/RFC.

## Ordering note

Proxy-wasm filters that need to read the emitted metadata (via `get_property`) must be ordered after the wasm-shim in the filter chain. We verified this in practice: a proxy-wasm capture filter ordered before the wasm-shim could not read Authorino's dynamic metadata; reordering it after the wasm-shim resolved the issue. Envoy-native consumers (access logging, OTEL, ext_proc) are not affected by this ordering constraint.

# Drawbacks
[drawbacks]: #drawbacks

- Adds configuration surface area to `TokenRateLimitPolicy`.
- The structured metadata emission path (preserving Objects rather than stringifying) touches the value conversion logic that the CEL evaluation path also uses. Care is needed to avoid regressions in the scalar extraction path.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

**Why dynamic metadata?** It adds no new external dependencies. Envoy's access logging, OTEL export, and ext_proc filters can all read dynamic metadata natively. Operators choose the sink with existing Envoy and Gateway API telemetry configuration.

**Alternative: structured log line.** The shim could log a structured JSON line instead of writing metadata. This works but forces operators into log parsing rather than leveraging Envoy's telemetry integrations.

**Alternative: ext-proc side-emission.** A separate ext_proc filter could read the metadata and emit records. This is viable as a downstream consumer of the proposed metadata emission but is more complex to deploy and operate than access-log-based consumption.

**Why not change the Limitador API?** The proposal is explicitly additive. It does not change `hits_addend` behavior or the Limitador API. Rate limiting and usage reporting are orthogonal concerns that happen to share a parse pass.

# Prior art
[prior-art]: #prior-art

- RFC 0021 (Token rate limit reservations, merged PR #190) established the token-usage extraction pipeline that this proposal extends.
- The RHOAI platform-billing-operator project built a standalone WASM capture filter to produce per-request usage records, demonstrating the demand and the approach. A reusable fault-injectable fake-OpenAI test server was built as part of that work and is available to contribute with the implementation PR.
- The llm-d project's Inference Payload Processor (external-metering plugin) independently implemented response body parsing for usage extraction via ext_proc.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Should the metadata key namespace be `kuadrant.usage` or follow a different naming convention established by the project?
- Should `usageReporting` live on `TokenRateLimitPolicy` or on a separate/gateway-level configuration object?
- Should the `stream_options` injection (the optional companion) be part of this RFC or a separate proposal?

# Future possibilities
[future-possibilities]: #future-possibilities

- Once usage metadata is emitted, downstream operators can build billing, chargeback, and FinOps pipelines without deploying additional body-parsing filters.
- The metadata emission mechanism could be generalized beyond token usage to other response-body fields that operators want to surface for telemetry.
- If Kuadrant adopts a native usage-record emission format, the `usage.v1` JSON schema (designed for this purpose by the platform-billing-operator project) is available as a starting point for standardization.
