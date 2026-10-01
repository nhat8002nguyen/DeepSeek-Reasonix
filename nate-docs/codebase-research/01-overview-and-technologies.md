# 01 · Overview and technologies

## What Reasonix is

Reasonix is a **coding agent** — "a thin harness driving multiple models, with
all capabilities supplied by configuration and plugins" (`docs/SPEC.md`). It is
deliberately *not* a pile of hardcoded model or tool logic:

- The **core knows only interfaces** (`Provider`, `Tool`); concrete models and
  tools are resolved by name from registries and declared in config.
- It ships as a **single static Go binary** (`CGO_ENABLED=0`), cross-compiled
  for `darwin|linux|windows × amd64|arm64`.
- **Four frontends, one engine**: terminal TUI (`reasonix`), browser
  (`reasonix serve`), desktop app (`desktop/`, Electron + Go), and editor
  integration over ACP (`reasonix acp` + a VS Code extension). All four drive
  the same `control.Controller`.

## Technology stack

| Layer | Technology | Notes |
| --- | --- | --- |
| Language | Go 1.26 (module `reasonix`, `go.mod`) | Single binary; stdlib-first |
| TUI | `charmbracelet/bubbletea/v2`, `bubbles`, `lipgloss` | CLI chat interface |
| HTTP frontend | `net/http` + Server-Sent Events | `internal/serve` |
| Desktop shell | Electron + TypeScript (`desktop/`, separate Go module) | Talks to a Go service over host RPC |
| Editor protocol | ACP (Agent Client Protocol), JSON-RPC | `internal/acp`, `sdk/go` |
| MCP client | `modelcontextprotocol/go-sdk` | stdio / streamable-http / SSE transports |
| Config | TOML (`BurntSushi/toml`) | `reasonix.toml`, `config.toml`, `.mcp.json` |
| Storage | SQLite (`modernc.org/sqlite`, pure Go) + bbolt | projection/catalog DBs + recovery store |
| Diff/merge | `go-udiff` | tool `Previewer` and transcript diffs |
| Shell | `mvdan.cc/sh` (parser) | shell analysis / safety |
| Tree-sitter | `go-tree-sitter` (js/py/rust/ts grammars) | `code_index` symbol tool |

Notable non-dependencies: no gRPC, no heavyweight web framework, no OS package
managers at runtime — the binary is self-contained.

## Top-level layout

```
reasonix/
├── go.mod / go.sum            # module reasonix; Go 1.26 toolchain
├── Makefile                   # build / cross / vet / fmt / test / lint
├── reasonix.example.toml      # sample configuration
├── README.md / README.zh-CN.md
├── docs/                      # engineering spec + design/contract docs (mostly bilingual)
├── cmd/
│   ├── reasonix/              # CLI entry point; blank-imports built-in providers/tools
│   ├── reasonix-plugin-example/  # reference MCP stdio plugin (echo, wordcount)
│   └── ...                    # launcher, legacy migrator, protocol generators, e2ebench
├── internal/                  # all engine code (see package map below)
├── desktop/                   # Electron app + Go service (SEPARATE Go module)
├── npm/                       # npm distribution shim for the CLI
├── sdk/                       # Go SDK for building on top of the engine
└── tools/                     # repo tooling: repolint, pathidentityprobe, …
```

## Dependency direction

Acyclic and strictly enforced (by `tools/repolint/layers.go`):

```
cli → {agent, plugin, config} → {tool, provider}
```

- Built-in subpackages (`provider/openai`, `provider/anthropic`,
  `provider/responses`, `tool/builtin`) import their **parent** to self-register
  via `init()`; parents never import children.
- `control` depends on `agent` (and many support packages) but no frontend
  depends on it going the other way: `cli`, `serve`, and the desktop service all
  *use* `control`, and `control` knows nothing about any of them.
- `desktop/` is its own Go module and consumes the engine the same way an
  external consumer would.

## Design principles (from `docs/SPEC.md` §1)

1. **Config- and plugin-driven core.** No hardcoded `switch model`.
2. **Single static binary.** Cross-compile with one command.
3. **Lean dependencies.** Stdlib by default; a dependency must be pure-Go,
   lightweight, and not compromise single-binary distribution.
4. **Two extension tiers.** Compile-time built-ins (`init()` self-registration)
   and runtime external plugins (MCP subprocesses/servers).
5. **Interface-first & registry-based.** `Provider` and `Tool` are interfaces.
6. **Evolve, don't over-engineer.**

English is the primary language for all code and model-facing strings; UI text
is i18n'd (see `internal/i18n`).

## Package map

The `internal/` tree, grouped by responsibility. (A complete inventory lives in
the repo; this is the map you need to navigate.)

**The core (read these first)**
- `agent/` — the turn loop, session, coordinator, compaction, sub-agents,
  tool dispatch, recovery. The largest package.
- `provider/` — `Provider` interface, wire-neutral types, registry; subpackages
  `openai/`, `anthropic/`, `responses/` are the concrete adapters.
- `control/` — transport-agnostic `Controller` (session driver + event sink).
- `boot/` — assembles a `Controller` from config (models, tools, gate, prompt).

**Context / prompt assembly**
- `config/` — TOML loading and the config model.
- `memory/` — `REASONIX.md`/`AGENTS.md`/`CLAUDE.md` hierarchy + auto-recall.
- `skill/` — Markdown skill discovery and the skills index shown to the model.
- `instruction/` — standing-instruction resolution/rendering.
- `outputstyle/` — the output-style contract appended to the system prompt.
- `i18n/` — UI strings (English/Chinese); never model-visible.

**Capabilities / safety**
- `tool/` + `tool/builtin/` — `Tool` interface, registry, built-in tools.
- `permission/` — per-call policy (`allow/ask/deny`), `Gate`, presets.
- `sandbox/` — OS-level confinement (Seatbelt on macOS, bubblewrap on Linux).
- `hook/` — shell hooks (`PreToolUse`, `PostToolUse`, `PostLLMCall`, …).
- `guardian/` — optional safety reviewer for high-risk actions.
- `plugin/` — MCP client; `mcplaunch`/`mcpregistry`/`mcpinteraction`/`mcpdiag`
  support it.
- `capability/` — the `use_capability` capability catalog/ledger.
- `extension/` + `extensioncontract/` — Extension Protocol sidecars.

**State / persistence**
- `session/`, `sessioncatalog/`, `sessioncontent/`, `sessionexport/`,
  `sessioninbox/`, `sessiontitle/` — session catalog & storage services.
- `checkpoint/` — snapshot-based rewind (pre-edit file capture).
- `history/` + `historycatalog/` — BM25 history search.
- `store/`, `projectiondb/` — storage backends (SQLite/bbolt).
- `transcript/` — transcript representation & projection.
- `event/` — the typed event stream.

**Frontends / transport**
- `cli/` — subcommands, TUI, setup wizard.
- `serve/` — HTTP/SSE server.
- `acp/` — Agent Client Protocol server (editor integration).
- `remote/` — SSH transport for Remote-SSH.
- `browser/` — CDP/browser runtime glue.

**Everything else** — `billing`, `telemetry`, `crashreport`, `doctor`,
`recovery`, `migration`, `eval`, `ablation`, `goal`, `planmode`, `jobs`,
`workspacelease`, `fileops`, `lsp`, `websearch`, … each a coherent subsystem
with its own `doc.go`-style package comment.

Next: [02 · The agent engine core](02-agent-engine-core.md).
