# 04 — Tool System

Tools are the agent's hands. Each tool is a self-contained module declaring its
input schema, permission model, concurrency/safety properties, execution logic,
and UI rendering. This document covers the `Tool` interface, the registry, the
execution pipeline, Bash security, deferred loading, and the full catalog.

## Anatomy of a `Tool` — `src/Tool.ts`

The generic `Tool<Input, Output, Progress>` interface (`Tool.ts:362-695`)
defines the complete contract. Highlights:

**Identity & discovery**
- `name`, `aliases?` (back-compat for renamed tools), `searchHint?` (keywords
  for deferred-tool search), `isMcp?`, `isLsp?`.

**Schema** (`:394-400`)
- `inputSchema: Input` — always a Zod schema.
- `inputJSONSchema?` — raw JSON Schema alternative used by MCP tools.
- `outputSchema?`, `inputsEquivalent(a,b)?` (custom equality for dedup).

**Execution** (`:379-385`)
```ts
async call(args, context: ToolUseContext, canUseTool, parentMessage, onProgress?)
  : Promise<ToolResult<Output>>   // { data, newMessages?, contextModifier?, mcpMeta? }
```

**Safety & concurrency** (`:402-406`)
- `isConcurrencySafe(input)` — may run in parallel with other safe tools.
- `isReadOnly(input)` — does not mutate state.
- `isDestructive?(input)` — irreversible (delete/send).
- `interruptBehavior?()` → `'cancel' | 'block'`.

**Permission & validation** (`:489-503`)
- `validateInput?(input, context)` — schema/constraint checks.
- `checkPermissions(input, context)` — permission-engine integration.
- `preparePermissionMatcher?(input)` — custom rule matching (e.g. `Bash(git *)`).

**Rendering** (`:604-694`)
- `renderToolUseMessage`, `renderToolUseProgressMessage`,
  `renderToolResultMessage`, `renderToolUseRejectedMessage`,
  `renderToolUseErrorMessage`, `renderGroupedToolUse`.

**Deferred loading** (`:442, :449`)
- `shouldDefer?` — sent to the API with `defer_loading: true` (must be
  discovered via `ToolSearchTool` before use).
- `alwaysLoad?` — full schema always sent on turn 1 (essential tools).

### `buildTool()` and fail-closed defaults (`:757-792`)

`buildTool(def)` merges a partial `ToolDef` with `TOOL_DEFAULTS`, which are
deliberately conservative:

```ts
TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,   // assume NOT safe
  isReadOnly: () => false,          // assume it writes
  checkPermissions: () => ({ behavior: 'allow' }),
  …
}
```

So a tool that forgets to declare safety is treated as unsafe — fail-closed.

## Registry & lookup — `src/tools.ts`

- **`getAllBaseTools()`** (`tools.ts:193-251`) — the single source of truth for
  built-in tools, with conditional inclusion based on environment/flags
  (e.g. embedded search hides Glob/Grep; `AGENT_TRIGGERS` adds cron tools;
  `KAIROS` adds Sleep/PushNotification; tool-search adds `ToolSearchTool`).
- **`findToolByName(tools, name)`** (`Tool.ts:358-360`) — matches primary name
  or any alias.
- **`assembleToolPool(permissionContext, mcpTools)`** (`tools.ts:345-367`) —
  combines built-ins + MCP tools, filters by deny rules, dedupes (built-ins
  win), and sorts for prompt-cache stability.
- **`getTools(permissionContext)`** (`tools.ts:262-327`) — filters deny rules,
  hides REPL-only tools, drops disabled tools. Returns ~80+ tools normally,
  but only 3 in `CLAUDE_CODE_SIMPLE` mode.
- **`filterToolsByDenyRules`** — removes tools with blanket deny rules; an
  `mcp__<server>` prefix rule strips all tools from that server.

## Execution & orchestration — `src/services/tools/`

### Concurrency batching — `toolOrchestration.ts`

`runTools(toolUseBlocks, …)` (`:19-82`) partitions a turn's tool calls
(`:91-116`):
- **Concurrent batch** — all `isConcurrencySafe` (read-only) tools run in
  parallel.
- **Serial batch** — non-safe tools run one at a time, with context modifiers
  applied between batches to preserve isolation.

Example: `[Bash(find), Bash(grep), Edit]` → `[find, grep]` in parallel, then
`[Edit]`.

### Streaming executor — `StreamingToolExecutor.ts`

Powers in-flight execution during model streaming (`:40-150`). It manages a
concurrency queue, blocks each tool's start until permission is granted, streams
progress, and on a tool error aborts parallel siblings via a
`siblingAbortController`. User ESC triggers abort according to each tool's
`interruptBehavior`.

### Per-tool execution path — `toolExecution.ts`

`runToolUse(toolUse, …)` (`:337-491`):
1. Find tool by name/alias; log `tool_use_start` telemetry (sanitized).
2. Validate input (Zod); early-return `CANCEL_MESSAGE` if aborted.
3. Stream `checkPermissionsAndCallTool()`:
   - **Pre-tool hooks** (`toolHooks.ts`) — may mutate input or deny.
   - **Permission check** — rules → classifier → user prompt (see
     [07](07-permissions-and-hooks.md)).
   - **Tool call** — `tool.call(...)`, errors wrapped in `TelemetrySafeError`.
   - **Post-tool hooks** — may validate output or block continuation.
4. Yield progress messages and the final result.

### Result handling

`mapToolResultToToolResultBlockParam` (`Tool.ts:557-560`) converts output to an
API `tool_result` block. Large results are persisted to
`.claude/tool-results/<uuid>.txt`, with a preview + path sent to the model
(e.g. `BashTool.maxResultSizeChars = 100,000`); a per-session budget evicts old
files LRU-style.

## Bash security — `src/tools/BashTool/`

The most security-sensitive tool. Input schema (`BashTool.tsx:227-247`):
`{ command, timeout?, description?, run_in_background?,
dangerouslyDisableSandbox?, _simulatedSedEdit? }`.

**Concurrency safety** is computed by parsing the command with
`splitCommandWithOperators()` and checking each subcommand's read-only-ness —
the whole command is safe only if every part is (`find`, `grep`, `ls`, `cat`, …).

**Read-only / safety validation** (`bashSecurity.ts`, `readOnlyValidation.ts`)
runs 15+ static checks, each with a numeric ID for minified builds. Among them:

- Incomplete commands; `jq` `system()` (arbitrary code) blocking.
- Command-substitution attacks: `$()`, `$(< file)`, `<(cat file)`, `=()`, Zsh
  EQUALS expansion.
- Shell metacharacters in unquoted literals; dangerous variables
  (IFS injection, `BASH_COMMAND` overwrite); newlines in commands.
- Input/output redirection target validation; brace expansion `{a,b}`.
- Control characters (`\x00`–`\x1F`); Unicode whitespace lookalikes
  (U+00A0, U+2000–U+200D); mid-word `#` comment hiding.
- Zsh-dangerous builtins (`zmodload`, `emulate`, `sysopen`, `zpty`, `ztcp`);
  malformed-token injection; backslash-escaped operators (`\;`); quote/comment
  desync.

Underlying machinery: an AST parser (`utils/bash/ast.ts`), shell-quote
validation, and heredoc extraction (`utils/bash/`).

**Permission matching** (`bashPermissions.ts:161-251`):
`getSimpleCommandPrefix()` extracts a rule-matchable prefix (`git commit`,
`npm run`), skipping safe env-var assignments and falling back to exact match
when an env var is unsafe. Rules look like `Bash(git *) → Allow`,
`Bash(npm run) → Ask`, `Bash(curl) → Deny`.

**Sandboxing** — `shouldUseSandbox()` (gated `SANDBOXED_BASH_TOOL`, opt-out via
`dangerouslyDisableSandbox`) runs commands in a restricted environment via
`@anthropic-ai/sandbox-runtime`, wrapped by `src/utils/sandbox/sandbox-adapter.ts`.
The package is stubbed in this tree, but its real behavior (OS-level Seatbelt/
bubblewrap sandboxing, filesystem deny/allow rules, allowlist network proxy) is
documented in [12 — Vendored Packages](12-vendored-packages.md).

A parallel **PowerShell** tool exists for Windows
(`src/tools/PowerShellTool/`) with its own parser, path validation, and
read-only analysis.

## File tools

- **FileReadTool** (`tools/FileReadTool/`) — `{file_path, offset?, limit?,
  pages?, as_base64?}`. Handles images (compress to token budget), PDFs (page
  ranges, OCR), Jupyter notebooks (flattened), binary detection, and **blocked
  device paths** (`/dev/zero`, `/dev/random`, …). Enforces per-file/session
  token limits.
- **FileEditTool** — `{file_path, old_string, new_string, replace_all?}`.
  Requires old≠new; 1 GiB size cap; **rejects UNC paths** (NTLM credential-leak
  prevention); checks deny rules and team-memory secret guard; tracks file
  history; notifies the VS Code SDK; clears LSP diagnostics; auto-activates
  conditional skills when editing `.claude/skills/`.
- **FileWriteTool** — `{file_path, content}`; simpler than Edit, same
  size/secret/LSP/skill logic.

## Search tools

- **GlobTool** — `{pattern, path?}`; respects `.gitignore`/`.claudeignore`;
  capped at 100 files; concurrency-safe.
- **GrepTool** — ripgrep-backed; three output modes (`content`,
  `files_with_matches`, `count`); context flags (`-A/-B/-C`); excludes VCS dirs;
  default `head_limit` 250; offset pagination; concurrency-safe.

## Agents, skills, MCP, LSP

- **AgentTool** (`tools/AgentTool/AgentTool.tsx`) — spawns sub-agents with
  isolated token budgets; supports `subagent_type`, `model`,
  `run_in_background`, multi-agent `name`/`team_name`/`mode`, and isolation
  (`worktree` / `remote`, `cwd`). See [08](08-bridge-remote-multiagent.md).
- **SkillTool** (`tools/SkillTool/`) — executes a skill in a forked context;
  resolves local → bundled → MCP skills; supports model overrides and usage
  tracking. See [09](09-skills-and-plugins.md).
- **MCPTool** — a template tool overridden per MCP server. Name pattern
  `mcp__<server>__<tool>`; `inputSchema` is passthrough (server-defined);
  permissions `behavior: 'passthrough'` (server decides). Large/binary content
  is truncated or persisted.
- **LSPTool** — language-server operations (`goToDefinition`, `findReferences`,
  `hover`, symbols, call hierarchy); deferred; 10 MB file cap; read-only.

## Tasks, teams, planning, worktrees

`TaskCreate/Get/Update/List/Stop/Output`, `SendMessageTool`,
`TeamCreate/TeamDelete`, `ListPeers`, `EnterPlanMode`/`ExitPlanModeV2`,
`EnterWorktree`/`ExitWorktree`. Most are deferred. Multi-agent ones are covered
in [08](08-bridge-remote-multiagent.md).

## Deferred tool loading — `ToolSearchTool`

Tools with `shouldDefer: true` are advertised to the model with
`defer_loading: true`; the model must call **`ToolSearchTool`** first
(`tools/ToolSearchTool/`):

- `select:ToolName` → returns that tool's full schema directly.
- keyword query → ranks tools by name/description/hint (cosine similarity),
  returns up to `max_results`, and reports `pending_mcp_servers` if matches live
  on unconnected MCP servers.

If a deferred tool is called without being discovered,
`buildSchemaNotSentHint()` (`toolExecution.ts:578-597`) tells the model to run
`ToolSearch` with `select:ToolName`.

## Tool catalog (categorized)

| Category | Tools |
|---|---|
| Shell | BashTool, PowerShellTool (Windows) |
| Files | FileReadTool, FileWriteTool, FileEditTool, NotebookEditTool |
| Search | GlobTool, GrepTool, ToolSearchTool |
| Agents/Skills | AgentTool, SkillTool |
| Code intel | LSPTool |
| Tasks | TaskCreate/Get/Update/List/Stop/Output |
| Teams/peers | SendMessageTool, TeamCreate/TeamDelete, ListPeers |
| Web | WebFetchTool, WebSearchTool |
| Planning | EnterPlanMode, ExitPlanModeV2 |
| Worktree | EnterWorktree, ExitWorktree |
| MCP resources | ListMcpResources, ReadMcpResource, MCPTool (template) |
| Utility | TodoWrite, AskUserQuestion, Brief, Config, SendUserFile, PushNotification, RemoteTrigger, Monitor |
| Feature-gated | Sleep (KAIROS/PROACTIVE), Cron* (AGENT_TRIGGERS), WebBrowser, Snip (HISTORY_SNIP), Workflow, CtxInspect (CONTEXT_COLLAPSE), TerminalCapture, … |

## Telemetry

Every execution logs sanitized fields: tool name, `isMcp`, MCP server type/URL,
query chain id & depth, permission mode, decision source (hook/rule/classifier/
user), duration, and a classified error (bash check ID, errno, error type).

## Key files

| File | Role |
|---|---|
| `src/Tool.ts` | `Tool` interface, `buildTool`, defaults, lookup |
| `src/tools.ts` | registry, pool assembly, deny filtering |
| `src/services/tools/toolOrchestration.ts` | concurrency batching |
| `src/services/tools/StreamingToolExecutor.ts` | in-flight execution |
| `src/services/tools/toolExecution.ts` | per-tool exec, permission/hook flow |
| `src/services/tools/toolHooks.ts` | pre/post tool hooks |
| `src/tools/BashTool/*` | bash parsing, security, permissions, sandbox |
| `src/tools/FileReadTool|FileEditTool|FileWriteTool/*` | file ops |
| `src/tools/ToolSearchTool/*` | deferred-tool discovery |
