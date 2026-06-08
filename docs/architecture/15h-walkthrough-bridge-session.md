# 15h — Walkthrough: A Remote / IDE-Bridge Session

A tool call originates on a remote ("CCR") session or an IDE extension, but the
**permission prompt renders locally** and the decision is sent back. This is the
runtime view of [08](08-bridge-remote-multiagent.md).

## Two related topologies

- **IDE bridge** (`src/bridge/`): a VS Code / JetBrains extension drives a CLI
  session over a WebSocket.
- **Remote / CCR** (`src/remote/`): the agent runs on a server; the local REPL
  subscribes and approves.

Both share the same shape: the model + tools run "over there," messages stream
"over here," and `control_request`s ferry permission decisions across.

## The trace (remote/CCR)

### 1 — Establish the session
`RemoteSessionManager.connect()` (`src/remote/RemoteSessionManager.ts:108`)
opens a `SessionsWebSocket` (`src/remote/SessionsWebSocket.ts:82`) to
`/v1/sessions/ws/{sessionId}/subscribe`, authenticated with an OAuth bearer
token, with reconnect/backoff (2 s → 120 s) and 30 s pings.

### 2 — Stream server messages locally
Server `SDKMessage`s arrive over the socket and are converted to the REPL's
internal `Message` types by `sdkMessageAdapter.ts`
(`convertAssistantMessage`, `convertStreamEvent`, `convertResultMessage`,
`convertStatusMessage`) — so the local UI renders them with the normal Ink
pipeline ([14b](14b-deep-dive-ink-rendering.md)).

### 3 — A tool needs permission → `control_request`
When the remote agent wants to run a tool, the server sends a
`control_request` (`can_use_tool`). `RemoteSessionManager`'s control handler
(`:189-214`) stores it in `pendingPermissionRequests` and fires
`onPermissionRequest(request, requestId)`.

### 4 — Synthesize a local tool + message
The real `tool_use` lives on the server, so the local side fabricates just enough
to drive its permission UI:
- `createSyntheticAssistantMessage(...)`
  (`remotePermissionBridge.ts:12-46`) builds a local `AssistantMessage` carrying
  the `tool_use`;
- `createToolStub(...)` (`:53-78`) builds a minimal local `Tool` for tools the
  client doesn't know (e.g. a remote MCP tool) — its `call()` returns empty;
  it exists only so the prompt can render.

### 5 — Local permission prompt
This synthetic tool-use flows into the *same* interactive permission handler as
[15f](15f-walkthrough-permission-denial.md) — the user sees a normal prompt.

### 6 — Decision returns over the wire
On resolve, the bridge sends the decision back: for the IDE bridge,
`BridgePermissionCallbacks.sendResponse(requestId, response)`
(`bridgePermissionCallbacks.ts`) emits an `SDKControlResponse`; for CCR, the
manager replies on the WebSocket. The server then actually runs (or skips) the
tool and streams the result back to step 2.

### 7 — Cancellation
If the server cancels a pending prompt (e.g. the turn aborted), a
`control_cancel` arrives; `RemoteSessionManager` (`:159-172`) removes it from the
pending map and fires `onPermissionCancelled`, dismissing the local dialog.

## IDE bridge specifics (`src/bridge/`)

`bridgeMain.ts` manages the session lifecycle (`POST /v1/sessions`, poll, archive)
with multi-session spawn gated by `tengu_ccr_bridge_multi_session`.
`replBridge.ts` exposes `writeMessages` / `writeSdkMessages` /
`sendControlRequest/Response`. `bridgeMessaging.ts`'s `isEligibleBridgeMessage`
filters which internal messages forward to the editor (user/assistant/system
local-command only — not virtual messages, tool results, or progress). JWT
refresh on 401 is handled in `jwtUtils.ts`.

## What this illustrates

- **Execution and approval are decoupled**: tools run wherever the agent lives;
  the human-in-the-loop decision is always rendered where the human is.
- **`control_request`/`control_response` is the permission RPC** across the
  socket — the local UI doesn't need the real tool, just a *synthetic* stand-in
  to render the prompt and capture a yes/no.
- **The local renderer is provider-agnostic** ([13](13-design-decisions.md) §14):
  whether messages come from a local `queryLoop` or a remote socket, they're
  normalized to the same `Message` types and drawn by the same Ink pipeline.
