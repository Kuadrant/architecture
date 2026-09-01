# Token Rate Limit API Format Compatibility

- Feature Name: `token_rate_limit_api_format_compatibility`
- Status: Draft
- Start Date: 2026-08-27
- RFC PR: <https://github.com/Kuadrant/architecture/pull/199>
- Issue tracking: <https://github.com/Kuadrant/kuadrant-operator/issues/1864>
- Related: [RFC 0013 (AI policies)](./0013-ai-policies.md), [RFC 0021 (Token rate limit reservations)](./0021-token-rate-limit-reservations.md), [kuadrant-operator#1864](https://github.com/Kuadrant/kuadrant-operator/issues/1864)

# Summary

`TokenRateLimitPolicy` extracts token usage from a model response by evaluating exactly one hardcoded JSON pointer, `/usage/total_tokens`, against the response body. That pointer only exists in OpenAI-shaped responses (and the many OpenAI-compatible servers that clone that shape). Any other provider, Anthropic, Google Gemini, and anything with a differently-shaped `usage` object, silently fails to extract a token count. This RFC replaces the single hardcoded pointer with a platform engineer configurable, **ordered list of JSON Pointer (RFC 6901) expressions** per policy: the first expression that resolves to a value against the response wins, and evaluation stops there. The mechanism also generalizes correctly to streaming (SSE) responses via a new, provider-agnostic extraction strategy in wasm-shim, replacing today's OpenAI-only, positionally-hardcoded SSE parsing. The API is deliberately shaped so a later RFC can add sibling fields (`inputTokens`, `outputTokens`, `cachedTokens`) and a custom computation over them, without a breaking change, but that computation itself is out of scope here; this RFC only carries `totalTokens` end to end.

# Motivation

Today's behavior is documented as a known, named limitation, not an oversight:

— From kuadrant-operator repo <https://github.com/Kuadrant/kuadrant-operator/blob/main/doc/reference/tokenratelimitpolicy.md>
> What's actually checked: Token extraction looks for a single JSON pointer, `/usage/total_tokens`... Not currently supported: Providers that return token usage under a different field name: Anthropic's `/v1/messages` (`usage.input_tokens`/`usage.output_tokens`, no `total_tokens`) or Google Gemini's native endpoint (`usageMetadata.totalTokenCount`) are not parsed. Multi-provider support is tracked in [#1864](https://github.com/Kuadrant/kuadrant-operator/issues/1864).
>

— From kuadrant-operator repo <https://github.com/Kuadrant/kuadrant-operator/blob/main/doc/overviews/token-rate-limiting.md>
> Only OpenAI-style completions responses are supported today... Other provider formats (e.g. Anthropic, Google Gemini) are not yet parsed; see #1864.
>

[RFC 0021](0021-token-rate-limit-reservations.md) explicitly deferred solving this while building the reservation layer on top of the existing extraction mechanism:

> Out of scope: this RFC does not define a generic mechanism for extracting model/usage fields from arbitrary LLM response formats (RFC-9535 JSON path configuration). That is addressed separately; this RFC assumes token/usage data is available via the same mechanism `TokenRateLimitPolicy` already uses today (CEL over `requestBodyJSON`/`responseBodyJSON`).

This RFC is that follow-up, though it resolves the gap with JSON Pointer, not JSONPath, for the reasons expanded on in [Rationale and alternatives](#rationale-and-alternatives).

# Guide-level explanation

This RFC is driven by two structural design goals, not just the immediate limitation of one missing provider:

1. **Provider-agnostic by construction.** Supporting a new or unrecognized LLM API response format must be achievable with a policy spec change alone, never a kuadrant-operator or wasm-shim code change. A genuinely new provider is supported the moment its pointer is added to the ordered list below, no release required, and that guarantee holds for streaming (SSE) responses as much as for plain JSON ones, once the response actually arrives as SSE; see [Streaming responses](#streaming-responses) and the SSE strategy in [Reference-level explanation](#reference-level-explanation).
2. **Shaped from the start to carry multiple named token types and a user-defined computation over them, even though this RFC only delivers one.** A single anonymous number was never going to be the end state: a policy owner should eventually be able to extract `inputTokens`, `outputTokens`, and `cachedTokens` as independent named values and combine them with their own expression into whatever number should actually be charged against a limit (e.g. summing input and output for a provider with no native total, or weighting cached tokens at a fraction of their face value). The API (`dataExtraction.response.<name>`, one named field per token type) is deliberately structured so that extension is additive later rather than a breaking restructure, but it only wires `totalTokens` end to end today; see [Future possibilities](#future-possibilities).

## For platform engineers

Nothing changes for existing `TokenRateLimitPolicy` resources. A policy that doesn't set the new field gets the operator's built-in defaults, which are exactly as capable as today's hardcoded behavior for OpenAI-shaped responses, plus, for free, extraction now also works against other providers' response shapes; see the caveats in [Built-in defaults](#built-in-defaults).

The one new thing available is an optional `dataExtraction` field on the policy spec:

```yaml
apiVersion: kuadrant.io/v1alpha1
kind: TokenRateLimitPolicy
metadata:
  name: token-limits
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: my-llm-gateway
  dataExtraction:
    response:
      # Ordered list of JSON Pointer (RFC 6901) expressions. Evaluated top
      # to bottom against the response body; the first one that resolves
      # to a value wins, and the rest are never evaluated.
      totalTokens:
        - "/usage/total_tokens"               # OpenAI, and OpenAI-compatible servers
        - "/usageMetadata/totalTokenCount"    # Google Gemini
        - "/response/usage/total_tokens"      # OpenAI Responses API, streaming
        - "/usage/totalTokens"                # AWS Bedrock
  limits:
    free:
      rates:
      - limit: 20000
        window: 1d
      counters:
      - expression: auth.identity.userid
```

`dataExtraction` is grouped by direction first (`response` today; a future `request` sits alongside it, see below), and then by named field (`totalTokens` today; `inputTokens`/`outputTokens`/`cachedTokens` later). That ordering, direction outermost, is deliberate: it means everything extracted from the response lives in one place, and a policy reader can see the whole set of response-derived fields at a glance, rather than each field repeating its own "this comes from the response" wrapper.

This is the same syntax already used elsewhere in this system: `requestBodyJSON`/`responseBodyJSON` CEL functions have always taken a JSON Pointer argument, so a platform engineer who has already written a `when` predicate or a `counters` expression referencing the request or response body is reusing a syntax they already know, not learning a second one.

If `dataExtraction.response.totalTokens` is omitted, the operator uses exactly the entries shown above as its built-in default. This is what makes existing policies unaffected. Setting the field replaces the default list outright; it doesn't merge with it. If your gateway fronts a provider not covered by the defaults (a self-hosted model server with a custom response shape, for instance), add its pointer to the list. Putting the most specific/likely-to-match provider first is a minor optimization (fewer failed lookups per request), not a correctness requirement, since only one expression ever needs to succeed.

There's no `provider: openai | anthropic | gemini` selector anywhere in this API, by design. See [Rationale and alternatives](#rationale-and-alternatives).

There's also no `dataExtraction.request` today. Token usage is exclusively a response-side concern, no provider sends it in the request, so there's nothing for it to extract yet. The field is left out entirely rather than shipped empty, but the shape is deliberately prepared for it; see [Reference-level explanation](#reference-level-explanation) and [Future possibilities](#future-possibilities).

### Why `totalTokens` only, for now

The field is named `totalTokens`, not `tokens` or `usage`, deliberately. This RFC carries exactly one number end to end, the same one `hits_addend` has always carried, and does not attempt to sum `input_tokens + output_tokens` for providers (like Anthropic) that have no native total field. A policy targeting an Anthropic-only backend can point `dataExtraction.response.totalTokens` at `/usage/output_tokens` if that approximation is good enough for its purposes, but computing a true sum is deferred; see [Future possibilities](#future-possibilities) for why the API is still shaped to support it later without a breaking change.

### Streaming responses

No configuration is needed to make this work for SSE (`text/event-stream`) responses. The same `dataExtraction.response.totalTokens` pointer list applies unchanged, whether the response is a single JSON object or a stream of `data:` events. Internally, wasm-shim uses a different, provider-agnostic algorithm to locate the right event in a stream (see [Reference-level explanation](#reference-level-explanation)); that algorithm needs no per-provider knowledge or configuration, which is why there's no separate "streaming pointers" field.

One consequence of sharing a single list across both forms: some providers put the value at different paths depending on whether the response is streamed or not (OpenAI's Responses API is one, see [Built-in defaults](#built-in-defaults)). Nothing here infers one shape from the other, so both paths need their own entry in the list.

## Migration

Existing `TokenRateLimitPolicy` resources need no changes; the operator's built-in default list reproduces today's exact OpenAI-only pointer as its first (and previously only) candidate, with other providers' pointers added after it. No policy's observed behavior against an OpenAI-shaped backend changes. A policy fronting a Gemini backend, which previously extracted nothing and rate-limited every request at a token cost of zero, starts extracting real usage the moment both the operator and wasm-shim ship this change. Worth calling out explicitly since it's a behavior change for that one case, not a pure no-op, even though no CR edit is required to get it.

# Reference-level explanation

The design below has to clear two concrete obstacles in the current implementation, which make the status quo actively unworkable, not merely incomplete:

1. **The current `/usage/total_tokens` pointer is a Go string literal, not a field.** It lives in `internal/controller/ratelimit_workflow_helpers.go`, hardcoded inside the function that builds the response-phase wasm action for every `TokenLimit`. There is no CRD field a platform engineer could set today, even as a workaround. Supporting a new provider requires a kuadrant-operator code change and release, for every provider, forever.
2. **Streaming makes it worse, not just "also unsupported."** wasm-shim's SSE parser (`SseBodyParser`) assumes the [OpenAI Chat Completions streaming][openai-chat-streaming] convention specifically: it retains only the last two events it has seen and reads usage from the penultimate one (blind position N-1), which for OpenAI Chat Completions happens to be the usage-bearing chunk sitting just before the terminal `data: [DONE]` event. The parser never inspects event contents to locate usage, nor does it actually match the `[DONE]` sentinel; the position is the entire heuristic, and it is baked into working code, not just a missing pointer. [Anthropic's streaming protocol][anthropic-messages-streaming] has no `[DONE]` sentinel and splits usage across two different named events (`message_start` for input tokens, `message_delta` for output tokens). [Gemini's SSE mode][gemini-generate-content] omits `usageMetadata` from every intermediate chunk entirely, and only includes the complete, final totals in the last chunk once generation finishes, with no terminal sentinel marking it as the last one; a strategy built around "skip the very last event, use the one before it" targets exactly the wrong event for Gemini, since the second-to-last chunk has no usage data at all. Even OpenAI itself is no longer one shape: its newer [Responses API][openai-responses-streaming] streams named events (`event: response.completed`) rather than Chat Completions' bare `data:` chunks. None of these fit the "look at position N-1" model, so a fix has to change the extraction *strategy*, not just add a pointer.

## kuadrant-operator

### CRD changes

A new optional field on `TokenRateLimitPolicySpecProper` (`api/v1alpha1/tokenratelimitpolicy_types.go`), placed there, not inside individual `TokenLimit` entries, because extraction is a property of *how the response is parsed*, shared by every limit the policy defines, matching today's implicit behavior where the one hardcoded pointer already applies uniformly across all of a policy's limits:

```go
type TokenRateLimitPolicySpecProper struct {
    kuadrantv1.MergeableWhenPredicates `json:""`
    Limits         map[string]TokenLimit `json:"limits,omitempty"`
    DataExtraction *DataExtraction       `json:"dataExtraction,omitempty"`
}

// DataExtraction configures how LLM-related data is read out of the
// request/response exchange for this policy. Response is the only
// direction implemented today. The type is shaped to add a sibling
// Request field once extracting data from the request body becomes
// useful (e.g. a provider's max_tokens field for reservation sizing,
// see RFC 0021's unresolved questions on request-body availability at
// Reserve time), without a breaking change: Request would be added here,
// not by restructuring Response.
type DataExtraction struct {
    Response *ResponseDataExtraction `json:"response,omitempty"`
}

type ResponseDataExtraction struct {
    // TotalTokens is an ordered list of JSON Pointer (RFC 6901) expressions,
    // evaluated against the response body. The first expression that
    // resolves to a value is used; the rest are not evaluated.
    // Additional sibling fields (e.g. InputTokens, OutputTokens,
    // CachedTokens) are expected future additions, not part of this type
    // yet; see Future possibilities.
    // +kubebuilder:validation:MinItems=1
    // +kubebuilder:validation:MaxItems=8
    // +kubebuilder:validation:items:Pattern=`^(/([^/~]|~[01])*)*$`
    TotalTokens []string `json:"totalTokens,omitempty"`
}
```

Because `DataExtraction` lives inside `TokenRateLimitPolicySpecProper`, it automatically participates in the policy's existing `defaults`/`overrides` hierarchy ([RFC 0009](./0009-defaults-and-overrides.md)) with no new merge machinery. A gateway-level policy can set a cluster-wide default extraction list, and a route-level policy can override it, using the same Defaults & Overrides semantics every other field on this type already has.

The `MaxItems=8` cap bounds worst-case work per response; see [Security considerations](#security-considerations).

### Validation

Each entry in `dataExtraction.response.totalTokens` is validated as syntactically well-formed JSON Pointer with an OpenAPI validation `pattern` on the list items (shown in [CRD changes](#crd-changes) above), enforced by the API server at write time:

```go
// +kubebuilder:validation:items:Pattern=`^(/([^/~]|~[01])*)*$`
```

That regex admits the empty pointer and any `/`-led sequence of reference tokens, and rejects both a missing leading `/` and a bare `~` not followed by `0` or `1` (`~0`/`~1` being the only valid escapes of `~` and `/`), per RFC 6901. It is anchored (`^…$`) because Kubernetes does not implicitly anchor `pattern`. Together with the `MinItems`/`MaxItems` markers, an invalid entry is rejected at `kubectl apply` time and never reaches the reconciler.

### Built-in defaults

Built-in defaults cover the known OpenAI, Gemini, and AWS Bedrock response shapes out of the box, so existing policies keep working with zero changes.

Used whenever a policy (after defaults/overrides merging) has no `dataExtraction.response.totalTokens` set:

| Order | JSON Pointer | Provider shape |
|---|---|---|
| 1 | `/usage/total_tokens` | [OpenAI][openai-chat], Azure OpenAI, and OpenAI-compatible servers ([vLLM][vllm-openai-compat], TGI, SGLang, llama.cpp server, Groq, Together, Fireworks, DeepSeek, Mistral, etc.); also OpenAI's [Responses API][openai-responses], non-streaming |
| 2 | `/usageMetadata/totalTokenCount` | [Google Gemini][gemini-generate-content] |
| 3 | `/response/usage/total_tokens` | OpenAI's [Responses API, streaming][openai-responses-streaming]: the `response.completed` event nests the whole response object |
| 4 | `/usage/totalTokens` | [AWS Bedrock Converse API][bedrock-token-usage], non-streaming. Streaming API ([`ConverseStream`][bedrock-converse-stream]) is not over SSE at all |

Anthropic has no default entry here: its native response has no total-tokens field (`usage.input_tokens` and `usage.output_tokens` only), so a single-pointer default can't produce a meaningful total for it without summation, which is out of scope for this RFC. See [Future possibilities](#future-possibilities).

### Wasm-shim config generation (`internal/wasm/action_spec.go`)

- **Emission.** Instead of generating `responseBodyJSON("/usage/total_tokens")` (a single hardcoded string) for `hits_addend`, the operator takes whatever `dataExtraction.response.totalTokens` resolves to after defaults/overrides merging and serializes it, unchanged, in order, as a CEL list literal: e.g. `responseBodyJSON(["/usage/total_tokens", "/usageMetadata/totalTokenCount"])`.
- **Extraction (scan/dedup).** The scan/dedup step that turns these calls into a `StoreAction` (`bodyJSONPattern` / `extractBodyRefs` / `BuildActions`) also needs to recognize this list-literal shape and its own way to name/dedupe it, since today it assumes exactly one pointer per call.

## wasm-shim

### How `responseBodyJSON` is processed today

Two phases, on different data structures.

**Parsing**: the pointer a `responseBodyJSON(...)` call references is extracted from the CEL expression's AST before any body bytes exist. Then the pointer is used to build a body parser (`JsonBodyParser`, or `SseBodyParser` for `text/event-stream` responses) that watches for that pointer as response bytes arrive, writing the resolved value into `BodyContext` once found.

**Evaluation**: the expression only runs once `BodyContext` holds a value for every pointer it references; `responseBodyJSON` itself does no parse at call time, it just looks its argument up in a map already populated from `BodyContext`. If a referenced pointer isn't there yet, evaluation returns `Pending` and the task is requeued to feed more bytes. Then try again until either the value appears or the body ends. "Which pointers to watch for" and "is it safe to evaluate yet" are consequently two separate pieces of logic, both currently built around exactly one pointer per field, which is why the changes below touch both, not just one.

### `responseBodyJSON`/`requestBodyJSON` accept a list argument

`data/cel.rs`'s `responseBodyJSON`/`requestBodyJSON` functions are extended to also accept a list of JSON Pointers where they accept a single one today. Semantically: evaluate the pointers in list order against the same parsed body; return the value of the first one that resolves; if none resolve, behave exactly as today's "no such value" case (evaluation error / absent, per existing single-pointer semantics).

Accepting the list argument at the function level is not, by itself, sufficient: the static analysis that walks a CEL expression ahead of evaluation, to decide which body fields it depends on before any body bytes have arrived, has to separately learn to recognize a list-literal argument the same way it recognizes a single string literal today. Without that, a list-argument call would be invisible to that analysis, no body dependency would be registered for it, and the field would silently never be looked for at all.

### Non-streaming: `JsonBodyParser`

Today, `JsonBodyParser` registers one `acutejson` callback per requested field, keyed by that field's single pointer string. This RFC changes what "a requested field" means: a field is now identified by its **ordered list** of candidate pointers, and `JsonBodyParser` registers a callback for every pointer in every field's list (same as today, just more of them, since the underlying `acutejson` engine already supports registering an arbitrary number of pointers and firing whichever ones the document happens to contain). What changes is `finalize_extracted()`: instead of "this field's pointer either matched or it didn't," it now picks, per field, the **highest-priority pointer (by configured order) that actually matched**. A match on the field's second candidate is only used if the first candidate never fired during the parse. This requires no change to the streaming/incremental nature of the parser: extra pointers only add more registered callbacks over the same single forward pass, not multiple passes.

This isn't just a parser fix, though. The same "must all match" assumption also lives in the logic that decides when it's safe to evaluate a field's expression. Today, that logic waits for every referenced pointer to have a value. If even one is missing, it treats the expression as not ready yet. That has to change too: only one candidate per field needs to resolve, not all of them. Otherwise a response that only matches a non-first candidate would never become ready. The same fix applies to the end-of-stream failure check. Today it fails the moment any single pointer is missing. It should only fail when every candidate for a field comes up empty, since it's now normal for all but one candidate to never appear.

### Streaming: `SseBodyParser`, universal, provider-agnostic strategy

The current implementation's "look at the second-to-last event" heuristic is removed entirely, since it's specifically an [OpenAI Chat Completions streaming][openai-chat-streaming]-convention assumption and produces wrong results for [Anthropic][anthropic-messages-streaming] (no terminal sentinel; usage split across two named events), wrong results for [Gemini][gemini-generate-content] (usage is withheld from every chunk except the true last one, and there's no terminal sentinel after it), and doesn't generalize to OpenAI's own [Responses API][openai-responses-streaming] (named events, not bare `data:` chunks). It's replaced with one strategy that needs no knowledge of *which* provider or event-naming convention is in play, and in particular no need to detect named vs. unnamed events at all:

1. For every field being extracted, precompute the leaf-key substring of each of its candidate pointers (e.g. `/usage/total_tokens` -> `"total_tokens"`, quotes included).
2. As each SSE event is dispatched (after full event-framing via the existing `EventParser`/`sse_line_parser` machinery, which is unchanged), do a cheap `str::contains` scan of the event's raw `data` string against every precomputed leaf-key substring. This is *not* a JSON parse, just a substring search.
3. If no leaf key matches, discard the event's data and move to the next event. This is the common case and is why this stays cheap: the overwhelming majority of events in a real completion stream (content deltas, ids, pings) contain none of the configured keys and are never parsed as JSON.
4. If a leaf key matches, parse *that one event's* `data` as JSON and walk the matching field's candidate pointers in priority order; the first one that resolves is the field's new value.
5. Overwrite, don't stop. Matching once doesn't end extraction for that field. A later event can still replace the value, so a provider that progressively updates the same candidate across events is handled correctly. But a later lower priority event must not overwrite an earlier, higher-priority candidate's value just because it arrived later; priority order still wins over recency.
6. At end-of-stream, if none of the candidates resolved, the task fails. Exactly as today's single-pointer case does when its one pointer never resolves.

## `limitador`

No changes. Limitador has no knowledge of LLM response formats, JSON, or "tokens" as a concept. It receives a plain numeric `hits_addend` (the standard Envoy RLS field) and increments a counter by it.

# Security considerations

- **Unbounded candidate lists as a DoS vector.** Without a cap, a policy (or a compromised/careless platform engineer) could configure an arbitrarily long `dataExtraction.response.totalTokens` list. On the SSE path, each additional candidate is one more leaf-key substring the per-event scan has to check, and, on a real match, one more full JSON parse; a crafted or unusual upstream response could exploit a long list to force many of those real parses per request. The `MaxItems=8` CRD validation bounds this at the API layer; wasm-shim additionally treats the configured list length as fixed per policy (not attacker-influenced at runtime), so the only variable an upstream response actually controls is *how many events* the substring filter has to scan, not how many expensive parses can be forced beyond the number of configured candidates that could plausibly match.

# Drawbacks

- The SSE extraction rewrite changes wasm-shim's internal streaming parser meaningfully (new strategy, not an incremental patch to the existing one), which is more surface to test and review than a pure additive change would be, even though the external behavior for already-supported OpenAI-shaped streams is unchanged.

# Rationale and alternatives

- **JSON Pointer (RFC 6901)**, not **JSONPath (RFC 9535)**, is the format used at every layer. JSON Pointer (RFC 6901) is designed to pinpoint exactly one specific location in a JSON document. JSONPath (RFC 9535) is a query language that can legitimately return zero, one, or many values from a single expression, using wildcards, filters, and slices. Since every consumer of an extracted field here needs exactly one scalar, JSON Pointer's narrower guarantee is a better fit than JSONPath's broader one, and it happens to be exactly what wasm-shim's existing `requestBodyJSON`/`responseBodyJSON` mechanism already speaks, so this RFC introduces no new expression language or translation layer between the CRD and wasm-shim.
- **Full RFC 9535 JSONPath engine inside wasm-shim**, with no restriction at all. Rejected for this RFC. All the known-provider default pointers, and the realistic universe of "where does a provider put a token count," are simple scalar lookups; none need wildcards, filters, or slices. A full engine would mean parsing complete documents into memory (losing today's incremental/streaming JSON-pointer walk) and a materially larger dependency footprint in a WASM binary built under `panic = "deny"` / `unwrap_used = "deny"` / `expect_used = "deny"`, for expressiveness nothing in this RFC's scope needs.
- **Compiling the ordered list into nested CEL `has()`/ternary fallback chains at the operator, with no wasm-shim engine change.** Considered because the operator already has the `bodyJSONPattern` scan/dedup pipeline this could reuse almost as-is for the non-streaming case. Rejected: it doesn't actually avoid a wasm-shim change even there — evaluation today requires every pointer an expression references to have resolved before the expression runs at all, so a non-matching candidate leaves the whole thing pending forever without ever reaching the ternary's fallback branch.
- **A `provider: openai | anthropic | gemini` selector field**, with per-provider extraction logic branching on it. Rejected: a provider not on the enum has no path forward short of a kuadrant-operator code change and release. The ordered-candidate-list approach has no such limitation: a novel provider is supported the moment its pointer is added to the list.
- **Runtime detection of named vs. unnamed SSE events, branching between two different extraction strategies accordingly.** This is the approach proposed in the research notes on [kuadrant-operator#1864](https://github.com/Kuadrant/kuadrant-operator/issues/1864): if events carry a non-default `event:` name, scan named events for the expected fields; otherwise fall back to the existing penultimate-event strategy. Rejected in favor of this RFC's single universal per-event key-substring scan, which needs no such branch at all: it treats every event identically whether or not it carries an `event:` name, so it handles OpenAI Responses API's named `response.completed` event and Anthropic's named events with the exact same code path already used for OpenAI Chat Completions' and Gemini's unnamed events.

# Prior art

- RFC 0013 (AI policies) introduced `TokenRateLimitPolicy` with the single hardcoded OpenAI pointer as an explicitly interim measure.
- RFC 0021 (Token rate limit reservations) named this exact gap as out of scope and pointed to a future RFC to close it; this is that RFC.
- [kuadrant-operator#1864](https://github.com/Kuadrant/kuadrant-operator/issues/1864) is the long-standing tracked issue for multi-provider token extraction that this RFC resolves.

# Unresolved questions

- **Fail open vs. fail closed when no candidate resolves.** Today, an unmatched extraction field causes the wasm-shim task to fail, which in turn is subject to whatever `failureMode` the calling action's service is configured with. This RFC preserves that behavior unchanged, consistent with "keep the current token rate limiting behavior as is": if none of the configured candidates resolve, the same failure path triggers as it would today for the single hardcoded pointer. The [kuadrant-operator#1864](https://github.com/Kuadrant/kuadrant-operator/issues/1864) research notes recommend the opposite default specifically for this case: log a warning and skip the rate-limit increment (fail open), on the reasoning that blocking traffic because a response came from an unrecognized format is worse than under-counting once. Whether to change that default behavior is a real, separate decision this RFC does not make.
- RFC 0021 separately notes that a smarter default `reservation.amount` (derived from the request's own `max_tokens`-equivalent field) is blocked on `requestBodyJSON` not being usable at `Reserve` time, not on the extraction mechanism itself. This RFC's ordered-candidate-list primitive would be a natural fit for that once the underlying body-availability-at-`Reserve`-time question resolves, tracked in [CONNLINK-1608](https://redhat.atlassian.net/browse/CONNLINK-1608), but doing so is not part of this RFC's scope. `DataExtraction` today only has a `response` side, deliberately, since `totalTokens` has no legitimate request-side source; see [Future possibilities](#future-possibilities) for where a `request` side would go.

# Future possibilities

## A `request` side of `dataExtraction`

`DataExtraction` gains a sibling field once there's a real consumer for it:

```go
type DataExtraction struct {
    Response *ResponseDataExtraction `json:"response,omitempty"`
    Request  *RequestDataExtraction  `json:"request,omitempty"`
}
```

The most concrete motivating use case is RFC 0021's reservation sizing, see [Feeding RFC 0021's reservation sizing](#feeding-rfc-0021s-reservation-sizing) below, not token usage itself, since usage is exclusively a response-side concern for every provider surveyed.

## Sibling fields for input/output/cached tokens

`dataExtraction.response` gains sibling fields alongside `totalTokens` without a breaking change, each one just another ordered pointer list, with built-in defaults that, unlike `totalTokens` today, would give [Anthropic][anthropic-messages] real coverage:

```yaml
dataExtraction:
  response:
    inputTokens:
      - "/usage/prompt_tokens"               # OpenAI, and OpenAI-compatible servers
      - "/usage/input_tokens"                # Anthropic
      - "/usageMetadata/promptTokenCount"     # Google Gemini
    outputTokens:
      - "/usage/completion_tokens"            # OpenAI, and OpenAI-compatible servers
      - "/usage/output_tokens"                # Anthropic
      - "/usageMetadata/candidatesTokenCount"  # Google Gemini
    cachedTokens:
      - "/usage/prompt_tokens_details/cached_tokens"  # OpenAI, and OpenAI-compatible servers
      - "/usage/cache_read_input_tokens"              # Anthropic
      - "/usageMetadata/cachedContentTokenCount"       # Google Gemini
```

## Custom computation over named fields

Once multiple named fields exist, a policy could compute the value used for `hits_addend` from them with a CEL expression, rather than only extracting one directly, e.g. summing input and output for that same Anthropic-shaped backend, which has no native total:

```yaml
dataExtraction:
  response:
    inputTokens: ["/usage/input_tokens"]
    outputTokens: ["/usage/output_tokens"]
tokenCompute: "tokens.inputTokens + tokens.outputTokens"   # illustrative only: the operands are the *extracted values*, not the pointer-list config fields above; exact CEL surface/namespace TBD
```

## Further default coverage

[kuadrant-operator#1864](https://github.com/Kuadrant/kuadrant-operator/issues/1864)'s survey identifies other formats a future default list could cover: Ollama's [native (non-OpenAI-compatible) API][ollama-native-api], and Bedrock's streaming [`ConverseStream`][bedrock-converse-stream]. Both are the same category of gap, not just a missing pointer: Ollama's native streaming is newline-delimited JSON, and `ConverseStream`'s is AWS's own binary [`application/vnd.amazon.eventstream`][amazon-eventstream] framing; neither is SSE, so neither would ever route into `SseBodyParser` under today's `content-type: text/event-stream` detection, and adding either would need its own transport-level parser, not just another candidate pointer.

## Feeding RFC 0021's reservation sizing

If wasm-shim's request-body-availability constraint at `Reserve` time (see RFC 0021's Unresolved questions, tracked in [CONNLINK-1608](https://redhat.atlassian.net/browse/CONNLINK-1608)) is resolved independently, a request-side extraction (a provider's `max_tokens`/`max_completion_tokens`/`generationConfig.maxOutputTokens` field) under the `dataExtraction.request` sketched above would plug directly into this RFC's same ordered-candidate-list primitive to give `reservation.amount` a real, protocol-declared default instead of today's flat constant.

# References

- [OpenAI chat][openai-chat]
- [OpenAI chat, streaming][openai-chat-streaming]
- [OpenAI Responses API][openai-responses]
- [OpenAI Responses API, streaming][openai-responses-streaming]
- [Anthropic Messages][anthropic-messages]
- [Anthropic Messages, streaming][anthropic-messages-streaming]
- [Google Gemini generateContent][gemini-generate-content]
- [Google Gemini, OpenAI-compatible][gemini-openai-compat]
- [AWS Bedrock TokenUsage][bedrock-token-usage]
- [AWS Bedrock ConverseStream][bedrock-converse-stream]
- [Amazon EventStream format][amazon-eventstream]
- [vLLM OpenAI-compatible server][vllm-openai-compat]
- [Ollama native API][ollama-native-api]
- [Ollama, OpenAI-compatible][ollama-openai-compat]

[openai-chat]: https://platform.openai.com/docs/api-reference/chat/create
[openai-chat-streaming]: https://platform.openai.com/docs/api-reference/chat/streaming
[openai-responses]: https://platform.openai.com/docs/api-reference/responses/object
[openai-responses-streaming]: https://platform.openai.com/docs/api-reference/responses/streaming
[anthropic-messages]: https://docs.anthropic.com/en/api/messages
[anthropic-messages-streaming]: https://docs.anthropic.com/en/api/messages-streaming
[gemini-generate-content]: https://ai.google.dev/api/generate-content#v1beta.GenerateContentResponse
[gemini-openai-compat]: https://ai.google.dev/gemini-api/docs/openai
[bedrock-token-usage]: https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_TokenUsage.html
[bedrock-converse-stream]: https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_ConverseStream.html
[amazon-eventstream]: https://github.com/smithy-lang/smithy/blob/main/docs/source-2.0/aws/amazon-eventstream.rst
[vllm-openai-compat]: https://docs.vllm.ai/en/latest/serving/openai_compatible_server/
[ollama-native-api]: https://github.com/ollama/ollama/blob/main/docs/api.md
[ollama-openai-compat]: https://github.com/ollama/ollama/blob/main/docs/openai.md
