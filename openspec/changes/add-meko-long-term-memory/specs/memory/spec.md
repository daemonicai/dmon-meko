## ADDED Requirements

### Requirement: Long-term memory is backed by Meko over MCP

`ILongTermMemory` SHALL be implemented against the Meko MCP server (`https://mcp.mekodata.ai/mcp`, Streamable HTTP, `mko_tkn_` bearer auth) using only the `memory_*` tools. The implementation SHALL map: `AddFactAsync`→`memory_add(text)`, `RecordAsync`→`memory_add(messages)` (subject to opt-in capture), `SearchAsync`→`memory_search`, `GetAsync`→`memory_get_by_id`, `ListAsync`→`memory_get_all`, `UpdateAsync`→`memory_update`, `DeleteAsync`→`memory_delete_by_id`, and `FlushAsync`→`flush_pending_memory_candidates`. The implementation SHALL NOT use Meko's `conversation_*` or `knowledgebase_*` tools. All Meko-specific coupling SHALL be confined behind `ILongTermMemory`.

#### Scenario: Fact assertion vs. raw capture
- **WHEN** `AddFactAsync(fact)` is called
- **THEN** the implementation calls `memory_add` with the `text` input; **WHEN** `RecordAsync(turns)` results in capture, **THEN** it calls `memory_add` with the `messages` input

#### Scenario: Distillation is owned by the store
- **WHEN** turns or text are submitted to long-term memory
- **THEN** the implementation does not pre-curate which facts to keep; Meko's extraction pipeline derives the durable facts/relations

#### Scenario: Out-of-scope tools are never called
- **WHEN** any long-term operation is invoked
- **THEN** no `conversation_*` or `knowledgebase_*` tool is called

### Requirement: Long-term results are parsed defensively

The implementation SHALL parse Meko's loosely-structured, LLM-tuned tool results tolerantly into `MemoryHit` records (`Source = LongTerm`), mapping graph edges into `Relations` and additional fields into `Metadata`. Missing or unexpected fields SHALL NOT throw into the caller; the implementation SHALL log and degrade (return what was recognized, or an empty result) instead.

#### Scenario: Well-formed result is mapped
- **WHEN** `memory_search` returns recognizable hits
- **THEN** each is mapped to a `MemoryHit` with `Source = LongTerm`, a populated `Text`/`Score`, and `Relations` set from any graph edges present

#### Scenario: Malformed or partial result degrades gracefully
- **WHEN** a Meko tool result is missing expected fields or contains unexpected structure
- **THEN** the implementation does not throw; it maps what it recognizes (or returns an empty list) and logs the anomaly

### Requirement: Long-term capture is opt-in

The Meko-backed `ILongTermMemory` SHALL apply a configurable capture policy to `RecordAsync`, defaulting to conservative (capture little or nothing) so that recording a turn does not incur hosted distillation cost unless explicitly enabled. A `RecordAsync` call whose policy keeps nothing SHALL complete successfully and SHALL NOT throw to signal "nothing kept".

#### Scenario: Default does not auto-capture every turn
- **WHEN** a turn is recorded and no capture policy has been opted into
- **THEN** the long-term store does not call `memory_add` for that turn

#### Scenario: Opted-in capture persists per policy
- **WHEN** a capture policy is configured (e.g. capture at session end)
- **THEN** the long-term store calls `memory_add` for the selected material at the policy's chosen point

### Requirement: Scope model binds identity once and carries scope per call

The implementation SHALL bind `datapack_id`, `agent_id`, and `conversation_id` once per session via an ambient memory context (`datapack_id` from configuration, `agent_id` = `"dmon"`, `conversation_id` = the dmon session id). The `scope` of a memory SHALL be supplied per write and per search as a `MemoryScope` (`Session`, `Agent`, `User`, `Shared`; default `Agent`). The implementation SHALL map `MemoryScope` to Meko's `scope` input through a single mapping point that can be adjusted once Meko's accepted values are confirmed. The implementation SHALL NOT populate Meko's `run_id`.

#### Scenario: Ambient identity is not repeated per call
- **WHEN** any long-term operation is invoked
- **THEN** `datapack_id` / `agent_id` / `conversation_id` are taken from the bound context, not passed by the caller

#### Scenario: run_id is unused
- **WHEN** the implementation issues any `memory_*` call
- **THEN** it does not set `run_id`

### Requirement: Long-term consistency and flush are best-effort

Long-term memory SHALL be treated as eventually consistent and SHALL NOT promise read-your-writes. `FlushAsync` SHALL be a best-effort materialization barrier for long-term: it triggers Meko's pending-candidate handling (`flush_pending_memory_candidates`) and, if that returns an agent-directive, acts on it. `FlushAsync` SHALL return `ValueTask`.

#### Scenario: Flush triggers pending-candidate handling
- **WHEN** `FlushAsync()` is called on the Meko store
- **THEN** `flush_pending_memory_candidates` is invoked and any returned directive is acted upon

#### Scenario: Read-your-writes is not promised for long-term
- **WHEN** material is captured to long-term and immediately searched
- **THEN** the contract does not guarantee it is already recallable (distillation/indexing may lag)

### Requirement: Long-term memory can be disabled

The system SHALL provide a disabled/no-op `ILongTermMemory` so that dmon functions with long-term memory turned off: writes no-op and searches return an empty list, without contacting Meko.

#### Scenario: Disabled long-term no-ops
- **WHEN** long-term memory is not configured (disabled)
- **THEN** `RecordAsync`/`AddFactAsync`/`UpdateAsync`/`DeleteAsync`/`FlushAsync` complete as no-ops and `SearchAsync`/`ListAsync`/`GetAsync` return empty/null without any MCP call
