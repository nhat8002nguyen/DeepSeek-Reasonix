# 08 · Design patterns for agentic tools (beyond a coding agent)

This is the payoff for reading the code: the transferable design patterns
Reasonix embodies. Each is general — useful for a support agent, a data-analysis
agent, a browser agent, or any "LLM + tools + feedback loop" product.

## 1. Interface + registry, not a switch statement

The core depends only on `Provider` (models) and `Tool` (capabilities). Concrete
implementations **self-register** from `init()` into a kind/name registry, and
configuration selects them by name. Result: adding a vendor or a capability is a
config entry or a new file — never a change to the loop.

**Pattern**: define minimal interfaces; let plugins register themselves; resolve
by name at runtime.

## 2. Transport-agnostic controller + typed event stream

One `Controller` owns the run loop and session lifecycle, exposes **command
methods** (`Send`, `Cancel`, `Approve`, …), and emits **one typed event stream**
to a single `Sink`. Every frontend (TUI, HTTP/SSE, desktop, editor protocol) is
just a command emitter + event renderer.

**Pattern**: separate *orchestration* from *presentation* with a small command
surface and a typed event channel. It keeps cancellation/approval/persistence
correct in exactly one place.

## 3. Policy vs. enforcement are two layers

Permission (`Policy`/`Decision`) decides *whether*; sandbox (`workspace_root`,
OS jail) enforces *what is physically possible*. A permitted call still can't
escape its roots; a denied call never runs. They're intentionally separate so
policy can be fast/pure/auditable while enforcement is OS-level.

**Pattern**: don't let an allow/deny check be your only boundary — add an
enforcement layer that holds even when policy is bypassed or misconfigured.

## 4. Cache-stable context assembly

The expensive part of an agent loop is re-sending context every turn. Reasonix
assembles the system prompt + tool schemas **once**, hashes them
(`RuntimeSnapshot.CacheHash`), keeps the transcript **append-only**, and puts
all *changing* state in a per-turn tail message. Compaction writes a
**projection** while canonical storage keeps the originals.

**Pattern**: design the model-visible prefix to be byte-stable; separate "the
stable policy" from "the changing state"; keep full history recoverable even
when you summarize.

## 5. Append-only transcript + projection

The canonical log is never rewritten; derived views (provider-visible history,
UI transcript, summaries) are **projections**. This makes rewind, replay,
crash-recovery, and caching all fall out of one durable source of truth.

**Pattern**: event-source your conversation (append-only JSONL), and derive
everything else. Rewind = recompute/truncate the view, not mutate the log.

## 6. Checkpoint before mutation

Before any write, record the pre-edit state keyed to the current turn. Rewind
restores files and the conversation to an earlier turn. Combined with #5, it
gives the user "undo" over a long autonomous run.

**Pattern**: for any agent that mutates state, capture a reversible snapshot
and tie mutations to a turn/step id so you can rewind by intent, not by time.

## 7. Sub-agents as a strict boundary

Delegation is split into five concepts — **profile** (how a worker thinks),
**TaskSpec** (what this call wants), **CapabilityGrant** (what it may touch),
**ContextCapsule** (what it starts from), **SchedulerPolicy** (when it runs) —
and a child inherits *nothing* implicitly. One construction primitive
(`RunProfileSpec`) applies every safety layer.

**Pattern**: make child contexts explicit (no ambient inheritance), and funnel
all child-spawning entry points through one code path so a safety layer can't
be forgotten on one of them.

## 8. Host-adjudicated completion, not prose

A sub-agent's "I'm done" is a **claim**, checked against host-recorded receipts
(commands actually run, files actually written). Unverifiable claims are
downgraded. This is how you measure *whether work happened* rather than
*whether the model said it did*.

**Pattern**: have the model emit structured claims (criteria + evidence) and
have the host verify each against its own ledger before trusting the outcome.

## 9. Bounded budgets and fail-closed defaults

Budgets (tokens, spend, steps, wall-clock) are explicit and resumable;
non-interactive runs fail closed when a decision is needed; a broken provider
stream is an explicit `StreamInterruptedError` the agent decides to retry —
never a silent hang or an unbounded replay.

**Pattern**: bound every loop with a *named* budget so pauses are resumable and
diagnosable, and make "I can't verify this" fail closed by default.

---

These nine patterns, not any single model or prompt, are what make an agentic
tool feel trustworthy at scale. Reasonix is a clean, test-backed reference
implementation of all of them — which is why studying it transfers to building
your own agentic products, coding-tool or otherwise.
