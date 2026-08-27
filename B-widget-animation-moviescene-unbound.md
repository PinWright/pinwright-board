---
id: B-widget-animation-moviescene-unbound
title: "Widget animations created via MCP are born unbound — MovieScene named `<Anim>_MovieScene`, so the generated property stays null"
status: OPEN
severity: High
category: bug
tags: [umg, widget-animation, silent-false-success]
---

# Widget animations created via MCP are born unbound

Reproduced live on a scratch widget, UE 5.8, plugin `d195a55d`:

```
widget.create_widget_animation animationName="ProbeAnim" -> success, movieSceneCreated:true

ProbeAnim_MovieScene -> FOUND MovieScene
ProbeAnim            -> not found
GetMovieScene()      = ProbeAnim_MovieScene

after blueprint.compile:
  ProbeAnim property value = None
```

The animation compiles clean, the generated `$ProbeAnim` property exists and is readable — and it is
null. Any event that plays the animation silently does nothing.

## Mechanism

The engine binds the generated property by the **MovieScene's** FName —
`Runtime/UMG/Private/WidgetBlueprintGeneratedClass.cpp:306-315`:
`InPropertyMap.Find(Animation->GetMovieScene()->GetFName())`. The property map is keyed on the
generated property name, which the compiler takes from the **animation** —
`Editor/UMGEditor/Private/WidgetBlueprintCompiler.cpp:1578`: `AnimVariableDesc.VarName = Animation->GetFName()`.

So the invariant is `MovieScene->GetFName() == Animation->GetFName()`. The engine's own designer
upholds it at all three mutation points: create (`TabFactory/AnimationTabSummoner.cpp:637-641`),
rename (`:281-282`, which renames *both* objects), duplicate (`:863`).

PinWright mints `<AnimName>_MovieScene` in two independent places:

- `Handlers/UI/WidgetAnimationHandler.cpp:68-84` (`EnsureAnimationMovieScene`) — reached by
  `create_widget_animation` (`:286`), and as a repair path by `add_animation_track` (`:352`),
  `add_animation_keyframe` (`:483`), `set_animation_speed` (`:708`)
- `Handlers/UI/WidgetAnimationJsonSerializer.cpp:2474-2483` — `import_animations_json`'s own copy

Nothing warns: `ValidateWidgetAnimations` (`WidgetBlueprintCompiler.cpp:1389-1417`) only checks that
each `FWidgetAnimationBinding` resolves to a live widget.

## Invisible to our own read-back

`widget.get_animation_info` never reports the MovieScene name (`WidgetAnimationHandler.cpp:757-770`,
`:803-826` return frame rate, range and bindings only), and the `pinwright.widget-animations.v1`
sidecar schema has no `movieSceneName` field. Neither `asset.dump` nor `widget.export_animations_json`
can detect this. "Read the value back" does not help — consider adding the field as part of the fix.

## Fix note — the tests currently bake the bug in

`Tests/WidgetAnimationJson/WidgetAnimationJsonTestUtils.cpp:181`, `:281` and `:407` all construct
fixtures as `%s_MovieScene`. A correct fix must update those three lines or the import tests fail on
the corrected name. The only existing name assertion (`WidgetAnimationJsonTests.cpp:311`) checks the
*animation's* name and that a MovieScene merely exists, never its name.

Missing: a test that instantiates the widget and asserts the generated `$<AnimName>` property is
non-null. That is the only test that proves a fix.

## Repair for already-broken assets

`unreal.load_object(None, '<pkg>.<BP>:MyAnim.MyAnim_MovieScene').rename('MyAnim')`, then `asset.save`
and `asset.reload`. Matches the engine's own rename at `AnimationTabSummoner.cpp:282`.

## History

- `#1-reported-with-live-repro` `OPEN` reporter — Confirmed end-to-end at runtime on `d195a55d` / UE 5.8: MovieScene named `ProbeAnim_MovieScene`, no `ProbeAnim` inner object, generated property reads None after compile. Engine binding path verified in engine source. Cross-checked 144 hand-authored animations across 96 widgets in the host project — all correctly named, so no MCP-created animation has yet been committed there.
