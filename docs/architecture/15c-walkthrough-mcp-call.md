# 15c — Walkthrough: An MCP Tool Call

How a tool provided by an external **MCP server** gets discovered, connected,
invoked, permission-checked, and its (possibly huge) result handled. This ties
together [04](04-tool-system.md) and [06](06-services-layer.md).

## The trace

### 1 — Server tools become `Tool` objects
On connection, `fetchToolsForClient` (`src/services/mcp/client.ts:1743`) calls
`client.request({ method: 'tools/list' }, …)` (`:1752`) and maps each server tool
into a Claude `Tool` by spreading the `MCPTool` template:

```ts
return toolsToProcess.map((tool): Tool => ({
  ...MCPTool,
  name: buildMcpToolName(client.name, tool.name),   // mcp__server__tool  (:1768)
  mcpInfo: { serverName: client.name, toolName: tool.name },  // (:1774)
  isMcp: true,
  inputJSONSchema: tool.inputSchema,   // server-defined schema (:1813)
  async call(args, context, …) { … }   // overridden executor (:1833)
}))
```

The name pattern is `mcp__<server>__<tool>` (`buildMcpToolName`,
`mcpStringUtils.ts:50`).

### 2 — Discovery when deferred
If MCP tools are deferred, `ToolSearchTool`
(`src/tools/ToolSearchTool/ToolSearchTool.ts:328`) surfaces them, and — crucially
— reports servers that aren't connected yet by filtering
`AppState.mcp.clients` for `type === 'pending'` and returning
`pending_mcp_servers` (`:331-338`). The model learns it must wait for / trigger a
connection.

### 3 — Lazy connection
`connectToServer` (`client.ts:595`) establishes the transport — stdio / SSE /
HTTP / WS / in-process ([06](06-services-layer.md)) — and tracks connection state
(`pending` → `connected`). `MCPConnectionManager.tsx` owns this state on the
React side.

### 4 — Permission is *passthrough*
The wrapped tool's `checkPermissions()` returns
`{ behavior: 'passthrough' }` (`client.ts:1814-1816`) — Claude Code does **not**
adjudicate MCP tools itself; the server's own auth governs. An optional channel
allowlist (`channelPermissions.ts:177` `filterPermissionRelayClients`) can gate
servers that opt into the channel-permission capability.

### 5 — Invocation
The overridden `call()` (`client.ts:1833`) ensures the client is connected and
calls `callMCPToolWithUrlElicitationRetry(...)` (`:2813`), which wraps
`callMCPTool(...)` (`:3029`). The actual JSON-RPC happens via the MCP SDK:

```ts
client.callTool(
  { name: tool, arguments: args, _meta: meta },  // tool = original (un-prefixed) name
  CallToolResultSchema,
  { signal, timeout: getMcpToolTimeoutMs(), onprogress: … }   // (:3092-3116)
)
```

`onprogress` is bridged to the tool's `onProgress` as `mcp_progress` updates, so
long-running MCP tools stream progress like native tools.

### 6 — Error classification
`callMCPTool` maps failures (`:3124-3231`): `result.isError` →
`McpToolCallError`; HTTP 401 → `McpAuthError` (triggers token refresh,
[06](06-services-layer.md)); 404 + JSON-RPC `-32001` → `McpSessionExpiredError`
(clears cache, reconnects).

### 7 — Result processing & size control
`processMCPResult(result, tool, name)` (`client.ts:2720`):
- `transformMCPResult(...)` normalizes content (`:2725`);
- if it fits, return as-is (`:2735`);
- else, if large-output files are disabled, `truncateMcpContentIfNeeded(...)`
  trims to a token budget (`:2747`);
- otherwise **persist to a file** with id `mcp-<server>-<tool>-<timestamp>`
  (`persistToolResult`, `:2769-2773`) and return instructions pointing the model
  at the file + size estimate (`getLargeOutputInstructions`, `:2794`).
Binary blobs (images/audio) go through `persistBinaryContent`
(`mcpOutputStorage.ts`) and come back as file-reference text blocks.

The tool returns `{ data: mcpResult.content, mcpMeta?: { _meta, structuredContent } }`
(`client.ts:1897-1909`), preserving server metadata.

### 8 — Elicitation (server asks the user)
If the server sends an `ElicitRequest`, the handler registered by
`registerElicitationHandler` (`elicitationHandler.ts:68`) first tries
programmatic resolution via `runElicitationHooks` (`:91`), and otherwise queues
a dialog into `AppState.elicitation.queue` (`:127-150`) and awaits the user's
response. URL-mode elicitations are confirmed via an
`ElicitationCompleteNotification` (`:174-207`). URL-required errors (`-32042`)
trigger up to 3 retries (`client.ts:2850`).

## What this illustrates

- **MCP tools are first-class `Tool`s**, just produced dynamically from a server
  listing rather than statically registered — same execution and rendering path,
  different `call()`.
- **Passthrough permission** is the key policy difference: the server owns
  authorization, so Claude Code's permission engine steps aside (with an optional
  channel-allowlist guard).
- **Large results don't blow up context**: truncate-or-persist keeps a 50 MB tool
  result from poisoning the window — the model gets a pointer and can `Read` it.
- **Connections are lazy and stateful**: deferred discovery + `pending` servers
  mean MCP startup cost is paid only when a server's tools are actually wanted.
