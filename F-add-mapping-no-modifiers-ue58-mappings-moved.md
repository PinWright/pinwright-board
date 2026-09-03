---
id: F-add-mapping-no-modifiers-ue58-mappings-moved
title: "input.add_mapping cannot set per-mapping Modifiers/Triggers, so a WASD Axis2D bind is functionally broken; and on UE 5.8 the wiki-implied `Mappings` property is deprecated and EMPTY (data moved to DefaultKeyMappings.mappings)"
status: IN-REVIEW
severity: High
category: feature
tags: [input, enhanced-input, add_mapping, modifiers, swizzle, negate, ue5.8, deprecated-property, wiki, silent-no-op]
---

# `input.add_mapping` has no Modifiers surface, and the fallback the wiki implies writes to a dead UE 5.8 property

## Part 1 — the missing feature

`input.add_mapping` binds `(IMC, IA, key)` and nothing else. There is no parameter,
and no other verb in the `input` namespace, for the per-mapping `Modifiers` or
`Triggers` arrays on `FEnhancedActionKeyMapping`.

That is not a cosmetic gap: **it makes the single most common Enhanced Input bind
produce wrong values.** An `Axis2D` `IA_Move` bound to W/A/S/D through
`input.add_mapping` alone gives all four keys the identical value `(X=1, Y=0, Z=0)`,
because a digital key feeds the X axis and nothing redirects or negates it. The
canonical authoring (UE's own templates) needs:

| key | required modifiers |
|---|---|
| `W` | `InputModifierSwizzleAxis` (YXZ) |
| `S` | `InputModifierNegate`, then `InputModifierSwizzleAxis` |
| `A` | `InputModifierNegate` |
| `D` | none |

None of that is reachable through the namespace. Every project that binds movement
keys through PinWright ships a broken IMC unless the author notices and drops to
Python.

## Part 2 — the documented fallback is a silent no-op on UE 5.8

`Saved/PinWright/wiki/input.md` says, verbatim:

> **Authoring triggers and modifiers**
> There are no MCP verbs for an IA's `Triggers` / `Modifiers` arrays. Author them in
> the editor or use `property.set` on the IA (the same reflection route as
> `ValueType`).

Following that for the **IMC** side (`Mappings`, where the per-key modifiers live)
silently does nothing on UE 5.8, because the property it names no longer holds the
data:

```
call("python.execute", { code: "imc.get_editor_property('mappings')" })
-> DeprecationWarning: InputMappingContext: Property 'mappings' on 'InputMappingContext'
   is deprecated: Use the DefaultKeyMappings struct instead.
-> []            # empty, even though input.get_input_info reports mappingCount: 24
```

The 24 real mappings live at `DefaultKeyMappings` (a `FInputMappingContextMappingData`
struct) in its own `mappings` array:

```python
data = imc.get_editor_property('default_key_mappings')   # InputMappingContextMappingData
maps = data.get_editor_property('mappings')              # the real 24 entries
```

So a caller who reads the wiki, reads back `Mappings`, sees `[]`, and concludes the
IMC is empty is being misled by PinWright's own documentation; a caller who *writes*
`Mappings` gets `success: true` and changes nothing. `input.get_input_info` reads the
right place (`mappingCount: 24` is correct), which makes the disagreement between the
two surfaces especially confusing.

## Repro

1. `input.create_input_mapping_context {name:"IMC_FPS", path:"/Game/FPS/Player/Input"}`
2. `input.create_input_action {name:"IA_Move", path:"/Game/FPS/Player/Input"}` then
   `property.set {objectPath:".../IA_Move.IA_Move", propertyName:"ValueType", value:"Axis2D"}`
3. `input.add_mapping` x4 for `W`, `S`, `A`, `D`.
4. `input.get_input_info {assetPath:"/Game/FPS/Player/Input/IMC_FPS"}` -> `mappingCount: 24` (correct).
5. `python.execute` reading `imc.get_editor_property("mappings")` -> `[]` + DeprecationWarning.
6. In PIE the character walks forward-only on all four keys.

## Asked for

- **Feature:** an optional `modifiers` (and `triggers`) parameter on `input.add_mapping`,
  accepting modifier class names plus their properties, e.g.
  `modifiers: [{class:"InputModifierNegate"}, {class:"InputModifierSwizzleAxis", order:"YXZ"}]`.
  A `input.set_mapping_modifiers {contextPath, actionPath, key, modifiers}` verb would
  serve equally well. Without one of these the namespace cannot author a correct
  movement bind.
- **Wiki:** `input.md` must say that on UE 5.8 the IMC's key mappings live in
  `DefaultKeyMappings.mappings`, that the legacy `Mappings` property is deprecated and
  reads back empty, and that per-key modifiers are IMC-side (not IA-side) — the current
  text points only at the IA and at a property that no longer holds data.

## Workaround used

`python.execute` against `default_key_mappings`, constructing modifiers with
`unreal.new_object(unreal.InputModifierSwizzleAxis, outer=imc)` /
`unreal.InputModifierNegate`, writing the mutated struct back with
`imc.set_editor_property('default_key_mappings', data)`, then
`PinWrightPackageLibrary.mark_package_dirty` +
`EditorAssetLibrary.save_asset(only_if_is_dirty=False)`. Verified on disk: file mtime
moved and `grep -a` finds `InputModifierSwizzleAxis` and `InputModifierNegate` in the
`.uasset` bytes.

severity rationale: impact=wrong-output (every keyboard movement bind authored through
the namespace is silently incorrect) x reach=every-project-with-Enhanced-Input -> High

## Fix

The ticket was true: the handler could only create a mapping with empty per-key
configuration because it exposed only context, action, and key, while the wiki described
only action-level objects and did not identify the live IMC mapping store.
`input.add_mapping` now accepts ordered
`modifiers` and `triggers` arrays of `{class, properties}`, creates validated concrete
objects with the IMC as their outer, applies properties through
`ApplyJsonValueToProperty`, attaches them to the new `FEnhancedActionKeyMapping`, saves,
and returns the values read from the stored mapping. `input.get_input_info` now returns
the complete default mapping list, including modifier/trigger classes and editable
properties; `mappingStorage` reports `DefaultKeyMappings.Mappings` on UE 5.7+ and
`Mappings` on UE 5.3–5.6. The public `MapKey` / `GetMappings` API is deliberately retained
for both engine layouts because each installed engine version routes those accessors to
its correct backing array.

Files changed:
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Input\InputHandler.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Input\TestInputMappingModifiers.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Docs\wiki-src\input.md`

Automation test: `PinWright.input.add_mapping.PerMappingModifiersAndTriggersRoundTrip`.
It drives the registered production handlers and checks declaration, object classes,
context ownership, applied properties, add-response readback, mapping storage naming,
`input.get_input_info` readback, and invalid-class refusal before mapping mutation. Per
the worker brief, it was authored but not run.
No batch-mapping verb, action-level property surface, generated wiki output, or unrelated
existing Input tests were changed.

## History
- `#1-filed` `OPEN` reporter — Hit while building the first-person player for the FPS stream on UE 5.8 / EAContentExamples58. `input.add_mapping` bound W/A/S/D to an `Axis2D` `IA_Move` correctly as far as it goes (`input.get_input_info` -> `mappingCount: 24`), but there is no verb anywhere in the `input` namespace for the per-mapping `Modifiers` array, so all four keys resolve to `(1,0,0)` and the character can only walk forward. Reaching for the fallback `Saved/PinWright/wiki/input.md` names — `property.set` on the reflected mapping array — then hit the second half of this: on UE 5.8 `UInputMappingContext::Mappings` is deprecated and reads back **empty** while the live data sits in `DefaultKeyMappings` (an `FInputMappingContextMappingData` whose own `mappings` array holds the 24 entries). `python.execute` on `mappings` returned `[]` with `DeprecationWarning: Property 'mappings' on 'InputMappingContext' is deprecated: Use the DefaultKeyMappings struct instead`, directly contradicting `input.get_input_info`'s correct `mappingCount: 24` on the same asset in the same second. Worked around in Python against `default_key_mappings`, then verified against the file rather than an in-memory read-back: `IMC_FPS.uasset` mtime advanced and `grep -a` finds both `InputModifierSwizzleAxis` and `InputModifierNegate` in the bytes. Asked for a `modifiers`/`triggers` parameter on `input.add_mapping` (or a `set_mapping_modifiers` sibling), plus a wiki correction on `input.md` naming `DefaultKeyMappings.mappings` for UE 5.8 and stating that per-key modifiers are IMC-side rather than IA-side. Adjacent but distinct: `F-add-mapping-batch-keys` (batching, not modifiers), `E-create-input-action-valuetype-undiscoverable` (IA ValueType discovery, not IMC mapping data).
- `#2-per-mapping-objects` `IN-REVIEW` developer — Added validated ordered `{class, properties}` modifiers/triggers to `input.add_mapping`, stored-object and full IMC readback, UE 5.7+ mapping-store guidance, and regression test `PinWright.input.add_mapping.PerMappingModifiersAndTriggersRoundTrip`.
