# Token Rate Limit Soft / Hard Boundaries

- Feature Name: `token_rate_limit_soft_hard_boundaries`
- Status: Draft
- Start Date: 2026-07-24
- RFC PR: (to be filled when opened against [Kuadrant/architecture](https://github.com/Kuadrant/architecture))
- Issue tracking: (to be filled)
- Authors: James Land (MaaS / RHOAI), with input from Eguzki Astiz Lezaun / Craig Brookes (Kuadrant)
- Related: [RFC 0013 — AI Policies / `TokenRateLimitPolicy`](./0013-ai-policies.md)
- Context: Soft/hard framing from Kuadrant community call (idea attributed to Alex); follow-up with MaaS

# Summary
[summary]: #summary

Today `TokenRateLimitPolicy` treats every limit as a **hard** gate: when a counter is exhausted the request is rejected with `429`. That is necessary for capacity protection, but it is too blunt for real user use cases. Customers need **flexible business logic** after a budget is crossed — deprioritize traffic, route to a cheaper model, keep metering against a team pool, degrade quality — without Kuadrant encoding every product policy in the rate-limit plane.

**Ask:** support **soft and hard boundaries** on token rate limits:

- **Soft boundary exceeded** → request is **still admitted**, and the gateway injects **configurable headers** (both key and value under policy control) marking which limit(s) were exceeded.
- **Hard boundary exceeded** → request is **rejected** with `429` (today’s behavior).

**Every matching limit should be evaluated**, not fail-fast on the first over-limit. One request may hit multiple soft buckets (and possibly a hard one), producing multiple headers or multiple values. Downstream plugins own what to do with those signals; Kuadrant owns metering and signaling.

# Motivation
[motivation]: #motivation

Conversations with customers make one point clear: **flexibility matters**. Hard “deny with 429” cannot cover every enforcement story:

| Need (examples) | Hard 429 alone |
|---|---|
| Over personal quota → deprioritize / smaller model, still count against team | Blocks traffic that should continue under different handling |
| Over soft SLA → mark non-priority / lower QoS | Cannot signal; only deny or allow |
| Over budget → route to a cheaper model | Needs a signal, not a reject |

Hard limiting should remain available as a **safety ceiling**. Soft boundaries separate *“did this request cross a budget?”* from *“what should we do about it?”*.

## Soft / hard framing

Aligned with the community call discussion:

1. Cross the **soft** boundary → traffic continues, with classification / rate-limit headers attached.
2. Cross the **hard** boundary → traffic is rejected with `429`.

Existing informational headers (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`) stay useful. We also need **policy-authored headers** (configurable name + value) so platforms can drive downstream behavior.

## Multi-bucket evaluation

Limits are often layered, not a single counter. One illustrative idea:

1. **Personal soft limit exceeded** — traffic is deprioritized and may go to a smaller / cheaper model, but the request is still admitted.
2. **Team limit still applies** — that traffic continues to consume tokens against the team budget.
3. **Optional hard ceiling** — e.g. team or model capacity exhausted → `429`.

Today’s all-or-nothing deny cannot express (1) while still metering (2). Evaluating **all matching rules**, accumulating soft signals, and only denying on **hard** failure is what unlocks that.

## Expected outcome

1. Per-limit **soft vs hard** behavior (default remains today’s hard deny for compatibility).
2. On soft exceed: **admit** and attach **author-controlled header name(s) and value(s)**.
3. **Evaluate every matching limit** so multiple soft signals can fire on one request (and a hard limit can still 429).
4. Soft-exceeded traffic should **still count** against the relevant counters — soft means “don’t reject,” not “stop metering.”

How Kuadrant wires Limitador, wasm-shim, and CRD details is up to the Kuadrant team; the guide-level example below is only to make the ask concrete.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Not a prescribed design — one way `TokenRateLimitPolicy` *could* look once soft/hard boundaries exist:

```yaml
apiVersion: kuadrant.io/v1alpha1
kind: TokenRateLimitPolicy
metadata:
  name: example-token-limits
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: my-llm-gateway
  limits:
    # Soft: over personal budget → annotate, still allow
    personal-daily:
      when:
      - predicate: 'auth.identity.userid != ""'
      rates:
      - limit: 50000
        window: 1d
      counters:
      - expression: auth.identity.userid
      boundary: Soft   # default remains Hard for compatibility
      softSignal:
        # Both key and value policy-controlled
        headers:
        - name: "X-Kuadrant-Limit-Exceeded"
          value: "personal-daily"
        - name: "X-MaaS-Traffic-Class"
          value: "non-priority"

    # Still metered (soft or hard) — e.g. team budget
    team-daily:
      when:
      - predicate: 'auth.identity.organization != ""'
      rates:
      - limit: 500000
        window: 1d
      counters:
      - expression: auth.identity.organization
      boundary: Soft
      softSignal:
        headers:
        - name: "X-Kuadrant-Limit-Exceeded"
          value: "team-daily"

    # Hard: absolute capacity — reject with 429
    model-capacity:
      rates:
      - limit: 2000000
        window: 1m
      counters:
      - expression: '"model:" + requestBodyJSON("/model")'
      boundary: Hard
```

Example outcomes with that idea:

| Personal | Team | Model capacity | Result |
|---|---|---|---|
| under | under | under | Admit; no soft headers from these limits |
| **over** | under | under | Admit; soft headers (e.g. `personal-daily`, `non-priority`) |
| **over** | **over** | under | Admit; both soft signals (e.g. merged `X-Kuadrant-Limit-Exceeded: personal-daily,team-daily`) |
| under | under | **over** | **429** |
| **over** | under | **over** | **429** (whether soft headers also appear on the deny is an open question) |

Downstream filters can then deprioritize, pick a smaller model, etc. Hard `model-capacity` stays the ceiling.

**For existing users:** policies with no new fields keep today’s hard-deny behavior.

**For new users:** think of soft limits as *meters that signal*, and hard limits as *meters that reject*. Product logic (priority, routing, QoS) lives downstream of Kuadrant.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This change request intentionally does **not** prescribe Limitador RPCs, wasm-shim control flow, or exact CRD types. Those belong to Kuadrant design once the ask is accepted.

Minimum behavioral contract from a consumer perspective:

- Soft exceed → admit + configurable request (and/or response) headers; counters still updated.
- Hard exceed → `429`.
- All matching limits evaluated; multiple soft signals allowed on one request.
- Omitted boundary ⇒ Hard (backward compatible).

# Drawbacks
[drawbacks]: #drawbacks

- Admission path is more than binary check → 429.
- Soft-admit past a numeric limit may surprise operators who assume “limit” always means “deny”; naming and docs must be clear.
- Downstream plugins become responsible for product behavior; misconfigured soft limits can burn capacity until a hard ceiling hits.
- Evaluate-all may cost more Limitador work than fail-fast (to be measured).

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Headers (configurable key and value) keep Kuadrant focused on metering/signaling and leave product actions (deprioritize, change model, etc.) to downstream plugins. Alternatives such as embedding routing/priority actions in `TokenRateLimitPolicy`, shadow mode without signals, or a fixed header vocabulary are either too rigid or push product logic into the wrong layer.

Impact of not doing this: platforms that need graduated handling after a budget is crossed cannot use `TokenRateLimitPolicy` alone without either hard-denying useful traffic or abandoning accurate multi-bucket metering.

# Prior art
[prior-art]: #prior-art

Soft/hard quotas appear in batch/HPC schedulers and cloud “burst” SKUs. Many API gateways emit `X-RateLimit-*` while still serving (client-driven soft). This proposal is closer to *server-side classification*: continue serving, but mark the request for downstream policy.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Request headers vs response headers (or both) for soft signals?
- Soft headers on `429` responses when a hard limit also fired?
- Default header if `boundary: Soft` but no explicit softSignal — e.g. `X-Kuadrant-Limit-Exceeded: <limit-name>`?
- CEL for header values, or static strings first?
- Same model useful later for request-based `RateLimitPolicy`?

# Future possibilities
[future-possibilities]: #future-possibilities

- Structured dynamic metadata instead of/in addition to headers for internal hops.
- Applying the same soft/hard model to `RateLimitPolicy`.
- Richer downstream ecosystems (priority, model selection) keyed off soft signals.

# Success criteria

- Soft limit exceeded → traffic continues with configurable headers; no `429` from that limit alone.
- Multiple matching soft limits → multiple signals on the same request.
- Hard limit exceeded → `429`.
- Header **name and value** are policy-configurable (example convention: `X-Kuadrant-Limit-Exceeded: personal-daily`).
- Existing policies with no new fields keep today’s hard-deny behavior.
