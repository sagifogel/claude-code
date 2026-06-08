# 05 — Command System & Terminal UI

This document covers two things: the **slash-command system** (user-facing `/`
commands) and the **terminal UI/rendering pipeline** (a custom React-to-ANSI
renderer built on `react-reconciler` + Yoga).

---

## Part A — Slash command system

### Command types — `src/types/command.ts`

A command is a discriminated union (`Command = CommandBase & (PromptCommand |
LocalCommand | LocalJSXCommand)`):

| Type | Discriminator | Behavior |
|---|---|---|
| **PromptCommand** | `'prompt'` (`:25-57`) | Expands to content sent to the model via `getPromptForCommand(args, context) → ContentBlockParam[]`. Model-invocable; supports `allowedTools`, `effort`, and `context: 'inline' | 'fork'`. This is what skills compile to. |
| **LocalCommand** | `'local'` (`:74-78`) | Text-only, no UI. Lazy-loaded module with `call(args, context) → LocalCommandResult` (`text` / `compact` / `skip`). |
| **LocalJSXCommand** | `'local-jsx'` (`:144-152`) | Interactive Ink UI. Lazy-loaded `call(onDone, context, args) → React.ReactNode`. Examples: `/help`, `/config`, `/settings`. |

`CommandBase` (`:175-203`) adds `name`, `aliases`, `description`,
`availability` (auth gating), `isEnabled()` (feature flags), `isHidden`,
`loadedFrom` (source badge), `disableModelInvocation`.

### Commands vs. tools

Both are model-invocable, but they differ: **tools** are the agent's primitive
actions (Bash, Edit, …) defined in `Tool.ts`; **commands** are user-facing
shortcuts triggered with `/`. PromptCommands/skills are exposed to the model via
`SkillTool`; many commands are local-only UI.

### Registry & loading — `src/commands.ts`

- **Static built-ins** (`:258-346`) imported at module load, memoized via
  `COMMANDS()` to defer config reads. Conditional imports via `feature()` for
  platform features (VOICE_MODE, DAEMON, …); internal-only commands are stripped
  from external builds.
- **`loadAllCommands(cwd)`** (`:449-517`) concurrently gathers: dir skills
  (`~/.claude/skills/`), plugin skills, bundled skills, builtin-plugin skills,
  plugin commands, workflow commands — plus built-in/MCP commands. Failures are
  caught per-source.
- **`getCommands(cwd)`** (`:476-517`, memoized by cwd) adds dynamically
  discovered skills, then filters by `meetsAvailabilityRequirement()` &&
  `isCommandEnabled()` and dedupes.
- **Availability** (`:417-443`): `claude-ai` (OAuth subscriber only),
  `console` (direct API key only), or universal.

### Lookup & dispatch

`parseSlashCommand(input)` (`src/utils/slashCommandParsing.ts`) extracts
`{ commandName, args, isMcp }`, detecting the ` (MCP)` suffix.
`findCommand`/`getCommand` (`commands.ts:688-719`) resolve by name, display
name, or alias.

### Execution flow

```
User types "/foo args" + Enter
  → PromptInput captures string
  → handlePromptSubmit()                     (src/utils/handlePromptSubmit.ts)
  → processUserInput()                       (src/utils/processUserInput/)
       parseSlashCommand → findCommand
       route by type:
         PromptCommand   → getPromptForCommand(args, ctx) → ContentBlockParam[] → model
         LocalCommand    → load() → call(args, ctx) → LocalCommandResult
         LocalJSXCommand → load() → call(onDone, ctx, args) → React.ReactNode (rendered)
  → append result messages; trigger onQuery() if shouldQuery
```

### Command sub-categories

- **Remote-safe** (`:619-637`) — runnable in `--remote` mode (session, exit,
  clear, help, theme, vim, cost, plan, …).
- **Bridge-safe local** (`:651-660`) — runnable over the IDE bridge (compact,
  clear, cost, summary, releaseNotes, files).
- **Skill-tool commands** (`:563-581`) — prompt commands exposed to the model.
- **Slash-command-tool skills** (`:586-608`) — visible in the `/skills` picker.

Representative commands: `/commit`, `/review`, `/compact`, `/mcp`, `/config`,
`/doctor`, `/login`, `/logout`, `/memory`, `/skills`, `/tasks`, `/vim`, `/diff`,
`/cost`, `/theme`, `/context`, `/pr_comments`, `/resume`, `/share`, `/insights`,
`/desktop`, `/mobile`.

---

## Part B — Terminal UI & rendering

Claude Code ships its **own** Ink-style renderer in `src/ink/` rather than the
npm `ink` package. It maps a React tree to a virtual DOM, lays it out with Yoga,
paints to a cell buffer, and diffs that buffer to ANSI escape sequences.

```
React components (REPL.tsx, PromptInput.tsx, …)
   │  react-reconciler HostConfig
   ▼
Virtual DOM (DOMElement tree, src/ink/dom.ts)
   │  Yoga flexbox layout (src/ink/layout/, src/native-ts/yoga-layout)
   ▼
renderNodeToOutput → Output (operation buffer) → Screen (2D styled cell grid)
   │  log-update diff
   ▼
ANSI escape sequences → stdout
```

### The reconciler — `src/ink/reconciler.ts`

Implements `react-reconciler`'s HostConfig, bridging React to the virtual DOM:
- `createInstance` → `createNode(type)` + `setStyle`/`setAttribute`.
- `appendChild`/`insertBefore`/`removeChild` → mutate `DOMElement` tree, manage
  Yoga nodes, `markDirty(parent)` (walks ancestors).
- `commitUpdate` → apply diffed props.
- Event handlers are tracked **separately** from attributes so handler-identity
  changes don't force a re-blit. Text-only nodes (`ink-virtual-text`,
  `ink-link`, `ink-progress`) skip Yoga node creation.

### The DOM node — `src/ink/dom.ts`

`DOMElement` carries `nodeName`, parent/children, `style` (Yoga props),
`attributes`, `textStyles`, an optional `yogaNode`, a `dirty` flag, scroll state
(`scrollTop`, `pendingScrollDelta`, `stickyScroll`, clamps, anchor,
`scrollHeight`), focus/event wiring, and lifecycle hooks
(`onComputeLayout`, `onRender`, `onImmediateRender`).

### Layout — Yoga (`src/ink/layout/`, `src/native-ts/yoga-layout/`)

A WebAssembly/native flexbox engine. `YogaLayoutNode` wraps the native node and
exposes flex direction, justify/align, dimensions, padding/margin/gap,
position, flex grow/shrink/basis, overflow, and `setMeasureFunc` for text
measurement. Layout results are cached between frames.

### Rendering pipeline — `renderer.ts` + `render-node-to-output.ts`

`createRenderer` validates Yoga dimensions, resets the `Output` buffer, and
calls `renderNodeToOutput(root, output, { prevScreen })`, which recursively:
- skips hidden (`display: none`) nodes;
- pushes a clip region per node;
- **blits unchanged regions** from `prevScreen` when a node is clean (O(1)
  instead of re-walking the subtree);
- renders boxes (recurse), text (squash → segments → wrapped writes), raw ANSI,
  and progress bars;
- handles scroll boxes (drains `pendingScrollDelta`, hardware-scrolls via
  DECSTBM, fills newly exposed rows) and absolutely-positioned nodes;
- tracks **layout shift** — steady-state frames (spinner/stream) keep
  `layoutShifted = false`, enabling a narrow damage region.

### Output & screen buffer — `output.ts`, `screen.ts`

`Output` collects operations (`write`, `blit`, `clip`/`unclip`, `clear`,
`shift`, `noSelect`) and finalizes a `Screen`: a 2D grid of `Cell`s
(`char`, `width`, `styleId`, `hyperlink?`). Styles, grapheme clusters, and
OSC-8 hyperlinks are interned via pools (`StylePool`, `CharPool`,
`HyperlinkPool`) to cut per-frame allocations. Per-row `softWrap[]` tracks
word-wrap continuations.

### Frame diff to terminal — `log-update.ts`

Converts `Frame` differences to ANSI. Full redraw when the screen overflows the
viewport or the previous frame is "contaminated" (e.g. selection mutated);
otherwise per-cell diff with cursor moves, hardware scroll when a scroll hint is
present, and full redraw in alternate-screen mode.

### The `Ink` instance — `src/ink/ink.tsx` (~1,722 lines)

Owns the Fiber root, root `DOMElement`, `FocusManager`, the renderer, a
double-buffer (front/back `Frame`), selection state, search highlights, mouse
hover tracking, and alt-screen state. It subscribes to `SIGWINCH` (resize) and
`SIGCONT` (resume), patches console output, and throttles renders via
`scheduleRender` (~16 ms frame interval).

### Focus & input — `focus.ts`

`FocusManager` maintains `activeElement` and a bounded focus stack, dispatching
`focus`/`blur` events, handling auto-focus on mount, click-focus, and cleanup
when the focused node is removed.

### Selection & search

`selection.ts` tracks char/word/line selection (double/triple-click), preserves
text across scroll, and yields `getSelectedText()`. Search highlighting inverts
matching cells, supports position-based `n`/`N` navigation, and highlights the
current match.

### Main screen vs. alternate screen

- **Main (REPL):** dynamic height, scrollback, incremental log-update diff +
  scroll optimization.
- **Alt-screen (pickers/help):** fixed viewport height, full redraw per frame,
  mouse tracking enabled, cursor clamped inside the viewport.

### Vim mode — `src/vim/`

A state machine (`vim/types.ts`) over `INSERT` and `NORMAL` modes with substates
(idle/count/operator/find/g/replace/indent…). Supports motions
(`w/b/e/j/k/h/l/0/$/G`), operators (`d/c/y`), text objects (`iw/aw/ip/i"`…),
find (`f/F/t/T`), counts (`3dw`), and dot-repeat (`.`).

### Key components & screens

- **`components/PromptInput/PromptInput.tsx`** (~2,338 lines) — the input box:
  multi-line word-wrap, slash-command autocomplete, vim toggle, paste handling
  with image resizing, undo/redo, history search (Ctrl-R), suggestion bubble,
  plus modals (model picker, history search, quick-open, global search,
  background tasks).
- **`screens/REPL.tsx`** (~5,005 lines) — the main interactive screen.
- Other screens: Doctor (diagnostics), Resume (session picker).
- **`components/Settings/Config.tsx`** (~1,821 lines) — the `/config` UI.

### Rendering optimizations (summary)

Double-buffering; blit of unchanged regions; layout-shift gating of full
damage; Yoga layout caching; style/char/hyperlink interning; DECSTBM hardware
scroll; lazy component loading for local/local-jsx commands; memoized
`getCommands`/`loadAllCommands`; throttled render loop; selection state kept
separate from the screen buffer.

## Key files

| File | Role |
|---|---|
| `src/commands.ts` | command registry & loading |
| `src/types/command.ts` | command type union |
| `src/utils/slashCommandParsing.ts` | `/cmd args` parsing |
| `src/utils/handlePromptSubmit.ts` | input dispatch |
| `src/ink/ink.tsx` | renderer instance, frame loop |
| `src/ink/reconciler.ts` | React→DOM HostConfig |
| `src/ink/dom.ts` | virtual DOM node model |
| `src/ink/render-node-to-output.ts` | tree walk → operations |
| `src/ink/renderer.ts` | render orchestration |
| `src/ink/output.ts` / `screen.ts` | operation & cell buffers |
| `src/ink/log-update.ts` | frame diff → ANSI |
| `src/ink/focus.ts` | focus management |
| `src/native-ts/yoga-layout/` | Yoga flexbox binding |
| `src/components/PromptInput/PromptInput.tsx` | input component |
| `src/screens/REPL.tsx` | main screen |
| `src/vim/*` | vim state machine |
