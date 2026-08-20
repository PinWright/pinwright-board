---
id: B-editor-start-blocks-on-zenserver-modal
title: "editor_start hangs to its readiness timeout on the boot-time 'Wait for ZenServer?' modal — a native MessageBoxExt no in-editor guard can reach"
status: IN-REVIEW
severity: High
category: bug
tags: [proxy, mcp-proxy, editor-launch, modal-dialog, game-thread-hang, zenserver, unattended, editor_start]
encounters: 2
lastSeen: 2026-08-20T00:00:00Z
---

# `editor_start` wedges on the boot-time "Wait for ZenServer?" modal

## What's wrong

An `editor_start` launch hung until its 180 s readiness timeout with the editor
sitting on a modal titled "Wait for ZenServer?". Passing `unattended_script: true`
cleared it.

This is the boarded modal-deadlock class again, in a window commit `7ff9dcf3`
("Stop modal dialogs from deadlocking the game thread during RPCs") structurally
cannot cover. Two independent reasons:

1. **It fires before PinWright exists.** The prompt is raised while the Zen storage
   server is being waited on during engine startup, long before
   `UPinWrightSubsystem::Initialize`. `FScopedUnattendedRpc` only spans a dispatched
   handler body, and `ModalStateProbe` only reports once the transport is up.
2. **It is not a Slate modal.** It is a native `FPlatformMisc::MessageBoxExt`
   (`Engine/Source/Developer/Zen/Private/ZenServerInterface.cpp:2400-2405`), so even a
   process-wide `GIsRunningUnattendedScript` set from inside the editor would not
   cancel a window that the Slate `AddModalWindow` guard never sees.

The engine's only off switches are on the guard itself:

```cpp
else if (!(FApp::IsUnattended() || IsRunningCommandlet() || GIsRunningUnattendedScript)
         && ZenWaitDuration > 20.0 && (DurationPhase == EWaitDurationPhase::Medium))
{
    FText ZenLongWaitPromptTitle = NSLOCTEXT("Zen", "Zen_LongWaitPromptTitle", "Wait for ZenServer?");
    ...
    FPlatformMisc::MessageBoxExt(EAppMsgType::YesNo, ...)
```

so only `-unattended`, a commandlet, or `-RunningUnattendedScript` prevent it. There
is **no** targeted switch analogous to `-AutoDeclinePackageRecovery`, and
`-NoZenAutoLaunch` is not one — it changes DDC behaviour rather than suppressing a
dialog.

The visible-launch case is therefore uncovered by construction: `-unattended` is
deliberately confined to `_HEADLESS_FLAGS` because it makes
`FEditorFileUtils::PromptForCheckoutAndSave` return `PR_Cancelled` and save nothing
(see `Docs/wiki-src/unattended.md`), and the default `editor_start({})` path uses the
OS file association, which carries no command line at all.

Detect-and-dismiss was considered and rejected: the proxy cannot see the dialog (no
endpoint yet) and synthesising a click on an arbitrary native message box is not
something an automation layer should do.

## What shipped

Partial, and deliberately so.

- `editor_start`'s new `map` parameter defaults `unattended_script` to **true**,
  because a map launch is by definition an agent-driven boot and already forces a
  direct spawn. Keyed on the new parameter so no existing call shape changes
  behaviour (`F-editor-start-map-parameter`).
- For every other launch shape the default stays `false` — the documented reason
  (a human sharing a visible window would silently lose every confirmation prompt)
  is unchanged and still right. Instead, `_startup_modal_hint(cmdline)` appends a
  sentence to `EDITOR_START_TIMEOUT` (both the direct-spawn and association waits)
  naming this hazard and `unattended_script: true`, suppressed when the launch
  already carried `-RunningUnattendedScript`. A readiness timeout is the only honest
  signal available at that layer.
- `Docs/wiki-src/unattended.md` documents the prompt, its engine guard, and why the
  in-editor suppression cannot reach it.

## Open question for the maintainer

Whether `unattended_script` should default on for **every** `visible: false` launch
(no human is sharing a hidden window, and `-unattended` is already in that flag set,
so the prompt is already suppressed there — this would be belt-and-braces), or stay
opt-in. Not taken here because the brief required existing calls to behave exactly
as today.

## Not verified

No runtime verification — the editor was owned by another agent. Needs a live pass:
a cold `editor_start {map}` on a machine where Zen takes over 20 s to start.

## See also

- `B-physics-asset-factory-modal-hang` — same class, in-editor factory modal.
- `F-editor-start-map-parameter` — the map parameter this default is keyed on.

## History
- `#1-initial-repro` `OPEN` reporter — An `editor_start` launch hung on a modal titled "Wait for ZenServer?" and did not recover; `unattended_script: true` cleared it. Root-caused by engine source read to `ZenServerInterface.cpp:2400-2405`: a native `FPlatformMisc::MessageBoxExt` raised during startup, gated only by `!(FApp::IsUnattended() || IsRunningCommandlet() || GIsRunningUnattendedScript)`. Neither `FScopedUnattendedRpc` (handler-body scoped, and the dialog fires before the subsystem loads) nor `ModalStateProbe` (Slate-loop based, and likewise post-startup) can cover it, so commit `7ff9dcf3` does not apply. severity rationale: impact=game-thread-hang with no error and no recovery x reach=any cold visible launch on a machine where Zen needs maintenance -> High.
- `#2-map-default-plus-timeout-hint` `IN-REVIEW` developer — Partial fix. `editor_start`'s new `map` parameter defaults `unattended_script` to true (map launches already force a direct spawn and are agent-driven by definition), keyed on the new parameter so existing call shapes are byte-identical. For all other shapes the default stays false and `_startup_modal_hint` now names this hazard plus `unattended_script: true` in the `EDITOR_START_TIMEOUT` text on both the direct-spawn and association waits, suppressed when the launch already carried `-RunningUnattendedScript`. Detect-and-dismiss rejected: the proxy has no endpoint to see the dialog through and must not synthesise clicks on native message boxes. Left open for the maintainer: whether `visible:false` should also default it on. Files: `Content/Python/mcp_proxy.py`, `Docs/wiki-src/unattended.md`. Not gate-verified live.
- `#3-slate-modals-need-the-other-flag` `IN-REVIEW` reporter — Additional evidence, and a correction to the Open question above. Its parenthetical — "`-unattended` is already in that flag set, so the prompt is already suppressed there" — is **false for Slate modals**, which is the class the decision turns on. `FSlateApplication::AddModalWindow` gates solely on `GIsRunningUnattendedScript` (`Engine/Source/Runtime/Slate/Private/Framework/Application/SlateApplication.cpp:2134`, `if (GIsRunningUnattendedScript && !bSlowTaskWindow)`) and never consults `FApp::IsUnattended()`; `LaunchEngineLoop.cpp:6858-6860` is the only global setter of that variable, and only `-RunningUnattendedScript` sets it. `-unattended` does cover the native `MessageBoxExt` ZenServer prompt this ticket was filed for, whose guard does read `FApp::IsUnattended()` (`ZenServerInterface.cpp:2400`), so the two flags cover different dialog classes and `_HEADLESS_FLAGS` covers only one of them. Current state confirmed post-wave: `Content/Python/mcp_proxy.py:1071` is `["-RenderOffScreen", "-unattended", "-nopause", "-nosplash", "-nocefaccelpaint"]`, and `:1794` keys `unattended_script` on `map` alone (`bool(args.get("unattended_script", start_map is not None))`) without consulting `visible`. So `editor_start {visible: false}` with no `map` can still wedge on a startup Slate modal — the same exposure `c35348dd` closed for `editor_run_tests` by adding the flag to its argv (`mcp_proxy.py:2516`), whose comment justifies the asymmetry with "editor_start can hand a VISIBLE editor to a person", a rationale that does not apply to a launch spawned with `CREATE_NO_WINDOW` and a hidden `STARTUPINFO` (`_spawn_kwargs`, `:1168-1177`). Note for whoever takes the decision: the present behaviour is deliberately pinned by `Content/Python/tests/test_mcp_proxy_editor_start.py:784-787` (`test_no_map_keeps_modal_suppression_off`, commented "Backwards compatibility: an existing call shape must behave exactly as before"), which drives the real `_editor_start` with only `Popen` mocked — so a patch must retire or amend that test, and this is a decision rather than a straightforward fix. Status unchanged; filed as evidence, not as a verification.
