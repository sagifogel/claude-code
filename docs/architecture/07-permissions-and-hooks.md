# 07 — Permissions & Hooks

Every tool invocation passes through a permission decision and a set of
lifecycle hooks before (and after) it runs. This is the primary safety boundary
between the model and the user's machine.

## Permission model — types (`src/types/permissions.ts`)

### Permission modes (`:16-38`)

| Mode | Behavior |
|---|---|
| `default` | Ask when no rule matches (secure by default) |
| `plan` | Ask + require a synthesis/approval step before executing |
| `acceptEdits` | Auto-allow edits |
| `bypassPermissions` | Allow everything (dangerous; killswitch-gated) |
| `dontAsk` | Deny everything not explicitly allowed (enterprise lockdown) |
| `auto` *(ant-only)* | Use the classifier when available, else ask |
| `bubble` *(ant-only)* | — |

### Rule sources (`:50-79`)

Rules carry a `toolName`, optional `ruleContent` (matcher detail, e.g.
`git *`), and a `behavior` (`allow` / `deny` / `ask`). They originate from seven
sources, resolved in precedence order:

`policySettings` → `flagSettings` (CLI) → `localSettings` → `projectSettings` →
`userSettings`, plus ephemeral `command` and `session` rules.

### Decisions (`:152-246`)

- **Allow** (`:174-184`) — may carry `updatedInput` (user edited the call),
  `userModified`, and optional `contentBlocks` (e.g. an image as feedback).
- **Ask** (`:199-226`) — prompts the user; may carry a `pendingClassifierCheck`
  for async bash/transcript classification before the dialog shows.
- **Deny** (`:231-236`) — blocks with a reason.

## The decision pipeline — `src/utils/permissions/permissions.ts`

End to end, on each tool call:

1. **Load rules** for the context, grouped by source (`getAllowRules`,
   `:122-132`).
2. **Apply the mode baseline** (`:33-38`) — `default`/`plan` ask, `bypass`
   allows, `dontAsk` denies, `auto` defers to the classifier.
3. **Run permission-request hooks** (see below) — a hook may `allow`/`deny`,
   rewrite the input, or add a permanent rule.
4. **Classifier** (`:59-64`, `:174-215`, gated `TRANSCRIPT_CLASSIFIER`) — a
   lazily-loaded two-stage classifier (fast scan → extended-thinking) returns a
   `YoloClassifierResult` with confidence + token/cache telemetry + request IDs.
5. **User prompt** if the resolved behavior is `ask`.
6. **Denial tracking** (`denialTracking.ts`) — consecutive denials trigger a
   fallback to plain prompting (prevents classifier spam).

### Rule persistence (`permissionsLoader.ts`)

`settingsJsonToRules()` (`:91-114`) parses `permissions: { allow, deny, ask }`
from settings.json. `shouldAllowManagedPermissionRulesOnly()` (`:31-44`)
enforces enterprise policy by ignoring user/project rules when set. Lenient
parsing for edits avoids dropping unrelated rules on a validation failure.

### Permission context (`src/hooks/toolPermission/PermissionContext.ts`)

A frozen per-invocation object (`:45-389`) providing: `logDecision()` (analytics
with tool/input/timing), `persistPermissions()` (apply `PermissionUpdate[]` to
state + disk), `runHooks()`, `tryClassifier()` (for bash), `handleUserAllow()`
(persist rule, detect input modification), and queue operations to push/update/
remove `ToolUseConfirm` items in React state.

### Interactive dialog (`src/hooks/toolPermission/handlers/interactiveHandler.ts`)

`:34-180+`: pushes a `ToolUseConfirm` to the React queue, shows the dialog
(input preview, always-allow suggestions, risk explanation), and **races**
background tasks (hooks + classifier) against user interaction. A `userInteracted`
flag (with a 200 ms grace period for accidental keypresses) prevents the
classifier from auto-approving once the user starts responding. Exactly one
resolver wins: `onAbort` / `onAllow` / `onReject` or a hook/classifier decision.

## The hooks system

Hooks are user/plugin-defined extensions that fire at lifecycle points. They can
be shell commands, prompts (LLM), HTTP calls, or agentic verifiers
(`src/schemas/hooks.ts`).

### Hook types & events (`src/types/hooks.ts`)

Lifecycle events include `PreToolUse`, `PostToolUse`, `PermissionRequest`,
`UserPromptSubmit`, `SessionStart`, `Stop`, `FileChanged`, and more. Each hook
has an optional `matcher` (glob/rule pattern) filtering which tools/events
trigger it (`:228-232`).

- **Sync vs async** (`:182-193`) — `isSyncHookJSONOutput` checks for
  `async: true`; sync hooks return immediately, async hooks may run in the
  background with a timeout.
- **Output schema** (`:50-176`) — `syncHookResponseSchema` validates hook JSON
  output: `continue`, `suppressOutput`, `permissionDecision`, `updatedInput`,
  blocking errors, etc.

### Execution (`src/utils/hooks.ts`, ~5,022 lines)

`executeHooks()` is the generator that runs matching hooks and yields
`AggregatedHookResult`s. `executePermissionRequestHooks()` (`:4157-4192`) builds
a `PermissionRequestHookInput` (tool name/input/suggestions) and yields hook
results that can `allow`/`deny`, supply `updatedInput`, or persist permanent
rule updates.

Hook configuration is **snapshotted** at session start
(`captureHooksConfigSnapshot` in `setup.ts`) and watched for changes, so a hook
file can't be swapped out mid-session without detection — an integrity measure
given hooks execute arbitrary code.

## End-to-end flow

```
tool invoked
  → useCanUseTool / hasPermissionsToUseTool()
      rules + mode → decision or "ask"
  → run PreToolUse + PermissionRequest hooks (sync now, async raced)
  → if "ask": create PermissionContext → push ToolUseConfirm → show dialog
        race(hooks, classifier, user)
  → resolve: extract updatedInput if user/hook modified the call
  → log decision (source: user | hook | classifier | rule; permanent updates?)
  → execute tool
  → run PostToolUse hooks (may block continuation)
```

## Filesystem permissions (`src/utils/permissions/filesystem.ts`)

`checkReadPermissionForTool` / `checkWritePermissionForTool` apply glob-style
path rules (`/src/**/*.ts`) integrated with the deny list. File tools also
enforce hard safety rules independent of permission rules: blocked device paths
(read), UNC-path rejection (edit/write), and the team-memory secret guard
(see [04](04-tool-system.md)).

## Key files

| File | Role |
|---|---|
| `src/types/permissions.ts` | modes, rules, decisions, classifier types |
| `src/utils/permissions/permissions.ts` | the decision pipeline |
| `src/utils/permissions/permissionsLoader.ts` | rule parsing/persistence |
| `src/utils/permissions/filesystem.ts` | path-based read/write rules |
| `src/hooks/toolPermission/PermissionContext.ts` | per-invocation context |
| `src/hooks/toolPermission/handlers/interactiveHandler.ts` | the dialog & race |
| `src/types/hooks.ts` | hook types, events, output schema |
| `src/utils/hooks.ts` | hook execution engine |
| `src/schemas/hooks.ts` | hook config Zod schema (command/prompt/http/agent) |
