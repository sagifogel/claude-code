# 15e — Walkthrough: Auto-Compaction Firing Mid-Conversation

A long session approaches the model's context window. Before the next request,
the loop proactively summarizes the history so the conversation can continue.
This is the runtime view of the mechanism dissected in
[14d](14d-deep-dive-compaction.md); see also [03](03-query-engine.md) §7.

## The trace

### 1 — Turn begins, context is measured
At the top of a `queryLoop` iteration ([14a](14a-deep-dive-agent-loop.md)),
`messagesForQuery` is assembled from messages after the last compact boundary
(`query.ts:365`). The cheaper rungs run first: tool-result budget (`:379`), snip
(`:401`, recording `snipTokensFreed`), microcompact (`:414`), context collapse
(`:440`).

### 2 — Threshold check
Autocompact runs at `query.ts:454`. `shouldAutoCompact(...)`
(`autoCompact.ts:160`) computes
`tokenCount = tokenCountWithEstimation(messages) - snipTokensFreed` (`:225`) and
compares against `getAutoCompactThreshold(model)` (`:72`):

```
effectiveWindow      = contextWindow(model) − min(maxOutputTokens, 20_000)   // :33-49
autocompactThreshold = effectiveWindow − 13_000                              // :62, :76
```

It **skips** if disabled, if the query source is `session_memory`/`compact`
(recursion guard, `:171`), or if reactive-compact / context-collapse own overflow
(`:195`, `:220`). Subtracting `snipTokensFreed` is why snip runs first — its
savings count toward staying under threshold.

### 3 — Over threshold → compact
`autoCompactIfNeeded(...)` (`autoCompact.ts:241`):
- **circuit breaker** — bail if `consecutiveFailures >= 3` (`:257-265`; the
  comment cites *"1,279 sessions had 50+ failures, wasting 250K API calls/day"*);
- **cheap path first** — `trySessionMemoryCompaction(...)` (`:287`): if session
  memory is ready, keep recent turns verbatim and lean on the archived memory,
  *but only if the result lands under threshold* — otherwise fall through
  (`sessionMemoryCompact.ts:604-613`);
- **full compaction** — otherwise `compactConversation(...)`
  (`compact.ts:387`).

### 4 — Generate the summary
`compactConversation` issues a dedicated LLM call with the summary prompt
(`prompt.ts`), whose `NO_TOOLS_PREAMBLE` (`:19-26`) insists on **text only**
(*"Tool calls will be REJECTED and will waste your only turn"*) and a 9-section
`<summary>` (intent, key concepts, files & code, errors & fixes, pending tasks,
current work, next step…). `formatCompactSummary` (`:311`) strips the
`<analysis>` scratchpad.

### 5 — Rebuild the message list
`buildPostCompactMessages(result)` (`compact.ts:331`) orders: **boundary marker →
summary → kept tail → attachments → hook results**. The system **re-injects**
recent file attachments (≤5 files / 50k tokens), delta attachments for deferred
tools/agents/MCP, and recent skills — but deliberately *not* the full skill
listing (`:524-529`). Discovered tool names ride forward on the boundary
(`:606-611`).

### 6 — Cleanup
`runPostCompactCleanup(querySource)` (`postCompactCleanup.ts:31`) resets stale
state **for the main thread only** (`repl_main_thread`/`sdk`, not `agent:*`
subagents sharing the process, `:22-29`): microcompact state, context-collapse
store, memoized `CLAUDE.md`/memory caches, system-prompt sections, classifier
approvals — leaving invoked-skill content intact.

### 7 — Continue with the compacted context
The loop proceeds with the new, smaller `messages`. From here on,
`getMessagesAfterCompactBoundary` (`query.ts:365`) means subsequent turns only
process post-boundary messages — the summary stands in for everything before it.

## The reactive variant (recovery, not proactive)

If autocompact's estimate was off and the API returns a 413, the loop's recovery
state machine ([14a](14a-deep-dive-agent-loop.md)) kicks in: first drain staged
context-collapses (`query.ts:1089`), then `tryReactiveCompact(...)` (`:1119`) —
single-shot, guarded by `hasAttemptedReactiveCompact` so a still-too-long retry
surfaces the error instead of spiraling.

## What this illustrates

- **Proactive vs. reactive**: autocompact fires *before* the request based on an
  estimate; reactive compaction is the *rescue* after an actual overflow. Both
  reach the same `compactConversation`.
- **Cheap-before-lossy inside autocompact itself**: even once it decides to
  compact, it tries the lossless session-memory path before the lossy summary.
- **Compaction is a context boundary, not a delete**: the boundary marker +
  `getMessagesAfterCompactBoundary` mean old messages still exist on disk (and
  for resume) — they're just no longer sent to the model.
- **Guards everywhere**: recursion guard, circuit breaker, main-thread-only
  cleanup — compaction is high-blast-radius, so it's heavily fenced.
