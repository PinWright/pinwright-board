---
id: E-property-get-deprecated-field-silent-null
title: "property.get on a migrated top-level field (e.g. material BaseColor) silently resolves the *_DEPRECATED shadow and returns null with no hint the live data moved to a subobject"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [property-get, material, editoronlydata, deprecated-field, readback, docs]
---

# Intuitive top-level property path resolves to a deprecated shadow and reads null silently

When a UE field has been migrated out of its owner object into a subobject —
the canonical case being `UMaterial`'s input pins (`BaseColor`, `Opacity`,
`Roughness`, …) which moved to `UMaterialEditorOnlyData` in UE 5.5+, leaving
`BaseColor_DEPRECATED` etc. as inert shadows on `UMaterial` — `property.get`
against the **intuitive top-level path** resolves the deprecated shadow,
serializes it, and returns a benign-looking empty value (`Expression: null`)
with **no signal** that the field is deprecated or that the live data now lives
on a subobject. The caller reads "null" and reasonably concludes the field is
unset/unwired, when in fact it is correctly populated one path segment away at
`…EditorOnlyData.BaseColor`.

There is no discovery affordance for the correct nested path: `property.get`
neither flags the resolved property as `Deprecated` nor points at the
`EditorOnlyData` subobject, and a naive subobject probe at
`…M_FrostedGlass:MaterialEditorOnlyData` returns `[OBJECT_NOT_FOUND]` (the
subobject is reached via the `EditorOnlyData` *property* on the material, not as
a colon-suffixed inner-object path). The agent only recovered by reading the
engine `Material.h` header to learn the field had moved and to find the
`EditorOnlyData.` prefix.

## Why it's process friction (clean outcome, misuse-then-correct detour)

The task completed cleanly, but the "confirm the pins are wired" readback turned
into a misuse-then-correct chain on `property.get`:

- 3× `property.get` on top-level `BaseColor` / `Opacity` / `Roughness` →
  each returned the `*_DEPRECATED` shadow as `Expression: null` (looks unwired).
- 1× `property.get` on the colon-suffixed subobject path
  `/Game/ArchViz/Materials/M_FrostedGlass.M_FrostedGlass:MaterialEditorOnlyData`
  → `[OBJECT_NOT_FOUND]` (wrong way to reach the subobject).
- read engine `Material.h` to discover the migration + the `EditorOnlyData.`
  property prefix.
- 3× `property.get` on `EditorOnlyData.BaseColor` / `.Opacity` / `.Roughness`
  → finally the live, populated values.

So 4 of 7 `property.get` calls were dead-ends driven purely by the top-level
path silently aliasing a deprecated field, plus an out-of-band engine-source
read. Friction note verbatim: *"The first property.get on top-level
BaseColor/Opacity/Roughness returned Expression:null (those are the *_DEPRECATED
fields); in UE5.7 the real inputs live on UMaterialEditorOnlyData, so I had to
read engine Material.h and use the nested path EditorOnlyData.BaseColor to
confirm wiring."*

This is distinct from the judge-filed material-readback ticket
`E-material-main-output-no-node-readback` (which is scoped to "is the *main
output* wired?" and proposes making the Main node inspectable + a
`material.authoring.md` docs note). This ticket is about `property.get`'s own
ergonomics for **any** EditorOnlyData-migrated field — the silent
deprecated-shadow resolution and the missing nested-path discoverability — and
it is broader than materials (any owner→subobject field migration hits it). It
is also distinct from `B-property-path-silent-world-fallback` (unresolvable
level-path tail → World) and `B-property-list-hides-reflected-props` (filtered
reflected props): those are different silent-resolution paths.

## What it should do

Pick downstream — any of these reduces the dead-end chain:

- **Flag deprecated resolution.** When `property.get` resolves a `FProperty`
  carrying `CPF_Deprecated` (or a name ending `_DEPRECATED`), include a
  `deprecated: true` marker in the response and, where derivable, a
  `seeAlso`/`movedTo` hint (e.g. point materials' `BaseColor` at
  `EditorOnlyData.BaseColor`). At minimum, don't return a deprecated shadow as
  if it were the live field with no caveat.
- **Document the EditorOnlyData hop (minimum, immediately).** In the
  `property.get` overlay (`docs/wiki-src/property.md`, the `### property.get`
  area around the optional-params block ~L72) add a short note: for UE 5.5+
  materials the input pins (`BaseColor`/`EmissiveColor`/`Opacity`/`Roughness`/…)
  live on `UMaterialEditorOnlyData`; read them via the nested
  `EditorOnlyData.<Pin>` path, not the top-level name (which resolves the inert
  `*_DEPRECATED` shadow and reads `Expression: null`). Note the subobject is
  reached through the `EditorOnlyData` property, not a colon-suffixed
  `:MaterialEditorOnlyData` inner-object path. Cross-link to
  `E-material-main-output-no-node-readback`'s recommendation to use
  `material.decompile_mgir` for authoritative output-wiring readback.

**Workaround:** prefix material input-pin reads with `EditorOnlyData.`
(`property.get EditorOnlyData.BaseColor`), or skip `property.get` for output
wiring entirely and use `material.decompile_mgir`.

Filed E-/`docs` — the result was correct and recoverable; the gap is the silent
deprecated-shadow resolution + missing nested-path discoverability in
`property.get` and its wiki.

## History
- `#1-initial-audit` `OPEN` reporter — Surfaced in a clean material.authoring.set_blend_mode task (focus material.authoring.set_blend_mode, outcome clean) authoring /Game/ArchViz/Materials/M_FrostedGlass. The success-check readback for the wired BaseColor/Opacity/Roughness pins ran 3× `property.get` on the top-level pin names (each returned the `*_DEPRECATED` shadow as `Expression:null`, looking unwired), then 1× `property.get` on the colon-suffixed `…:MaterialEditorOnlyData` subobject path (`[OBJECT_NOT_FOUND]`), required an out-of-band read of engine `Material.h` to learn the fields moved to `UMaterialEditorOnlyData` and that the access path is the `EditorOnlyData.` property prefix, then 3× `property.get` on `EditorOnlyData.BaseColor/.Opacity/.Roughness` finally returned live values — 4 of 7 property.get calls dead-ended on the silent top-level→deprecated aliasing. Distinct PROCESS angle from the judge-filed E-material-main-output-no-node-readback (scoped to main-output node readback + material.authoring.md docs): this is `property.get`'s own ergonomics for EditorOnlyData-migrated fields (silent deprecated-shadow resolution + no nested-path hint), broader than materials. Propose: flag `CPF_Deprecated`/`_DEPRECATED` resolutions with a `deprecated`/`movedTo` marker; minimum, document the `EditorOnlyData.<Pin>` hop in docs/wiki-src/property.md (~L72). Low severity — recoverable, but the silent null read is actively misleading (looks unset when correctly populated on the subobject).
- `#2-retriage` `OPEN` triage — Low→Medium: top-level path silently aliases a *_DEPRECATED shadow and reads null, a silent-wrong readback that looks unset, on a common property.get verify path bumped down for the niche EditorOnlyData case.
- `#3-fix` `IN-REVIEW` developer — Fixed the silent deprecated-shadow read in `property.get`. Root cause: UHT strips the `_DEPRECATED` suffix and registers the shadow under the bare name + `CPF_Deprecated`, so a top-level `FindPropertyByName("BaseColor")` resolves the inert shadow and serializes it as `Expression: null` with no signal (`includeMetadata` defaults off, so even the flags block was suppressed). Added `AddDeprecatedResolutionHint()` in `Source/PinWright/Private/Handlers/Utility/UtilityPropertyHandler.cpp`, called UNCONDITIONALLY in the generic `property.get` path right after value export: it emits `deprecated: true` on any `CPF_Deprecated` resolution and, when the root exposes an `EditorOnlyData` object property whose subobject class declares a same-named live property (the UMaterial→UMaterialEditorOnlyData migration), a `movedTo: "EditorOnlyData.<Pin>"` hint. Also added `deprecated` to the `flags{}` sub-object in `AddPropertyMetadataFields`. Docs: added a `*_DEPRECATED`/`EditorOnlyData.<Pin>` note to the `property.get` section of `docs/wiki-src/property.md` (covers the colon-suffix `[OBJECT_NOT_FOUND]` trap and cross-links `material.decompile_mgir`). Regression test `Source/PinWright/Private/Tests/Utility/TestPropertyGetDeprecatedShadow.cpp` (`PinWright.property.get.DeprecatedShadowFlaggedWithMovedToHint`): builds a transient UMaterial, asserts top-level `BaseColor` carries `CPF_Deprecated`, then calls the production `property.get` handler with NO `includeMetadata` and asserts `deprecated:true` + `movedTo=="EditorOnlyData.BaseColor"`; a live `BlendMode` control asserts the marker is not attached spuriously. Reverting the helper drops both fields and the test fails.
