# 11 — Configuration & State

How Claude Code is configured (settings files, migrations, schemas) and how it
manages live in-memory state (the app-state store).

## Configuration hierarchy

Configuration resolves through layered sources, highest precedence first:

```
inline flag settings (--settings '{…}')
  → file flag settings (--settings /path.json)
  → policy / managed settings (enterprise)
  → local settings   (.claude/settings.local.json)
  → project settings (.claude/settings.json)
  → user settings    (~/.claude/settings.json)
  → built-in defaults
```

`eagerLoadSettings()` (called early in `main.tsx`) pre-reads the flag settings
before `init()` so they're available during initialization.

### Config files (`src/utils/config.ts`, ~1,817 lines)

- **Global config** `~/.claude/config.json` — install metadata (`numStartups`,
  `installMethod`), UI state (`theme`, `editorMode`), cached feature gates
  (`cachedGrowthBookFeatures`, `growthBookOverrides`), onboarding state, skill
  usage counts, companion state, and enterprise bridge-auth state. A re-entrancy
  guard (`insideGetConfig`, `:51`) prevents stack overflow when analytics reads
  config during config load.
- **Project config** `~/.claude/projects/<slug>/.claude/config.json` —
  `allowedTools`, MCP servers/URIs, `hasTrustDialogAccepted`,
  onboarding-seen count, last-run metrics (cost/duration/lines), active worktree
  session, and per-project MCP enable/disable lists.

### Settings schema (`src/utils/settings/types.ts`, ~1,200 lines)

`SettingsSchema` is a Zod schema with `.passthrough()` (preserves unknown
fields). It covers credentials (`apiKeyHelper`, `awsCredentialExport`,
`gcpAuthRefresh`), `env` (injected into Bash), customization (`skills`,
`agents`, `hooks`, `mcp`), `permissions` (allow/deny/ask + `defaultMode` +
`disableBypassPermissionsMode`), `marketplaces`/`plugins`,
`mcpServerAllowlist`/`mcpServerDenylist`, the enterprise lock
`strictPluginOnlyCustomization`, and `growthBookOverrides`.

**Backward compatibility** is explicitly maintained (`:212-241`): you may add
optional fields, enum values, and union members; you may **not** remove fields/
enum values or make optional fields required. Invalid fields are preserved in
the JSON rather than dropped; a dedicated test suite validates compatibility.

### Settings cache (`src/utils/settings/settingsCache.ts`)

Per-source memoized reads (`userSettings`, `projectSettings`, `policySettings`)
invalidated by file-watch triggers (`resetSettingsCache`). Plugin settings use a
separate base path.

## Migrations (`src/migrations/`)

Versioned, numbered migrations applied at startup (`runMigrations()` in the
`preAction` hook) to bring older configs to the current schema — e.g. moving
fields from `config.json` into `settings.json`, renaming settings, model-name
renames, and type coercion.

## Schemas (`src/schemas/`, `src/utils/settings/`)

Zod schemas validate every external input:

- Settings — `SettingsSchema` (`utils/settings/types.ts`).
- Hooks — `HooksSchema` / `HookCommandSchema` (`src/schemas/hooks.ts`, kept
  separate to break import cycles).
- Permissions — `PermissionRuleSchema`.
- Plugins — `PluginManifestSchema`, `PluginMarketplaceSchema`
  (`utils/plugins/schemas.ts`).
- MCP — `McpServerConfigSchema` (`services/mcp/types.ts`).

A `lazySchema()` helper defers schema construction (returning `() => ZodType`) to
break circular imports.

## App-state store (`src/state/`)

A Zustand-like store holds live UI/runtime state, separate from on-disk config.

### `AppState` (`src/state/AppStateStore.ts`)

Mostly `DeepImmutable<>`, with a few mutable fields (`tasks`,
`agentNameRegistry`, `mcp`, `plugins`). Notable slices:

- Core: `settings`, `mainLoopModel`, `verbose`.
- UI: `statusLineText`, `expandedView` (`none`/`tasks`/`teammates`),
  `footerSelection`.
- **Plugins**: `enabled`/`disabled` plugins, contributed `commands`, `errors`,
  `installationStatus`, `needsRefresh`.
- **MCP**: `clients`, `tools`, `commands`, and `pluginReconnectKey` (bumped to
  force MCP re-init after a plugin reload).
- Worktree/remote: `tungstenActiveSession`, `replBridgeEnabled`,
  `replBridgeConnected`, `remoteSessionUrl`.
- `speculation` (predicted future turns).

### Store mechanics (`src/state/store.ts`, `AppState.tsx`)

`getState()` / `setState()` / `subscribe(listener)`; React binding via
`useAppState(selector)` over `useSyncExternalStore()`. Only one
`AppStateProvider` may exist (throws if nested). `useSettingsChange()` listens
for external settings edits and re-applies them via `applySettingsChange()`.
`onChangeAppState.ts` registers callbacks that fire on mutations (persist UI
state, sync to teammates, log metrics).

## How config flows at runtime

```
Edit ~/.claude/settings.json
  → file watcher → resetSettingsCache()
  → settings reload → useSettingsChange() → applySettingsChange()
  → if plugins/mcp changed: AppState.plugins.needsRefresh = true,
                            bump mcp.pluginReconnectKey
  → loaders re-run; commands/hooks/MCP updated live in the UI
```

## Key files

| File | Role |
|---|---|
| `src/utils/config.ts` | global & project config types, `getConfig()` |
| `src/utils/settings/types.ts` | `SettingsSchema` + back-compat rules |
| `src/utils/settings/settings.ts` | load/parse/watch settings |
| `src/utils/settings/settingsCache.ts` | per-source cache |
| `src/migrations/*` | versioned config migrations |
| `src/schemas/hooks.ts` | hook config schema |
| `src/state/AppStateStore.ts` | `AppState` shape |
| `src/state/store.ts` / `AppState.tsx` | store + React binding |
| `src/state/onChangeAppState.ts` | mutation callbacks |
