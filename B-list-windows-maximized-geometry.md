---
id: B-list-windows-maximized-geometry
title: "drive.list_windows misreports a maximized window: restored (pre-maximize) geometry and no maximized/state field, contradicting editor.resize_window's WINDOW_MAXIMIZED gate"
status: OPEN
severity: Medium
category: bug
tags: [window-maximized-state, drive, editor_chrome, list_windows, stale-readback, doc-mismatch]
encounters: 1
lastSeen: 2026-07-10T22:58:29.9710450+03:00
---

# drive.list_windows misreports a maximized window

## What's wrong
For a top-level editor window that is **maximized** (`SWindow::IsWindowMaximized() == true`),
`drive.list_windows` reports a `geometry.absolute` that is the window's **restored /
pre-maximize** rectangle (a non-maximized `1280x720 @ (1039,273)` on this host, not the
on-screen maximized/fullscreen bounds), and the per-window record carries **no
`maximized` / window-state field at all**. An agent therefore has no way to learn from
`list_windows` that a window is maximized, and the geometry it does return does not match
the window's actual on-screen state.

This directly contradicts a sibling verb: `editor.resize_window` on the very same window
refuses with `WINDOW_MAXIMIZED` ("... is maximized and will not visibly resize; restore it
first"). So one verb says "here is a plain 1280x720 window" while the other says "this
window is maximized." The window is genuinely maximized — the Slate title bar renders a
"Restore Down" affordance (only shown for a maximized window) and clicking it un-maximizes
the window, after which `resize_window` succeeds. `resize_window`'s gate is correct;
`list_windows`' readback is the one that misreports.

The harm lands on exactly the common task of normalizing a window to a target size and then
verifying it: the caller reads `list_windows` geometry as ground truth, sees `1280x720`,
and is misled about both the window's real size and the fact that it must be restored first.

## What it should do
`drive.list_windows` should let an agent detect the maximized condition that
`editor.resize_window` enforces. Concretely: add a per-window `maximized` boolean (query
`SWindow::IsWindowMaximized()`, the same signal `resize_window` uses), and/or report the
window's true current on-screen bounds when maximized rather than the stale restored
geometry. Either fix makes the two verbs agree.

## Verbatim repro (live, replay-confirmed at HEAD)
1. `mcp__pinwright__call` method=`drive.list_windows` args=`{}` -> main window index 0:
   `{"title":"EAContentExamples57 - Unreal Editor","type":"Normal","index":0,"geometry":{"absolute":{"x":1039,"y":273,"w":1280,"h":720}}}`
   (no `maximized`/state field anywhere in the record).
2. `mcp__pinwright__call` method=`editor.resize_window` args=`{"window_index":0,"width":1600,"aspect":"16:9"}` ->
   `[WINDOW_MAXIMIZED] Window 'EAContentExamples57 - Unreal Editor' is maximized and will not visibly resize; restore it first`

Same window, same instant: `list_windows` presents it as a non-maximized `1280x720 @ (1039,273)`
window while `resize_window` reports it as maximized.

## Guilty source (ground truth, read verbatim)
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveEditorChrome.cpp:387-389` — the only geometry `list_windows` exposes, with no maximized query:
  - `const FGeometry WindowGeometry = Window->GetWindowGeometryInScreen();`
  - `Info.AbsolutePosition = WindowGeometry.GetAbsolutePosition();`
  - `Info.AbsoluteSize = WindowGeometry.GetAbsoluteSize();`
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveEditorChrome.h:25-37` — `FDriveWindowInfo` fields are `Title`, `Type`, `AbsolutePosition`, `AbsoluteSize`, `Index`. There is no maximized / window-state member, so the handler at `DriveListWindowsHandler.cpp:27-44` cannot emit one.
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Editor/EditorWindowHandlers.cpp:701` — the authoritative, correct maximized signal that `list_windows` never surfaces:
  - `const bool bWasMaximized = Window->IsWindowMaximized();`

## Related (out of scope for this ticket)
Once an agent knows a window is maximized, there is still **no window-state control RPC**
(no restore / un-maximize / maximize / minimize verb) in the `editor` or `drive` namespaces,
so `resize_window`'s "restore it first" instruction is only actionable via fragile UI
automation (`drive.observe` the editor chrome + `drive.click` the title-bar "Restore Down"
button). That capability gap is distinct from this readback bug; noted here only as context.

severity rationale: impact=misleading/stale readback trusted on a normal path + omitted
state field (a sibling verb, resize_window, provides a cross-check that keeps it from a
fully-silent lie) x reach=list_windows is the every-session editor-chrome discovery verb but
the maximized mismatch is a specific state -> Medium. Could be High if the restored-geometry
readback (not just the missing flag) reproduces on non-fuzz hosts.

## History
- `#1-initial-repro` `OPEN` reporter — Filed: `drive.list_windows` reports a maximized main
  window (`IsWindowMaximized()==true`, confirmed by `resize_window` returning WINDOW_MAXIMIZED
  and by the presence/effect of the title-bar "Restore Down" button) as a non-maximized
  `1280x720 @ (1039,273)` rectangle with no maximized/state field, directly contradicting
  `editor.resize_window`'s WINDOW_MAXIMIZED gate. Seeded from an `editor.resize_window`
  window-normalization task (seed method `editor.resize_window`; culprit `drive.list_windows`).
  Live repro shows `list_windows` and `resize_window` disagreeing about the same window at the
  same instant.
