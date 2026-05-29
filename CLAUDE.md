# dmon-meko

`dmon-meko` is a C#/.NET (`net10.0`) **memory library** for **dmon** (the .NET-native coding agent),
shipped as NuGet package(s). It provides dmon's first-class **memory abstraction** and its durable,
**Meko-backed long-term memory**: the `IMemory` / `IShortTermMemory` / `ILongTermMemory` contracts, a
per-session **hybrid semantic index** (short-term), a Meko-driven **long-term store**, the scope model,
and the consistency/flush semantics. It owns memory mechanics; the consumer (dmon) owns sessions,
agents, and conversation semantics.

It is developed alongside **`dmon-core`** (`../dmon-core`, an additional working directory) — that repo
holds the agent core, `ISessionStore`, and the binding ADRs. `dmon-meko` evolves `ISessionStore` into a
searchable short-term tier and adds the long-term tier on top.

Key technical shape (from the active change's `design.md`):
- **Short-term**: per-session `index.db` via raw `Microsoft.Data.Sqlite` + the `sqlite-vec` loadable
  extension + FTS5, fused with Reciprocal Rank Fusion (RRF). A **derived, rebuildable projection** of the
  canonical JSONL. Embeddings from a local model (LlamaSharp) behind `IEmbeddingGenerator<string, Embedding<float>>`.
- **Long-term**: a Meko-backed `ILongTermMemory` driving the Meko MCP server (`memory_*` tools) over
  Streamable HTTP. **Opt-in capture** — the facade always calls `LongTerm.RecordAsync`; the implementation
  decides what (if anything) to persist. dmon must function with long-term memory **disabled**.
- **Native footprint**: two native loadables (`sqlite-vec`, `llama.cpp` via LlamaSharp) enter the process —
  mind per-RID packaging and (future) trimming/AOT implications.

Spec-driven development is managed with **OpenSpec** (`openspec/`, schema `spec-driven`). All feature
work flows through a change in `openspec/changes/`.

### Where to look for historical context on a shipped change

The OpenSpec archive (`openspec/changes/archive/YYYY-MM-DD-<name>/`) contains the spec deltas for every
shipped change. Alongside `proposal.md` / `design.md` / `specs/**/*.md` / `tasks.md` there may also be a
**`DEVLOG.md`** — a working note kept by the orchestrator while the change was being applied. It captures
the *narrative* the spec files don't: per-section status, decisions made under uncertainty, deviations
from the original plan, bugs surfaced during implementation, and human-in-the-loop verifications. When a
new session needs context on *how* a prior change was built (not just *what* it specified), read the
archived `DEVLOG.md` for that change. Newly-active changes also keep a `DEVLOG.md` inside the change
directory while in-flight; when the change archives, the DEVLOG moves with it.

### Commands

> The solution itself is created by section 1 of the first change. Before that, only `openspec` commands apply.

- Build: `dotnet build` — must be clean (analyzers run as **warnings-as-errors**; nullable enabled).
- Test: `dotnet test` — all green.
- Format: `dotnet format --verify-no-changes` — clean.
- Validate a change: `openspec validate <change-name> --strict`.
- List changes: `openspec list` (or the directories under `openspec/changes/`, excluding `archive/`).

---

# OpenSpec Apply Workflow

**This section is authoritative.** `/opsx:apply` is the entry point for implementing a change; if the
skill's behavior ever conflicts with what's written here, **follow this document**.

## Roles — the main thread never writes feature code

- **Orchestrator** = the main thread (you). You read specs, select work, brief agents, run the gates,
  tick boxes, and commit. **You do not implement feature code directly.**
- **`worker`** agent — implements the tasks.
- **`reviewer`** agent — audits the worker's diff.

Both agents are defined for this repo. Delegate; don't shortcut by writing the implementation yourself.

## 1. Select the change

1. List active changes = directories in `openspec/changes/` **excluding `archive/`**.
2. **Always ask the user which change to apply**, even when there is exactly one. If there are none,
   say so and stop.
3. Resume point = the **first unticked `- [ ]` task** in that change's `tasks.md`.

## 2. Pre-flight (orchestrator, before any section)

1. Read `proposal.md`, `design.md`, and the relevant `specs/<capability>/spec.md` for the section(s)
   you're about to work. The binding decisions live in the change's `design.md` **`## Decisions`** (there
   are no ADRs or `coding-agent-brief.md` in this repo).
2. **Working tree must be clean** (`git status`). If it's dirty, stop and ask.
3. **Change must validate**: `openspec validate <change-name> --strict`. If it doesn't, stop and ask.
4. **Be on the change branch** `change/<change-name>`. Create it from `main` if missing:
   `git switch -c change/<change-name>`.

## 3. Implement — section by section

The unit of work is a **`## N.` section**. Walk sections in order from the resume point. For each:

1. **Brief the worker.** Hand it: the section's tasks (`N.1`…`N.k`), the relevant spec excerpts, the
   design decisions/constraints that bind them, and the done-gates below. The worker should not need to
   go hunting — give it what it needs to stay focused.
2. **Worker implements the whole section.** If a section is large or complex (e.g. the hybrid search
   index or the Meko MCP client), split it into sub-chunks across multiple `worker` calls — but it remains
   **one commit at section end**.
3. **Audit.** Spawn `reviewer` on the section diff (correctness, design-decision compliance, OpenSpec
   scope, C# idiom, agentic-AI design quality).
4. **Review loop.** Feed the reviewer's findings back to the `worker`; worker fixes; `reviewer`
   re-audits. **Repeat until the reviewer signs off.**
5. **Gates — all four must pass before ticking any box:**
   - `dotnet build` clean (no errors; analyzers/warnings-as-errors clean)
   - `dotnet test` green — new tests for the section **and** all existing tests
   - `openspec validate <change-name> --strict`
   - `dotnet format --verify-no-changes` clean
   If a gate fails, it's back to step 4, not a commit.
6. **Tick the boxes.** Mark every `- [x] N.M` in the section in `tasks.md`.
7. **Commit — one conventional commit per section:**
   ```
   feat(<change-name>): <section title> (section N)

   - N.1 <task summary>
   - N.2 <task summary>
   ...

   Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>
   ```

## 4. Stop and ask — do not push on

Stop **immediately** and ask the user (do not improvise a fix) when:

- a spec/design is **ambiguous**, or two specs **contradict** each other;
- doing the task properly needs changes **outside this change's scope** (its proposal/specs);
- a task is **blocked by an unresolved Open Question** in `design.md`;
- implementation or tests reveal the **spec itself is wrong** (not just the code);
- a task **requires human-in-the-loop verification** that can't be settled by automated gates — e.g.
  confirming behavior that depends on a native loadable (`sqlite-vec`, LlamaSharp/`llama.cpp`), the first-run
  embedding-model download, real semantic-search relevance/RRF fusion quality, or a live round-trip against
  the Meko MCP endpoint. Implement and self-test as far as possible, then hand the user a precise,
  copy-pasteable way to verify (exact command, what to do, what they should see) and **wait for their
  confirmation before ticking that task**.

**On stopping mid-section:** leave the WIP **uncommitted**, do **not** tick the section, do **not**
revert. Report the **exact task (`N.M`)** that stopped you and why. The WIP stays in the working tree
for the user to inspect.

## 5. Done

When every task in the change is ticked and the final review is clean:

1. Report status: sections completed, commits made, test summary.
2. **Propose archiving** — offer to run `/opsx:archive` and **wait for the user's confirmation**.
   Do not archive automatically.
