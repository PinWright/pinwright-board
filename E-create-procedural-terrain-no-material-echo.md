---
id: E-create-procedural-terrain-no-material-echo
title: "environment.build.create_procedural_terrain applies the material but never echoes it in the response — confirming the material landed costs a separate actor.describe readback"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [environment, terrain, procedural-mesh, material, response-shape, readback, round-trip, silent-swallow]
---

# `create_procedural_terrain` applies the `material` it was given but drops it from the success response, forcing an extra `actor.describe` to confirm the material took

`environment.build.create_procedural_terrain` accepts a `material` asset path,
loads it, and calls `ProcMesh->SetMaterial(0, Material)` — but its success
response echoes none of that. A caller who passed `material=...` and wants to
confirm it landed on the terrain's `ProceduralMeshComponent` has to fire a
separate `actor.describe` readback, because the create response carries no
`material` field at all.

This is the same "the mutator already holds the answer but doesn't carry it, so
a working readback verb gets spammed" shape as `E-geometry-deformer-echo-mesh-counts`
(OPEN) and `E-niagara-modify-parameter-no-override-readback` (OPEN) — here the
omitted echo is the **applied material**, not vertex/triangle counts.

## Source (handler confirmed)

`Source/EditorAutomationRpcGateway/Private/Handlers/Environment/EnvironmentHandler.cpp`,
`environment.build.create_procedural_terrain` (handler ~:1006; line numbers below
are approximate and drift with the file):

- Material *is* applied:
  ```cpp
  if (Payload->TryGetStringField(TEXT("material"), MaterialPath) && !MaterialPath.IsEmpty())
  {
      UMaterialInterface* Material = LoadObject<UMaterialInterface>(nullptr, *MaterialPath);
      if (Material)
      {
          ProcMesh->SetMaterial(0, Material);
      }
  }
  ```
- But the success response sets only
  `{actorName, actorPath, vertices, triangles, sizeX, sizeY, subdivisions}` plus
  whatever `AddActorVerification(Resp, TerrainActor)` adds — **no `material`
  field**. (Note on the readback fields: the handler first writes `actorName` =
  `TerrainActor->GetName()`, but `AddActorVerification` runs *after* and
  **overwrites** `actorName` with `Actor->GetActorLabel()` and sets
  `actorClass` = `Actor->GetClass()->GetName()` — `AssetUtils.cpp:939,942` — so the
  final `actorName` is the actor **label**, not the internal object name. The
  terrain is a plain `AActor` carrying a `UProceduralMeshComponent`, so `actorClass`
  *does* read back as the bare `"Actor"` — it just can't be more specific because
  the actor genuinely is an untyped `AActor`. That field already exists; the real
  gap is purely the missing **material** echo.)

Two secondary smells on the same branch, worth noting for the fix:
- **Silent material-load swallow:** if `LoadObject` returns null (bad/missing
  material path) the `if (Material)` simply skips `SetMaterial` — no error, no
  warning field. The call still returns `success:true` with no material applied
  and nothing in the response distinguishing "material applied" from "material
  path didn't resolve." (In this task the path was valid, so this stayed latent —
  but it means the missing echo also hides a real silent-failure mode.)

## Friction evidence (canyon-overlook greybox task, outcome `done`)

The story (step 2) asked to apply `M_ForestRock` to the terrain and (step 8) to
"read back the terrain actor so I can confirm the greybox landed." Because the
create response omits the material, the agent inserted a dedicated
`actor.describe CanyonOverlookTerrain (confirm material)` call right after the
create — a readback that a `material` echo on the create response would have
eliminated. Friction note (verbatim): *"create_procedural_terrain's response
omits any material field and reports actorName/actorClass as bare 'Actor', so
material confirmation required a separate actor.describe readback rather than
being in the create response."* (Correction to the verbatim note: the response's
`actorName` is the actor **label**, not `"Actor"` — only `actorClass` reads back
as the bare `"Actor"`, and that field already exists; the load-bearing gap is the
missing `material` echo.) All calls succeeded; this is pure PROCESS round-trip
overhead, not an outcome bug (the seed method itself worked and the material was
correctly applied on the ProceduralMeshComponent).

## What it should do

Echo what the handler already did, in the create response:
- Add a `material` field reporting the applied material's path (and, ideally, a
  `materialApplied` bool) so "did the material land?" is answerable in the same
  call — mirroring how `E-geometry-deformer-echo-mesh-counts` argues the
  topology counts belong inline on the deform response.
- When a non-empty `material` path fails to resolve (`LoadObject` null), don't
  swallow it silently — surface a `warning` field (or a soft error) so the
  caller learns the material was skipped instead of seeing a clean
  `success:true` with no material.

Cheap: the handler already holds both the requested path and the loaded
`Material` pointer at the point it returns.

## Not a duplicate of

- `B-export-snapshot-empty-stub` (judge-filed this task) — `export_snapshot`
  empty-body bug; unrelated method.
- `E-geometry-deformer-echo-mesh-counts` / `E-niagara-modify-parameter-no-override-readback`
  — same "echo the answer to avoid a readback" family, different methods; neither
  touches `environment.build.create_procedural_terrain` or the material echo.
- `E-environment-build-create-no-name-param` (this audit's sibling ticket) — that
  is the no-name-slot friction on `create_sky_sphere`/`create_fog_volume`; this is
  the material echo gap on `create_procedural_terrain` (which *does* take a name).

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the canyon-overlook
  greybox task (seed `environment.build.create_procedural_terrain`; outcome
  `done`; judge filed `B-export-snapshot-empty-stub` for the snapshot bug).
  Distinct PROCESS angle on the seed method itself: it applies the `material`
  (`EnvironmentHandler.cpp`, `ProcMesh->SetMaterial(0, Material)`) but
  the success response omits any `material` field, so confirming the material
  landed forced a separate `actor.describe` readback. Same echo-the-answer family
  as `E-geometry-deformer-echo-mesh-counts` (OPEN). Also noted: the `if (Material)`
  branch silently swallows a failed material load (no error/warning), so the
  missing echo doubles as a hidden silent-failure mode. Fix: echo the applied
  material path (+`materialApplied`/warning on load failure) in the create
  response. Workaround: `actor.describe` the terrain actor after create to read
  the material off its ProceduralMeshComponent. (Reworded in `#2`: the original
  "reports the actor as the bare `Actor` class" applies only to `actorClass`,
  which already exists; the response `actorName` is the actor label, not the
  internal object name — the genuine gap is the missing `material` echo.)
- `#2-create-procedural-terrain-material-echo` `IN-REVIEW` developer — Reworded the
  ticket to drop the incorrect "actorName is the internal object name / actor reports
  as bare 'Actor'" framing (`AddActorVerification` overwrites `actorName` with
  `GetActorLabel()` and `actorClass` already carries `"Actor"`; `AssetUtils.cpp:939,942`)
  — the load-bearing gap is purely the missing `material` echo. Implemented the
  root-cause fix in `EnvironmentHandler.cpp` `create_procedural_terrain`: the
  apply-material branch now records `bMaterialApplied`/the requested path/a load-failure
  warning, and the success response echoes `materialApplied` (always present) +
  `material` (the requested path, when one was given) + `materialWarning` (when a
  non-empty path fails `LoadObject`), so "did the material land?" is answerable in the
  create response and a failed load is no longer swallowed silently. Regression test
  `FEnvironmentCreateProceduralTerrainEchoesMaterialTest`
  (`Tests/World/TestEnvironmentHandlers.cpp`,
  `environment.build.create_procedural_terrain.EchoesAppliedMaterial`) drives the real
  handler via `InvokeHandlerWithCapture` for three cases: valid material
  (`/Engine/EngineMaterials/WorldGridMaterial`) → `materialApplied=true` + echoed
  `material` + no warning; bogus path → `materialApplied=false` + a `materialWarning`;
  no material → `materialApplied=false` + no `material` field. Fails if any echo is
  reverted. Files: `EnvironmentHandler.cpp`, `TestEnvironmentHandlers.cpp`.
