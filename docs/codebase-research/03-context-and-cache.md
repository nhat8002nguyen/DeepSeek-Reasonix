# 03 · Context assembly and prompt-cache invariants

Everything the model sees — the system prompt, standing instructions, memory,
skills index, and tool schemas — is assembled once at startup by `internal/boot`
and kept **byte-stable** across turns. This document explains how, and why.

## The assembly pipeline

`internal/boot/boot.go` (`Boot`/`Build`) turns configuration into a ready
`control.Controller`. The prompt-relevant part is a single `build()`:

1. **`internal/config`** — load and merge config
   (`flag > project reasonix.toml > user config.toml > defaults`, plus
   `.mcp.json`). The system prompt comes from `system_prompt`,
   `system_prompt_file`, or `DefaultSystemPrompt`; reasoning/response language
   and effort also resolve here.
2. **Base system prompt** → append **`internal/outputstyle`** (the output-style
   contract) → append **core policies** (user-decision policy, work-practice
   policy, language policy — these are the fixed "how to behave" blocks) → append
   the **session-context *policy*** → append a static **Workspace/Environment**
   summary → append **`internal/memory`** standing docs (see below) → append the
   **skills *policy*** → optional V4-Pro persona.
3. **Tools** are assembled into a `tool.Registry` (enabled built-ins + MCP
   tools), each schema **canonicalized** once (`provider.CanonicalizeSchema`).
4. **Freeze** — the whole thing becomes an `extension.RuntimeSnapshot`, whose
   `CacheHash` is a SHA-256 of (system prompt + canonicalized tool schemas)
   (`internal/extension/snapshot.go`). This hash is the cache-stability guard:
   tests assert the prefix stays identical across turns
   (`internal/boot/prompt_stability_test.go`).

### Memory: `REASONIX.md` / `AGENTS.md` / `CLAUDE.md` (`internal/memory`)

- `memory.Load` resolves the instruction hierarchy via `instruction.Resolve`
  (user → ancestor → project → local), expanding `@import` with dedup.
- `memory.Compose` folds **only** the `SystemBlock` (policy + standing docs)
  into the stable prefix. The `BackgroundDataBlock` (pinned facts + retrieval
  index) is deliberately kept **out** of the prefix.
- Per-turn **auto-recall**: before each real user turn, bounded BM25 recall
  selects relevant facts from the raw message and appends them as a
  low-authority user-turn suffix — it never mutates the system prompt or tool
  schemas.

### Skills: the index (`internal/skill`)

Skills are Markdown (`<name>/SKILL.md` or `<name>.md`). The **skills index**
shown to the model is a stable `indexHeader` + a `CatalogBlock` (≤4000 chars;
`invocation: manual` profiles are excluded from implicit discovery). The skill
bodies themselves are retrieved on demand, not embedded in the prefix.

### Instruction / output style / i18n

- `internal/instruction` — scoped standing-instruction resolution and rendering
  (broad → specific).
- `internal/outputstyle` — the "keep coding vs. reply" persona block, applied
  once before the other appends.
- `internal/i18n` — host-facing UI strings only. The provider prompt stays
  static English via `config.LanguagePolicy`.

## The cache-first invariants

This is the invariant that drives the whole design (`REASONIX.md`, `SPEC.md` §3.6):

1. **The system prefix and tool schemas are byte-stable across turns.** They
   are assembled once, hashed (`RuntimeSnapshot.CacheHash`), and never rewritten
   mid-session.
2. **Changing state rides in the turn tail**, not the prefix: environment,
   pinned facts, the memory index, and the skills catalog are delivered as a
   digest-validated `<session-context>` host message injected immediately before
   each user turn (`internal/control/session_context.go`,
   `internal/agent/session_context.go`).
3. **The transcript is append-only.** Every request re-sends history verbatim;
   only the compaction boundary writes a *projection* (never rewriting the
   canonical storage).
4. **Tool results are bounded and stable.** `Message.Content` is the
   provider-visible ≤32KB form; `RawContent` holds the full local original and
   is returned only via an explicit paged `use_capability` read.

## Why cache hits are product behavior

`CONTRIBUTING.md` calls high prompt-cache hit rate "product behavior". A large
fraction of the cost of a long agent run is re-sending the (mostly identical)
context each turn; at ~90% cache hit the marginal cost of a turn collapses.
That is why:

- changes touching `internal/provider/`, `internal/tool/`, `internal/boot/`,
  `internal/agent/agent.go`, `internal/config/`, `internal/memory/`,
  `internal/outputstyle/`, `internal/skill/` require a `Cache-impact:` line in
  the PR body and a `Cache-guard:` test (see [07](07-contributing.md));
- switching models *inside* one conversation is avoided — it would break the
  prefix and tank cache hits (hence separate planner/executor sessions);
- stream retries reuse the frozen request byte-for-byte (`freezeProviderRequest`).

Next: [04 · tools, permissions, sandbox](04-tools-permissions-sandbox.md).
