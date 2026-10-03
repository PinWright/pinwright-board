---
id: F-niagara-read-curve-keys
title: "No verb reads a Niagara curve data interface's keys back, so a curve is the one authored value that can never be verified or even discovered"
status: DONE
severity: Medium
category: feature
tags: [niagara, curve, data-interface, readback, verification, inspect]
encounters: 2
costly: 2
lastSeen: 2026-09-05
---

# The write side shipped; the read side did not

`F-niagara-curve-authoring` is `DONE`: `niagara.set_curve_keys` and
`niagara.bind_curve_asset` exist and write `FRichCurve` keys on a
`UNiagaraDataInterfaceCurve` / `ColorCurve` / `VectorCurve`. Nothing reads them.

Every other authored value on a Niagara stack has a readback path. A curve does not:

- `niagara.inspect {includeStack:true}` reports the module input as
  `{"name":"Scale Alpha","valueMode":"dynamicInput","value":null}` and the dynamic-input
  node as `FloatFromCurve001` with `{"name":"FloatCurve","valueMode":"data","value":null}`.
  No keys, no key count, not even a "this DI has N keys".
- `niagara.inspect {includeGraphs:true}` on the owning emitter (250 175 chars for
  `/Game/FPS/VFX/Emitters/E_ImpactWater_Crown`) emits the DI-typed **pins**
  (`"name":"DefaultCurve"`, `subCategoryObject:"/Script/Niagara.NiagaraDataInterfaceCurve"`,
  `defaultValue:""`, `defaultObject:""`) and nothing about the curve behind them. Searching
  the whole payload for `keys` / `CurveData` / `RichCurve` returns zero hits.
- `niagara.inspect {assetPath:"/Niagara/Modules/..."}` on a module **script** asset returns
  only `{assetPath, assetClass, assetKind}` — four fields, no body — so the DI cannot be read
  from the script side either.
- There is no `niagara.get_curve_keys`, and `property.get` needs the DI object's resolved
  path, which no verb publishes for a dynamic input nested under a module input.

## What it cost, concretely

Retuning `/Game/FPS/VFX/NS_Impact_Water` so a bullet strike throws a crown and a column, the
timing of the effect at 250 ms and 400 ms is governed by the alpha-over-life ramp on
`ScaleColor.Scale Alpha` → `FloatFromCurve001`. I could set the particles' positions and
sizes at those times to the centimetre from physics I control, and could not say whether the
sprite is at alpha 0.5 or alpha 0.05 there, because the ramp is unreadable. The report that
went back to the capture operator had to hedge exactly one axis — opacity — and only that one.

Two second-order consequences:

1. **A `set_curve_keys` write cannot be verified.** `CLAUDE.md` § "Verify a write against
   disk" requires checking the file, and `FRichCurve` keys are binary in the `.uasset` — a
   `grep -a` for a key value does not land. So the one verb in the family whose result cannot
   be confirmed by the project's own standard is the curve verb.
2. **An agent cannot inherit a curve.** Retuning an emitter someone else authored means
   either re-authoring the ramp blind (destroying a deliberate shape) or leaving it and
   reasoning about it as an unknown. I chose the second and said so; the first is the tempting
   move and it silently discards work.

## Proposal

`niagara.get_curve_keys {assetPath, entryId, inputName, emitter?, scriptUsage?}` — the exact
addressing `niagara.set_curve_keys` already takes — returning per-channel
`{channel, keys:[{time, value, interpMode, arriveTangent, leaveTangent}], boundAsset?}`.
`boundAsset` covers the `bind_curve_asset` flavour, where the answer is a `UCurveFloat` path
rather than inline keys.

Cheaper partial that would still have unblocked this task: have
`niagara.inspect {includeStack:true}` replace a data-valued module input's `"value": null`
with `{"keys": N, "range": [t0, t1], "valueRange": [v0, v1]}`. Four numbers are enough to say
"this ramp is still 0.9 at four fifths of life" without serialising every key.

## History

- `#1-filed` `OPEN` reporter — Hit while retuning `NS_Impact_Water`'s Crown and Column
  emitters. `niagara.set_curve_keys` (from `F-niagara-curve-authoring`, `DONE`) can write a
  curve; no verb in the namespace reads one. Confirmed against a live editor on port 27145:
  `niagara.inspect` with `includeStack` reports the input as `valueMode:"data", value:null`;
  with `includeGraphs` on `/Game/FPS/VFX/Emitters/E_ImpactWater_Crown` the 250 KB payload
  carries the DI-typed pins but no key data (zero matches for `keys`/`CurveData`/`RichCurve`);
  `niagara.inspect` on the module script asset returns three fields and no body. Consequence
  recorded above: the alpha-over-life shape of an effect I was retuning was unknowable, so the
  handover to the capture operator could state positions and sizes exactly and had to hedge
  opacity, and a `set_curve_keys` write has no readback to verify against — binary
  `FRichCurve` keys defeat the project's `grep -a` disk check. No workaround found; I left the
  inherited curve untouched rather than re-author it blind.
- `#2-second-encounter` `OPEN` VFX — Same gap, different system and different reason for needing the read. Retuning the AR muzzle smoke tail on `/Game/FPS/VFX/NS_Muzzle_AR` (emitter `Smoke`): the brief is "low alpha, must not out-read the flash or the sky", and the effective opacity at the two sampled instants (12 ms and 40 ms of a 220-400 ms life) is `InitializeParticle.Color.A` multiplied by the `ScaleColor.Scale Alpha` ramp — a `FloatFromCurve001` → `NiagaraDataInterfaceCurve`. Confirmed unreadable on this build against the live editor on port 27145: `niagara.inspect {includeStack:true}` on `/Game/FPS/VFX/Emitters/E_FPS_MuzzleAR_Smoke` reports the module input as `valueMode:"data", value:null`; `niagara.decompile_nir` on the same emitter (80144 chars) emits the wiring (`link 'Float from Curve 001'.Value -> 'Map Set'.'ScaleColor.Scale Alpha'`, `input 'Scale Alpha.FloatCurve001' : NiagaraDataInterfaceCurve`) and zero key data — searching the NIR text for `Keys`/`InterpMode`/`ArriveTangent`/`LookupTable` returns 0 hits, as does the same search over the inspect JSON. So NIR does not close the gap either, which is worth recording because NIR is the obvious place a reader would look next after `inspect`. Consequence: the alpha cut (0.72 → 0.42 on the base colour) had to be calibrated from a rendered frame — measuring the puff's luminance at 168/255 against sky 56 and backing out a target — rather than from the authored ramp, and the curve was left untouched rather than re-authored blind. The cheaper partial this ticket proposes (`{keys:N, range:[t0,t1], valueRange:[v0,v1]}` on a data-valued module input) would have been sufficient here as well.
- `#3-dynamic-input-curve-discoverable-and-summarised` `IN-REVIEW` developer — Partly stale, partly real. **Stale:** the proposed verb exists — `niagara.get_curve_keys` shipped with `B-niagara-set-curve-keys-unreachable-module-input-di` `#10` (same `entryId` + `inputName` addressing as `set_curve_keys`, per-channel `{channel, keys:[{time, value, interp, arriveTangent, leaveTangent, ...}]}`, `curveAsset` for the bound-asset flavour), and that ticket's `#11` verified the round trip live; `#11` also shows `FRichCurve` keys ARE greppable on disk as float32, which corrects `#1`'s "binary defeats the disk check". **Still real, for exactly the shape both encounters here hit:** the ramp was `ScaleColor.Scale Alpha` <- dynamic input `FloatFromCurve` <- its `FloatCurve` DI. `get_curve_keys` resolves any function-call node in the graph by `entryId` (`FindNode`), so the dynamic-input node is addressable — but no readback published that node's id (the `niagara.inspect` stack entry carried only the script path), and the inspect stack entry for the curve input said `valueMode:"data"` and nothing more. Fixed both in the one builder that feeds `niagara.inspect {includeStack:true}` and `niagara_stack.json` (`NiagaraDumpBuilder::BuildModuleInputsJson`): a `dynamicInput` entry carries `dynamicInputEntryId` (the placed node's guid — the `entryId` to pass), and a curve-valued `data` entry carries the cheaper-partial summary `#1`/`#2` asked for, `curve: {dataInterfaceClass, channels:[{channel, keyCount, timeRange:[t0,t1], valueRange:[v0,v1]}], curveAsset}`, built beside get_curve_keys' channel table so channel names match. Not done: `valueMode:"default"` curve inputs (un-overridden, script-default DI) still get no summary and `get_curve_keys` still refuses them `DATA_INTERFACE_NOT_FOUND` — that is `B-niagara-set-curve-keys-unreachable-module-input-di` `#11`'s residue / `B-inspect-default-module-input-has-no-effective-value`, not this ticket. Files (plugin): `Source/PinWright/Private/Handlers/Niagara/NiagaraEditTypes.h/.cpp` (`FModuleInputBindingInfo::DataInterfaceNode`, set by `ClassifyModuleInputBindings` for a DI override), `NiagaraModuleInputDataInterface.h` (declares `NiagaraModuleInputDI::BuildCurveSummaryJson`), `NiagaraCurveHandler.cpp` (defines it), `NiagaraDumpBuilder.cpp` (the two fields), `Handlers/Asset/AssetDumpCache.cpp` (`niagara_stack.json` 10 -> 11), `docs/wiki-src/niagara.md` (`niagara.inspect` moduleInputs bullets; `niagara.get_curve_keys` dynamic-input addressing paragraph), `CHANGELOG.md`; new test `Source/PinWright/Private/Tests/Niagara/TestNiagaraCurveDynamicInputReadback.cpp` — `PinWright.niagara.get_curve_keys.DynamicInputCurveDiscoverableFromInspect` (ScaleColor + FloatFromCurve on a transient system; writes 3 keys through `set_curve_keys` at the dynamic-input node, asserts the inspect entry publishes the node's id and the 3-key / [0,1] / [0.05,1] summary, then reads the keys via `get_curve_keys` using the PUBLISHED id; fails on either field being reverted). Compile-checked: `NiagaraEditTypes.cpp` with UBT `-SingleFile` (Succeeded), every other changed .cpp with clang `-fsyntax-only` on UBT's module flags (OK); not run here.
- `#4-verified-linux` `DONE` tester — Verified on Linux, UE 5.8, PinWright 7230b41d (commit 1ae8922e). run3/full passed non-skipped: `PinWright.niagara.get_curve_keys.DynamicInputCurveDiscoverableFromInspect`, plus `PinWright.niagara.set_curve_keys.ModuleInputRoundTrip` and `.AddressingFormsAreExclusive`. The proposal's `niagara.get_curve_keys` already existed. The real gap was the `ScaleColor.Scale Alpha <- FloatFromCurve <- FloatCurve` shape, and it is closed. `niagara.inspect {includeStack:true}` now publishes `dynamicInputEntryId` on the dynamic-input entry and the cheaper-partial `curve` summary on the data entry. That summary is `channels[]` with `keyCount` 3, `timeRange` [0,1] and `valueRange` [0.05,1]. `get_curve_keys` then reads the keys through the published id. Correction to `#3`, per review: `get_curve_keys` does read a `valueMode:"default"` script-default curve (`writable:false`). Only the inspect `curve` summary on default entries is missing, and that belongs to `B-inspect-default-module-input-has-no-effective-value`.
