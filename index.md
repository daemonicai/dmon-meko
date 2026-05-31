---
layout: home
title: dmon-meko
tagline: long-term memory · .NET 10
lead: >-
  dmon's durable, cross-session memory tier — an ILongTermMemory implementation
  driven over the Meko agent-native data layer (YugabyteDB) via its MCP server.
repo: daemonicai/dmon-meko
ctas:
  - { label: "View on GitHub →", href: "https://github.com/daemonicai/dmon-meko", primary: true }
  - { label: "dmon — the agent core", href: "https://daemonicai.github.io/dmon-core/" }
features:
  - icon: "❖"
    title: Durable cross-session recall
    body: Implements ILongTermMemory by calling Meko's memory_* MCP tools over Streamable HTTP. The rest of dmon never sees an McpClient.
  - icon: "⊞"
    title: Clean tier split
    body: Long-term lives here; short-term (sqlite-vec + FTS5) lives in dmon-core. The two meet behind the IMemory facade.
  - icon: "✋"
    title: Opt-in capture
    body: RecordAsync is gated by MekoCaptureMode (None by default), so recording a turn never silently incurs hosted distillation cost.
  - icon: "⟳"
    title: Eventually consistent
    body: Not read-your-writes — freshly added memories may not be immediately searchable while Meko distils and indexes asynchronously.
  - icon: "○"
    title: Disabled-tier fallback
    body: With no API key, registration falls back to a no-op store — the MCP client is never on the critical path.
  - icon: "🧪"
    title: Offline tests
    body: Unit tests use a hand-rolled fake MCP invoker; an environment-gated, self-cleaning live smoke test hits the real service.
---

## How it works

`MekoLongTermMemory` implements `ILongTermMemory` by calling Meko's `memory_*` MCP
tools over **Streamable HTTP**. All Meko/MCP coupling is confined behind the interface.

| `ILongTermMemory` | Meko tool |
|-------------------|-----------|
| `AddFactAsync(text)` | `memory_add` (text) |
| `RecordAsync(turns)` | `memory_add` (messages) — gated by opt-in capture |
| `SearchAsync(query)` | `memory_search` |
| `GetAsync(id)` | `memory_get_by_id` |
| `ListAsync()` | `memory_get_all` |
| `UpdateAsync(id, text)` | `memory_update` |
| `DeleteAsync(id)` | `memory_delete_by_id` |

## Configuration & DI

```csharp
services.AddMekoLongTermMemory(options =>
{
    options.ApiKey   = configuration["Meko:ApiKey"];   // mko_tkn_...
    options.Endpoint = "https://mcp.mekodata.ai/mcp";  // default
    options.CaptureMode = MekoCaptureMode.None;        // opt-in
});
```

If `ApiKey` is empty, registration falls back to the disabled (no-op) store. The
memory contracts come from the `Dmon.Abstractions` NuGet package, so a clean
checkout builds without dmon-core present.

## Status

The Meko long-term tier is implemented and live-verified against the real Meko
service. See the [README](https://github.com/daemonicai/dmon-meko#readme) for the
full behaviour notes and the spec-driven change history.
