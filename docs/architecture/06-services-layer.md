# 06 — Service Layer

`src/services/` holds reusable, UI-agnostic integrations with external systems.
The query-engine pieces (`api/claude.ts`, `compact/`) are covered in
[03](03-query-engine.md); this document covers the rest: **MCP**, **OAuth**,
**LSP**, **analytics & feature flags**, the **multi-provider API client**, and
the supporting services.

## MCP — Model Context Protocol (`src/services/mcp/`)

MCP lets external servers expose tools, resources, and prompts to the agent.
This is the largest service (`client.ts` ~3,348 lines, `auth.ts` ~2,465).

### Connection management (`client.ts`)

`connectToServer(name, serverRef, stats)` is a memoized factory that builds the
right transport + MCP SDK client. Supported transports:

| Transport | Notes |
|---|---|
| `stdio` (default) | spawn subprocess, `StdioClientTransport` |
| `sse` | `SSEClientTransport` with auth provider |
| `http` | `StreamableHTTPClientTransport` (async POST) |
| `ws` | WebSocket with mTLS/proxy support |
| `sse-ide` / `ws-ide` | IDE-provided MCP servers (unauthenticated, restricted toolset) |
| `sdk` | pass-through to the host SDK (`mcp_tool_call`) |
| `claudeai-proxy` | session-ingress SHTTP gateway |
| in-process | Chrome MCP & Computer Use (avoid subprocess overhead) |

Lifecycle details: per-request timeout 60s; connection timeout 30s; batched
connection limits (3 stdio + 20 remote). `wrapFetchWithTimeout` applies the
timeout to POSTs (SSE GET streams exempt) and normalizes the Streamable-HTTP
`Accept` header. Error recovery detects session expiry (404 + JSON-RPC -32001),
tracks terminal errors for auto-reconnect, and caches which servers need auth
(15-min TTL). IDE MCP servers are restricted to an allowlist
(`mcp__ide__executeCode`, `mcp__ide__getDiagnostics`).

### Configuration & discovery (`config.ts`)

Servers resolve across scopes with precedence
**enterprise (exclusive) > local > project > user > dynamic (SDK) > plugin**.
`getClaudeCodeMcpConfigs()` loads, content-dedupes (by stdio command or
unwrapped URL signature — manual servers override plugin servers), applies
policy (allow/deny by name/command/URL with wildcards), and gates project
servers behind an approval workflow. Project config lives in `.mcp.json`
(atomic temp-file + rename writes).

### Authentication (`auth.ts`, `xaa*.ts`)

`ClaudeAuthProvider` implements the MCP SDK's `OAuthClientProvider`: metadata
discovery (RFC 9728 + RFC 8414), automatic token refresh, step-up auth
(403 → re-auth on `Www-Authenticate: Bearer realm="step-up"`), platform-native
secure storage, and lockfile-based refresh synchronization to avoid concurrent
refresh races. `xaaIdpLogin.ts` adds cross-app-access identity federation with
enterprise IDPs (Okta, Azure AD) via token exchange. Non-standard error bodies
(e.g. Slack's HTTP-200 errors) are normalized.

### Tool exposure

MCP tools are wrapped as `Tool[]` (see [04](04-tool-system.md)). Output that is
too large is truncated (`truncateMcpContentIfNeeded`) or persisted to disk
(`persistBinaryContent`). `elicitationHandler.ts` services server→client
prompts (e.g. confirmation requests). `MCPConnectionManager.tsx` owns the
React-side connection state; `officialRegistry.ts` exposes the Anthropic/
community registry; the `channel*` files implement user/team ACLs.

## OAuth 2.0 (`src/services/oauth/`)

Authorization-code flow with PKCE:

- **`index.ts`** orchestrates `startOAuthFlow`: spawn `AuthCodeListener` on a
  random port, generate PKCE challenge + state, build automatic (localhost
  callback) and manual (paste-code) URLs, exchange code → tokens, fetch profile,
  cleanup.
- **`client.ts`** — `buildAuthUrl` (scopes, PKCE, `login_hint`, `org_uuid`),
  `exchangeCodeForTokens` (validates state for CSRF, redeems code),
  `fetchProfileInfo` (`GET /api/oauth/user_account` → subscription type, rate
  tier).
- **`auth-code-listener.ts`** — minimal callback HTTP server; serves a
  success/error redirect page; dedupes concurrent callbacks.
- **`crypto.ts`** — `generateCodeVerifier` (RFC 7636), `generateCodeChallenge`
  (SHA-256, base64url), `generateState`.

## LSP — Language Server Protocol (`src/services/lsp/`)

Provides IDE-grade diagnostics & navigation, decoupled from the UI.

- **`LSPServerManager.ts`** — loads configs, maps file extension → server,
  lazily spawns a server on first use (`ensureServerStarted`), routes requests,
  and syncs `textDocument` notifications (open/change/save/close).
- **`LSPServerInstance.ts`** — per-server state machine
  (`not-started → starting → running → error/stopped`); stdio spawn + RPC
  parsing + recovery.
- **`LSPClient.ts`** — JSON-RPC client (`vscode-jsonrpc`).
- **`LSPDiagnosticRegistry.ts`** — aggregates `publishDiagnostics` by file.
- **`config.ts`** — extension/language mapping from settings.
- **`passiveFeedback.ts`** — surfaces actionable insights from diagnostics.

## Analytics & feature flags (`src/services/analytics/`)

A unified event pipeline with strict PII discipline.

- **`index.ts`** — public `logEvent` / `logEventAsync`; metadata restricted to
  `boolean | number | undefined` (no strings — prevents code/path leaks);
  `_PROTO_*`-prefixed keys carry PII-tagged values destined only for first-party
  proto columns. Events are queued until a sink attaches (no external deps to
  avoid cycles).
- **`sink.ts`** — routes events to Datadog + the first-party exporter; applies
  dynamic sampling from the `tengu_event_sampling_config` GrowthBook flag; strips
  `_PROTO_*` before non-1P backends.
- **`datadog.ts`** — Datadog APM intake.
- **`firstPartyEventLogger.ts` / `firstPartyEventLoggingExporter.ts`** —
  batched BigQuery export, hoisting `_PROTO_*` to structured columns.
- **`growthbook.ts`** — GrowthBook SDK integration; the **runtime** feature-flag
  system (distinct from the build-time `feature()` gates in
  [02](02-startup-and-runtime-modes.md)). Used by policy limits, settings sync,
  MCP approval, bridge multi-session, compaction config, and many rollouts.

## Multi-provider API client (`src/services/api/`)

Beyond `claude.ts`/`client.ts`/`withRetry.ts` (see [03](03-query-engine.md)):

- **`bootstrap.ts`** — fetches runtime config from `/api/claude_cli/bootstrap`
  (`client_data`, model options); OAuth-preferred, API-key fallback; cached.
- **`usage.ts`** — `fetchUtilization()` (`GET /api/oauth/usage`) returns rate-
  limit utilization windows (5-hour, 7-day, 7-day Opus, extra usage) and reset
  times.
- **`filesApi.ts`** — downloads session file attachments (`--file=<id>:<path>`),
  with retry and a 500 MB cap.
- **`sessionIngress.ts`** — obtains a JWT for the session-ingress SHTTP gateway
  used by MCP/API calls in remote sessions.
- **`errors.ts` / `errorUtils.ts`** — error classification;
  `TelemetrySafeError`.
- **`referral.ts`, `overageCreditGrant.ts`, `ultrareviewQuota.ts`,
  `adminRequests.ts`, `grove.ts`, `metricsOptOut.ts`, `dumpPrompts.ts`,
  `promptCacheBreakDetection.ts`, `firstTokenDate.ts`** — referral program,
  credit/quota management, enterprise admin endpoints, the internal Grove
  provider, telemetry opt-out, prompt dumping, and cache-break detection.

## Supporting services

| Service | Purpose |
|---|---|
| `voice/`, `voiceStreamSTT.ts`, `voiceKeyterms.ts` | push-to-talk speech-to-text (native audio capture, with `rec`/`arecord` fallback; silence detection) |
| `tokenEstimation.ts` | fast pre-call token estimation |
| `settingsSync/` | upload/download settings to the cloud (OAuth, interactive only; 500 KB/file) |
| `policyLimits/` | org policy enforcement (`/api/policy_limits`, OAuth Team+); ETag-cached, background-polled, fail-open |
| `remoteManagedSettings/` | enterprise managed-settings push with a security check |
| `PromptSuggestion/` | context-aware prompt suggestions + speculation |
| `tips/` | contextual CLI tips with sampling |
| `notifier.ts` | desktop/terminal notifications |
| `preventSleep.ts` | keep the machine awake during long runs |
| `vcr.ts` | record/replay API responses for tests |
| `internalLogging.ts`, `diagnosticTracking.ts` | internal diagnostics |
| `mcpServerApproval.tsx` | MCP server approval UI |
| `skillSearch/signals.ts` | signals for remote skill search ranking |
| `AgentSummary/`, `toolUseSummary/`, `MagicDocs/` | LLM-generated summaries & docs |

## Cross-cutting patterns

- **Memoized connections** keyed by server signature, cleared on error for
  auto-reconnect.
- **Fail-open policy** — missing policy means "allowed" unless explicitly
  denied.
- **PII protection** — metadata restricted to primitives; paths/code never
  logged directly.
- **Lazy initialization** — analytics queue events until the sink attaches; LSP
  servers spawn on first file; provider SDKs and audio capture load on demand.
- **Lockfile token sync** — prevents concurrent OAuth refresh races across
  processes.
- **Platform-native secure storage** — Keychain (macOS) / Credential Manager
  (Windows) / Secret Service (Linux).

## Key files

| File | Role |
|---|---|
| `src/services/mcp/client.ts` | MCP connection factory & transports |
| `src/services/mcp/config.ts` | MCP server discovery, scopes, policy |
| `src/services/mcp/auth.ts` | MCP OAuth provider |
| `src/services/oauth/index.ts` | OAuth flow orchestration |
| `src/services/lsp/LSPServerManager.ts` | LSP routing & lifecycle |
| `src/services/analytics/index.ts` | event logging API |
| `src/services/analytics/growthbook.ts` | runtime feature flags |
| `src/services/api/client.ts` | multi-provider client factory |
| `src/services/api/bootstrap.ts` | runtime config fetch |
| `src/services/api/usage.ts` | rate-limit utilization |
