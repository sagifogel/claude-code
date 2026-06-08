# 01 — System Overview

## What this is

Claude Code is Anthropic's official terminal CLI for software-engineering tasks.
It is an interactive agent: it takes natural-language prompts, calls the Claude
API in a streaming loop, and executes **tools** (shell commands, file edits,
searches, sub-agents, MCP servers, …) on the user's machine to accomplish the
work — all rendered in a rich terminal UI.

It is, in effect, a complete agent runtime: an LLM query engine, a tool
execution engine, a permission/security layer, a terminal UI framework, an
extensibility system (skills + plugins), a persistent memory system, and
several transports for running remotely or inside IDEs.

## Tech stack

| Concern | Choice |
|---|---|
| Runtime | [Bun](https://bun.sh) (≥ 1.3) |
| Language | TypeScript (strict) |
| Terminal UI | React 19 (canary) + a **custom Ink-style renderer** built on `react-reconciler` |
| Layout engine | Yoga (flexbox) via a native binding in `src/native-ts/yoga-layout` |
| CLI parsing | Commander.js (`@commander-js/extra-typings`) |
| Schema validation | Zod |
| Code search | ripgrep (via GrepTool); Bun-native search when available |
| Protocols | MCP (`@modelcontextprotocol/sdk`), LSP (`vscode-languageserver-protocol`) |
| LLM API | `@anthropic-ai/sdk` (+ Bedrock / Vertex / Foundry SDKs, dynamically imported) |
| Telemetry | OpenTelemetry + Datadog + first-party BigQuery export |
| Feature flags | GrowthBook (runtime) + `bun:bundle` `feature()` gates (build-time DCE) |
| Auth | OAuth 2.0 (PKCE), JWT, OS-native credential stores (Keychain / Credential Manager / Secret Service), API keys |

Dependencies on Anthropic-internal packages (`@ant/computer-use-*`,
`@ant/claude-for-chrome-mcp`, `@anthropic-ai/mcpb`,
`@anthropic-ai/sandbox-runtime`) are satisfied by **no-op stubs** in `stubs/`
(`package.json:14-21`). The two public ones (sandbox runtime, MCPB) are
documented from their real npm releases in
[12 — Vendored Packages](12-vendored-packages.md).

## Directory map (top-level `src/`)

```
src/
├── entrypoints/         Process entry (cli.tsx) + SDK schemas/control types
├── main.tsx             CLI kernel: Commander parse, mode dispatch, REPL launch (~4.7K lines)
├── bootstrap/           STATE singleton — session/project identity, runtime config
├── setup.ts             Per-session preparation (worktree, hooks, sinks)
├── cli/                 Non-interactive "print" / headless execution (print.ts)
│
├── QueryEngine.ts       Core LLM query engine (submitMessage generator)
├── query.ts / query/    The streaming agent loop (queryLoop)
├── context.ts/context/  System & user context assembly (git, CLAUDE.md, date)
├── cost-tracker.ts      Per-model token/cost accounting
│
├── Tool.ts              Base Tool interface + buildTool()
├── tools.ts             Tool registry & assembly
├── tools/               ~40+ tool implementations (Bash, File*, Grep, Agent, …)
├── services/tools/      Tool execution & orchestration pipeline
│
├── commands.ts          Slash-command registry
├── commands/            Slash-command implementations (~50)
│
├── ink/                 Custom React→terminal renderer (reconciler, yoga, output)
├── native-ts/           Native bindings (yoga-layout, file-index, color-diff)
├── components/          Ink UI components (~140)
├── screens/             Full-screen UIs (REPL, Doctor, Resume)
├── hooks/               React hooks (incl. toolPermission/)
├── vim/                 Vim-mode state machine
│
├── services/            External-service integrations (see below)
│   ├── api/             Multi-provider Claude client, retries, usage, bootstrap
│   ├── mcp/             Model Context Protocol clients & connection mgmt
│   ├── oauth/           OAuth 2.0 PKCE flow
│   ├── lsp/             Language Server Protocol manager
│   ├── analytics/       Event logging + GrowthBook feature flags
│   ├── compact/         Context compression (auto/micro/reactive)
│   ├── SessionMemory/   Mid-session memory extraction
│   ├── autoDream/       Background memory consolidation
│   ├── extractMemories/ On-demand memory extraction
│   ├── teamMemorySync/  Shared team memory
│   ├── plugins/         Plugin install/marketplace operations
│   ├── policyLimits/    Org policy enforcement
│   └── …                voice, tips, settingsSync, remoteManagedSettings, …
│
├── bridge/              IDE bridge (VS Code / JetBrains) over WebSocket
├── remote/              Remote ("CCR") sessions over WebSocket
├── coordinator/         Multi-agent coordinator mode
├── tasks/ Task.ts       Background task framework (agents, bash, teammates, dream)
├── ssh/ server/         SSH-remote & server modes
├── upstreamproxy/       Proxy configuration
│
├── skills/              Skill loading (bundled / dir / mcp)
├── plugins/             Plugin system entry
├── memdir/              Persistent memory directory
├── state/               Global app-state store (Zustand-like)
├── utils/               566 files: config, settings, permissions, bash parsing, …
├── migrations/          Config migrations
├── schemas/             Zod schemas (hooks, etc.)
├── constants/ types/    Shared constants & type definitions
├── keybindings/         Keybinding configuration
├── buddy/               Companion sprite (Easter egg)
└── outputStyles/        Output styling
```

## The request lifecycle (end to end)

A single user turn flows through the system like this:

1. **Input** — `PromptInput` captures the typed string. If it starts with `/`,
   it is parsed as a slash command (see [05](05-command-and-ui-system.md)).
2. **Submit** — `handlePromptSubmit` → `processUserInput` resolves commands,
   expands `@file` references and attachments, and produces user message blocks.
3. **Query engine** — `QueryEngine.submitMessage()` (a generator) assembles the
   system prompt + context, persists the transcript, and enters `query()`.
4. **Agent loop** — `queryLoop()` prepares messages (snip / microcompact /
   context-collapse / autocompact), selects the model, and calls the Anthropic
   API with streaming enabled (see [03](03-query-engine.md)).
5. **Streaming + tools** — as `tool_use` blocks stream in, the
   `StreamingToolExecutor` begins executing them **in parallel with** the
   model's output. Each tool passes through the permission pipeline + hooks
   (see [04](04-tool-system.md) and [07](07-permissions-and-hooks.md)).
6. **Tool results** are fed back as `tool_result` blocks; the loop re-queries
   the model until it stops requesting tools (or hooks/budgets intervene).
7. **Recovery** — context-overflow (413), max-output-token, and media-size
   errors are caught and recovered via collapse/compaction/escalation.
8. **Render** — every message (assistant text, tool calls, results, progress)
   is yielded up the generator stack and rendered to the terminal by the Ink
   pipeline. Cost/usage is accumulated per model.
9. **Persistence** — the transcript is written to disk for resume; memory
   extraction may fire in the background.

## Cross-cutting design principles

These themes recur across every subsystem and are worth internalizing first:

- **Generator-based streaming.** The whole pipeline
  (`ask → submitMessage → query → queryLoop → callModel`) is composed of async
  generators that yield messages, so output streams to the UI/SDK the instant
  it is produced rather than after a turn completes.

- **Startup parallelization.** Slow I/O (MDM settings, OS keychain reads,
  command/agent discovery, GrowthBook init) is kicked off as early side-effects
  and overlapped with module evaluation so it finishes "for free" during boot
  (`main.tsx` fires `startMdmRawRead()` / `startKeychainPrefetch()` before other
  imports). See [02](02-startup-and-runtime-modes.md).

- **Build-time dead-code elimination.** `feature('FLAG')` gates from
  `bun:bundle` let entire features be stripped from production bundles; in dev
  they are toggled via the `FEATURE_FLAGS` env var
  (`plugins/bunBundleDev.ts`). `MACRO.*` constants are inlined at build time
  (`bunfig.toml`).

- **Fail-closed security defaults.** Tool defaults assume the worst:
  `isConcurrencySafe → false`, `isReadOnly → false` (`Tool.ts`). Bash commands
  run through 15+ static security checks. Permission mode defaults to "ask".

- **Lazy / deferred loading.** Heavy modules (OpenTelemetry, gRPC, provider
  SDKs) are dynamically `import()`-ed only when needed; many tools are sent to
  the model with `defer_loading: true` and must be discovered via
  `ToolSearchTool` before use.

- **Layered hierarchy + precedence.** Settings, permissions, MCP configs, and
  plugins all resolve through the same conceptual stack: enterprise/policy →
  CLI flags → local → project → user → built-in defaults.

- **Immutable state with explicit getters/setters.** The bootstrap `STATE`
  singleton and the app-state store expose typed accessors so reads/writes are
  trackable and the UI can subscribe reactively.

- **Multi-provider, multi-transport.** The same `Tool`/`Command`/message
  abstractions run against four API providers and across local, headless,
  SDK-driven, IDE-bridged, remote (CCR), and SSH execution contexts.

## Where to go next

- New to the codebase? Read [02 — Startup & Runtime Modes](02-startup-and-runtime-modes.md).
- Working on the agent loop or API? [03 — Query Engine](03-query-engine.md).
- Adding or changing a tool? [04 — Tool System](04-tool-system.md).
- Touching the UI? [05 — Command System & Terminal UI](05-command-and-ui-system.md).
