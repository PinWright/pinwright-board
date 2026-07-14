---
id: F-editor-set-immersive-mode-proper
title: "Reimplement editor.set_immersive_mode via the real LevelEditor immersive toggle (removed as a bare-exec stub)"
status: OPEN
severity: Low
category: feature
tags: [editor, viewport, immersive, level-editor, reimplement, rpc-cull]
---

# Reimplement `editor.set_immersive_mode` properly (viewport immersive toggle)

`editor.set_immersive_mode` was **removed** in the RPC cull recorded in
[`E-rpc-cull-151-record`](E-rpc-cull-151-record.md) because it execed a bare
string that matches no exec route. Toggling the active level viewport into
immersive (F11 full-viewport) mode is a legitimate, if low-frequency, screenshot/
presentation convenience. Severity Low: pure viewport convenience, rare path,
no data at stake.

## What the removed version did wrong

The old handler (`Handlers/Editor/ViewportHandler.cpp:388-396`) ran
`GEditor->Exec(..., "ToggleImmersive")`, ignored the return, and echoed the
caller's `enabled` param back as `immersiveModeEnabled` with `success:true`. The
param spec itself admitted the lie: `"Reported state to echo back; the
underlying command toggles unconditionally"`. There is **no exec route for bare
`ToggleImmersive`** in UE 5.3–5.8 — the console command is the full-name
`LevelEditor.ToggleImmersive` (`LevelEditor.cpp:90`), and it otherwise exists
only as the F11 `FUICommandInfo` (`LevelViewportActions.cpp:45`). Console-object
lookup is exact-name, so the bare exec was a guaranteed no-op.

## Proper implementation

Drive the real toggle instead of a bogus exec string. Either:

- Exec the correctly-named `LevelEditor.ToggleImmersive` console command; or
- Resolve the active level viewport client and call its immersive API directly
  (`SLevelViewport::MakeImmersive` / the `FLevelViewportCommands::ToggleImmersive`
  UI command through the LevelEditor module) so the `enabled` param actually sets
  the state rather than blind-toggling.

Prefer the direct API path so the response can report the **actual** immersive
state read back from the viewport, not an echo. If only a blind toggle is
feasible, rename the param honestly (it is a toggle, not a set).

**Fix:** New handler in `ViewportHandler.cpp` (6 other handlers remain; shared
`ForceRedrawActiveViewport`/`CaptureActiveLevelViewportToScreenshot` helpers stay).
Regression coverage is limited (immersive mode needs a real level viewport), so a
smoke test that the handler resolves a viewport client and does not crash, plus a
readback assertion where a viewport is available.

## History
- `#1-reimpl-after-cull` `OPEN` reporter — Filed to reinstate the wanted capability removed by the RPC cull ([`E-rpc-cull-151-record`](E-rpc-cull-151-record.md)). The removed `editor.set_immersive_mode` execed a bare `ToggleImmersive` string with no exec route (ViewportHandler.cpp:388-396) and echoed the `enabled` param back as success. Proper impl: execute the real `LevelEditor.ToggleImmersive` command, or better, drive the LevelEditor viewport immersive API directly and read back actual state. Severity Low (viewport convenience, rare path).
