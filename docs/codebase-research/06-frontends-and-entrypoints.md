# 06 · Frontends and entrypoints

One engine, four ways in. All frontends reduce to the same contract: **send
commands to a `control.Controller`, render its typed event stream.**

## `control.Controller` — the transport-agnostic driver (`internal/control`)

From its package doc (`controller.go`):

> A Controller owns the agent run loop and session lifecycle, takes commands
> (Send/Cancel/Approve/SetPlanMode/Compact/NewSession/…), and emits everything
> that happens — reasoning, tool calls, approvals, turn completion — as a typed
> event stream to a single `event.Sink`. … The Controller depends on no frontend.

It owns the `agent.Runner` (`*agent.Agent` or `*agent.Coordinator`), the
permission/sandbox wiring, goal/plan drivers, session lifecycle, and the
recovery machinery. Frontends (`cli`, `serve`, desktop service, `acp`) each
implement an `event.Sink` to render events and call the command methods. The
event types live in `internal/event`.

## The frontends

### CLI / TUI (`internal/cli`, `cmd/reasonix`)

- `cmd/reasonix/main.go` — entry; blank-imports built-in providers/tools so they
  self-register.
- `internal/cli` — subcommand routing, flag parsing, config assembly, exit
  codes, and the Bubble Tea chat TUI (`chat_tui.go`). Slash commands, `@`
  references, approvals, and modals are rendered here.
- Headless commands: `reasonix run`, `reasonix serve`, `reasonix acp`,
  `reasonix setup`, `reasonix subagent run|try`, `reasonix config`, etc.

### HTTP/SSE (`internal/serve`)

`serve.Server` wires a `Controller` to an HTTP surface. Its `Broadcaster` is the
same sink the controller was constructed with, so every event reaches SSE
clients; submissions come back as commands. Auth modes (`none|token|password`)
and the launch token are handled here (`auth.go`, `submission.go`,
`events_http.go`). `reasonix serve` is the browser/headless frontend.

### Desktop (`desktop/`, separate Go module)

An Electron shell (TypeScript) + a Go service. The Go service (`desktop/main.go`,
`desktop/host_rpc.go`) drives the same `boot`/`control` engine; the Electron
main process bridges the webview UI to it over a host RPC channel
(`desktop/electron/src/main/index.ts`). `desktop/README.md` documents the build
(`scripts/desktop-build.sh`, Node 24 + pnpm 10).

### Editor integration — ACP (`internal/acp`, `sdk/go`)

`reasonix acp` serves the Agent Client Protocol (JSON-RPC) so an editor
extension (the VS Code extension starts `reasonix acp`) can chat, send editor
context, approve tool calls, and manage sessions. `sdk/go` is the public Go SDK
for building on the engine; `sdk/go/README.md` documents it.

## End-to-end data flow

```
User types a message (TUI / webview / SSE POST / ACP)
  → frontend calls Controller.Submit/Run(input)
  → Controller → agent.Runner.Run
      → agent loop builds provider.Request (system prompt + history + tool schemas)
      → provider.Stream → chunk channel
      → events (Reasoning/Text/ToolDispatch/ToolResult/…) emitted to the Sink
  → frontend's Sink renders each event (and, for approvals, shows a prompt and
    calls back Controller.Approve)
  → turn completes → TurnDone event + persisted session
```

Key point: **cancellation, approval, plan mode, compaction, and session
lifecycle all live in `control`/`agent`**, so none of the four frontends
re-implements them — which is exactly what keeps the surface (and its bugs)
small and consistent.

## Where a change lands (quick map)

| You want to change… | Look in… |
| --- | --- |
| A built-in tool's behavior/schema | `internal/tool/builtin/*.go` |
| A model backend's wire format | `internal/provider/{openai,anthropic,responses}/*.go` |
| The agent loop / turn logic | `internal/agent/{agent,run_loop,execute_one,context_manager}.go` |
| The system prompt / memory / skills prefix | `internal/boot/`, `internal/memory/`, `internal/skill/`, `internal/outputstyle/` |
| Permission/sandbox policy | `internal/permission/`, `internal/sandbox/` |
| Session/checkpoint/history | `internal/agent/session_*.go`, `internal/checkpoint/`, `internal/history/` |
| TUI behavior | `internal/cli/` |
| Browser/SSE surface | `internal/serve/` |
| Desktop UI/shell | `desktop/` |

Next: [07 · contributing](07-contributing.md).
