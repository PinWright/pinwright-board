---
id: E-widget-anim-loop-speed-phantom-authoring-verbs
title: "widget.set_animation_loop / set_animation_speed method pages over-advertise persistable loop/speed params for verbs that ALWAYS hard-error [NOT_SUPPORTED] (the NOT_SUPPORTED guard isn't surfaced on the per-method page)"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [widget, animation, umg, wiki, not-supported]
---

# `widget.set_animation_loop` / `widget.set_animation_speed` method pages advertise a capability the engine can never persist

`widget.set_animation_loop` and `widget.set_animation_speed` are registered
RPCs whose **registration summary + auto-generated per-method page** present
them as ordinary persistable authoring verbs, complete with documented
parameters and **default values**:

- `widget.set_animation_loop` — summary "Configure loop settings for a widget
  animation"; params `loop` (boolean, default true) and `loopCount`
  (number, "0 = infinite (default 0)").
- `widget.set_animation_speed` — summary "Set playback speed for a widget
  animation"; param `speed` (number, "Playback speed multiplier (default 1.0)").

There are no hand-authored wiki pages for these verbs — the per-method page is
auto-generated from the registration `Summary` + `RPC_PARAMS` by
`WikiHandler::RenderMethodPage` (`Catalog/WikiHandler.cpp:410-426`).

In reality **every** call to either verb hard-errors `[NOT_SUPPORTED]`,
regardless of arguments. There is no input for which they succeed. The error
text is itself accurate about the engine, but the registration summaries and the
auto-generated method pages should not over-advertise, because a calling agent
that lands on `call("widget.set_animation_loop")` reads the summary, sees
documented params with defaults, and reasonably expects the call to persist onto
the asset.

**Where the warning is and isn't surfaced.** The namespace overlay
`docs/wiki-src/widget.md:468` already carries a caveat — "Runtime playback
settings such as loop count and playback speed are not persisted in the
animation asset. Set those at runtime through `PlayAnimation` in Blueprint or
C++." — and because it sits under a `## `-level structure it DOES render on the
`call("widget")` namespace page. So the gap is NOT "nothing in the wiki warns";
it is narrower: that caveat sits under the `### widget animation JSON` H3
(widget.md:447), so per the wiki rendering rules it does NOT render on the
**per-method** page, and there is no `### widget.set_animation_loop` /
`### widget.set_animation_speed` H3 overlay section either. An agent that calls
`call("widget.set_animation_loop")` directly therefore lands on a page that
advertises `loop`/`loopCount`/`speed` with defaults and shows zero caveat — the
registration summary and per-method page over-advertise.

The error is correct because loop count and playback speed are **runtime-only**
arguments to `UUserWidget::PlayAnimation(...)`, not serialized properties of the
`UWidgetAnimation` asset. Confirmed against engine source
`C:/UE_5.7/Engine/Source/Runtime/UMG/Public/Animation/WidgetAnimation.h`: the
asset's only `UPROPERTY`s are `MovieScene`, `AnimationBindings`,
`bLegacyFinishOnStop`, and `DisplayLabel` — no loop-count and no playback-speed
field exists to persist into. Correspondingly, `widget.get_animation_info`
returns no loop/loopCount/speed fields at all (only `durationSeconds`,
`frameRate`, `startFrame`, `endFrame`, `bindings`), so even a successful-looking
call could never round-trip.

This is the discoverability defect: the MCP surface (registered verb + summary +
auto-generated per-method page advertising defaulted params) promises
asset-level loop/speed authoring that the engine fundamentally cannot provide,
so a task like "configure this animation to loop infinitely at speed 1.0" leads
an agent straight into two unconditional `[NOT_SUPPORTED]` errors with no advance
warning on the page the agent reads first.

## Verbatim repro (replay-confirmed via `mcp__editor-automation__call`)

Setup:
- `widget.create_widget_blueprint` `{name: WBP_OracleReplayLoop, folder: /Game/UI}` -> ok
- `widget.create_widget_animation` `{widgetPath: /Game/UI/WBP_OracleReplayLoop, animationName: PulseStart, duration: 1.5}` -> ok

Failing verbs (these are exactly the documented params, incl. documented defaults):
- `widget.set_animation_loop` `{widgetPath: /Game/UI/WBP_OracleReplayLoop, animationName: PulseStart, loop: true, loopCount: 0}`
  -> `[NOT_SUPPORTED] Loop count is a runtime-only parameter controlled via PlayAnimation(). It cannot be persisted in the animation asset. Use PlayAnimation(Animation, 0.0, NumLoopsToPlay) in Blueprint or C++.`
- `widget.set_animation_speed` `{widgetPath: /Game/UI/WBP_OracleReplayLoop, animationName: PulseStart, speed: 1}`
  -> `[NOT_SUPPORTED] Playback speed is a runtime-only parameter controlled via PlayAnimation(). It cannot be persisted in the animation asset. Use PlayAnimation(Animation, 0.0, 1, EUMGSequencePlayMode::Forward, PlaybackSpeed) in Blueprint or C++.`

Readback confirming the fields can never round-trip:
- `widget.get_animation_info` `{widgetPath: /Game/UI/WBP_OracleReplayLoop, animationName: PulseStart}`
  -> `{success:true, durationSeconds:1.5, frameRate:60000, startFrame:0, endFrame:90000, bindings:[]}` (no loop/loopCount/speed keys)

## Fix

Follow the project's accepted pattern for always-`NOT_SUPPORTED`/`NOT_IMPLEMENTED`
verbs: make the **registration `Summary` self-documenting** so it auto-renders
both into the `## Methods` index AND into the per-method page (precedents:
`input.set_input_trigger`/`set_input_modifier` `InputHandler.cpp:190,207`;
`texture.create_cube_texture`/`create_volume_texture`/`create_texture_array`
`TextureHandler.cpp:2944-2956`; `misc.set_viewport_resolution`
`MiscHandler.cpp:255`; `debug.spawn_category` `DebugHandler.cpp:5` — all begin
"Returns NOT_IMPLEMENTED/NOT_SUPPORTED: ..."). Concretely:

- Prefix both registration summaries with "Returns NOT_SUPPORTED: ..." stating
  that loop count / playback speed are runtime `PlayAnimation()` arguments, NOT
  serialized `UWidgetAnimation` properties, and that the verb is a deliberate
  guard that always errors — pointing callers at the BPIR/Blueprint path for
  wiring `PlayAnimation` with the desired NumLoopsToPlay / PlaybackSpeed.
- Add `### widget.set_animation_loop` / `### widget.set_animation_speed` H3
  overlay sections to `docs/wiki-src/widget.md` so the per-method page carries
  the same runtime-only caveat in a Notes block (matching the IK-rig precedent
  `E-ik-rig-family-wiki-advertises-compiled-out-workflow`, which added per-method
  overlay H3 sections).
- Add a `WikiHandler::RenderPage` regression test (texture-precedent style,
  `FWikiHandlerTextureDocumentsCreateStubsTest`) that renders each method page and
  asserts the NOT_SUPPORTED / runtime-only caveat is present — it would fail if
  the summary prefix or overlay were reverted.

The runtime guidance string the handler already returns is good; the gap is that
the per-method page advertises the opposite, so the contradiction only surfaces
after the agent has already attempted the (impossible) authoring call. (The
`## Notes`-level namespace-page caveat at widget.md:468 stays as-is; it is not
the surface the agent reads when calling the method directly.) The weaker
"remove both verbs" option is rejected: it would discard the genuinely useful
runtime-`PlayAnimation` guidance the error currently returns.

**Workaround:** Do not call `widget.set_animation_loop` /
`widget.set_animation_speed`. To get looping/speed, wire
`UUserWidget::PlayAnimation(Anim, StartAtTime, NumLoopsToPlay, PlayMode,
PlaybackSpeed)` in the widget's Blueprint graph (e.g. via the BPIR/graph
authoring RPCs) — these are runtime play parameters, not asset state.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed via `mcp__editor-automation__call`: on a freshly created `/Game/UI/WBP_OracleReplayLoop` + `PulseStart` animation, both `widget.set_animation_loop` (`loop:true, loopCount:0`) and `widget.set_animation_speed` (`speed:1`) returned the verbatim `[NOT_SUPPORTED]` errors quoted above (identical to the seed attempt's log), and `widget.get_animation_info` surfaced no loop/loopCount/speed fields (only durationSeconds/frameRate/startFrame/endFrame/bindings). Verified the error is engine-accurate: `WidgetAnimation.h` (UE 5.7) shows `UWidgetAnimation` has only `MovieScene`/`AnimationBindings`/`bLegacyFinishOnStop`/`DisplayLabel` UPROPERTYs — no loop-count or speed field. Classified ergonomic (not bug): the calls fail *correctly*, but the wiki pages for both verbs advertise persistable `loop`/`loopCount`/`speed` params with documented defaults and give no warning that the call is a guaranteed `[NOT_SUPPORTED]` — a concrete, quotable discoverability gap. Dedup: ripgrep over the board for `set_animation_loop|set_animation_speed|loopCount|playback speed|NOT_SUPPORTED|PlayAnimation` found no existing ticket about these verbs (nearest neighbors `B-configure-sense-config-silent-noop`, `E-niagara-validate-strict-empty-system-undocumented`, `F-widget-screenshot-transient-overrides` are different namespaces/methods); animation tickets on the board concern dump/export fidelity and handler refactor, not loop/speed authoring.
- `#2-additional-loading-screen` `OPEN` reporter — Additional evidence (seed `widget.get_animation_info`; culprit `widget.set_animation_loop`): a separate "WBP_LoadingScreen pulsing spinner" task hit the identical wall. Replay-confirmed on a fresh `/Game/UI/WBP_OracleReplayLoop2` + `PulseStart` (duration 1.5): `widget.set_animation_loop` `{loop:true, loopCount:0}` -> `[NOT_SUPPORTED] Loop count is a runtime-only parameter controlled via PlayAnimation(). It cannot be persisted in the animation asset. Use PlayAnimation(Animation, 0.0, NumLoopsToPlay) in Blueprint or C++.` and `widget.set_animation_speed` `{speed:0.5}` -> `[NOT_SUPPORTED] Playback speed is a runtime-only parameter controlled via PlayAnimation(). It cannot be persisted in the animation asset. Use PlayAnimation(Animation, 0.0, 1, EUMGSequencePlayMode::Forward, PlaybackSpeed) in Blueprint or C++.` (verbatim-identical to `#1`). `widget.get_animation_info` `{animationName:PulseStart}` again returned only `{durationSeconds:1.5, frameRate:60000, startFrame:0, endFrame:90000, bindings:[]}` — no loop/loopCount/speed keys. Confirms the wiki-vs-engine contradiction is reproducible across tasks and arg values (loopCount:0 and speed:0.5 both rejected), and that the discoverability gap directly causes otherwise-complete authoring tasks (blueprint+image+animation+track+keyframes all succeeded) to fail their loop/speed success check. No change to the proposed fix (remove or loudly re-document both verbs + drop the phantom `loop`/`loopCount`/`speed` params from the wiki).
- `#3-reword-and-fix` `IN-REVIEW` developer — REWORD + implemented. Reworded title/body/Fix: the prior "Nothing in the wiki warns" claim was overstated — `docs/wiki-src/widget.md:468` already carries the runtime-only caveat under a `## `-level structure and DOES render on the `call("widget")` namespace page. Restated the real gap as registration-summary / **per-method-page** over-advertising (the caveat sits under the `### widget animation JSON` H3 so it never reaches the per-method page, and there were no `### widget.set_animation_loop` / `### widget.set_animation_speed` overlay sections), and re-pointed the Fix at the established self-documenting-summary precedent instead of removing/re-documenting nonexistent hand-authored wiki pages. Implemented per that pattern: (1) prefixed both registration summaries with "Returns NOT_SUPPORTED: ..." stating loop count / playback speed are runtime `PlayAnimation()` arguments, not serialized `UWidgetAnimation` properties, pointing at the BPIR/Blueprint path — `Source/EditorAutomationRpcGateway/Private/Handlers/UI/WidgetAnimationHandler.cpp:639,680` (auto-renders to the `## Methods` index AND the per-method page via `WikiHandler::RenderMethodPage`); (2) added `### widget.set_animation_loop` / `### widget.set_animation_speed` H3 overlay Notes sections to `docs/wiki-src/widget.md` so the per-method page carries the caveat (matching the IK-rig per-method-overlay precedent). Regression test (texture-precedent style, `WikiHandler::RenderPage`): `FWikiHandlerWidgetAnimationLoopSpeedDocumentsRuntimeOnlyTest` in `Source/EditorAutomationRpcGateway/Private/Tests/Infra/TestWikiHandler.cpp` renders both per-method pages and asserts each carries the NOT_SUPPORTED / runtime-`PlayAnimation` caveat (with an overlay-exclusive marker so it fails if the H3 section is reverted), and that neither is a Not-Found page. Did not compile or run tests (later phase).
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. No body citation was rewritten here. 2 citations sit in history rows and are left verbatim per the append-only rule. The one citation is in a history row and stays verbatim. Map: `Handlers/UI/WidgetAnimationHandler.cpp:639` → `Source/PinWright/Private/Handlers/UI/WidgetAnimationHandler.cpp:646`, and its sibling `:680` → `:687`; `:639` at HEAD is an `insert_keyframe` note string. The row's claim holds — both summaries carry the “Returns NOT_SUPPORTED:” prefix. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
