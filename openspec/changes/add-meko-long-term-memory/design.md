## Context

dmon-meko is a new, empty repo (net10.0, nullable, warnings-as-errors per its `CLAUDE.md`). This change introduces its first code. The memory abstraction was designed in the `add-memory-abstraction` umbrella; the contracts live in dmon-core's `Dmon.Abstractions.Memory` and are consumed here by project reference (see proposal prerequisite).

Meko (YugabyteDB agent-native data layer, early access) is consumed **only via MCP** — no REST API or SDK exists today (the programmatic "level 3" path is documented as under development). Its memory operations are MCP tools (`memory_add`, `memory_search`, `memory_get_by_id`, `memory_get_all`, `memory_update`, `memory_delete_by_id`, `flush_pending_memory_candidates`). `memory_add` is an *extraction pipeline*: it distills entities/relationships from text/messages via an LLM and stores vectors (pgvector) + graph edges (Meko AGE). Meko's memory is built on **mem0**, whose `user_id`/`agent_id`/`run_id` scoping vocabulary informs the scope model.

The verified MCP C# SDK shape (`ModelContextProtocol`): `new HttpClientTransport(new HttpClientTransportOptions { Endpoint, TransportMode = HttpTransportMode.StreamableHttp, AdditionalHeaders })` → `await McpClient.CreateAsync(transport)` → `ListToolsAsync()` / `CallToolAsync(name, Dictionary<string, object?> args, cancellationToken)`; results expose typed content blocks (e.g. `TextContentBlock`).

## Goals / Non-Goals

**Goals:**
- A Meko-backed `ILongTermMemory` whose surface is informed by Meko's `memory_*` tools, with opt-in capture.
- All Meko/MCP coupling confined behind `ILongTermMemory`; defensive parsing of early-access, loosely-structured output.
- A disabled/no-op path so dmon runs with long-term memory off.
- Unit-testability without a live Meko server.

**Non-Goals:**
- The `IMemory` facade and cross-tier RRF fusion + `AddDmonMemory()` (a later dmon-core change).
- The short-term tier (dmon-core §2/§3).
- Meko's `conversation_*` (verbatim history is short-term's job) and `knowledgebase_*` tools.
- Multi-persona `agent_id`, cross-scope queries, and scope-widening/promotion (future; the scope seam is designed in).

## Decisions

### D7. Long-term = Meko over MCP, `memory_*` only
`ILongTermMemory(Meko)` is an MCP client to `https://mcp.mekodata.ai/mcp` (Streamable HTTP, `mko_tkn_` bearer). Tool→method mapping: `AddFactAsync`→`memory_add(text)`; `RecordAsync`→`memory_add(messages)` (capture-gated); `SearchAsync`→`memory_search`; `GetAsync`→`memory_get_by_id`; `ListAsync`→`memory_get_all`; `UpdateAsync`→`memory_update`; `DeleteAsync`→`memory_delete_by_id`; `FlushAsync`→`flush_pending_memory_candidates`. Ignore `conversation_*`/`knowledgebase_*`.

### D8. Opt-in capture
`RecordAsync` is gated by a capture policy (`MekoLongTermOptions`), defaulting to conservative (capture little or nothing) so recording a turn does not incur hosted distillation cost unless explicitly enabled. `RecordAsync` on a store that keeps nothing completes successfully (it must not throw to signal "nothing kept"). `AddFactAsync` always targets `memory_add(text)` (an explicit assertion, not gated by the capture policy).

### D9. Scope model
Bind once per session (ambient `MemoryContext`): `datapack_id` (config), `agent_id` (`"dmon"`), `conversation_id` (session id). Carry `MemoryScope` per call (default `Agent`). **Do not** set `run_id` (mem0's auto-expiring ephemeral tier — owned by short-term). A single `MemoryScope`→Meko-`scope`-string mapping point is the only place to adjust when Meko's accepted values are confirmed.

### D10. `MemoryHit` is the result currency
Parse Meko results into `MemoryHit { Id, Text, Source = LongTerm, Score, Metadata?, Relations? }`. `Metadata` is `IReadOnlyDictionary<string, JsonElement>?` (Meko returns nested/loose JSON — no lossy stringification). `Relations` (Meko AGE graph edges) is populated for long-term and is the field short-term leaves null.

### D11. Conventions
`Task<T>` for I/O, `ValueTask` for `FlushAsync`, `CancellationToken = default` everywhere, `IReadOnlyList<T>` collections, nullable optional, `record` DTOs. A focused `AddMekoLongTermMemory(...)` DI helper + options binding; the cross-tier `AddDmonMemory()` is the facade change's job.

### D12. Testability seam — `IMekoToolInvoker`
The SDK's `McpClient` is a concrete connection, awkward to mock and impossible to exercise offline. Introduce a thin internal seam — `IMekoToolInvoker` with a single `CallToolAsync(string tool, IReadOnlyDictionary<string, object?> args, CancellationToken)` returning the raw tool result — implemented for real over `McpClient`, and faked in tests. `ILongTermMemory(Meko)` depends on the seam, not on `McpClient`. This matches dmon-core's hand-rolled-fakes test style (no mocking framework).
- *Why:* keeps tool mapping + defensive parsing fully unit-testable; isolates the preview SDK behind one swappable point.

### D13. Defensive parsing of early-access output
Meko's tool outputs are tuned for LLM consumption (loose structure). Parse tolerantly: never assume a field exists; treat missing/extra fields as non-fatal; map only what is recognized into `MemoryHit`; surface parse failures via logging and degrade (empty result / no-op) rather than throwing into the agent loop. All Meko coupling stays behind `ILongTermMemory`.

## Risks / Trade-offs

- **`flush_pending_memory_candidates` may be an agent-directive, not a server barrier** (its description: *"return a directive instructing the agent to scan recent user turns … and call `memory_add`"*) → `FlushAsync(Meko)` may not guarantee read-your-writes. Mitigation: have the adapter *act on* the directive (scan recent turns + `memory_add`); treat long-term flush as best-effort. Verify on Discord (assumed default 5.1).
- **Long-term eventual consistency** (distillation lag, async candidates) → freshly captured material may not be immediately recallable. Mitigation: the contract does not promise long-term read-your-writes; short-term covers same-session recency (via the facade, later).
- **Meko is early-access** → tool schemas/semantics may shift; output is loosely structured. Mitigation: D12 seam + D13 defensive parsing; confine all coupling behind `ILongTermMemory`.
- **M.E.AI ABI** → `ModelContextProtocol` depends on `Microsoft.Extensions.AI.Abstractions`; a version drift from dmon-core's 10.6.0 pin risks the ABI mismatch that dmon-core's `fix-meai-abi-pin` addresses. Mitigation: pin/align the transitive M.E.AI version to 10.6.0.
- **Cross-repo project reference** → couples the build to a sibling dmon-core checkout and is not cleanly packable. Mitigation: documented as temporary; swap to a `PackageReference` once `Dmon.Abstractions` is published.

## Migration Plan

Additive — dmon-meko's first code. Long-term memory is opt-in and disable-able (no-op store), so dmon runs without it. Rollout: (1) solution bootstrap + abstraction reference; (2) MCP client + connection + scope binding; (3) `ILongTermMemory` mapping + parsing + capture + flush + no-op; (4) tests (mocked invoker); (5) live smoke (HITL). Rollback = disable long-term / remove the DI registration.

## Open Questions

- **[assumed → verify on Discord]** Semantics of `flush_pending_memory_candidates` (agent-directive vs. server barrier) and whether `memory_add` is synchronous or candidate-queued. *Default (5.1):* treat `memory_add` as synchronous-enough; `FlushAsync` acts on the directive; do not rely on long-term read-your-writes.
- **[assumed → verify on Discord]** Accepted values for Meko's `scope` input and the mapping from mem0's id-filter model. *Default (5.2):* `MemoryScope { Session, Agent, User, Shared }`, default `Agent`; adjust the single mapping point when confirmed.
- **[deferred — local choice]** Shape of the opt-in capture policy (per-turn vs. session-end, role/content filters, sampling). Decide during group 3; no external dependency.
- **[process]** When to switch the `Dmon.Abstractions` project reference to a published `PackageReference`.
