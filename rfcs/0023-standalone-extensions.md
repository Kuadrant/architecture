# Standalone Extensions

- Feature Name: `standalone-extensions`
- Start Date: 2026-07-27
- RFC PR: [Kuadrant/architecture#197](https://github.com/Kuadrant/architecture/pull/197)
- Issue tracking: [CONNLINK-1064](https://redhat.atlassian.net/browse/CONNLINK-1064)

# Summary

[summary]: #summary

Enable Kuadrant extensions to run as standalone components that communicate with the operator over an authenticated gRPC channel, rather than as in-container subprocesses launched and managed by the operator. Standalone extensions connect to a single shared gRPC endpoint exposed by the operator, complete a handshake that establishes their identity and authenticates them with a pre-shared key, and then manage their own policy kind through the extension SDK for the lifetime of an authenticated session. This RFC covers the transport, the handshake and authentication model, credential management, and the session and state-synchronization lifecycle that a standalone extension requires.

# Motivation

[motivation]: #motivation

The extensions mechanism is currently in dev preview. In that model every extension is an in-container subprocess: the operator ships the extension binary inside its own image and launches it as a child process, handing it a local socket to talk back over. That was the right shape to prove the concept, but it has a hard ceiling — you cannot run an extension the operator does not already ship.

The reason is lifecycle ownership. The operator's own deployment is a managed artifact, so there is no supported way to add your extension to it and have that change stick. In practice this means someone with an extension of their own has nowhere to run it against a released operator.

Moving extensions to a standalone model removes that ceiling. An extension becomes an ordinary workload the author owns and deploys themselves — something they can build, roll out, and roll back on their own terms — that connects to the operator over the network. This is what turns extensions from a Kuadrant-internal mechanism into something the community can actually try.

The audience for this work is community authors: people building extensions that each add a new policy kind against a stock operator, on their own release cadence, without contributing them back into the operator itself.

The expected outcome: an author can deploy an extensioqn they wrote or obtained, authenticate it to the operator with a credential they control, and have it manage its policy kind — without forking Kuadrant and without modifying the operator image. This standalone model is the tech-preview milestone for the extensions feature.

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

An **extension** is a controller that owns a single policy kind — and may read and interact with other resources in the cluster topology — and cooperates with the Kuadrant operator, exchanging topology queries and data bindings over gRPC using the extension SDK. In the standalone model an extension is a workload you deploy and own; it connects to the operator instead of being launched by it.

Three concepts are introduced or made user-visible:

- **Extension identity** — a name the extension announces for itself (for example `my-threat-policy`) and the policy kind it manages. The operator uses the identity to decide whether the extension is allowed to connect and which policy kind it may claim.
- **Pre-shared key** — an opaque token the extension presents to prove it is allowed to connect. For extensions you deploy, you generate the token and give it to both the operator (via a Secret) and the extension.
- **Session** — once an extension has handshaked successfully, it holds an authenticated session for as long as it stays connected. All of its subsequent calls ride on that session.

The operator exposes a single gRPC endpoint (a Kubernetes `Service`) that every extension connects to. Built-in extensions and standalone extensions use the same endpoint and the same handshake; they differ only in where their credential comes from.

## Deploying a standalone extension

From a cluster admin's point of view the flow is:

1. **Generate a token** and store it in the operator's extension-auth Secret, keyed by the extension's name:

   ```console
   $ kubectl create secret generic kuadrant-extension-auth \
       --namespace kuadrant-system \
       --from-literal=my-threat-policy=$(openssl rand -hex 32)
   ```

2. **Deploy the extension** as your own Deployment — in any namespace you like — and point it at the operator's extension endpoint. It reads its credential from a Secret in its own namespace:

   ```yaml
   env:
     - name: KUADRANT_EXTENSION_NAME
       value: my-threat-policy
     - name: KUADRANT_EXTENSION_ADDRESS
       value: kuadrant-operator-extensions.kuadrant-system.svc:50052
     - name: KUADRANT_EXTENSION_CREDENTIAL
       valueFrom:
         secretKeyRef:
           name: my-threat-policy-credential
           key: token
   ```

3. **The extension connects and handshakes.** On startup the SDK dials the endpoint, calls `Handshake` with the extension's name, version, policy kind, and credential, and — on success — receives a session token it attaches to every later call automatically. The extension then behaves exactly as it does today: it watches its CRD, resolves topology, and publishes data bindings.

The token the extension presents must be byte-for-byte identical to the one stored under its name in the operator's `kuadrant-extension-auth` Secret — that shared value is what proves the extension is allowed to connect. The extension can live in any namespace and read the token from a Secret of its own (as above); only the operator's Secret is consulted at handshake time, so the two values simply have to match. If the credential is wrong, missing, too short, or the name is not present in the operator's Secret, the handshake is rejected with a clear reason and the extension does not start managing policies. If you remove the extension's key from the Secret, its session is revoked and its in-flight calls begin failing authentication. Rotating the credential later means updating both Secrets to the new value and rolling out the extension pod so it re-reads its env var; changing the operator-side value revokes the current session, so the extension re-handshakes with the new credential (see [Authentication and credentials](#authentication-and-credentials)).

Because the extension is now a workload the author owns rather than a subprocess of the operator, it runs under its own `ServiceAccount` and needs its own RBAC: permission to watch and reconcile its CRD as well as any other resources it may need to manage. This is standard controller RBAC and ships alongside the extension's own manifests — the extension no longer inherits the operator's permissions.

## What changes for extension authors

Almost nothing. The transport switch and the handshake are handled by the SDK. An author writes reconciler logic against the same `KuadrantCtx` API as before; the SDK reads the endpoint address, name, and credential from the environment and performs the handshake before any reconcile runs. The difference is operational: the extension is now something you package and deploy, not something baked into the operator image.

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
  | (your Deployment)             | ================> | extensions Service -> gRPC server   |
  |                               |                   |                                     |
  |  SDK:                         |  1. Handshake     |  +-------------------------------+  |
  |   dial ADDRESS  ------------- | ----------------> |  | auth interceptor              |  |
  |   Handshake(name, version,    |     (credential)  |  |  Handshake        -> allow    |  |
  |     policy_kind, credential)  |                   |  |  every other RPC  -> require  |  |
  |                               |  2. session_token |  |                      session  |  |
  |   attach x-kuadrant-session   | <---------------- |  +---------------+---------------+  |
  |   reconcile + AddDataTo / ... |                   |                  | validates        |
  +---------------+---------------+  3. RPCs w/ token |                  v                  |
                  | reads             ==============> |  +---------------+---------------+  |
                  v credential                        |  | session store (in-memory)     |  |
  +-------------------------------+                   |  +---------------+---------------+  |
  | Secret (yours): token: <tok>  |                   |                  ^ key == name       |
  +-------------------------------+                   |  +---------------+---------------+  |
      same value  ..............................>     |  | Secret kuadrant-extension-auth|  |
                     must match                       |  |  data: my-threat-policy: <tok>|  |
                                                      |  +-------------------------------+  |
                                                      |     ^ operator watches / validates  |
                                                      +-------------------------------------+
```

An extension must call the `Handshake` RPC before any other RPC on the connection. The handshake carries identity and a credential and, on success, returns a session token:

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
  string name = 1;                        // extension identity, e.g. "my-threat-policy"
  string version = 2;                     // extension version, for compatibility and logging
  bytes  credential = 3;                  // pre-shared key
  string policy_kind = 4;                 // policy kind this extension manages
  repeated ResourceID owned_policies = 5; // CRs the extension currently owns; lets the operator prune stale state on connect (see State synchronization)
}

message HandshakeResponse {
  bool   accepted = 1;
  string session_token = 2; // attached as gRPC metadata on subsequent RPCs
  string reason = 3;        // human-readable rejection reason (empty on success)
}
```

On a successful handshake the operator generates a random session token and records it in memory against the extension's identity. The extension attaches this token as gRPC metadata (`x-kuadrant-session`) on every subsequent call. A server-side interceptor enforces the rule on every RPC: `Handshake` is always allowed; every other RPC requires a valid session token or is rejected with `Unauthenticated`. The interceptor makes the authenticated identity available to downstream handlers.

The session token, rather than the connection itself, is the unit of authentication. This is deliberate: gRPC transparently re-establishes the underlying TCP connection, and a token that survives reconnection lets a brief network blip resolve without forcing a full re-handshake. Concretely, a transport-level reconnection that still presents a valid token continues the _same_ session; a _new_ session is established only by a fresh `Handshake` — on first connect, or after the previous session was lost (operator restart, revocation, or a token that no longer validates). The re-sync described under [State synchronization](#state-synchronization) — re-reconciling every CR and pruning what is stale — is tied to that handshake, not to every TCP reconnect. The token map is in-memory only; if the operator restarts, every extension must re-handshake, which they will because their connections drop.

Only one active session is permitted per extension name and per policy kind (first to register wins). Built-in extensions register before the endpoint accepts external connections, so they always claim their policy kinds first.

## Authentication and credentials

Authentication is a pre-shared key presented in the handshake. There are two credential sources, validated by the same path:

- **Standalone extensions** — the operator reads credentials from a Kubernetes Secret (default name `kuadrant-extension-auth`, configurable via `EXTENSION_AUTH_SECRET`) in its own namespace. Each data key is an extension name; each value is that extension's token. The operator watches the Secret, so additions, rotations, and removals take effect without a restart. Removing a key revokes any active session for that name; changing a key's value likewise revokes the session established with the old value, forcing a re-handshake with the new credential. Rotation on the extension side is a pod concern: because the extension typically receives its credential as a `secretKeyRef` environment variable, it only presents a rotated value after its pod is rolled out. Mounting the credential as a reloadable projected volume with in-process reload would avoid the restart, but is not required for tech preview.
- **Built-in extensions** — the operator generates a random ephemeral token per extension at startup and passes it to the subprocess via environment variable. These tokens are never written to disk or to a Secret and are regenerated on every operator restart. They are registered in the same in-memory credential store as Secret-based tokens, so validation is identical.

Credentials must be at least 32 bytes; shorter values are rejected at handshake time. No format is imposed (hex, base64, UUID, and raw bytes are all acceptable); documentation recommends `openssl rand -hex 32`.

## Session lifecycle

A session begins at a successful handshake and ends when the connection is lost, the operator revokes it (for example on credential removal), or the operator restarts. Extensions retry the connection and re-handshake on failure, with **randomized backoff** so that a large number of extensions reconnecting at once — for example after an operator restart — do not all handshake at the same instant. gRPC keepalive lets both sides notice a dead peer promptly.

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

- Pre-shared keys live in a Kubernetes Secret governed by RBAC; only the operator's ServiceAccount and cluster admins should be able to read it.
- For tech preview the endpoint is reachable only within the cluster and the mitigation for interception is in-cluster network controls. The credential is sent in the clear in the handshake request itself (the `credential` field of `HandshakeRequest`), which is the primary reason transport security is an open question (see below) rather than settled future work.
- Built-in credentials are ephemeral and never persisted.
- Session tokens are random and long enough to make guessing impractical; the minimum credential length prevents weak user-chosen keys.
- The interceptor denies every non-handshake RPC without a valid session, so the handshake cannot be bypassed.

# Drawbacks

[drawbacks]: #drawbacks

- It introduces a network-reachable, authenticated surface where there was previously only a local socket, along with the credential management (a Secret, rotation, revocation) that comes with it.
- A pre-shared key is weaker than mutual TLS, and the credential is transmitted without transport encryption in the tech-preview scope.
- An operator restart briefly drops extension-contributed data from the data plane until each extension re-syncs (the accepted restart gap); core policies are unaffected, but this is a real, if short, window that only closes with the deferred hardening work.
- Ownership shifts to the user. An extension is now a workload someone has to deploy, credential, and operate, rather than something that simply ships with the operator.

# Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

**Why a single shared TCP endpoint rather than keeping per-extension Unix sockets?** A Unix socket is local by construction and cannot serve a workload running in another pod, which is the whole point of standalone extensions. A single TCP server also collapses N near-identical per-extension servers into one implementation and one place to apply authentication and keepalive. The same endpoint serves both built-in and standalone extensions, so there is one code path to reason about.

**Why a pre-shared key with a session token rather than mTLS from the start?** A pre-shared key in a Secret is simple to operate for a tech preview: generate a token, put it in a Secret, hand it to the extension. mTLS brings certificate issuance, rotation, and trust distribution, which is a larger investment than the tech-preview milestone requires. The session token exists so that authentication survives gRPC's transparent reconnection without a re-handshake on every blip. That said, transport security is genuinely on the table for this work rather than deferred indefinitely — see [Unresolved questions](#unresolved-questions).

**Why a handshake at all, rather than just a credential on each call?** The handshake gives a single point to establish identity and version, to enforce single-claimant-per-policy-kind, and to anchor state synchronization — it is where the extension declares the policies it owns (`owned_policies`) so the operator can prune stale state before the re-reconcile rebuilds the rest. Putting a credential on every call would authenticate requests but would not give us that reconnection anchor.

**Impact of not doing this.** Extensions stay confined to what the operator ships. Community authors have no supported way to deploy their own against a released operator, and the feature cannot progress past dev preview.

# Unresolved questions

[unresolved-questions]: #unresolved-questions

- **Transport security (mTLS/TLS).** Currently scoped as in-cluster-only with the credential in cleartext. Whether to add TLS or mutual TLS as part of this work — rather than after it — is an open decision. mTLS would also let the client certificate carry identity, at which point the pre-shared key could become optional.
- **Registration and discovery details.** How an operator learns which standalone extensions to expect, and the RBAC an extension workload needs, may warrant more definition as real extensions are deployed.
- **NetworkPolicies.** The extension endpoint is reachable by any pod that can route to its Service, and for tech preview the credential crosses that link in cleartext. The outcome of the `NetworkPolicy` work in RFC022 should be taken into account, restricting which workloads may reach the extensions port — and, symmetrically, which peers the operator accepts connections from.
- **Removing stale intra-policy bindings.** The additive write path (`AddDataTo` → `Set`, upstream registration → `SetUpstream`) never removes a binding dropped from a policy that still exists, so it lingers until the policy is cleared on delete. This is a pre-existing gap in the write model, independent of standalone extensions, but we would like to close it for tech preview. The leading candidate keeps writes firing immediately (preserving inline validation — `RegisterActionMethod` returns `ErrUpstreamUnreachable` at the call site): the SDK records the keys it writes for a policy during a reconcile and, on successful completion, sends one closing prune call listing the keys that should remain, and the operator drops the rest for that policy. It is fail-safe (a failed reconcile sends nothing, so nothing is pruned) and never opens a gap (the store holds a superset until the closing prune). The mechanism is not yet locked — the alternative, generalizing the atomic `PipelineCommit` model to the other write paths, changes those calls from fire-immediately to deferred-until-commit and moves error surfacing to commit time, which is why it is not the default choice.

# Future possibilities

[future-possibilities]: #future-possibilities

- **Out-of-cluster extensions.** Nothing in this design should prevent an extension running outside the cluster; it would require exposing the endpoint (Ingress/LoadBalancer) and stronger transport security. This is explicitly left as future work.
- **Closing the operator-restart gap.** Rather than accepting the brief window where extension contributions are absent after an operator restart, the operator could defer regenerating extension-touched managed resources until the owning extensions have re-applied their contributions — identifying those resources by a marker written when the resource is generated, so the existing resource keeps serving without any fragile merge of live state. Completion is read from the handshake `owned_policies` set: the hold releases once every owned policy contributing to a resource has re-reconciled, or a bounded timeout elapses for a genuinely absent extension.
- **Authorization scoping.** Today a credential authorizes an extension _by name_, and the handshake only checks that the `policy_kind` it claims is not already owned — so any valid credential holder can claim any unclaimed kind; nothing binds a credential to a _specific_ authorized kind. A fuller model would constrain what an extension identity is permitted to do — not only which policy kind it may claim, but which operations it may perform and which resources it may read or affect. An extension is a privileged participant in the operator's reconciliation, so scoping what it can register, mutate, and observe is a natural next step. Binding credential to policy kind could be brought into tech preview if warranted.
- **Multi-instance / HA extensions.** Allow more than one instance of the same extension for availability, with coordinated registration.
- **HMAC or challenge-response credentials.** Replace opaque tokens with a scheme that resists replay.
