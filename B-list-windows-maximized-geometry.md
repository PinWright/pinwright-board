---
id: B-list-windows-maximized-geometry
title: "drive.list_windows omits per-window maximize/minimize state, contradicting editor.resize_window's WINDOW_MAXIMIZED gate and editor.set_window_state's readback"
status: IN-REVIEW
severity: Medium
category: bug
tags: [window-maximized-state, window-minimized-state, drive, editor_chrome, list_windows, doc-mismatch]
encounters: 1
lastSeen: 2026-07-10T22:58:29.9710450+03:00
---

# drive.list_windows omits per-window window state

## What's wrong
For a top-level editor window that is **maximized** (`SWindow::IsWindowMaximized() == true`),
`drive.list_windows` carries **no `maximized` / window-state field at all** in the per-window
record. An agent therefore has no way to learn from `list_windows` — the every-session
editor-chrome discovery verb it hits first — that a window is maximized (or minimized).

This directly contradicts a sibling verb: `editor.resize_window` on the very same window
refuses with `WINDOW_MAXIMIZED` ("... is maximized and will not visibly resize; restore it
first via `editor.set_window_state {state:'restored'}`"). So one verb reports a plain window
while the other reports it as maximized. `resize_window`'s gate is correct; `list_windows`'
readback is the one that under-reports — it omits the state entirely.

The authoritative signal exists and is already surfaced by two sibling `editor.*` verbs:
`editor.resize_window` reads `Window->IsWindowMaximized()` (`EditorWindowHandlers.cpp:701`),
and `editor.set_window_state` reports both `isMaximized` and `isMinimized`
(`EditorWindowHandlers.cpp:847-857`) from `IsWindowMaximized()`/`IsWindowMinimized()`.
`drive.list_windows` simply never queries them.

The harm lands on the common task of normalizing/targeting a window: the caller reads
`list_windows`, sees a window with no state, and is not told it must be restored first (via
`editor.set_window_state {state:'restored'}`) before `resize_window` will act.

## What it should do (Fix)
Add a per-window **`maximized`** boolean and a **`minimized`** boolean to `drive.list_windows`,
sourced from `SWindow::IsWindowMaximized()` / `SWindow::IsWindowMinimized()` — the same signals
`editor.resize_window` and `editor.set_window_state` use — so the discovery verb agrees with
the control verbs. This mirrors `editor.set_window_state`'s `isMaximized`/`isMinimized`
readback shape (`EditorWindowHandlers.cpp:847-857`).

Concretely: add `bMaximized`/`bMinimized` to `FDriveWindowInfo` (`DriveEditorChrome.h`),
populate them in `FDriveEditorChrome::ListWindows()` (`DriveEditorChrome.cpp`), emit
`maximized`/`minimized` in `DriveListWindowsHandler.cpp`, and document the two fields in the
`drive` wiki overlay + the handler summary string.

## Out of scope — the geometry claim (dropped, unverified)
The original report also alleged `list_windows` returns the **restored / pre-maximize**
geometry (`1280x720 @ (1039,273)`) for a maximized window rather than its on-screen maximized
bounds, and proposed rewriting the geometry readback. That half is **dropped**: it is
unverified and reporter-hedged ("Could be High if the restored-geometry readback reproduces on
non-fuzz hosts"). `GetWindowGeometryInScreen()` returns the window's *current* geometry, and a
genuinely OS-maximized `SWindow` reshapes to the work area — so the fuzz-host `1280x720`
observation most likely reflects a headless / `-RenderOffScreen` borderless quirk (the state
flips maximized without a real window manager reshaping the window), not a systematic
stale-geometry bug. `GetWindowGeometryInScreen()` is a load-bearing readback; it is left
untouched unless the staleness is independently reproduced on a real host — that would be a
separate ticket.

## Verbatim repro (live)
1. `drive.list_windows {}` -> main window index 0:
   `{"title":"EAContentExamples57 - Unreal Editor","type":"Normal","index":0,"geometry":{"absolute":{...}}}`
   — no `maximized` / `minimized` / state field anywhere in the record.
2. `editor.resize_window {"window_index":0,"width":1600,"aspect":"16:9"}` ->
   `[WINDOW_MAXIMIZED] Window 'EAContentExamples57 - Unreal Editor' is maximized and will not
   visibly resize; restore it first via editor.set_window_state {state:'restored'}, then retry`

Same window, same instant: `list_windows` presents it with no state while `resize_window`
reports it as maximized.

## Guilty source (ground truth, read verbatim)
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveEditorChrome.h:25-37` —
  `FDriveWindowInfo` = `Title`, `Type`, `AbsolutePosition`, `AbsoluteSize`, `Index`. No
  window-state member, so `DriveListWindowsHandler.cpp:29-42` (which emits only
  title/type/index/geometry) cannot surface one.
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveEditorChrome.cpp:379-393` —
  the per-window loop populates geometry via `GetWindowGeometryInScreen()` and never calls
  `IsWindowMaximized()` / `IsWindowMinimized()`.
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Editor/EditorWindowHandlers.cpp:701`
  (`editor.resize_window`) and `:835-857` (`editor.set_window_state`) — the authoritative
  maximized/minimized signals `list_windows` never surfaces.

severity rationale: impact=omitted state field on a normal discovery path (a sibling verb,
`resize_window`, provides a cross-check that keeps it from a fully-silent lie) x reach=
`list_windows` is the every-session editor-chrome discovery verb but the mismatch is a specific
window state -> Medium.

## History
- `#1-initial-repro` `OPEN` reporter — Filed: `drive.list_windows` reports a maximized main
  window (`IsWindowMaximized()==true`, confirmed by `resize_window` returning WINDOW_MAXIMIZED
  and by the presence/effect of the title-bar "Restore Down" button) as a non-maximized
  `1280x720 @ (1039,273)` rectangle with no maximized/state field, directly contradicting
  `editor.resize_window`'s WINDOW_MAXIMIZED gate. Seeded from an `editor.resize_window`
  window-normalization task (seed method `editor.resize_window`; culprit `drive.list_windows`).
  Live repro shows `list_windows` and `resize_window` disagreeing about the same window at the
  same instant.
- `#2-reword-scope` `IN-REVIEW` developer — Reworded (over-scoped title + false "no
  window-state RPC" note) and fixed. Confirmed defect from source: `drive.list_windows` omitted
  per-window window state, so a maximized window read as plain and contradicted
  `editor.resize_window`'s WINDOW_MAXIMIZED gate. Fix: added `bMaximized`/`bMinimized` to
  `FDriveWindowInfo` (`DriveEditorChrome.h`), populated from `SWindow::IsWindowMaximized()`/
  `IsWindowMinimized()` in `FDriveEditorChrome::ListWindows()` (`DriveEditorChrome.cpp`), emitted
  `maximized`/`minimized` per record in `DriveListWindowsHandler.cpp` (+ summary/doc comment), and
  documented both fields in the `drive` wiki overlay (`docs/wiki-src/drive.md`) — mirroring
  `editor.set_window_state`'s isMaximized/isMinimized readback (`EditorWindowHandlers.cpp:847-857`).
  DROPPED the ticket's speculative geometry-rewrite half (unverified/reporter-hedged; the
  fuzz-host `1280x720` is a likely headless/borderless artifact; `GetWindowGeometryInScreen()`
  left untouched). Regression: adopted + strengthened the red test
  `PinWright.drive.editorint.ListWindowsReportsMaximizedState`
  (`Tests/Drive/TestDriveListWindowsMaximizedState.cpp`) — asserts both fields' presence +
  value-equality against live `IsWindowMaximized()`/`IsWindowMinimized()`; observed RED pre-fix,
  now GREEN against a genuinely-maximized fixture (`Result={Success}`, "Maximize() took effect").
  Plugin built clean (`Result: Succeeded`).
- `#3-test-phase-fix` `IN-REVIEW` developer — Resolved the escalated adversarial-review
  correctness concern by DROPPING the always-false `minimized` field (kept `maximized`).
  Confirmed against engine source that `FSlateApplication::GetAllVisibleWindowsOrdered`
  (`SlateApplication.cpp:3742/3751`) filters every top-level and child window through
  `IsVisible() && !IsWindowMinimized()`, and that both `ListWindows()` and the shared
  `window_index` selector (`ResolveSelectedWindow`, `DriveEditorChrome.cpp:222`) enumerate that
  same minimized-excluded set — so `bMinimized` could only ever read `false`, a minimized window
  never appears in `list_windows`, and broadening the enumeration would shift the `window_index`
  targeting order for every editor-chrome drive verb (resize_window/set_window_state/screenshot)
  — a design change that belongs in its own ticket. Removed `bMinimized` from `FDriveWindowInfo`
  (`DriveEditorChrome.h`), the `IsWindowMinimized()` populate in `ListWindows()`, the `minimized`
  emit + summary/doc in `DriveListWindowsHandler.cpp`, the `minimized` field from the `drive` wiki
  overlay, and the now-tautological `minimized` presence + value-equality assertions in
  `TestDriveListWindowsMaximizedState.cpp`. `maximized` — which fixes the ticket's actual reported
  bug (the `resize_window` WINDOW_MAXIMIZED contradiction) — ships intact with its presence +
  value-equality + genuinely-maximized-fixture assertions. Full suite GREEN (3634 `Result={Success}`,
  0 `Result={Fail}`, `TEST COMPLETE. EXIT CODE: 0`); `PinWright.drive.editorint.ListWindowsReportsMaximizedState`
  `Result={Success}` with "Maximize() took effect".
