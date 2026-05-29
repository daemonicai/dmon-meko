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
- [x] 4.6 Live smoke test against the real Meko endpoint with `mko_tkn_` credentials [needs human verification — requires real creds; cannot be settled by automated gates]

## 5. Assumed defaults (proceed now; verify async on Discord)

These are best-guess defaults so implementation is not blocked. Each is sealed behind `ILongTermMemory`, so a wrong guess is cheap to correct.

- [ ] 5.1 **Partially verified live (2026-05-29):** `memory_add` is synchronous-enough (returns the created memory + id immediately) and long-term is NOT read-your-writes (search/get_all returned empty right after add — indexing lag confirmed). **Still open:** the exact behavior of `flush_pending_memory_candidates` (server barrier vs. agent-directive) was not exercised live; `FlushAsync` remains best-effort. Confirm flush semantics (Meko Discord or a dedicated flush probe) before relying on it.
- [x] 5.2 **Verified live (2026-05-29):** Meko's `scope` is the fixed string `"admin"`; partitioning is via `agent_id` + `run_id`. `MemoryScope` now maps onto `run_id` (Session→run_id; durable omit), not onto `scope`. Mapping adjusted accordingly (see section 6 + D9).
- [x] 5.3 Decided + implemented: `MekoCaptureMode { None, EveryTurn }`, default `None` (conservative). (Local choice.)

## 6. Live-verified Meko schema alignment (rework after 4.6 probe)

The live diagnostic probe (2026-05-29) showed the assumed arg/scope model was wrong. Rework the implementation to the real schema (see revised design D7/D9):

- [x] 6.1 Send `scope = "admin"` (fixed required constant) on every `memory_*` / `conversation_create` call — replace the `MemoryScope`→scope-string mapping in `MekoScopeMapping`
- [x] 6.2 Add a `conversation_create` flow: lazily create one Meko conversation UUID per dmon session, cache it, and pass it as the required `conversation_id` on `memory_*` calls (lift the "ignore conversation_*" ban for `conversation_create` only)
- [x] 6.3 Map `MemoryScope` onto `run_id`: `Session` → `run_id` = dmon session id; `Agent`/`User`/`Shared` → omit `run_id` (cross-conversation). Single adjustable policy point
- [x] 6.4 Pass `messages` and `metadata` as JSON **strings** (serialize), per the tool schemas; send `agent_id` from context; send `datapack_id` only when a real UUID is configured
- [x] 6.5 Fix `MekoResultParser` against the **real** `memory_search`/`memory_add`/`memory_get_all` response envelope (captured via the corrected diagnostic probe), not the assumed `results[]` shape
- [x] 6.6 Update the fake-invoker unit tests (4.1–4.5) to assert the corrected args (scope=admin, conversation_id present, run_id policy, JSON-string messages/metadata) and the real result shape
- [x] 6.7 Re-run the live smoke (4.6) green: `conversation_create` → `memory_add` → `memory_search` recalls the marker → cleanup
