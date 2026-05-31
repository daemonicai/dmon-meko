## 1. Abstractions (Dmon.Abstractions)

- [ ] 1.1 Define `IMemoryStore` base interface (`RecordAsync`, `SearchAsync`, `FlushAsync : ValueTask`) with `CancellationToken = default`
- [ ] 1.2 Define `IShortTermMemory : IMemoryStore` (adds verbatim/chronological reads carried over from `ISessionStore`)
- [ ] 1.3 Define `ILongTermMemory : IMemoryStore` (adds `AddFactAsync`, `GetAsync`, `ListAsync`, `UpdateAsync`, `DeleteAsync`)
- [ ] 1.4 Define `IMemory : IMemoryStore` facade with `ShortTerm`/`LongTerm` read-only properties
- [ ] 1.5 Define DTOs: `MemoryHit` (`Id`, `Text`, `Source`, `Score`, `Metadata?`, `Relations?`), `MemoryRelation`, `MemorySource` enum, `MemoryScope`, `MemoryContext`
- [ ] 1.6 Document the consistency contract on the interface XML docs (short-term read-your-writes; long-term eventually consistent; flush semantics)

## 2. Embedding generator (local)

- [ ] 2.1 Add `LlamaSharp` + `LLamaSharp.Backend.Cpu`; choose and pin the GGUF model + dimension
- [ ] 2.2 Implement a long-lived singleton `IEmbeddingGenerator<string, Embedding<float>>` over `LLamaEmbedder` (`PoolingType.Mean`, `Embeddings = true`)
- [ ] 2.3 L2-normalize all output vectors
- [ ] 2.4 Apply nomic task prefixes at the call sites (`search_document:` on store, `search_query:` on query) — above the generator
- [ ] 2.5 Serialize/pool embedder access (context not thread-safe); add a batched ingest path
- [ ] 2.6 First-run model acquisition/caching

## 3. Short-term hybrid index (index.db)

- [ ] 3.1 Add `Microsoft.Data.Sqlite` (`SQLitePCLRaw.bundle_e_sqlite3`) and bundle the `sqlite-vec` native loadable per-RID
- [ ] 3.2 Open per-session `index.db`, enable + `LoadExtension("vec0")`
- [ ] 3.3 Create schema: content table, `vec0` virtual table (cosine), FTS5 virtual table (`trigram` tokenizer for code)
- [ ] 3.4 Pin embedding model id + dimension in the db; rebuild-on-mismatch logic
- [ ] 3.5 Implement the single-query weighted-RRF hybrid search
- [ ] 3.6 Implement rebuild-from-JSONL (derived projection)
- [ ] 3.7 Wire `IShortTermMemory` over evolved `ISessionStore` + `index.db` (Record appends JSONL + embeds + upserts; Flush completes pending indexing)

## 4. Long-term store (Meko over MCP)

- [ ] 4.1 Add an MCP client; connect to `https://mcp.mekodata.ai/mcp` with `mko_tkn_` bearer auth (Streamable HTTP)
- [ ] 4.2 Bind `MemoryContext` (datapack from config, `agent_id="dmon"`, `conversation_id`=session id); never set `run_id`
- [ ] 4.3 Implement `ILongTermMemory(Meko)` mapping methods → `memory_*` tools; parse loosely-structured results into `MemoryHit` (+ graph `Relations`)
- [ ] 4.4 Implement the opt-in capture policy via options (default: conservative)
- [ ] 4.5 Implement `FlushAsync` against `flush_pending_memory_candidates` (and act on the returned directive if confirmed agent-driven)
- [ ] 4.6 Tolerate long-term being disabled (no-op implementation / null object)

## 5. Facade + fusion + DI

- [ ] 5.1 Implement `IMemory`: `RecordAsync` fan-out to both tiers; `AddFactAsync`→long-term; `FlushAsync`→both
- [ ] 5.2 Implement fused `SearchAsync` with rank-based RRF across tiers + provenance (`Source`) and dedupe
- [ ] 5.3 Add `AddDmonMemory()` `IServiceCollection` extension wiring short-term, long-term (optional), embedder, and facade

## 6. Tests

- [ ] 6.1 Embedding: normalization, prefix application, determinism
- [ ] 6.2 Short-term hybrid: single-modality recall, no-match empty, rebuild-from-JSONL, model-mismatch rebuild
- [ ] 6.3 Facade: record fan-out, fused search provenance, long-term-disabled path, flush barrier (short-term searchable after flush)
- [ ] 6.4 Long-term (Meko) mapping with a mocked MCP client; opt-in capture default behavior

## 7. Assumed defaults (proceed now; verify async on Discord)

These are best-guess defaults so implementation is not blocked. Each is sealed behind `ILongTermMemory` (or is a local-only concern), so a wrong guess is cheap to correct.

- [ ] 7.1 Proceed assuming `memory_add` is synchronous-enough and `FlushAsync(Meko)` acts on the flush directive itself (scan recent turns + `memory_add`); do NOT rely on long-term read-your-writes (short-term covers recency). Verify Meko flush/`memory_add` semantics on Discord.
- [ ] 7.2 Proceed with `MemoryScope { Session, Agent, User, Shared }`, default `Agent`, mapped to Meko's `scope` string. Verify accepted `scope` values on Discord and adjust the mapping.
- [ ] 7.3 Proceed with application-level upsert in one transaction for `index.db` (content + `vec0` + FTS5); revisit triggers only if it bottlenecks. (Local-only — no external verification needed.)
- [ ] 7.4 Proceed with bare `ValueTask` for `FlushAsync` (no payload). (Settled — no external verification needed.)
