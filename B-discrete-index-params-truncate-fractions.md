---
id: B-discrete-index-params-truncate-fractions
title: "Discrete index parameters accept fractional JSON numbers, truncate them to int32, and can mutate the wrong array element or local player"
status: IN-REVIEW
severity: High
category: bug
tags: [parameters, integer, coercion, index, container, session]
encounters: 1
lastSeen: 2026-09-03T23:17:31+03:00
---

# Fractional indices are silently truncated

## What happens

The declared-type gate treats `number` and `integer` identically and accepts every JSON number
(`Handlers/ParamTypeCheck.h:337-345`). `FHandlerContext::GetInt` then calls
`FJsonObject::GetIntegerField` (`Handlers/HandlerContext.cpp:45-50`), whose UE 5.8 implementation
is a direct C-style cast from double to `int32`
(`C:/UE_5.8/Engine/Source/Runtime/Json/Private/Dom/JsonObject.cpp:409-412`).

All four Utility array index slots are declared `number` and read through that path:
remove (`UtilityPropertyHandler.cpp:2299-2304`), insert (`:2371-2377`), get (`:2438-2443`), and
set (`:2492-2498`). Thus `index:1.9` becomes `1`; remove/set mutates element 1 and returns success
for that truncated index. `session.remove_local_player` repeats the same mechanism for
`playerIndex` (`SessionsHandler.cpp:109-116`), so a fractional selector can remove the wrong player.

## Why it matters

Range checks run after truncation, so they certify a different target than the caller supplied.
This is High silent wrong-target behavior on destructive methods.

## What should happen

Add one exact-int reader/validator that accepts only finite, integral values in `int32` range, and
use it for discrete slots. Retype these declarations to `integer` and stop normalizing `integer`
to unrestricted `number`, or enforce the distinction at the handler helper if wire compatibility
requires keeping existing declarations. Failure must occur before `Modify`, removal, insertion, or
player teardown. Add negative cases for fractions and out-of-range numbers.

**Workaround:** Send whole JSON numbers only and confirm the returned/read-back target identity.

## Fix

The root cause was two-layered: the shared context readers still used UE's truncating integer
accessor, and the declared-type gate normalized `integer` to unrestricted `number`. `GetIntOr`,
`GetInt`, `GetIntFirstOf`, and `RequireInt` now use `TryParseStrictJsonInteger` with the inclusive
int32 range; malformed present values emit typed `INVALID_PARAMS` and `GetIntOr` returns unset.
On the normal dispatcher path, `ParamTypeCheck.h` keeps `integer` distinct, so schema rejection is
`PARAM_TYPE_MISMATCH` before the handler body. Direct handler strict-reader rejection is
`INVALID_PARAMS`. The four `container.array.*` index slots and
`session.remove_local_player.playerIndex` are declared `integer` and use `RequireInt`.

The follow-up converted 83 schema declarations after restoring semantic-name exceptions (paths,
names, booleans, arrays, fractional rates, and float LOD/reverb values). The source ratchet checks
camel/underscore discrete-name tokens with an explicit exception allowlist and checks literal
`Ctx.GetInt`, `Ctx.GetIntOr`, and `Ctx.RequireInt` readers where declared; discrete-name matching
continues to cover the known `GetIntFirstOf` integer parameters. It
also skips non-scalar declarations for name-only matching, so object-valued `wrapperSlot` is not
treated as an integer. `GetInt` remains a compatibility getter: an
invalid present value reports `INVALID_PARAMS` and returns its caller-supplied fallback; callers
that must distinguish invalid from absent use `GetIntOr`.

The suite-11 follow-up converted seven additional genuinely discrete declarations: actor and
spatial `expectedCount`, four MetaSound `intValue` parameters, and `sequencer.set_playhead.frame`.
The latter now uses the strict integral reader and leaves fractional `time` support unchanged.
The two level-structure asset-path defaults were restored to `path`, allowing the handler's
`INVALID_ARGUMENT` name validation to run before path sanitization.

Changed files:

- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\HandlerContext.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\ParamTypeCheck.h`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Utility\UtilityPropertyHandler.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\System\SessionsHandler.cpp`
- 83 discrete-reader schema declaration hunks across the PinWright source modules (semantic exceptions listed above were left unchanged)
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\HandlerContext.h`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Infra\TestParamTypeGate.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Infra\TestHandlerContext.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Infra\TestDeclaredParamCoverage.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Utility\TestDiscreteIndexParams.cpp`
- Suite-11 follow-up: `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Drive\TestDriveWindowSelectorParams.cpp`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Infra\TestAutoRegistration.cpp`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Infra\TestParamTypeGate.cpp`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Actor\InstancedMeshHandler.cpp`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Spatial\GroundPlacementHandler.cpp`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Audio\MetaSound\MetaSoundVariableHandler.cpp`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Audio\MetaSound\MetaSoundNodeInputDefaultHandler.cpp`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Audio\AudioAuthoringHandler.cpp`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Sequencer\SequenceHandler.cpp`, and `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Level\LevelStructureHandler.cpp`

Exact automation IDs added are `PinWright.infra.dispatcher.ParamTypeGate.RefusesNonIntegralIntegerValues`,
`PinWright.infra.dispatcher.ParamTypeGate.DiscreteReadersHaveIntegerDeclarations`,
`PinWright.infra.handler_context.GetIntRejectsFractionalAndOutOfRange`, and
`PinWright.container.DiscreteIndexParams.RejectsFractionalAndOverflow`. Coverage exercises
fractional and both overflow directions through the shared context reader and the dispatcher gate,
plus `InvokeHandlerWithCapture` for array remove/set and local-player removal with unchanged state.
No live editor, build, or automation run was performed in this source-only pass.

## Related

- Catalog: `coercion-slot-unit-direction-drift`, `wrong-target-scope-or-identity`
- `B-param-type-never-validated` — fixed wrong JSON shapes while deliberately preserving number
  coercions; this follow-up gives the `integer` declaration its strict integral policy.

## History
- `#1-fractional-index-truncation` `OPEN` reporter — Source-read the shared accessor, declared-type
  gate, UE JSON cast, and destructive array/player call sites. No runtime mutation was performed.
- `#2-strict-integral-index-policy` `IN-REVIEW` developer — Wired the shared strict int32 parser into
  `RequireInt`, preserved the pre-mutation failure path in four array index verbs and local-player
  removal, made `integer` distinct in the dispatcher gate, and added dispatcher plus handler-level
  fraction/overflow regression coverage. Static review only; no build or live editor verification.
- `#3-shared-context-reader-and-ratchet` `IN-REVIEW` developer — Added strict `GetIntOr`/`GetInt`
  validation and context coverage for fractional, positive-overflow, negative-overflow, and valid
  integral values; added the source/registry integer-declaration ratchet; converted 83 discrete
  reader declarations while documenting explicit semantic exceptions. Dispatcher schema failures
  are `PARAM_TYPE_MISMATCH`; direct strict-reader failures are `INVALID_PARAMS`. Static inspection
  only; no build, test, editor, MCP, or Git command was run.
- `#4-suite-11-gate-and-schema-corrections` `IN-REVIEW` developer — Corrected stale integer
  expectations for `PinWright.drive.aliases.VerbsDeclareWindowSelectorParams` and
  `PinWright.infra.auto_registration.ParamSpecData`; converted seven discrete declarations,
  including strict integral `sequencer.set_playhead.frame`; constrained the ratchet to exclude
  object-valued `widget.wrap.wrapperSlot`; and restored both level asset-path declarations to
  `path` so `PinWright.level.structure.configure_hlod_layer.NameCarryingAPathIsRefused` and
  `PinWright.level.structure.create_data_layer.NameCarryingAPathIsRefused` reach handler-level
  `INVALID_ARGUMENT` validation. Static inspection only; no suite rerun.
- `#5-one-escapee-layertag-and-audit` `IN-REVIEW` developer — Cross-ticket note: the sweep had exactly one escapee, `layerTag`, whose name-token heuristic matched on `layer` while the parameter carries a CommonGame/Lyra UI layer gameplay tag read as a string. Declared `integer` on all four `ui.activatable_*` verbs through one shared macro, it made every tag value unreachable at the dispatcher's declared-type gate (`PARAM_TYPE_MISMATCH` before the handler ran) and silently killed `F-activatable-push-by-layer-tag`. Fixed under `B-layertag-declared-integer-blocks-tag-addressing` (declaration back to `"string"` in `Source/PinWrightCommonUI/Private/Handlers/UI/UiActivatableStackHandler.cpp`). A mechanical audit of the rest found no second case: every `integer`-declared param was checked against its reader, and no other `integer` declaration is read through `Ctx.GetString`. The classifier is also closed against a repeat — `Tests/Infra/TestParamTypeGate.cpp` `IsDiscreteReaderName` now excludes any name carrying a `tag` token, verified safe by grep (no param name containing the token `tag` is declared `integer`, and no `GetInt`/`GetIntOr`/`RequireInt`/`GetIntFirstOf` call site reads a tag-named key; `stageIndex` matches only as a substring and tokenizes to `stage`+`index`, so it is unaffected). Static inspection only; no compile, no suite rerun.
