# 15i — Walkthrough: Background Memory Extraction & Consolidation

While a conversation runs, Claude Code quietly extracts durable facts into its
file-based memory, and periodically consolidates weeks of sessions — all in
forked background agents that never block the main loop. This is the runtime view
of [10](10-memory-and-context.md).

## Two mechanisms, both off the critical path

- **Session memory** — extracts key facts *mid-session* from the live
  conversation.
- **Auto-dream** — *across sessions*, periodically reviews transcripts and
  consolidates/improves the persisted memory files.

Both run as **forked sub-agents** ([15b](15b-walkthrough-subagent-fork.md)) with
Read/Write tools (Bash disabled), so they can't interfere with the user's turn.

## Trace A — Session memory extraction (mid-session)

### 1 — Should we extract now?
After a turn, `shouldExtractMemory()`
(`src/services/SessionMemory/sessionMemory.ts:134`) gates on **all** of:
- context grew past `minimumTokensForInitialization`;
- enough tokens since the last extraction (`minimumTokensBetweenUpdate`);
- enough tool calls since last update (`toolCallsBetweenUpdates`);
- the conversation is at a natural break (last assistant turn had no tool calls).

### 2 — Fork an extraction agent
If gated open, a forked subagent runs (`:183-250`) with a `/dream`-style prompt,
Read/Write only, appending extracted points to the **session memory file**. Token
budget is bounded by `maxExtractionTokens`.

### 3 — It feeds compaction
The extracted session memory is what `trySessionMemoryCompaction`
([14d](14d-deep-dive-compaction.md), `sessionMemoryCompact.ts`) later uses to do
*lossless* compaction — keeping recent turns verbatim and leaning on the archive
instead of an LLM summary. So extraction quietly pays off when the window fills
([15e](15e-walkthrough-autocompaction.md)).

## Trace B — Auto-dream consolidation (across sessions)

### 1 — Throttled scan
`autoDream.ts` checks periodically (`SESSION_SCAN_INTERVAL_MS = 10 min`, `:56`) to
avoid rapid re-scans.

### 2 — Three gates
Consolidation fires only when: ≥ `minHours` since last (default 24); ≥
`minSessions` sessions touched since (default 5); and the lock is free
(`tryAcquireConsolidationLock()`). Config comes from the `tengu_onyx_plover`
GrowthBook flag.

### 3 — The lock is the timestamp
The lock file's **mtime** doubles as "last consolidated at"
(`consolidationLock.ts`); on kill it rewinds the mtime (abort semantics), so a
crashed consolidation doesn't reset the clock.

### 4 — Fork a consolidation agent
`buildConsolidationPrompt(memoryRoot, transcriptDir, …)` →
`runForkedAgent(..., querySource: 'auto_dream')` (`autoDream.ts:224-248`). It
reads recent transcripts and **creates or improves** memory files, then appends
an "Improved N memories" note to the main transcript and calls
`completeDreamTask()` / `failDreamTask()`.

## How memory is stored & recalled

Memories live at `~/.claude/projects/<slug>/memory/` as `<topic>.md` files with
frontmatter (`name`, `description`, `type`, optional `scope: team`), plus an
always-loaded index `MEMORY.md` capped at 200 lines / 25 KB
(`truncateEntrypointContent`, `memdir.ts:57`). The two-step save protocol —
write `<topic>.md`, then add a one-line pointer to `MEMORY.md` — lets the model
discover relevant memories from the small index without reading every file.

A separate **relevant-memory prefetch** runs once per user turn
(`startRelevantMemoryPrefetch`, `query.ts:301`) and is consumed (never blocking)
after tools — surfacing pertinent memories into the turn.

## Guards & isolation

- **Team safety**: `scope: team` memories sync via `teamMemorySync/`, and a
  secret scanner (`secretScanner.ts`, `teamMemSecretGuard.ts`) blocks secrets
  from shared memory ([10](10-memory-and-context.md)).
- **Subagent-safe cleanup**: post-compact cleanup resets memory caches for the
  main thread only, not for `agent:*` forks ([14d](14d-deep-dive-compaction.md)).

## What this illustrates

- **Learning is background work**: every memory operation is a forked agent, so
  the user's turn is never blocked by extraction or consolidation.
- **Gated and throttled**: token/tool/time/session thresholds + a single
  consolidation lock keep memory work from running too often or concurrently.
- **The index is the recall trick**: a tiny always-loaded `MEMORY.md` plus
  per-turn relevance prefetch means the model can use a large memory corpus
  without paying to load all of it.
- **Memory and compaction are linked**: session-memory extraction is what makes
  *lossless* session-memory compaction possible.
