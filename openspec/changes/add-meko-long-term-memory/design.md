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

### D7. Long-term = Meko over MCP, `memory_*` + `conversation_create`
`ILongTermMemory(Meko)` is an MCP client to `https://mcp.mekodata.ai/mcp` (Streamable HTTP, `mko_tkn_` bearer). Tool→method mapping: `AddFactAsync`→`memory_add(text)`; `RecordAsync`→`memory_add(messages)` (capture-gated); `SearchAsync`→`memory_search`; `GetAsync`→`memory_get_by_id`; `ListAsync`→`memory_get_all`; `UpdateAsync`→`memory_update`; `DeleteAsync`→`memory_delete_by_id`; `FlushAsync`→`flush_pending_memory_candidates`.

**Live-verified correction (2026-05-29, against the real server):** `memory_add`/`memory_search`/`memory_get_all` **require** a `conversation_id` that is a UUID from **`conversation_create`**. So the implementation additionally uses **`conversation_create`** solely to obtain that required id (created lazily once per dmon session and cached). It still does **not** use the other `conversation_*` tools (`conversation_add_message`/`get`/`list`/`update`/`delete` — verbatim history is short-term's job) or `knowledgebase_*`. `messages` and `metadata` are passed as **JSON strings** (not structured objects), per the tool schemas. All Meko coupling stays behind `ILongTermMemory`.

### D8. Opt-in capture
`RecordAsync` is gated by a capture policy (`MekoLongTermOptions`), defaulting to conservative (capture little or nothing) so recording a turn does not incur hosted distillation cost unless explicitly enabled. `RecordAsync` on a store that keeps nothing completes successfully (it must not throw to signal "nothing kept"). `AddFactAsync` always targets `memory_add(text)` (an explicit assertion, not gated by the capture policy).

### D9. Scope model — **revised against the live schema (2026-05-29)**
The original D9 (map `MemoryScope`→Meko `scope` string; never set `run_id`) was wrong. The live tool schemas show Meko's actual model:
- **`scope`** is a fixed required string — the schema literally says *"Pass `admin`."* It is **not** a memory-partition selector. The implementation sends `scope = "admin"` as a constant (one place; revisit if Meko widens it).
- **`agent_id`** selects the isolated pgvector collection + AGE graph → bound to `"dmon"`.
- **`conversation_id`** must be a UUID from `conversation_create`; it is used for trace nesting and does **not** filter results. The impl creates one lazily per dmon session and caches it.
- **`run_id`** is the actual filter — it restricts a search to memories tagged with that conversation. So `MemoryScope` maps onto **`run_id`**, not `scope`: `MemoryScope.Session` → set `run_id` = the dmon session id (the value carried as `MemoryContext.ConversationId`), scoping add/search to this conversation; **`Agent`/`User`/`Shared`** (the durable scopes) → **omit** `run_id` for cross-conversation recall. (`User`/`Shared` distinctions beyond "durable/global" await a Meko mechanism — see Open Questions.)
- **`datapack_id`** is an optional UUID; omit to use the caller's default datapack (only send it when a real datapack UUID is configured — a human-readable name is not accepted).

The dmon-core `MemoryContext` abstraction is unchanged (`DatapackId`/`AgentId`/`ConversationId` = dmon session id); the Meko impl translates: `agent_id`←`AgentId`, `run_id`←`ConversationId` (when session-scoped), the Meko `conversation_id`←a cached `conversation_create` UUID, `datapack_id`←`DatapackId` (if a UUID). The `MemoryScope`→`run_id` policy and the `scope="admin"` constant are each a single adjustable point.

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
- **M.E.AI ABI** → `ModelContextProtocol` depends on `Microsoft.Extensions.AI.Abstractions`. **Realized:** `ModelContextProtocol` 1.3.0 requires `≥ 10.5.2`, so dmon-meko explicitly pins `Microsoft.Extensions.AI.Abstractions` 10.5.2; dmon-core's `Dmon.Abstractions` pins `Microsoft.Extensions.AI` 10.5.1. Both expose `AssemblyVersion 10.5.0.0`, so the CLR binds them identically — no ABI mismatch (the hazard dmon-core's `fix-meai-abi-pin` addresses is a *minor/major* drift, not this patch bump). Mitigation/coordination: keep both repos on the same `10.5.x` minor; if dmon-core's pin moves to a different minor or below 10.5.2, re-check.
- **Cross-repo project reference** → couples the build to a sibling dmon-core checkout and is not cleanly packable. Mitigation: documented as temporary; swap to a `PackageReference` once `Dmon.Abstractions` is published.

## Migration Plan

Additive — dmon-meko's first code. Long-term memory is opt-in and disable-able (no-op store), so dmon runs without it. Rollout: (1) solution bootstrap + abstraction reference; (2) MCP client + connection + scope binding; (3) `ILongTermMemory` mapping + parsing + capture + flush + no-op; (4) tests (mocked invoker); (5) live smoke (HITL). Rollback = disable long-term / remove the DI registration.

## Open Questions

- **[assumed → verify on Discord]** Semantics of `flush_pending_memory_candidates` (agent-directive vs. server barrier) and whether `memory_add` is synchronous or candidate-queued. *Default (5.1):* treat `memory_add` as synchronous-enough; `FlushAsync` acts on the directive; do not rely on long-term read-your-writes.
- **[VERIFIED live, 2026-05-29 — supersedes the old 5.2 assumption]** Meko's `scope` is the fixed string `"admin"` (not a partition selector); partitioning is `agent_id` + `run_id` (+ `datapack_id`); `conversation_id` must come from `conversation_create`. `MemoryScope` now maps onto `run_id` (D9). Still open against Meko: whether there is any first-class mechanism for `User`/`Shared` (cross-agent / promotion) scopes beyond per-agent `run_id` filtering — for now they behave as "durable/global to the agent".
- **[VERIFY live during rework]** The exact **result envelope** of `memory_search`/`memory_add`/`memory_get_all` (field names for id/text/score/relations/metadata) — the diagnostic probe is being re-run with correct args to capture a successful response, and `MekoResultParser` (D13) will be fixed against that real shape rather than the assumed `results[]` shape.
- **[deferred — local choice]** Shape of the opt-in capture policy (per-turn vs. session-end, role/content filters, sampling). Decide during group 3; no external dependency.
- **[process]** When to switch the `Dmon.Abstractions` project reference to a published `PackageReference`.
