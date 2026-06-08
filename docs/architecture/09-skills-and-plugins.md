# 09 — Skills & Plugins

These are the two extensibility mechanisms. **Skills** are reusable prompts/
workflows; **plugins** are installable packages that can contribute skills,
commands, hooks, agents, and MCP servers.

---

## A. Skills (`src/skills/`)

A skill is, at runtime, a `PromptCommand` (see [05](05-command-and-ui-system.md))
exposed to the model via `SkillTool`. Skills come from four tiers:

### 1. Bundled skills (`src/skills/bundledSkills.ts`)

Registered programmatically at startup via `registerBundledSkill()` (`:53`).
A `BundledSkillDefinition` has `name`, `description`, `aliases`, `allowedTools`,
and an async `getPromptForCommand()`. Optional **reference files** are extracted
to `~/.claude/bundled-skills/<name>/` on first invocation (`:67`, memoized to
avoid write races). `resolveSkillFilePath()` (`:196`) guards against path
traversal.

### 2. File-based skills (`src/skills/loadSkillsDir.ts`)

Discovered from a `/skills/` directory (or legacy `/commands/`):

```
skills/<skill-name>/SKILL.md  →  parse frontmatter + markdown  →  Command
```

**Frontmatter fields** (`:185-264`): `name`, `description` (auto-extracted if
missing), `when-to-use`, `allowed-tools`, `argument-hint`, `user-invocable`
(default true), `model` (inherits parent by default),
`disable-model-invocation`, `effort`, `context: 'fork'` (run in a subprocess),
`agent: <name>` (delegate), `paths: [...]` (gitignore-style availability
filter), and `hooks: {...}`.

**Prompt building** (`:344-398`): prepend `Base directory: <skillRoot>` when the
skill has reference files; substitute `${CLAUDE_SKILL_DIR}` and
`${CLAUDE_SESSION_ID}`; execute inline shell `` !`…` `` (unless the skill came
from MCP); substitute `{arg}` placeholders.

The legacy `/commands/` loader (`:563-625`) supports both `cmd.md` files and
`cmd/SKILL.md` directories, building namespaces from directory structure
(`subdir:nested:skill-name`).

### 3. Plugin skills

Contributed by installed plugins (see below); namespaced
`<plugin>:<subdir>:<skill>`.

### 4. MCP skills (`src/skills/mcpSkillBuilders.ts`)

`loadedFrom: 'mcp'`. **Never execute inline shell** (untrusted). Tokens are
estimated from frontmatter only since full content isn't available upfront.

### Invocation

`/skill <name> [args]` (or model invocation through `SkillTool`) resolves
local → bundled → MCP. The `getPromptForCommand(args, context)` receives a
`ToolUseContext` (`getAppState`, `appendSystemMessage`, …). Duplicate skills are
detected via realpath (`getFileIdentity`). Usage is recorded for analytics, and
file-edit events can auto-activate conditional skills whose `paths` match.

---

## B. Plugins

Plugins are versioned, installable packages with an official marketplace,
custom marketplaces (GitHub/git/npm/local), versioned caching, dependency
resolution, and license checking.

### Manifest & schemas (`src/utils/plugins/schemas.ts`, ~1,681 lines)

`plugin.json` declares `name`, `description`, `version`, `author`, and arrays of
contributions: `commands`, `agents`, `hooks`, `mcp`, plus `dependencies`,
`license`, `keywords`, `repository`.

A `marketplace.json` lists entries with `source` (local path or git URL),
version, author, license, `autoUpdate`, dependencies.

Name validation is strict (`:216-246`): no spaces/path separators/`..`, not
reserved (`.`, `inline`, `builtin`); an official-name pattern blocks
impersonation (`:72`); non-ASCII homograph detection (`:79`); the `anthropics`
reserved namespace requires a `github.com/anthropics/` origin (`:119`).

### Loader (`src/utils/plugins/pluginLoader.ts`, ~3,302 lines)

Loading order: **session plugins (`--plugin-dir`) → marketplace plugins →
builtin plugins.** Key mechanics:
- Versioned cache at `~/.claude/plugins/cache/{marketplace}/{plugin}/{version}/`
  (`getVersionedCachePath`, `:172`); `seed/` probing for bring-your-own-cache
  (`:195`, `:220`).
- Acquisition: `gitClone` (shallow, submodules; `:534`), `installFromNpm`
  (global npm cache; `:492`), local copy.
- Optional **ZIP cache mode** (`:373-461`) trades disk for load latency
  (`convertDirectoryToZipInPlace` / `extractZipToDirectory`).
- Versioning via SemVer, git refs/tags/SHAs, or npm specifiers
  (`pluginVersioning.ts`).
- Validation (`validatePlugin.ts`): manifest Zod schema, source-file existence,
  path-traversal guard (`validatePathWithinBase`), dependency
  resolution/demotion; warnings collected as `PluginError[]` without failing the
  whole load.

### Contributions

| Contribution | Loader | Notes |
|---|---|---|
| Commands / skills | `loadPluginCommands.ts` | recursive walk; `SKILL.md` wins in a dir; namespaced; variable substitution |
| Hooks | `loadPluginHooks.ts` | `{ matcher, hooks: [...] }`; conditional `if: "Bash(git *)"`; command/prompt/http/agent types |
| Agents | `loadPluginAgents.ts` | `.md` definitions parsed like skills |
| MCP servers | `mcpPluginIntegration.ts` | stdio (`command: [...]`) or embedded `.mcpb`/`.dxt`; dynamic per-session ports |

### Marketplaces (`src/utils/plugins/marketplaceManager.ts`, ~2,643 lines)

Sources: `github` (`owner/repo`), `git` (url), `npm` (package), `local` (path).
Loading checks the cache, fetches + validates against `PluginMarketplaceSchema`,
and caches with a timestamp for auto-update checks. Official marketplaces default
to `autoUpdate: true`; `performStartupChecks()` refreshes stale ones (GrowthBook-
gated); `reconciler.ts` installs/uninstalls to match desired state. Enterprise
blocklists/allowlists are enforced (`isSourceAllowedByPolicy`,
`isSourceInBlocklist`).

### Lifecycle (`src/services/plugins/`)

`pluginCliCommands.ts` / `pluginOperations.ts` / `PluginInstallationManager.ts`
implement the `/plugin` command: `list`, `install name@marketplace`,
`uninstall`, `reload`, `enable`/`disable`. Install flow: parse `name@marketplace`
→ resolve version → resolve source → clone/download to temp → validate → copy to
versioned cache → load contributions → update `settings.json` `plugins: [...]` →
trigger a UI refresh.

After install/reload, app state sets `plugins.needsRefresh` and bumps
`mcp.pluginReconnectKey` to force MCP re-initialization (see
[11](11-configuration-and-state.md)).

## Skill ⇄ plugin synergy

```
Plugin loaded
  ├─ commands/ , skills/   → skill system
  ├─ agents/               → Agent commands
  ├─ hooks                 → hooks system
  └─ mcp                   → MCP servers in app state
```

Enterprise can lock customization to plugins only via
`strictPluginOnlyCustomization: 'disable'` in settings (ignores ad-hoc skills/
agents/hooks/MCP).

## Key files

| File | Role |
|---|---|
| `src/skills/bundledSkills.ts` | bundled skill registration/extraction |
| `src/skills/loadSkillsDir.ts` | file-based skill discovery & prompt building |
| `src/skills/mcpSkillBuilders.ts` | MCP skills (no inline shell) |
| `src/services/skillSearch/signals.ts` | remote skill-search signals |
| `src/utils/plugins/pluginLoader.ts` | plugin discovery, caching, validation |
| `src/utils/plugins/schemas.ts` | manifest/marketplace/name schemas |
| `src/utils/plugins/marketplaceManager.ts` | marketplace loading & auto-update |
| `src/services/plugins/pluginCliCommands.ts` | `/plugin` command |
| `src/services/plugins/PluginInstallationManager.ts` | install state |
