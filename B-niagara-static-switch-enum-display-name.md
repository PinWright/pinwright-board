---
id: B-niagara-static-switch-enum-display-name
title: "niagara.set_static_switch resolves an enum value by authored entry name only, so every label the Niagara editor shows for a user-defined enum is rejected with INVALID_VALUE — and index 0 is not the option a caller would guess"
status: DONE
severity: Medium
category: bug
tags: [niagara, set_static_switch, enum, display-name, user-defined-enum, newenumerator, invalid-value, discoverability]
encounters: 1
lastSeen: 2026-08-28T09:15:00+05:00
---

# `niagara.set_static_switch` rejects the enum labels the editor displays, because it only ever looks up the authored entry name

Passing the value shown in the Niagara editor UI for an enum static switch fails:

```
[INVALID_VALUE] Enum value 'Sphere' not found on 'ENiagaraKillVolumeOptions'
```

Niagara's stock module enums are **user-defined (Blueprint) enums**. Their authored
entry names are `NewEnumerator0`, `NewEnumerator1`, … and the human-readable labels
(`Sphere`, `Box`, `Once`, `Infinite`) live in the enum's display-name metadata. The
handler resolves by entry name only and never consults display names, so **no label
a caller can see in the editor is accepted** — only `NewEnumeratorN`, which is
meaningless to a caller, or a bare index, which is easy to get wrong (see the second
trap below).

## Root cause (guilty source lines)

`niagara.set_static_switch` is registered at
`Plugins/PinWright/Source/PinWright/Private/Handlers/Niagara/NiagaraEditHandler.cpp:2761`;
its body resolves the value at `:2804` via `NiagaraStaticSwitch::EncodePinDefault`.
That function lives in
`Plugins/PinWright/Source/PinWright/Private/Handlers/Niagara/NiagaraEditTypes.cpp:1913`,
and the enum branch (`:1954`) is the guilty code, `:1966-1974`:

```cpp
            else if (Json.IsValid() && Json->Type == EJson::String)
            {
                const FString FullName = Enum->GenerateFullEnumName(*Json->AsString());
                const int32 Index = Enum->GetIndexByName(FName(*FullName));
                if (Index == INDEX_NONE)
                {
                    OutError = FString::Printf(TEXT("Enum value '%s' not found on '%s'."), *Json->AsString(), *Enum->GetName());
                    return false;
                }
                EnumIndex = Index;
            }
```

`GenerateFullEnumName` + `GetIndexByName` is an **authored-entry-name** lookup.
`UEnum::GetDisplayNameTextByIndex` (and the `DisplayName` metadata behind it) is
never called. Neither symbol appears anywhere in the Niagara handler cluster —
repo-wide, `GetDisplayNameTextByIndex` / `DisplayNameMap` hit only
`Handlers/Blueprint/BlueprintEnumHelpers.cpp:66`,
`Handlers/Render/ViewModeVocabulary.cpp:223`, and three test files. So the
capability exists in this codebase; it was simply never wired into the Niagara
static-switch path.

Note the comment at `NiagaraEditTypes.cpp:1977-1978`:

```cpp
            // ResolveConstantValue expects the short (un-prefixed) display name for enum static switches.
            OutPinDefault = Enum->GetNameStringByIndex(EnumIndex);
```

The comment says "display name"; `GetNameStringByIndex` returns the **authored
name**. The comment is wrong about what the code does, which is likely how the gap
survived review.

## Independent corroboration that these enums really are `NewEnumeratorN`

`F-niagara-reset-module-input` `#5-verify-static-switch-reset` (a `DONE` tester
entry from an unrelated session) records `niagara.reset_module_input` on
`/Game/UltraDynamicSky/Particles/Puddle_Ripple`, entry
`8CBE34B74E80AB6F1A9DE8B98442D668`, input `Loop Behavior` returning
`previousValue:"NewEnumerator1"`. A real production asset's `Loop Behavior` switch
surfacing a raw `NewEnumerator1` through a different verb, in a different session,
confirms the naming this ticket depends on.

## Verbatim repro

```
niagara.set_static_switch {assetPath, emitter, entryId,
                           inputName: "<volume shape switch>", value: "Sphere"}
  -> [INVALID_VALUE] Enum value 'Sphere' not found on 'ENiagaraKillVolumeOptions'
```

`"Sphere"` is the label the Niagara editor shows for that switch on the
`KillParticlesInVolume` module. The accepted forms are the raw `NewEnumeratorN`
entry name or a numeric index.

## Second, quieter trap in the same surface: guessing the index is silently wrong

Because labels are rejected, the natural recovery is to pass an index — and the
obvious guess is wrong on at least one very common switch. On
`ENiagara_EmitterStateOptions` (the `EmitterState` module's loop behaviour),
**index 0 is `Infinite`, not `Once`**. A caller who assumes the "first" option is
the simple one-shot case gets an endlessly looping emitter, with **no error** —
the numeric branch at `NiagaraEditTypes.cpp:1963-1966` takes any number verbatim
and never range-checks or name-checks it. So the loud failure (a rejected label)
pushes callers onto a silent one (a wrong branch).

This ordering was observed 2026-08-27 in this session and is carried over from the
session log; it was **not** re-verified against the enum asset in this tree.

The readback side has the same weakness statically: `DecodePinDefault`
(`NiagaraEditTypes.cpp:1883`) enum branch `:1902-1904` does the same name-only
lookup and then **silently coerces a miss to index 0**:

```cpp
                const int32 Found = Enum->GetIndexByName(FName(*FullName));
                Index = Found != INDEX_NONE ? Found : 0;
```

So an unresolvable stored name reads back as whatever index 0 happens to be. Static
observation from source only — not exercised in this session.

## What it should do

Resolve an enum static-switch string by trying, in order: the authored entry name
(current behaviour, unchanged), then the **display name** via
`UEnum::GetDisplayNameTextByIndex` across the enum's entries, matched
case-insensitively and ignoring spaces. On a miss, the `INVALID_VALUE` message
should list the accepted values as `<displayName> (index N)` pairs rather than
naming only the rejected string — that alone removes the `asset.dump` round-trip.
Range-check the numeric branch too, so an out-of-range index is refused rather than
written. Fix the `:1977` comment while the file is open.

## Workaround

`asset.dump` the enum, hand-map the label to its index, and pass the index — then
re-read the switch to confirm the branch is the intended one, because a wrong index
is accepted silently.

## Distinct from related tickets

- `B-niagara-static-switch-not-decoded` (DONE, Medium) is the **readback** gap —
  the resolved static-switch branch was missing from the IR entirely. That was fixed
  by adding the decode; this ticket is the **write** path's name resolution.
- `F-niagara-reset-module-input` (DONE) is cited above only as corroboration for the
  `NewEnumeratorN` naming; it is not about value resolution.
- `E-rendering-set-settings-enum-token-discovery` /
  `E-rendering-readback-enum-token-asymmetry` are the same *class* of problem in the
  `rendering` namespace (enum token discovery/asymmetry), different code path.
- `B-enum-raw-integers` / `B-bpir-byte-pin-accepts-bare-enum-literal` are Blueprint
  BPIR enum handling, unrelated to Niagara static switches.

severity rationale: impact=soft blocker — a documented input form is rejected and the task is only doable via a documented workaround (`asset.dump`, hand-map label to index, extra calls per switch), plus a silent wrong-branch hazard once the caller falls back to indices x reach=every static switch on a user-defined enum, which is how Niagara's stock modules expose their branch choices (`EmitterState` loop behaviour, `KillParticlesInVolume` shape, and their kin) -> Medium

## History
- `#1-initial-repro` `OPEN` reporter — Found building the Atlantis level (map as forcing function; host `CLAUDE.md` § "What this project is for"), 2026-08-27, UE 5.8, PinWright at `8e76cad5` in this checkout. `niagara.set_static_switch` with `value:"Sphere"` on the `KillParticlesInVolume` volume-shape switch returned `[INVALID_VALUE] Enum value 'Sphere' not found on 'ENiagaraKillVolumeOptions'` — the label the Niagara editor displays. Source-confirmed in this tree: registration `NiagaraEditHandler.cpp:2761`, resolution at `:2804` into `NiagaraStaticSwitch::EncodePinDefault` (`NiagaraEditTypes.cpp:1913`), enum branch `:1954`, guilty lines `:1966-1974` — `GenerateFullEnumName` + `GetIndexByName`, an authored-entry-name lookup, with no `GetDisplayNameTextByIndex` / `DisplayNameMap` anywhere in the Niagara handler cluster (repo-wide those symbols hit only `Handlers/Blueprint/BlueprintEnumHelpers.cpp:66`, `Handlers/Render/ViewModeVocabulary.cpp:223` and three test files). The `:1977` comment claims "display name" while the call is `GetNameStringByIndex` (authored name) — comment wrong, code name-only. Corroboration from an unrelated session that these enums really are `NewEnumeratorN`: `F-niagara-reset-module-input` `#5-verify-static-switch-reset` records `previousValue:"NewEnumerator1"` for a real asset's `Loop Behavior` switch. Second trap recorded in the body: `ENiagara_EmitterStateOptions` index **0 is `Infinite`, not `Once`**, so a caller guessing an index instead of a label silently gets the wrong branch — carried over from the session log, **not** re-verified against the enum asset here; the numeric branch (`:1963-1966`) takes any number verbatim with no range check. Third, static-only observation: the decode side (`DecodePinDefault`, `:1883`) does the same name-only lookup at `:1902-1904` and silently coerces a miss to index 0.
- `#2-accept-editor-display-names` `IN-REVIEW` developer — Fixed together with `B-niagara-static-switch-enum-value-map-undiscoverable` (same function, same mechanism); see that ticket for the published branch-table half. Replaced the enum branch of `NiagaraStaticSwitch::EncodePinDefault` (`Handlers/Niagara/NiagaraEditTypes.cpp`) with a call to the new `ResolveEnumOption`, which resolves a string via `BlueprintEnumHelpers::TryResolveEnumLiteralToValue` — the shared resolver already covering authored name, fully qualified name, redirects and `GetDisplayNameTextByIndex`, reused per the DRY rule rather than reimplemented — then falls back to a space- and underscore-collapsed case-insensitive match against both the authored name and the editor's label. Authored `NewEnumeratorN` input is unchanged, so nothing that worked before stops working. The numeric branch is now range-checked against the same table: an index that is not a selectable branch (out of range, or the trailing `_MAX`) is refused instead of being written and silently clamped to branch 0 by `FNiagaraCompilationNodeStaticSwitch::GetBaseInputChannel`, and a value that is neither a number nor a string is refused instead of falling through to index 0. Every rejection now lists the whole table as `<displayName> / <name> (index N)`, and the handler attaches `enumPath` + `enumOptions[]` to the `INVALID_VALUE` payload. The wrong `// ResolveConstantValue expects the short (un-prefixed) display name` comment is corrected — the pin stores the AUTHORED name, which is what Niagara reads back. Test `PinWright.niagara.set_static_switch.EnumBranchResolution` (`Tests/Niagara/TestNiagaraStaticSwitchEnum.cpp`) drives a synthetic `UUserDefinedEnum` whose entry order permutes its `NewEnumeratorN` names: it asserts "Random Uniform" and "  randomUNIFORM " both select branch 2 (both returned `INVALID_VALUE` before), that `NewEnumerator3` and the number `1` still both select branch 1, and that index 7, the `_MAX` index and a boolean are all refused with the table in the message.

- `#3-verified-against-built-binary` `DONE` verifier - Behavioural repro against the running editor (PinWright HEAD `b79ba53e`), 2026-08-28, UE 5.8, on a scratch `asset.duplicate` of `SimpleExplosion`. Subject: `OmnidirectionalBurst`/`InitializeParticle` `Sprite Size Mode`, enum `/Niagara/Enums/ENiagara_SizeScaleMode` - a real asset whose authored order is genuinely permuted (index 1 = `NewEnumerator3` = "Uniform", index 2 = `NewEnumerator1` = "Random Uniform"), so this is the same trap `#1` describes for `ENiagaraKillVolumeOptions`. **Display names are accepted**: `value:"Non-Uniform"` returned `success:true, value:"NewEnumerator4", index:3, displayName:"Non-Uniform"`, and the deliberately sloppy `value:"  randomUNIFORM "` returned `index:2, value:"NewEnumerator1", displayName:"Random Uniform"` - the space/underscore/case collapse is live, and the *authored* name is what lands on the pin, as `#2` said. **The write lands**: a fresh `niagara.inspect {includeStack:true}` reads that switch back as `value: 2, source:"override"`. **The numeric branch is range-checked**: `value:7` is refused with `[INVALID_VALUE] Enum branch index 7 is not selectable on 'ENiagara_SizeScaleMode'; accepted values: Unset / NewEnumerator0 (index 0), Uniform / NewEnumerator3 (index 1), Random Uniform / NewEnumerator1 (index 2), Non-Uniform / NewEnumerator4 (index 3), Random Non-Uniform / NewEnumerator2 (index 4).` with `enumPath` + `enumOptions[]` attached - where before it would have been written and silently clamped to branch 0 by `GetBaseInputChannel`. Not re-run: the literal `"Sphere"` / `ENiagaraKillVolumeOptions` call from `#1` - `KillParticlesInVolume` is not in this fixture, and the code under test is the one `ResolveEnumOption` every enum switch goes through.
