# 04 · Tools, permissions, sandbox, and MCP

The model is given a set of **tools**; when it calls one, the engine checks
**policy** (permission) and **enforcement** (sandbox), runs it, and feeds the
result back. This document covers the whole capability/safety stack.

## Tool system (`internal/tool`)

- **`Tool` interface** (`tool.go`): `Name`, `Description`, `Schema`,
  `Execute(ctx, args) (string, error)`, `ReadOnly`. Optional interfaces:
  `ContextualTool`, `Previewer` (a dry-run `diff.Change` without touching disk),
  `ImageTool`, `PlanModeClassifier`, and MCP metadata interfaces.
- **Built-ins** self-register into a process-global set via
  `tool.RegisterBuiltin` in `init()` (`internal/tool/builtin/`); `main`
  blank-imports `builtin`.
- **`Registry`** is the per-run set the agent sees: enabled built-ins (filtered
  by config) **plus** plugin/MCP tools. `Add` canonicalizes each schema once;
  `Schemas` produces the provider-visible tool list; `ResolveCall` resolves by
  exact name / shell alias / MCP alias and refuses ambiguous calls.
- **Schema canonicalization** (`internal/provider/schema_canonicalize.go`)
  forces `type:"object"`, `properties:{}`, `required:[]` and sorts `required`,
  so the tool prefix is deterministic. `docs/TOOL_CONTRACT.md` is the built-in
  contract, backed by tests that compare the documented surface against the
  canonical schema path.

### Built-in tools (representative)

File/workspace: `read_file`, `write_file`, `edit_file`, `multi_edit`,
`move_file`, `notebook_edit`, `delete_range`, `delete_symbol`, `ls`, `glob`,
`grep`, `code_index` (tree-sitter symbol index), `present`, `view_image`.
Shell: `bash` (renamed `pwsh` on PowerShell hosts). Network:
`web_fetch` (SSRF-guarded). Memory/history: `remember`, `forget`, `memory`,
`history`. Delegation: `task`, `fleet`, `parallel_tasks`, `read_only_task`,
`run_skill`/`read_only_skill`, `complete_subtask`, `todo_write`, `ask`,
`complete_step`, `use_capability`. The exact set and schemas are in
`internal/tool/builtin/`; `complete_step` and a receipt tool are hidden from
model discovery.

## Permissions (`internal/permission`) — policy

A pure, I/O-free `Policy` evaluates static rules; a `Gate` wraps it with an
optional interactive `Approver`.

```go
type Decision int            // Allow | Ask | Deny
type Policy struct { Mode Decision; Allow, Ask, Deny []Rule }
func (p Policy) Decide(toolName string, readOnly bool, args json.RawMessage) Decision
```

- **Rule syntax**: `Tool` (whole family) or `Tool(specifier)` — e.g.
  `Bash(go test:*)`, `Edit(docs/**)`, `Bash(rm -rf*)`. The subject is extracted
  from known args keys (`command`, `path`/`file_path`, `pattern`).
- **Precedence**: `deny > ask > allow > fallback` (read-only tools fall back to
  `Allow`; writers to the preset `Mode`).
- **Presets**: `read-only`, `workspace-write` (default), `danger-full-access`.
  These are *separate* from Plan Mode and from the collaboration mode (normal /
  plan / goal).
- A non-interactive run that needs a prompt **fails closed**. A `Deny` is a
  hard block in every preset.

## Sandbox (`internal/sandbox`) — enforcement

Permissions say *may I*; the sandbox says *can I physically*. Two layers:

- **File-writer confinement** (`workspace_root`, `allow_write`, `forbid_read`):
  `write_file`/`edit_file`/`multi_edit`/`move_file` resolve targets to
  absolute, symlink-free paths and refuse anything outside the roots.
- **Bash OS jail** (`bash = "enforce"`): Seatbelt on macOS, bubblewrap on
  Linux — writes confined to the same roots plus toolchain temp/cache dirs,
  `forbid_read` honored, network allowed only when `network = true`. When no
  sandbox backend is available it **fails closed** rather than running
  unconfined (unless explicitly `off`). Windows has no OS-level Bash sandbox:
  effective mode is `off` (in-process file tools still enforce roots).

## Hooks (`internal/hook`)

Shell hooks fire around the lifecycle: `PreToolUse` (may block a call after
permission, before execution), `PostToolUse`/`PostToolUseFailure`,
`PostLLMCall` (rewrites the reasoning block), plus agent/extension lifecycle
events.

## Guardian (`internal/guardian`)

An optional safety reviewer that can veto high-risk actions (e.g. destructive
shell) by asking a model for a structured allow/deny verdict; it has a circuit
breaker (consecutive denials → interrupt) and fails closed on unparseable
output. It answers *safety*, not the user's `ask` questions.

## MCP / plugins (`internal/plugin`)

External capabilities come from MCP servers (config `[[plugins]]` or a
`.mcp.json`):

- Transports: `stdio` (subprocess), `http` (streamable HTTP, with OAuth
  PKCE when no static key), `sse` (legacy). Protocol negotiation is delegated
  to the official MCP Go SDK; one concurrency-safe session per server.
- Lifecycle: `initialize` → `notifications/initialized` → `tools/list`;
  invocation via `tools/call`.
- Each remote tool is adapted to `Tool` and namespaced `mcp__<server>__<tool>`
  (matching Claude Code). `readOnlyHint` maps to `ReadOnly()` (default false —
  remote tools are opaque). Installation is the trust decision; per-call
  `deny` rules and the process sandbox remain the containment boundary.
- `prompts/list`+`prompts/get` surface as `/mcp__<server>__<prompt>`; resources
  as `@<server>:<uri>`.
- `internal/mcplaunch`, `mcpregistry`, `mcpinteraction`, `mcpdiag` handle
  launch, registry, OAuth/interaction, and diagnostics respectively.

## The `use_capability` proxy (`internal/capability`, `internal/agent/usecapability*.go`)

A stable built-in proxy the model uses to discover and call deferred
capabilities (MCP tools, sub-agent/skill tools, memory, sessions) *without* the
dynamic `mcp__*` schemas entering the stable tool prefix. It re-checks
enablement/authorization/connection identity immediately before dispatch. This
is what keeps the tool prefix byte-stable while still exposing a growing
capability surface.

## Extensions (`internal/extension`, `extensioncontract`)

Extension Protocol sidecars (separate processes) can contribute **providers**,
**structured UI**, and intercept **runtime events** (e.g. `agent.before_start`,
`provider.response`, `tool.after`). The wire protocol lives in
`docs/EXTENSION_PROTOCOL.md`.

## The execute-a-tool flow (recap)

```
tool call received
  → Registry.ResolveCall        (unknown/ambiguous → blocked)
  → Gate.Check (permission)     (allow | ask user | deny)
  → PreToolUse hook             (may block)
  → sandbox stamp (preset + write roots) → Execute  (confinement inside the tool)
  → PostToolUse / PostToolUseFailure
  → result size-bounded and appended as role=tool
```

Next: [05 · sessions and persistence](05-sessions-and-persistence.md).
