---
id: F-search-api-console-commands
title: "No search for console commands / CVars (escape-hatch is blind)"
status: DONE
severity: Medium
category: feature
tags: [search, console, cvar, discovery, escape-hatch]
---

# No search for console commands / CVars (escape-hatch is blind)

`editor.console_command` and `system.console_command` are
**execute-only** — they run a command string. There is no RPC to list,
search, or get help text for the thousands of registered console objects
(`r.*`, `p.*`, `ai.*`, `net.*`, `t.*`, `Slate.*`, `Stat *`, `showflag.*`,
`wp.*`, and per-plugin namespaces). The agent must already know the
exact command name to invoke it.

This is the **highest-leverage** discovery gap on the board because the
wiki explicitly designates `console_command` as the universal escape
hatch:

> This is the right escape hatch when no typed RPC exists for what you
> need — most editor cvars and many built-in commands have no
> first-class wrapper and have to be issued through here.

The whole point of an escape hatch is to cover *what isn't yet
wrapped*. Without discovery, the escape hatch only works when the
agent already knew the answer from training data or prior sessions —
defeating the purpose.

**Use cases blocked:**

1. "How do I disable virtual shadow maps for this test?" — agent knows
   the rendering category but not the exact CVar name
   (`r.Shadow.Virtual.Enable`). Today: read engine source or guess.
2. "Show me every rendering CVar related to lumen" — exploratory
   discovery for performance tuning.
3. Help-text access — every console object has a `GetHelp()` string
   that explains what it does. Currently invisible to the agent.
4. Current-value / default-value lookup before mutation — to write a
   safe `r.X 100`, the agent should be able to read `r.X`'s current
   and default values first.
5. Flag-aware filtering — `ECVF_Cheat`, `ECVF_ReadOnly`,
   `ECVF_RenderThreadSafe` flags matter for automation contexts
   (don't issue cheats in shipping config; don't try to set ReadOnly
   CVars; know which CVars are RT-safe).

**Current workarounds:**

- Read `Engine/Source/Runtime/RenderCore/Private/*.cpp` etc. for
  `IConsoleManager::Get().RegisterConsoleVariable` calls.
- `editor.console_command` with `Help`, `DumpConsoleCommands`, or
  `r.*` (typed prefix) — but the **output is not captured by the RPC**
  (per the `editor.console_command` notes); it goes to
  `Saved/Logs/PDS.log`, requiring a log-read round-trip.
- `python.execute` walking `unreal.SystemLibrary` or directly poking
  `IConsoleManager` via FFI.

All bypass the typed-RPC surface and add latency / fragility.

**Proposal:** Add `system.console.search` (substring/prefix search)
and `system.console.get` (exact lookup). The plumbing already exists in
`IConsoleManager`:

```cpp
IConsoleManager::Get().ForEachConsoleObjectThatStartsWith(prefix, callback);
IConsoleManager::Get().ForEachConsoleObjectThatContains(substring, callback);
IConsoleManager::Get().FindConsoleObject(name);
// per-object:
IConsoleObject::GetHelp();
IConsoleObject::IsVariable() / IsCommand();
IConsoleObject::GetFlags();   // ECVF_Cheat, ECVF_ReadOnly, ...
IConsoleVariable::GetString(); // current value
```

API shape:

```
system.console.search(
    query: string,
    matchMode?: "prefix"|"contains",  // default "contains"
    kind?: "variable"|"command"|"any",
    cheatOnly?: bool,                 // ECVF_Cheat filter
    excludeCheat?: bool,              // default true in shipping-aware contexts
    limit?: number                    // default 50
) -> {
    results: [{
        name: "r.ScreenPercentage",
        kind: "variable",
        help: "To render in lower resolution and upscale for better performance.\n  100 = 100% (default)\n  50 = 50% (half resolution per axis)",
        currentValue: "100.000000",
        defaultValue: "100.000000",
        flags: ["RenderThreadSafe", "Scalability"],
        setBy: "Scalability"          // ECVF_SetByMask source
    }, ...],
    totalMatches: number
}

system.console.get(name: string) -> { ...same single row..., found: bool }
```

Optional companions:

- `system.console.list_prefixes()` — top-level prefix histogram
  (`r.*: 1240`, `ai.*: 87`, `Slate.*: 42`, …) so the agent can narrow
  before searching. Cheap to compute by walking the registry once.
- `system.console.read(name)` — pure value-read variant of `get`
  (returns only current/default, no help) for hot paths.

**Cross-ref:** This is the canonical discovery gap. Every other ticket
in this batch ([anim graph nodes][1], [material expressions][2],
[Niagara modules][3], [native UClasses][4], [MetaSound][5], [Niagara
graph nodes][6]) addresses a domain-specific catalog; this one
addresses the **escape-hatch's own discoverability**. Highest signal of
the set.

[1]: F-search-api-anim-graph-nodes.md
[2]: F-search-api-material-expressions.md
[3]: F-search-api-niagara-modules.md
[4]: F-search-api-native-uclasses.md
[5]: F-search-api-metasound-nodes.md
[6]: F-search-api-niagara-graph-nodes.md

## History
- `#1-blind-escape-hatch` `OPEN` reporter — `editor.console_command` and `system.console_command` are execute-only with no peer for discovery. The wiki explicitly designates `console_command` as the universal escape hatch ("right escape hatch when no typed RPC exists"), but without search the escape hatch only works when the agent already knew the command name. `IConsoleManager` provides `ForEachConsoleObjectThatStartsWith` / `ForEachConsoleObjectThatContains` / per-object `GetHelp` / value accessors — the plumbing is trivial. Proposes `system.console.search` + `system.console.get` with keyword / kind / flag filters, returning name + kind + help + current value + default value + flags + setBy source. Highest-leverage of the discovery-gap ticket batch because it unblocks every command/CVar the typed surface hasn't wrapped.
- `#2-reviewed-and-confirmed` `OPEN` tester — Verified by source reading (editor offline). Confirmed `editor.console_command` (`Handlers/Editor/EditorCommandHandler.cpp:33`) and `system.console_command` are pure exec wrappers around `GEditor->Exec` with no output capture; no `console.search`/`console.list`/`console.get` handler exists anywhere under `Source/Handlers/`. `IConsoleManager` usage in-tree is limited to hardcoded named lookups in `PerformanceHandler.cpp` and friends — never exposed as a generic search surface. No duplicate ticket. Downgraded severity High→Medium: the `DumpConsoleCommands` + log-parse workaround is slow/fragile but not blocking, and sibling search tickets (anim graph, material expressions, etc.) have no analogous log-dump escape valve so their High claim is stronger than this one. MVP scope should be a single `system.console.search(query, kind?, limit?) -> [{name, kind, help, currentValue, flags}]` using `ForEachConsoleObjectThatContains`; the proposed `list_prefixes` / separate `read` / `cheatOnly`+`excludeCheat` filters / `setBy` source are YAGNI for the first cut.
- `#3-implemented-console-search` `IN-REVIEW` developer — Added `system.console.search` at `Private/Handlers/System/ConsoleSearchHandler.cpp` via `IConsoleManager::ForEachConsoleObjectThatContains` with kind / limit / flag-array shape per #2 MVP. Empty query rejected; flags surfaced via TestFlags on ECVF_* agent-relevant bits; `truncated` flag in response. 4 regression tests appended to `Tests/Private/EditorOps/TestSystemHandlers.cpp`.
- `#4-verify-console-search` `DONE` tester — Verified: `system.console.search` wiki exposes `query`, `kind`, and `limit`; live call `system.console.search({"query":"r.Shadow.Virtual","kind":"variable","limit":5})` returned `r.Shadow.Virtual.Enable` with `kind:"variable"`, help text, `currentValue:"0"`, flags `RenderThreadSafe`, `Scalability`, `Preview`, `totalMatches:106`, and `truncated:true`.
