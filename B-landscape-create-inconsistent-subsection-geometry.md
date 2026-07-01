---
id: B-landscape-create-inconsistent-subsection-geometry
title: "landscape.create silently builds an inconsistent landscape (ComponentSizeQuads != SubsectionSizeQuads * NumSubsections) when quadsPerComponent is not divisible by sectionsPerComponent"
status: IN-REVIEW
severity: High
category: bug
tags: [landscape, silent-corruption, validation, geometry]
---

# landscape.create produces inconsistent ComponentSizeQuads vs SubsectionSizeQuads*NumSubsections

`landscape.create` derives the three core landscape geometry fields independently
from the inputs instead of from a single consistent source:

`Handlers/Environment/LandscapeHandler.cpp` (the async create lambda):

```cpp
Landscape->ComponentSizeQuads  = CaptQuadsPerComponent;                       // = quadsPerComponent
Landscape->SubsectionSizeQuads = CaptQuadsPerComponent / CaptSectionsPerComponent; // INTEGER division
Landscape->NumSubsections      = CaptSectionsPerComponent;                    // = sectionsPerComponent
```

A valid UE landscape requires the invariant
`ComponentSizeQuads == SubsectionSizeQuads * NumSubsections`. When
`quadsPerComponent` is **not** an exact multiple of `sectionsPerComponent`, the
integer division silently truncates `SubsectionSizeQuads`, breaking the
invariant. The handler then sets `ComponentSizeQuads` directly to the raw input,
so the proxy is left with three mutually inconsistent fields. No validation
rejects the combination and no warning is emitted — the call returns
`success:true` and even echoes the bogus `quadsPerComponent` back to the caller.

This is the exact combination an agent reaches by taking the task at face value:
the user asked for "31 quads per component edge" and "4 sections per component",
both of which the doc text appears to permit (31 is in the documented
`7,15,31,63,127,255` list; 4 is a documented section count). The handler accepts
it, reports success, and produces a corrupt landscape.

There is a second, related defect in the same lines: `NumSubsections` is set
directly to `sectionsPerComponent`, so `sectionsPerComponent=4` yields
`NumSubsections=4`. UE only supports `NumSubsections` of 1 or 2 (a 1x1 or 2x2
subsection grid → 1 or 4 total sections). The doc says "Sections per component
(1 or 4)" but stores that value verbatim into `NumSubsections`, so the value 4
(meant to be "4 total sections" = a 2x2 grid = `NumSubsections=2`) is written as
`NumSubsections=4`, which is itself not a valid subsection count.

## What it should do

- Compute the three fields from a consistent source so the invariant always
  holds, e.g. require `quadsPerComponent % sectionsPerComponent == 0` (or
  interpret `sectionsPerComponent` as the **total** section count and map
  4 → `NumSubsections=2`), and set
  `ComponentSizeQuads = SubsectionSizeQuads * NumSubsections`.
- Reject invalid combinations with a clean error (e.g. `INVALID_ARGUMENT`)
  rather than silently truncating and reporting success.
- Validate `SubsectionSizeQuads` (not `quadsPerComponent`) against the allowed
  set `7,15,31,63,127,255`, and validate `NumSubsections` ∈ {1,2}.
- Clarify the doc: `quadsPerComponent` must equal `SubsectionSizeQuads *
  NumSubsections`, and `sectionsPerComponent` maps to a 1x1 or 2x2 subsection
  grid (NumSubsections 1 or 2 → 1 or 4 total sections), not a raw NumSubsections.

## Verbatim repro

RPC: `landscape.create`
args:
```json
{"name":"DesertBasinReplay","location":{"x":10000,"y":-5000,"z":0},
 "componentsX":4,"componentsY":4,"quadsPerComponent":31,
 "sectionsPerComponent":4,"materialPath":"/Engine/EngineMaterials/WorldGridMaterial"}
```
Response (silent success):
```json
{"success":true,"landscapePath":".../PersistentLevel.Landscape_14",
 "actorLabel":"DesertBasinReplay","componentsX":4,"componentsY":4,
 "quadsPerComponent":31,"message":"Landscape created successfully"}
```
Inspecting the created actor via `property.get` shows the broken invariant:
- `ComponentSizeQuads`  = 31
- `SubsectionSizeQuads` = 7   (31 / 4 truncated)
- `NumSubsections`      = 4

`SubsectionSizeQuads * NumSubsections = 7 * 4 = 28 ≠ 31 = ComponentSizeQuads`.
Both 31/4 truncation and the `NumSubsections=4` (only 1 or 2 are valid) are
wrong; the handler reports none of it.

## History
- `#1-initial-repro` `OPEN` reporter — Replayed `landscape.create` with the literal task inputs (quadsPerComponent=31, sectionsPerComponent=4) against a live editor. Handler returned `success:true` and echoed `quadsPerComponent:31`, but the spawned ALandscape has ComponentSizeQuads=31, SubsectionSizeQuads=7, NumSubsections=4 — violating the required invariant ComponentSizeQuads == SubsectionSizeQuads*NumSubsections (7*4=28). Root cause: LandscapeHandler.cpp derives the three fields independently (`ComponentSizeQuads = quadsPerComponent`, `SubsectionSizeQuads = quadsPerComponent / sectionsPerComponent` int-division, `NumSubsections = sectionsPerComponent`), with no divisibility check and no warning. Additionally NumSubsections is written as 4 though UE only supports 1 or 2. Confirmed reproducible and deleted the replay actor afterward.
- `#2-fix-single-source-geometry` `IN-REVIEW` developer — Fixed by deriving all three geometry fields from one consistent source and validating inputs synchronously (before the editor-world check) so the rejection depends only on the arguments. `quadsPerComponent` is now treated as the per-subsection size (`SubsectionSizeQuads`) and validated against UE's `{7,15,31,63,127,255}` set; `sectionsPerComponent` is the total section count mapped to `NumSubsections` (1→1, 4→2, plus 2→2 as an alias), validated to that set; `ComponentSizeQuads = NumSubsections * SubsectionSizeQuads` is computed from those. Non-conforming combinations (incl. the 31/4 repro) now return `INVALID_ARGUMENT` instead of silent success. The async spawn path, heightmap vertex extents, the legacy `Import()` call, and the success response were all switched to the canonical derived values (response now also reports `subsectionSizeQuads`, `numSubsections`, `componentSizeQuads`). Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Environment/LandscapeHandler.cpp` (param-doc strings updated too), `docs/wiki-src/landscape.md` (new "Geometry constraints" section). Regression test: `Source/EditorAutomationRpcGateway/Private/Tests/Core/LandscapeCreateGeometryValidationTest.cpp` (`EditorAutomationRpcGateway.landscape.create.GeometryValidation`) — asserts the 31/4 repro plus a bad quads value and a bad section count all return synchronous `INVALID_ARGUMENT`, and that the valid 63/1 default is not rejected. Fails if the single-source derivation is reverted to the independent int-division. Did not compile/run (later phase).
