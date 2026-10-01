# 02 · The agent engine core (LLM interaction)

This is the heart of Reasonix: how a user turn becomes model requests, how the
model's stream is consumed, how tool calls are executed and fed back, and how
the loop terminates. All types are in `internal/provider`; the loop is in
`internal/agent`.

## The two interfaces that matter

The engine is defined by two interfaces plus a few data types.

### `provider.Provider` — the model backend (`internal/provider/provider.go`)

```go
type Provider interface {
    Name() string
    Stream(ctx context.Context, req Request) (<-chan Chunk, error)
}
```

A provider is a **vendor endpoint** (one `base_url` + `api_key_env`) offering
one or more models. Concrete adapters self-register under a **kind**:

```go
type Factory func(cfg Config) (Provider, error)
func Register(kind string, f Factory)   // called from init()
func New(kind string, cfg Config) (Provider, error)
```

Registered kinds (subpackages of `internal/provider`): `openai`
(`/chat/completions`), `anthropic` (`/messages`), `responses` (OpenAI/DeepSeek
Responses API). **Adding another OpenAI-compatible vendor is a config edit, not
code** — the same `openai` kind covers DeepSeek, MiMo, MiniMax, Zhipu GLM,
LongCat, Kimi K3, Ollama Cloud, etc. The adapter picks per-vendor wire details
(reasoning knobs, headers, URL) from `base_url`/`model`/`extra`.

### `tool.Tool` — a capability the model may call (`internal/tool/tool.go`)

```go
type Tool interface {
    Name() string
    Description() string
    Schema() json.RawMessage   // JSON Schema for parameters
    Execute(ctx context.Context, args json.RawMessage) (string, error)
    ReadOnly() bool            // gates parallel batches + reader-default policy
}
```

See [04 · tools, permissions, sandbox](04-tools-permissions-sandbox.md).

## Provider-visible data types (`internal/provider/provider.go`)

- `Message{Role, Content, RawContent, ReasoningContent, ReasoningSignature,
  ThinkingBlocks, ToolCalls, ResponsesItems, Images, LocalOnly, …}` — one
  conversation record. `Content` is the **provider-visible** form (bounded, e.g.
  tool results ≤32KB); `RawContent` is the full local original, never sent.
  `LocalOnly` marks records that must never reach a model (interrupted stream
  fragments, host recovery data). `ModelMessages()` strips local-only fields.
- `Request{Messages, Tools []ToolSchema, Temperature *float64, MaxTokens,
  ResponseFormat, EffortOverride, ToolSearch}` — one completion request.
- `ToolSchema{Name, Description, Parameters json.RawMessage, Deferred, Strict,
  Namespace}`.
- `Chunk{Type, Text, Signature, ReasoningID/Status, ToolCall, ArgChars,
  ResponsesItem, ServerSearch, Usage, Err}` — one streamed increment. Types:
  `ChunkText`, `ChunkReasoning` (thinking), `ChunkToolCallStart`,
  `ChunkToolCallArgsDelta` (progress ticks), `ChunkToolCall` (complete call),
  `ChunkUsage`, `ChunkDone`, `ChunkError`, `ChunkResponsesItem`,
  `ChunkServerSearch`.
- `Usage{PromptTokens, CompletionTokens, TotalTokens, CacheHitTokens,
  CacheMissTokens, CacheWriteTokens, ReasoningTokens, FinishReason, …}` —
  normalized across OpenAI-style and Anthropic-style usage shapes
  (`normaliseUsage` in `openai.go`).

## The turn loop, end to end

Entry: `Agent.Run` (`internal/agent/agent.go`) → `beginRunTurn` → `runToolLoop`
(`internal/agent/run_loop.go`).

```
Run(ctx, input)
  └─ beginRunTurn           build turnRuntime; append the user Message to the session
  └─ runToolLoop            per-round loop (bounded by maxSteps / budgets / grace round)
       │  1. consumeSteer?           mid-turn guidance → append as a user Message
       │  2. schemas = providerToolSchemas()      // canonicalized tool list
       │  3. prefixShape = capturePrefixShape()   // hash of prompt+tools+history, for cache diagnostics
       │  4. streamed = streamWithSamplingRecovery(ctx, step)
       │        └─ stream → streamWithFrozen      // see "Streaming" below
       │  5. commit assistant Message (text + reasoning + tool calls) to the session
       │  6. if no tool calls → handleFinalResponse (finish, or retry empty final)
       │     else             → handleToolRound → executeBatch (execute tools) → append results → loop
       └─ (each round) contextManager().ObserveUsage(usage)   // compaction trigger check
```

Key facts about the loop:

- The **session is append-only** except at the compaction boundary. Every
  provider request re-sends the system prefix + history, so the prompt prefix
  stays cacheable.
- `streamWithSamplingRecovery` is the sampling entry: it owns retry budgets and
  replay. **Providers never replay bodies themselves** — mid-stream errors are
  reported as `provider.StreamInterruptedError` and the Agent decides
  commit-vs-replay (see `internal/agent/sampling_recovery.go`,
  `sampling_request.go`).
- `runToolLoop` captures a `PrefixShape` before sampling and compares it after,
  to attribute any cache miss to the operation that caused it (compaction,
  prune, rewind, steer).

### Streaming (`Agent.streamWithFrozen`, `internal/agent/agent.go`)

`streamWithFrozen` calls `prepareSamplingRequest` (builds the `provider.Request`
from the system prompt + `ModelMessages(session)` + tool schemas), then
`provider.Stream`, then consumes the `<-chan Chunk`:

- `ChunkReasoning` → buffered into reasoning (and streamed live unless a
  `PostLLMCall` hook will rewrite it).
- `ChunkText` → appended to `text`, emitted as a `event.Text`.
- `ChunkToolCallStart` / `ChunkToolCallArgsDelta` / `ChunkToolCall` → partial
  tool cards are emitted to the UI the moment a call's *name* is known, so the
  user sees work start before the arguments finish streaming.
- `ChunkUsage` → recorded (cache hit/miss/reasoning tokens).
- `ChunkError` / channel close → `interceptProviderResponse` lets extensions
  rewrite or block the assembled terminal response; the result becomes the
  committed assistant turn.

On the provider side (`internal/provider/openai/openai.go`), `Stream` →
`openStream` marshals the body, POSTs with `Accept: text/event-stream`, and
`readStream` parses the SSE line-by-line (with an idle watchdog that turns a
half-open connection into a recoverable `StreamInterruptedError`). Tool-call
deltas are **accumulated by index** and emitted as complete `ToolCall`s on
`[DONE]`; usage is folded from whatever cache/reasoning shape the vendor
returns.

### Tool execution (`internal/agent/execute_one.go`, `tool_dispatch.go`)

When the model returns tool calls, `handleToolRound` → `executeBatch` runs them.
Per call, `executeOne` walks a fixed pipeline:

1. **Parse** — `Registry.ResolveCall` (exact name, shell alias, MCP alias;
   ambiguous → blocked, never executed).
2. **Permission** — `Gate.Check` (allow/ask/deny), with a trusted-MCP fast path
   and extension interception.
3. **Sandbox stamp** — the permission preset + per-call write roots are placed
   in the context (`sandbox.WithPermissionPreset`); enforcement happens *inside*
   each tool's `Execute` (bash wraps itself in `sandbox-exec`/`bwrap`, file
   writers confine targets to the write roots).
4. **Hooks** — `PreToolUse` (may block) before execution.
5. **Execute** — dispatch to built-in or plugin tool.
6. **Result** — `PostToolUse`/`PostToolUseFailure`, then the result is
   size-bounded (`boundProviderVisibleResult`) and appended to the session as a
   `role=tool` message.

Errors from `Execute` are **fed back to the model, not fatal** — the model can
self-correct.

## The reasoning contract

"Thinking" (chain-of-thought) is first-class, not an afterthought:

- Reasoning arrives as `ChunkReasoning` deltas and is persisted on the assistant
  message as `ReasoningContent` (+ `ReasoningSignature` for Anthropic signed
  thinking, `ReasoningID`/`ReasoningStatus` for Responses items).
- On the **next** turn it must round-trip back into the request: DeepSeek
  thinking mode requires the `reasoning_content` key on assistant history turns,
  Anthropic requires replaying the signed thinking block, Kimi K3/GLM require
  the complete message. The optional provider policies
  (`RequiresToolCallReasoning`, `RequiresReasoningRoundTrip`,
  `WarnOnMissingToolCallReasoning`) declare each backend's contract; the
  adapters implement them. See `docs/REASONING_CONTRACT.md` and
  `docs/REASONING_PROVIDERS.md`.
- A display-translated reasoning copy never round-trips — the raw provider text
  is what goes back to the API.

## Context management / compaction

Long tasks outgrow the model window. Reasonix keeps a **cache-first,
append-only canonical transcript** and installs a short **provider-visible
checkpoint** only when one threshold is crossed. See
`docs/SPEC.md` §3.6 and `internal/agent/context_manager.go`,
`compact*.go`.

- `triggerTokens = floor(context_window × compact_ratio)`; default
  `compact_ratio = 0.80`.
- At the trigger, a singleflight maintenance transaction first prunes tool
  results >8192 code points to `4096 head + marker + 1024 tail`. If that
  relieves pressure, no summary is made.
- Otherwise the oldest prefix is **summarized** (replaying the original system
  message + prefix + tool schemas, so the summary request reuses the provider
  KV cache), keeping the newest **16%** verbatim. Summaries are capped at 8192
  tokens; every summary reply feeds its real prompt count back into the
  estimator.
- Compaction writes a **projection** — canonical storage keeps every original,
  so lost detail stays recoverable through the read-only `history` tool.

## Two-model collaboration (`internal/agent/coordinator.go`)

When `agent.planner_model` is configured, a `Coordinator` runs **two models in
separate sessions** (so neither model's prompt prefix is disturbed by the
other's turns):

- The **planner** runs in its own session with the same standing memory plus a
  filtered **read-only** tool set, and delivers a plan via the single
  `submit_plan` channel (prose without a submitted plan is a protocol error).
- The plan is handed to the **executor** — a full tool-using `Agent` — as
  structured text; the executor validates assumptions and executes.
- A deterministic host policy decides the route (executor-only vs
  plan-and-execute vs plan-for-approval vs plan-only); it never infers
  complexity from wording or file counts.
- Both `Agent` and `Coordinator` satisfy the `Runner` interface, so the CLI is
  agnostic to which mode is active.

## Why it is shaped this way (the north star)

Almost every decision in this file traces back to **prompt-cache stability**:
the system prompt and tool schemas are byte-stable across turns; the transcript
grows prepend-only; per-turn state rides in a separate host message; and
two-model collaboration uses separate sessions *because* switching models inside
one shared conversation would break the prefix and tank cache hits. Read
[03 · context and cache](03-context-and-cache.md) for the assembly side.
