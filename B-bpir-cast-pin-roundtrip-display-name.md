---
id: B-bpir-cast-pin-roundtrip-display-name
title: "BPIR decompile emits cast result pin with display name (spaces, no _C); compile expects FName form"
status: DONE
severity: Medium
category: bug
tags: [bpir, decompiler, round-trip, cast, pin-name]
---

# Cast pin reference round-trip mismatch

`blueprint.decompile` emits cast-result pin references using the pin's
FRIENDLY display name (with spaces inserted between camel-cased words,
backtick-quoted, `_C` suffix stripped). Example from
`W_MissionSelectButton::Button.OnClicked`:

```
%n1 = cast<B_DroneGameInstance_C>(%n0) [success -> @ok]

@ok:
    set %n1.`AsB Drone Game Instance`.bMissionIsJustOpened = true
```

Feeding that text back into `blueprint.compile_bpir` fails:

```
COMPILE_FAILED: Cannot set '%n1.`AsB Drone Game Instance`.bMissionIsJustOpened':
Reference '%n1.`AsB Drone Game Instance`' not found in emitted nodes.
```

The compiler expects the canonical FName form, which per
`bpir-examples.md:743` strips underscores from the class suffix and
omits spaces:

```
set %n1.AsBDroneGameInstance.bMissionIsJustOpened = true
```

So the documented round-trip form is `AsBDroneGameInstance` (no spaces,
no underscores, no `_C`), but the decompiler emits the user-friendly
form `AsB Drone Game Instance` (display formatter inserts spaces).

## Repro

1. Find any BP with a cast through a user-named class with underscores
   or camelCase (e.g. `B_DroneGameInstance_C`, `W_TrackLoaderUtil_C`).
2. `blueprint.decompile` of any graph that does the cast.
3. Observe the dot-accessor on the cast result is emitted with
   spaces (e.g. `%n1.\`AsB Drone Game Instance\``).
4. Feed the same body into `blueprint.compile_bpir`.
5. `COMPILE_FAILED: Reference '...' not found in emitted nodes.`
6. Strip spaces and underscores (`AsBDroneGameInstance`); recompile.
7. Now succeeds.

## Impact

- Breaks decompile → edit → compile for any graph containing a cast
  through a class name with underscores or multiple words. Very common
  — most project-asset BP classes match (`B_*`, `W_*`, `BP_*`, `DA_*`,
  `_C`-suffixed generated classes).
- Cast through engine classes with single-word names (e.g.
  `cast<Pawn>` → `.AsPawn`) round-trips OK, hiding the bug from quick
  smoke tests.

## Workaround

Manually strip spaces and `_C` suffix from every cast accessor in the
decompiled BPIR before re-feeding it to the compiler. Easy to miss when
there are many casts; backtick-quoted display names look syntactically
valid.

## Fix

Decompiler should emit the canonical compile-side form from the active
output-reference path in `BpirDecompiler.cpp`, not from `BpirTextEmitter`
or `PinFriendlyName`. For `UK2Node_DynamicCast` result output pins, the
runtime `PinName` can already be display-derived with spaces (for
example `As B Drone Game Instance`). Strip spaces from dynamic-cast
result pin names before passing them through `FIrTextUtils::FormatNameToken`,
and use that helper in both `ResolveInputValue` output-ref formatting
paths. Do not widen compiler parsing to accept backtick display names;
the decompiler should keep emitting the canonical accessor the compiler
already understands.

Regression coverage: cast through a generated class whose cast-result
pin is display-derived with spaces, compile using the canonical no-space
accessor, decompile, assert the canonical accessor is emitted without
backticks/spaces, strip authored positions, and recompile successfully.

Related: this is the same shape as `B-bpir-subsystem-getter-roundtrip`
(decompile-side emits non-recompilable form). The compiler is correct;
the decompiler should match it.

## History
- `#1-initial-repro` `OPEN` reporter — Session recompile of `W_MissionSelectButton::LoadMission` macro. Took the existing decompile (which had `%n1.\`AsB Drone Game Instance\``), pasted into `compile_bpir`, got `COMPILE_FAILED: Reference '%n1.\`AsB Drone Game Instance\`' not found in emitted nodes`. Iterated through several variants: `AsB Drone Game Instance` (decomp form), `AsB_DroneGameInstance` (with underscore), `AsBDroneGameInstance` (no spaces, no underscores) — only the last succeeded. The successful form matches the documented rule in `bpir-examples.md:743`: "strips underscores from the class-name suffix: `W_ReplaySaveHandler_C` → `.AsWReplaySaveHandler`." Decompiler emits the display formatter's spaced version instead.
- `#2-additional-repro-photo-popup` `OPEN` reporter — Re-encountered on `/App/App/UI/W_PhotoPopup` `ShowPopup` authoring. `cast<PhotoInspectionTrack>(%n1)` decompiled to `%n2.\`AsPhoto Inspection Track\`` (with backticks + spaces), inferred from existing decompiled `OnAcceptClicked` body. Pasted that form into a fresh `compile_bpir` and got `COMPILE_FAILED: Line 8: Could not resolve value '%n2.\`AsPhoto Inspection Track\`' for pin 'Target'`. Stripping spaces to `%n2.AsPhotoInspectionTrack` (no backticks) compiled. Cost ~3 retry cycles iterating syntax. Confirms the bug is reproducible on engine-tonameless project classes (single-token prefix `A` followed by camelCased name `PhotoInspectionTrack`), not just multi-token `B_*`/`W_*` classes.
- `#3-canonical-cast-accessor` `IN-REVIEW` developer — Updated `BpirDecompiler.cpp` so both `ResolveInputValue` output-reference paths format dynamic-cast result pins through a helper that strips spaces before `FIrTextUtils::FormatNameToken`, leaving non-cast output refs unchanged. Added `FBpirCastPinCanonicalAccessorRoundTripTest` to compile a canonical cast accessor, assert decompile keeps the no-space accessor without backticks, and recompile the decompiled BPIR after stripping authored positions.
- `#4-verify-fix` `DONE` tester — Verified via `blueprint.decompile` on both repro assets. `/App/App/UI/LobbyAndMenu/Elements/W_MissionSelectButton` now emits 25 instances of canonical `AsBDroneGameInstance` and zero instances of the spaced `` `AsB Drone Game Instance` `` form. `/App/App/UI/W_PhotoPopup` emits 4 instances of canonical `AsPhotoInspectionTrack` and zero of `` `AsPhoto Inspection Track` ``. Both no longer use backticks for the cast accessor.
