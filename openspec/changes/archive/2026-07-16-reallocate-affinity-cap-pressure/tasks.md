## 1. Cap-safe affinity policy

- [x] 1.1 Add an internal affinity capability that is true only for bare process-level session-header `codex_session` mappings.
- [x] 1.2 Propagate the capability through HTTP/SSE, compact, direct WebSocket, and HTTP bridge selection while keeping other request families default-false.
- [x] 1.3 Intersect cap mobility with preferred-owner and request-stage state at the shared selection boundary.
- [x] 1.4 Make existing hard-sticky account-cap bypass conditional on the validated namespaced session-header capability while retaining legacy hard turn-state lookup and stable cap errors.
- [x] 1.5 Defer mapping settlement until each transport owns all required account-local and process-wide admission, then compare-and-set the observed owner and emit a key-safe diagnostic.
- [x] 1.6 Fail closed for unresolved `conversation` continuity unless a hard mapping resolves or exactly one account is eligible.
- [x] 1.7 Keep cap-selected HTTP bridge replacements unpublished until final admission and CAS, then promote them to the canonical lane.
- [x] 1.8 Make hard Codex mappings required-owner constraints and make compact timeout deletion owner-conditional.
- [x] 1.9 Serialize bridge marker/CAS/durable/canonical publication, protect canonical targets, and consume direct-WebSocket cap mobility after send.

## 2. Regression coverage

- [x] 2.1 Cover session-header versus turn-state affinity classification for Responses and compact requests.
- [x] 2.2 Cover capped-owner rebind, unsaturated-owner retention, default hard fail-closed behavior, and no-alternate mapping preservation in the load balancer.
- [x] 2.3 Cover shared service-boundary fail-closed guards for preferred owners and replay/reattach stages.
- [x] 2.4 Cover capability propagation on HTTP/SSE, compact, direct WebSocket, and HTTP bridge selection surfaces.
- [x] 2.5 Cover opaque owner-bearing payloads, source-key collisions, raw-key rejection, stale compare-and-set races, and second-stage cap failures across streaming, WebSocket, and HTTP bridge handoffs.
- [x] 2.6 Cover conversation ambiguity, compact namespaced timeout cleanup, deferred bridge publication, CAS conflict preservation, and canonical bridge promotion.
- [x] 2.7 Cover excluded hard owners, bridge publication ordering/rollback/marker loss, internal fork promotion, capacity preservation, and WebSocket replay pinning.

## 3. Verification

- [x] 3.1 Run focused routing, bridge, WebSocket, compact, and load-balancer suites plus lint, formatting, typing, architecture, and simplicity gates.
- [x] 3.2 Run strict change/all-spec validation and verify implementation coverage before archive.
