## Why

A `codex_session` mapping created from a process-level session header is currently treated as hard ownership when its account reaches a local response-create or stream cap. Even when the request is a self-contained, pre-visible turn with another eligible account available, the proxy returns local overload and leaves usable pool capacity idle.

## What Changes

- Distinguish bare session-header affinity from owner-bearing Codex continuity.
- Allow only self-contained, pre-visible session-header affinity to rebind to an eligible account below the local cap.
- Preserve fail-closed routing for client- or proxy-supplied `previous_response_id`, file/account pins, client turn state, live or durable bridge ownership, reattach/replay state, and any request with downstream-visible output.
- Preserve the existing stable local cap error when no safe alternate account exists.
- Keep the behavior zero-config and enabled by default because it only relaxes a locality hint, not a correctness dependency.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `proxy-admission-control`: Define bare `codex_session` affinity as soft for pre-visible account-cap selection while retaining fail-closed hard-continuity admission.
- `sticky-session-operations`: Distinguish session-header locality from owner-bearing turn, response, file, bridge, and replay continuity when rebinding persistent affinity.

## Impact

- Affinity classification in `app/modules/proxy/affinity.py` and compact Responses classification.
- Account selection propagation through HTTP/SSE, compact, direct WebSocket, and HTTP bridge Responses paths.
- Hard-sticky account-cap filtering in `app/modules/proxy/load_balancer.py`.
- Focused policy, service-boundary, load-balancer, and externally visible routing regressions.
- No setting, environment variable, migration, response-schema, or dashboard rendering change.
