## MODIFIED Requirements

### Requirement: Account-local Responses work is capped before upstream creation

For `/v1/responses`, `/backend-api/codex/responses`, and compact Responses traffic, the proxy MUST enforce account-local response-create and streaming concurrency limits in addition to process-wide admission limits, and the configured limits MUST be cluster-wide per-account targets enforced across all replicas rather than per-replica allowances. Because per-account caps are partitioned per replica via the bridge ring and cannot be safely partitioned across intra-pod worker processes, each instance MUST run a single worker process; horizontal scaling is achieved by adding replicas. The default account response-create cap MUST be 4 and the default account stream cap MUST be 8 unless operators configure a different value. When an account is at either cap, new soft-affinity work MUST prefer another eligible account before returning local overload. A `codex_session` mapping derived only from a process-level session header MUST be treated as soft for this cap decision when the request is pre-visible and self-contained. Requests carrying nonblank `conversation`, nonblank `previous_response_id`, or account-scoped file references MUST NOT qualify as self-contained even when owner lookup does not resolve an account. When a request carries `conversation` and no hard Codex-session mapping or other authoritative owner resolves, selection MUST proceed only if the eligible account pool contains exactly one account; otherwise it MUST fail closed with the stable account-neutral reason `conversation_owner_unavailable`. A resolved hard Codex-session mapping MUST constrain selection to its mapped account and MUST NOT be deleted or rebound when that account is excluded, outside model/API-key scope, unhealthy, budget-pressured, or capped. Hard-continuity work MUST fail closed when the required owner account is saturated and no continuity-preserving alternative exists. A selected replacement MUST NOT become the persistent affinity owner until every account-local stream/response-create lease and process-wide response-create admission required by that transport has succeeded. For HTTP bridge traffic, the replacement MUST also remain absent from canonical live and durable bridge ownership until that admission and affinity settlement succeeds, then replace the canonical lane rather than remain under an internal creation key. Publication eligibility, affinity CAS, durable claim, and canonical registry replacement MUST use one serialization order so an expired waiter or older reroute cannot partially publish or overwrite a newer lane. Capacity eviction MUST preserve the canonical publication target while its replacement is pending. The persistence operation MUST compare the previously observed owner atomically so a stale request cannot overwrite a newer mapping. If later bridge publication fails after affinity settlement, the service MUST attempt a compare-safe rollback without overwriting newer ownership.

#### Scenario: Soft work avoids saturated account

- **GIVEN** account A is at its account response-create cap
- **AND** account B is eligible and below cap
- **WHEN** a soft-affinity `/v1/responses` request is routed
- **THEN** the proxy selects account B instead of queueing on account A

#### Scenario: Bare Codex session affinity rebinds before visibility

- **GIVEN** a self-contained Responses request carries only a process-level session header as its `codex_session` affinity
- **AND** the mapped account is at its local stream or response-create cap
- **AND** another eligible account remains below cap
- **WHEN** account selection occurs before upstream dispatch and downstream-visible output
- **THEN** the proxy selects the alternate account
- **AND** persists that account as the session's new affinity mapping only after all required account-local and process-wide admission succeeds

#### Scenario: Unsaturated bare session owner remains sticky

- **GIVEN** a request carries only a process-level session header as its `codex_session` affinity
- **AND** the mapped account remains eligible and below its local cap
- **WHEN** account selection occurs
- **THEN** the mapped account remains selected
- **AND** the affinity mapping is not rebound

#### Scenario: Hard continuity owner saturation fails closed

- **GIVEN** a follow-up request requires a specific previous-response owner account
- **AND** that account is at its account stream or response-create cap
- **WHEN** no safe continuity-preserving alternative exists
- **THEN** the proxy returns a bounded local overload/continuity failure
- **AND** the failure reason is stable and low-cardinality

#### Scenario: No alternate preserves the local cap error

- **GIVEN** a mobility-qualified bare session mapping is saturated
- **AND** no eligible account remains below the applicable local cap
- **WHEN** account selection occurs
- **THEN** the proxy returns `account_stream_cap` or `account_response_create_cap`
- **AND** leaves the existing affinity mapping unchanged

#### Scenario: Second-stage account cap preserves the old mapping

- **GIVEN** a mobility-qualified request selected an alternate account and acquired its stream lease
- **AND** that account reaches its response-create cap before the request acquires the response-create lease
- **WHEN** the transport rejects or retries the request
- **THEN** the previous affinity owner remains unchanged
- **AND** a later successful request MAY compare-and-set the mapping from that previous owner

#### Scenario: Unresolved conversation does not choose an arbitrary account

- **GIVEN** a Responses request carries a nonblank `conversation`
- **AND** no hard Codex-session mapping or other authoritative owner resolves
- **WHEN** more than one account is eligible
- **THEN** selection fails closed with `conversation_owner_unavailable`
- **AND** no account is selected or penalized

#### Scenario: Single-account conversation scope is unambiguous

- **GIVEN** a Responses request carries a nonblank `conversation`
- **AND** no authoritative owner resolves
- **WHEN** exactly one account is eligible after model and API-key scope filtering
- **THEN** the request MAY select that account

#### Scenario: HTTP bridge replacement publishes only after final admission

- **GIVEN** a mobility-qualified HTTP bridge request creates a replacement process for a capped account
- **WHEN** response-create or process admission fails, or the affinity compare-and-set loses a race
- **THEN** the replacement is closed without entering canonical live or durable bridge ownership
- **AND** the previous canonical bridge remains unchanged
- **WHEN** all admission and affinity settlement succeeds
- **THEN** the replacement is published under the canonical key and the previous lane is retired

#### Scenario: Internal bridge fork promotes the successful affinity owner

- **GIVEN** an unanchored internal bridge fork wins a bare-session affinity CAS
- **WHEN** its request completes final admission
- **THEN** it publishes under the original canonical bridge key rather than its request-scoped creation key

#### Scenario: Capacity cannot evict a pending replacement target

- **GIVEN** an HTTP bridge replacement targets an existing canonical lane
- **AND** the bridge registry is at its configured session limit
- **WHEN** no other lane is safely evictable
- **THEN** replacement creation returns bounded local overload
- **AND** the existing canonical lane remains registered

#### Scenario: Direct WebSocket cap mobility is one-shot

- **GIVEN** a direct WebSocket request reroutes before send because its first account is response-create-capped
- **WHEN** the alternate account accepts the response-create frame
- **THEN** the request changes to account-bound reattach state for later recovery
- **AND** a retained response-create lease MUST match the account receiving any replayed frame

#### Scenario: Compact timeout cleanup is compare-safe

- **GIVEN** a self-contained soft compact request times out on its selected account
- **WHEN** timeout cleanup clears its mapping
- **THEN** deletion is conditional on that account still being the current owner
- **AND** owner-bearing Codex-session mappings are never cleared by this timeout path
