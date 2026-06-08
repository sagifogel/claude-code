# 03 — Query Engine & Agent Loop

The query engine is the heart of Claude Code: it turns a user prompt into a
streaming conversation with the Claude API, executes tools in a loop, manages
context size, recovers from errors, and tracks cost. It is built as a stack of
composed async generators.

```
ask()                        convenience wrapper (one-shot)
  └─ QueryEngine.submitMessage()   session orchestration, system prompt, transcript
       └─ query()                  per-turn setup
            └─ queryLoop()         the actual agent loop (while true)
                 └─ deps.callModel()   streaming Anthropic API call
```

## Entry points

- **`ask(prompt, options)`** (`QueryEngine.ts:1186-1295`) — instantiates a
  `QueryEngine`, calls `submitMessage`, yields its messages, and on completion
  persists the read-file cache.

- **`QueryEngine.submitMessage()`** (`QueryEngine.ts:209-1156`) — an
  `async *` generator and the main entry for both the interactive REPL and SDK
  callers. Responsibilities:
  - Assemble the system prompt by concatenating
    `[customPrompt | defaultSystemPrompt, memoryPrompt?, appendSystemPrompt?]`
    (`:321-325`). Memory mechanics are injected when
    `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` is set.
  - `processUserInput()` parses slash commands, attachments, and model
    overrides, and updates the `ToolPermissionContext`.
  - **Persist the transcript to disk before entering the loop** (`:451`) so
    sessions are resumable even if the process dies mid-query.
  - Yield a `buildSystemInitMessage()` (tools, agents, plugins, fast-mode state).
  - **No-query fast path** (`:556-639`): purely local slash commands
    (`/compact`, `/model`, …) produce synthetic assistant output without an API
    call.
  - Drive the `query()` loop and record each streamed message
    (assistant messages recorded fire-and-forget to keep the stream flowing;
    others awaited to preserve order).

## The agent loop — `query.ts` / `queryLoop()`

`queryLoop` (`src/query.ts`, ~1,730 lines) is a `while (true)` loop. Each
iteration carries a `State` object (`query.ts:204-217`):

```ts
type State = {
  messages, toolUseContext,
  maxOutputTokensOverride?, maxOutputTokensRecoveryCount,
  hasAttemptedReactiveCompact, autoCompactTracking?,
  stopHookActive?, pendingToolUseSummary?,
  turnCount, transition?,   // 'collapse_drain_retry' | 'reactive_compact_retry' | …
}
```

A single iteration does:

### 1. Pre-API message preparation

Run in this deliberate order so cheaper reductions can avoid expensive ones:

1. **Snip** (feature `HISTORY_SNIP`, `:401-410`) — bounded history truncation
   for long headless sessions; yields a boundary message with the token delta.
   (REPL projects full history on demand; only SDK truncates.)
2. **Microcompact** (`:413-426`) — client-side deduplication/caching of tool
   results. With `CACHED_MICROCOMPACT`, the boundary message is deferred until
   after the API response so it can report the true `cache_deleted_input_tokens`.
3. **Context collapse** (feature `CONTEXT_COLLAPSE`, `:440-447`) — a read-time
   projection that folds granular context without mutating messages; runs
   *before* autocompact so that if it gets under threshold, autocompact is a
   no-op.
4. **Autocompact** (`:453-543`) — proactive summarization before the context
   limit (see "Compaction" below).

### 2. Blocking-limit check (`:592-648`)

If context is at the hard limit, the loop returns a synthetic error instead of
calling the API — *except* when a compaction/snip just changed the counts, or
when reactive-compact / context-collapse are enabled (those own overflow), to
avoid deadlocks.

### 3. Model selection (`:572-578`)

`getRuntimeMainLoopModel({ permissionMode, mainLoopModel, exceeds200kTokens })`.
In `plan` mode, the model can downgrade Opus→Sonnet when the most recent
assistant message exceeds 200k tokens.

### 4. Streaming API call + tool execution (`:659-1049`)

```ts
for await (const message of deps.callModel({ messages: prependUserContext(...),
                                             systemPrompt: fullSystemPrompt, … })) {
  // backfill observable inputs on tool_use blocks
  // withhold recoverable errors (prompt_too_long / media_size / max_output_tokens)
  if (message.type === 'assistant') {
    collect tool_use blocks
    streamingToolExecutor?.addTool(block, message)   // execute DURING streaming
  }
  yield completed tool results as they finish
  if (!withheld) yield message
}
```

The **`StreamingToolExecutor`** is the key performance trick: as soon as a
`tool_use` block arrives, the tool is queued for execution while the model keeps
streaming. By the time streaming ends, results are often already done — turning
sequential round-trips into parallel work. (Details in [04](04-tool-system.md).)

### 5. Error recovery (layered)

Recoverable API errors are *withheld* from the stream and handled in priority
order; each path either fixes the problem and `continue`s the loop, or surfaces
the error and returns:

- **Prompt too long (413)** — first drain staged context-collapses
  (`transition='collapse_drain_retry'`), then try reactive compaction
  (`tryReactiveCompact`, `transition='reactive_compact_retry'`); if both fail,
  surface the error (`:1070-1183`).
- **Max output tokens** — escalate the cap once to 64k
  (`ESCALATED_MAX_TOKENS`), then up to 3 multi-turn "resume directly" recovery
  messages, then surface (`:1185-1256`).
- **Media size** — strip oversized media via reactive compact.

### 6. Stop hooks, token budget, continuation (`:1267-1341`)

- **Stop hooks** can prevent continuation or inject blocking errors that
  re-trigger a query.
- **Token budget** (feature `TOKEN_BUDGET`) can inject a nudge message to
  continue within budget, or emit a completion event.
- A **tool-use summary** (small/fast model) may be generated asynchronously per
  turn and threaded into the next iteration (skipped for sub-agents).

## Context assembly

System and user context are memoized for the session (`src/context.ts`):

- **`getSystemContext()`** (`:116-150`) — git branch/status/recent commits
  (unless `DISABLE_GIT_INSTRUCTIONS` / remote), plus an optional cache-breaker
  injection. Injected into the system prompt via `appendSystemContext`.
- **`getUserContext()`** (`:155-189`) — auto-discovered `CLAUDE.md` files and
  `Today's date is …`. **Prepended** to messages via `prependUserContext`.

## Compaction (`src/services/compact/`)

Several complementary strategies keep context within the window:

| Strategy | File | When |
|---|---|---|
| **Snip** | `compact/` (gated `HISTORY_SNIP`) | SDK long sessions; truncate old history |
| **Microcompact** | `microCompact.ts` / `apiMicrocompact.ts` | dedupe/cache tool results every turn |
| **Context collapse** | `compact/` (gated `CONTEXT_COLLAPSE`) | fold granular context (reversible) |
| **Autocompact** | `autoCompact.ts` | proactive summarization near the limit |
| **Reactive compact** | `compact.ts` | recovery after a 413/media error |
| **Session-memory compact** | `sessionMemoryCompact.ts` | cheaper compaction that preserves more detail; tried before full compaction |

**Autocompact** (`autoCompact.ts`): threshold ≈ `effectiveContextWindow - 13k`
buffer (overridable via `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`). `shouldAutoCompact`
skips when disabled, for `session_memory`/`compact` query sources (recursion
guard), and when reactive/collapse modes own overflow. Execution
(`autoCompactIfNeeded`) has a circuit breaker (skip after 3 consecutive
failures), tries session-memory compaction first, then full
`compactConversation`, then runs `postCompactCleanup` (resets collapse state,
invalidates memdir caches, notifies cache-break detection).

## API client & retries (`src/services/api/`)

`deps.callModel` (`services/api/claude.ts`, ~3,400 lines) normalizes messages,
assembles betas (prompt caching, thinking, structured outputs, effort, fast
mode, task budgets), selects the provider client, and wraps the call in
`withRetry`.

### Multi-provider abstraction (`services/api/client.ts:88-150`)

`getAnthropicClient()` selects a provider and auth source:

- **First-party** — OAuth (claude.ai) preferred; else API key (incl.
  `apiKeyHelper` script).
- **Bedrock** (`CLAUDE_CODE_USE_BEDROCK`) — `@anthropic-ai/bedrock-sdk`, AWS
  credential chain, region (with a small-fast-model region override).
- **Vertex** (`CLAUDE_CODE_USE_VERTEX`) — `@anthropic-ai/vertex-sdk`,
  `google-auth-library`, per-model regions.
- **Foundry** (`CLAUDE_CODE_USE_FOUNDRY`) — `@anthropic-ai/foundry-sdk`, API key
  or `DefaultAzureCredential`.

Provider SDKs are **dynamically imported** so unused ones add no cost. Custom
headers include `x-app: cli`, the session id, and client-app identifiers.

### Retry strategy (`services/api/withRetry.ts:170-822`)

- Exponential backoff with jitter; honors `Retry-After`.
- **Auth recovery** — 401 "expired" / 403 "revoked" → `handleOAuth401Error`
  refreshes tokens and retries; Bedrock/Vertex credential refresh handled too.
- **Stale connection** — ECONNRESET/EPIPE on keep-alive sockets → retry with
  keep-alive disabled.
- **Context overflow at request time** — parses
  `"input length and max_tokens exceed context limit"` and lowers the
  `max_tokens` ceiling, then retries.
- **Fast-mode fallback** — on 429/529 with fast mode, short `Retry-After` →
  wait and retry at fast speed (preserves prompt cache); long → enter cooldown
  and switch to standard speed; overage-disabled header → permanently disable.
- **Model fallback** — 3 consecutive 529s → `FallbackTriggeredError`; the loop
  retries with `fallbackModel` (Sonnet/Haiku), stripping thinking signatures if
  needed.
- **Foreground-only 529 retries** — only certain query sources
  (`repl_main_thread*`, `sdk`, `agent:*`, `compact`, …) retry on 529; background
  queries (summaries/classifiers) bail to avoid cascades (`:62-89`).
- **Persistent/unattended retry** (`CLAUDE_CODE_UNATTENDED_RETRY`) — retry
  429/529 indefinitely with 30s heartbeat chunks (yielding status messages) so
  unattended sessions don't time out.

## Special API features

- **Prompt caching** — per-request `cache_control: { type: 'ephemeral', ttl?,
  scope? }`; 1h TTL and global scope are eligibility-gated.
- **Extended thinking** — `disabled` / fixed-budget / `adaptive`; signature
  blocks are model-specific and stripped on fallback to an unprotected model.
- **Structured output** — JSON-schema output wrapped via a synthetic output
  tool with bounded retries.
- **Fast mode** — reduced-latency Sonnet; gated, with cooldown and overage
  handling.

## Cost tracking (`src/cost-tracker.ts`, `src/costHook.ts`)

Usage is re-accumulated from streaming `message_start` / `message_delta` /
`message_stop` events (not just the final snapshot). `addToTotalSessionCost`
(`cost-tracker.ts:278-323`) updates session totals + per-model breakdown +
OpenTelemetry counters (annotating fast-mode), and **recursively** adds advisor
sub-call costs. Per-model aggregation rolls up by canonical model name with
cache-read/creation token tracking.

## Message handling (`src/utils/messages.ts`, ~5,500 lines)

- Creation helpers: `createAssistantMessage`, `createUserMessage`,
  `createAssistantAPIErrorMessage`, `createProgressMessage`,
  `createAttachmentMessage`.
- **`normalizeMessages`** (`:730-823`) splits multi-block messages into
  single-block messages, assigning stable derived UUIDs (`deriveUUID(parent,
  index)` encodes the index in the last 12 hex chars) so resumed sessions keep
  consistent IDs.
- **`ensureToolResultPairing`** inserts synthetic placeholders for any
  `tool_use` lacking a `tool_result` (and marks them so they poison training
  data rather than being submitted).
- **`reorderMessagesInUI`** (`:854-1026`) groups tool-use → pre-hooks →
  tool-result → post-hooks so results never render before their call.

## Key files

| File | Role |
|---|---|
| `src/QueryEngine.ts` | `submitMessage` orchestration, system prompt, transcript |
| `src/query.ts` | `queryLoop` agent loop, recovery, tool execution |
| `src/context.ts` | system/user context assembly |
| `src/services/api/claude.ts` | `callModel`: betas, normalization, provider call |
| `src/services/api/client.ts` | multi-provider client factory |
| `src/services/api/withRetry.ts` | retry/fallback/auth-refresh logic |
| `src/services/compact/*` | snip / micro / collapse / auto / reactive compaction |
| `src/cost-tracker.ts` | per-model token & cost accounting |
| `src/utils/messages.ts` | message creation, normalization, ordering |
