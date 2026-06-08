# 12 — Vendored / External Packages (Sandbox & MCPB)

The leaked `src/` tree depends on several Anthropic-published packages that were
**not** part of the source-map leak. In this repo they exist only as no-op
**stubs** under `stubs/` so the project compiles (`package.json:14-21`). This
document fills in the **real behavior** of the two that are **public on npm**,
fetched and inspected directly:

| Package | Public? | npm version inspected | Stub in repo |
|---|---|---|---|
| `@anthropic-ai/sandbox-runtime` | ✅ public | `0.0.54` | `stubs/anthropic-ai/sandbox-runtime/` |
| `@anthropic-ai/mcpb` | ✅ public | `2.1.2` | `stubs/@anthropic-ai/mcpb/` |
| `@ant/computer-use-mcp`, `@ant/computer-use-input`, `@ant/computer-use-swift`, `@ant/claude-for-chrome-mcp` | ❌ private (`@ant` internal registry) | — not obtainable | `stubs/@ant/*` |

> **Caveat:** the leaked CLI was built against whatever versions existed at leak
> time. The public versions documented here are current and may differ in
> detail, but every symbol the leaked `src/` imports was verified to still exist
> in them (see below). The hand-written stubs in this repo do **not** faithfully
> mirror the real APIs — e.g. the sandbox stub exposes a `SandboxManager` class
> with `start()`/`stop()`, whereas the real export is a singleton object with
> `initialize()`/`wrapWithSandbox()`; the mcpb stub exports
> `mcpbRun`/`mcpbInstall`/`validateManifest`, none of which the real code
> actually imports.

---

## `@anthropic-ai/sandbox-runtime` (SRT)

> *"A lightweight sandboxing tool for enforcing filesystem and network
> restrictions on arbitrary processes at the OS level, without requiring a
> container."* — package README

This is the engine behind Claude Code's sandboxed Bash/PowerShell execution
(see [04 — Tool System](04-tool-system.md), `shouldUseSandbox`). It is a
**research-preview, open-source** package.

### How it sandboxes

Secure-by-default, using native OS primitives (no container needed):

- **macOS** — `sandbox-exec` with dynamically generated Seatbelt (`.sb`)
  profiles.
- **Linux** — `bubblewrap` (`bwrap`) for containerization + network-namespace
  isolation, optionally `socat` and a seccomp BPF filter (to block unix-domain
  sockets).
- **Windows** — a dedicated OS group + Windows Filtering Platform (WFP) sublayer
  (install/uninstall flow).

**Dual isolation model** (both required to be effective):
- **Filesystem — read** uses *deny-then-allow*: read is allowed everywhere by
  default; you deny broad regions (e.g. `/Users`) then re-allow specific paths
  (e.g. `.`). `allowRead` beats `denyRead`.
- **Filesystem — write** uses *allow-only*: write is denied everywhere by
  default; you must explicitly allow paths. `denyWrite` beats `allowWrite`.
- **Network** is allowlist-based, enforced by a local HTTP/SOCKS proxy. It can
  even MITM-terminate TLS for per-request filtering (`mitm-ca`, `mitm-leaf`,
  `tls-terminate-proxy`, `request-filter`), and chain through a corporate
  `parentProxy`.

### Real API surface (from `dist/index.d.ts`)

The real package exports (not the stub's invented shape):

- **`SandboxManager`** — a **singleton object** implementing `ISandboxManager`,
  run on the host (outside the sandbox). Key methods:
  - `initialize(runtimeConfig, sandboxAskCallback?, enableLogMonitor?)`
  - `isSupportedPlatform()`, `isSandboxingEnabled()`, `checkDependencies()`
  - `wrapWithSandbox(command, binShell?, customConfig?, abortSignal?)` →
    sandboxed shell string, and `wrapWithSandboxArgv(...)` → `{ argv, env }`
  - `getFsReadConfig()` / `getFsWriteConfig()` / `getNetworkRestrictionConfig()`
  - `waitForNetworkInitialization()`, proxy-port getters, `getMitmCA()`
  - `getSandboxViolationStore()`, `annotateStderrWithSandboxFailures()`,
    `getLinuxGlobPatternWarnings()`, `cleanupAfterCommand()`, `reset()`
- **`SandboxViolationStore`** — an in-memory ring buffer of
  `SandboxViolationEvent`s with `addViolation`, `getViolations(limit?)`,
  `getViolationsForCommand(command)`, `subscribe(listener)`. On macOS it taps
  the system sandbox-violation log store for real-time alerts.
- **`SandboxRuntimeConfigSchema`** (+ `NetworkConfigSchema`,
  `FilesystemConfigSchema`, `IgnoreViolationsConfigSchema`, `RipgrepConfigSchema`,
  `WindowsConfigSchema`, …) — Zod schemas. The top-level config is:
  ```ts
  SandboxRuntimeConfig = {
    network: { allowedDomains, deniedDomains, allowUnixSockets?,
               allowLocalBinding?, allowMachLookup?, httpProxyPort?,
               socksProxyPort?, mitmProxy?, filterRequest?, tlsTerminate?,
               parentProxy? },
    filesystem: { denyRead, allowRead?, allowWrite, denyWrite, allowGitConfig? },
    ignoreViolations?, enableWeakerNestedSandbox?, enableWeakerNetworkIsolation?,
    allowAppleEvents?, ripgrep?, allowPty?, seccomp?, bwrapPath?, socatPath?,
    windows?
  }
  ```
- Windows install helpers, `getDefaultWritePaths()`, `getWslVersion()`, etc.

It also ships an `srt` CLI: `srt "curl example.com"` runs the command sandboxed;
blocked network/filesystem access fails with `Operation not permitted` /
`Connection blocked by network allowlist`.

### How the leaked CLI uses it

Via an **adapter** at `src/utils/sandbox/sandbox-adapter.ts`, which imports the
real `SandboxManager`, `SandboxRuntimeConfigSchema`, and `SandboxViolationStore`
and bridges them to Claude's settings system and path conventions. The adapter:
- translates Claude Code path patterns (`~/path`, `./path`, `.claude/...`) into
  SRT's filesystem rules and protects dangerous directories;
- handles the `argv0='rg'` embedded-ripgrep dispatch;
- exposes a Claude-specific manager (`sandbox-adapter.ts:925`) consumed by
  `BashTool` (`shouldUseSandbox.ts`, `BashTool.tsx`, `bashPermissions.ts`,
  `readOnlyValidation.ts`), `PowerShellTool`, `main.tsx`, and the Doctor
  diagnostics (`SandboxDependenciesTab.tsx` shows `bwrap`/`socat`/`seccomp`
  install status).

So Claude Code's "sandboxed Bash" works by calling `wrapWithSandbox()` to rewrite
a command into an OS-sandboxed invocation, then surfacing any violations from the
`SandboxViolationStore` (including annotating failing stderr).

---

## `@anthropic-ai/mcpb` (MCP Bundles)

> *"MCP Bundles (`.mcpb`) are zip archives containing a local MCP server and a
> `manifest.json`… enabling end users to install local MCP servers with a single
> click."* — package README

This is the format/toolchain behind plugin-bundled MCP servers (see
[09 — Skills & Plugins](09-skills-and-plugins.md)). It is the renamed successor
to **DXT** (`.dxt` → `.mcpb`, `@anthropic-ai/dxt` → `@anthropic-ai/mcpb`) — which
is why the leaked source has both `src/utils/dxt/` and `src/utils/plugins/`
references.

### What it is

An `.mcpb` file is a zip archive (spiritually like `.crx`/`.vsix`) containing a
complete local MCP server plus a versioned `manifest.json` describing the server,
its capabilities, and user-configurable options. Apps that support MCPB can
install such a server in one click, with automatic updates and config prompts.

### Real API surface (from `dist/index.d.ts`)

The barrel re-exports CLI ops (`init`, `pack`, `unpack`), node ops
(`files`, `sign`, `validate`), the versioned manifest schemas
(`schemas/0.1`…`0.4`, exporting `McpbManifestSchema`), shared config/constants,
and the `McpbManifest` union type. The symbols the leaked CLI actually imports —
all verified present in v2.1.2 — are:

- **`McpbManifest`**, **`McpbUserConfigurationOption`** (types) — used in
  `src/utils/plugins/mcpbHandler.ts` and `src/utils/dxt/helpers.ts`.
- **`McpbManifestSchema`** (Zod, from `schemas/`) — used by
  `src/utils/dxt/helpers.ts:validateManifest()` to validate a manifest.
- **`getMcpConfigForManifest(options)`** (from `shared/config.js`) — resolves a
  manifest into a runnable MCP server config
  (`mcpbHandler.ts:420`), returning `manifest.server.mcp_config`.

Both imports are **lazy** (`await import('@anthropic-ai/mcpb')`) precisely
because the barrel eagerly pulls in ~700 KB of Zod v3 schemas — the source
comments call this out (`mcpbHandler.ts:418`, `dxt/helpers.ts:8`).

### How the leaked CLI uses it

When a plugin contributes an MCP server as an `.mcpb`/`.dxt` bundle
([09](09-skills-and-plugins.md), `mcpPluginIntegration.ts`), the CLI:
1. validates the bundle's `manifest.json` against `McpbManifestSchema`
   (`validateManifest` in `utils/dxt/helpers.ts` and `utils/plugins/validatePlugin.ts`,
   surfaced by `/plugin validate` → `commands/plugin/ValidatePlugin.tsx`);
2. resolves it to a concrete MCP server config via `getMcpConfigForManifest()`
   (`mcpbHandler.ts`), substituting user-configuration options;
3. registers that server through the normal MCP connection path
   ([06 — Service Layer](06-services-layer.md)).

---

## The private `@ant/*` packages (not obtainable)

These remain stubs because they live in Anthropic's internal `@ant` registry and
were never published publicly:

- **`@ant/computer-use-mcp`** — the computer-use MCP server
  (`buildComputerUseTools`, `createComputerUseMcpServer`, grant flags, image
  resizing). Invoked via `cli.tsx --computer-use-mcp` and in-process MCP
  ([06](06-services-layer.md)).
- **`@ant/computer-use-input`**, **`@ant/computer-use-swift`** — native input
  injection / macOS Swift helpers for computer use.
- **`@ant/claude-for-chrome-mcp`** — the Chrome-integration MCP server
  (`cli.tsx --claude-in-chrome-mcp`, `--chrome-native-host`).

For these, the documentation describes only the **call sites** in `src/`
(arguments, where they're wired in); the implementations are genuinely absent
from any public source.

## Reproduction

```bash
npm pack @anthropic-ai/mcpb @anthropic-ai/sandbox-runtime   # download tarballs
tar xzf anthropic-ai-sandbox-runtime-*.tgz                  # extract → package/
# inspect package/dist/index.d.ts and package/README.md
```
