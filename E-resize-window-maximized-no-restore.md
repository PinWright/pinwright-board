---
id: E-resize-window-maximized-no-restore
title: "editor.resize_window rejects a maximized window and says 'restore it first', but no restore/un-maximize RPC exists — stranding the caller in a heavyweight UI-automation workaround"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [window-maximized-no-restore, resize-window, window-state-control, no-recovery-path]
encounters: 1
lastSeen: 2026-07-10T23:02:14.0957993+03:00
claimedBy: fuzz2
claimedAt: 2026-07-11T00:36:42.5397482+03:00
---

# editor.resize_window dead-ends on a maximized window with no restore verb to recover

## What's awkward
`editor.resize_window` refuses a maximized top-level window with
`[WINDOW_MAXIMIZED] Window '...' is maximized and will not visibly resize; restore it first`
and its wiki repeats the instruction "restore it first" — yet **no un-maximize / restore /
maximize / minimize / set-window-state verb exists in any namespace** (`editor` or `drive`).
The gate is correct (it uses `SWindow::IsWindowMaximized()` and already computes+returns
`wasMaximized`), but the remedy it names is not callable, so the caller is dead-ended.

The UE editor commonly **launches maximized**, so normalizing the MAIN window — the headline
use of `editor.resize_window` and the natural first step of any "frame my captures at a clean
size" goal — is blocked out of the box, and the tool's own suggested recovery is a no-op verb.

## What it should do
Make the maximized state recoverable through the API, one of:
- **Auto-restore before applying the target size** inside `editor.resize_window` (optionally
  gated by an `allowRestore` / `force` flag) — the handler already knows `wasMaximized`, so it
  can un-maximize then resize in one call and report that it did so; or
- expose a dedicated **`editor.restore_window` / `editor.set_window_state {maximized:false}`**
  (and its inverse) so the "restore it first" instruction names a real, callable method.

Either removes the dead-end. Until then the wiki should at minimum name the actual recovery
path so it is discoverable.

## Evidence (this task — window-normalization goal, outcome=done but with heavyweight friction)
Attempt agent's plan assumed the first `editor.resize_window {width:1600, aspect:'16:9'}` would
just work; it returned `WINDOW_MAXIMIZED` instead. With no restore RPC to call, the agent had
to **reverse-engineer a recovery**, per the CallAnalyzer ground-truth trace (12 calls):
- read the plugin C++ `EditorWindowHandlers.cpp` as a last resort to confirm no restore RPC exists;
- searched the `editor` and `drive` namespace indexes for a restore method (none);
- `drive.observe {surface:editor_chrome, window_index:0, interactables_only, max_elements:0}`
  dumped a ~1,035,819-char payload to disk;
- two greps to locate a "Restore Down" `SButton`;
- `drive.click` on the title-bar `SButton` handle to un-maximize;
- only THEN did the retry `editor.resize_window` succeed (clientSize 1600x900, wasMaximized=false).

So one error + a ~1 MB, multi-call UI-automation workaround + a C++ source read were spent to
recover from a state `resize_window` itself already detects and reports.

**Distinct from the readback bug `B-list-windows-maximized-geometry`:** that ticket is about
`drive.list_windows` misreporting the maximized window's geometry/state (a data bug on a
different method). It explicitly carves this capability gap out as "Related (out of scope for
this ticket) ... no window-state control RPC ... That capability gap is distinct from this
readback bug." This ticket is that carved-out gap: the missing recovery verb on
`editor.resize_window`. Different method, different root cause (no control verb vs stale
readback), different category (ergonomic/capability vs bug).

**Workaround:** un-maximize the window first via `drive.observe`(editor_chrome) to find the
title-bar "Restore Down" `SButton`, then `drive.click` it, then retry `editor.resize_window`.

**Fix:** auto-restore inside `editor.resize_window` (gated by an `allowRestore`/`force` flag),
or add `editor.restore_window` / `editor.set_window_state`. Wiki overlay to update:
`docs/wiki-src/editor.md` (`editor.resize_window`).

severity rationale: impact=soft blocker, doable but only via a heavyweight source-dive +
~1 MB UI-automation workaround (Medium) x reach=resize_window on the main window is a common
first step and the editor commonly launches maximized (leans every-session), balanced by the
existing drive.click workaround -> Medium.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from a struggle audit of an `editor.resize_window`
  window-normalization task (outcome=done). `editor.resize_window` correctly gates a maximized
  window with WINDOW_MAXIMIZED and says "restore it first", but no restore/un-maximize RPC
  exists, so the agent burned one error + a ~1 MB `drive.observe`/`drive.click` UI-automation
  workaround + a plugin C++ source read to recover. Propose auto-restore in resize_window (or an
  `editor.restore_window` / `editor.set_window_state` verb). Distinct from the readback bug
  `B-list-windows-maximized-geometry`, which carves this gap out as out-of-scope context.
- `#2-go` `IN-REVIEW` developer — GO. Defect confirmed in current source: `editor.resize_window`
  (`EditorWindowHandlers.cpp:701-707`) hard-errors WINDOW_MAXIMIZED "restore it first" while no
  restore/maximize/minimize/window-state verb exists in any namespace (only unrelated
  `actor.restore_snapshot`). Not a duplicate of `B-list-windows-maximized-geometry` (a distinct
  readback bug). Implementing the dedicated-verb remedy — a new `editor.set_window_state`
  {state:'restored'|'maximized'|'minimized'} verb — favored over the resize_window mode-flag by
  the split-verb convention; plus renaming the WINDOW_MAXIMIZED message + wiki to name it so
  "restore it first" is a real callable/discoverable path. This is one of the ticket's two offered
  "one of" alternatives (dedicated verb vs auto-restore-in-resize); the ticket goal (maximized
  state recoverable through the API) is fully met, no scope dropped, no follow-on ticket needed.
