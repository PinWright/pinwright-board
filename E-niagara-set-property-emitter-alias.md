---
id: E-niagara-set-property-emitter-alias
title: "niagara.set_property emitterHandle kind never wires a reflected container, so handle UPROPERTYs (bIsEnabled) are unsettable; error names the resolved kind and wiki lists no valid kinds"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [niagara, set-property, target-kind, emitter, emitter-handle, discoverability, misleading-error]
---

# `niagara.set_property` emitter-handle target exposes no reflected properties

The `"emitter"` / `"emitterHandle"` target kind resolves to a valid
`FNiagaraEmitterHandle` but is then treated as carrying **no** reflected
properties — every property set through it is rejected with
`UNSUPPORTED_TARGET`. This is wrong: `FNiagaraEmitterHandle` is a `USTRUCT()`
with reflected `UPROPERTY()` fields (notably `bool bIsEnabled`,
`NiagaraEmitterHandle.h:143-144`), so handle-level properties *should* be
settable through that kind. The actual defect is a **wiring gap**, not a
property-less handle.

In `NiagaraEditTypes.cpp` the alias map (`ParseTargetKind`) maps both
`"emitter"` and `"emitterhandle"` to `EmitterHandle` (lines 101-103).
`ResolveTarget` already populates `OutTarget.EmitterHandle` (line 954) before
the kind switch, but the `EmitterHandle` case (lines 979-982) returns success
**without** setting `ReflectedObject`/`ReflectedStruct`/`ReflectedContainer`.
By contrast the `EmitterData` case (lines 984-991) wires
`ReflectedStruct = FVersionedNiagaraEmitterData::StaticStruct()` +
`ReflectedContainer`, which is why `bLocalSpace` (a field on
`FVersionedNiagaraEmitterData`, `NiagaraEmitter.h:276`) works via
`"emitterData"`. Because the handle case wires nothing,
`ValidatePropertyPayload` falls through to the `UNSUPPORTED_TARGET` branch
(line 1186) for any handle property.

Two further ergonomic problems compound the wiring gap:

1. **The error names the resolved kind, not the typed one.** The caller passes
   `kind: "emitter"`; the rejection reports `'emitterHandle'`
   (`TargetKindToString` of the *resolved* enum, line 1186) — the typed
   spelling is preserved in `FNiagaraEditTargetSpec::KindText` (line 83,
   populated at line 151) but unused — so the message names a kind the agent
   never typed and gives no steer toward the right kind for the property.
2. **The wiki enumerates no valid target kinds.** `niagara.set_property`'s page
   documents `target` only as "Target descriptor with kind plus
   emitter/index/script/node fields as needed" — it never lists the kind
   values (`system`, `emitter`/`emitterHandle`, `emitterData`, `renderer`,
   `parameterStore`, `dataInterface`, `module`, `eventHandler`,
   `simulationStage`, ...) nor which kind owns which class of property. Nothing
   anywhere in the generated wiki enumerates them.

Note: the closed ticket `F-niagara-remove-emitter` asserts (lines 38-39, 56)
that `niagara.set_property` with `target.kind: emitterHandle` "can edit
properties on the handle (e.g. `bIsEnabled`)". That was incorrect against the
old source — `emitterHandle` resolved with no reflected container — but the fix
here makes it true: handle `UPROPERTY()` fields like `bIsEnabled` become
settable through the `emitter`/`emitterHandle` kind. Not a regression of that
ticket's remove-emitter subject.

## Repro (verbatim, replayed)

```
niagara.set_property {
  assetPath: "/Game/ExampleContent/Niagara/Simple/RendererOverrides_System.RendererOverrides_System",
  target: { kind: "emitter", emitter: "RendererOverridesEmitter" },
  propertyPath: "bIsEnabled",
  value: false
}
-> [UNSUPPORTED_TARGET] Target kind 'emitterHandle' does not expose reflected properties.
```

`bIsEnabled` is a handle `UPROPERTY()`, so it is reachable *only* through the
handle kind, never through `emitterData`. `bLocalSpace` (an `emitterData`
field) succeeds today via `kind: "emitterData"`.

## What it should do

- **Wire the `EmitterHandle` case** to the handle struct:
  `ReflectedStruct = FNiagaraEmitterHandle::StaticStruct()` +
  `ReflectedContainer = OutTarget.EmitterHandle`, so handle `UPROPERTY()`
  fields (e.g. `bIsEnabled`) are settable through `emitter`/`emitterHandle`.
  Do **not** alias `emitter` to `EmitterData` — that would make handle-only
  properties like `bIsEnabled` permanently unreachable and silently change
  which container a caller edits.
- **Make the error actionable**: echo the kind the caller actually typed
  (`KindText`) and, for the handle/emitter-data case, steer toward the sibling
  kind (`emitterData` for `bLocalSpace`-class fields, `emitter` for handle
  fields), since these two scopes are easy to confuse.
- **List the valid `target.kind` values** (and which property class each owns)
  on the `niagara.set_property` wiki page / `niagara` overlay.

**Workaround:** handle fields like `bIsEnabled` have no in-MCP workaround
through `set_property` today (the handle kind rejects them and `emitterData`
doesn't carry them); `niagara.set_stack_enabled` covers the enable/disable use
case at the module level. Emitter-data fields like `bLocalSpace` use
`target.kind: "emitterData"`.

## History
- `#1-initial-repro` `OPEN` reporter — Replayed `niagara.set_property` with `target.kind:"emitter"` (the documented-looking short spelling) against `/Game/ExampleContent/Niagara/Simple/RendererOverrides_System`; got `[UNSUPPORTED_TARGET] Target kind 'emitterHandle' does not expose reflected properties.` verbatim. Root-caused in `NiagaraEditTypes.cpp`: `ParseTargetKind` aliases `"emitter"`→`EmitterHandle`, whose `ResolveTarget` case sets no reflected container, so `ValidatePropertyPayload` returns `UNSUPPORTED_TARGET`. The correct kind `"emitterData"` succeeds but is undiscoverable — the wiki page lists no kind values and the error reports the resolved `'emitterHandle'` rather than the typed `"emitter"`. Filed as ergonomic with a quotable demonstration.
- `#2-retriage` `OPEN` triage — Low→Medium: misleading-error: emitter alias resolves to a property-less handle and the error names a kind never typed, recoverable only via source-read, niche path.
- `#3-wired-handle-struct` `IN-REVIEW` developer — Reworded first: the root cause is a wiring gap, not a property-less handle — `FNiagaraEmitterHandle` is a `USTRUCT()` with `UPROPERTY() bool bIsEnabled` (`NiagaraEmitterHandle.h:143-144`), and `bIsEnabled` is reachable *only* through the handle kind (the old body wrongly claimed it lives on `emitterData` and proposed aliasing `emitter`→`EmitterData`, which would make `bIsEnabled` permanently unreachable). Fix in `NiagaraEditTypes.cpp`: `ResolveTarget`'s `EmitterHandle` case now wires `ReflectedStruct = FNiagaraEmitterHandle::StaticStruct()` + `ReflectedContainer = OutTarget.EmitterHandle` (it already populated `OutTarget.EmitterHandle` before the switch), so handle `UPROPERTY()` fields are settable through `kind:"emitter"`/`"emitterHandle"`; `emitterData` (e.g. `bLocalSpace`) is unchanged. The write persists via the existing `ModifyResolvedTarget`/`NotifyNiagaraObjectChanged` path (System owns the handle array and is Modify()'d + dirtied). Made the `UNSUPPORTED_TARGET` error actionable: it now echoes the typed `KindText` instead of the resolved enum and steers between the emitter/emitterData sibling scopes. Enumerated all `target.kind` values (and which property class each owns) in `docs/wiki-src/niagara.md` under a new `## set_property target kinds` section. Regression test `FNiagaraEditSetPropertyEmitterHandleEnabledTest` (`PinWright.Assets.Niagara.Edit.SetProperty.EmitterHandleEnabled`) in `Tests/Assets/TestNiagaraEditHandler.cpp` builds a transient system + emitter handle and flips `bIsEnabled` via `kind:"emitter"`, asserting success and the live handle value change — it reverts to `UNSUPPORTED_TARGET`/no-mutation if the wiring is removed. This also makes the side-claim in DONE `F-niagara-remove-emitter` (#38-39) true; not a regression of its remove-emitter subject.
