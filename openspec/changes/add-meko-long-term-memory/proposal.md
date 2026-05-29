> **Prerequisite — the memory abstractions.** This change implements `ILongTermMemory` against the memory contracts in **dmon-core's `Dmon.Abstractions.Memory`** (`IMemoryStore`, `ILongTermMemory`, `MemoryHit`, `MemoryRelation`, `MemorySource`, `MemoryScope`, `MemoryContext`), introduced by the `add-memory-abstraction` umbrella. Per the agreed decision, dmon-meko consumes those types via a **`ProjectReference` to `../dmon-core/src/Dmon.Abstractions/Dmon.Abstractions.csproj`** (dmon-core is checked out as a sibling working directory). When `Dmon.Abstractions` is published as a NuGet package, switch this to a `PackageReference`. If the `Dmon.Abstractions.Memory` types are absent, stop — do not redefine them here.
>
> **Provenance.** This is the dmon-meko half of the cross-repo memory plan (umbrella: dmon-meko `add-memory-abstraction`). The abstractions + short-term tier land in dmon-core; this change builds the Meko-backed long-term tier here. It does **not** depend on the short-term tier (§2/§3) and can be developed in parallel — the two tiers only meet in the facade (a later, separate change).

## Why

The memory abstraction defines `ILongTermMemory` for durable, cross-session/cross-agent recall, but ships no implementation. Meko (YugabyteDB's agent-native data layer, early access) is the chosen durable backend: its mem0-based memory operations distill entities/relations from turns and store vectors (pgvector) + graph edges. Meko is consumed **only via MCP** — there is no REST API or SDK today — so this change adds an MCP-client-backed `ILongTermMemory` confined to Meko's `memory_*` tools, with **opt-in** capture so recording a turn never silently incurs hosted distillation cost.

## What Changes

- Bootstrap the dmon-meko solution (first code in this repo): `src/Dmon.Memory.Meko` + `test/Dmon.Memory.Meko.Tests`.
- Add an **MCP client** (`ModelContextProtocol`) connecting to `https://mcp.mekodata.ai/mcp` over **Streamable HTTP** with `Authorization: Bearer mko_tkn_…`.
- Implement **`ILongTermMemory(Meko)`** mapping its methods to Meko's `memory_*` tools (D7), parsing Meko's loosely-structured, LLM-tuned results defensively into `MemoryHit` (with graph edges → `Relations`).
- Add an **opt-in capture policy** (`MekoLongTermOptions`, default conservative/none) gating `RecordAsync` → `memory_add(messages)`.
- Bind the **scope model**: ambient `MemoryContext` (datapack from config, `agent_id="dmon"`, `conversation_id`=session id), per-call `MemoryScope`, **never** `run_id`; a single `MemoryScope`→Meko-`scope`-string mapping point.
- Implement **`FlushAsync`** over `flush_pending_memory_candidates`, and a **disabled/no-op** `ILongTermMemory` so dmon runs with long-term memory off.
- Add a focused `AddMekoLongTermMemory(...)` DI helper + options binding (the cross-tier `AddDmonMemory()` lives in the facade change).

**Out of scope:** the `IMemory` facade and cross-tier RRF fusion (a later change, in dmon-core); the local embedder and `index.db` short-term tier (dmon-core §2/§3); Meko's `conversation_*` and `knowledgebase_*` tools (explicitly ignored).

## Capabilities

### New Capabilities

- `memory`: the dmon long-term memory tier — the Meko-over-MCP `ILongTermMemory` implementation, opt-in capture, the scope model's long-term binding, result provenance/relations, the long-term consistency/flush contract, and the disabled/no-op path. (The short-term tier and the facade are specified and built elsewhere; this change adds only the long-term requirements to the `memory` capability.)

### Modified Capabilities

<!-- None. -->

## Impact

- **New project / code**: `src/Dmon.Memory.Meko` (referencing dmon-core's `Dmon.Abstractions`) + `test/Dmon.Memory.Meko.Tests`; a solution file and `Directory.Build.props` (net10.0, nullable, warnings-as-errors). Gates: `dotnet build` / `dotnet test` / `dotnet format` / `openspec validate --strict`.
- **New dependencies**: `ModelContextProtocol` (preview MCP SDK); `Microsoft.Extensions.Options`, `Microsoft.Extensions.Configuration.Abstractions`, `Microsoft.Extensions.DependencyInjection.Abstractions`, `Microsoft.Extensions.Logging.Abstractions`. `Microsoft.Extensions.AI.Abstractions` (`ChatMessage`) and `System.Text.Json` arrive transitively. Tests: `xunit` + `xunit.runner.visualstudio` + `Microsoft.NET.Test.Sdk` + `coverlet.collector` (matching dmon-core; **no** mocking framework — hand-rolled fakes).
- **Version alignment**: `ModelContextProtocol` pulls in `Microsoft.Extensions.AI.Abstractions`; align it to dmon-core's **10.6.0** M.E.AI pin to avoid an ABI mismatch.
- **External service**: Meko (`https://mcp.mekodata.ai/mcp`, `mko_tkn_` API key). Optional/opt-in; dmon must function with long-term memory disabled. Live round-trips need real credentials → a human-in-the-loop verification step.
- **Native footprint**: none — this tier is a pure MCP/HTTP client (the native loadables belong to the short-term tier in dmon-core).
