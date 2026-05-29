## ADDED Requirements

### Requirement: Memory store base contract

The system SHALL define an `IMemoryStore` interface in `Dmon.Abstractions` that every memory store implements, exposing `RecordAsync` (ingest raw turns), `SearchAsync` (query for relevant entries), and `FlushAsync` (materialization barrier). `FlushAsync` SHALL return `ValueTask`; the read/record operations SHALL return `Task`/`Task<T>` and SHALL accept a `CancellationToken` defaulting to `default`.

#### Scenario: Every store is recordable, searchable, and flushable
- **WHEN** any concrete memory store (`IShortTermMemory`, `ILongTermMemory`, or the `IMemory` facade) is used
- **THEN** it exposes `RecordAsync`, `SearchAsync`, and `FlushAsync` with the signatures defined by `IMemoryStore`

#### Scenario: Record is best-effort, not a persistence guarantee
- **WHEN** `RecordAsync` is called on a store whose policy chooses to keep none of the supplied turns
- **THEN** the call completes successfully and the store persists only what its policy selects (it SHALL NOT throw to signal "nothing kept")

### Requirement: Memory facade fans out writes and fuses reads

The system SHALL provide an `IMemory` facade exposing read-only `ShortTerm` and `LongTerm` properties (typed `IShortTermMemory` / `ILongTermMemory`). `IMemory.RecordAsync` SHALL invoke both `ShortTerm.RecordAsync` and `LongTerm.RecordAsync` unconditionally. `IMemory.AddFactAsync` SHALL target long-term memory. `IMemory.SearchAsync` SHALL return results fused from both tiers. `IMemory.FlushAsync` SHALL flush both tiers.

#### Scenario: Record reaches both tiers
- **WHEN** `IMemory.RecordAsync(turns)` is called
- **THEN** the turns are passed to both `ShortTerm.RecordAsync` and `LongTerm.RecordAsync` (the long-term store then applies its own opt-in capture policy)

#### Scenario: Explicit single-tier access
- **WHEN** a caller needs to search exactly one tier
- **THEN** it calls `memory.ShortTerm.SearchAsync(...)` or `memory.LongTerm.SearchAsync(...)` directly, bypassing fusion

#### Scenario: Long-term disabled
- **WHEN** long-term memory is not configured
- **THEN** `IMemory` still functions: writes go to short-term, `SearchAsync` returns short-term results only, and long-term operations no-op

### Requirement: Short-term memory preserves files and adds a derived semantic index

`IShortTermMemory` SHALL retain the directories-and-files model of `ISessionStore` (canonical per-session JSONL as the source of truth) and SHALL add a per-session SQLite `index.db` providing hybrid semantic search. The `index.db` SHALL be a derived, rebuildable projection of the JSONL: deleting it and replaying the JSONL SHALL reproduce the searchable state. The `index.db` SHALL pin the embedding model identifier and vector dimension, and on a mismatch SHALL rebuild rather than mix vector spaces.

#### Scenario: Index is rebuildable from canonical storage
- **WHEN** `index.db` is deleted and the session is re-opened
- **THEN** the index is rebuilt from the JSONL with no loss of recallable content

#### Scenario: Embedding model change triggers rebuild
- **WHEN** the configured embedding model or dimension differs from the values pinned in an existing `index.db`
- **THEN** the index is rebuilt; vectors from different models are never mixed in the same index

#### Scenario: Verbatim reads remain available
- **WHEN** a caller requests chronological/verbatim session content
- **THEN** `IShortTermMemory` serves it from the canonical JSONL (the semantic index does not replace verbatim reads)

### Requirement: Short-term search is hybrid keyword + vector with rank-based fusion

`IShortTermMemory.SearchAsync` SHALL combine a vector (KNN) result and a full-text (FTS5) result over the session `index.db` and fuse them using Reciprocal Rank Fusion. Fusion SHALL operate on rank position, not on raw scores. Stored text SHALL be embedded with the document task-prefix and queries with the query task-prefix, and all vectors SHALL be L2-normalized before storage and comparison.

#### Scenario: A result strong in only one modality still surfaces
- **WHEN** an entry is a top vector match but a weak keyword match (or vice versa)
- **THEN** RRF includes it in the fused ranking rather than discarding it

#### Scenario: Cross-modality scores are never directly compared
- **WHEN** vector distances and FTS5 BM25 scores are combined
- **THEN** the fusion uses each result's rank within its own list, not the raw distance/BM25 magnitude

#### Scenario: No matches
- **WHEN** neither index returns a candidate for the query
- **THEN** `SearchAsync` returns an empty list

### Requirement: Long-term memory is backed by Meko over MCP

`ILongTermMemory` SHALL be implementable against the Meko MCP server using only the `memory_*` tools. The implementation SHALL map: `AddFactAsync`→`memory_add(text)`, `RecordAsync`→`memory_add(messages)` (subject to opt-in capture), `SearchAsync`→`memory_search`, `GetAsync`→`memory_get_by_id`, `ListAsync`→`memory_get_all`, `UpdateAsync`→`memory_update`, `DeleteAsync`→`memory_delete_by_id`, and `FlushAsync`→`flush_pending_memory_candidates`. The implementation SHALL NOT use Meko's `conversation_*` or `knowledgebase_*` tools. All Meko-specific coupling SHALL be confined behind `ILongTermMemory`.

#### Scenario: Fact assertion vs. raw capture
- **WHEN** `AddFactAsync(fact)` is called
- **THEN** the implementation calls `memory_add` with the `text` input; **WHEN** `RecordAsync(turns)` results in capture, **THEN** it calls `memory_add` with the `messages` input

#### Scenario: Distillation is owned by the store
- **WHEN** turns or text are submitted to long-term memory
- **THEN** the implementation does not pre-curate which facts to keep; Meko's extraction pipeline derives the durable facts/relations

### Requirement: Long-term capture is opt-in

The Meko-backed `ILongTermMemory` SHALL apply a configurable capture policy to `RecordAsync`, defaulting to conservative (capture little or nothing) so that recording a turn does not incur hosted distillation cost unless explicitly enabled.

#### Scenario: Default does not auto-capture every turn
- **WHEN** the facade records a turn and no capture policy has been opted into
- **THEN** the long-term store does not call `memory_add` for that turn

#### Scenario: Opted-in capture persists per policy
- **WHEN** a capture policy is configured (e.g. capture at session end)
- **THEN** the long-term store calls `memory_add` for the selected material at the policy's chosen point (e.g. on `FlushAsync`)

### Requirement: Scope model binds identity once and carries scope per call

The system SHALL bind `datapack_id`, `agent_id`, and `conversation_id` once per session via an ambient memory context (`datapack_id` from configuration, `agent_id` = `"dmon"`, `conversation_id` = the dmon session id). The `scope` of a memory SHALL be supplied per write and per search as a `MemoryScope` with values `Session`, `Agent`, `User`, and `Shared`, defaulting to `Agent` (a durable scope). The long-term implementation SHALL map `MemoryScope` to Meko's `scope` input through a single mapping point that can be adjusted once Meko's accepted values are confirmed. The system SHALL NOT populate Meko's `run_id`; ephemeral/session scope is owned by short-term memory.

#### Scenario: Ambient identity is not repeated per call
- **WHEN** any long-term operation is invoked
- **THEN** `datapack_id` / `agent_id` / `conversation_id` are taken from the bound context, not passed by the caller

#### Scenario: Scope varies per memory
- **WHEN** a durable user preference and a task-local fact are written in the same session
- **THEN** each may carry a different `MemoryScope`, supplied at the call site

#### Scenario: run_id is unused
- **WHEN** the Meko implementation issues a `memory_*` call
- **THEN** it does not set `run_id`

### Requirement: Memory results carry provenance

`SearchAsync` SHALL return `MemoryHit` records exposing at least `Id`, `Text`, `Source` (`ShortTerm` or `LongTerm`), and `Score`, with optional `Metadata` and optional `Relations`. `Relations` SHALL be populated only for long-term (graph) results and SHALL be null/empty for short-term results.

#### Scenario: Fused results are attributable to a tier
- **WHEN** `IMemory.SearchAsync` returns fused results
- **THEN** each `MemoryHit` indicates via `Source` whether it came from short-term or long-term memory

#### Scenario: Graph relations only from long-term
- **WHEN** a short-term result is returned
- **THEN** its `Relations` is null/empty (the keyword+vector index has no graph)

### Requirement: Consistency contract and flush barrier

Short-term memory SHALL be read-your-writes within a session (subject only to indexing latency). Long-term memory SHALL be treated as eventually consistent and SHALL NOT promise read-your-writes. `FlushAsync` SHALL act as a materialization barrier: authoritative for short-term (it completes pending indexing), and best-effort for long-term (it triggers Meko's pending-candidate handling).

#### Scenario: Flush makes short-term writes searchable
- **WHEN** turns are recorded and `ShortTerm.FlushAsync()` then completes
- **THEN** those turns are searchable via `ShortTerm.SearchAsync`

#### Scenario: Recency is covered within a session despite long-term lag
- **WHEN** material has been captured to long-term but not yet distilled/indexed there
- **THEN** the same-session caller still recalls it via short-term memory (the facade fuses both tiers)

#### Scenario: Flush before a handoff
- **WHEN** a session/agent handoff is imminent and `IMemory.FlushAsync()` completes
- **THEN** short-term pending writes are materialized and long-term pending-candidate handling has been triggered
