# dmon-meko

**Meko-backed long-term memory for [dmon](https://github.com/daemonicai/dmon-core).**

`dmon-meko` provides dmon's durable, cross-session memory tier: an implementation of
`ILongTermMemory` driven over the [Meko](https://mekodata.ai) agent-native data layer
(YugabyteDB, early access) via its MCP server. It is a C#/.NET (`net10.0`) library shipped
as a NuGet package.

dmon's memory abstraction is split across two repos:

| Tier | Lives in | What |
|------|----------|------|
| Contracts (`IMemory`, `IShortTermMemory`, `ILongTermMemory`, DTOs) | **dmon-core** (`Dmon.Abstractions.Memory`) | the shared interfaces |
| Short-term (per-session hybrid `sqlite-vec`+FTS5 index, local embeddings) | **dmon-core** (`Dmon.Memory`) | semantic recall within a session |
| **Long-term (Meko over MCP)** | **dmon-meko** (this repo, `Dmon.Memory.Meko`) | durable cross-session/agent recall |

This repo owns only the long-term tier. The two tiers meet behind the `IMemory` facade (a
later change).

## How it works

`MekoLongTermMemory` implements `ILongTermMemory` by calling Meko's `memory_*` MCP tools over
**Streamable HTTP** (`https://mcp.mekodata.ai/mcp`, `mko_tkn_` bearer auth). All Meko/MCP
coupling is confined behind the interface; the rest of dmon never sees an `McpClient`.

Method → tool mapping:

| `ILongTermMemory` | Meko tool |
|-------------------|-----------|
| `AddFactAsync(text)` | `memory_add` (text) |
| `RecordAsync(turns)` | `memory_add` (messages) — gated by opt-in capture |
| `SearchAsync(query)` | `memory_search` |
| `GetAsync(id)` | `memory_get_by_id` |
| `ListAsync()` | `memory_get_all` |
| `UpdateAsync(id, text)` | `memory_update` |
| `DeleteAsync(id)` | `memory_delete_by_id` |
| `FlushAsync()` | *(best-effort no-op — see below)* |

### Behaviour notes (verified against the live Meko server)

- **Scope.** Meko's `scope` argument is the fixed string `"admin"`. Partitioning/filtering is by
  `agent_id` (`"dmon"`) + `run_id`, so dmon's `MemoryScope` maps onto **`run_id`**:
  `Session` → `run_id` = the dmon session id (normalized to hex, since Meko parses it as
  `int(x, 16)`); the durable scopes (`Agent`/`User`/`Shared`) omit `run_id` for cross-conversation
  recall.
- **Conversations.** `memory_*` calls require a `conversation_id` that is a UUID from
  `conversation_create`; the store creates one lazily per session and caches it. No other
  `conversation_*` tool (nor `knowledgebase_*`) is used.
- **Opt-in capture.** `RecordAsync` is gated by `MekoCaptureMode` (`None` by default), so recording a
  turn never silently incurs hosted distillation cost. `AddFactAsync` always persists.
- **Eventual consistency.** Long-term memory is **not** read-your-writes — freshly added memories
  may not be immediately searchable (Meko distills/indexes asynchronously). `memory_add` itself is
  synchronous enough to return the created id.
- **`FlushAsync` is a best-effort no-op.** Meko's `flush_pending_memory_candidates` performs no
  server-side write (it returns an agent-directive for hook-less agents); dmon captures explicitly
  at `RecordAsync`, so there is nothing to materialize.
- **Disabled tier.** A no-op `ILongTermMemory` lets dmon run with long-term memory turned off
  (writes no-op, searches return empty) — the MCP client is never on the critical path.

## Getting started

Requires the .NET 10 SDK.

```bash
dotnet build Dmon.Meko.slnx          # warnings-as-errors, nullable enabled
dotnet test  Dmon.Meko.slnx          # unit tests (offline; uses a fake MCP invoker)
dotnet format --verify-no-changes    # style gate
```

The memory contracts come from the **`Dmon.Abstractions`** NuGet package (a normal
`PackageReference`), so a clean checkout builds without dmon-core present.

### Live smoke test (optional, hits the real Meko service)

The integration test is environment-gated and skipped unless a key is present. It is self-cleaning
(adds a uniquely-marked memory, then deletes it):

```bash
MEKO_API_KEY=mko_tkn_... dotnet test Dmon.Meko.slnx --filter "Category=Live"
```

Never hard-code or commit the key — it is read only from the environment.

## Configuration & DI

```csharp
services.AddMekoLongTermMemory(options =>
{
    options.ApiKey   = configuration["Meko:ApiKey"];   // mko_tkn_...
    options.Endpoint = "https://mcp.mekodata.ai/mcp";  // default
    options.CaptureMode = MekoCaptureMode.None;        // opt-in
    // options.DatapackId = "<uuid>";                  // optional; omit for the default datapack
});
```

If `ApiKey` is empty, the registration falls back to the disabled (no-op) store.

## Project layout

```
src/Dmon.Memory.Meko/        # the long-term implementation
test/Dmon.Memory.Meko.Tests/ # xUnit tests (hand-rolled fakes; no mocking framework)
openspec/                    # spec-driven change history (see below)
```

## Development

Feature work is spec-driven via **OpenSpec** (`openspec/`). See [`CLAUDE.md`](./CLAUDE.md) for the
authoritative apply workflow (orchestrator + `worker`/`reviewer` agents, section-by-section commits,
the build/test/format/validate gates). Shipped changes are archived under
`openspec/changes/archive/`.

## Dependencies

- `Dmon.Abstractions` — dmon's memory contracts (`ILongTermMemory`, `MemoryHit`, `MemoryScope`, …).
- [`ModelContextProtocol`](https://github.com/modelcontextprotocol/csharp-sdk) — the C# MCP SDK.
- `Microsoft.Extensions.AI.Abstractions` (pinned to dmon-core's `10.5.x` line) — `ChatMessage`, `IEmbeddingGenerator` contracts.
- `Microsoft.Extensions.{Options,Configuration,DependencyInjection,Logging}.Abstractions`.

## Status

The Meko long-term tier is implemented and live-verified against the real Meko service. The
`IMemory` facade, cross-tier fusion, and `AddDmonMemory()` wiring are tracked separately (they
combine this tier with dmon-core's short-term tier).

## License

See [`LICENSE`](./LICENSE).
