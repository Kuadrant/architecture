# Standalone Extensions

- Feature Name: `standalone-extensions`
- Start Date: 2026-07-27
- RFC PR: [Kuadrant/architecture#197](https://github.com/Kuadrant/architecture/pull/197)
- Issue tracking: [CONNLINK-1064](https://redhat.atlassian.net/browse/CONNLINK-1064)

# Summary

[summary]: #summary

Enable Kuadrant extensions to run as standalone components that communicate with the operator over an authenticated gRPC channel, rather than as in-container subprocesses launched and managed by the operator. Standalone extensions connect to a single shared gRPC endpoint exposed by the operator, complete a handshake that authenticates them with a Kubernetes `ServiceAccount` token and authorizes the policy kind they may claim through Kubernetes RBAC, and then manage that policy kind through the extension SDK for the lifetime of an authenticated session. This RFC covers the transport, the handshake and authentication model, the RBAC-based ownership model, and the session and state-synchronization lifecycle that a standalone extension requires.

# Motivation

[motivation]: #motivation

The extensions mechanism is currently in dev preview. In that model every extension is an in-container subprocess: the operator ships the extension binary inside its own image and launches it as a child process, handing it a local socket to talk back over. That was the right shape to prove the concept, but it has a hard ceiling — you cannot run an extension the operator does not already ship.

The reason is lifecycle ownership. The operator's own deployment is a managed artifact, so there is no supported way to add your extension to it and have that change stick. In practice this means someone with an extension of their own has nowhere to run it against a released operator.

Moving extensions to a standalone model removes that ceiling. An extension becomes an ordinary workload the author owns and deploys themselves — something they can build, roll out, and roll back on their own terms — that connects to the operator over the network. This is what turns extensions from a Kuadrant-internal mechanism into something the community can actually try.

The audience for this work is community authors: people building extensions that each add a new policy kind against a stock operator, on their own release cadence, without contributing them back into the operator itself.

The expected outcome: an author can deploy an extension they wrote or obtained, authenticate it to the operator with a ServiceAccount an admin authorizes through RBAC, and have it manage its policy kind — without forking Kuadrant and without modifying the operator image. This standalone model is the tech-preview milestone for the extensions feature.

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

An **extension** is a controller that owns a single policy kind — and may read and interact with other resources in the cluster topology — and cooperates with the Kuadrant operator, exchanging topology queries and data bindings over gRPC using the extension SDK. In the standalone model an extension is a workload you deploy and own; it connects to the operator instead of being launched by it.

Three concepts are introduced or made user-visible:

- **Extension identity** — the Kubernetes `ServiceAccount` the extension workload runs as. The extension presents a token for that ServiceAccount at handshake; the operator verifies it and derives the identity (`system:serviceaccount:<namespace>:<name>`) from the token rather than trusting a name the extension asserts.
- **Ownership via RBAC** — which policy kind an extension may claim is a Kubernetes authorization decision. The extension's ServiceAccount is granted permission to `register` a specific policy kind through an ordinary `Role`/`ClusterRole`, and the operator checks that permission at handshake. There is no separate credential to generate or distribute.
- **Session** — once an extension has handshaked successfully, it holds an authenticated session for as long as it stays connected. All of its subsequent calls ride on that session.

The operator exposes a single gRPC endpoint (a Kubernetes `Service`) that every extension connects to. Built-in extensions and standalone extensions use the same endpoint and the same handshake; they differ only in where their token comes from.

## Deploying a standalone extension

From a cluster admin's point of view the flow is:

1. **Create a ServiceAccount and grant it the policy kind** it is allowed to manage. Ownership is expressed as an RBAC rule: the `register` verb on the virtual `policyregistrations` resource, scoped by `resourceNames` to the policy kind this extension owns.

   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: my-threat-policy
     namespace: my-namespace
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRole
   metadata:
     name: my-threat-policy-register
   rules:
     - apiGroups: ["extensions.kuadrant.io"]
       resources: ["policyregistrations"]
       resourceNames: ["ThreatPolicy"] # the policy kind this extension may claim
       verbs: ["register"]
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: my-threat-policy-register
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: my-threat-policy-register
   subjects:
     - kind: ServiceAccount
       name: my-threat-policy
       namespace: my-namespace
   ```

2. **Deploy the extension** as your own Deployment — in any namespace you like — running as that ServiceAccount, and point it at the operator's extension endpoint. It presents an audience-scoped ServiceAccount token, mounted as a projected volume that the kubelet auto-rotates:

   ```yaml
   spec:
     serviceAccountName: my-threat-policy
     containers:
       - name: extension
         env:
           - name: KUADRANT_EXTENSION_ADDRESS
             value: kuadrant-operator-extensions.kuadrant-system.svc:50052
           - name: KUADRANT_EXTENSION_TOKEN_FILE
             value: /var/run/secrets/kuadrant/token
         volumeMounts:
           - name: kuadrant-token
             mountPath: /var/run/secrets/kuadrant
             readOnly: true
     volumes:
       - name: kuadrant-token
         projected:
           sources:
             - serviceAccountToken:
                 path: token
                 audience: kuadrant-extensions
                 expirationSeconds: 3600
   ```

3. **The extension connects and handshakes.** On startup the SDK dials the endpoint, reads the current token from the projected file, and calls `Handshake` with the extension's version, the policy kind it intends to claim, and that token. The operator verifies the token (`TokenReview`) and checks that its ServiceAccount is authorized to register the claimed kind (`SubjectAccessReview`); on success it returns a session token the SDK attaches to every later call automatically. The extension then behaves exactly as it does today: it watches its CRD, resolves topology, and publishes data bindings.

Identity and ownership are both decided by Kubernetes, not asserted by the extension. The token proves _who_ the extension is (its ServiceAccount), and the RBAC rule decides _what_ it may claim — so an extension can only take a policy kind an admin has explicitly granted it. If the token is missing, expired, or issued for the wrong audience, or the ServiceAccount lacks the `register` grant for the kind it claims, the handshake is rejected with a clear reason and the extension does not start managing policies. Revoking an extension is an RBAC change: delete the binding and its next handshake fails authorization (existing sessions are unaffected until they re-handshake — see [Authentication and credentials](#authentication-and-credentials)). There is no shared secret to rotate; the kubelet rotates the token automatically and the SDK re-reads the file.

Because the extension is now a workload the author owns rather than a subprocess of the operator, its ServiceAccount also needs ordinary controller RBAC: permission to watch and reconcile its CRD as well as any other resources it may need to manage. This ships alongside the extension's own manifests — the extension no longer inherits the operator's permissions.

## What changes for extension authors

Almost nothing. The transport switch and the handshake are handled by the SDK. An author writes reconciler logic against the same `KuadrantCtx` API as before; the SDK reads the endpoint address and the token file path from the environment, reads the current ServiceAccount token, and performs the handshake before any reconcile runs. The difference is operational: the extension is now something you package and deploy — with a ServiceAccount and an RBAC grant — not something baked into the operator image.

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

## Transport

The operator runs a single shared gRPC server for all extensions on a configurable TCP port (`EXTENSIONS_SERVICE_PORT`, default `50052`), exposed in-cluster through a Kubernetes `Service`. This replaces the earlier model of one gRPC server per extension on a per-extension Unix domain socket. A single server hosting a single `ExtensionService` implementation is simpler than N servers that all forward to the same logic, and — critically — it is reachable over the network, which is what allows an extension to be a separate workload rather than a child process.

Both built-in and standalone extensions connect to this same endpoint and go through the same handshake. Built-in extensions dial `localhost:<port>`; standalone extensions dial the `Service` DNS name. The SDK takes the address from the `KUADRANT_EXTENSION_ADDRESS` environment variable, so the two cases differ only in configuration.

The server is configured with gRPC keepalive so that dead connections are detected at the transport level rather than lingering as stale sessions.

## Handshake protocol

The standalone model, end to end:

```text
  your namespace                                      kuadrant-system
  +-------------------------------+                   +-------------------------------------+
  | my-threat-policy              |    TCP :50052     | kuadrant-operator                   |
  | (your Deployment,             | ================> | extensions Service -> gRPC server   |
  |  runs as its ServiceAccount)  |                   |                                     |
  |                               |  1. Handshake     |  +-------------------------------+  |
  |  SDK:                         | ----------------> |  | auth interceptor              |  |
  |   dial ADDRESS  ------------- |     (SA token,    |  |  Handshake        -> verify   |  |
  |   read token file             |      policy_kind) |  |  every other RPC  -> require  |  |
  |   Handshake(version,          |                   |  |                      session  |  |
  |     policy_kind, token)       |  2. session_token |  +-------+---------------+-------+  |
  |   attach x-kuadrant-session   | <---------------- |          | 1a. TokenReview |          |
  |   reconcile + AddDataTo / ... |                   |          | 1b. SubjectAccessReview   |
  +---------------+---------------+  3. RPCs w/ token |          v (authn + authz)  |        |
                  | reads             ==============> |  +-------+---------------+-------+  |
                  v projected token                   |  | kube-apiserver                |  |
  +-------------------------------+                   |  |  TokenReview  -> identity     |  |
  | projected serviceAccountToken |                   |  |  SAR: register policy_kind?   |  |
  |  audience: kuadrant-extensions|                   |  +---------------+---------------+  |
  |  (kubelet auto-rotates)       |                   |                  | on success       |
  +-------------------------------+                   |                  v                  |
                                                      |  +---------------+---------------+  |
                                                      |  | session store (in-memory)     |  |
                                                      |  +-------------------------------+  |
                                                      +-------------------------------------+
```

An extension must call the `Handshake` RPC before any other RPC on the connection. The handshake carries a ServiceAccount token and the policy kind the extension intends to claim and, on success, returns a session token:

```protobuf
service ExtensionService {
  rpc Handshake(HandshakeRequest) returns (HandshakeResponse) {}
  // ... other RPCs
}

message ResourceID {
  string kind = 1;
  string namespace = 2;
  string name = 3;
}

message HandshakeRequest {
  string version = 1;                     // extension version, for compatibility and logging
  bytes  token = 2;                       // audience-scoped ServiceAccount token
  string policy_kind = 3;                 // policy kind this extension intends to claim
  repeated ResourceID owned_policies = 4; // CRs the extension currently owns; lets the operator prune stale state on connect (see State synchronization)
}

message HandshakeResponse {
  bool   accepted = 1;
  string session_token = 2; // attached as gRPC metadata on subsequent RPCs
  string reason = 3;        // human-readable rejection reason (empty on success)
}
```

The extension's _identity_ is not a field it sets — there is no self-asserted `name`. The operator derives it from the verified token, so the extension cannot impersonate another. On handshake the operator:

1. **Authenticates** the token with a `TokenReview` (`authentication.k8s.io`), passing the expected audience (`kuadrant-extensions`). A valid token yields the caller's username, `system:serviceaccount:<namespace>:<name>`, which becomes the extension identity.
2. **Authorizes** the claimed `policy_kind` with a `SubjectAccessReview` (`authorization.k8s.io`) for that user: verb `register`, resource `policyregistrations` in group `extensions.kuadrant.io`, with `Name` set to the claimed policy kind. This is what binds an identity to the kind it may own — the check passes only if an admin granted that ServiceAccount the matching RBAC rule.

`policyregistrations` is a _virtual_ resource: it has no CRD and nothing is ever persisted under it. It exists only as the RBAC coordinate the SAR is written against, so that "may this ServiceAccount register kind X" is expressible as an ordinary Kubernetes authorization rule (`resourceNames: ["X"]`). Custom verbs and `resourceNames` are both first-class in RBAC — Kubernetes itself uses this pattern for the `bind` and `escalate` verbs on `roles`/`clusterroles`.

On a successful handshake the operator generates a random session token and records it in memory against the extension's identity. The extension attaches this token as gRPC metadata (`x-kuadrant-session`) on every subsequent call. A server-side interceptor enforces the rule on every RPC: `Handshake` is always allowed; every other RPC requires a valid session token or is rejected with `Unauthenticated`. The interceptor makes the authenticated identity available to downstream handlers.

The session token, rather than the connection itself, is the unit of authentication. This is deliberate: gRPC transparently re-establishes the underlying TCP connection, and a token that survives reconnection lets a brief network blip resolve without forcing a full re-handshake. Concretely, a transport-level reconnection that still presents a valid token continues the _same_ session; a _new_ session is established only by a fresh `Handshake` — on first connect, or after the previous session was lost (operator restart, revocation, or a token that no longer validates). The re-sync described under [State synchronization](#state-synchronization) — re-reconciling every CR and pruning what is stale — is tied to that handshake, not to every TCP reconnect. The token map is in-memory only; if the operator restarts, every extension must re-handshake, which they will because their connections drop.

Only one active session is permitted per identity and per policy kind (first to register wins). RBAC decides _who may_ claim a kind; single-claimant decides _who currently holds_ it. The ownership check and the session creation are one atomic step, so two handshakes racing for the same identity or the same policy kind cannot both succeed — exactly one wins and the other is rejected with a deterministic reason (the identity already holds a session, or the policy kind is already owned by another identity). A fresh handshake from an identity that already holds an active session is rejected on the same grounds: the operator neither returns the existing session nor replaces it, so a live session can never be hijacked by a second handshake. A new session for that identity is established only once the previous one has been lost (connection drop, revocation, or operator restart — see above). Built-in extensions register before the endpoint accepts external connections, so they always claim their policy kinds first.

## Authentication and credentials

For standalone extensions authentication uses Kubernetes as the trust root — there is no pre-shared key: the handshake carries a token and the operator asks the API server who it belongs to and what it may do. The handshake shape is the same for both kinds of extension (present a credential, receive a session token); only how that credential is validated differs:

- **Standalone extensions** — the extension presents an audience-scoped ServiceAccount token, mounted as a projected volume (`serviceAccountToken` with `audience: kuadrant-extensions`). The kubelet issues and auto-rotates it before expiry, and the SDK re-reads the file each time it (re-)handshakes, so there is no `TokenRequest` API call in the extension's own code and nothing to rotate by hand. At handshake the operator runs a `TokenReview` with the expected audience to authenticate the token, then a `SubjectAccessReview` to authorize the claimed policy kind (see [Handshake protocol](#handshake-protocol)). Because the token is audience-bound, a token minted for some other service cannot be replayed against the extensions endpoint, and vice versa.
- **Built-in extensions** — subprocesses of the operator, with no distinct identity to verify, so the operator vouches for them directly rather than running `TokenReview`/`SubjectAccessReview` against itself: it generates a random ephemeral credential per built-in at startup, passes it over the subprocess environment, and issues a session token on handshake as usual. The credential is never persisted, lives for one operator process, and only crosses `localhost`. Ownership of built-in kinds is therefore not RBAC-gated but guaranteed by order — built-ins claim their kinds before the endpoint accepts external connections, so a standalone extension can never take one.

Ownership is not self-asserted. Authentication establishes _who_ the extension is (its ServiceAccount, from the verified token); authorization establishes _what_ it may claim (the `register` grant on the policy kind). An extension with a valid token but no matching RBAC grant is rejected — closing the gap where any credential holder could claim any unclaimed kind.

Revocation follows Kubernetes semantics. Deleting the RBAC binding causes the next handshake to fail authorization; deleting or disabling the ServiceAccount invalidates its tokens at `TokenReview` time. Neither tears down an _established_ session immediately — a session lives in the operator's in-memory store until the connection drops or the operator restarts, at which point the extension must re-handshake and the revocation takes effect. Tightening this into prompt session revocation (for example, re-checking authorization periodically or on token expiry) is left as future hardening.

The operator itself needs permission to run these reviews: `create` on `tokenreviews.authentication.k8s.io` and `subjectaccessreviews.authorization.k8s.io`. The built-in `system:auth-delegator` ClusterRole grants exactly this, so the operator's ServiceAccount is bound to it. This replaces the previous plan to grant the operator read access to a credential Secret — there is no such Secret any more.

## Session lifecycle

A session begins at a successful handshake and ends when the connection is lost or the operator restarts. Because a re-handshake re-runs authentication and authorization, an RBAC or ServiceAccount change made while a session is live takes effect the next time the extension handshakes. Extensions retry the connection and re-handshake on failure, with **randomized backoff** so that a large number of extensions reconnecting at once — for example after an operator restart — do not all handshake at the same instant. gRPC keepalive lets both sides notice a dead peer promptly.

Reconnecting is the easy part. The harder question is what state the two sides hold when they reconnect, and how it is made to agree again.

## State synchronization

An extension and the operator both hold state that must agree: the extension has published data bindings, registered upstreams, and pipeline actions; the operator holds those registrations in memory (`RegisteredDataStore`) and reflects them into managed resources (AuthConfig, the wasm configuration, Envoy clusters). A restart or a partition can break that agreement, so on every (re)connection the two must re-converge.

### Source of truth

The extension is a level-triggered controller — it derives every registration by reconciling its own CRs. The real source of truth is therefore the **cluster resources**, and both the operator's store and the extension's local view are caches of that. This is deliberately chosen over carrying serialized state in the handshake or persisting the operator's store to the cluster: it keeps the operator stateless, reuses the reconcile logic the extension already has, and collapses the restart and partition scenarios into a single mechanism — **rebuild from reconciles, prune what is stale** — that runs on every new session.

The work is distributed across extensions (each re-reconciles its own CRs) rather than concentrated in the operator, which — combined with reconnect jitter — keeps an operator restart from becoming a thundering herd. This is the same re-list-on-startup behaviour every Kubernetes controller already exhibits.

### Rebuilding the store

On a new session (any fresh `Handshake`, not a transparent transport reconnect), the SDK re-drives the extension's own reconcile loop over every CR it owns:

1. Enumerate every CR of the managed policy kind from the synced cache and enqueue each into the normal reconcile queue. The SDK cannot reconstruct registrations itself — only the author's reconcile code knows how a CR maps to `AddDataTo` and other registration calls.
2. Each CR runs through the author's own reconcile handler, which re-issues its registrations exactly as it would for a live change.

This is what rebuilds the operator's store after the operator has restarted: the extension process may still be running, so nothing in the cluster changed and the informer would not re-fire on its own — the SDK has to re-enqueue explicitly. Registrations are idempotent and keyed by policy `ResourceID`, so re-issuing unchanged state against a store that still holds it is a harmless overwrite.

### Pruning stale state

Re-reconciling can only add or refresh; it cannot remove state that should no longer exist. The case that matters for a disconnect is a whole policy deleted while the extension was away:

- **A whole policy deleted while the extension was disconnected (inter-policy).** The handshake carries `owned_policies` — the set of CR `ResourceID`s the extension currently owns. On handshake the operator prunes any registration it holds for that extension's policy kind whose `ResourceID` is _not_ in that set. This is deterministic and immediate: the operator has the authoritative set up front, so it needs no sync generation and no completion signal. The completeness that makes this safe comes from the extension side — the SDK builds `owned_policies` only after its informer cache has synced (`WaitForCacheSync`), so the operator always prunes against a full snapshot of what the extension owns, never a partial one. A reconcile that fails during the rebuild re-issues nothing — the policy keeps its previous, still-valid registrations and is retried — so partial failure never drops live data.

A second, narrower kind of staleness — **a single binding removed from a policy that still exists (intra-policy)** — sits alongside this core mechanism as a separate, still-open piece rather than part of it. The write path for mutator bindings and upstreams is additive (`AddDataTo` sets a binding by key and never removes one), so a binding dropped from a live CR lingers until the whole policy is cleared on delete. This is a pre-existing gap in the additive write model, independent of standalone extensions; a candidate fix targeted at tech preview is described under [Unresolved questions](#unresolved-questions).

### The four cases

- **Extension restarted, operator up** — the store still holds that extension's registrations, so there is no data-plane gap. The handshake's `owned_policies` prunes anything whose CR disappeared while it was down, and the re-reconcile refreshes the rest.
- **Operator restarted, extension up** — the store starts empty; the extension re-handshakes and re-reconciles to rebuild it. Pruning has nothing to remove. (See the restart gap below.)
- **Partition, both up** — a transient the gRPC channel rides out transparently loses nothing: pending reconciles, including deletes, resume when it clears. A partition long enough to break the session ends in a re-handshake, which prunes via `owned_policies` and re-reconciles from current cluster state — converging to the cluster: **cluster wins**.
- **Both restarted** — the ordinary first-boot path; both reconcile from cluster state and register through normal RPCs.

### The operator-restart gap

When the operator restarts, its store is empty until extensions reconnect and re-reconcile. During that window the operator regenerates managed resources from core policies (AuthPolicy, RateLimitPolicy, and the like) without the extensions' contributions, so an extension's additions are briefly absent from the data plane — a few seconds, until each extension re-syncs.

For the tech preview this gap is accepted. Core policies are unaffected; only extension-contributed data is momentarily missing, and only across an operator restart or upgrade. Closing it would mean the operator deferring regeneration of extension-touched resources (identified by a marker written when the resource is generated, so no fragile merge of live resources is required) until the owning extensions have re-applied them — which the operator can detect from the handshake `owned_policies` set, releasing the hold once every owned policy contributing to a resource has re-reconciled or a bounded timeout elapses. This is recorded as future hardening rather than built now. Note this gap is specific to operator restart — an extension restart never causes it, because the operator's store stays intact.

## Security considerations

- There is no long-lived shared secret to store or leak. Identity and ownership are decided by Kubernetes (`TokenReview` + `SubjectAccessReview`) against RBAC an admin controls, so authorizing an extension is auditable and revocable through ordinary cluster tooling.
- The ServiceAccount token is short-lived and audience-bound (`kuadrant-extensions`), so it cannot be replayed against other services and its exposure window is small. It is still a **bearer** token, though: anyone who captures it can replay it against the extensions endpoint until it expires. For tech preview the endpoint is in-cluster only and the mitigation for interception is in-cluster network controls; adding TLS to the channel remains an open question (see below) precisely because the token crosses it in cleartext.
- Built-in credentials are ephemeral and never persisted.
- Session tokens are random and long enough to make guessing impractical.
- The interceptor denies every non-handshake RPC without a valid session, so the handshake cannot be bypassed.

# Drawbacks

[drawbacks]: #drawbacks

- It introduces a network-reachable, authenticated surface where there was previously only a local socket, along with the RBAC an admin must set up (a ServiceAccount, a `register` grant per extension) to authorize each extension.
- A bearer token over an unencrypted channel is weaker than mutual TLS: it is short-lived and audience-bound, but replayable until expiry, and it is transmitted without transport encryption in the tech-preview scope.
- An operator restart briefly drops extension-contributed data from the data plane until each extension re-syncs (the accepted restart gap); core policies are unaffected, but this is a real, if short, window that only closes with the deferred hardening work.
- Ownership shifts to the user. An extension is now a workload someone has to deploy, authorize (a ServiceAccount and an RBAC grant), and operate, rather than something that simply ships with the operator.

# Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

**Why a single shared TCP endpoint rather than keeping per-extension Unix sockets?** A Unix socket is local by construction and cannot serve a workload running in another pod, which is the whole point of standalone extensions. A single TCP server also collapses N near-identical per-extension servers into one implementation and one place to apply authentication and keepalive. The same endpoint serves both built-in and standalone extensions, so there is one code path to reason about.

**Why ServiceAccount tokens and RBAC rather than a pre-shared key?** A pre-shared key means generating a secret, distributing it to both sides, and building rotation and revocation around it — and, on its own, it only proves _membership_, not _entitlement_: any holder of a valid key could claim any unclaimed policy kind. ServiceAccount tokens reuse machinery every cluster already has: the kubelet mints and rotates the token, `TokenReview` verifies it, and `SubjectAccessReview` against a virtual `policyregistrations` resource turns "which kind may this extension own" into an ordinary RBAC rule an admin writes and audits. That closes the entitlement gap and removes the shared secret entirely. The cost is that authorization is a Kubernetes concept an admin must configure (a ServiceAccount and a `register` grant), but that is standard controller-deployment work.

**Why a session token on top of the ServiceAccount token?** The session token exists so that authentication survives gRPC's transparent reconnection without re-running `TokenReview`/`SubjectAccessReview` on every blip: a transport reconnect that still carries a valid session token continues the same session, and only a genuine session break forces a fresh handshake. mTLS from the start would bring certificate issuance, rotation, and trust distribution, a larger investment than the tech-preview milestone requires — but transport security is genuinely on the table rather than deferred indefinitely (see [Unresolved questions](#unresolved-questions)).

**Why a handshake at all, rather than a token on each call?** The handshake gives a single point to authenticate, authorize the claimed kind, establish version, enforce single-claimant-per-policy-kind, and anchor state synchronization — it is where the extension declares the policies it owns (`owned_policies`) so the operator can prune stale state before the re-reconcile rebuilds the rest. Putting a token on every call would authenticate requests but would neither give us that reconnection anchor nor avoid a `TokenReview`/`SubjectAccessReview` per call.

**Impact of not doing this.** Extensions stay confined to what the operator ships. Community authors have no supported way to deploy their own against a released operator, and the feature cannot progress past dev preview.

# Unresolved questions

[unresolved-questions]: #unresolved-questions

- **Transport security (mTLS/TLS).** Currently scoped as in-cluster-only with the ServiceAccount token crossing the channel in cleartext. The token is short-lived and audience-bound, but replayable until expiry, so whether to add TLS or mutual TLS as part of this work — rather than after it — is an open decision. mTLS would let the client certificate carry identity as an alternative to (or defence in depth alongside) the token, which matters most for the out-of-cluster case where `TokenReview` may not be reachable.
- **Prompt session revocation.** An RBAC or ServiceAccount change only takes effect on the next handshake; an established session outlives it. Whether to re-authorize periodically, on session-token expiry, or on a watch of the relevant RBAC objects is an open hardening question.
- **Registration and discovery details.** How an operator learns which standalone extensions to expect may warrant more definition as real extensions are deployed. The RBAC an extension workload needs is now defined (a ServiceAccount plus a `register` grant on its policy kind), but conventions for packaging and naming those grants may still evolve.
- **NetworkPolicies.** The extension endpoint is reachable by any pod that can route to its Service, and for tech preview the token crosses that link in cleartext. The outcome of the `NetworkPolicy` work in RFC022 should be taken into account, restricting which workloads may reach the extensions port — and, symmetrically, which peers the operator accepts connections from.
- **Removing stale intra-policy bindings.** The additive write path (`AddDataTo` → `Set`, upstream registration → `SetUpstream`) never removes a binding dropped from a policy that still exists, so it lingers until the policy is cleared on delete. This is a pre-existing gap in the write model, independent of standalone extensions, but we would like to close it for tech preview. The leading candidate keeps writes firing immediately (preserving inline validation — `RegisterActionMethod` returns `ErrUpstreamUnreachable` at the call site): the SDK records the keys it writes for a policy during a reconcile and, on successful completion, sends one closing prune call listing the keys that should remain, and the operator drops the rest for that policy. It is fail-safe (a failed reconcile sends nothing, so nothing is pruned) and never opens a gap (the store holds a superset until the closing prune). The mechanism is not yet locked — the alternative, generalizing the atomic `PipelineCommit` model to the other write paths, changes those calls from fire-immediately to deferred-until-commit and moves error surfacing to commit time, which is why it is not the default choice.

# Future possibilities

[future-possibilities]: #future-possibilities

- **Out-of-cluster extensions.** Nothing in this design should prevent an extension running outside the cluster; it would require exposing the endpoint (Ingress/LoadBalancer) and stronger transport security. It would also need a different identity mechanism, since a workload outside the cluster has no projected ServiceAccount token — mTLS client certificates are the natural fit. This is explicitly left as future work.
- **Closing the operator-restart gap.** Rather than accepting the brief window where extension contributions are absent after an operator restart, the operator could defer regenerating extension-touched managed resources until the owning extensions have re-applied their contributions — identifying those resources by a marker written when the resource is generated, so the existing resource keeps serving without any fragile merge of live state. Completion is read from the handshake `owned_policies` set: the hold releases once every owned policy contributing to a resource has re-reconciled, or a bounded timeout elapses for a genuinely absent extension.
- **Finer-grained authorization scoping (GA work).** Binding an identity to the policy kind it may claim is handled in this design by the `register` SubjectAccessReview. That is deliberately coarse: it decides _whether_ an extension may participate for a given kind, not _what_ it may then do. A fuller model — expected as GA work rather than part of this tech preview — would constrain an extension's operations once connected, for example:
  - which SDK methods it may call (e.g. permit `AddDataTo` but not upstream registration);
  - which resources it may read or affect, and which specific resources it may modify, rather than trusting any connected extension with the full surface.

  An extension is a privileged participant in the operator's reconciliation, so scoping this matters, but it is a substantial design in its own right. The virtual-resource + SAR pattern established here extends naturally to it (additional verbs, resources, or resourceNames checked at the relevant call sites rather than only at handshake), so this design is a foundation for that work rather than something it would need to replace.

- **Multi-instance / HA extensions.** Allow more than one instance of the same extension for availability, with coordinated registration.
- **HMAC or challenge-response credentials.** Replace opaque tokens with a scheme that resists replay.
