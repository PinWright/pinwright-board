---
id: E-spawn-category-not-implemented-no-in-mcp-recipe
title: "debug.spawn_category's NOT_IMPLEMENTED error names raw gdt.* engine verbs but no in-MCP recipe (editor.console_command 'gdt.EnableCategoryName <name>' against a live PIE world), and the debug.md overlay still advertises the dead GameplayDebuggerCategory wrap — caller burns one dead-end call per category and abandons the goal even with PIE running"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, debug, gameplay-debugger, spawn-category, error-actionability, not-implemented, pie, console-command]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
---

# `debug.spawn_category`'s fail-loud error and the `debug.md` overlay don't give an actionable in-MCP fallback

Once the fail-loud fix (`B-spawn-category-silent-noop-fake-existsafter`,
IN-REVIEW) lands, `debug.spawn_category` returns a well-formed
`[NOT_IMPLEMENTED]` instead of a fake success — a strict improvement. But the
error's remediation is phrased entirely in **raw engine terms** and stops short
of the one thing a caller of *this MCP* needs: the concrete in-MCP recipe. The
error says:

> *"The real commands are 'gdt.ToggleCategory <CategoryIdx>' and
> 'gdt.EnableCategoryName <name>', which require a live game world ... Drive the
> gameplay debugger from a running PIE session instead — this editor-scope RPC
> cannot confirm the toggle landed."*

"Drive the gameplay debugger from a running PIE session instead" is a pointer to
a workflow, not an action. It never names the in-MCP verb that actually issues
those commands (`editor.console_command`), so the caller is left to infer that
the fallback is `editor.console_command {command:"gdt.EnableCategoryName AI"}`
against the live PIE world (and, for `gdt.ToggleCategory`, that they must first
discover the numeric `<CategoryIdx>` — which the error doesn't help with either).

The `docs/wiki-src/debug.md` overlay compounds this. It still markets the
namespace as a working wrapper:

> *"Toggle UE Gameplay Debugger categories at runtime — tiny namespace wrapping
> the `GameplayDebuggerCategory <name>` console command ..."*

After the fail-loud fix, `GameplayDebuggerCategory <name>` is **not** a real
console command and the wrap never toggles anything — so the overlay advertises
a capability the method no longer (and never did) deliver. It does name
`editor.console_command` as "the escape hatch," but generically, without the
`gdt.EnableCategoryName <name>` recipe or the live-PIE-world precondition that
makes the escape hatch usable.

## Process evidence (this task)

A realism-mode AI-diagnostics task started PIE first, **then** asked to toggle
three categories. The agent had a **live PIE session up** — the exact condition
the error says is required — yet still:

- Called `debug.spawn_category` three times (`categoryName=AI`, `=EQS`,
  `=Behavior`), each returning the **identical** `[NOT_IMPLEMENTED]` — three
  predictably-futile calls, one dead-end per requested category.
- **Abandoned all three toggles** rather than pivoting to the suggested
  fallback. The friction note, verbatim: *"the MCP can only return a
  NOT_IMPLEMENTED pointing at gdt.* console verbs (which would have to be issued
  via editor.console_command against the PIE world), so the captured screenshot
  shows the stat HUDs but no debugger overlays."* The agent *understood* the
  fallback existed but did not execute it — the recipe was inferable but not
  on-band, and the goal (debugger overlays in the screenshot) went unmet.

So the cost here is a **process** one, separable from the underlying gap: even a
correct, fail-loud error left the caller without a directly actionable in-MCP
path, producing N dead-end calls (N = requested categories) and an abandoned
objective despite PIE being live.

## What it should do

1. **Make the `NOT_IMPLEMENTED` message name the in-MCP fallback explicitly.**
   Add the concrete recipe to `DebugHandler.cpp`'s error string: with PIE
   running, issue the toggle via `call("editor.console_command",
   {command:"gdt.EnableCategoryName <name>"})` against the PIE world (and note
   that `gdt.ToggleCategory` needs a numeric index, so `gdt.EnableCategoryName`
   is the name-friendly path). Turning "drive it from PIE somehow" into a
   copy-pasteable in-MCP call is what removes the dead-end-and-abandon pattern.
2. **Correct the `docs/wiki-src/debug.md` overlay** so it no longer claims the
   namespace wraps a working `GameplayDebuggerCategory <name>` command. State
   that `debug.spawn_category` cannot toggle from editor scope (it fails loud),
   and document the supported path: start PIE, then
   `editor.console_command "gdt.EnableCategoryName <name>"`. Pair this with the
   category-name list owned by `E-spawn-category-name-discovery` so a caller
   gets both the *names* and the *verb* in one place.

## Distinct from existing spawn_category tickets

- `B-spawn-category-silent-noop-fake-existsafter` (IN-REVIEW) owns the
  *result-reporting* defect — stop fabricating success, fail loud. Its deferred
  "fuller future enhancement" (route to the world-bound command when a PIE world
  exists) is the *feature* path; this ticket is the cheaper *error-actionability
  + overlay-accuracy* path that helps callers today, before any such routing
  exists.
- `E-spawn-category-name-discovery` (OPEN) owns *which category names* are valid
  (the param value, the wrong `'Behavior'`→`'BehaviorTree'` example). This
  ticket owns *which verb/recipe* to fall back to and the stale overlay claim —
  the method/escape-hatch angle, not the name angle. Both touch
  `docs/wiki-src/debug.md` but for non-overlapping content (names vs. recipe).

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the gameplay-debugger overlay struggle audit (process/friction lens; outcome `clean`). The fail-loud `[NOT_IMPLEMENTED]` from `debug.spawn_category` (post-`B-spawn-category-silent-noop-fake-existsafter` fix) names raw `gdt.ToggleCategory`/`gdt.EnableCategoryName` engine verbs and "drive from a running PIE session" but never the in-MCP recipe (`editor.console_command "gdt.EnableCategoryName <name>"` against a live PIE world); the `docs/wiki-src/debug.md` overlay still advertises the dead `GameplayDebuggerCategory <name>` wrap. Process evidence: task had PIE live, called `spawn_category` 3× (AI/EQS/Behavior) → 3 identical NOT_IMPLEMENTED dead-ends → abandoned all toggles, leaving debugger overlays absent from the requested screenshot; friction note confirms the agent knew the gdt.* fallback existed but it was inferable-not-actionable. Proposed (E-/docs): add the copy-pasteable `editor.console_command "gdt.EnableCategoryName <name>"` recipe to the handler's NOT_IMPLEMENTED string, and correct `docs/wiki-src/debug.md` to drop the working-wrap claim and document the PIE+gdt path. Distinct from the result-shape bug ticket (IN-REVIEW) and the name-discovery ticket (OPEN) — both spawn_category, different seams.
