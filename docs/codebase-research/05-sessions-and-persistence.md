# 05 · Sessions, persistence, and rewind

Reasonix keeps a durable, replayable record of every conversation and every
pre-edit file state, so a long autonomous run is "something you can still read
and undo".

## Session storage: an append-only DAG (`internal/agent/session_dag*.go`)

A session is a schema-2, append-only event log written as `.events.jsonl`
(one JSON object per line). Key pieces:

- `session_dag.go` — the DAG model (nodes = messages/turns, edges = lineage).
- `session_dag_writer.go` / `session_writer.go` — the single writer (one
  authority owns writes; see `session_write_authority.go`).
- `session_load.go` / `session_load_context.go` — loading and restoring the
  conversation, with `NormalizeSessionMessages` repairing tool-call pairing on
  load without dropping stored history.
- `save*.go` (`save_persist.go`, `save_dag.go`, `save_tool_checkpoint.go`, …)
  — the commit paths; a save is transactional and crash-safe.
- `session_lease.go` / `session_lock_unix.go` — single-process/single-writer
  lease and lock.
- `session_checkpoint.go` — per-turn checkpoint bookkeeping.
- `migrate.go` / `migrate_native.go` — upgrades from older storage formats.

The on-disk shape (conceptually): a session directory holds the append-only
`.events.jsonl` transcript, per-turn checkpoints, and listing/projection
metadata. The **canonical transcript is never rewritten** — compaction only
adds a *projection*; the originals remain recoverable.

## Rewind / checkpoints (`internal/checkpoint`)

`internal/checkpoint` is the **snapshot-based edit safety net**:

- Before a writer tool changes a file, the engine records the file's **pre-edit
  content**, keyed to the current user turn.
- A frontend can then **rewind the workspace** (and, via a `ConversationApplier`,
  the conversation) to an earlier turn: `PrepareRewind` / `CommitRewind`
  (`checkpoint.go`, `transaction.go`). Layout is v3 per-turn directories.
- `internal/checkpoint` is *workspace* rewind; conversation rewind lives in
  `control/rewind*.go` / `agent/session_checkpoint.go`.

## History search (`internal/history`, `historycatalog`)

- The read-only `history` tool gives the model on-demand **BM25 retrieval** over
  saved session JSONL files (`scope=project|global`, `operation=around` for a
  bounded transcript window). Same BM25 machinery powers `memory` recall.
- `historycatalog` maintains a search catalog/index for the desktop UI's
  history search.

## Storage backends (`internal/store`, `internal/projectiondb`, `internal/session*`)

- `internal/projectiondb` — SQLite (`modernc.org/sqlite`, pure Go) for
  projections/catalogs.
- `internal/store` — path authority and remote store; `internal/session` has a
  `recovery_store` (bbolt) for crash recovery.
- `sessioncatalog`, `sessioncontent`, `sessionexport`, `sessioninbox`,
  `sessiontitle`, `taskcatalog`, `usagecatalog` — catalog/content services over
  those stores (session list, content, export, the inbox queue, auto-titles,
  task and usage records).

## Transcript display (`internal/transcript`)

`internal/transcript` represents the conversation for the UI: `message.go`
(message model), `history.go`, `projection.go` (provider-visible vs. local
view), `buffer.go`/`snapshot.go`. The desktop frontend additionally follows the
scroll/virtualization contract in `docs/TRANSCRIPT_SCROLL_CONTRACT.md`.

## How rewind/checkpoint fits the loop

1. Turn N runs; each write tool records a pre-edit snapshot in `checkpoint`.
2. The user rewinds to turn K: the conversation is truncated back to K (the
   DAG/`ConversationApplier` path) and each file modified since K is restored
   from its pre-edit snapshot.
3. The session remains consistent because writes are transactional and the
   transcript is append-only — rewind rewrites the *view*, not the raw history.

Next: [06 · frontends and entrypoints](06-frontends-and-entrypoints.md).
