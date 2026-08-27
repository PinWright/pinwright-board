---
id: B-niagara-static-switch-enum-value-map-undiscoverable
title: "niagara.set_static_switch takes an integer no published read can supply, so an enum switch silently selects the wrong branch"
status: OPEN
severity: High
category: bug
tags: [niagara, static-switch, user-defined-enum, asset-dump, discoverability, silent-noop]
encounters: 1
lastSeen: 2026-08-27
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
