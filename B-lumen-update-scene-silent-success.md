---
id: B-lumen-update-scene-silent-success
title: "render.lumen_update_scene runs an unrecognized console command but always returns executed: true"
status: IN-REVIEW
severity: High
category: bug
tags: [render, lumen, console-command, silent-failure, exec]
---

# render.lumen_update_scene reports executed:true for a console command the engine rejects

`render.lumen_update_scene` is documented as "Trigger a Lumen scene
recapture". The handler (`Handlers/Render/RenderHandler.cpp` lines
492–512) does:

```cpp
GEngine->Exec(World, TEXT("r.Lumen.Scene.Recapture"));
TSharedPtr<FJsonObject> Result = MakeShared<FJsonObject>();
Result->SetStringField(TEXT("action"), TEXT("lumen_update_scene"));
Result->SetStringField(TEXT("command"), TEXT("r.Lumen.Scene.Recapture"));
Result->SetBoolField(TEXT("executed"), true);
Ctx.SendSuccess(Result);
```

Two compounding problems:

1. **The console command does not exist.** `r.Lumen.Scene.Recapture` is
   not a registered console object in UE 5.7. The real Lumen cvars are
   under the `r.LumenScene.*` prefix (e.g.
   `r.LumenScene.SurfaceCache.Reset`,
   `r.LumenScene.SurfaceCache.RecaptureEveryFrame` —
   `Engine/Source/Runtime/Renderer/Private/Lumen/LumenSceneRendering.cpp`).
   There is no `r.Lumen.Scene.Recapture` (note the `Lumen.Scene.*`
   dotting) anywhere in the engine. `GEngine->Exec` therefore finds no
   handler, logs "Command not recognized", and returns `false`.

2. **The bool return of `Exec` is discarded.** The handler ignores the
   `false` and unconditionally sets `executed: true` / `SendSuccess`. So
   the RPC reports a successful Lumen recapture when, in fact, no command
   ran and Lumen's surface cache was never invalidated.

This is the silent-success-with-no-effect defect class already accepted
and fixed for `material.authoring.*`
(`B-material-stub-handlers-silent-success`, DONE) and `input.set_input_*`
(`B-input-trigger-modifier-stub-silent-success`, IN-REVIEW) — but for a
different method (`render.lumen_update_scene`), so it is filed separately
per the board's one-ticket-per-method convention.

The contrast with the sibling `system.console_command` is decisive: that
handler captures the same bool (`bOk = GEditor->Exec(TargetWorld, *Cmd)`,
`SystemControlHandler.cpp:561`) and returns `success: bOk`, sending
`[EXEC_FAILED] Command not executed` when the engine rejects the command.
Running the *exact same command string* through it
(`system.console_command {command:"r.Lumen.Scene.Recapture"}`) returns
`[EXEC_FAILED]` — proving the command is unrecognized and that
`render.lumen_update_scene`'s `executed:true` is a false success.

**Why it matters:** A realistic task — retune a key light, recapture
Lumen, screenshot the result for before/after comparison — calls this RPC,
gets `executed: true`, and concludes Lumen's GI re-evaluated the brighter
light. It did not. The "after" screenshot reflects whatever the running
viewport already had (Lumen updates incrementally on its own schedule),
and the agent reports a verified recapture that never happened. The
JSON-RPC layer gives no signal that the intended operation was a no-op.

**Workaround:** none that is honest — there is no correct one-shot
recapture cvar exposed; `r.LumenScene.SurfaceCache.Reset 1` (then back to
0) is the closest real lever, via `system.console_command`, which at least
reports `success` truthfully.

**Fix options (any of):**
1. **Use a real command and honor the return.** Replace the bogus
   `r.Lumen.Scene.Recapture` with an actual Lumen invalidation lever
   (e.g. toggle `r.LumenScene.SurfaceCache.Reset`), capture the `Exec`
   bool, and report `executed: bOk` / `SendError("EXEC_FAILED", …)` when
   false — matching `system.console_command`.
2. **Fail loud.** At minimum, stop discarding the `Exec` return: set
   `executed` to the actual bool and `SendError("EXEC_FAILED", …)` when
   the command is not handled, so callers can detect the no-op. (This
   alone would surface the invalid command name immediately.)

## Repro

1. `render.lumen_update_scene {}` →
   `{"action":"lumen_update_scene","command":"r.Lumen.Scene.Recapture","executed":true}`
   (claims success).
2. `system.console_command {"command":"r.Lumen.Scene.Recapture"}` →
   `[EXEC_FAILED] Command not executed` — the identical command string,
   run through a handler that checks the `Exec` return, is rejected by the
   engine as unrecognized.
3. Conclusion: step 1's `executed: true` is a false success; the Lumen
   recapture never ran.

## History
- `#1-initial-repro` `OPEN` reporter — Realism/seed-mode lighting task replayed live against `mcp__editor-automation__call`. `render.lumen_update_scene {}` returned `{"command":"r.Lumen.Scene.Recapture","executed":true}`. The same command via `system.console_command {command:"r.Lumen.Scene.Recapture"}` returned `[EXEC_FAILED] Command not executed`, confirming the command is unrecognized in UE 5.7 (real Lumen cvars are `r.LumenScene.*`, e.g. `r.LumenScene.SurfaceCache.Reset`/`RecaptureEveryFrame` per `LumenSceneRendering.cpp`; `r.Lumen.Scene.Recapture` does not exist). The handler (`RenderHandler.cpp:501-505`) discards the `GEngine->Exec` bool and hardcodes `executed:true` → silent success-with-no-effect. Sibling `system.console_command` (`SystemControlHandler.cpp:561,597-604`) shows the correct pattern (returns `success: bOk`, `EXEC_FAILED` on false). No existing render/lumen ticket covers this (ripgrep over board clean; `F-rendering-project-settings` only mentions the method in passing as a runtime recapture, not this bug). Same defect class as DONE `B-material-stub-handlers-silent-success` / IN-REVIEW `B-input-trigger-modifier-stub-silent-success`, different method, filed separately.
- `#2-fix` `IN-REVIEW` developer — Applied Fix option 1 (real command + honor the bool). `RenderHandler.cpp` (`render.lumen_update_scene`) now runs the genuine one-shot Lumen recapture lever `r.LumenScene.SurfaceCache.Reset 1` (verified registered at `Engine/Source/Runtime/Renderer/Private/Lumen/LumenSceneRendering.cpp:84-90`; the renderer resets all atlases/captured cards next frame and auto-clears the cvar back to 0 at `:2087-2092`, so it is a true one-shot trigger, not a sticky toggle), captures the `GEngine->Exec` bool, echoes the actual command, sets `executed: bOk`, and `SendError("EXEC_FAILED", "Command not executed")` when the engine rejects it — matching the `system.console_command` convention. Regression test `FRenderLumenUpdateSceneRunsRealCommandTest` (`render.lumen_update_scene.RunsRealCommand`) added in `Tests/EditorOps/TestRenderHandlers.cpp`: invokes the production handler via the registry, asserts the echoed `command` is not the bogus `r.Lumen.Scene.Recapture`, and cross-checks that the echoed command run through `system.console_command` returns success (no `EXEC_FAILED`) — both assertions fail if the handler is reverted to the unrecognized command. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Render/RenderHandler.cpp`, `Source/EditorAutomationRpcGateway/Private/Tests/EditorOps/TestRenderHandlers.cpp`. Not compiled/tested here (later phase).
