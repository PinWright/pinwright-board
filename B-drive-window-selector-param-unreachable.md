---
id: B-drive-window-selector-param-unreachable
title: "drive editor-chrome window selector (window_index/window_title) rejected as UNKNOWN_PARAMS despite being documented + implemented"
status: IN-REVIEW
severity: Medium
category: bug
tags: [drive, editor_chrome, param-spec, unreachable-code, doc-mismatch]
encounters: 1
lastSeen: 2026-07-01T15:42:51.1506675+03:00
---

# drive editor-chrome window selector rejected as UNKNOWN_PARAMS

## What's wrong
Every editor-chrome `drive.*` verb (`observe`, `expect`, and the action verbs
hover/click/etc.) reads a window selector from `window_title`/`title` and
`window_index`/`index` in its handler body — but NONE of them declare those
params in their `RPC_PARAMS`, so the dispatcher's param allowlist rejects them
with `[UNKNOWN_PARAMS]` before the handler ever runs. The window-targeting code
is therefore dead/unreachable: an agent can only ever drive the **active
top-level window**, never a specific one by title or index.

This directly breaks the workflow both wiki pages tell agents to follow:
- `drive.list_windows` Notes: "A window's `index` selects it via the
  `window_index` selector and its `title` via `window_title` (substring) on the
  editor-chrome drive verbs. Use first whenever `surface=editor_chrome`."
- `drive.observe` Notes: "Params: ... `window_title`/`window_index` (editor
  chrome) ..."

The auto-generated `**Parameters**` list on the same `drive.observe` wiki page
(built from the live handler registry) already exposes the mismatch — it omits
`window_title`/`window_index` entirely, contradicting its own Notes overlay.

## What it should do
`drive.observe` / `drive.expect` / the drive action verbs should accept
`window_title` (+`title` alias) and `window_index` (+`index` alias) when
`surface=editor_chrome`, resolving to the window `list_windows` reported. Either
add these to each verb's `RPC_PARAMS` (the handler code already consumes them),
or stop documenting them.

## Verbatim repro
1. `mcp__pinwright__call` method=`drive.list_windows` args=`{}` -> returns 3
   targetable windows, each with an `index`:
   `{"windows":[{"title":"EAContentExamples57 - Unreal Editor","type":"Normal","index":0,...},{"title":"","type":"Notification","index":1,...},{"title":"","type":"Menu","index":2,...}],"count":3}`
2. `mcp__pinwright__call` method=`drive.observe`
   args=`{"surface":"editor_chrome","window_index":0}` ->
   `[UNKNOWN_PARAMS] Unknown parameter(s) for 'drive.observe': [window_index]. Valid parameters: [surface, instance_name, instanceName, root_index, rootIndex, screenshot, mark_cap, interactables_only, max_elements, include_telemetry, telemetry_since]. Call 'drive.observe' with no 'args' field to fetch its wiki page.`
3. `mcp__pinwright__call` method=`drive.observe`
   args=`{"surface":"editor_chrome","window_title":"Notification"}` ->
   `[UNKNOWN_PARAMS] Unknown parameter(s) for 'drive.observe': [window_title]. Valid parameters: [surface, instance_name, instanceName, root_index, rootIndex, screenshot, mark_cap, interactables_only, max_elements, include_telemetry, telemetry_since]. ...`

The "Valid parameters" set never contains any window selector, so the documented
`list_windows -> window_index/window_title` targeting is impossible on any drive
verb.

## Guilty source (ground truth, read verbatim)
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveObserveHandler.cpp:13-25`
  — the `RPC_PARAMS(...)` block declares only `surface, instance_name,
  instanceName, root_index, rootIndex, screenshot, mark_cap, interactables_only,
  max_elements, include_telemetry, telemetry_since`. It does NOT declare
  `window_title`/`window_index`/`title`/`index`.
- Yet the same handler consumes them —
  `DriveObserveHandler.cpp:40`:
  `const FDriveWindowSelector WindowSelector = FDriveHandlerCommon::ParseWindowSelector(Ctx);`
  and passes it into `BuildObservation(...)` at line 55.
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveHandlerCommon.cpp:133-134`:
  `Selector.Title = Ctx.GetStringFirstOf({ TEXT("title"), TEXT("window_title") });`
  `Selector.Index = Ctx.GetIntFirstOf({ TEXT("window_index"), TEXT("index") });`
- Same omission across the sibling drive verbs that also call
  `ParseWindowSelector`: `DriveExpectHandler.cpp:49` and
  `DriveActionCommon.cpp:145` (used by the action verbs), plus
  `DriveHandlerCommon.cpp:183` (`ResolveForSurface`). A grep for
  `RPC_PARAM.*window_title|RPC_PARAM.*window_index` across
  `Handlers/Drive/` returns zero matches — no drive verb declares them.
- The rejecting guard is the dispatcher's UNKNOWN_PARAMS allowlist (confirmed by
  the existing test comment at `Tests/EditorOps/TestSystemsHandlers.cpp:793`:
  "the dispatcher then rejects it with UNKNOWN_PARAMS").

severity rationale: impact=blocker-with-workaround (only the active top-level
window is reachable; documented positional/title targeting is impossible) ×
reach=editor_chrome drive verbs are every-session, but window-targeting is a
sub-path -> Medium

## History
- `#1-initial-repro` `OPEN` reporter — Filed: `drive.observe` (and every other
  editor-chrome drive verb) rejects the documented `window_index`/`window_title`
  selector with `[UNKNOWN_PARAMS]` because the params are missing from
  `RPC_PARAMS`, even though the handler body reads them via `ParseWindowSelector`
  and both the `drive.observe` and `drive.list_windows` wiki pages document them.
  Live repro shows `list_windows` returning 3 targetable windows while
  `observe {surface:editor_chrome, window_index:0}` and
  `observe {surface:editor_chrome, window_title:"Notification"}` both fail
  UNKNOWN_PARAMS. Editor-chrome window targeting is unreachable; only the active
  top-level window can be observed/driven.
- `#2-fix-declare-window-selector-params` `IN-REVIEW` developer — Root cause: the
  dispatcher's UNKNOWN_PARAMS allowlist (RpcDispatcher.cpp ValidateHandlerParams) is
  built only from each verb's declared RPC_PARAMS, but the window selector keys
  (`window_title`/`title`, `window_index`/`index`) that the handler bodies read via
  `FDriveHandlerCommon::ParseWindowSelector` were never declared. Fix declares the four
  keys as optional params (window_title/title = string, window_index/index = number) at
  all three declaration sites, matching the drive alias convention (separate RPC_PARAM_OPT
  lines): `DriveObserveHandler.cpp` (drive.observe), `DriveExpectHandler.cpp`
  (drive.expect), and the shared `DRIVE_COMMON_ACTION_PARAMS` macro in
  `DriveActionHandlers.cpp` (the six action verbs click/hover/scroll/type/key/drag). No
  handler-body change needed — ParseWindowSelector already reads all four names, so the
  documented list_windows -> window_index/window_title targeting is now reachable and the
  auto-generated Parameters list stops contradicting the wiki Notes. Regression test added
  at `Tests/Drive/TestDriveWindowSelectorParams.cpp`:
  `PinWright.drive.aliases.VerbsDeclareWindowSelectorParams` asserts all 8 editor-chrome
  drive verbs register the four selector params (fails if any of the three edits is
  reverted), and `PinWright.drive.aliases.ObserveAcceptsWindowSelectorOnWire` routes
  `{surface:editor_chrome, window_title/window_index}` through the real dispatcher and
  asserts it is no longer rejected UNKNOWN_PARAMS (the ticket's verbatim drive.observe
  repro).
