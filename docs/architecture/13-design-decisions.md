# 13 — Design Decisions & Rationale

The other documents explain *how* each subsystem works. This one explains *why*
the recurring design choices were made — the trade-offs, the failure modes they
guard against, and the mental models behind them. Where the codebase states its
own rationale in comments, those are quoted directly.

---

## 1. Generators everywhere — streaming as the default

**Decision.** The whole request pipeline is composed of async generators:
`ask → QueryEngine.submitMessage → query → queryLoop → callModel`, each
`yield`-ing messages upward.

**Why.** A coding agent produces output incrementally — assistant text, then a
tool call, then a result, then more text — over many seconds. Buffering a whole
turn before showing anything would feel dead. Generators let every layer emit a
message *the instant it exists*, so the UI (or the SDK consumer, or the IDE
bridge) renders progressively. The same generator stream feeds the terminal, the
JSON `stream-json` output mode, and remote consumers without special-casing.

**The deeper payoff:** because the loop *is* a generator, control can be
suspended and resumed cleanly. Tool execution, error recovery (re-`continue`ing
`queryLoop`), and compaction all slot into the same stream without callbacks or
a separate event bus.

**Trade-off.** Generator composition is harder to reason about than a flat
function — ordering subtleties matter (e.g. assistant transcripts are recorded
fire-and-forget so the stream keeps flowing, but other messages are awaited to
preserve order; see [03](03-query-engine.md)).

---

## 2. Fail-closed where it matters, fail-open where it's safe

**Decision.** Tool capability defaults are conservative. `buildTool()` merges a
tool definition over `TOOL_DEFAULTS` (`Tool.ts:748-792`), whose comment reads
*"Defaults (fail-closed where it matters)"*:

```ts
isConcurrencySafe → false   // "assume not safe"
isReadOnly        → false   // "assume writes"
isDestructive     → false
toAutoClassifierInput → ''  // "skip classifier — security-relevant tools must override"
```

**Why.** A tool author who *forgets* to declare a property gets the safe
behavior: the tool runs serially, is treated as a writer, and isn't auto-cleared
by the classifier. You have to *opt in* to parallelism and read-only fast paths.
The cost of a wrong default is asymmetric — wrongly marking a destructive tool
"safe" could corrupt the user's machine, while wrongly marking a safe tool
"unsafe" just costs a little latency. The defaults bias toward the cheap mistake.

**But not everywhere.** Permission handling deliberately chooses *fail-open* in
some spots. In auto-mode, when the classifier is unavailable the code can
*"fall back to normal permission handling (fail open)"*
(`permissions.ts:872`) rather than hard-denying — because hard-denying every
tool when a background classifier hiccups would make the agent useless. Policy
limits are also fail-open (a failed policy fetch means "allowed unless
explicitly denied", [06](06-services-layer.md)). The principle is: **fail-closed
on irreversible local actions, fail-open on availability of an optional
optimization.**

The same fail-open spirit shows up in retries: an *untagged* call path is
treated as retryable — *"undefined → retry (conservative for untagged call
paths)"* (`withRetry.ts:85`).

---

## 3. Startup is a latency-hiding exercise, not a sequence

**Decision.** Boot does as little *sequentially* as possible. Slow I/O is kicked
off as early side effects and **overlapped** with module evaluation and with
each other (see [02](02-startup-and-runtime-modes.md)).

**Why.** Every millisecond before the prompt appears is felt on every launch.
The codebase is littered with comments naming the specific overlap windows it's
exploiting:

- *"Spawn git status/log/branch now so the subprocess execution overlaps…"*
  (`main.tsx:1967`) — *"runs during the ~280ms overlap window before the
  context"* (`main.tsx:1980`).
- *"Kicked off here to overlap with setup(); awaited [later]"* (`main.tsx:1781`),
  *"overlap config I/O with setup(), commands loading, and trust dialog"*
  (`main.tsx:1802`).
- *"Start hooks early so they run in parallel with MCP connections"*
  (`main.tsx:2432`).

**The mental model:** the critical path is "process start → first prompt
render." Anything not on that path (migrations, analytics, attribution hooks,
release notes) is **fire-and-forget** — *"Async migration - fire and forget
since it's non-blocking"* (`main.tsx:348`). Anything that *is* needed later but
slow (commands, agents, MCP configs, git status) is **started early and joined
just-in-time**.

**Trade-off.** This makes the boot sequence genuinely hard to follow — work is
in flight across many promises, and ordering bugs (awaiting too early/late) are
easy to introduce. The startup profiler ([02](02-startup-and-runtime-modes.md))
exists precisely because this complexity needs measurement to defend.

---

## 4. Tool execution streams *alongside* model output

**Decision.** The `StreamingToolExecutor` begins running a tool the moment its
`tool_use` block finishes streaming — while the model is still producing later
blocks (`StreamingToolExecutor.ts`, [04](04-tool-system.md)).

**Why.** The naive loop is "wait for the full assistant message → run all tools
→ send results → repeat." But the model often emits several tool calls in one
message. Running them only *after* the message completes wastes the streaming
window. Starting concurrency-safe tools as they arrive turns sequential
round-trips into overlapped work, so by the time the message ends, results are
often already in hand.

**Why it's safe.** This only parallelizes tools that declare themselves
concurrency-safe (read-only) — and remember, that property is fail-closed
(§2), so only deliberately-marked tools run early. Non-safe tools still serialize,
and a tool error aborts its in-flight siblings via a `siblingAbortController`.

---

## 5. Prompt-cache stability is a first-class constraint

**Decision.** Things that go into the API request in a list — the tool pool,
system-prompt segments — are **sorted/ordered deterministically**, even when
order is otherwise irrelevant. `assembleToolPool` *"Sort[s] each partition for
prompt-cache stability"* and places a *"global cache breakpoint after the last
prefix-matched built-in tool"* (`tools.ts:354-356`).

**Why.** Anthropic's prompt cache keys on a *prefix* of the request. If the tool
list or system prompt is rebuilt in a different order between turns (e.g. because
a `Map` iteration order shifted, or an MCP server reconnected), the cached prefix
no longer matches and the cache *breaks* — every token gets re-billed and
re-processed at full latency. So "irrelevant" ordering is not irrelevant: it's a
cost and latency lever. The system even ships **cache-break detection**
(`promptCacheBreakDetection.ts`) that watches for `cache_read` dropping >5%
turn-over-turn to catch regressions.

**Mental model:** treat the request prefix as a cache key, and treat anything
that perturbs it as a (real money) cost. This is why compaction defers its
boundary messages to report true `cache_deleted_input_tokens`
([03](03-query-engine.md)), and why background forked agents are *excluded* from
cache-break tracking — they're *"short-lived forked agents where cache break
detection provides no [value]"* (`promptCacheBreakDetection.ts:144`).

---

## 6. Recovery is layered, ordered, and bounded

**Decision.** Recoverable API errors (context overflow, max-output-tokens, media
size) are *withheld* from the stream and run through an **ordered cascade** of
recovery strategies, each of which either fixes the problem and re-enters the
loop or hands off to the next ([03](03-query-engine.md)).

**Why.** Different failures have different cheapest fixes, and you want the
cheapest one that works:

1. **Context overflow** → first *drain staged context-collapses* (cheap,
   reversible, preserves detail), then *reactive compaction* (lossy summary),
   then surface the error.
2. **Max output tokens** → escalate the cap *once* to 64k, then up to *3*
   multi-turn "resume" nudges, then surface.

Each path is **bounded** (one escalation, three recoveries,
`hasAttemptedReactiveCompact` latched) so a pathological turn can't loop forever.
The ordering encodes a preference: lossless before lossy, local before
re-querying, recover a few times before giving up.

**Mental model:** the loop is a small state machine whose `transition` field
(`'collapse_drain_retry'`, `'reactive_compact_retry'`, `'max_output_tokens_escalate'`, …)
names exactly which recovery put it back in the loop — so the next iteration can
behave differently and avoid re-trying a strategy that already failed.

---

## 7. Compaction is a graduated ladder, not a single hammer

**Decision.** There isn't one "shrink the context" operation; there are six,
tried in increasing order of information loss: snip → microcompact → context
collapse → autocompact → session-memory compact → reactive compact
([03](03-query-engine.md)).

**Why.** Summarizing the whole conversation (full compaction) is expensive (an
LLM call) and lossy (detail is gone forever). Most of the time you can avoid it:
dedup repeated tool results (microcompact), or fold granular context that can be
re-expanded (collapse), or drop only ancient history (snip). The ladder spends
the cheap, reversible reductions first and only reaches for the lossy summary
when nothing else gets under the threshold. Autocompact runs *proactively* near
the limit; reactive compaction is the *last resort* after an actual 413.

**Trade-off.** Six interacting mechanisms is a lot of surface area, gated behind
several feature flags, with careful guards so they don't fight (e.g. autocompact
is a no-op if collapse already got under threshold; both are skipped for
`session_memory`/`compact` query sources to prevent recursion).

---

## 8. Build-time feature flags = pay only for what ships

**Decision.** Optional features are gated behind `feature('FLAG')` from
`bun:bundle`, which the production bundler dead-code-eliminates
([02](02-startup-and-runtime-modes.md)).

**Why.** A single binary serves many cohorts (consumer, enterprise, internal,
experiments). Runtime `if`-flags would still ship all the code — and its
transitive imports — to everyone, bloating the bundle and startup parse time.
DCE means a build with `KAIROS` off contains *zero* Kairos code. The dev shim
(`plugins/bunBundleDev.ts`) reads `FEATURE_FLAGS` so the same source runs
locally with flags toggled.

**Why not just GrowthBook?** GrowthBook (runtime flags, [06](06-services-layer.md))
and `feature()` (build-time) coexist deliberately: build-time gates decide what
*code exists*; GrowthBook decides what *enabled code does* per user/org/cohort,
enabling gradual rollouts without a redeploy. The split is "compile-time
presence vs. runtime behavior."

---

## 9. Layered precedence as a universal pattern

**Decision.** Settings, permission rules, MCP server configs, and plugins all
resolve through the *same* conceptual stack: enterprise/policy → CLI flags →
local → project → user → built-in defaults.

**Why.** Claude Code runs for individuals *and* inside managed enterprises. A
single precedence model means "an org admin can always override a user, and a
user can always override a default" holds everywhere, predictably. It's also why
enterprise lock-downs (`strictPluginOnlyCustomization`,
`shouldAllowManagedPermissionRulesOnly`) work by *suppressing lower layers*
rather than needing bespoke enforcement per subsystem.

**Mental model:** every configurable thing is a merge of sources ranked by
authority; the only per-subsystem question is *how* entries merge (last-wins,
union, exclusive).

---

## 10. Immutable state with narrow, deliberate escape hatches

**Decision.** The app-state store is typed `DeepImmutable<{…}>`
(`AppStateStore.ts:89`), with only a few fields excluded — and each exclusion is
*justified in a comment*: tasks are excluded *"because TaskState contains
function types"* (`AppStateStore.ts:159`).

**Why.** Immutability makes React re-render decisions trivial (reference
equality), prevents action-at-a-distance bugs in a large concurrent UI, and lets
the store use `useSyncExternalStore` cleanly. The deliberate exclusions
acknowledge that a few things genuinely can't be frozen (live task handles with
callbacks, MCP client objects) — so rather than fighting the type, they're
carved out explicitly and documented, keeping the immutability guarantee honest
for everything else. There may be only one `AppStateProvider` (it throws if
nested), reinforcing a single source of truth.

---

## 11. Security is defense-in-depth, not a single gate

**Decision.** Dangerous operations pass through *multiple independent* layers
rather than one check. Bash, for example: static AST/shell-quote analysis → 15+
read-only/safety checks → permission rules → optional classifier → user prompt →
OS-level sandbox ([04](04-tool-system.md), [07](07-permissions-and-hooks.md),
[12](12-vendored-packages.md)).

**Why.** Each layer has blind spots. Static analysis can't know user intent; the
permission system can't catch a novel injection trick; the sandbox can't tell
"rm in /tmp" from "rm in ~". Stacking them means a bypass of one is caught by
another. The file tools add their own *hard* rules independent of permission
configuration — blocked device paths on read, UNC-path rejection on write
(NTLM-leak prevention), team-memory secret scanning — because some risks
shouldn't be configurable away at all.

**Mental model:** treat the model as untrusted input and the user's machine as
the asset. Configurable policy handles the common case; non-negotiable hard rules
handle the catastrophic case.

---

## 12. Hooks: extensibility with an integrity guarantee

**Decision.** Lifecycle hooks let users/plugins run arbitrary code at tool and
session boundaries — but the hooks config is **snapshotted at session start**
(`captureHooksConfigSnapshot` in `setup.ts`) and watched for changes
([07](07-permissions-and-hooks.md)).

**Why.** Hooks are powerful precisely because they execute arbitrary commands —
which is also the risk. Snapshotting means a hook file can't be swapped
mid-session to smuggle in new behavior after the user has already granted trust.
It's the same instinct as the trust dialog: establish what's trusted *once*, at a
well-defined boundary, and detect drift.

---

## 13. Telemetry that structurally can't leak code

**Decision.** Analytics metadata is type-restricted to
`boolean | number | undefined` — strings are *not allowed* — and PII-tagged
values must use a `_PROTO_*` key prefix routed only to first-party columns
([06](06-services-layer.md)).

**Why.** The biggest privacy risk in a coding tool's telemetry is accidentally
logging a file path, a snippet, or a prompt. Rather than relying on reviewers to
catch that, the *type system* forbids putting a string into an event — you
physically can't log free-form text through the normal path. PII that genuinely
must be captured takes an explicit, separately-routed, clearly-named channel.
This is "make the wrong thing impossible" rather than "make the wrong thing
discouraged."

---

## 14. Multi-provider / multi-transport behind stable abstractions

**Decision.** The same `Tool`, `Command`, and message types run against four API
providers (1P, Bedrock, Vertex, Foundry) and across local, headless, SDK,
IDE-bridge, remote (CCR), and SSH execution — with provider/transport differences
isolated behind `getAnthropicClient()` and the transport layer
([03](03-query-engine.md), [06](06-services-layer.md), [08](08-bridge-remote-multiagent.md)).

**Why.** Enterprises mandate specific clouds; some users run headless in CI;
others drive from an IDE or a phone. If those differences leaked into the agent
loop, every feature would need N variants. Instead the loop is written once
against an abstract client and message stream; providers are *dynamically
imported* so unused SDKs cost nothing; and remote execution is bridged by
*synthesizing local message/tool stubs* (`remotePermissionBridge.ts`) so the
permission UI doesn't even need to know a tool lives on a server.

**Mental model:** keep the core ignorant of where it runs and who serves the
model; push all the variance to the edges.

---

## Cross-cutting themes

If you abstract over the fourteen decisions above, a few values recur:

- **Make the wrong thing impossible or expensive** (fail-closed defaults,
  string-free telemetry, hooks snapshotting) rather than relying on discipline.
- **Latency and cache cost are correctness concerns**, not afterthoughts
  (startup overlap, streaming tools, prompt-cache stability, the compaction
  ladder).
- **Prefer cheap/reversible before expensive/lossy**, in an explicit ordered
  cascade (recovery, compaction).
- **One model applied everywhere** (layered precedence, generator streaming,
  stable abstractions) beats many special cases.
- **Bound everything** that could loop (recoveries, retries, classifier
  fallbacks) so no single turn can run away.

Next: [Phase 2 — Algorithm deep-dives](14-deep-dive-agent-loop.md) walks the
cleverest of these mechanisms line by line in the actual code.
