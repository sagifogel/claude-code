# 15b — Walkthrough: A Skill That Forks a Sub-Agent

A skill declared with `context: fork` (or an explicit `AgentTool` call) runs in
an **isolated sub-agent** rather than expanding inline into the current
conversation. This trace follows a forked skill and the background-agent variant.
See [08](08-bridge-remote-multiagent.md) for the multi-agent overview.

## The trace

### 1 — Command resolution
`parseSlashCommand("/name args")` (`src/utils/slashCommandParsing.ts:25`) →
`findCommand(commandName, commands)` (`src/commands.ts:688`) returns the skill's
`PromptCommand`.

### 2 — `context: fork` comes from the skill frontmatter
When the skill was loaded, `loadSkillsDir.ts` mapped its frontmatter:
`executionContext: frontmatter.context === 'fork' ? 'fork' : undefined`
(`src/skills/loadSkillsDir.ts:260`), threaded into the command as `context`
(`createSkillCommand`, `:316-331`). **`inline`** (default) expands the skill into
the current turn; **`fork`** runs it in a separate agent with its own context,
token budget, and isolated permissions ([05](05-command-and-ui-system.md),
`src/types/command.ts:42-48`).

### 3 — SkillTool routes on context
`SkillTool.call({ skill, args }, …)` (`src/tools/SkillTool/SkillTool.ts:580`)
branches: `if (command?.type === 'prompt' && command.context === 'fork') return
executeForkedSkill(...)` (`:622`).

### 4 — Prepare the forked context
`executeForkedSkill` (`:122`) calls `prepareForkedCommandContext(...)` (`:205`)
which uses `createSubagentContext(parentContext, …)`
(`src/utils/forkedAgent.ts:345`). The isolation is concrete:
- fresh mutable state (read-file cache, discovery, content-replacement) — `:376-406`;
- `shouldAvoidPermissionPrompts: true` (`:371`);
- a **new agent id** `createAgentId()` (`:448`);
- a **new query chain** `{ chainId: randomUUID(), depth: parentDepth + 1 }`
  (`:452-454`).

### 5 — Run the sub-agent
`executeForkedSkill` consumes `runAgent({ agentDefinition, promptMessages,
toolUseContext: { …, getAppState: modifiedGetAppState }, isAsync: false, … })`
(`SkillTool.ts:223`). `isAsync: false` means the fork runs **synchronously** from
the caller's view — it blocks the parent tool call until done. The skill's
`effort` is merged into the agent definition (`:209-212`).

### 6 — The sub-agent's own loop
`runAgent` (`src/tools/AgentTool/runAgent.ts:200`) is itself an async generator
that imports and drives `query` (`runAgent.ts:15`) — i.e. **the sub-agent runs
the same `queryLoop`** ([14a](14a-deep-dive-agent-loop.md)) recursively, with its
isolated context. Messages stream into a local buffer (`agentMessages.push`,
`SkillTool.ts:237`).

### 7 — Extract the result
`extractResultText(agentMessages, …)` (`SkillTool.ts:264`) pulls the sub-agent's
final text and returns it as the skill's tool result (`:276-284`), which flows
back to the parent loop like any other tool result.

## The background variant (`run_in_background`)

`AgentTool` (rather than a forked skill) can detach the agent entirely:

### B1 — Decide async
`AgentTool.call()` computes `shouldRunAsync` from `run_in_background`, the agent's
`background` flag, coordinator mode, etc. (`src/tools/AgentTool/AgentTool.tsx:567`).

### B2 — Register the task and return immediately
`registerAsyncAgent({...})`
(`src/tasks/LocalAgentTask/LocalAgentTask.tsx:466`) creates a
`LocalAgentTaskState` with `status: 'running'`, `isBackgrounded: true` (`:487-504`),
`registerTask(...)` into `AppState.tasks` (`:513`), and the tool **returns the
task handle right away**.

### B3 — Fire-and-forget execution
`void runWithAgentContext(…, () => runAsyncAgentLifecycle({...}))`
(`AgentTool.tsx:733`) — the `void` makes it detached. The lifecycle
(`agentToolUtils.ts:508`) consumes the agent stream
(`for await (const message of makeStream(...))`, `:554`), updating task state as
it goes, then `finalizeAgentTool(...)` + `completeAsyncAgent(...)` (`:597-603`).

### B4 — Delivery via task-notification
On completion, `enqueueAgentNotification({ status: 'completed', finalMessage,
usage, … })` (`agentToolUtils.ts:624`) builds a `<task-notification>` XML message
(`LocalAgentTask.tsx:197-257`):

```xml
<task-notification>
  <task-id>…</task-id><status>completed</status>
  <summary>Agent "…" completed</summary>
  <result>…final text…</result>
  <usage>…</usage>
</task-notification>
```

It's `enqueuePendingNotification(...)` (`messageQueueManager.ts:142`) onto the
command queue, then **the main loop dequeues it** (`drainCommandQueue`,
`cli/print.ts:1935`), parses the XML, and feeds it to the parent via `ask(...)`
as a user message — so the parent agent learns the outcome and can act on it.

## What this illustrates

- **`fork` = a nested `queryLoop`** with an isolated `ToolUseContext`. The same
  engine runs at every depth; `queryTracking.depth` records nesting.
- **Sync vs. background** is one flag: a forked skill blocks the parent tool
  call; a backgrounded agent returns a task handle and reports back later.
- **The task-notification XML is the swarm's message bus** — the same mechanism
  the coordinator uses ([08](08-bridge-remote-multiagent.md)) to collect worker
  results and resume them with synthesized follow-ups.
