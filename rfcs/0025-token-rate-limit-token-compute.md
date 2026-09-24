# Token Rate Limit Multi-Field Extraction and Custom Token Compute

- Feature Name: `token_rate_limit_token_compute`
- Status: Draft
- Start Date: 2026-09-22
- RFC PR: <https://github.com/Kuadrant/architecture/pull/0000>
- Issue tracking: TBD
- Related: [RFC 0013 (AI policies)](./0013-ai-policies.md), [RFC 0021 (Token rate limit reservations)](./0021-token-rate-limit-reservations.md), [RFC 0024 (Token rate limit API format compatibility)](./0024-token-rate-limit-api-format-compatibility.md), [kuadrant-operator#2299](https://github.com/Kuadrant/kuadrant-operator/pull/2299) (RFC 0024 implementation, open), [wasm-shim#424](https://github.com/Kuadrant/wasm-shim/pull/424) (RFC 0024 implementation, open)

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

`inputTokens` and `outputTokens` above are not special, recognized names: they're names this policy's author picked, in this case ones that read naturally against the values they're pointing at. A target name is free to be `promptSize`, `reasoningTokens`, or anything else that's a valid CEL identifier (letters, digits, underscore, not starting with a digit — see [Reference-level explanation](#reference-level-explanation) for why this constraint exists now, when it didn't before). Unlike `totalTokens`, any other target name has **no built-in default**: the operator has no way to guess a candidate pointer list for a name it has no hardcoded concept of, so every other entry must spell out its own pointer list explicitly. To make this easy for the token dimensions most providers already expose, this RFC ships documentation (a "recipe" table, not code) with ready-to-copy candidate lists for common dimensions like input/output/cached tokens across OpenAI, Anthropic, and Gemini shapes — see [Extraction recipes](#extraction-recipes-non-normative). Copying one of those recipes under whatever name you pick is the expected way to use this, not a requirement to use those exact names.

### Computing the charged value: `tokenCompute`

A new, optional, policy-level field, `tokenCompute`, is a CEL expression that replaces `totalTokens` as the source of the value charged against every limit in the policy (`hits_addend` in Optimistic mode, `reservation.actualAmount` in Reservation mode). It references any target name present in `dataExtraction.response` as `tokens.<name>`:

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
  tokenCompute: "uint(tokens.inputTokens + tokens.outputTokens)"
  limits:
    free:
      rates:
      - limit: 20000
        window: 1d
      counters:
      - expression: auth.identity.userid
```

Or, weighting cached tokens at 40% of a normal token's cost:

```yaml
spec:
  dataExtraction:
    response:
      inputTokens: ["/usage/input_tokens"]
      outputTokens: ["/usage/output_tokens"]
      cachedTokens: ["/usage/cache_read_input_tokens"]
  tokenCompute: "uint(tokens.outputTokens + tokens.inputTokens - 0.6 * tokens.cachedTokens)"
```

Every name referenced as `tokens.<name>` in `tokenCompute` must be either `totalTokens` (which always resolves, explicit or defaulted) or a target name actually present under this policy's (effective, post defaults/overrides) `dataExtraction.response` — nothing else resolves, by design; see [Validation](#validation-1). `tokenCompute` must evaluate to a `uint`, not an `int` or a `double`. Every extracted quantity is a floating-point number (a JSON number can be fractional, and this system never assumes otherwise), so any arithmetic over them is a `double`, and CEL's own type checker treats `0.4 * tokens.inputTokens + 0.6 * tokens.outputTokens` as a `double` expression, not a `uint` one, even though the values it reads are, in practice, almost always integral token counts. This is why every example above wraps the whole expression in CEL's built-in `uint(...)` conversion: it's not decorative, it's what makes the expression pass validation. `uint(...)` also gives an existing, real CEL guarantee for free: it truncates any fractional part, and it raises a CEL evaluation error if the value is negative, rather than silently wrapping to a huge unsigned number. A `tokenCompute` that happens to go negative for a given response (e.g. a large negative weight over-subtracting) therefore fails that one response's extraction the same way an unresolved pointer does today, it does not corrupt a limit's count; see RFC 0024's [Observability of extraction misses](./0024-token-rate-limit-api-format-compatibility.md#observability-of-extraction-misses).

`tokenCompute` is a single, policy-level field (like `dataExtraction`, it participates in the existing `defaults`/`overrides` hierarchy) — every limit in the policy is charged with the same computed value. A future RFC could move it, or add a per-limit override, if a real need for different limits in the same policy to be charged differently emerges; see [Future possibilities](#future-possibilities).

### Extraction recipes (non-normative)

Not part of the API, just documentation shipped alongside it, for the dimensions most policy authors will actually reach for:

| Concept | Suggested target name | Candidate pointers |
|---|---|---|
| Prompt/input tokens | `inputTokens` | `/usage/prompt_tokens` (OpenAI-compatible), `/usage/input_tokens` (Anthropic), `/usageMetadata/promptTokenCount` (Gemini) |
| Completion/output tokens | `outputTokens` | `/usage/completion_tokens` (OpenAI-compatible), `/usage/output_tokens` (Anthropic), `/usageMetadata/candidatesTokenCount` (Gemini) |
| Cached/reused tokens | `cachedTokens` | `/usage/prompt_tokens_details/cached_tokens` (OpenAI-compatible), `/usage/cache_read_input_tokens` (Anthropic), `/usageMetadata/cachedContentTokenCount` (Gemini) |

These are suggestions, not reserved or recognized names: nothing in the operator treats `inputTokens` as special, the same way nothing treats a policy's counter names or limit names as special. A policy fronting only Anthropic could just as well name the target `promptTokens` and point it only at `/usage/input_tokens`.

### Validation: unknown target names and non-`uint` results are rejected

Two mistakes are caught before a bad `tokenCompute` can silently mis-charge a limit:

- **Referencing a name that isn't `totalTokens` and isn't declared under this policy's `dataExtraction.response`** (a typo, or a name from a target the policy forgot to declare) is rejected.
- **An expression whose statically checked result type isn't `uint`** (e.g. forgetting the `uint(...)` wrapper) is rejected, exactly as illustrated above.

Both are reported the same way RFC 0021's reservation `ttl`/`amount` CEL validation errors are reported today: the policy's `Enforced` status condition is set to `False`, with a message naming the problem (e.g. `tokenCompute references target "promptTokens", which is not "totalTokens" and not declared under dataExtraction.response` or `tokenCompute must evaluate to uint, got double`). This is a reconcile-time check, not (at least initially) an admission-webhook check; see [Validation](#validation-1) for why, and [Unresolved questions](#unresolved-questions) for tightening this to admission time later.

### Migration

Existing `TokenRateLimitPolicy` resources need no changes. `tokenCompute` unset behaves exactly as RFC 0024 specifies: the value charged against every limit is `tokens.totalTokens`, resolved via `dataExtraction.response.totalTokens` (explicit or defaulted). Nothing about `dataExtraction.response`'s CRD schema changes for this RFC other than the one additional constraint described in [Reference-level explanation](#reference-level-explanation); a policy that already declares extra target names it doesn't use from anywhere is unaffected, since the operator ignoring an unused target name is exactly today's behavior too.

# Reference-level explanation

## kuadrant-operator

### CRD changes

RFC 0024's implementation ([kuadrant-operator#2299](https://github.com/Kuadrant/kuadrant-operator/pull/2299)) already shipped `dataExtraction.response` as an open map, not a fixed-field struct, specifically so that "additional extraction targets can be added later — including by extensions — without further CRD/API changes" (the PR's own words):

```go
// ResponseDataExtractionKeyTotalTokens is the key under which an ordered list of JSON Pointer
// candidates for total token usage extraction is stored in a ResponseDataExtraction map.
const ResponseDataExtractionKeyTotalTokens = "totalTokens"

// JSONPointerCandidates is an ordered list of JSON Pointer (RFC 6901) expressions evaluated against
// the response body. The first pointer that resolves to a numeric value is used.
// +kubebuilder:validation:MinItems=1
// +kubebuilder:validation:MaxItems=8
// +kubebuilder:validation:items:Pattern=`^(/([^/~"\\]|~[01])*)+$`
type JSONPointerCandidates []string

// ResponseDataExtraction maps an extraction target name (e.g. "totalTokens", potentially
// contributed by an extension) to the JSONPointerCandidates used to resolve it.
// +kubebuilder:validation:MaxProperties=16
type ResponseDataExtraction map[string]JSONPointerCandidates
```

This RFC needs exactly one addition to that already-shipped shape: a constraint on the map's *keys*, needed only now that a target name is about to become something referenced from CEL (`tokens.<name>`), which `totalTokens`-by-fixed-constant-lookup never required:

```go
// ResponseDataExtraction maps an extraction target name ...
// +kubebuilder:validation:MaxProperties=16
// +kubebuilder:validation:XValidation:rule="self.all(k, k.matches('^[A-Za-z_][A-Za-z0-9_]*$'))",message="extraction target names must be valid CEL identifiers"
type ResponseDataExtraction map[string]JSONPointerCandidates
```

`TokenRateLimitPolicySpecProper` gains one new field, at the same level as `DataExtraction`, for the same reason `DataExtraction` lives there and not inside individual `TokenLimit` entries — it's a policy-wide property, shared by every limit:

```go
type TokenRateLimitPolicySpecProper struct {
    kuadrantv1.MergeableWhenPredicates `json:""`
    Limits         map[string]TokenLimit `json:"limits,omitempty"`
    DataExtraction *DataExtraction       `json:"dataExtraction,omitempty"`

    // TokenCompute is a CEL expression computing the value charged against every limit in this
    // policy (hits_addend in Optimistic mode, reservation.actualAmount in Reservation mode), in
    // place of tokens.totalTokens, the default. It may reference tokens.<name> for "totalTokens"
    // or any target name present in dataExtraction.response, and must evaluate to uint. If unset,
    // tokens.totalTokens is used directly, unchanged from RFC 0024.
    // +optional
    TokenCompute *string `json:"tokenCompute,omitempty"`
}
```

`TokenCompute` participates in `Rules()`/`SetRules()` and the `defaults`/`overrides` merge hierarchy exactly like `DataExtraction` does (a new `RulesKeyTokenCompute` sentinel key, mirroring `RulesKeyDataExtraction`), and gets the same mutual-exclusivity `XValidation` rules already guarding `dataExtraction` against implicit defaults/overrides:

```go
// +kubebuilder:validation:XValidation:rule="!(has(self.defaults) && has(self.tokenCompute))",message="Implicit tokenCompute and explicit defaults are mutually exclusive"
// +kubebuilder:validation:XValidation:rule="!(has(self.overrides) && has(self.tokenCompute))",message="Implicit tokenCompute and overrides are mutually exclusive"
```

`DataExtraction` as a whole remains a single `MergeableRule` (unchanged from RFC 0024): a more specific policy's `dataExtraction` replaces the entire `response` map atomically — it does not merge key-by-key with a less specific policy's `response`. A route-level policy that wants to keep a gateway-level policy's `inputTokens` entry and add its own `cachedTokens` entry must repeat both under its own `dataExtraction.response`, exactly as it already has to repeat `totalTokens` today if it needs to add anything else to `dataExtraction.response`.

A generalized accessor complements the existing `ResponseTotalTokensPointers()` (unchanged):

```go
// ResponseTargetPointers resolves the effective ordered list of JSON Pointer candidates for an
// arbitrary target name: an explicit entry in dataExtraction.response.<name> if present, else
// DefaultTotalTokensPointers if name is "totalTokens", else not found. Safe to call on a nil receiver.
func (d *DataExtraction) ResponseTargetPointers(name string) ([]string, bool) {
    if d != nil {
        if pointers := d.Response[name]; len(pointers) > 0 {
            return []string(pointers), true
        }
    }
    if name == ResponseDataExtractionKeyTotalTokens {
        return DefaultTotalTokensPointers, true
    }
    return nil, false
}
```

### Validation

Two checks run against a policy's effective `tokenCompute` string, both reusing the CEL validator infrastructure `internal/cel/kuadrant_validator.go` already provides for this policy kind's other user-authored CEL fields (`when`/limit predicates, `reservation.amount`, `reservation.ttl`):

1. **Known-target check.** `tokenCompute` is parsed into a CEL AST (`cel-go`'s parser, already a direct dependency). Every `Select` expression whose operand is the bare identifier `tokens` is collected; each such target name must resolve via `ResponseTargetPointers` above — i.e. it's either `totalTokens`, or a key actually present in *this policy's own, effective (post defaults/overrides)* `dataExtraction.response`. This membership set is computed per policy from what that policy actually declares, not from a fixed enum, so introducing a new target name is purely a data change to one policy, never a code change to the operator — exactly the property `dataExtraction.response`'s open map shape was already built for. This is a plain Go-level AST walk, not something delegated to `cel-go`'s type checker, because a map-typed binding (see below) cannot statically distinguish a valid key from an invalid one — CEL's checker has no concept of "this map only has these particular keys, computed from another part of the same resource."
2. **Result-type check.** `NewRootValidatorBuilder()` gains a new binding, `tokens`, declared as `cel.MapType(cel.StringType, cel.DoubleType)` — a map from target name to a floating-point number, which is what every extracted quantity actually is (a JSON number, extracted the same way `totalTokens` is today). `tokenCompute` is then run through the existing `validator.Validate(...)` (parse + check), and the resulting AST's `OutputType()` must equal `cel.UintType`, exactly the same pattern `internal/cel/kuadrant_validator.go`'s `ValidateWasmActionSpec` already applies to `reservation.ttl`, which must check as `cel.DurationType`. This is what forces the `uint(...)` wrapper illustrated throughout [Guide-level explanation](#guide-level-explanation): without it, arithmetic over `double`-typed values checks as `double`, and the policy is rejected. This binding's shape (`map[string]double`) is the same regardless of which target names a given policy happens to declare, since the *set* of valid names is enforced separately by the known-target check above, not by the type of the `tokens` binding itself.

Both checks produce a `cel.Issue` (`internal/cel`'s existing type), collected into the same `IssueCollection` stored under `cel.StateCELValidationErrors` in reconcile-scoped `state` that `istio_extension_reconciler.go`/`envoy_gateway_extension_reconciler.go` already populate for this policy kind's other CEL fields. `TokenRateLimitPolicyStatusUpdater.enforcedCondition` already reads exactly this state to compose the `Enforced` condition; a `tokenCompute` validation failure surfaces there with no new status plumbing.

This validation runs at reconcile time, not (initially) at admission. That matches this policy kind's existing precedent: `reservation.amount` and `reservation.ttl` are CEL-validated the same way today, at reconcile time via this same pipeline, not via a validating webhook. A cheap admission-time syntax check (is `tokenCompute` parseable CEL at all) could be added as a fast-fail improvement without needing the full `tokens`-aware environment, but the semantic checks above (known targets for *this* policy, result type) are not moved to admission time by this RFC; see [Unresolved questions](#unresolved-questions).

### Emission: rewriting `tokenCompute` into `responseBodyJSON` calls (`internal/controller/ratelimit_workflow_helpers.go`)

Today (RFC 0024), `ResponseBodyJSONTotalTokensCEL(totalTokensPointers)` builds the single CEL string, e.g. `responseBodyJSON(["/usage/total_tokens", "/usageMetadata/totalTokenCount"], "number")`, used verbatim as `ratelimit.hits_addend`'s value (Optimistic mode, `tokenCheckReportSpecs`) and as `reservation.actualAmount` (Reservation mode, `tokenReservationSpecs`). This RFC adds a second path, used whenever the effective policy sets `tokenCompute`:

1. Parse `tokenCompute` into a CEL AST (the same parse already performed for validation).
2. Walk the AST, replacing every `Select` node of the form `tokens.<name>` with a `Call` node for `responseBodyJSON([...pointers...], "number")`, where `...pointers...` comes from `ResponseTargetPointers(<name>)` — guaranteed to resolve by the [Known-target check](#validation-1) that already ran before emission is ever reached, so no further fallback applies here.
3. Render the rewritten AST back to CEL source text (`cel-go`'s unparser) and use that string as `hits_addend` / `actualAmount`, in place of `ResponseBodyJSONTotalTokensCEL(totalTokensPointers)`.

When `tokenCompute` is unset, nothing changes: `ResponseBodyJSONTotalTokensCEL(totalTokensPointers)` is used exactly as RFC 0024 specifies.

The rewritten expression is, syntactically, no different from any other CEL string this policy kind already generates: an arithmetic expression over one or more `responseBodyJSON([...], "number")` calls. It therefore flows unmodified through `internal/wasm/action_spec.go`'s existing `BuildActions` body-ref dedup pipeline, which RFC 0024 built specifically to collapse every `responseBodyJSON`/`requestBodyJSON` call across a policy's action specs into a single `StoreAction` per direction, keyed by field name (`kuadrant.internal.response.body.<fieldName>`). A `tokenCompute` referencing `tokens.inputTokens + tokens.outputTokens` becomes, after this rewrite and then dedup, two ordinary body refs (`input_tokens`, `output_tokens`) parsed in the same single forward pass over the response body that RFC 0024 already performs for one field — no new dedup logic, no new store-path convention, and no increase in the number of body parses per response beyond what the number of *distinct target names the expression actually references* requires. A policy that declares three target names under `dataExtraction.response` but only references two of them in `tokenCompute` never emits pointers for the third: emission is driven by what the AST walk visits, not by what's configured.

## wasm-shim

No changes. `tokens.*` never reaches wasm-shim; the substitution in [Emission](#emission-rewriting-tokencompute-into-responsebodyjson-calls-internalcontrollerratelimit_workflow_helpersgo) happens entirely inside kuadrant-operator, before wasm config is generated. From wasm-shim's perspective, this policy shape is indistinguishable from one that always existed: an arbitrary CEL arithmetic expression over multiple `responseBodyJSON([...], "number")` calls feeding into `hits_addend`, which wasm-shim's CEL evaluator already supports today (`hits_addend` has never been restricted to a single body reference; RFC 0021's reservation `amount` is already an arbitrary CEL expression, for instance). The `uint(...)` wrapper this RFC's validation mandates is itself an ordinary CEL standard-library conversion, needing no wasm-shim-side support beyond what evaluating any other CEL expression already requires. Because the set of extractable target names is entirely open at the kuadrant-operator/CRD layer, wasm-shim needs no change at all to support a name nobody has thought of yet — it was never aware of names in the first place, only pointers.

## `limitador`

No changes, for the same reason RFC 0024 needed none: Limitador receives a plain numeric `hits_addend` and has no knowledge of how it was computed.

# Security considerations

- **Bounding total extraction cost.** `dataExtraction.response`'s `MaxProperties=16` cap and each entry's `MaxItems=8` candidate-pointer cap already exist as of RFC 0024's implementation, independent of this RFC; worst case a policy can name up to 16 targets with up to 8 candidates each. As with RFC 0024's own reasoning for that cap, it is fixed at configuration time (not attacker-influenced at runtime); the only thing an upstream response actually controls is how many of those configured candidates it takes to find a match, not how many could exist. This RFC does not change that bound, and `tokenCompute` cannot reference more targets than are declared, so it introduces no new way to exceed it.
- **Target-name identifier validation is also an injection guard.** Requiring `dataExtraction.response` keys to match `^[A-Za-z_][A-Za-z0-9_]*$` (enforced by CRD validation at admission) is necessary for `tokens.<name>` to parse as CEL attribute access at all, but it also means a target name can never itself be crafted to break out of the generated CEL syntax during the AST-based rewrite in [Emission](#emission-rewriting-tokencompute-into-responsebodyjson-calls-internalcontrollerratelimit_workflow_helpersgo) — the rewrite operates on parsed CEL AST nodes, not string concatenation, so this is defense in depth rather than the primary protection, but it closes off the class of bug entirely at the schema layer.

# Drawbacks

- `tokenCompute`'s two-phase design (validate against a `tokens`-typed CEL environment, then rewrite into a differently-shaped CEL string wasm-shim actually evaluates) is more machinery than RFC 0024's direct "emit the pointer list as-is" path, and the AST rewrite step is new surface in kuadrant-operator with no precedent elsewhere in this codebase (the closest existing thing, `BuildActions`' body-ref dedup, rewrites known call *shapes*, not arbitrary user-authored expressions).
- An open, per-policy-defined target namespace means the "known target" check's error messages and behavior depend on what else the same policy declares, which is a slightly less predictable validation surface than checking against one fixed, globally documented list of names — a typo in a target's declared name under `dataExtraction.response` and a typo in `tokenCompute`'s reference to it are two different mistakes that both surface as "unknown target," and a policy author has to cross-reference both parts of the spec to tell them apart.
- Reconcile-time-only validation (see [Validation](#validation-1)) means a policy with an invalid `tokenCompute` is accepted by the API server and only shows the problem in its `Enforced` status condition afterward, which is a weaker guarantee than admission-time rejection, though it matches this policy kind's existing precedent for `reservation.amount`/`reservation.ttl`.

# Rationale and alternatives

- **Referencing `dataExtraction.response`'s existing open map directly**, rather than adding a new, separate mechanism (a fixed set of sibling fields, or a further-nested sub-object) for the target names `tokenCompute` can use. Rejected any such addition: `dataExtraction.response` was already generalized, in RFC 0024's own implementation, into an open `map[string]JSONPointerCandidates` precisely so new target names never need a CRD change; introducing a second, differently-shaped mechanism alongside it for this RFC's purposes would duplicate that generalization for no reason. The only real gap `tokenCompute` exposes is that target names, once referenced from CEL, need to be valid identifiers — a constraint the existing map didn't need when its only consumer looked up one fixed constant.
- **No built-in defaults for target names other than `totalTokens`.** Considered shipping built-in default pointer lists for a documented handful of "common" names (e.g. treating `inputTokens`/`outputTokens`/`cachedTokens` as reserved, defaulted names). Rejected: a default pointer list is only meaningful for a name the operator has a hardcoded semantic understanding of, which is precisely what an open namespace gives up, and `totalTokens` already occupies that one special-cased role for backward compatibility with RFC 0024. The [Extraction recipes](#extraction-recipes-non-normative) documentation table replaces implicit defaults with copy-paste guidance instead, keeping the mechanism itself fully generic.
- **AST rewrite of `tokens.<name>` into `responseBodyJSON(...)` calls**, rather than teaching wasm-shim a new `tokens` namespace natively. Considered the alternative of exposing a `tokens` map (or well-known attribute) directly inside wasm-shim's CEL environment, populated by a `StoreAction` per target (the `store` action type's `path`/`exportToHost` mechanism already exists for cross-action CEL references). Rejected for this RFC: it would require wasm-shim to learn a new well-known-attribute namespace and kuadrant-operator to emit an extra `StoreAction` per referenced target, for no material benefit over rewriting at the operator, since the operator already fully controls and validates `tokenCompute` before any wasm config exists. Keeping the rewrite entirely inside kuadrant-operator also composes better with `dataExtraction.response`'s open shape: wasm-shim's contract (arbitrary CEL over `responseBodyJSON`/`requestBodyJSON` calls) doesn't need to grow at all, no matter how many target names, or which ones, a given policy or extension declares.
- **Bare identifiers** (`inputTokens + outputTokens`, no `tokens.` prefix) instead of the `tokens.` namespace. Rejected: bare identifiers risk colliding with other well-known attributes this system may add later (`auth.*`, `request.*`, `response.*`, `ratelimit.*` all already exist as dotted namespaces), and with an open, policy-defined target namespace this risk is worse, not better, since a policy author could unknowingly pick a name that collides with a future built-in attribute.
- **A `token('inputTokens')` function call** instead of `tokens.inputTokens` attribute access. Rejected: every other reference into extracted or contextual data in this system already uses dotted attribute access, not a function call; introducing a function here would be an inconsistent, one-off surface for no expressiveness gain.
- **Requiring `tokenCompute` to statically check as `uint`**, rather than allowing `int`/`double` and having kuadrant-operator or wasm-shim silently round or truncate on its behalf. Rejected: an implicit, silent numeric conversion hides a lossy step from the policy author exactly where correctness matters most (what gets charged against a limit), and the explicit `uint(...)` wrapper this RFC requires is not a new burden, it's the same standard CEL conversion function available today, and its runtime failure mode (error on negative, not wraparound) is strictly safer than a silent implicit cast would be.
- **Per-limit `tokenCompute`** instead of policy-level. Considered, since different limits in one policy could plausibly want different charged values from the same extracted targets. Deferred: no concrete use case motivates it yet, policy-level keeps the field consistent with `dataExtraction`'s own placement and rationale, and adding a per-limit override later is additive, not a breaking change; see [Future possibilities](#future-possibilities).

# Prior art

- RFC 0024's own written text sketched a fixed-sibling-field version of this idea (`inputTokens`, `outputTokens`, `cachedTokens` as dedicated CRD fields) in its [Future possibilities](./0024-token-rate-limit-api-format-compatibility.md#future-possibilities) section, flagging the exact CEL surface/namespace as "TBD." Its implementation, [kuadrant-operator#2299](https://github.com/Kuadrant/kuadrant-operator/pull/2299), went further than that written sketch before this RFC was drafted: `dataExtraction.response` shipped as an open `map[string]JSONPointerCandidates` from the start, explicitly "laying the ground" (the PR's own description) for target names beyond `totalTokens`, including ones contributed by extensions rather than a policy author. This RFC's design is a direct continuation of that already-shipped shape, not a competing one: it adds the one constraint (CEL-identifier-safe keys) and the one new consumer (`tokenCompute`) needed to make those target names actually usable, rather than revisiting the shape itself.
- This policy kind's own `reservation.amount`/`reservation.ttl` (RFC 0021) are the direct precedent for a user-authored CEL field validated via `internal/cel`'s validator infrastructure and reported through the `Enforced` status condition, reused here unchanged for `tokenCompute`.

# Unresolved questions

- Whether `tokenCompute`'s known-target and result-type checks should move to admission-time (a validating webhook), not just reconcile-time, once the operator has a general pattern for admission-time CEL validation against a custom, per-resource environment (the per-policy nature of the known-target set makes this somewhat more involved than a fixed global environment would be). Reconcile-time validation (this RFC's design) is consistent with existing precedent for this policy kind's other CEL fields, but is a weaker guarantee than rejecting a bad policy at `kubectl apply` time.
- Whether a per-limit `tokenCompute` override becomes necessary once real usage surfaces a case for different limits in the same policy charging differently from the same extracted targets; see [Future possibilities](#future-possibilities).
- The exact wording and machine-readability (e.g. a dedicated reason string) of the `Enforced` condition message for each of `tokenCompute`'s validation failure modes is left to implementation.

# Future possibilities

## Per-limit `tokenCompute`

If a real need emerges for different limits within one policy to charge differently from the same extracted targets (e.g. a "free" tier counting only `outputTokens` while a "premium" tier counts a full weighted sum), `tokenCompute` could be added at the `TokenLimit` level as well, with the limit-level value overriding the policy-level one for that limit only. This is additive to this RFC's design, not a restructure: the policy-level field remains the default, a limit-level field simply shadows it.

## Extension-contributed target names

[kuadrant-operator#2299](https://github.com/Kuadrant/kuadrant-operator/pull/2299)'s own description calls out that `dataExtraction.response`'s open shape allows target names to be "contributed by an extension," not just typed in directly by a policy author. Should that materialize — an out-of-process extension (per this operator's existing Extension Architecture) populating a target name a `TokenRateLimitPolicy` didn't declare itself — `tokenCompute` would reference it exactly the same way, as `tokens.<name>`, with no change to the mechanism this RFC describes.

## Request-side fields feeding `tokenCompute`

RFC 0024's sketched `dataExtraction.request` side (for RFC 0021's reservation-sizing use case) would, if it materializes, plug into the same open-map pattern and the same rewrite mechanism this RFC describes, once a real request-side consumer exists.

# References

- [OpenAI chat][openai-chat]
- [Anthropic Messages][anthropic-messages]
- [Google Gemini generateContent][gemini-generate-content]
- [CEL conversion functions (`uint()`, etc.)][cel-conversions]

[openai-chat]: https://platform.openai.com/docs/api-reference/chat/create
[anthropic-messages]: https://docs.anthropic.com/en/api/messages
[gemini-generate-content]: https://ai.google.dev/api/generate-content#v1beta.GenerateContentResponse
[cel-conversions]: https://github.com/google/cel-spec/blob/master/doc/langdef.md#conversions
