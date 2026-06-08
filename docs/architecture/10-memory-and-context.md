# 10 — Memory & Context

Claude Code maintains **persistent, file-based memory** that survives across
sessions, plus automatic extraction and consolidation so the agent learns over
time without bloating its context window.

## The memory directory (`src/memdir/`)

Memories live per project under:

```
~/.claude/projects/<project-slug>/memory/
  ├── MEMORY.md          # index: one line per memory (≤200 lines / 25KB)
  ├── user_*.md          # who the user is, preferences
  ├── feedback_*.md      # how to work; behaviors to repeat/avoid
  ├── project_*.md       # codebase architecture, patterns, conventions
  ├── reference_*.md      # external docs/resources (NO secrets)
  └── subdirs/           # optional semantic grouping
```

`paths.ts` resolves project, team, and auto (cross-project) memory locations
(`getAutoMemPath`, `isAutoMemoryEnabled` — GrowthBook-gated). Memory types and
their guidance live in `memoryTypes.ts`.

**Each memory file** has frontmatter (`name`, `description`, `type`, optional
`scope: private | team`) followed by the fact. The **index** `MEMORY.md` is
always loaded into context (it's small, capped) so the model can discover which
memory files to read. `truncateEntrypointContent()` (`memdir.ts:57`) enforces
the 200-line / 25 KB cap (line-truncate then byte-truncate at a newline) and
appends a warning when truncation occurs.

**What not to store** (enforced by guidance, `memdir.ts:199-250`): no large
source files, no secrets/tokens, no fast-changing state (build results, git
history), no obvious duplicates.

## Session memory — mid-session extraction (`src/services/SessionMemory/`)

`sessionMemory.ts` extracts key facts from the live conversation without
interrupting the main loop. `shouldExtractMemory()` (`:134`) fires only when
all gates pass:

1. context has grown past an initialization threshold;
2. enough tokens since the last extraction (`minimumTokensBetweenUpdate`);
3. enough tool calls since the last update;
4. the conversation is at a natural break (last assistant turn had no tool
   calls).

Extraction runs as a **forked background subagent** (`:183-250`) using a
`/dream`-style prompt with Read/Write tools only (Bash disabled), appending
extracted points to the session memory file. Config (`sessionMemoryUtils.ts`):
`enabled`, `minimumTokensForInitialization`, `minimumTokensBetweenUpdate`,
`toolCallsBetweenUpdates`, `maxExtractionTokens`.

Session memory also feeds a cheaper compaction path
(`compact/sessionMemoryCompact.ts`) that is tried before full compaction (see
[03](03-query-engine.md)).

## Auto-dream — background consolidation (`src/services/autoDream/`)

`autoDream.ts` periodically reviews accumulated transcripts and consolidates
durable memories — the `/dream` workflow run automatically. Three gates:

1. **Time** — ≥ `minHours` since last consolidation (default 24).
2. **Sessions** — ≥ `minSessions` sessions touched since (default 5).
3. **Lock** — no concurrent consolidation (`tryAcquireConsolidationLock`).

A scan throttle (`SESSION_SCAN_INTERVAL_MS = 10 min`, `:56`) avoids rapid
re-scans. Execution (`:224-248`): `buildConsolidationPrompt(...)` →
`runForkedAgent(..., querySource: 'auto_dream')` → create/improve memory files →
append an "Improved N memories" note to the transcript →
`completeDreamTask`/`failDreamTask`.

The **consolidation lock** (`consolidationLock.ts`) is a file whose **mtime**
doubles as the "last consolidated at" timestamp; on kill it rewinds the mtime
(abort semantics). Config comes from the `tengu_onyx_plover` GrowthBook gate.

## On-demand extraction (`src/services/extractMemories/`)

`extractMemories.ts` powers `/consolidate-memory`: a forked subagent with
Read/Write/Edit (Bash read-only) that extracts and improves memories from recent
history, then appends "Saved N memories" to the transcript. Prompts live in
`prompts.ts`.

## Team memory sync (`src/memdir/teamMemPaths.ts`, `src/services/teamMemorySync/`)

When `TEAMMEM` is enabled, memories with `scope: team` are shared across users
via `~/.claude/team-memory/`. The service watches local edits and syncs them to
the shared store. A **secret scanner** (`secretScanner.ts`,
`teamMemSecretGuard.ts`) prevents secrets from being written to shared memory
(also enforced by the file tools — see [04](04-tool-system.md)).

## Memory in the system prompt

The memory mechanics prompt is injected when
`CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` is set (see [03](03-query-engine.md)). It
instructs the model on the two-step save protocol: write
`<topic>.md` with frontmatter + content, then add a one-line pointer to
`MEMORY.md`. Because `MEMORY.md` is always in context, the model can locate
relevant memories without reading every file.

## Other generated docs

- **`src/services/MagicDocs/`** — LLM-generated documentation.
- **`src/services/AgentSummary/`**, **`toolUseSummary/`** — summaries of agent
  runs and tool usage threaded back into the loop.

## Key files

| File | Role |
|---|---|
| `src/memdir/memdir.ts` | memory dir mgmt, entrypoint truncation, guidance |
| `src/memdir/memoryTypes.ts` | memory type definitions |
| `src/memdir/paths.ts` / `teamMemPaths.ts` | memory locations |
| `src/services/SessionMemory/sessionMemory.ts` | mid-session extraction |
| `src/services/autoDream/autoDream.ts` | background consolidation |
| `src/services/autoDream/consolidationLock.ts` | mtime-based lock |
| `src/services/extractMemories/extractMemories.ts` | `/consolidate-memory` |
| `src/services/teamMemorySync/*` | shared team memory + secret scanning |
