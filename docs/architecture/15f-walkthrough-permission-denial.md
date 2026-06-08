# 15f — Walkthrough: A Permission Prompt, Denial, and Re-Prompt

The model proposes a `Bash` command that no rule pre-allows. Claude Code prompts
the user, the user denies it (or allows-once / allows-always), and the loop
continues accordingly. This is the runtime view of [07](07-permissions-and-hooks.md).

## The trace

### 1 — Tool reaches the permission gate
During tool execution ([15a](15a-walkthrough-prompt-to-edit.md) step 7),
`checkPermissionsAndCallTool` (`toolExecution.ts:599`) runs PreToolUse hooks then
evaluates permission via the pipeline in
`src/utils/permissions/permissions.ts`.

### 2 — Rules + mode produce a decision
`getAllowRules(context)` (`permissions.ts:122`) loads rules grouped by source
([07](07-permissions-and-hooks.md): policy → flag → local → project → user →
session). The active permission **mode** sets the baseline (`default` → ask).
For Bash, the matcher key is derived by `getSimpleCommandPrefix`
([14c](14c-deep-dive-bash-security.md)). With no matching allow rule, the
decision is **Ask** — a `PermissionAskDecision` (`types/permissions.ts:199`),
possibly carrying a `pendingClassifierCheck`.

### 3 — Permission-request hooks get a say
`executePermissionRequestHooks(...)` (`utils/hooks.ts:4157`) runs any configured
hooks, which may `allow`/`deny`, rewrite `updatedInput`, or add a permanent rule.
If a hook resolves it, the dialog is skipped.

### 4 — The interactive dialog
Unresolved, `PermissionContext` (`hooks/toolPermission/PermissionContext.ts:45`)
pushes a `ToolUseConfirm` onto the React queue and the interactive handler
(`handlers/interactiveHandler.ts:34`) renders the prompt: the command preview,
always-allow suggestions, and a risk explanation. It then **races** background
tasks (async hooks, the bash/transcript classifier) against the user
(`tryClassifier`, `PermissionContext.ts:174`). A `userInteracted` flag with a
~200 ms grace period prevents the classifier from auto-approving once the user
starts typing — whoever resolves first wins.

### 5a — User denies
`onReject` resolves the decision as **deny**. `logDecision(...)`
(`PermissionContext.ts:113`) records it (source: user). The denial is also fed to
**denial tracking** (`denialTracking.ts`): repeated denials flip the system to
plain prompting instead of re-asking the classifier (`permissions.ts:97-101`).

### 6a — Denial flows back to the model
The tool does **not** run. A `tool_result` is synthesized indicating the user
declined, yielded back through `runToolUse`, and appended to the conversation.
The loop `continue`s and re-queries — the model sees "the user denied that" and
typically proposes an alternative or asks why. (This is an ordinary tool result,
not an API error, so it does **not** trip the stop-hook death-spiral guard at
`query.ts:1262`.)

### 5b — User allows (once / always)
`handleUserAllow(...)` (`PermissionContext.ts:291`) resolves **allow**, detecting
whether the user edited the input (`userModified` / `updatedInput`). "Allow
always" calls `persistPermissions(PermissionUpdate[])` (`:139`) to write a new
rule to the chosen settings scope ([11](11-configuration-and-state.md)) — so the
*next* matching command skips the prompt entirely. Execution then proceeds to
`tool.call(...)` as in [15a](15a-walkthrough-prompt-to-edit.md).

## Auto mode and the classifier

In `auto` mode (or when a classifier check is pending), the bash classifier can
decide without a human. If it's **unavailable**, the code chooses deliberately
(see [13](13-design-decisions.md) §2): it may *fail closed* — *"Auto mode
classifier unavailable, denying with retry guidance"* (`permissions.ts:854`) —
or *fail open* to normal prompting (`:872`), depending on context.

## What this illustrates

- **The race is the heart of the UX**: hooks, classifier, and human all compete
  to resolve one decision; the grace period stops a stray keypress from being
  read as consent.
- **Allow-always is a write to the precedence stack** ([11](11-configuration-and-state.md)),
  which is why a one-time grant and a permanent grant diverge here.
- **A denial is just a tool result**, so the agent loop handles it as data and
  keeps going — it doesn't error out. The death-spiral guards
  ([14a](14a-deep-dive-agent-loop.md)) are reserved for *API* errors, not user
  denials.
- **Denial tracking** prevents the classifier from nagging: enough denials and
  the system stops asking it and just prompts.
