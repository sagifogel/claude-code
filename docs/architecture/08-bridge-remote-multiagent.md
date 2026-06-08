# 08 — Bridge, Remote Sessions & Multi-Agent Orchestration

This document covers how Claude Code runs *beyond* a single local terminal: the
IDE bridge, remote ("CCR") sessions, and the task/team/coordinator machinery for
multiple agents.

---

## A. IDE bridge (`src/bridge/`)

The bridge connects IDE extensions (VS Code / JetBrains) to the CLI over a
WebSocket, streaming messages and routing permission prompts back to the editor.

- **`bridgeMain.ts`** (~2,999 lines) — session lifecycle: create a session
  (`POST /v1/sessions`), establish the WebSocket, poll for work. Multi-session
  spawn (`--spawn`, `--capacity`, `--create-session-in-dir`) is gated by the
  GrowthBook flag `tengu_ccr_bridge_multi_session`. Connection failures use
  exponential backoff (cap 2 min for connect, 30 min general, 10 min give-up).
- **`replBridge.ts`** (~2,406 lines) — the REPL-side bridge implementation.
  `ReplBridgeHandle` exposes `writeMessages`, `writeSdkMessages`, and
  `sendControlRequest/Response`. `BridgeCoreParams` injects session
  create/archive, title refresh, SDK message conversion, an OAuth-401 handler,
  poll config, and initial-message replay. `onSetPermissionMode` validates mode
  transitions before applying them.
- **`bridgeMessaging.ts`** — `isEligibleBridgeMessage` filters which internal
  messages forward to the IDE (user/assistant/system local-command only;
  excludes virtual messages, tool results, progress). Type guards discriminate
  `SDKMessage` / `SDKControlRequest` / `SDKControlResponse`. `extractTitleText`
  derives a session title from user messages.
- **`bridgePermissionCallbacks.ts`** — `{ sendResponse(requestId, response),
  cancelRequest(requestId) }`, letting the interactive permission handler send
  decisions back to the editor via an `SDKControlResponse`.
- **`jwtUtils.ts`** — JWT/token refresh on 401.

---

## B. Remote sessions / CCR (`src/remote/`)

Remote ("Claude Code Remote") sessions run the agent on a server while the local
REPL renders and approves.

- **`SessionsWebSocket.ts`** — a persistent WebSocket to
  `/v1/sessions/ws/{sessionId}/subscribe`, authenticated with an OAuth bearer
  token. Streams `SDKMessage`s; reconnects with exponential backoff
  (2 s → 120 s, 600 s give-up) and retries on a session-not-found (`4001`) code
  for compaction-induced staleness; pings every 30 s.
- **`RemoteSessionManager.ts`** (`:95-242`) — bridges CCR ↔ REPL. Routes
  messages: `control_request` (e.g. `can_use_tool`) → store in
  `pendingPermissionRequests` and fire `onPermissionRequest`; `control_cancel` →
  remove + fire `onPermissionCancelled`; SDK messages → `onMessage`.
- **`remotePermissionBridge.ts`** — because the real tool-use message lives on
  the server, `createSyntheticAssistantMessage()` synthesizes a local
  `AssistantMessage` with the `tool_use`, and `createToolStub()` builds a minimal
  local tool object so unknown (e.g. remote MCP) tools can still flow through the
  local permission UI.
- **`sdkMessageAdapter.ts`** — converts CCR SDK message formats to the REPL's
  internal `Message` types (`convertAssistantMessage`, `convertStreamEvent`,
  `convertResultMessage`, `convertStatusMessage`).

Related: `src/ssh/` (SSH-remote sessions — probe/deploy a remote, forward an
auth socket), `src/server/` (server mode), `src/upstreamproxy/` (proxy/relay).

---

## C. Tasks, teams & the coordinator

### Task framework (`src/Task.ts`, `src/tasks.ts`)

Background work is modeled as **tasks**. Task types (`Task.ts:6-108`):
`local_bash`, `local_agent`, `remote_agent`, `in_process_teammate`,
`local_workflow`, `monitor_mcp`, `dream`. Status:
`pending → running → completed | failed | killed` (terminal states are fixed).
A `TaskHandle` id is prefixed by type (`a`=agent, `r`=remote, `t`=teammate,
`b`=bash, `m`=monitor, `w`=workflow, `d`=dream). `getAllTasks()` /
`getTaskByType()` (`tasks.ts:22-39`) mirror the tool-registry pattern for
polymorphic kill/dispatch.

### Sub-agents — `AgentTool` (`src/tools/AgentTool/AgentTool.tsx`)

Spawns an isolated sub-agent with its own token budget (`:62-120`). Input:
`description`, `prompt`, `subagent_type` (role), `model`, `run_in_background`,
and (multi-agent) `name` (addressable id), `team_name`, `mode` (permission
override), plus isolation via `worktree` (temp git checkout) or `remote` (CCR
container) and a `cwd` override (gated `KAIROS`). `runAsyncAgentLifecycle()`
registers the task and monitors completion via task notifications.

### Inter-agent messaging — `SendMessageTool` (`src/tools/SendMessageTool/`)

Routes a message to a recipient (`:67-87`): a teammate name, `*` (broadcast),
`uds:<socket>` (local peer), or `bridge:<session-id>` (remote). Beyond plain
text, it supports **structured control messages** (`:46-65`): `shutdown_request`,
`shutdown_response`, `plan_approval_response` — used for swarm coordination.

### Teams — `TeamCreateTool` (`src/tools/TeamCreateTool/`)

Creates a shared team context (`:37-49`): `team_name`, optional `description`
and `agent_type`. Returns `team_name`, `team_file_path`, `lead_agent_id`. Agents
on a team share state via the team file and collaborate via `SendMessage`. A
unique name is auto-generated (`generateWordSlug`) on collision.

### Coordinator mode (`src/coordinator/coordinatorMode.ts`)

A secondary orchestration mode, feature-gated (`COORDINATOR_MODE`) and enabled
via `CLAUDE_CODE_COORDINATOR_MODE` (`:36-110`). Its system prompt (`:111-369`)
encodes a ~multi-agent orchestration discipline:

- **Role** — the coordinator spawns workers, **synthesizes** their findings, and
  never delegates understanding. Tools: `AgentTool` (spawn), `SendMessageTool`
  (continue a worker), `TaskStopTool` (stop mid-flight), PR-subscription tools.
- **Workers** receive a filtered toolset (Bash, Read, Edit, MCP tools, Skills),
  the list of available MCP servers, and a scratchpad directory for durable
  cross-worker state.
- **Phases** (`:202-237`) — Research → Synthesis (coordinator) → Implementation →
  Verification.
- **Synthesis directive** (`:251-368`) — the coordinator must turn research into
  *specific* specs (file paths, line numbers, exact changes), not pass-through
  ("based on your findings, fix it" is explicitly bad).
- Workers return results as `<task-notification>` XML (`:142-184`) carrying
  task id, status, summary, result text, and token usage. The coordinator
  consumes these and resumes a worker with a synthesized follow-up via
  `SendMessage`.

### Orchestration flow (coordinator)

```
Coordinator → AgentTool(description, prompt, subagent_type)
  → runAsyncAgentLifecycle() spawns LocalAgentTask / RemoteAgentTask
  → task registered, gets task_id
  → worker runs with coordinator context (tools, MCP servers, scratchpad)
  → worker completes → emits <task-notification> XML (delivered as user message)
  → coordinator → SendMessageTool(to: task_id, message: synthesized_spec)
  → queuePendingMessage() resumes the worker with new context
```

### Buddy companion (`src/buddy/`)

An animated companion sprite (idle/thinking/celebrating/concerned states), an
Easter egg decoupled from the agent logic — `CompanionSprite.tsx`,
`companion.ts`, `sprites.ts`, `useBuddyNotification.tsx`.

## Key files

| File | Role |
|---|---|
| `src/bridge/bridgeMain.ts` | IDE bridge session lifecycle |
| `src/bridge/replBridge.ts` | REPL-side bridge |
| `src/bridge/bridgeMessaging.ts` | message eligibility & type guards |
| `src/remote/SessionsWebSocket.ts` | remote session WebSocket |
| `src/remote/RemoteSessionManager.ts` | CCR ↔ REPL routing |
| `src/remote/remotePermissionBridge.ts` | synthetic messages & tool stubs |
| `src/remote/sdkMessageAdapter.ts` | SDK → internal message conversion |
| `src/Task.ts` / `src/tasks.ts` | task framework & registry |
| `src/tools/AgentTool/AgentTool.tsx` | sub-agent spawning |
| `src/tools/SendMessageTool/SendMessageTool.ts` | inter-agent messaging |
| `src/tools/TeamCreateTool/TeamCreateTool.ts` | team creation |
| `src/coordinator/coordinatorMode.ts` | coordinator orchestration |
