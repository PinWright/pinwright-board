---
id: B-compile-error-bp-wedges-next-play
title: "blueprint.compile can leave a Blueprint in BS_Error from a transient 'Leak Detected' check, and the next editor.play then wedges the editor on the 'Blueprint Asset Compilation Error' modal (process kill needed)"
status: OPEN
severity: High
category: bug
tags: [blueprint, blueprint.compile, editor.play, pie, modal, compile-error, leak-detected, unattended, editor-restart]
encounters: 1
costly: 1
lastSeen: 2026-09-23T20:55:00Z
---

# A compile-only check leaves the editor one call away from a modal it cannot dismiss

UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`. Editor started visible from the OS association (no `-RunningUnattendedScript`),
no PIE running, `L_Core` open.

1. `blueprint.compile {path: /App/App/UI/LobbyAndMenu/W_DroneSelect_EditDrone}` (asset unmodified, used as a
   read-only compile check) returned `compiled: false, status: Error` with a single error:
   `Leak Detected!  PhysicsWebBrowser (AppWebBrowserWidget) still has living Slate widgets ...`. That is
   UMG's compile-time leak check firing because a Slate widget from an earlier preview/PIE was still
   alive; it is environmental, not a graph error. A second compile gave the same result. After an editor
   restart the same Blueprint loads and runs without that error.
2. `editor.play {}` then failed with a transport timeout (`stream read failed: timed out`), not a coded error.
3. The next call (`editor.pie_status`) returned `EDITOR_BLOCKED_ON_MODAL` for the modal
   "Blueprint Asset Compilation Error", 122 s old. No RPC can dismiss it; the editor process had to be
   killed and restarted.

What is missing:
- `blueprint.compile` does not say that an `Error` status blocks PIE behind an interactive modal, and does not
  separate environmental errors (leak detection) from graph errors.
- `editor.play` does not check for Blueprints in `BS_Error` before requesting PIE. It could refuse with
  a coded error that lists them, or set the play-session flag that skips the "play anyway?" prompt.
- `editor.play` reports a timeout instead of `EDITOR_BLOCKED_ON_MODAL`, although the modal was already up.

**Workaround:** do not compile Blueprints that host `UAppWebBrowserWidget` (or other CEF widgets) in a
visible editor before PIE. Or start the editor with `unattended_script: true` so modals auto-answer.

## History
- `#1-compile-then-play-modal-wedge` `OPEN` reporter - UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`. Costly: editor killed and restarted (about 2.5 min), and the restart itself then triggered `B-editor-start-silent-module-rebuild`.
