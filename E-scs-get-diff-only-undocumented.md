---
id: E-scs-get-diff-only-undocumented
title: "blueprint.scs.get is silently diff-only (emits only properties differing from the component CDO) — a property set to its default value (e.g. bAutoActivate=true on AudioComponent) never appears in the readback, trapping set-then-verify; undocumented in the wiki"
status: OPEN
severity: Low
category: ergonomic
tags: [blueprint, scs, scs-get, diff-only, readback, cdo, default-value, docs, discovery]
encounters: 2
lastSeen: 2026-07-01T23:01:51+03:00
---

# `blueprint.scs.get`'s diff-only readback silently omits a property set to its component-CDO default

`blueprint.scs.get` renders each SCS node's `properties` block by **diffing the
component template against the component-class CDO and emitting only the fields
that differ** (`AddOverriddenProperties`, `PinWright_SCSHandlers.cpp:262`). A node
with no property that differs from its CDO gets **no `properties` field at all**.
This is the correct delta semantics for an "overrides" view, but two things make
it an ergonomic trap:

1. **It is undocumented.** The `docs/wiki-src/blueprint.scs.md` overlay has no
   `### blueprint.scs.get` section and never states that the readback is
   diff-only / shows overrides only. A caller who reads the tree back to
   "confirm everything is configured the way I described" reasonably expects to
   see every property they set.
2. **`blueprint.scs.set_property` reports `success:true … source:"local"` for a
   property whose value equals the CDO default**, yet that property then never
   surfaces in `blueprint.scs.get`. So the natural set-then-verify loop returns
   an empty `properties` block with no signal distinguishing "the set failed"
   from "the value equals the default and was therefore elided." The caller
   cannot confirm the very property they set through the same verb they used to
   inspect the tree.

The canonical bite is `bAutoActivate` on a `UAudioComponent`: `UAudioComponent`
**defaults `bAutoActivate` to true**, so an author who explicitly sets it true to
"make the audio start playing automatically" has set it to its default — a no-op
override that `scs.get` correctly-but-invisibly drops. The value IS reachable
via `property.get {includeDefault:true}` (`value:true, defaultSource:"class_cdo",
defaultValue:true`), but that fallback is non-obvious, and in the observed task
the agent only understood the elision after reading the SCSHandler C++ as an
explicit last resort.

A second confirmed example (replayed live, see history `#2`) is
`ForwardAxis` on a `USplineMeshComponent`: the component's CDO defaults
`ForwardAxis` to `ESplineMeshAxis::X`, so authoring a spline-mesh rail with the
natural forward axis `X` — set here through the **first-class authoring verb
`spline.create_spline_mesh_component {forwardAxis:"X"}`**, not `scs.set_property`
— produces a node whose `scs.get` `properties` block shows only `StaticMesh` and
**no `ForwardAxis`**, so the sensible default axis is invisible on read-after-write.
This shows the trap is not AudioComponent-specific and can be sprung by a
namespace's own create/configure verb, not just an explicit default-valued
`scs.set_property`.

## What it should do

Purely a docs/discoverability fix (the delta semantics themselves are correct and
intentional):

- Add a `### blueprint.scs.get` section to `docs/wiki-src/blueprint.scs.md`
  stating that the per-node `properties` block is **diff-only vs the component
  CDO** (only overridden values appear; a node with no overrides emits no
  `properties` field), and that a property set to a value **equal to its default**
  (e.g. `bAutoActivate=true` on `AudioComponent`, `bVisible=true`, etc.) will
  **not** appear even though `set_property` reported success. Route such a
  confirmation to `property.get {objectPath:<componentPath>, propertyName:<name>,
  includeDefault:true}`, which returns both the effective `value` and the
  `defaultValue`/`defaultSource` so the caller can see it matches the default.
- This mirrors the readback-routing guidance already shipped for the sibling
  `E-blueprint-get-defaults-always-empty` (route CDO-default readback to
  `property.get includeDefault`).

## Repro (replayed live against `mcp__pinwright__call`)

1. `blueprint.create {name:"BP_OracleTorchProbe", parentClass:"Actor",
   savePath:"/Game"}` → `/Game/BP_OracleTorchProbe`.
2. `blueprint.scs.add_component {blueprintPath:"/Game/BP_OracleTorchProbe",
   componentName:"CrackleAudio", componentClass:"AudioComponent"}` → success,
   `componentPath:"/Game/BP_OracleTorchProbe.BP_OracleTorchProbe_C:CrackleAudio_GEN_VARIABLE"`.
3. `blueprint.scs.set_property {blueprintPath:"/Game/BP_OracleTorchProbe",
   componentName:"CrackleAudio", propertyName:"bAutoActivate", propertyValue:true}`
   → `{"success":true,"message":"Property 'bAutoActivate' set on component
   'CrackleAudio'","source":"local","compiled":true,"saved":true,…}` (claims a
   local override was written).
4. `blueprint.scs.get {blueprintPath:"/Game/BP_OracleTorchProbe"}` →
   `{"success":true,"components":[{"name":"CrackleAudio","class":"AudioComponent",
   "source":"scs","transform":{…},"child_count":0}],"count":1,…}` — the
   `CrackleAudio` node has **no `properties` field at all**; `bAutoActivate` is
   absent despite the successful set in step 3.
5. Control proving the value did take and equals the default:
   `property.get {objectPath:"…:CrackleAudio_GEN_VARIABLE",
   propertyName:"bAutoActivate", includeDefault:true}` →
   `{"propertyName":"bAutoActivate","value":true,"defaultSource":"class_cdo",
   "defaultValue":true,…}`.

So the write succeeded and the effective value is `true`, but because it equals
the component CDO default the diff-only `scs.get` elides it, and nothing in the
wiki warns of this.

## Guilty source

`Plugins/PinWright/Source/PinWright/Private/PinWright_SCSHandlers.cpp:262`
(inside `AddOverriddenProperties`, lines 226-270):
```cpp
if (!Property->Identical_InContainer(Component, BaseContainer))
{
  TSharedPtr<FJsonValue> Value = ExportPropertyToJsonValue(Component, Property);
  if (Value.IsValid())
  {
    PropsObj->SetField(Property->GetName(), Value);
  }
}
```
A property whose template value is `Identical_InContainer` to the diff base
(here the component-class CDO — `bAutoActivate==true==default`) is skipped, so it
never enters the emitted `properties` object. Correct delta behavior; the gap is
that it is undocumented and traps a set-then-verify of a default-valued property.

severity rationale: impact=docs/discoverability (value fully reachable via
`property.get`; delta semantics are correct) × reach=rare (the trap only bites
when a caller sets a property to a value equal to its CDO default) -> Low.

## Distinct from

- `E-blueprint-get-defaults-always-empty` (IN-REVIEW) — same "route CDO-default
  readback to `property.get includeDefault`" remedy, but on `blueprint.get`'s
  advertised-but-empty `defaults{}` field, not `blueprint.scs.get`'s diff-only
  component-property readback.
- `B-component-diff-vs-class-cdo` (DONE) / `F-dump-ich-overrides` (DONE) — those
  fixed the diff *base* (parent template vs class CDO, ICH overrides) so
  inherited values aren't reported as local overrides; this ticket is not about
  the diff base being wrong but about the diff-only contract being undocumented
  so a default-valued set is invisible.
- `E-scs-get-no-limit-spills` (OPEN) — response-size/projection gap on the same
  verb; orthogonal to the diff-only-omission docs gap here.

## History
- `#1-initial-repro` `OPEN` reporter — Filed from a REALISM struggle audit (build a `BP_WallTorch` actor BP: sconce mesh root, torch staff, warm point light, looping crackle `AudioComponent` with `bAutoActivate=true` for auto-play; task completed, outcome done). Friction note verbatim: *"blueprint.scs.get is diff-only (reports only properties differing from the component CDO), so CrackleAudio's bAutoActivate=true does not appear because UAudioComponent already defaults bAutoActivate to true — the very 'auto-play' property the user asked to set is unverifiable through scs.get. I fell back to property.get (includeDefault) to confirm it, and had to read the plugin SCSHandler C++ (last resort) to understand the diff-only readback that hides default-valued set properties."* Replayed live against `mcp__pinwright__call` (repro above): `blueprint.scs.set_property bAutoActivate=true` → `success:true, source:"local"`; `blueprint.scs.get` → CrackleAudio node with **no `properties` field**; `property.get {includeDefault:true}` → `value:true, defaultSource:"class_cdo", defaultValue:true` (confirms the value took and equals the default). Guilty line `PinWright_SCSHandlers.cpp:262` (`AddOverriddenProperties` skips `Identical_InContainer` properties). Docs-only fix: add a `### blueprint.scs.get` note in `docs/wiki-src/blueprint.scs.md` that the readback is diff-only vs the component CDO and default-valued sets won't appear — verify those via `property.get includeDefault`. Dedup: ripgrep across OPEN/DONE/WONTFIX — no existing ticket documents scs.get's diff-only contract or the default-valued-set elision; distinct from `E-blueprint-get-defaults-always-empty` (blueprint.get defaults field), `B-component-diff-vs-class-cdo`/`F-dump-ich-overrides` (diff-base correctness), and `E-scs-get-no-limit-spills` (response size). Severity Low (docs/discoverability; value reachable via property.get; rare-path trigger).
- `#2-additional-splinemesh-forwardaxis` `OPEN` reporter — Additional evidence (broader scope + a new trigger). Seed `spline.create_spline_mesh_component` (SEED mode); realism task: author a `BP_FenceRail` Actor with a spline-mesh rail component set up with an existing mesh and a sensible forward axis, then confirm the rail's mesh/material/axis on both the saved BP and a placed instance. Task completed (outcome done); the sole friction was this same diff-only readback trap, now on a **different component/property family** and sprung by a **first-class authoring verb** rather than an explicit default-valued `scs.set_property`. `USplineMeshComponent` defaults `ForwardAxis` to `ESplineMeshAxis::X`, so a rail authored with the natural forward axis `X` sets it to its CDO default; `blueprint.scs.get` then elides it, and the agent had to re-set `ForwardAxis=Y` (a non-default) to positively confirm the axis through the same readback verb. Replayed live against `mcp__pinwright__call` at HEAD: (1) `blueprint.create {name:"BP_OracleSplineAxisProbe", parentClass:"Actor", savePath:"/Game"}` → `/Game/BP_OracleSplineAxisProbe`. (2) `spline.create_spline_mesh_component {blueprintPath:"/Game/BP_OracleSplineAxisProbe", componentName:"RailMesh", meshPath:"/Engine/BasicShapes/Cube", forwardAxis:"X", save:true}` → `component_added`. (3) `blueprint.scs.get {blueprintPath:"/Game/BP_OracleSplineAxisProbe"}` → `RailMesh` (SplineMeshComponent) with `"properties":{"StaticMesh":"/Engine/BasicShapes/Cube.Cube"}` — **`ForwardAxis` absent** despite being explicitly passed as `X`, because `X` equals the component CDO default. (4) Control: `blueprint.scs.set_property {…, propertyName:"ForwardAxis", propertyValue:"Y"}` → `success:true, source:"local"`; re-reading `blueprint.scs.get` now returns `"properties":{"ForwardAxis":"Y","StaticMesh":"…"}` — proving the field is emitted only once it diverges from the CDO default. Same guilty line `PinWright_SCSHandlers.cpp:262` (`AddOverriddenProperties` skips `Identical_InContainer` properties); same docs-only remedy (note the diff-only contract on `docs/wiki-src/blueprint.scs.md` and route default-valued confirmation to `property.get includeDefault`). Bumped encounters 1→2. (Secondary, unfiled friction from the same task: `actor.get_components` on the Blueprint asset path returns an empty CDO — SCS templates aren't instanced there — and an `asset.search *pipe*` response spilled to a file; both already have coverage on the board and were not the primary finding.)
