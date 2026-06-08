# 15a — Walkthrough: A Prompt That Edits a File

The canonical end-to-end path. The user types *"fix the typo in config.ts"*, the
model responds with a `FileEdit` tool call, the edit lands on disk, and the
result renders. This trace ties together [03](03-query-engine.md),
[04](04-tool-system.md), [07](07-permissions-and-hooks.md), and
[14a](14a-deep-dive-agent-loop.md).

> Citations are precise; the `query.ts` references are verified against the
> source (see [14a](14a-deep-dive-agent-loop.md)), the surrounding chain is from
> targeted code reads.

## The trace

### 1 — Input leaves the prompt box
`PromptInput.tsx` `onSubmit` fires on Enter and calls the `onSubmit` prop
(`src/components/PromptInput/PromptInput.tsx:1100`).

### 2 — Submission routing
`handlePromptSubmit()` (`src/utils/handlePromptSubmit.ts:120`) receives the raw
string and calls `executeUserInput(...)` (`:368`), which loops over queued
commands calling `processUserInput(...)` (`:476`).

### 3 — Input → message blocks
`processUserInput` (`src/utils/processUserInput/processUserInput.ts`) detects
this is *not* a slash command, so `processTextPrompt(...)` (`:578`) turns the
text (plus any `@file` references / pasted images) into user `Message` blocks.

### 4 — Kick off the query
`handlePromptSubmit` calls `onQuery(newMessages, abortController, …)` (`:560`).
In the REPL, `onQuery` (`src/screens/REPL.tsx:2855`) drives the generator:
`for await (const event of query({ messages, … }))` (`:2793`).

### 5 — The agent loop
`query()` (`src/query.ts:219`) delegates to `queryLoop()` (`:241`). The loop
prepares context (compaction ladder, [14d](14d-deep-dive-compaction.md)), selects
the model, and opens the streaming call `deps.callModel({...})` (`:659`).

### 6 — The model streams a `tool_use`
As the assistant message streams in, its `tool_use` block (name `FileEdit`,
input `{file_path, old_string, new_string}`) is detected (`:829-834`), sets
`needsFollowUp = true`, and is handed to the **streaming executor** *immediately*
— `streamingToolExecutor.addTool(toolBlock, message)` (`:842`) — so it begins
running while the model keeps streaming.

### 7 — Tool dispatch
`addTool` (`src/services/tools/StreamingToolExecutor.ts:76`) queues and
`processQueue()`s the tool → `runTools` (`toolOrchestration.ts:19`) →
`runToolUse` (`toolExecution.ts:337`) →
`streamedCheckPermissionsAndCallTool` (`:492`) → `checkPermissionsAndCallTool`
(`:599`).

### 8 — Validation, hooks, permission
`checkPermissionsAndCallTool` validates the input (`:615`), runs **PreToolUse
hooks** (`:800`), then evaluates permission. `FileEdit` is a writer, so unless a
rule pre-allows it, this is where a permission prompt would appear
(see [15f](15f-walkthrough-permission-denial.md)). On allow, it calls
`tool.call(...)`.

### 9 — The edit executes
`FileEditTool.call()` (`src/tools/FileEditTool/FileEditTool.ts:387`):
- resolve path: `expandPath(file_path)` (`:402`);
- read current contents `readFileForEdit(...)` (`:444`) after `validateInput`
  enforced old≠new, the 1 GiB size cap, and UNC-path rejection;
- **write to disk**: `writeTextContent(absoluteFilePath, updatedFile, encoding,
  endings)` (`:491`) — preserving original encoding and line endings;
- **side effects**: clear LSP diagnostics for the file (`:497`), notify the LSP
  of the change + save (`:500`, `:508`), notify the VS Code SDK
  (`notifyVscodeFileUpdated`, `:517`), update the read-file timestamp so a later
  edit can't clobber a stale read (`:520-525`);
- build the `structuredPatch` and count lines changed (`:531`), and return
  `{ data }` (`:571`).

### 10 — Result streams back
The result is enqueued (`toolExecution.ts:558`), yielded up through
`runToolUse` → `runTools`, and surfaces in the loop as a `tool_result` message
(`query.ts:1384-1386`). Because tools ran, the loop `continue`s and re-queries
the model with the result appended — the model can now confirm the fix.

### 11 — Render
Each yielded event reaches `onQueryEvent` (`REPL.tsx:2802`), which updates
message state and triggers the Ink render pipeline
([14b](14b-deep-dive-ink-rendering.md)) — the user sees the tool call, a diff,
and the assistant's confirmation.

### 12 — Persist
The transcript is written via `recordTranscript` so the turn survives a crash
and is resumable (see [15d](15d-walkthrough-session-resume.md)).

## What this illustrates

- **The generator spine** ([01](01-overview.md) §1): input → `submitMessage` →
  `query` → `queryLoop` → `callModel`, each yielding upward so the UI streams.
- **Execute-while-streaming** ([04](04-tool-system.md)): the edit starts at step
  6, before the assistant message even finishes.
- **The permission boundary** (step 8) sits *between* the model proposing a tool
  and the tool touching disk — every writer passes through it.
- **Side effects are part of "edit"** (step 9): an edit isn't just a write, it's
  LSP sync + IDE notify + read-state bookkeeping + skill activation.
