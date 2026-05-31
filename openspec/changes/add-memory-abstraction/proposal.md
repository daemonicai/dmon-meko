## Why

dmon-core has no first-class memory abstraction. `ISessionStore` already gives us verbatim, per-session, chronological storage — effectively short-term memory — but there is no durable, cross-session recall, and no semantic search over either tier. We want agents to (a) semantically recall what happened earlier *this* session and (b) build a compounding, cross-session/cross-agent memory. Meko (YugabyteDB's agent-native data layer, early access) is a natural fit for the durable tier, and its mem0-based API gives us a proven shape to model the interface on.

## What Changes

- Introduce a `memory` capability in `Dmon.Abstractions` built around three interfaces over a shared base:
  - `IMemoryStore` — the common contract: `RecordAsync`, `SearchAsync`, `FlushAsync` (`ValueTask`).
  - `IShortTermMemory : IMemoryStore` — an evolution of `ISessionStore`. Keeps the directories-and-files model (canonical JSONL) and **adds** a per-session `index.db` for hybrid semantic search.
  - `ILongTermMemory : IMemoryStore` — durable, cross-session recall; adds `AddFactAsync`, `GetAsync`, `ListAsync`, `UpdateAsync`, `DeleteAsync`.
  - `IMemory : IMemoryStore` — a facade exposing `ShortTerm { get; }` / `LongTerm { get; }`, fanning writes to both and fusing reads across both.
- Add a **short-term semantic index**: per-session `index.db` via raw `Microsoft.Data.Sqlite` + the `sqlite-vec` extension + FTS5, fused with Reciprocal Rank Fusion (RRF). Embeddings come from a local model (LlamaSharp) wrapped as `IEmbeddingGenerator<string, Embedding<float>>`. The `index.db` is a **derived, rebuildable projection** of the canonical JSONL.
- Add a **Meko-backed `ILongTermMemory`** implementation that drives the Meko MCP server (`memory_*` tools) over Streamable HTTP. Capture is **opt-in**: the facade always calls `LongTerm.RecordAsync`, but the implementation decides what (if anything) to persist.
- Define the **scope model**: `datapack_id` / `agent_id` / `conversation_id` bound once per session (ambient context); `scope` carried per-call (defaulting to a durable scope); `run_id` unused (the short-term tier owns ephemeral scope).
- Define the **consistency contract**: short-term is read-your-writes within a session; long-term is eventually consistent; `FlushAsync` is a materialization barrier (authoritative for short-term, best-effort for Meko pending confirmation).

## Capabilities

### New Capabilities
- `memory`: the dmon memory abstraction — the `IMemory` / `IShortTermMemory` / `ILongTermMemory` contracts, the short-term hybrid search index, the Meko-backed long-term store, the scope model, and the consistency/flush semantics.

### Modified Capabilities
<!-- None. `ISessionStore` lives in Dmon.Core and has no existing OpenSpec spec; its evolution is captured here as new `memory` requirements rather than a delta. -->

## Impact

- **New project / code**: memory interfaces + DTOs in `Dmon.Abstractions`; a short-term implementation evolving `Dmon.Core/Session`; a new Meko long-term implementation (likely `Dmon.Memory.Meko` or an extension); DI wiring via an `AddDmonMemory()` extension method.
- **New dependencies**: `Microsoft.Data.Sqlite` (`SQLitePCLRaw.bundle_e_sqlite3`), the `sqlite-vec` native loadable extension (per-RID), `LlamaSharp` + `LLamaSharp.Backend.Cpu`, `Microsoft.Extensions.AI` (`IEmbeddingGenerator`), an MCP client for the Meko endpoint, and a local embedding GGUF model (~100 MB, downloaded on first run).
- **External service**: Meko (`https://mcp.mekodata.ai/mcp`, `mko_tkn_` API key). Optional/opt-in; dmon must function with long-term memory disabled.
- **Native-dependency footprint**: two native loadables (sqlite-vec, llama.cpp via LlamaSharp) enter the .NET process — packaging and (future) trimming/AOT implications.
