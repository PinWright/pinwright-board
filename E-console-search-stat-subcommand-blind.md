---
id: E-console-search-stat-subcommand-blind
title: "system.console.search docs oversell `Stat *` coverage — a `stat unit`-style multi-token query false-negatives, defeating 'confirm the exact name'"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs, console, search, discovery, system, stat]
---

# system.console.search docs oversell `Stat *` coverage — a `stat unit`-style multi-token query false-negatives

`system.console.search` is the discovery surface the wiki tells agents to
call *before* `system.console_command` to "confirm the exact name" of an
unfamiliar command. In practice, searching for a full multi-token command
line such as `stat unit` returns **zero** results — even though `stat unit`
is a perfectly valid command that executes successfully through
`system.console_command` on the very next call. The discovery surface fails
to confirm a command that demonstrably works, so the "search first" step it
advertises produces a false negative and the agent burns retries before
giving up on it.

## What's awkward

The search uses `IConsoleManager::ForEachConsoleObjectThatContains(*Query)`
(`Handlers/System/ConsoleSearchHandler.cpp:174`), which substring-matches
against registered console **object names**. Two things break the
`stat unit` lookup:

1. **`stat <group>` is not one console object.** `stat` is an `Exec`-style
   command and `unit` is a stat group resolved by the stats system, not a
   discrete `IConsoleObject` named `"stat unit"`. (The `Stat <Group>`
   entries live in `UConsole`'s autocomplete list, built in engine
   `Console.cpp` via `AutoCompleteList[].Command = "Stat " + groupName`, not
   registered through `IConsoleManager::RegisterConsoleCommand`.) So a
   literal `"stat unit"` substring never matches an object name and
   `totalMatches` is 0. The handler's own test
   (`TestSystemHandlers.cpp:652` `system.console.search.StatCommandsFound`)
   only proves that the bare prefix `"Stat "` (trailing space) returns
   command rows — it never asserts that a full `stat <subcommand>` line
   resolves, which is exactly the query shape an agent forms from a task like
   "confirm the exact name of `stat unit`".

2. **The wiki oversells the coverage.** `docs/wiki-src/system.md:120` lists
   `Stat *` among the prefixes `system.console.search` covers, with no
   caveat that you must search the bare token (`stat`) rather than the full
   command line (`stat unit`), and no note that stat subcommands / `Exec`
   commands aren't enumerable as console objects at all. An agent reading
   that line reasonably searches `"stat unit"` and concludes the command
   doesn't exist.

## What it should do

**This is a docs-only fix.** The handler behaves exactly as designed —
case-insensitive substring matching over console-object NAMES — and `"stat
unit"` simply isn't a registered object-name substring. The defect is that
the wiki overlay doesn't tell the agent the match is name-only/single-token,
so an agent forming `"stat unit"` from a task is following the docs into a
false negative. Fix the docs to match reality:

- On `docs/wiki-src/system.md` under "Console command discovery", state that
  the search matches *console-object names only* (single token, no embedded
  spaces) — so to confirm a `stat <group>` / `show <flag>` / multi-word
  command, search the bare leading token (`stat`, not `stat unit`) and scan
  the rows, and warn that pure `Exec` commands and stat-group subcommands may
  not appear at all. Qualify the `Stat *` claim on line 120 so it doesn't
  promise full-command-line resolution the registry can't deliver.

This keeps `system.console.search` honest as the "confirm before execute"
step the wiki sells it as, instead of bouncing the agent into trial-and-error.

**Scope note (dropped):** an earlier draft proposed an optional behavioral
fallback (auto-retry a whitespace query against the leading token, or emit a
0-result hint). That is dropped: it touches the load-bearing per-row search
handler and invents new response-shape surface (an implicit second query, a
`searchedToken`/`note` field) for a Low-severity confirm-step convenience —
on a file (`ConsoleSearchHandler.cpp`) + wiki section + test that were just
modified by the sibling `E-console-search-default-limit-spills` projection
work. Stacking a behavioral auto-retry on top of that in-flight change is
churn-for-churn that the cheap, honest docs correction already neutralizes.

**Fix:** Qualify the `Stat *` claim and document name-only/single-token
matching (bare-leading-token guidance + the `Exec`/stat-subcommand
invisibility caveat) in the "Console command discovery" section of
`docs/wiki-src/system.md`. No handler/response-shape change. Pin the
documented contract with a regression test asserting the multi-token
`"stat unit"` false-negative (0 matches) vs. the bare `"Stat "` token (rows
returned).

## Evidence

From a system-namespace health-audit fuzz task (seed `system.inspect.*`),
step 4 ("confirm the exact name" of `stat unit` via `system.console.search`):

- `system.console.search { query:"stat unit", kind:"command" }` -> **0 results**
- `system.console.search { query:"stat unit" }` (no kind) -> **0 results**
  (trial-and-error retry on the same method, dropping the `kind` filter, still empty)
- `system.console_command "stat unit"` -> **success** on the next call — the
  command the search couldn't find runs fine.

Friction note: *"Search for the 'stat unit' command ... to confirm the exact
name. ... stat unit kind=command -> 0 results; stat unit (no kind) -> 0
results."* The discovery surface the task was told to trust returned a clean,
believable empty result for a real command — the worst shape for a "confirm
before you run it" tool. Two wasted discovery calls before the agent fell
back to executing the command blind.

This is distinct from `B-system-run-tests-no-completion-signal` (the step-6
job-completion bug the judge filed) and from `F-search-api-console-commands`
(DONE — that added `system.console.search`; it never addressed multi-token /
stat-subcommand confirmation or documented the name-only matching limit).

## History
- `#1-stat-unit-search-blind` `OPEN` reporter — `system.console.search { query:"stat unit" }` returned 0 results twice (with and without `kind:"command"`) during a health-audit task that instructed "confirm the exact name" via search, then `system.console_command "stat unit"` succeeded immediately after. Root cause: `ForEachConsoleObjectThatContains` matches console-object names only, and `stat <group>` is not a single console object — so a full multi-token command line never matches. `docs/wiki-src/system.md:121` advertises `Stat *` as searchable with no caveat about token-only matching or `Exec`/stat-subcommand invisibility, and the handler's `StatCommandsFound` test (`TestSystemHandlers.cpp:611`) only covers the bare `"Stat "` prefix, not a full subcommand. Propose: (a) document the name-only matching limit + bare-token guidance on the `system.md` overlay and qualify the `Stat *` claim; optionally (b) auto-retry whitespace queries against the leading token (or at least emit a hint on 0-result whitespace queries) so the "confirm before execute" step stops producing false negatives.
- `#2-reword-docs-only-and-implement` `IN-REVIEW` developer — Rewored to **docs-only** scope and implemented. Three validity lenses agreed the false-negative is real and source-verified (handler at `ConsoleSearchHandler.cpp:174` substring-matches console-object NAMES; `stat unit` is built into `UConsole`'s autocomplete list in engine `Console.cpp`, not the console-object registry, so it never matches), but the adversarial lens correctly flagged the optional auto-retry/hint code change as low-value churn on a load-bearing handler already under the in-flight `E-console-search-default-limit-spills` edit. Dropped option (b); the handler behaves as designed — the defect is the docs overselling `Stat *`. Fixed `docs/wiki-src/system.md` "Console command discovery": qualified the `Stat *` claim and added a paragraph stating the match is console-object NAMES only (single token, no embedded spaces), with bare-leading-token guidance (`Stat`, not `stat unit`), the `Exec`/stat-subcommand-invisibility caveat, and "treat a 0-result whitespace query as 'not an object name', not 'command doesn't exist'". Pinned the documented contract with a new regression test `system.console.search.MultiTokenNoMatch` (`Tests/EditorOps/TestSystemHandlers.cpp`, beside the existing `StatCommandsFound`) that exercises the real handler: asserts bare `"Stat "` returns rows (totalMatches >= 1) while multi-token `"stat unit"` returns `totalMatches == 0` with an empty `results` array — it fails if the dropped leading-token auto-retry is ever silently reintroduced. Files: `docs/wiki-src/system.md`, `Source/EditorAutomationRpcGateway/Private/Tests/EditorOps/TestSystemHandlers.cpp`. No handler/response-shape change. Stale line cites in #1 corrected (handler :133→:174, test :611→:652, wiki :121→:120). Not compiled/tested here — left for the verify phase.
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
