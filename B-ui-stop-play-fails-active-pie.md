---
id: B-ui-stop-play-fails-active-pie
title: "ui.stop_play returns STOP_FAILED on a live PIE session because it Exec's a non-console command string instead of RequestEndPlayMap()"
status: IN-REVIEW
severity: Medium
category: bug
tags: [ui, pie, stop-play, exec-string]
---

# `ui.stop_play` reports STOP_FAILED on a live PIE session

With PIE started via `editor.play` (same session, alive — live widgets and
`python.execute` interacting with the PIE world seconds earlier),
`ui.stop_play {}` returned `[STOP_FAILED] Failed to stop play in editor`.
Immediately after, `python.execute` running
`unreal.get_editor_subsystem(unreal.LevelEditorSubsystem).editor_request_end_play()`
ended PIE cleanly in the same second.

Root cause (source-confirmed): `ui.stop_play` gates on `GEditor->PlayWorld`
(correctly non-null here — that is why it did not return `NOT_PLAYING`), then
stops PIE via `GEditor->Exec(nullptr, TEXT("Stop Play In Editor"))` and trusts
its bool return (`UiHandler.cpp:201-213`). `"Stop Play In Editor"` is a UI
command label, **not** a console-exec verb — there is no `FParse::Command`
routing for it anywhere in `Engine/Source/Editor/UnrealEd/Private` (UE 5.7), so
`Exec` returns `false` and the handler emits `STOP_FAILED` even though the
session is live. The failure is not session-state-dependent: this path fails
whenever `PlayWorld` is non-null. By contrast `editor.stop` calls
`GEditor->RequestEndPlayMap()` directly (`PIEHandler.cpp:97`) — the exact API
that `LevelEditorSubsystem.editor_request_end_play()` invokes — which is why
both the preferred sibling and the python workaround succeed.

Note: the sibling `ui.play_in_editor` uses the same fragile pattern
(`GEditor->Exec(nullptr, TEXT("Play In Editor"))`, `UiHandler.cpp:183`); fix it
in the same pass if it is confirmed to fail the same way.

**Workaround:** call `editor.stop` (preferred sibling, already documented in the
handler summary), or `python.execute` with
`unreal.get_editor_subsystem(unreal.LevelEditorSubsystem).editor_request_end_play()`.
**Fix:** replace the `GEditor->Exec("Stop Play In Editor")` call in
`ui.stop_play` with a direct `GEditor->RequestEndPlayMap()`, mirroring
`editor.stop`, and return `{status: "stopped"}` unconditionally after the gate.

## History
- `#1-initial-repro` `OPEN` reporter — Reproduced live: with PIE started via `editor.play` (same session, ~20 min earlier) and demonstrably alive, `ui.stop_play {}` returned `[STOP_FAILED] Failed to stop play in editor`; `python.execute` with `LevelEditorSubsystem.editor_request_end_play()` ended PIE cleanly in the same second. Grounded in source: `ui.stop_play` (`UiHandler.cpp:201-213`) gates on `GEditor->PlayWorld` (non-null, so no `NOT_PLAYING`) then stops via `GEditor->Exec(nullptr, TEXT("Stop Play In Editor"))` and trusts the bool. `"Stop Play In Editor"` is not a console-exec verb — no `FParse::Command` routing for it in `Editor/UnrealEd/Private` (UE 5.7), so `Exec` returns false → `STOP_FAILED`, independent of PIE state. `editor.stop` (`PIEHandler.cpp:97`) and the python workaround both call `GEditor->RequestEndPlayMap()` directly and succeed. No duplicate on the board (checked B-inspect-misses-pie-world, B-editor-save-all-pie-diagnostic, E-session-wiki-pie-prerequisite-undocumented — none touch ui.stop_play/ui.play_in_editor).
- `#2-already-fixed` `IN-REVIEW` developer — Defect already absent from current source (pre-probe); flipped to IN-REVIEW for tester verification.
