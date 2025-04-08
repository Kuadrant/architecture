# RFC Template

- Feature Name: `ai_policies`
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [Kuadrant/architecture#0013](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/issues/0000)

# Summary
[summary]: #summary

AI workloads are becoming more and more prominent of late, and there is considerable traction in a number of related Gateway API projects to provide policies and tooling for managing these workloads, from the perspectives of both platform engineering and the app developer.

This RFC proposes three new Kuadrant APIs in the general AI Gateway realm:

- `LLMTokenRateLimitPolicy`
- `LLMPromptRiskCheckPolicy`
- `LLMResponseRiskCheckPolicy`

# Motivation
[motivation]: #motivation

Why are we doing this? What use cases does it support? What is the expected outcome?

AI workloads are becoming extremely popular. Management of ingress with Gateway API and policies into running model servers is something our policies can augment and enhance.
The aim is to augement, not replace, the [Gateway API Inference Extensions](https://gateway-api-inference-extension.sigs.k8s.io/) project.

Using the [Inference Platform Admin](https://github.com/kubernetes-sigs/gateway-api-inference-extension/tree/main/docs/proposals/002-api-proposal#inference-platform-admin) and [Inference Workload Owner](https://github.com/kubernetes-sigs/gateway-api-inference-extension/tree/main/docs/proposals/002-api-proposal#inference-workload-owner) personas, some high-level use-cases we hope to enable are:

- As an Inference Platform Admin, I want to protect my LLM infrastrcture from being overwhelmed by limiting per user token usage across all models.
- As an Inference Workload Owner, I want to apply token usage limits to specific models so that costs are controlled based on the heaviest GPU using models.
- As an Inference Workload Owner, I want to apply different token usage limits to different user groups based on business requirements.
- As an Inference Workload Owner, I want to guard my running model servers from risky prompts (input) with a model such as [Granite Guardian](https://huggingface.co/ibm-granite/granite-guardian-3.1-2b), and reject them before they are routed to (expensive) running models
- As an Inference Workload Owner, I want guard against risky chat completion responses (output) with a model such as [Granite Guardian](https://huggingface.co/ibm-granite/granite-guardian-3.1-2b)
- As an Inference Workload Owner, I want to apply guards against different risky content categories to different user groups based on business requirements.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Concepts Introduced

- Token-based rate limiting: Controls usage by number of tokens, not just requests. Helps manage LLM costs.
- Prompt guarding: Filters risky or unwanted prompts before they hit model servers.
- Completion guarding: Reviews and filters model outputs before they're returned to users.

## Example Use Case

Say you're serving a chat LLM behind a Gateway. You want:

- Free-tier users limited to 20k tokens/day.
- Guardrails to reject prompts asking for harmful, violent or sexual content.
- Output filtering to block generally harmful completions.

You’d define the following policies:

```yaml
apiVersion: kuadrant.io/v1alpha1
kind: LLMTokenRateLimitPolicy
metadata:
  name: token-limit-free
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: my-llm-gateway
  limit:
    rate:
      limit: 4
      window: 10s
    predicate: 'request.auth.claims["kuadrant.io/groups"].split(",").exists(g, g == "free")' 
    counter: auth.identity.userid
```

```yaml
apiVersion: kuadrant.io/v1alpha1
kind: LLMPromptRiskCheckPolicy
metadata:
  name: prompt-check
spec:
  model:
    url: http://huggingface-granite-guardian-default.example.com/v1
    apiKey:
      secretRef:
        name: granite-api-key
        key: token
  categories:
    - harm
    - violence
    - sexual_content
  response:
    unauthorized:
      headers:
        "content-type":
          value: application/json
      body:
        value: |
          {
            "error": "Unauthorized",
            "message": "Request prompt blocked by content policy."
          }
```

```yaml
apiVersion: kuadrant.io/v1alpha1
kind: LLMResponseRiskCheckPolicy
metadata:
  name: completion-check
spec:
  model:
    url: http://huggingface-granite-guardian-default.example.com/v1
    apiKey:
      secretRef:
        name: granite-api-key
        key: token
  categories:
    - harm
  response:
    forbidden:
      headers:
        "content-type":
          value: application/json
      body:
        value: |
          {
            "error": "Forbidden",
            "message": "Response blocked by content policy."
          }
```

## User Impact

- Existing users: These policies extend existing workflows. Policy attachment is the same; what's different is the semantics (tokens vs requests, prompt filtering, etc).
- New users: Can adopt Kuadrant to secure and manage LLM workloads from the start.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- How error would be reported to the users.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- Does another project have a similar feature?
- What can be learned from it? What's good? What's less optimal?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other tentatives - successful or not, provide readers of your RFC with a fuller picture.

Note that while precedent set by other projects is some motivation, it does not on its own motivate an RFC.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would be and how it would affect the platform and project as a whole. Try to use this section as a tool to further consider all possible interactions with the project and its components in your proposal. Also consider how this all fits into the roadmap for the project and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but otherwise related.

Note that having something written down in the future-possibilities section is not a reason to accept the current or a future RFC; such notes should be in the section on motivation or rationale in this or subsequent RFCs. The section merely provides additional information.