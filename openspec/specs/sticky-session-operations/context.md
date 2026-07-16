# Sticky Session Operations Context

## Purpose and Scope

This capability covers operational control of sticky-session mappings after prompt-cache affinity was made bounded. It distinguishes durable backend/session routing from bounded prompt-cache affinity and defines the admin controls around those mappings.

See `openspec/specs/sticky-session-operations/spec.md` for normative requirements.

## Decisions

- Sticky-session rows store an explicit `kind` so prompt-cache cleanup can target only bounded mappings.
- Dashboard prompt-cache TTL is persisted in settings so operators can adjust it without restart.
- Background cleanup removes stale prompt-cache rows proactively, while manual delete and purge endpoints provide operator override.
- Codex process-session and turn-state values share a durable kind, but new process-session mappings use source-namespaced hashed keys while hard turn-state mappings retain their legacy raw lookup key. This isolates the mobile source without orphaning hard owners during rolling upgrades.
- Bare process-session affinity may move away from a capped account only for self-contained work. Selection returns a pending compare-and-set operation; the transport commits it after all of its account-local and process-wide admission stages succeed.

## Constraints

- Historical sticky-session rows created before the `kind` column are backfilled conservatively to a durable kind to avoid accidental purge.
- Durable `codex_session` and `sticky_thread` mappings are never deleted by automatic cleanup.
- Nonblank `conversation`, nonblank `previous_response_id`, file references, preferred owners, and replay/reattach state revoke bare-session mobility even when an owner lookup misses.

## Failure Modes

- Cleanup failures are logged and retried on the next interval; request handling continues.
- Manual purge and delete operations are dashboard-auth protected and return normal dashboard API errors on invalid input or missing keys.
- A response-create cap can reject work after a stream slot was selected. In that case the pending rebind is discarded and the old owner remains intact; a stale request also cannot overwrite a newer owner because settlement compares the owner observed during selection.

## Example

Session `S` maps to account A, whose stream cap is full. A self-contained request may select account B and reserve B's stream slot. If B's response-create slot is then unavailable, `S` stays mapped to A. If all required admission succeeds, the proxy atomically changes `S` from A to B and logs `internal_soft_affinity_reroute` without the raw session key.
