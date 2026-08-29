---
id: F-console-search-setby-priority
title: "`system.console.search` rows carry eight behaviour flags and none of the `ECVF_SetBy*` bits, so 'is this CVar pinned, and by what' is unanswerable over the wire"
status: OPEN
severity: Medium
category: feature
tags: [system, console, console-search, cvar, cvar-priority, ecvf-setby, readback, flags, scalability, diagnosis]
encounters: 1
lastSeen: 2026-08-29T21:30:00+03:00
---

# `system.console.search` reports what a CVar *is*, never who last *set* it

`system.console.search` returns a `flags` array per row. `FlagsToStrings`
(`Source/PinWright/Private/Handlers/System/ConsoleSearchHandler.cpp:23-48`) pushes exactly eight
**behaviour** flags — `Cheat`, `ReadOnly`, `RenderThreadSafe`, `Scalability`, `ScalabilityGroup`,
`Preview`, `ExcludeFromPreview`, `Unregistered` — and none of the `ECVF_SetBy*` bits. The
`ECVF_SetByMask` portion of the flags word (`IConsoleManager.h:150`) is never surfaced.
`currentValue` shows the value; nothing shows the priority it is held at.

The omission is **deliberate and already documented in the code**, which is why this is a feature
request and not a bug: `ConsoleSearchHandler.cpp:20-22` reads *"Maps the agent-relevant ECVF_* bits
(single-bit flags only) onto stable string tags. ECVF_SetByMask spans multiple bits and is
intentionally omitted per the MVP scope on this ticket."* This ticket is the request to lift that
MVP scope, with the evidence for why it now earns its cost.

So a caller can read what a CVar holds and cannot read whether that value is **pinned** — whether
the next `Set` at a lower priority will be silently discarded by `FConsoleVariableBase::CanChange`
(`ConsoleManager.cpp:275-311`). The only evidence available today is a
`LogConsoleManager: Warning` emitted at the moment of a *later, failed* write, in the editor log,
attributed to that write rather than to whatever created the pin.

## Why this is worth a ticket rather than a shrug

Split out of `B-console-command-sg-cvar-pin-freezes-scalability`, which named it explicitly as
separable and out of its own scope. That ticket is the demonstration: seven `sg.*` groups were
pinned at `ECVF_SetByConsole` by console lines, the editor's own Scalability panel — writing at
`ECVF_SetByScalability` — was silently outranked for the rest of the session, and **no verb in the
RPC surface could observe it**. The pin went unnoticed for a whole session by both the agent and,
for a while, the user. The refusal that ticket shipped stops the accident from *our* verbs; it does
nothing for a group pinned by a device profile, a `[ConsoleVariables]` ini section, a command line,
a `force: true` line, or another tool.

`performance.set_scalability` already pays for this gap in a narrower way: it reads the eleven
`sg.*` groups back and reports `requestedLevelApplied` precisely because a caller cannot otherwise
tell "the level landed" from "a higher pin shadowed it". That is a per-verb workaround for a
missing general readback.

## What it should do

Add the setter priority to each `system.console.search` variable row. Everything needed is already
in hand at `ConsoleSearchHandler.cpp:23-48` — the `IConsoleObject*` is the input:

```
GetFlags() & ECVF_SetByMask   ->   GetConsoleVariableSetByName(...)
```

`GetConsoleVariableSetByName` is the engine's own flag-to-name function, so the vocabulary comes
from the engine rather than from a table this plugin has to keep in sync with fifteen priorities.

Shape suggestion, deliberately additive so no existing consumer breaks: a new per-row string field
(e.g. `setBy: "SetByScalability"`) alongside `currentValue`, rather than another entry in the
`flags` array — `flags` documents what the CVar *is*, and mixing a mutable "who set it last"
into it invites callers to treat the two as the same kind of fact. A `setBy` filter on the query
would answer "what is pinned above Scalability right now" in one call, which is the diagnostic the
parent ticket wanted and could not express.

**Considered and rejected as the primary fix:** folding this into `F-console-batch-get-cvar-values`.
That ticket is about step count and value echo — exact/batch reads of values a caller already knows
the names of. This is a *missing field*, not a missing call shape: batching would still return rows
that cannot answer the question. The two are complementary and neither subsumes the other.

## History
- `#1-split-from-the-console-pin-ticket` `OPEN` reporter — Filed as the separable follow-up
  `B-console-command-sg-cvar-pin-freezes-scalability` named and explicitly kept out of its own
  scope, while implementing that ticket's refusal. Verified at that HEAD: `FlagsToStrings`
  (`Handlers/System/ConsoleSearchHandler.cpp:23-48`) emits eight behaviour flags and none of the
  `ECVF_SetBy*` bits, so `ECVF_SetByMask` (`IConsoleManager.h:150`) is unreachable over the wire and
  no verb can report that a CVar is pinned or at what priority. Evidence that this matters: the
  parent ticket's measured session pinned seven `sg.*` groups at `ECVF_SetByConsole`
  (`IConsoleManager.h:187`), which permanently outranked the editor's own Scalability panel at
  `ECVF_SetByScalability` (`:159`, written by `SScalabilitySettings.cpp:72` ->
  `Scalability.cpp:907`), and the only trace was a `LogConsoleManager: Warning` from
  `FConsoleVariableBase::CanChange` (`ConsoleManager.cpp:286-308`) raised hours later against an
  innocent write. The refusal shipped there covers only pins *this plugin's verbs* would create;
  device profiles, ini sections, command lines and `force: true` still pin invisibly. Proposed fix:
  surface `GetFlags() & ECVF_SetByMask` through the engine's own `GetConsoleVariableSetByName` as a
  new additive per-row `setBy` string (not another `flags` entry — `flags` describes what the CVar
  is, `setBy` describes who last wrote it), optionally filterable. Severity Medium on the rubric's
  "a readback omits a field and forces a fallback" band, where the fallback is grepping the editor
  log for a warning that only appears on a later failed write; reach modifier declined in both
  directions and the argument stated — `system.console.search` runs in a large fraction of sessions
  which argues up, while "who set this" is a rare question within it which argues down, so the
  bands cancel and Medium stands on impact alone.
