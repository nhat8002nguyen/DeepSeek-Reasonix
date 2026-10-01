# Reasonix codebase research

This directory is a study guide to the **Reasonix** codebase — a self-contained
coding-agent engine written in Go. It is written for a reader who wants to

1. understand the **technologies** and **architecture**,
2. reach the **agent engine core** (where the engine talks to LLM models) and
   the functionality around it,
3. learn **how to read the codebase comprehensively**,
4. contribute to the project, and
5. carry the design ideas over to building **agentic tools beyond a coding
   agent**.

Everything here is derived from the code and the project's own engineering
contract (`docs/SPEC.md`); file paths are real and current as of this branch.

## How to read the codebase (recommended order)

Reasonix is large (~120 packages under `internal/`), but it is layered and
almost every subsystem funnels into one place. Read in this order; each step
builds on the last:

1. **`README.md`** (project root) — what the product is and its feature list.
2. **`docs/SPEC.md`** — the *engineering contract*. It is unusually good: it
   states the design principles, the package layout, every core abstraction,
   and the invariants the code must honor. Read §1–§3 first, then come back to
   §4–§9 as you meet each topic in code.
3. **`internal/provider/provider.go`** — the `Provider` interface and the
   provider-visible data types (`Message`, `Request`, `ToolSchema`, `Chunk`,
   `Usage`). This is the "wire-neutral" vocabulary the whole engine speaks.
4. **`internal/provider/openai/openai.go`** — the concrete LLM interaction:
   how a `Request` becomes an HTTP body, how an SSE stream becomes `Chunk`s,
   and how reasoning/tool-calls/usage are parsed. This is the single best file
   for "how we talk to the model".
5. **`internal/agent/agent.go` → `run_loop.go` → `stream_sink.go`** — the
   agent turn loop: build request → stream → commit → execute tools or finish.
6. **`internal/agent/coordinator.go`** — the two-model planner/executor split.
7. **`internal/control/controller.go`** — the transport-agnostic driver that
   owns the loop and emits a typed event stream.
8. **`internal/boot/boot.go`** — how config becomes a ready-to-drive
   `Controller` (models, tools, permission gate, executor, system prompt).
9. **`internal/tool/tool.go` + `internal/tool/builtin/`** — the tool system.
10. **`internal/permission/` + `internal/sandbox/`** — policy vs. enforcement.
11. **`internal/plugin/`** — MCP servers as tools.
12. **`internal/agent/session_dag.go` + `internal/checkpoint/`** — persistence
    and rewind.

A faster route for a focused task: `grep` for the behavior you care about, then
open the owning `doc.go` / package comment and the `*_test.go` files — Reasonix
tests are often executable specifications (e.g. `profile_boundary_test.go`,
`tool_contract_surface_test.go`).

## Document index

| File | What it covers |
| --- | --- |
| [`01-overview-and-technologies.md`](01-overview-and-technologies.md) | Tech stack, top-level layout, dependency direction, design principles, package map |
| [`02-agent-engine-core.md`](02-agent-engine-core.md) | **The core**: `Provider`/`Tool` interfaces, the turn loop, streaming, reasoning replay, tool execution, compaction, two-model mode |
| [`03-context-and-cache.md`](03-context-and-cache.md) | How the system prompt / memory / skills / tool schemas are assembled, and the prompt-cache invariants |
| [`04-tools-permissions-sandbox.md`](04-tools-permissions-sandbox.md) | Tool registry, built-ins, permissions, OS sandbox, hooks, MCP, extensions |
| [`05-sessions-and-persistence.md`](05-sessions-and-persistence.md) | Session DAG (`.events.jsonl`), checkpoint/rewind, history search, storage |
| [`06-frontends-and-entrypoints.md`](06-frontends-and-entrypoints.md) | CLI/TUI, HTTP/SSE, desktop, ACP, and the end-to-end data flow |
| [`07-contributing.md`](07-contributing.md) | Dev environment, build/test/lint, PR metadata, where to start, "toward Cursor parity" |
| [`08-agentic-design-patterns.md`](08-agentic-design-patterns.md) | Reusable patterns for building any agentic tool, extracted from this codebase |

## The 30-second mental model

> One transport-agnostic `control.Controller` drives an `internal/agent` turn
> loop. Each loop iteration builds a cache-stable `provider.Request`, streams it
> through a `provider.Provider` (OpenAI/Anthropic/Responses adapters), and
> either commits a final answer or executes the tool calls it received — gated
> by `permission` (policy) and `sandbox` (enforcement) — then feeds the results
> back and repeats. Everything else (CLI, HTTP/SSE, desktop, ACP) is a frontend
> that sends commands to the Controller and renders its typed event stream.

The single most important product invariant, repeated throughout the code, is
**prompt-cache stability**: the system prefix and tool schemas stay byte-identical
across turns; changing state rides in per-turn context. Understand that and most
design decisions click into place.
