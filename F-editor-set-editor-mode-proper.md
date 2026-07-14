---
id: F-editor-set-editor-mode-proper
title: "Reimplement editor.set_editor_mode via GLevelEditorModeTools().ActivateMode / ULevelEditorSubsystem (removed as a MODE-exec stub)"
status: OPEN
severity: Medium
category: feature
tags: [editor, editor-mode, level-editor-mode-tools, reimplement, rpc-cull]
---

# Reimplement `editor.set_editor_mode` properly (activate an editor mode)

`editor.set_editor_mode` was **removed** in the RPC cull recorded in
[`E-rpc-cull-151-record`](E-rpc-cull-151-record.md) because it wrapped the `MODE`
exec, which cannot activate an inactive mode. Switching the level editor into
Placement / Landscape / Foliage / MeshPaint (etc.) is a real, wanted capability
with no RPC alternative.

## What the removed version did wrong

The old handler (`Handlers/Editor/EditorCommandHandler.cpp:563-568`) ran
`GEditor->Exec(..., "mode <name>")` and returned `success:true` echoing the mode
name. But the engine's `MODE` exec only **re-broadcasts a mode that is already
active** — it can never activate an inactive one. In
`UnrealEdSrv.cpp:3115-3126`,
`GLevelEditorModeTools().GetActiveMode(FName(name))` returns non-null only for
currently-active modes, so an inactive target is never broadcast; and the
listener (`FUnrealEdMisc::OnEditorChangeMode`) calls `ActivateMode(mode,
bToggle=true)`, which would toggle an already-active mode **off**. The only
unconditional effects of `Exec_Mode` are camera-roll reset + viewport redraw —
unrelated to the request. So the advertised switch never happened.

## Proper implementation

Activate the mode through the real API, not the exec:

- `GLevelEditorModeTools().ActivateMode(FEditorModeID)` (resolve the mode ID from
  the caller's name — map friendly names like `placement`/`landscape`/`foliage`/
  `mesh_paint` to the engine `FBuiltinEditorModes`/`FEditorModeID` constants, or
  accept the raw ID); or
- `ULevelEditorSubsystem` / the editor-mode manager API where it exposes a
  cleaner activate.

Verify with `GLevelEditorModeTools().IsModeActive(ModeID)` after activation and
report that real state (not an echo). Fail loud with a clear error if the mode
name does not resolve to a registered `FEditorModeID`.

**Fix:** New handler in `EditorCommandHandler.cpp` (16 other handlers remain; no
shared helper was exclusive to the removed one). Regression test activates a
known mode and asserts `IsModeActive` is true afterward (the removed exec-based
version could never make an inactive mode active — differential proof), and
restores the default mode in teardown.

## History
- `#1-reimpl-after-cull` `OPEN` reporter — Filed to reinstate the wanted capability removed by the RPC cull ([`E-rpc-cull-151-record`](E-rpc-cull-151-record.md)). The removed `editor.set_editor_mode` wrapped the `mode <name>` exec (EditorCommandHandler.cpp:563-568), which per UnrealEdSrv.cpp:3115-3126 can only re-broadcast an already-active mode and never activates an inactive one, yet returned success. Proper impl: `GLevelEditorModeTools().ActivateMode` / `ULevelEditorSubsystem`, with `IsModeActive` verification and a loud error on an unresolved mode name. No RPC alternative exists.
