# 15g — Walkthrough: From `claude` to the First Prompt

What happens between pressing Enter on `claude` and seeing the input box. This is
the runtime trace of [02](02-startup-and-runtime-modes.md), emphasizing the
*ordering* and the latency-hiding overlaps ([13](13-design-decisions.md) §3).

## The trace

### 1 — Fast-path check
`src/entrypoints/cli.tsx` inspects argv first (`:36-93`). `--version`,
`--mcp` servers, daemon workers, bridge mode, and background-session commands
short-circuit here via dynamic import. Plain `claude` falls through to the kernel.

### 2 — Module-load side effects (the overlap starts immediately)
As `main.tsx`'s imports resolve (~135 ms), three things are already in flight:
- `profileCheckpoint('main_tsx_entry')`;
- `startMdmRawRead()` — spawns `plutil`/`reg query` for managed settings;
- `startKeychainPrefetch()` — pre-reads OAuth tokens / API keys from the OS
  credential store.

They run *during* the import cost, so by the time they're needed they're warm.

### 3 — CLI parse & mode detection
`main()` sets signal handlers and the Windows PATH-hijack guard, parses deep
links, and detects the run mode:
`isNonInteractive = hasPrintFlag || hasInitOnlyFlag || hasSdkUrl || !stdout.isTTY`.
For interactive `claude`, this is the REPL path. `eagerLoadSettings()` pre-reads
`--settings` before `init()`.

### 4 — Commander `preAction` hook
- `await Promise.all([ensureMdmSettingsLoaded(), ensureKeychainPrefetchCompleted()])`
  — join the step-2 prefetches (usually already done);
- `await init()` — load & merge all settings, apply config env vars, resolve auth
  (OAuth / API key / `apiKeyHelper` / cloud creds), init plugins & skills, parse
  MCP configs, fetch remote-managed settings + policy limits (non-blocking), init
  GrowthBook + analytics sinks (~100–300 ms, mostly I/O);
- `runMigrations()` brings old configs to the current schema
  ([11](11-configuration-and-state.md)).

### 5 — The action handler kicks parallel work
- `setupPromise = setup(...)` — **not awaited yet**; its socket bind + worktree +
  hooks snapshot run concurrently;
- `getCommands()` and `getAgents()` — kicked in parallel
  (the comment: *"overlap config I/O with setup(), commands loading, and trust
  dialog"*);
- then `await setupPromise` and `await Promise.all([commands, agents])` join them.

`setup()` ([02](02-startup-and-runtime-modes.md)) restores any interrupted
terminal backup, **snapshots the hooks config** (integrity, see
[07](07-permissions-and-hooks.md)), creates the worktree if `--worktree`, inits
session memory, registers attribution hooks, and emits the `tengu_started`
beacon.

### 6 — Mount Ink, run setup screens
`getRenderContext()` + `createRoot()` mount the Ink renderer
([14b](14b-deep-dive-ink-rendering.md)). `await showSetupScreens(...)` shows the
**trust dialog**, OAuth/onboarding, and the resume picker if applicable —
**this is the one place that can block on the user** (0–∞).

### 7 — Fire-and-forget warmups, then render
Quota status, bootstrap data, and passes-eligibility are kicked
fire-and-forget; `processSessionStartHooks(...)` runs in parallel with MCP
connection. `launchRepl(...)` mounts the REPL component and produces the **first
paint** — the user sees the prompt.

### 8 — Deferred prefetches during the typing window
`startDeferredPrefetches()` runs after first render, populating git status,
system/user context, tips, and model capabilities while the user is reading/
typing — so the *first query* doesn't pay for them.

## Timing model

- **Critical path**: process start → first prompt ≈ 500–700 ms interactive
  (trust dialog excluded, since it's user-bound).
- **Everything off the path is overlapped or deferred**: MDM/keychain (step 2),
  commands/agents/setup (step 5), warmups + context (steps 7–8).

## What this illustrates

- **Startup is choreography, not a sequence** ([13](13-design-decisions.md) §3):
  the code repeatedly starts slow work early and joins it just-in-time.
- **One human gate**: the trust dialog is deliberately the only thing that blocks
  first render, because executing untrusted hooks/plugins requires consent
  ([07](07-permissions-and-hooks.md)).
- **The first query is pre-warmed**: by deferring git/context prefetch to after
  first paint but before the user submits, the perceived latency of *both* boot
  and first response drops.
