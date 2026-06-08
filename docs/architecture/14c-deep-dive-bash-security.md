# 14c — Deep Dive: Bash Command Security & Read-Only Validation

A line-by-line walk of how Claude Code decides whether a shell command is
*safe to auto-run* (read-only / concurrency-safe) and *not an injection attempt*.
This is the highest-stakes static-analysis in the codebase: the model proposes
arbitrary shell, and these checks stand between it and the user's machine. See
[04](04-tool-system.md) and [07](07-permissions-and-hooks.md) for context.

The defense is **layered** — five independent stages, any of which can reject:

```
command string
  → ast.ts pre-checks         (parser-differential traps: control chars, unicode ws, …)
  → bashSecurity.ts checks    (enumerated injection patterns)
  → ast.ts AST walk           (fail-closed allowlist of node types)
  → readOnlyValidation.ts     (flag-by-flag command allowlist)
  → bashPermissions.ts        (map to a permission rule)
  → shouldUseSandbox.ts       (OS sandbox containment)
```

## Stage 1 — Parser-differential pre-checks (`src/utils/bash/ast.ts:408-432`)

These run **before** any parser, because they target cases where the security
analyzer (shell-quote / tree-sitter) and the *actual shell* would disagree — the
classic way to smuggle a command past a validator:

- **Control characters** (`:408`) — block `0x00–0x08`, `0x0B–0x1F`, `0x7F`. These
  are silently dropped by some tokenizers but executed by bash.
- **Unicode whitespace** (`:411`) — block NBSP, zero-width space, line/paragraph
  separators, BOM. The user can't see them in the terminal, so they're a perfect
  hiding spot.
- **Backslash-whitespace** (`:414`) — block `\ ` and mid-word `\`-newline
  continuations that rejoin tokens differently than the analyzer expects.
- **Zsh `~[...]`** dynamic dir expansion (`:420`) and **Zsh `=cmd` EQUALS
  expansion** (`:426`).
- **Brace + quote** (`:432`).

## Stage 2 — Enumerated security checks (`src/tools/BashTool/bashSecurity.ts`)

Each check has a stable numeric ID in `BASH_SECURITY_CHECK_IDS` (`:77-101`) so
minified production builds can report *which* check fired without shipping the
descriptive string. The full set:

| # | Check | Lines | Attack blocked |
|---|---|---|---|
| 1 | `INCOMPLETE_COMMANDS` | `:244-286` | leading-operator / fragment chaining (`&&`, `;`, `\|\|`, leading tab/flags) |
| 2 | `JQ_SYSTEM_FUNCTION` | `:742-781` | `jq 'system("…")'` arbitrary command execution (`/\bsystem\s*\(/`) |
| 3 | `JQ_FILE_ARGUMENTS` | `:742-781` | `jq -f / --rawfile / -L …` file injection |
| 4 | `OBFUSCATED_FLAGS` | `:1130-1264` | ANSI-C `$'…'` / locale `$"…"` quoting, empty-quote-pair flag hiding |
| 5 | `SHELL_METACHARACTERS` | `:783-821` | `;\|&` inside `find -name`/`-regex`/`grep` patterns |
| 6 | `DANGEROUS_VARIABLES` | `:823-844` | `$VAR` adjacent to `<>\|` redirection operators |
| 7 | `NEWLINES` | `:905-941` | multi-command injection via embedded `\n`/`\r` |
| 8 | `…COMMAND_SUBSTITUTION` | `:846-873` | backticks, `$()`, `${}`, `$[]`, process subst, Zsh expansions |
| 9 | `…INPUT_REDIRECTION` | `:875-903` | `<` reading sensitive files |
| 10 | `…OUTPUT_REDIRECTION` | `:875-903` | `>`/`>>` overwriting system files |
| 11 | `IFS_INJECTION` | `:1017-1036` | `$IFS` / `${…IFS}` to rebuild blocked tokens from whitespace |
| 12 | `GIT_COMMIT_SUBSTITUTION` | `:612-740` | command substitution hidden in `git commit -m "…"`; chaining via remainder |
| 13 | `PROC_ENVIRON_ACCESS` | `:1041-1067` | reading `/proc/*/environ` to exfiltrate env vars |
| 14 | `MALFORMED_TOKEN_INJECTION` | `:1069-1128` | unbalanced tokens + separators that `eval` differently |
| 20 | `ZSH_DANGEROUS_COMMANDS` | `:43-74` | `zmodload`, `emulate`, `sysopen`, `zpty`, `ztcp`, `zsocket`, `mapfile`, `zf_*` |
| 22 | `QUOTED_NEWLINE` (`validateCarriageReturn`) | `:971-1015` | `\r` that shell-quote splits but bash treats literally |

### The command-substitution pattern table (`:16-41`)

`COMMAND_SUBSTITUTION_PATTERNS` enumerates every expansion form that could run a
nested command. The standouts, with the attack each blocks:

- `<(…)` / `>(…)` process substitution; `=(…)` Zsh process substitution.
- **Zsh EQUALS expansion `=cmd`** (`:20-26`): `=curl evil.com` expands to
  `/usr/bin/curl evil.com`, which **bypasses a `Bash(curl:*)` deny rule** because
  the parser sees the base command as `=curl`. Matches only word-initial
  `=[a-zA-Z_]`, never `VAR=val`.
- `$()`, `${}`, `$[…]` legacy arithmetic, `~[…]` Zsh param expansion.
- Zsh glob qualifiers `(e:…)` / `(+…)` and try/`always{…}` blocks.
- `<#` PowerShell comment — included as *defense-in-depth* (`:40`).

### Safe heredocs are explicitly allowed (`:317-514`)

Because the tool legitimately needs heredocs (e.g. `$(cat <<'EOF' … EOF)`),
`isSafeHeredoc` carves out a narrow exception. The gates are strict and the
comments explain why each exists:
- the delimiter **must** be single-quoted/escaped so the body is literal;
- matching is **line-based, not regex** (`:350-456`) to replicate bash's
  first-match closing behavior;
- the remaining text after stripping the heredoc must match a tight ASCII
  allowlist (`:500`) and then **recursively** pass the validator (`:510`).

## Stage 3 — The AST walk (`src/utils/bash/ast.ts`)

After pre-checks, the command is parsed and walked. `DANGEROUS_TYPES`
(`:186-205`) lists node types that immediately disqualify (command/process
substitution, subshells, expansions, control flow). The walk is **fail-closed**:
any node type the analyzer doesn't recognize returns `too-complex`, which means
"not provably safe → ask the user." Variable scope is tracked across pipes,
loops, and conditionals (placeholders like `__CMDSUB_OUTPUT__`,
`__TRACKED_VAR__`) so a value can't leak between scopes and produce a
false "safe."

## Stage 4 — Read-only allowlist (`src/tools/BashTool/readOnlyValidation.ts`)

Determines whether each subcommand only *reads*. Mechanics:

- **`stripSafeWrappers`** (`:524`) peels env-var assignments and known-safe
  wrappers (`timeout`, `time`, `nice`, `nohup`, `stdbuf`) in two phases
  (`:580-612`), but only **safe** env vars (the `SAFE_ENV_VARS` set, `:378-430`).
- **`isCommandSafeViaFlagParsing`** (`:1246`) matches multi-word commands
  longest-first (`git diff`, `git stash list`, `:1284-1300`) and validates flags
  via `validateFlags()`. It **rejects any token containing `$`** (`:1355`)
  because variable expansion defeats prefix-rule matching, and rejects brace
  expansion (`:1366`).
- For **deny/ask** rules it strips *all* leading env vars aggressively
  (`:826-853`) so `FOO=bar denied_command` can't dodge a deny rule, iterating to
  a fixed point.

`BashTool.isConcurrencySafe(input)` simply delegates to `isReadOnly`
(`BashTool.tsx:434-436`): a command is parallel-safe iff every part is read-only.

## Stage 5 — Permission-rule matching (`src/tools/BashTool/bashPermissions.ts`)

`getSimpleCommandPrefix` (`:161-188`) extracts the rule key (e.g. `git commit`,
`npm run`): it skips safe env vars, then returns the first two tokens if the
second looks like a subcommand (`/^[a-z][a-z0-9]*(-[a-z0-9]+)*$/`). If a
**non-safe** env var is present it returns `null` (fall back to exact match) so
it never generates a useless rule. A key security property: **compound commands
do not match prefix/wildcard rules** (`:884-931`) — so `Bash(cd:*)` will not
silently authorize `cd /path && python3 evil.py`.

## Stage 6 — Sandbox containment (`src/tools/BashTool/shouldUseSandbox.ts:130-153`)

Independent of the above, if OS sandboxing is enabled
(`SandboxManager.isSandboxingEnabled()`) and the command isn't on the
user/ANT exclusion list, the command runs inside the sandbox
([12](12-vendored-packages.md)) — so even a missed check is contained by the OS.

## The mental model

Treat the command string as **hostile input from an untrusted parser-author**.
The threat isn't just "dangerous command" — it's "command that *looks* safe to
my analyzer but means something else to the real shell." Hence:
- **Stage 1** closes the gap between analyzer and shell (control chars, invisible
  whitespace, EQUALS expansion).
- **Stages 2–3** enumerate known-bad and then **fail closed** on anything not
  provably benign.
- **Stage 4** is an *allowlist*, not a blocklist — only recognized read-only
  command/flag combinations pass.
- **Stages 5–6** bound the blast radius (rule matching that refuses to authorize
  compound commands, plus OS-level sandboxing).

No single layer is trusted to be complete; each assumes the others have holes.

Next: [14d — The compaction cascade](14d-deep-dive-compaction.md).
