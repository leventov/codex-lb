## Context

`StickySessionKind.CODEX_SESSION` currently represents several different strengths of affinity: a bare process-level session header, a client turn-state anchor, previous-response ownership, file/account pins, and live or durable bridge ownership. The load balancer cannot distinguish them, so any existing mapping bypasses account-cap filtering and fails closed on its owner. That is correct for owner-bearing continuity but unnecessarily rejects self-contained session-header work.

Account-cap selection occurs before upstream dispatch and downstream output. HTTP/SSE, compact, direct WebSocket, and HTTP bridge paths already resolve hard owner signals into a preferred account or preserve them in request/bridge state.

## Goals / Non-Goals

**Goals:**

- Use another eligible account when a bare session-header mapping is saturated and the request has no correctness dependency on that account.
- Preserve the mapping when its owner remains below cap.
- Keep every owner-bearing or post-visible path fail-closed.
- Apply one policy consistently across Responses transports without configuration.

**Non-Goals:**

- Move client/proxy `previous_response_id` anchors across accounts.
- Move uploaded file references, client turn state, required preferred accounts, live/durable bridge ownership, or replay/reattach state.
- Replay a request after downstream-visible output.
- Change cap values, retry budgets, overload envelopes, or upstream 429 classification.

## Decisions

### Classify cap mobility by affinity source, not sticky kind

Add an internal `_AffinityPolicy` capability that is true only when `codex_session` affinity came from a bare process-level session header. Client turn state and synthesized turn state remain false. Compact uses the same source distinction. The flag is separate from `reallocate_sticky`, whose broader meaning also controls unavailable-account and budget-pressure mapping behavior.

Persist source-namespaced SHA-256 selection keys for session-header affinity while retaining raw lookup keys for hard turn-state rows. The asymmetric encoding keeps equal source values distinct without orphaning owners written by old replicas during a rolling upgrade. The load balancer accepts cap mobility only for the session-header namespace, so a raw/legacy key remains fail-closed even if a caller accidentally forwards the capability bit.

Inferring mobility from `CODEX_SESSION` inside the load balancer was rejected because that kind deliberately includes hard owner state.

### Revalidate mobility at the service selection boundary

The shared account-selection service intersects the affinity capability with request state. Cap reallocation is passed to the load balancer only when no preferred account was resolved and the stage is an initial or ordinary follow-up selection, never reattach/replay/recovery. Payload classification independently revokes mobility for nonblank `conversation`, nonblank `previous_response_id`, and file references, including owner-lookup misses. Because `conversation` has no dedicated owner resolver, its provenance also reaches shared selection: a persisted hard Codex-session mapping may prove the owner, and otherwise selection proceeds only for a one-account eligible pool. Ambiguous pools fail with `conversation_owner_unavailable` before account health can be affected. This fail-closed intersection protects file pins, previous-response ownership, durable lookups, and live-session recovery even if a request also carries a session header.

Relying on each transport caller alone was rejected because future call sites could forget an owner override. The service boundary is the common enforcement point; transport propagation remains explicit and testable.

### Make hard-sticky cap bypass opt-in

Add an internal load-balancer argument for cap-only sticky reallocation. The default remains false. When the capability, `CODEX_SESSION` kind, account lease, and namespaced session-header key all agree, the existing owner no longer bypasses account-cap filtering: if it is below cap it remains in the filtered pool and retains ownership; if saturated it leaves the pool and an eligible alternate may be selected. If no alternate survives, the existing cap error returns and the mapping remains unchanged.

Selection does not write the replacement mapping. It returns an immutable pending rebind containing the key, kind, observed owner, and selected account. Compact settles after its account response-create lease and process admission; SSE settles after stream selection plus account response-create and process admission; direct WebSocket and HTTP bridge carry the token across their transport handoff and settle after their response-create admission. The repository uses insert-if-absent or update-if-current compare-and-set semantics so late retries cannot overwrite a newer owner. Successful commits emit `internal_soft_affinity_reroute` without logging the raw sticky key.

HTTP bridge process creation remains distinct from ownership publication. A bridge session carrying a pending rebind, including one created under an internal reroute key, stays out of the canonical in-memory registry and durable ownership while the concrete request acquires response-create admission. After a successful sticky compare-and-set, the bridge claims durable ownership and atomically replaces the canonical local lane before the first upstream send. Admission or CAS failure closes the unpublished process and leaves the previous canonical lane in place.

Bridge publication validates its in-flight marker and serializes the sticky CAS, durable claim, and canonical registry swap under the bridge registry lock. This prevents waiter timeout eviction and out-of-order reroutes from creating partial or stale publication. Internal unanchored forks that settle affinity target the original canonical key, while capacity eviction protects that target; when no independent slot is available, bounded overload preserves the old lane. Durable publication failure triggers a compare-safe sticky rollback. Direct WebSocket capacity reroute is similarly one-shot: successful send changes the request to required-owner reattach state, and replay validates that any retained response-create lease belongs to the selected account.

### Enable the safe capability by default

No setting is added. Bare session affinity is a locality hint when no owner signal exists, and using otherwise-idle account capacity is the safe zero-config behavior. Hard-continuity sources remain fail-closed without operator policy.

## Risks / Trade-offs

- **A bare session loses cache locality under cap pressure** -> only saturated owners move, and the new mapping restores stickiness on the alternate account.
- **A new owner signal is added without clearing mobility** -> the shared service guard defaults unknown stages and every preferred account to fail-closed; production comments and tests document this invariant.
- **Concurrent self-contained requests rebind the same session** -> each request remains semantically self-contained, account leases still enforce caps atomically, and owner compare-and-set lets only a request based on the current mapping commit.
- **A later response-create cap rejects a selected stream owner** -> the transport never settles its pending token, so the old mapping remains unchanged.
- **An HTTP bridge process exists before its request is admitted** -> it remains behind an in-flight marker and cannot become live/durable canonical ownership until admission and CAS succeed.
- **A waiter times out while publication is committing** -> marker cleanup shares the publication lock and cannot revoke eligibility between persistent writes.
- **An internal fork or max-session boundary hides/removes the canonical lane** -> successful forks promote to the canonical key, while failed replacement creation preserves the existing target.
- **PR 1 changes the same hard-sticky branch** -> this PR is independently based on `main`; merge PR 1 first and rebase this focused change.

## Migration Plan

No schema or configuration migration is required. New source-namespaced session-header keys supersede legacy raw process-session rows lazily as requests arrive, while raw turn-state rows remain readable across mixed versions. Superseded process-session rows remain harmless and eligible for manual cleanup. Rollback restores fail-closed cap behavior without persisted-data conversion.

## Open Questions

None.
