# 07 · Contributing to Reasonix

This doc is the practical, project-specific guide: environment, checks, PR
metadata, and where to start. The authoritative source is `CONTRIBUTING.md`
(root) and `docs/SPEC.md`; this distills them for a newcomer.

## Environment

- **Go 1.26+** — use the pinned toolchain (`GOTOOLCHAIN=auto`).
- **Node 24 + pnpm 10** — only for `desktop/` work (separate Go module).
- Isolated dev: set `REASONIX_HOME=/tmp/reasonix-dev` so a source build shares
  no on-disk state with a stable release (config, credentials, sessions, cache).

## Build / test / lint

```bash
make build      # go build ./...
make test       # go test ./...
make vet        # go vet ./...
make fmt        # gofmt -w .
make lint       # golangci-lint (pinned) + tools/repolint
make cross      # cross-compile all 6 targets
make hooks      # install git hooks
```

- Root Go tests do **not** cover `desktop/` (separate module): run
  `cd desktop && go test ./...` there.
- Desktop transcript/scroll changes follow `docs/TRANSCRIPT_SCROLL_CONTRACT.md`
  and `pnpm test:transcript` in `desktop/frontend/`.
- Correctness gates must be **deterministic** (channels / injected clocks /
  state transitions), not wall-clock races.

## PR metadata (enforced by CI)

Two scripts are the source of truth and both run locally:
`scripts/check-cache-impact.sh`, `scripts/check-docs-impact.sh`.

- **`Cache-impact: none|low|medium|high - reason`** + **`Cache-guard: …`**
  required when the diff touches cache-sensitive paths (`internal/tool/`,
  `internal/provider/`, `internal/boot/`, `internal/agent/agent.go`, and the
  rest of the list in the script). `none` is legitimate only when the
  provider-visible prefix stays byte-identical.
- **`System-prompt-review: …`** additionally required when touching
  `internal/config/`, `internal/memory/`, `internal/outputstyle/`,
  `internal/skill/`, or `internal/boot/` (it rejects `none`/`n/a` — name the
  reviewer/approval).
- **`Documentation-impact: updated - what`** (or `none - why docs stay
  correct`) required for user-visible diffs (`cmd/reasonix/`, `desktop/`,
  `npm/`, most of `internal/`).
- Separators must be ASCII `-` or `:` — an em dash fails the docs guard.

## Code style & repolint

- `gofmt` enforced; errors wrapped with `fmt.Errorf("...: %w", err)`.
- Library code never calls `os.Exit` or prints; only `cli`/`main` own exit
  codes and user messages.
- Exported identifiers need doc comments; explain *why*, not *what*.
- `TODO(#nnn)`/`HACK(#nnn)` need an issue anchor; `FIXME` is rejected.
- One responsibility per file; `tools/repolint` ratchets existing debt — an
  edit cannot silently increase a file's recorded debt.
- Conventional Commits (`feat(glob): …`, `fix: …`, `test(event): …`, …).
- Ordinary follow-up commits + fast-forward pushes; force-push only when
  authorized.

## Extending the system

- **New built-in tool** (`CONTRIBUTING.md`): create
  `internal/tool/builtin/mytool.go`, implement `tool.Tool`, register via
  `init() { tool.RegisterBuiltin(...) }`, add tests. It's automatically
  available because `main` blank-imports `builtin`.
- **New provider kind**: create `internal/provider/myprovider/`, implement
  `provider.Provider` (`Name`, `Stream`), register via
  `init() { provider.Register("mykind", New) }`.
- **New MCP server**: it's config, not code (see `[[plugins]]`).
- **i18n string**: add the field + `messages_en.go` + `messages_zh.go`
  (`TestCatalogsComplete` fails if you miss a locale).

## Where to start (by interest)

| Interest | Start here |
| --- | --- |
| The LLM loop | `internal/agent/run_loop.go`, `internal/provider/openai/openai.go` |
| Prompt/cache | `internal/boot/`, `internal/memory/`, `internal/skill/` |
| Tools | `internal/tool/builtin/` (pick one tool, read its `_test.go`) |
| Safety | `internal/permission/`, `internal/sandbox/`, `internal/guardian/` |
| Persistence | `internal/agent/session_dag*.go`, `internal/checkpoint/` |
| UI | `internal/cli/`, `internal/serve/`, `desktop/` |

Small, self-contained starter ideas: a new built-in tool, a bug fix in one
tool, an i18n string, a deterministic test for a lifecycle edge case, or a
doc-comment/repolint cleanup. Before coding a behavior change, read the owning
`*_test.go` — the tests encode the intended contract.

## "Making it like Cursor" — an honest framing

Reasonix already shares a lot of Cursor's *shape* (agent loop, plan mode,
permissions/sandbox, checkpoints, MCP, editor integration via ACP). The gap to
Cursor-class polish is mostly in **quality and coverage**, not missing magic:

- **Model-agnostic resilience** — reasoning replay, stream recovery, compaction
  are already first-class; the differentiators are correctness under real
  provider variance (the `live_*_test.go` and `*_recovery_test.go` files are
  where this is proven).
- **Editor experience** — context gathering, apply-edit fidelity, and low-latency
  streaming live in the ACP/extension and desktop layers.
- **Evaluation** — `internal/eval`, `internal/ablation`, `benchmarks/`, and the
  delegation counters in `docs/SPEC.md` §3.17 show how the project measures
  "did the agent actually do the work" (host-adjudicated claims) rather than
  trusting prose.

Contributing well means: reproduce with a failing test, keep the prefix
byte-stable (cache), keep gates deterministic, and satisfy the PR metadata
guards.

Next: [08 · agentic design patterns](08-agentic-design-patterns.md).
