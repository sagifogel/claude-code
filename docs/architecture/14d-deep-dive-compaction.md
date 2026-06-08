# 14d — Deep Dive: The Compaction Cascade

A line-by-line walk of how Claude Code keeps a long conversation inside the
model's context window. There isn't one "compaction" — there are six strategies,
applied in increasing order of information loss. This document follows them in
the exact order `queryLoop` invokes them, citing the real thresholds and guards.
See [03](03-query-engine.md) §7 for the summary and [14a](14a-deep-dive-agent-loop.md)
for the surrounding loop.

## The ladder, cheapest → most lossy

| Rung | Strategy | Cost | Lossy? | Reversible? | Fires when |
|---|---|---|---|---|---|
| 1 | **Snip** | free | yes | no | `HISTORY_SNIP`; old history past tail |
| 2 | **Cached microcompact** | free (cache op) | no | yes | count-based, cache hit, main thread |
| 2′ | **Time-based microcompact** | content clear | yes | no | >1h idle gap |
| 3 | **Context collapse** | free (projection) | no | yes | `CONTEXT_COLLAPSE` |
| 4 | **Session-memory compact** | disk archive | no | yes | session memory ready + flags |
| 5 | **Autocompact** | 1 LLM call | yes | no | within ~13k of effective window |
| 6 | **Reactive compact** | 1 LLM call | yes | no | after an actual 413 |

The guiding rule, stated in the source: run collapse *before* autocompact so
that *"if collapse gets us under the autocompact threshold, autocompact is a
no-op and we keep granular context instead of a single summary"*
(`query.ts:428-431`).

## Sequencing in the loop (`src/query.ts:365-543`)

Each iteration prepares `messagesForQuery`:

1. `applyToolResultBudget` (`:379`) — per-message tool-result size cap. Runs
   *before* microcompact because cached MC keys purely on `tool_use_id` and
   *"never inspects content, so the two compose cleanly"* (`:369-372`).
2. **Snip** (`:401-410`) → `snipTokensFreed`, *plumbed into autocompact's
   threshold check* so it reflects what snip removed (`:397-399`).
3. **Microcompact** (`:414-426`) → optional `pendingCacheEdits` (boundary
   deferred until after the API reports `cache_deleted_input_tokens`).
4. **Context collapse** (`:440-447`).
5. **Autocompact** (`:454-543`).

Reactive compaction is not here — it's a *recovery* path triggered by a withheld
413 later in the loop (`:1119-1166`, see [14a](14a-deep-dive-agent-loop.md)).

## Rung 1 — Snip

`HISTORY_SNIP`-gated. `snipCompactIfNeeded` truncates old history beyond a
protected tail and yields a boundary message with the freed-token count. It's
lossy (old turns dropped) but cheap (no LLM). The freed count is threaded to
autocompact because the token estimator can't otherwise see it — it reads usage
from the protected-tail assistant message, which snip leaves unchanged.

## Rung 2 — Microcompact (`src/services/compact/microCompact.ts`)

`microcompactMessages` (`:267-293`) tries two sub-strategies, time-based first:

### Time-based (`:446-530`) — lossy

`evaluateTimeBasedTrigger` (`:422-444`) fires only on the main thread when the
gap since the last assistant message exceeds `config.gapThresholdMinutes`
(default ~1h, from GrowthBook). It keeps the most recent N tool results and
**replaces older tool-result content** with the marker
`'[Old tool result content cleared]'` (`:36`, `:483`). Because the content
genuinely changed, it then `resetMicrocompactState()` and notifies cache-break
detection to suppress false alarms (`:511-527`). Lossy, not reversible.

### Cached (`:305-399`) — non-lossy

When `CACHED_MICROCOMPACT` is on, the model supported, and it's the main thread
(`:276-286`), it computes tool results to delete and emits a `cacheEdits` block
queued as `pendingCacheEdits` (`:332-338`). These are **API-layer cache-edit
directives**, not content changes — the conversation text is untouched and the
original cache stays valid until overwritten, so it's reversible. The boundary
message is deferred to report the API's true `cache_deleted_input_tokens`.

## Rung 3 — Context collapse

`CONTEXT_COLLAPSE`-gated. A **read-time projection** over the REPL's full history:
it folds granular context into a collapsed view without mutating the stored
messages (so it can be re-expanded). Running it before autocompact means the
expensive lossy summary is often avoided. On a 413, its staged collapses can be
*drained* as the first recovery step (`query.ts:1089-1117`).

## Rung 4 — Session-memory compact (`src/services/compact/sessionMemoryCompact.ts`)

The "preserve more, lose less" alternative to full compaction. Gated by **both**
`tengu_session_memory` and `tengu_sm_compact` (`:403-432`), with env overrides.
Config (`:47-61`): `minTokens: 10_000`, `minTextBlockMessages: 5`,
`maxTokens: 40_000`.

`trySessionMemoryCompaction` (`:514-630`):
- waits for the async session-memory extraction (`:527`, see
  [10](10-memory-and-context.md)) and bails if memory is empty (`:533-540`);
- keeps recent messages verbatim, expanding backward from `lastSummarizedIndex`
  to satisfy `minTokens`/`minTextBlockMessages` unless `maxTokens` is hit;
- **preserves API invariants** (`adjustIndexToPreserveAPIInvariants`, `:232-314`):
  backtracks to include the `tool_use` for any kept `tool_result`, and the
  `thinking` block sharing a message id with a kept assistant;
- **falls through to full compaction** if the result still exceeds the
  autocompact threshold (`:604-613`) — i.e. it only takes the cheap path when the
  cheap path actually helps.

It's non-lossy/reversible because the originals stay in the transcript; session
memory is a write-only archive.

## Rung 5 — Autocompact (`src/services/compact/autoCompact.ts`)

The proactive, lossy summary.

### Threshold math (`:33-91`)

```
effectiveWindow = contextWindow(model) − min(maxOutputTokens(model), 20_000)   // :33-49
autocompactThreshold = effectiveWindow − 13_000                                // :72-91
```

`MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000` (`:30`, *"p99.99 of summary is 17,387
tokens"*); `AUTOCOMPACT_BUFFER_TOKENS = 13_000` (`:62`). Both have env overrides
for testing.

### `shouldAutoCompact` skip conditions (`:160-239`)

Skips when `querySource` is `session_memory`/`compact` (recursion guard,
`:171-173`) or the collapse agent (`:180`), when disabled (`:185`), when
`REACTIVE_COMPACT` mode owns overflow (`:195-199`), and when context collapse is
enabled (`:220-222`). The trigger subtracts `snipTokensFreed` from the token
count before comparing (`:225`).

### `autoCompactIfNeeded` (`:241-351`)

- **Circuit breaker** (`:257-265`): after
  `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` (`:70`) it gives up. The comment
  cites the incident: *"1,279 sessions had 50+ failures, wasting 250K API
  calls/day."*
- **Session-memory first** (`:287-310`): tries `trySessionMemoryCompaction`
  (rung 4) and only falls through to full `compactConversation` if that returns
  null — cheap before expensive, even inside autocompact.

## Rung 6 — Full / reactive compaction (`src/services/compact/compact.ts`)

`compactConversation` (`:387-763`) generates the lossy summary via a dedicated
prompt and rebuilds the message list.

### The prompt (`src/services/compact/prompt.ts`)

`NO_TOOLS_PREAMBLE` (`:19-26`) is emphatic — *"CRITICAL: Respond with TEXT ONLY.
Do NOT call any tools… Tool calls will be REJECTED and will waste your only
turn."* The body (`:61-143`) asks for a 9-section summary (primary intent, key
concepts, files & code, errors & fixes, pending tasks, current work, next
step…), drafted in an `<analysis>` block then a `<summary>` block.
`formatCompactSummary` (`:311-335`) strips the `<analysis>` scratchpad and
reformats `<summary>` for re-injection.

### What survives the boundary (`:331-337`, `:524-585`)

`buildPostCompactMessages` orders: boundary marker → summary → kept messages →
attachments → hook results. After compaction the system **re-injects**
selected context: recent file attachments (≤5 files / 50k tokens), delta
attachments for deferred tools/agents/MCP, and recent skills (5k/skill, 25k
total). Notably it does **not** reset `sentSkillNames` — re-injecting the full
skill listing is *"pure cache_creation with marginal benefit"* (`:524-529`).
Discovered tool names are carried forward on the boundary (`:606-611`).

### Reactive variant

Triggered by a withheld 413 (or media-size error) in the loop. `tryReactiveCompact`
is single-shot per turn, guarded by `hasAttemptedReactiveCompact` so a
still-too-long retry surfaces the error instead of spiraling
(`query.ts:1119-1175`, [14a](14a-deep-dive-agent-loop.md)). For media errors it
strip-retries oversized images.

## Post-compact cleanup (`src/services/compact/postCompactCleanup.ts`)

`runPostCompactCleanup` (`:31-77`) resets state so stale caches don't survive a
boundary — but **only for the main thread** (`repl_main_thread`/`sdk`), not for
`agent:*` subagents that share the process (`:22-29`). It clears microcompact
state (`:41`), context-collapse store (`:47`), memoized `CLAUDE.md`/memory caches
(`:59-60`), system-prompt sections, classifier approvals, and speculative checks
— while deliberately **not** clearing invoked-skill content (re-injection is
cheaper than re-listing).

## API-side context management (`src/services/compact/apiMicrocompact.ts`)

Orthogonal to the client ladder, `getAPIContextManagement` (`:64-153`) can ask
the *server* to edit context: `clear_thinking_20251015` (drop old thinking) and,
ANT-only, `clear_tool_uses_20250919` triggered at ~180k input tokens, clearing
down toward ~40k by dropping clearable tool inputs (`:104-126`). This complements
the client strategies for very long sessions.

## The mental model

Think of context pressure as a budget you pay down with progressively more
painful instruments:
1. **Free & reversible** first — drop ancient history (snip), dedupe via cache
   edits (cached MC), fold granular context (collapse).
2. **Cheap & lossless** next — archive to session memory and keep recent turns
   verbatim, but only if it actually gets you under threshold.
3. **Expensive & lossy** last — summarize the whole conversation, proactively
   (autocompact) or as a 413 rescue (reactive compact).

Every rung is guarded so the strategies don't fight (recursion guards, "skip if
the lossless rung already solved it") and so failures can't spiral (circuit
breaker, single-shot reactive compact, latched guards). The ordering *is* the
design: spend information loss only when nothing cheaper will fit.

This completes the Phase 2 deep-dives. Next: Phase 3 — end-to-end data-flow
walkthroughs.
