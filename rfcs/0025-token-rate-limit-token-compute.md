# Token Rate Limit Multi-Field Extraction and Custom Token Compute

- Feature Name: `token_rate_limit_token_compute`
- Status: Draft
- Start Date: 2026-09-22
- RFC PR: <https://github.com/Kuadrant/architecture/pull/0000>
- Issue tracking: [kuadrant-operator#1654](https://github.com/Kuadrant/kuadrant-operator/issues/1654)
- Related: [RFC 0013 (AI policies)](./0013-ai-policies.md), [RFC 0021 (Token rate limit reservations)](./0021-token-rate-limit-reservations.md), [RFC 0024 (Token rate limit API format compatibility)](./0024-token-rate-limit-api-format-compatibility.md), [kuadrant-operator#2299](https://github.com/Kuadrant/kuadrant-operator/pull/2299) (RFC 0024 implementation, merged), [wasm-shim#424](https://github.com/Kuadrant/wasm-shim/pull/424) (RFC 0024 implementation, merged)

# Summary

[RFC 0024](./0024-token-rate-limit-api-format-compatibility.md) replaced `TokenRateLimitPolicy`'s single hardcoded `/usage/total_tokens` pointer with a configurable, ordered list of JSON Pointer candidates under `dataExtraction.response.totalTokens`, but it deliberately carries exactly one number end to end: whatever `totalTokens` resolves to becomes `hits_addend` (Optimistic mode) or the reservation's `actual_amount` (Reservation mode), unconditionally. It explicitly named its own limit: a provider like Anthropic, which has no native total-tokens field, gets no real coverage from a single-pointer-list mechanism, and a policy owner who wants to weight some token kinds differently (e.g. charge cached tokens at a fraction of their face value) has no way to express that. RFC 0024's Future possibilities named both gaps and sketched, without resolving, an API shape for them.

This RFC resolves both gaps, building directly on the fact that `dataExtraction.response`'s implementation ([kuadrant-operator#2299](https://github.com/Kuadrant/kuadrant-operator/pull/2299)) already shipped as an open, per-target-name map rather than a single fixed field — so a policy can already extract more than one named token quantity from a response today, with the set of names left entirely to the policy author. What's missing, and what this RFC adds, is a way to *use* those named quantities: a way to define, per policy, how they combine into the single value actually charged against its limits — covering both a real total for providers with no native one (Anthropic) and cost-weighted charging (e.g. discounting cached tokens). Existing policies that only use `totalTokens` are entirely unaffected.

# Motivation

RFC 0024 solved "extract one number from any provider's response shape." It did not solve "compute the number that should actually be charged," and said so directly:

> This RFC carries exactly one number end to end, the same one `hits_addend` has always carried, and does not attempt to sum `input_tokens + output_tokens` for providers (like Anthropic) that have no native total field. A policy targeting an Anthropic-only backend can point `dataExtraction.response.totalTokens` at `/usage/output_tokens` as an interim workaround, but this under-counts, it is not an approximation: input tokens are omitted entirely, and they routinely dominate the total in long-context usage, so a limit set this way isn't a real limit on total usage.

Concretely, three classes of gateway operator are left unserved by RFC 0024 alone:

1. **Providers with no native total.** Anthropic's `/v1/messages` response has `usage.input_tokens` and `usage.output_tokens`, never a total. Every workaround RFC 0024 leaves available (pointing `totalTokens` at just one of the two) is a genuine under-count, not an approximation, because input and output tokens routinely have very different magnitudes and neither alone tracks real usage.
2. **Operators who want the charged amount to reflect real cost, not raw token count.** Cached-prefix tokens (OpenAI's `prompt_tokens_details.cached_tokens`, Anthropic's `cache_read_input_tokens`, Gemini's `cachedContentTokenCount`) are typically billed by the upstream provider at a fraction of a normal input token's cost. A platform engineer passing that cost through to their own users' rate limits has no way to express "count cached tokens at 25% weight" today; `totalTokens` is an opaque, pre-summed number with no way to discount any component of it.
3. **Whatever comes next.** A closed, hardcoded set of extractable quantities — even a generous one — is a bet that this RFC's authors correctly anticipated every dimension a platform engineer will ever want to extract and combine. Reasoning-token counts, tool-call token counts, or a provider-specific quantity with no equivalent among today's major providers are all plausible near-future needs. RFC 0024's own central design goal, restated here, was that a new shape must be addressable with a policy spec change alone, never a kuadrant-operator code change; this RFC applies that same goal to the *set of named quantities* a policy can extract, not just to the pointer candidates for one fixed quantity.

All three require the same two underlying primitives: (a) extracting more than one named number per response, with the set of names left open, and (b) a way to combine those numbers into the one number a limit actually needs. RFC 0024 shaped its API so this could be added without a breaking change, but explicitly deferred doing so. This RFC is that follow-up.

# Guide-level explanation

## For platform engineers

Nothing changes for a `TokenRateLimitPolicy` that only uses `dataExtraction.response.totalTokens` (or omits `dataExtraction` entirely). This RFC is purely additive on top of RFC 0024.

### `dataExtraction.response` is already an open, named map

As shipped by RFC 0024's implementation, `dataExtraction.response` is not a fixed set of fields with `totalTokens` as the only one — it's a map from a **target name** to an ordered JSON Pointer candidate list, and `totalTokens` is simply the one target name the operator currently reads. Nothing stops a policy from declaring other target names today; they're just inert, because nothing consumes them yet:

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
      totalTokens:
        - "/usage/total_tokens"      # unchanged from RFC 0024
      inputTokens:
        - "/usage/input_tokens"      # Anthropic
      outputTokens:
        - "/usage/output_tokens"     # Anthropic
```

`inputTokens` and `outputTokens` above are not special, recognized names: they're names this policy's author picked, in this case ones that read naturally against the values they're pointing at. A target name is free to be `promptSize`, `reasoningTokens`, or anything else that's a valid CEL identifier and not a CEL reserved word like `in` or `true` (letters, digits, underscore, not starting with a digit — see [Reference-level explanation](#reference-level-explanation) for why this constraint exists now, when it didn't before). Unlike `totalTokens`, any other target name has **no built-in default**: the operator has no way to guess a candidate pointer list for a name it has no hardcoded concept of, so every other entry must spell out its own pointer list explicitly. To make this easy for the token dimensions most providers already expose, this RFC ships documentation (a "recipe" table, not code) with ready-to-copy candidate lists for common dimensions like input/output/cached tokens across OpenAI, Anthropic, and Gemini shapes — see [Extraction recipes](#extraction-recipes-non-normative). Copying one of those recipes under whatever name you pick is the expected way to use this, not a requirement to use those exact names.

### Computing the charged value: `tokenCompute`

A new, optional, policy-level field, `tokenCompute`, is a CEL expression that replaces `totalTokens` as the source of the value charged against every limit in the policy (`hits_addend` in Optimistic mode, `reservation.actualAmount` in Reservation mode). It references any target name present in `dataExtraction.response` as `extracted.response.<name>`. The `extracted` namespace is deliberately neutral — the values it holds are whatever JSON numbers a policy's pointers resolve to, not necessarily token counts — and it is qualified by source (`extracted.response`), so the same mechanism extends unchanged to request-side extraction (`extracted.request.<name>`) once `dataExtraction.request` materializes; see [Future possibilities](#request-side-fields-feeding-tokencompute). This RFC's scope is the response side, so every reference below is `extracted.response.<name>`:

```yaml
apiVersion: kuadrant.io/v1alpha1
kind: TokenRateLimitPolicy
metadata:
  name: anthropic-limits
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: my-llm-gateway
  dataExtraction:
    response:
      inputTokens:
        - "/usage/input_tokens"
      outputTokens:
        - "/usage/output_tokens"
  # Anthropic has no native total: charge the real sum of input + output.
  tokenCompute: "extracted.response.inputTokens + extracted.response.outputTokens"
  limits:
    free:
      rates:
      - limit: 20000
        window: 1d
      counters:
      - expression: auth.identity.userid
```

Or, weighting cached tokens at 40% of a normal token's cost. Cached tokens are absent whenever a response had no cache hit, so the expression defaults that field to `0` with a ternary; see [Missing fields](#missing-fields-best-effort-extraction-and-defaults):

```yaml
spec:
  dataExtraction:
    response:
      inputTokens: ["/usage/input_tokens"]
      outputTokens: ["/usage/output_tokens"]
      cachedTokens: ["/usage/cache_read_input_tokens"]
  tokenCompute: "extracted.response.outputTokens + extracted.response.inputTokens - 0.6 * (extracted.response.cachedTokens != null ? extracted.response.cachedTokens : 0)"
```

Every name referenced as `extracted.response.<name>` in `tokenCompute` must be either `totalTokens` (whose pointer list always resolves, explicit or defaulted) or a target name actually present under this policy's (effective, post defaults/overrides) `dataExtraction.response` — nothing else resolves, by design; see [Validation](#validation-1). `tokenCompute` must evaluate to a number; the operator clamps it to a non-negative integer (see below). An expression that can never be numeric (a boolean, a list) is rejected at validation.

The author writes plain arithmetic; the operator wraps it so the charged value (`hits_addend` in Optimistic mode, `actualAmount` in Reservation mode) is always a non-negative integer — fractional parts truncated toward zero, a negative result clamped to `0`. The sign is not statically knowable (it depends on the per-response extracted values), and a negative value reaching the counter would corrupt the limit, so guaranteeing the clamp is the operator's responsibility, not the author's.

`tokenCompute` is a single, policy-level field (like `dataExtraction`, it participates in the existing `defaults`/`overrides` hierarchy) — every limit in the policy is charged with the same computed value. A future RFC could move it, or add a per-limit override, if a real need for different limits in the same policy to be charged differently emerges; see [Future possibilities](#future-possibilities).

### Missing fields: best-effort extraction and defaults

Extraction is best-effort: if none of a target's pointer candidates resolve against a response, `extracted.response.<name>` evaluates to `null`. A `null` is a valid value, not an error — on its own it fails neither the request nor the limit. What it means is the author's decision, expressed in `tokenCompute`:

- **Default it** with a ternary: `extracted.response.cachedTokens != null ? extracted.response.cachedTokens : 0`. (`!= null` is the idiom; `has()` does not apply to an extracted value.)
- **Leave it** to propagate. A `null` in arithmetic or in the operator's integer conversion fails that one response's evaluation, so the limit is not charged and the request proceeds uncounted — fail-open, under the same control-plane-wide `FailureMode` RFC 0024 already governs. Not defaulting an optional field is thus an implicit "don't rate-limit a response I can't fully compute."

Default the genuinely optional dimensions (cached, reasoning, tool tokens); a field the target provider always sends (e.g. Anthropic's `input_tokens`/`output_tokens`) needs none. For a token count in a sum, `0` is the correct default — an absent count is a zero count.

### Extraction recipes (non-normative)

Not part of the API, just documentation shipped alongside it, for the dimensions most policy authors will actually reach for:

| Concept | Suggested target name | Candidate pointers |
| --- | --- | --- |
| Prompt/input tokens | `inputTokens` | `/usage/prompt_tokens` (OpenAI-compatible), `/usage/input_tokens` (Anthropic), `/usageMetadata/promptTokenCount` (Gemini) |
| Completion/output tokens | `outputTokens` | `/usage/completion_tokens` (OpenAI-compatible), `/usage/output_tokens` (Anthropic), `/usageMetadata/candidatesTokenCount` (Gemini) |
| Cached/reused tokens | `cachedTokens` | `/usage/prompt_tokens_details/cached_tokens` (OpenAI-compatible), `/usage/cache_read_input_tokens` (Anthropic), `/usageMetadata/cachedContentTokenCount` (Gemini) |

These are suggestions, not reserved or recognized names: nothing in the operator treats `inputTokens` as special, the same way nothing treats a policy's counter names or limit names as special. A policy fronting only Anthropic could just as well name the target `promptTokens` and point it only at `/usage/input_tokens`.

### Validation: unknown target names and non-numeric results are rejected

Two mistakes are caught before a bad `tokenCompute` can silently mis-charge a limit:

- **Referencing a name that isn't `totalTokens` and isn't declared under this policy's `dataExtraction.response`** (a typo, or a name from a target the policy forgot to declare) is rejected.
- **An expression whose result can never be numeric** (e.g. one that evaluates to a boolean or a list) is rejected. The operator supplies the integer conversion, so there is no missing-wrapper mistake to make; what's caught here is an expression that could never be a number at all. (This is a type check, not a value check — a numeric expression that happens to resolve to `null` at runtime is a per-response fail-open miss, not a validation failure; see [Missing fields](#missing-fields-best-effort-extraction-and-defaults).)

Both are reported the same way RFC 0021's reservation `ttl`/`amount` CEL validation errors are reported today: the policy's `Enforced` status condition is set to `False`, with a message naming the problem (e.g. `tokenCompute references target "promptTokens", which is not "totalTokens" and not declared under dataExtraction.response` or `tokenCompute must be numeric, got bool`); see [Validation](#validation-1).

### Migration

Existing `TokenRateLimitPolicy` resources need no changes. `tokenCompute` unset behaves exactly as RFC 0024 specifies: the value charged against every limit is `extracted.response.totalTokens`, resolved via `dataExtraction.response.totalTokens` (explicit or defaulted). Nothing about `dataExtraction.response`'s CRD schema changes for this RFC other than the one additional constraint described in [Reference-level explanation](#reference-level-explanation); a policy that already declares extra target names it doesn't use from anywhere is unaffected, since the operator ignoring an unused target name is exactly today's behavior too.

# Reference-level explanation

## kuadrant-operator

### CRD changes

RFC 0024 ([kuadrant-operator#2299](https://github.com/Kuadrant/kuadrant-operator/pull/2299)) already shipped `dataExtraction.response` as an open map — each target name mapped to an ordered JSON Pointer candidate list (`map[string]JSONPointerCandidates`, capped at 16 names × 8 candidates) — specifically so new target names need no CRD change. This RFC adds two things to that API.

**A constraint on the map's keys.** Target names must now be valid CEL identifiers — and not CEL reserved words — since they become CEL references (`extracted.response.<name>`), a constraint the map didn't need while its only consumer looked up one fixed constant. The identifier-shaped regex is necessary but not sufficient: CEL reserved words (`in`, `true`, `null`, …) match it yet cannot be used as attribute access, and RE2 (which backs CEL's `matches()`) has no negative lookahead to fold the exclusion into the regex, so a second rule lists them out:

```go
// +kubebuilder:validation:XValidation:rule="self.all(k, k.matches('^[A-Za-z_][A-Za-z0-9_]*$'))",message="extraction target names must be valid CEL identifiers"
// +kubebuilder:validation:XValidation:rule="self.all(k, !(k in ['as','break','const','continue','else','false','for','function','if','import','in','let','loop','namespace','null','package','return','true','var','void','while']))",message="extraction target names must not be CEL reserved words"
type ResponseDataExtraction map[string]JSONPointerCandidates
```

**One new policy-level field, `tokenCompute`**, a sibling of `dataExtraction` (policy-wide, shared by every limit, not per-`TokenLimit`):

```go
// TokenCompute is a CEL expression computing the value charged against every limit in this
// policy (hits_addend in Optimistic mode, reservation.actualAmount in Reservation mode), in
// place of extracted.response.totalTokens, the default. It may reference extracted.response.<name>
// for "totalTokens" or any target name present in dataExtraction.response; a target that does not
// resolve is null, which the expression may default with a ternary. The operator coerces and clamps
// the result to a non-negative integer when emitting the charged value, so it must be numeric. If
// unset, extracted.response.totalTokens is used directly, unchanged from RFC 0024.
// +optional
TokenCompute *string `json:"tokenCompute,omitempty"`
```

`tokenCompute` participates in the `defaults`/`overrides` hierarchy like `dataExtraction`, with the same mutual exclusivity between an implicit (bare) value and an explicit `defaults`/`overrides` block:

```go
// +kubebuilder:validation:XValidation:rule="!(has(self.defaults) && has(self.tokenCompute))",message="Implicit tokenCompute and explicit defaults are mutually exclusive"
// +kubebuilder:validation:XValidation:rule="!(has(self.overrides) && has(self.tokenCompute))",message="Implicit tokenCompute and overrides are mutually exclusive"
```

`dataExtraction` remains a single mergeable rule (unchanged from RFC 0024): a more specific policy's `dataExtraction` replaces the entire `response` map atomically rather than merging key-by-key, so a route-level policy must repeat any gateway-level entries it still wants alongside its own.

### Validation

A policy's effective `tokenCompute` must satisfy two checks. Both are surfaced the way RFC 0021's `reservation.amount`/`reservation.ttl` CEL errors are today — the policy's `Enforced` condition set to `False` with a message naming the problem — reusing the CEL validator infrastructure this policy kind already applies to its other CEL fields (`when`/limit predicates, `reservation.amount`, `reservation.ttl`):

1. **Known targets.** Every `extracted.response.<name>` referenced must resolve to a declared target: `totalTokens` (always available, explicit or defaulted) or a name present under this policy's effective (post `defaults`/`overrides`) `dataExtraction.response`. The valid set is whatever the policy itself declares, not a fixed enum — so adding a target name is a data change to one policy, never an operator change. A reference to any other source — `extracted.request.*`, or any other `extracted.<source>.*` — is rejected until that source exists; see [Future possibilities](#request-side-fields-feeding-tokencompute).
2. **Numeric result.** The operator clamps the result to a non-negative integer (see [Emission](#emission)), so the expression must be convertible to a number. Extracted values are dynamically typed — a JSON Pointer can resolve to any basic JSON type (string, integer, float, boolean, null) — so this check rejects only an expression that can *never* be numeric: one evaluating to a boolean (`extracted.response.inputTokens > 100`), a list, and so on.

### Emission

When `tokenCompute` is set, the operator computes the charged value (`hits_addend` in Optimistic mode, `reservation.actualAmount` in Reservation mode) by substituting each referenced `extracted.response.<name>` with the extraction of that target's resolved pointer candidates — the `responseBodyJSON([...candidates...], "number")` form RFC 0024 already emits for `totalTokens` — and wrapping the whole expression so the charged value is always a non-negative integer: fractional parts truncated toward zero, a negative result clamped to `0`. The author never writes the conversion, and a negative intermediate never reaches the counter. A target whose candidates do not resolve substitutes to a `null` value (best-effort extraction, see [wasm-shim](#wasm-shim)), which the author's `tokenCompute` either defaults with a ternary or lets propagate to a fail-open miss; see [Missing fields](#missing-fields-best-effort-extraction-and-defaults). A future `extracted.request.<name>` would substitute to `requestBodyJSON(...)` by the same rule. When `tokenCompute` is unset, emission is unchanged from RFC 0024 (`totalTokens`'s pointer list, used directly).

The result is an ordinary CEL expression over one or more `responseBodyJSON([...], "number")` calls — the shape this policy kind already generates — so it reuses RFC 0024's existing body extraction and deduplication with no new machinery: each distinct target name the expression references is parsed once, in the single forward pass over the response body RFC 0024 already performs. Only names actually referenced in `tokenCompute` are extracted; a declared-but-unreferenced target costs nothing.

## wasm-shim

One change: **best-effort extraction.** Today a `responseBodyJSON(...)` whose candidates never resolve fails the extraction task at end-of-stream, aborting the action. For `tokenCompute` to treat an unresolved field as a `null` the author can test (see [Missing fields](#missing-fields-best-effort-extraction-and-defaults)), an unresolved group must instead resolve to `null` and let evaluation proceed. Readiness is unchanged — a value is only finalized as `null` once the response stream has ended, never prematurely mid-stream. The existing "no candidate resolved" observability (log line / metric) is kept, moving from a task-failure signal to a per-field extraction-miss signal, so a miss stays diagnosable.

Everything else is unchanged and already supported: `responseBodyJSON` already returns `null` for an absent value, `x != null` and the ternary are standard CEL, and a `null` reaching arithmetic or the integer conversion already errors into the existing fail-open `FailureMode` path — so an unset-`tokenCompute` policy whose `totalTokens` does not resolve still fails open exactly as it does today. The substitution happens entirely inside kuadrant-operator — `extracted.*` never reaches wasm-shim, which sees only pointers and CEL and remains unaware of target names, so a new target name still needs no change there.

## limitador

No changes, for the same reason RFC 0024 needed none: Limitador receives a plain numeric `hits_addend`/`actualAmount` and has no knowledge of how it was computed.

# Security considerations

- **Bounding total extraction cost.** `dataExtraction.response`'s `MaxProperties=16` cap and each entry's `MaxItems=8` candidate-pointer cap already exist as of RFC 0024's implementation.
- **Target-name identifier validation is also an injection guard.** Requiring `dataExtraction.response` keys to match `^[A-Za-z_][A-Za-z0-9_]*$` and to not be CEL reserved words (both enforced by CRD validation at admission; see [CRD changes](#crd-changes)) is necessary for `extracted.response.<name>` to parse as CEL attribute access at all, but it also means a target name can never itself be crafted to break out of the generated CEL syntax when `tokenCompute` is rewritten into `responseBodyJSON` calls (see [Emission](#emission)) — the substitution operates on parsed CEL, not string concatenation, so this is defense in depth rather than the primary protection, but it closes off the class of bug entirely at the schema layer. (The pointer *values* already carry the analogous protection, via RFC 0024's `[^/~"\\]` pattern that keeps a pointer from embedding CEL/wasm string-escape characters.)

# Drawbacks

None significant.

# Rationale and alternatives

- **A `token('inputTokens')` function call** instead of `extracted.response.inputTokens` attribute access. Rejected: a target is fully determined by declaration (`dataExtraction.response` fixes its pointer candidates), so referencing one is just naming a declared value — dotted attribute access, not a runtime lookup. A function would misframe it, and diverge from every other reference in this system (`auth.*`, `request.*`, `ratelimit.*`).

# Prior art

- RFC 0024's own written text sketched a fixed-sibling-field version of this idea (`inputTokens`, `outputTokens`, `cachedTokens` as dedicated CRD fields) in its [Future possibilities](./0024-token-rate-limit-api-format-compatibility.md#future-possibilities) section, flagging the exact CEL surface/namespace as "TBD." Its implementation, [kuadrant-operator#2299](https://github.com/Kuadrant/kuadrant-operator/pull/2299), went further than that written sketch before this RFC was drafted: `dataExtraction.response` shipped as an open `map[string]JSONPointerCandidates` from the start, explicitly "laying the ground" (the PR's own description) for target names beyond `totalTokens`, including ones contributed by extensions rather than a policy author. This RFC's design is a direct continuation of that already-shipped shape, not a competing one: it adds the one constraint (CEL-identifier-safe keys) and the one new consumer (`tokenCompute`) needed to make those target names actually usable, rather than revisiting the shape itself.
- This policy kind's own `reservation.amount`/`reservation.ttl` (RFC 0021) are the direct precedent for a user-authored CEL field validated via `internal/cel`'s validator infrastructure and reported through the `Enforced` status condition, reused here unchanged for `tokenCompute`.

# Unresolved questions

- The exact wording and machine-readability (e.g. a dedicated reason string) of the `Enforced` condition message for each of `tokenCompute`'s validation failure modes is left to implementation.

# Future possibilities

## Per-limit `tokenCompute`

If a real need emerges for different limits within one policy to charge differently from the same extracted targets (e.g. a "free" tier counting only `outputTokens` while a "premium" tier counts a full weighted sum), `tokenCompute` could be added at the `TokenLimit` level as well, with the limit-level value overriding the policy-level one for that limit only. This is additive to this RFC's design, not a restructure: the policy-level field remains the default, a limit-level field simply shadows it.

## Built-in defaults for common target names

Like `totalTokens`, a documented handful of common dimensions — `inputTokens`, `outputTokens`, `cachedTokens` — could gain built-in default pointer lists, so a policy could reference `extracted.response.inputTokens` without declaring its candidates. This RFC ships those lists as documentation only (the [Extraction recipes](#extraction-recipes-non-normative) table), keeping the mechanism fully generic: a default pointer list is only meaningful for a name the operator has a hardcoded semantic understanding of, which `totalTokens` alone occupies today for RFC 0024 backward compatibility. If real usage shows authors reaching for the same few names repeatedly, promoting them to defaulted names is additive — it changes nothing about how an explicitly-declared target behaves.

## Request-side fields feeding `tokenCompute`

The source-qualified `extracted` namespace was chosen for this: once `dataExtraction.request` (RFC 0024's sketch, for RFC 0021 reservation sizing) materializes, request-side targets — e.g. `maxTokens` from the request body's `max_tokens` — would be referenced as `extracted.request.<name>` and substituted to `requestBodyJSON(...)` by the same rule, the only change being to accept `request` as a known source. The distinct sources also keep request- and response-side names from colliding.

# References

- [OpenAI chat][openai-chat]
- [Anthropic Messages][anthropic-messages]
- [Google Gemini generateContent][gemini-generate-content]
- [CEL conversion functions (`uint()`, etc.)][cel-conversions]

[openai-chat]: https://platform.openai.com/docs/api-reference/chat/create
[anthropic-messages]: https://docs.anthropic.com/en/api/messages
[gemini-generate-content]: https://ai.google.dev/api/generate-content#v1beta.GenerateContentResponse
[cel-conversions]: https://github.com/google/cel-spec/blob/master/doc/langdef.md#conversions
