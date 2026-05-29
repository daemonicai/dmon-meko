## Context

dmon-core is a .NET 10 agentic coding harness (nullable enabled, warnings-as-errors). It already has:
- `ISessionStore` (`Dmon.Core/Session`) — file/directory storage, JSONL messages + JSON metadata, atomic `.tmp`-rename writes, `IReadOnlyList<T>` reads. This is *de facto* short-term memory.
- An extension model (`IDmonExtension`, `IProviderExtension`), DI via `Microsoft.Extensions.DependencyInjection`, `Microsoft.Extensions.AI` for tools (`AIFunction`), and `ValueTask` for lightweight I/O (`ICredentialFileStore`).

We are introducing a memory abstraction with two tiers, modelled on the design of **Meko** (agent-native data layer on YugabyteDB) for the durable tier and on **memlite** (a hybrid FTS5+sqlite-vec engine) for the short-term tier. Meko's memory is built on **mem0**, whose `user_id` / `agent_id` / `run_id` scoping vocabulary directly informs our scope model.

Meko is consumed **only via MCP** — there is no REST API or SDK today; the programmatic ("level 3") path is documented as under development. Meko's memory operations are MCP tools (`memory_add`, `memory_search`, `memory_get_*`, `memory_update`, `memory_delete_*`, `flush_pending_memory_candidates`). `memory_add` is an *extraction pipeline*: it distills entities/relationships from text/messages via an LLM and stores vectors (pgvector) + graph edges (Meko AGE).

## Goals / Non-Goals

**Goals:**
- A clean `IMemory` facade over `IShortTermMemory` + `ILongTermMemory`, all sharing an `IMemoryStore` base, living in `Dmon.Abstractions`.
- Short-term: keep the directories-and-files model; add hybrid (vector + keyword) semantic search via a per-session, rebuildable `index.db`.
- Long-term: a Meko-backed `ILongTermMemory` whose interface shape is informed by Meko's `memory_*` tools, with **opt-in** capture so we never silently incur hosted distillation cost.
- An honest consistency contract and an explicit `FlushAsync` barrier.
- A scope model that maps Meko/mem0 identifiers onto dmon concepts and leaves room for "collective memory" later.

**Non-Goals:**
- The three-table (`content` / `vec0` / `fts5`) synchronization mechanism inside `index.db` — deferred as an implementation concern.
- Using Meko's `conversation_*` (verbatim history) or `knowledgebase_*` tools — verbatim history is the short-term tier's job; only `memory_*` is in scope.
- Multi-persona `agent_id` handling, cross-scope ("search another agent's memories") queries, and scope-widening / promotion to a shared knowledge base — future work, but the scope seam is designed in.
- A local `ILongTermMemory` implementation. The interface must permit one, but only Meko is built here.

## Decisions

### D1. Three interfaces over a shared `IMemoryStore` base
`IMemoryStore` = `RecordAsync(turns)`, `SearchAsync(query, …)`, `FlushAsync()`. `IShortTermMemory`, `ILongTermMemory`, and `IMemory` each extend it. The two tiers share *only* the fusable read plus record/flush; everything else (verbatim reads vs. CRUD-by-id) is tier-specific.
- *Why:* the stores are genuinely asymmetric; forcing one symmetric interface would leak. A thin shared base keeps the facade's fan-out/fuse logic uniform.
- *Alternatives:* (a) one fat `IMemory` — rejected, conflates tiers; (b) fully independent interfaces with no base — rejected, the facade could not treat them uniformly for record/search/flush.

### D2. Short-term = evolved `ISessionStore` + per-session `index.db`
Canonical storage stays JSONL files. A sibling `index.db` (SQLite) adds hybrid search and is a **derived, rebuildable projection** — delete it and replay the JSONL. Pin embedding model URL + dimension in the db; on mismatch, **rebuild** rather than mix vector spaces (memlite's `MODEL_MISMATCH` lesson, but we can afford rebuild because the index is not the source of truth).
- *Why:* preserves inspectability/diff-ability of files-and-dirs; makes the index disposable and model-version-safe.

### D3. Short-term hybrid search = raw `sqlite-vec` + FTS5 + RRF (no Semantic Kernel connector)
Use `Microsoft.Data.Sqlite` (`bundle_e_sqlite3`, which ships FTS5 and permits `LoadExtension`) + the `sqlite-vec` loadable. Express the hybrid as a single SQL query: a `vec0` KNN CTE and an FTS5 `MATCH` CTE, each rank-numbered, `full outer join`ed on a shared rowid, combined with weighted RRF arithmetic in SQL.
- *Why:* the `Microsoft.SemanticKernel.Connectors.SqliteVec` connector is vector-only (no `IKeywordHybridSearchable`), Preview, `EqualTo`-only filtering, and its auto-embed cannot apply nomic's dual prefixes (see D5). It abstracts the vec table away, making the single-query hybrid impossible and forcing a two-round-trip C# merge. Going raw gives one owned SQL surface, weighted RRF, full knob control, and transactional consistency.
- *Cost:* one more native loadable (`sqlite-vec`) and the `LoadExtension` dance — the same class of native-dependency management already accepted by choosing LlamaSharp.

### D4. RRF fusion is rank-based, never score-based — at two levels
Inside `index.db` (BM25 vs. cosine) and at the facade (local embedding model vs. Meko's server model), scores live in incomparable spaces. Fuse by rank with `1/(k+rank)` (conventionally `k=60`), with per-modality weights. Apply any confidence threshold *before* fusing, as a per-list cutoff.

### D5. Local embeddings via LlamaSharp wrapped as `IEmbeddingGenerator<string, Embedding<float>>`
~100 MB nomic-like GGUF, `PoolingType.Mean`, `Embeddings = true`, `LLamaSharp.Backend.Cpu` (covers CPU on all platforms + Metal on macOS).
- **L2-normalize** every vector — LlamaSharp does **not** normalize by default.
- **nomic dual prefixes** — embed stored text as `search_document: …`, queries as `search_query: …`. Because the prefix is call-site-dependent and `IEmbeddingGenerator` is prefix-blind, prefix logic lives *above* the generator at the two call sites (this is also why SK auto-embed is unusable here).
- One long-lived singleton embedder; serialize/pool access (the context is not thread-safe); batch on ingest.

### D6. FTS5 tokenizer tuned for code
Default `unicode61` fragments identifiers/paths/error codes; use `trigram` (or custom) on code-bearing fields — keyword recall on identifiers is the whole reason FTS5 is in the mix.

### D7. Long-term = Meko over MCP, `memory_*` only
`ILongTermMemory(Meko)` is an MCP client to `https://mcp.mekodata.ai/mcp`. Tool→method mapping: `memory_add(text)`→`AddFactAsync`; `memory_add(messages)`→`RecordAsync`/capture; `memory_search`→`SearchAsync`; `memory_get_by_id`→`GetAsync`; `memory_get_all`→`ListAsync`; `memory_update`→`UpdateAsync`; `memory_delete_by_id`→`DeleteAsync`; `flush_pending_memory_candidates`→`FlushAsync`. Ignore `conversation_*` and `knowledgebase_*`.

### D8. Opt-in capture; facade fan-out is unconditional
`IMemory.RecordAsync` always calls both `ShortTerm.RecordAsync` (keeps everything: verbatim + index) and `LongTerm.RecordAsync` (keeps only what its opt-in policy selects — often nothing by default). `AddFactAsync` targets long-term (`text`); short-term coverage of that fact rides the recorded turn. Capture cadence/policy lives in the Meko implementation's options, not the facade.
- *Why:* keeps the facade dumb and uniform; keeps cost/noise policy in one place; lets a disabled/no-op long-term store work transparently.

### D9. Scope model
Bind once per session (ambient `MemoryContext`): `datapack_id` (config), `agent_id` (`"dmon"`), `conversation_id` (session id). Carry **`scope`** per-call (a `MemoryScope`, default durable) — it is intrinsically per-memory (a user preference vs. a task-local fact). **Do not** use `run_id`: it is mem0's auto-expiring ephemeral tier, which the short-term store already owns. This makes the short-term/long-term split *also* a scope split (ephemeral/session vs. durable/user-agent), mirroring mem0's session-vs-user auto-recall; `MemoryHit.Source` is the "presented separately" provenance.

**Assumed default (verify on Discord):** `MemoryScope { Session, Agent, User, Shared }`, defaulting to `Agent`, mapped to Meko's `scope` string. `Session` is short-term's natural scope; `Agent`/`User` are the durable long-term scopes; `Shared` is the future collective/promotion scope. The exact accepted Meko values are unconfirmed; the enum→string mapping is the single point to adjust when confirmed.

### D10. `MemoryHit` is the fused result currency
`{ Id, Text, Source (ShortTerm|LongTerm), Score, Metadata?, Relations? }`. `Relations` is long-term-only (Meko AGE graph) and null from short-term. One shared type keeps facade fusion to a single list.

### D11. Conventions
Interfaces/DTOs in `Dmon.Abstractions`; `Task<T>` for I/O, `ValueTask` for `FlushAsync` (matches `ICredentialFileStore`), `CancellationToken = default` everywhere, `IReadOnlyList<T>` for collections, nullable for optional, `record` DTOs. DI via `AddDmonMemory()`.

## Risks / Trade-offs

- **Meko `flush_pending_memory_candidates` may be an agent-directive, not a server barrier** (its description: *"return a directive instructing the agent to scan recent user turns … and call `memory_add`"*) → `FlushAsync(Meko)` may not guarantee read-your-writes by itself. Mitigation: have the Meko adapter *act on* the directive (do the scan + `memory_add`), and treat the facade's flush as "authoritative for short-term, best-effort for long-term." Confirm with Meko early-access.
- **Long-term eventual consistency** (distillation lag, async candidates) → freshly captured material may not be immediately recallable across the session/agent boundary. Mitigation: short-term covers recency within a session; `FlushAsync` before a known handoff; the contract explicitly does not promise read-your-writes on long-term.
- **Per-turn embedding latency/cost (CPU)** → blocking the turn loop. Mitigation: singleton embedder, batched ingest, async indexing; accept a small "written-to-JSONL-but-not-yet-searchable" window.
- **Native-dependency footprint** (sqlite-vec + llama.cpp) → packaging weight, trimming/AOT friction, per-RID payloads. Mitigation: CPU backend by default; document the supported RIDs; keep both behind the abstraction so they're swappable.
- **Meko is early-access** → tool schemas/semantics may shift, and outputs are tuned for LLM consumption (loose structure). Mitigation: confine all Meko coupling to the long-term implementation behind `ILongTermMemory`; parse defensively.
- **Embedding-model swap** → silent vector-space mixing. Mitigation: pin model+dim in `index.db`, rebuild on mismatch (D2).

## Migration Plan

Additive — no existing OpenSpec specs to modify. Long-term memory is opt-in and dmon must run with it disabled (short-term only). Rollout: (1) interfaces + DTOs in `Dmon.Abstractions`; (2) short-term `index.db` + embedder; (3) Meko long-term implementation; (4) facade + DI. Rollback = disable long-term and/or delete `index.db` (canonical JSONL is untouched).

## Open Questions

We are proceeding now on best-guess defaults rather than blocking; all Meko-facing guesses are sealed behind `ILongTermMemory`, so corrections are cheap. Items below are tagged with their assumed default and whether they still need external confirmation.

- **[assumed → verify on Discord]** Semantics of `flush_pending_memory_candidates` (agent-directive vs. server barrier) and whether `memory_add` is synchronous or candidate-queued. *Default:* treat `memory_add` as synchronous-enough; `FlushAsync(Meko)` acts on the directive itself (scans recent turns + `memory_add`); do not rely on long-term read-your-writes (short-term covers recency).
- **[VERIFIED live, 2026-05-29 — corrects D7/D9]** Against the real Meko server: `scope` is the fixed string `"admin"` (not a partition selector); `memory_*` calls **require** a `conversation_id` from `conversation_create` (so the long-term impl uses `conversation_create`, contrary to the original "ignore `conversation_*`"); partitioning/filtering is via `agent_id` + `run_id`, so `MemoryScope` maps onto `run_id` (Session→run_id, durable scopes omit it) — **not** onto `scope`; `messages`/`metadata` are JSON strings. The corrected model lives in the `add-meko-long-term-memory` change's design (D7/D9) and specs; this umbrella decision is superseded there.
- **[settled]** `index.db` three-table sync mechanism. *Decision:* application-level upsert in one transaction; revisit triggers only if it bottlenecks.
- **[settled]** `FlushAsync` return shape. *Decision:* bare `ValueTask` (no payload).
- **[deferred — local choice]** GGUF embedding model/dimension, and whether to use nomic v1.5 Matryoshka truncation to shrink `index.db`. Decide during task group 2; no external dependency.
- **[deferred — local choice]** Shape of the opt-in capture policy (per-turn vs. session-end, role/content filters, sampling). Decide during task group 4.4; no external dependency.
