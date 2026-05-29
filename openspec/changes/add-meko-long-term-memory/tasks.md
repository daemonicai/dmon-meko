> **Before starting:** confirm dmon-core's `Dmon.Abstractions.Memory` contracts are reachable (see proposal prerequisite — `ProjectReference` to `../dmon-core/src/Dmon.Abstractions`). This change implements against them and MUST NOT redefine them. This is the first code in dmon-meko.

## 1. Solution bootstrap

- [x] 1.1 Create the dmon-meko solution and `src/Dmon.Memory.Meko` (project) + `test/Dmon.Memory.Meko.Tests`; add `Directory.Build.props` (net10.0, nullable enabled, `TreatWarningsAsErrors`, implicit usings) matching this repo's conventions
- [x] 1.2 Reference dmon-core's abstractions via `ProjectReference` to `../dmon-core/src/Dmon.Abstractions/Dmon.Abstractions.csproj` (TODO: swap to `PackageReference` once published); verify `dotnet build` and `dotnet format --verify-no-changes` are clean

## 2. MCP client + connection

- [x] 2.1 Add `ModelContextProtocol`; pin `Microsoft.Extensions.AI.Abstractions` to 10.5.2 (required by ModelContextProtocol 1.3.0; ABI-compatible with dmon-core's 10.5.1 — same `AssemblyVersion 10.5.0.0`)
- [x] 2.2 Build the connection: `HttpClientTransport` to `https://mcp.mekodata.ai/mcp` with `TransportMode = StreamableHttp` and `Authorization: Bearer <mko_tkn_…>` via `AdditionalHeaders`, key read from configuration; `McpClient.CreateAsync(transport)`
- [x] 2.3 Introduce the `IMekoToolInvoker` seam (single `CallToolAsync(tool, args, ct)` over `McpClient`) so the store is testable offline (D12)
- [x] 2.4 Bind ambient `MemoryContext` (datapack from config, `agent_id="dmon"`, `conversation_id`=session id); implement the single `MemoryScope`→Meko-`scope`-string mapping point; never set `run_id`

## 3. ILongTermMemory(Meko)

- [x] 3.1 Map all methods → `memory_*` tools (D7): `AddFactAsync`/`RecordAsync`→`memory_add`, `SearchAsync`→`memory_search`, `GetAsync`→`memory_get_by_id`, `ListAsync`→`memory_get_all`, `UpdateAsync`→`memory_update`, `DeleteAsync`→`memory_delete_by_id`, `FlushAsync`→`flush_pending_memory_candidates`; never call `conversation_*`/`knowledgebase_*`
- [x] 3.2 Defensive parsing (D13): map Meko's loose results into `MemoryHit` (`Source = LongTerm`, graph edges → `Relations`, extras → `Metadata` as `JsonElement`); missing/unexpected fields log + degrade, never throw into the caller
- [x] 3.3 Opt-in capture policy via `MekoLongTermOptions` (default conservative/none) gating `RecordAsync`→`memory_add(messages)`; `AddFactAsync` always asserts; a kept-nothing `RecordAsync` succeeds without throwing
- [x] 3.4 `FlushAsync` over `flush_pending_memory_candidates`; if it returns an agent-directive, act on it (scan recent turns + `memory_add`) — see assumed default 5.1
- [x] 3.5 Disabled/no-op `ILongTermMemory` (null object): writes no-op, reads return empty/null, no MCP call
- [x] 3.6 `AddMekoLongTermMemory(...)` `IServiceCollection` helper + options binding (the cross-tier `AddDmonMemory()` is the facade change)

## 4. Tests (mocked MCP via fake invoker)

- [x] 4.1 Tool mapping: each method calls the expected `memory_*` tool with the expected args (fake `IMekoToolInvoker`)
- [x] 4.2 Defensive parsing: well-formed → mapped `MemoryHit`(+`Relations`); partial/malformed → degrades (empty/recognized-only) and does not throw
- [x] 4.3 Opt-in capture: default keeps nothing (no `memory_add` on `RecordAsync`, call still succeeds); opted-in policy calls `memory_add`
- [x] 4.4 Disabled/no-op path: operations no-op, `SearchAsync`/`ListAsync` return empty, no invoker calls
- [x] 4.5 Scope mapping (`MemoryScope`→Meko `scope`) and that `run_id` is never set
- [ ] 4.6 Live smoke test against the real Meko endpoint with `mko_tkn_` credentials [needs human verification — requires real creds; cannot be settled by automated gates]

## 5. Assumed defaults (proceed now; verify async on Discord)

These are best-guess defaults so implementation is not blocked. Each is sealed behind `ILongTermMemory`, so a wrong guess is cheap to correct.

- [ ] 5.1 Proceed assuming `memory_add` is synchronous-enough and `FlushAsync` acts on the flush directive itself (scan recent turns + `memory_add`); do NOT rely on long-term read-your-writes. Verify Meko flush/`memory_add` semantics on Discord.
- [ ] 5.2 Proceed with `MemoryScope { Session, Agent, User, Shared }`, default `Agent`, mapped to Meko's `scope` string. Verify accepted `scope` values on Discord and adjust the single mapping point.
- [ ] 5.3 Decide the opt-in capture policy shape (per-turn vs. session-end, role/content filters, sampling). (Local choice — no external dependency.)
