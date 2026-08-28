---
id: B-niagara-static-switch-enum-value-map-undiscoverable
title: "niagara.set_static_switch takes an integer no published read can supply, so an enum switch silently selects the wrong branch"
status: DONE
severity: High
category: bug
tags: [niagara, static-switch, user-defined-enum, asset-dump, discoverability, silent-noop]
encounters: 1
lastSeen: 2026-08-28T09:15:00+05:00
---

# The integer `set_static_switch` wants cannot be derived from any read verb

`niagara.set_static_switch` refuses enum display names and requires the integer enum value.
The only published read of a Niagara user-defined enum is `asset.dump` on the enum asset, and its
`properties.json` carries exactly two keys — `DisplayNameMap` and `UniqueNameIndex`. `DisplayNameMap`
is a `TMap<FName,FText>` keyed by the *internal* enumerator names (`NewEnumerator0..N`). The ordered
`Names` array that maps an enumerator to its integer value is not dumped, so **there is no route from
a display label to the integer the setter requires.**

The natural assumption — that the `N` in `NewEnumeratorN` is the integer value — is false, and it is
not even consistently false, so it cannot be worked around by a fixed offset.

## Measured value -> enumerator mapping

Probed by calling `niagara.set_static_switch` with each integer and reading the echoed name
(`/Game/Atlantis/VFX/NS_Plankton_Drift`, emitter `Plankton`, module `InitializeParticle` V2,
`entryId 77322D3549C8C4517DEB0EA6AD30267B`):

`/Niagara/Enums/ENiagara_SizeScaleMode` — input `Sprite Size Mode`:

| passed `value` | echoed name | label from `DisplayNameMap` |
|---|---|---|
| 0 | `NewEnumerator0` | Unset |
| 1 | `NewEnumerator3` | **Uniform** |
| 2 | `NewEnumerator1` | **Random Uniform** |

`/Niagara/Enums/ENiagara_LifetimeMode` — input `Lifetime Mode`:

| passed `value` | echoed name | label |
|---|---|---|
| 1 | `NewEnumerator1` | Random |

So the mapping is identity for `ENiagara_LifetimeMode` and a permutation for
`ENiagara_SizeScaleMode`. A caller cannot tell which without probing.

`/Niagara/Enums/ENiagara_MassInitializationMode` makes it worse: its `DisplayNameMap` keys are
`NewEnumerator0`, `NewEnumerator1`, `NewEnumerator3` — **no `NewEnumerator2`** — so "index into the
sorted key list" is wrong too. (Its `NewEnumerator0` label is `Unset / (Mass of 1)`, which is also
worth knowing: an unset mass is 1, not 0, so force integration does not divide by zero.)

## Repro

```
asset.dump {assetPath: "/Niagara/Enums/ENiagara_SizeScaleMode.ENiagara_SizeScaleMode"}
  -> properties.json contains only DisplayNameMap {NewEnumerator0:"Unset",
     NewEnumerator1:"Random Uniform", NewEnumerator2:"Random Non-Uniform",
     NewEnumerator3:"Uniform", NewEnumerator4:"Non-Uniform"} and UniqueNameIndex.
     No value<->name table anywhere in the dump.

niagara.set_static_switch {assetPath:"/Game/Atlantis/VFX/NS_Plankton_Drift",
                           emitter:"Plankton",
                           entryId:"77322D3549C8C4517DEB0EA6AD30267B",
                           inputName:"Sprite Size Mode", value: 1}
  -> success:true, value:"NewEnumerator3"      // "Uniform", NOT the "Random Uniform" asked for
```

## Evidence that it fails silently

Selecting `Uniform` instead of `Random Uniform` made `InitializeParticle` read
`Uniform Sprite Size` rather than `Uniform Sprite Size Min` / `Max`, so every particle got one fixed
size instead of a random range. After the wrong write:

- `niagara.set_static_switch` returned `success: true`.
- `niagara.compile` succeeded.
- `niagara.validate {level:"strict"}` returned `valid: true`, `errors: []`.
- `niagara.decompile_nir` shows the resolved switch only as
  `input \`Sprite Size Mode\` = NewEnumerator3` — again the internal name, not the label.

Nothing at any layer says the selected branch is not the one the caller named. The only signal is
the echoed `NewEnumerator3`, and decoding *that* needs the same table that is missing.

## Impact

Static switches on user-defined enums are how every V2 Niagara module selects behaviour —
`InitializeParticle` (V2) alone exposes 14 of them (`Lifetime Mode`, `Sprite Size Mode`, `Mass Mode`,
`Color Mode`, `Position Mode`, `Sprite UV Mode`, ...), and `ShapeLocation` (V2) exposes `Shape
Primitive`, `Box / Plane Mode`, `Cone Mode` and more. Any authoring run that touches one can select a
different branch than it asked for and get a green compile, a green strict validate and a plausible
looking asset. The failure surfaces later as wrong-looking output with no call to blame.

## Workaround

Probe the enum before relying on it: call `niagara.set_static_switch` once per candidate integer with
`save: false`, read the echoed enumerator name, and cross-reference `DisplayNameMap` from
`asset.dump` to recover the label. One round trip per candidate value per enum, per asset.

## Suggested fix

Any one of these closes it; (a) plus (b) is the complete fix.

- **(a)** Accept the display name in `set_static_switch` and resolve it through
  `FUserDefinedEnumEditorData::DisplayNameMap` -> `UEnum::GetValueByName`. This is what callers
  already try first.
- **(b)** Emit an ordered `entries[]` of `{value, name, displayName}` in the enum's `asset.dump`,
  built from `UEnum::NumEnums()` + `GetValueByIndex()` + `GetNameStringByIndex()` +
  `GetDisplayNameTextByIndex()`. That is the table the dump is currently missing.
- **(c)** Echo `displayName` next to the internal name in the `set_static_switch` response, so the
  existing confirmation is readable without a second lookup.

## History
- `#1-initial-repro` `OPEN` reporter — Found while authoring `/Game/Atlantis/VFX/NS_Plankton_Drift`
  on the Atlantis map build. Passing `1` for `Sprite Size Mode` intending "Random Uniform" (its
  `DisplayNameMap` key index) silently selected "Uniform"; correct value turned out to be `2`.
  Probed both enums above to establish that the name index is a permutation of the value for
  `ENiagara_SizeScaleMode` and identity for `ENiagara_LifetimeMode`, so no fixed rule recovers it.
  `asset.dump` on the enum asset carries no value<->name table at all.
- `#2-publish-enum-branch-table` `IN-REVIEW` developer — Fixed together with `B-niagara-static-switch-enum-display-name` (same function, same mechanism); see that ticket for the resolver half. Added `NiagaraStaticSwitch::FEnumSwitchOption` + `BuildEnumOptions` / `MakeEnumOptionsJson` / `ResolveEnumOption` to `Handlers/Niagara/NiagaraEditTypes.h/.cpp`: the branch table is `{index, name, displayName}` per selectable entry, built over `NumEnums()` with the trailing `_MAX` sentinel and `Hidden` / `Spacer` entries dropped, mirroring the filter in `UNiagaraNodeStaticSwitch::GetOptionValues`. `index` is the branch selector, confirmed against `FNiagaraEditorUtilities::ResolveConstantValue` (engine `NiagaraEditorUtilities.cpp`), which resolves the caller-pin default through `GenerateFullEnumName` + `GetIndexByName` and uses the resulting INDEX, so the integer this verb takes is unchanged. The table is now published from three places: `niagara.set_static_switch`'s success response (`index`, `displayName`, `enumPath`, `enumOptions[]`, in `Handlers/Niagara/NiagaraEditHandler.cpp`), its `INVALID_VALUE` rejection payload, and — the read the ticket asks for — every enum entry of `staticSwitchInputs` in `NiagaraDumpBuilder::BuildStaticSwitchInputs`, which feeds `niagara_stack.json` and `niagara_model.json`; both aspect versions bumped 2 -> 3 in `Handlers/Asset/AssetDumpCache.cpp`. Test `PinWright.niagara.set_static_switch.EnumBranchTablePublished` (`Tests/Niagara/TestNiagaraStaticSwitchEnum.cpp`) builds a synthetic `UUserDefinedEnum` whose entry order permutes the `NewEnumeratorN` names and asserts the published table pairs index 2 with `NewEnumerator1` / "Random Uniform" — the mapping that previously existed in no response at all. NOT changed: `DecodePinDefault`'s silent coerce-to-0 on an unresolvable stored name (`NiagaraEditTypes.cpp`), because its one caller reports `source:"override"` off a successful decode and turning the miss into a failure would make the dump claim an override while printing the declared default; flagged for its own ticket.

- `#3-verified-against-built-binary` `DONE` verifier - Behavioural repro against the running editor (PinWright HEAD `b79ba53e`), 2026-08-28, UE 5.8, on a scratch `asset.duplicate` of `SimpleExplosion`. **The read the ticket asked for exists and carries the right numbers.** `niagara.inspect {includeStack:true}` now emits, on every enum entry of `staticSwitchInputs`, `enumPath` plus `enumOptions[]` of `{index, name, displayName}`. For `Sprite Size Mode` / `ENiagara_SizeScaleMode` that table is exactly the permutation `#1` had to establish by probing: `{0, NewEnumerator0, "Unset"}`, `{1, NewEnumerator3, "Uniform"}`, `{2, NewEnumerator1, "Random Uniform"}`, `{3, NewEnumerator4, "Non-Uniform"}`, `{4, NewEnumerator2, "Random Non-Uniform"}` - so the originating mistake (passing `1` for "Random Uniform" and silently getting "Uniform") is now a one-read lookup. The identity enum is published the same way (`Lifetime Mode` / `ENiagara_LifetimeMode`: `{0, NewEnumerator0, "Direct Set"}`, `{1, NewEnumerator1, "Random"}`), and no table carries a `_MAX` sentinel. **The write side publishes it too**: `set_static_switch` success responses carry `index`, `displayName`, `enumPath` and the same `enumOptions[]` (measured: `"Non-Uniform"` -> `index:3`), and so does the `INVALID_VALUE` rejection payload for an out-of-range index - evidence in `B-niagara-static-switch-enum-display-name` `#3`. Three publication points, all live in the built binary. Not separately checked: the on-disk `niagara_stack.json` / `niagara_model.json` sidecars, which `#2` says share `BuildStaticSwitchInputs` with the inspect path that was measured here.
