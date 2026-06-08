# 15d — Walkthrough: Resuming a Previous Session

How `claude --resume` / `--continue` / `/resume` reconstructs a past conversation
from disk and continues it as if it never stopped. This depends on the
transcript persistence described in [03](03-query-engine.md) and the message
model in [14a](14a-deep-dive-agent-loop.md).

## Where transcripts live

Each session is a JSONL file at
`~/.claude/projects/<project>/<sessionId>.jsonl`. Every turn, `recordTranscript`
(`src/utils/sessionStorage.ts:1408`) appends only the *new* messages
(`insertMessageChain`), wiring each to its `parentUuid` — so the file is an
append-only DAG of the conversation, with metadata entries (mode, cost, worktree
state) and `system_compact_boundary` markers interleaved.

## The trace

### 1 — Trigger
CLI flags are parsed in `main.tsx` (`--continue`, `--resume <id>`, `:763-764`).
`--continue` → `loadConversationForResume(undefined, undefined)`
(`main.tsx:3101`). The interactive `/resume` command is a `local-jsx` command
(`src/commands/resume/index.ts:3-12`) whose UI calls
`context.resume?.(sessionId, log, entrypoint)` (`resume.tsx:195`).

### 2 — Discover prior sessions
The picker lists sessions via `loadSameRepoMessageLogs(...)`
(`sessionStorage.ts:4073`) / `loadAllProjectsMessageLogs()`, which enumerate the
`.jsonl` files **stat-only** first (lite `LogOption`s with `sessionId`+`fullPath`
but empty `messages`) — `loadMessageLogs` (`:3939`). The current session is
filtered out: `filterResumableSessions` (`resume.tsx:191`).

### 3 — Load the chosen transcript
`loadFullLog(log)` (`sessionStorage.ts:2949`) → `loadTranscriptFile(filePath)`
(`:3472`) parses the JSONL (`parseJSONL`, `:3614`) into a
`Map<uuid, TranscriptMessage>`. For `--resume <id>`, `getLastSessionLog(id)`
(`:3869`) → `loadSessionFile` (`:3831`) resolves the path under the project dir.

### 4 — Rebuild the conversation chain
`buildConversationChain(messages, leaf)` (`:2069`) walks `parentUuid` backwards
from the leaf, reverses, and then **`recoverOrphanedParallelToolResults(...)`** —
the single-parent walk would lose sibling `tool_result`s from parallel tool calls,
so this post-pass reconstructs them. Legacy `progress` entries are bridged first.

### 5 — Deserialize
`deserializeMessages` (`conversationRecovery.ts:154`) →
`deserializeMessagesWithInterruptDetection` (`:164`) migrates legacy attachment
types and `filterUnresolvedToolUses` — dropping a dangling `tool_use` that never
got a `tool_result` (e.g. the process died mid-tool), so the resumed history is
API-valid.

### 6 — Restore session state
`processResumedConversation(...)` (`sessionRestore.ts:409`) (or the REPL's
`resume` callback, `REPL.tsx:1735`):
- `switchSession(sessionId, projectDir)` (`bootstrap/state.ts:468`) atomically
  swaps the session id + project dir and emits `sessionSwitched`;
- `resetSessionFilePointer()` so new writes append to the resumed file;
- `restoreCostStateForSession(...)`, `restoreSessionMetadata(...)`,
  `restoreWorktreeForResume(...)`, `adoptResumedSessionFile()`;
- `setMessages(() => messages)` (`REPL.tsx:1929`) loads the history into the
  component for rendering.

### 7 — Continue
There is **no special "resumed" query path**. Once `setMessages` has the
restored `Message[]`, the normal loop ([14a](14a-deep-dive-agent-loop.md)) runs.
New turns append via `recordTranscript`, which dedupes against the already-loaded
set (`getSessionMessages` memoized cache, `:3842`) and chains new messages off
the restored leaf's `startingParentUuid` (`:1408-1449`).

## Notable details

- **UUID stability across resume.** `normalizeMessages` splits multi-block
  messages and assigns derived UUIDs via `deriveUUID(parent, index)`
  (`messages.ts:725-728`) — the index encoded in the last 12 hex chars — so the
  same logical block gets the same id every time it's loaded.
- **Lite vs. full logs.** Discovery is stat-only for speed; full message arrays
  load on selection. A large file (>5 MB) uses compact-boundary offsets to skip
  stale pre-boundary data.
- **Compact boundaries persist.** A resumed session that had been compacted keeps
  its boundary markers, so `getMessagesAfterCompactBoundary`
  (`query.ts:365`) still works correctly post-resume.

## What this illustrates

- The transcript is the **source of truth**, written incrementally during the
  live session precisely so resume is "just load the DAG and keep going."
- Reconstruction is non-trivial because of **parallel tool calls** (orphan
  recovery) and **interrupted turns** (unresolved-tool filtering) — the file is a
  DAG, not a flat list.
- Resume is **transparent to the engine**: restored messages are
  indistinguishable from fresh ones once in state, which is why the same
  `queryLoop` drives both.
