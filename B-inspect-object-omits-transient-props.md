---
id: B-inspect-object-omits-transient-props
title: "system.inspect.inspect_object silently drops Transient UPROPERTYs (e.g. a Replicated Transient bool), with no marker that properties were filtered"
status: DONE
severity: Medium
category: bug
tags: [system-inspect, inspect-object, properties, transient, pie, readback, silent-omission]
encounters: 1
lastSeen: 2026-09-29T13:05:00Z
---

# inspect_object omits Transient properties without saying so

`system.inspect.inspect_object {objectPath: "/Game/Maps/UEDPIE_0_L_PDS_Stadium.L_PDS_Stadium:PersistentLevel.DronePlayerState_1"}`
on a live PIE `ADronePlayerState` returned 127 properties, but not `bIsDroneSpectator`, declared
`UPROPERTY(Transient, Replicated, ReplicatedUsing=Rep_IsDroneSpectator) bool bIsDroneSpectator`
(`Plugins/App/Source/App/Drone/DronePlayerState.h:152`). The response carries no count of skipped
properties and no hint that a flag filter applied, so a caller looking for the field concludes
it does not exist. `property.get {objectPath: <same>, propertyName: "bIsDroneSpectator"}` read it
fine (`value: true`).

For live PIE objects, Transient properties are often exactly the runtime state being inspected
(replicated gameplay flags, caches), so dropping them hurts most on this method's main use case.

**Workaround:** `property.get` per property name, which means knowing the name from source.

**Fix (proposed):** include Transient properties (flag them `Transient` in `flags`, as other flags
already are), or, if they must be filtered, report `omittedProperties: {transient: N}` and document
the filter on the wiki page.

## History
- `#1-drone-spectator-flag-missing` `OPEN` reporter - Filed from a PDS multiplayer repro on UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt1`, plugin `61c243f5`, listen-server PIE. Needed the server-side spectator flag of a player; `inspect_object` did not list it, so I first used the possessed pawn class as a stand-in, then found `property.get` returns it. Cheap once found (a few calls).
- `#2-transient-props-reported` `IN-REVIEW` developer - `inspect_object` now builds `properties` with `BuildClassPropertyJson(Target, nullptr, /*bIncludeNonPersistent=*/true)`: Transient / DuplicateTransient / SkipSerialization UPROPERTYs are reported, each with its flag in `flags`; asset mirrors keep the old filter (the new parameter defaults to false, so `asset.dump` output is unchanged). Every reflected property still left out (deprecated, editor-regenerated noise) is named in the new `omittedProperties` array. Files: `Source/PinWright/Private/Utils/PropertyExport.h/.cpp` (`bIncludeNonPersistent` on `BuildClassPropertyJson` + `ShouldEmitClassDumpProperty`), `Handlers/Environment/EnvironmentHandler.cpp`, `docs/wiki-src/system.inspect.md`, `CHANGELOG.md`. Test: `PinWright.system.inspect.inspect_object.IncludesTransientProperties` (every AActor Transient UPROPERTY present and flagged; every reflected property in exactly one of `properties` / `omittedProperties`). Verify live: `inspect_object` on a PIE `PlayerState` lists a `Transient, Replicated` bool with `flags` containing `Transient`.
- `#3-verified-linux` `DONE` tester — PinWright `10212ee4` (on origin/master `6283b63b`), UE 5.8 Linux Vulkan. Runs: w23-final = offscreen full suite, no DISPLAY, 5568/5568 passed; w23-xfinal = DISPLAY=:0 offscreen, drive.os_input+click_occlusion+os_gesture+input, 31/31; w23-vis = DISPLAY=:0 windowed drive.input.ModifierChord, 2/2; Python = Content/Python/tests, 427 OK / 5 skipped (all skips Windows-only or an absent local script). `PinWright.system.inspect.inspect_object.IncludesTransientProperties` passed in w23-final: every AActor Transient UPROPERTY is listed with `Transient` in `flags`, and every reflected property is in exactly one of `properties` / `omittedProperties`. The proposed fix (include Transient properties, flag them, report what is omitted) is met. Limit: not re-run on a live PIE `ADronePlayerState`; the path is the same class reflection.
