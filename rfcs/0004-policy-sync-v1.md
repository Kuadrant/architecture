# RFC - Policy Sync

- Feature Name: `policy_sync_v1`
- Start Date: 2023-10-10
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)
- Issue tracking: [https://github.com/Kuadrant/architecture/issues/26](https://github.com/Kuadrant/architecture/issues/26)

# Summary
[summary]: #summary

When a gateway in the hub is targeted by a policy in the hub, enable the gateway controller to be able to sync both the gateway and policy together to spoke clusters.

# Motivation
[motivation]: #motivation

Currently, any policies targeting gateways in the spokes need to be defined in the spokes, and it can be cumbersome, time-consuming and error prone to require these to be duplicated across multiple spoke clusters.

Gateway-admin has a set of homogeneous clusters and needs to apply per cluster rate limits across the entire set.

Platform-admin with a set of clusters with rate limits applied needs to change rate limit for one particular cluster. 

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Make explicit that OCM policy is different to a gateway API policy, and that this work is not related at all to the OCM Policy framework.

The policy sync system will allow a gateway-admin to create or modify a gateway class at the hub level and specify a series of GVKs for that gatewayClass (for example the AuthPolicy GVK).

When a gateway is created in the hub that uses this gatewayClass, any AuthPolicies that target that gateway will be watched by the MGC.

When these gateways are placed on a spoke, any AuthPolicies targeting that gateway will also be placed on that same spoke.

When the AuthPolicies are placed on the relevant spokes, they will be manipulated to target the new gateway in the spoke.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Process overview
- gateway controller updated to monitor params in gateway class
- Set up dynamic watches against each listed GVK
  - error when GVK not present - report in gatewayclass
  - confirm GVK is a policy - report in gatewayclass if not, don't create watch
- find all matching GVKs that target reconciling gateway
- copy GVK CR into manifestwork for same clusters as gateway
  - mutate GVK CR to target the spoke instance of the gateway
  - mutate GVK CR to annotate that it came from the hub
  - add common policy status fields to manifestwork
    - handle error if status fields are not present?
- read status from manifestwork and update status in hub GVK CR
- read errors on gateway if conflicting policies are present in spoke?

### Policy Hierarchy Details

- kuadrant operator to encode policy heirarchy
    - hub policy overidden by spoke policy overridden by route policy.
    - annotate gateway when a hub policy is overridden in this manner.

# Drawbacks
[drawbacks]: #drawbacks

cluster-admins can already create policies in spoke clusters that affect spoke level gateways, without this solution.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Consequences of not implementing
Gateway-admins will have no centralized system for handling spoke-level policies targeting a gateway created there from the hub.

#### We will not be using the policy framework to complete this objective:
The policy framework is a system designed to make assertions about the state of a spoke, and potentially take actions based on that state, as such it is not a suitable replacement for manifestworks in the case of syncing resources to a spoke.

### We could eventually migrate from manifestworks to manifestworkreplicasets
Manifestworkreplicasets maybe a future improvement that we could change the MGC to use but not as part of this RFC.

# Prior art
[prior-art]: #prior-art

No applicable prior art.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

# Future possibilities
[future-possibilities]: #future-possibilities

If the policy-framework is updated to enable syncing of resources status back to the hub, that could be a good time to refactor the MGC to use the policy framework in place of the current approach of creating manifestworks directly.

This system could mutate over time to dynamically sync more CRDs than policies to spoke clusters.