## ADDED Requirements

### Requirement: Long-term memory is backed by Meko over MCP

`ILongTermMemory` SHALL be implemented against the Meko MCP server (`https://mcp.mekodata.ai/mcp`, Streamable HTTP, `mko_tkn_` bearer auth) using the `memory_*` tools plus `conversation_create`. The implementation SHALL map: `AddFactAsync`→`memory_add(text)`, `RecordAsync`→`memory_add(messages)` (subject to opt-in capture), `SearchAsync`→`memory_search`, `GetAsync`→`memory_get_by_id`, `ListAsync`→`memory_get_all`, `UpdateAsync`→`memory_update`, and `DeleteAsync`→`memory_delete_by_id`. (`FlushAsync` is a best-effort no-op — see the consistency requirement — since `flush_pending_memory_candidates` performs no server-side write and only returns an agent-directive.) Because `memory_add`/`memory_search`/`memory_get_all` require a `conversation_id` that is a UUID returned by `conversation_create`, the implementation SHALL call `conversation_create` to obtain one (lazily, once per dmon session, cached). The implementation SHALL pass `scope = "admin"` (the server's required fixed value) and SHALL serialize `messages` and `metadata` as JSON strings. The implementation SHALL NOT use any other `conversation_*` tool (`conversation_add_message`/`get`/`list`/`update`/`delete`) nor `knowledgebase_*`. All Meko-specific coupling SHALL be confined behind `ILongTermMemory`.

#### Scenario: Fact assertion vs. raw capture
- **WHEN** `AddFactAsync(fact)` is called
- **THEN** the implementation calls `memory_add` with the `text` input; **WHEN** `RecordAsync(turns)` results in capture, **THEN** it calls `memory_add` with the `messages` input

#### Scenario: Distillation is owned by the store
- **WHEN** turns or text are submitted to long-term memory
- **THEN** the implementation does not pre-curate which facts to keep; Meko's extraction pipeline derives the durable facts/relations

#### Scenario: Only the permitted tools are called
- **WHEN** any long-term operation is invoked
- **THEN** only `memory_*` tools and `conversation_create` (for the required `conversation_id`) are called; no other `conversation_*` tool and no `knowledgebase_*` tool is called

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

The implementation SHALL bind identity once per session via an ambient memory context (`agent_id` = `"dmon"`, the dmon session id, and an optional configured `datapack_id`). It SHALL send Meko's `scope` as the fixed required value `"admin"`. It SHALL obtain Meko's required `conversation_id` from `conversation_create` (cached per session). It SHALL map the per-call `MemoryScope` (`Session`, `Agent`, `User`, `Shared`; default `Agent`) onto Meko's `run_id`: `Session` sets `run_id` to the dmon session id (scoping the operation to this conversation), while the durable scopes (`Agent`/`User`/`Shared`) omit `run_id` for cross-conversation recall. The `scope="admin"` constant and the `MemoryScope`→`run_id` policy SHALL each live at a single adjustable point. The implementation SHALL only send `datapack_id` when a real datapack UUID is configured (omit to use the caller's default).

#### Scenario: Ambient identity is not repeated per call
- **WHEN** any long-term operation is invoked
- **THEN** `datapack_id` / `agent_id` / `conversation_id` are taken from the bound context, not passed by the caller

#### Scenario: Session scope filters by run_id; durable scopes do not
- **WHEN** an operation is invoked with `MemoryScope.Session`
- **THEN** the call sets Meko's `run_id` to the dmon session id (scoping it to this conversation); **WHEN** invoked with a durable scope (`Agent`/`User`/`Shared`), **THEN** `run_id` is omitted so recall spans conversations

#### Scenario: Scope is always the server's fixed value
- **WHEN** the implementation issues any `memory_*` or `conversation_create` call
- **THEN** it passes `scope = "admin"`

### Requirement: Long-term consistency and flush are best-effort

Long-term memory SHALL be treated as eventually consistent and SHALL NOT promise read-your-writes. **Live-verified (2026-05-30):** Meko's `flush_pending_memory_candidates` performs no server-side write — it only returns an *agent-directive* (a checklist telling a hook-less agent to scan recent turns and call `memory_add`). Because dmon captures explicitly via `memory_add` on `RecordAsync` (there is no client-side buffer to materialize) and acting on the directive is an agent-loop concern outside the store, `FlushAsync` for the Meko long-term tier SHALL be a best-effort no-op (it SHALL NOT depend on a server barrier and SHALL NOT promise that captured material is now recallable). `FlushAsync` SHALL return `ValueTask`.

#### Scenario: Flush is a best-effort no-op for long-term
- **WHEN** `FlushAsync()` is called on the Meko store
- **THEN** it completes successfully without promising server-side materialization (Meko's flush tool performs no write; capture already happened at `RecordAsync`)

#### Scenario: Read-your-writes is not promised for long-term
- **WHEN** material is captured to long-term and immediately searched
- **THEN** the contract does not guarantee it is already recallable (distillation/indexing may lag)

### Requirement: Long-term memory can be disabled

The system SHALL provide a disabled/no-op `ILongTermMemory` so that dmon functions with long-term memory turned off: writes no-op and searches return an empty list, without contacting Meko.

#### Scenario: Disabled long-term no-ops
- **WHEN** long-term memory is not configured (disabled)
- **THEN** `RecordAsync`/`AddFactAsync`/`UpdateAsync`/`DeleteAsync`/`FlushAsync` complete as no-ops and `SearchAsync`/`ListAsync`/`GetAsync` return empty/null without any MCP call
