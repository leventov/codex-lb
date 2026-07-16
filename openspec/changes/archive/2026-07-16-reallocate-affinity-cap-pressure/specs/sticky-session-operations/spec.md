## MODIFIED Requirements

### Requirement: Soft bridge affinity can reroute under local pressure

Prompt-cache and sticky-thread bridge affinity that does not carry a hard continuity dependency MUST be treated as soft. A `codex_session` mapping derived only from a process-level session header MUST also be treated as soft for pre-visible account-cap selection when no preferred owner or replay/reattach state is present. Session-header mappings MUST use a source-namespaced opaque persistence key distinct from legacy-compatible raw turn-state keys so an equal client value cannot let soft affinity control a hard continuation mapping. Hard turn-state lookup MUST remain compatible with rows written before this distinction during rolling upgrades. A client-supplied or proxy-derived `prompt_cache_key` and a bare process-level session header are locality hints, not correctness dependencies; the proxy MAY reroute them under local pressure and accept lower cache-hit rates. When the preferred soft bridge session is saturated by queue depth, response-create gate pressure, bridge capacity, or account-local caps, the service MUST evaluate other eligible accounts/sessions before returning a local overload response. The service MUST emit internal diagnostics such as `internal_soft_affinity_reroute` for successful reroutes without adding those diagnostic names to the stable failure taxonomy or logging the raw sticky key.

#### Scenario: Prompt-cache bridge queue reroutes to an eligible account

- **GIVEN** a prompt-cache request's preferred bridge session queue is full
- **AND** another eligible account/session is below cap
- **WHEN** the request has no hard previous-response or turn-state continuity dependency
- **THEN** the proxy routes to the alternate account/session
- **AND** records an internal soft-affinity reroute diagnostic

#### Scenario: Bare session-header mapping reroutes on account cap

- **GIVEN** a pre-visible self-contained request carries a process-level session header
- **AND** it carries no previous-response, file, client turn-state, preferred-owner, live/durable bridge, or replay dependency
- **WHEN** its mapped account is capped and another eligible account is below cap
- **THEN** the proxy routes the request to the alternate account
- **AND** rebinds the `codex_session` mapping to that account

#### Scenario: Equal raw affinity values remain source-isolated

- **GIVEN** a process session header and a client turn-state header contain the same raw value
- **WHEN** the proxy derives their persistent selection keys
- **THEN** the namespaced session key is distinct from the legacy-compatible turn-state key
- **AND** session-header cap mobility cannot alter the turn-state mapping

#### Scenario: Prompt cache key does not override hard previous-response continuity

- **GIVEN** a `/v1/responses` request carries both `previous_response_id` and `prompt_cache_key`
- **AND** the previous response owner is known
- **WHEN** the prompt-cache preferred account differs from the previous-response owner
- **THEN** the proxy treats the request as hard owner-bound to the previous-response owner
- **AND** it does not route to the prompt-cache account when that account cannot preserve the stored response continuation

### Requirement: Hard continuity remains owner-bound and bounded

Requests that depend on client- or proxy-supplied `previous_response_id`, `conversation`, hard client turn-state, account-scoped `input_file.file_id` pins, a required preferred account, live or durable bridge ownership, replay/reattach state, downstream-visible output, or another required owner continuity source MUST NOT silently reroute to an account that cannot preserve continuity. A `previous_response_id` or `conversation` is a stored-object continuation reference and remains owner-bound even when owner lookup misses or the same request also carries a process-level session header, `prompt_cache_key`, or another soft locality key. A persisted hard Codex-session mapping MAY establish the owner for a request carrying `conversation`; without such a mapping or another authoritative owner, only an eligible pool containing exactly one account is unambiguous. Once a hard mapping resolves, it MUST remain a required-owner constraint: filtering, exclusions, health, quota pressure, or account caps MUST produce a fail-closed result rather than mapping deletion, fallback persistence, or cross-account selection. If the owner account/session is unavailable, ambiguous, or saturated, the service MUST fail closed with an explicit retryable continuity/local overload reason instead of flooding the owner queue indefinitely.

#### Scenario: Previous-response owner queue is saturated

- **WHEN** a `/v1/responses` follow-up requires a previous-response owner
- **AND** the owner session queue or account cap is saturated
- **THEN** the service fails closed with `hard_affinity_saturated` or `previous_response_owner_unavailable`
- **AND** it does not route to an unrelated account that lacks continuity state

#### Scenario: File-pinned request owner is capped

- **WHEN** a `/v1/responses` request references an `input_file.file_id` pinned to an owner account
- **AND** the owner account is at its account stream or response-create cap
- **THEN** the service returns a local account-cap overload for the owner
- **AND** it does not route the file reference to another account

#### Scenario: Session header cannot weaken durable ownership

- **GIVEN** a request carries a process-level session header
- **AND** a durable bridge lookup or proxy-injected previous-response anchor resolves an owner account
- **WHEN** that owner is saturated
- **THEN** the proxy retains hard owner routing and fails closed
- **AND** it does not apply bare-session cap reallocation

#### Scenario: Replay and visible work remain owner-bound

- **GIVEN** a request is reattaching, replaying, or has emitted downstream-visible output
- **WHEN** its owner account is saturated
- **THEN** the proxy does not use bare-session cap reallocation
- **AND** it preserves the existing bounded continuity or overload behavior

#### Scenario: Conversation owner lookup miss is account-neutral

- **GIVEN** a request carries a nonblank `conversation`
- **AND** neither a hard Codex-session mapping nor another authoritative owner resolves
- **WHEN** the eligible account pool does not contain exactly one account
- **THEN** the service fails with `conversation_owner_unavailable`
- **AND** it does not route to an arbitrary account or write account health

#### Scenario: Hard mapping owner leaves the eligible pool

- **GIVEN** a hard Codex-session mapping resolves account A
- **WHEN** account A is excluded or filtered from model/API-key scope while account B remains eligible
- **THEN** the service fails closed without selecting account B
- **AND** the mapping to account A remains unchanged
