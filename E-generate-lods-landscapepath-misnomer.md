---
id: E-generate-lods-landscapepath-misnomer
title: "asset.generate_lods single-mesh slot is named landscapePath (a self-admitted misnomer) for a StaticMesh — no assetPath alias"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [asset, generate_lods, lod, static-mesh, param-alias, docs]
---

# `asset.generate_lods` single-mesh param is `landscapePath` for a StaticMesh

`asset.generate_lods` generates an LOD chain on **StaticMesh** assets. Its
two input slots are `landscapePath` (single mesh) and `assetPaths` (array,
batch) — `AssetWorkflowHandler.cpp:684-690`. The single-mesh slot named
`landscapePath` has nothing to do with landscapes: the handler loads it as a
`UStaticMesh` (`AssetWorkflowHandler.cpp:743-745`) and the method's own
registered param description literally calls it out:

```
RPC_PARAM_OPT("landscapePath", "string",
  "Asset path to a single StaticMesh; the param name is a misnomer kept for backward compatibility.")
```

So the source already concedes the name is wrong and only retained for
backward compatibility. The problem is the misnomer is what the caller must
type: to LOD a StaticMesh the agent sends `landscapePath=/Game/.../Display_Main_B`.
That is a confusing, guess-defeating name in the asset namespace — an agent
reasoning about "LOD this static mesh" will not reach for a param spelled
`landscapePath`, and one that reads it will second-guess whether the method
even applies to static meshes.

This is the same param-name-drift class already swept in
`E-material-editor-param-name-drift` (DONE), `E-blueprint-param-name-path-vs-assetpath`
(DONE), `E-widget-asset-path-alias-drift` (DONE), `E-asset-list-path-ignored`
(DONE): the canonical asset-path slot across the codebase is `assetPath`
(singular) / `assetPaths` (plural). `asset.generate_lods` uniquely calls its
singular slot `landscapePath`, breaking that convention inside the very
namespace (`asset.*`) where `assetPath`/`assetPaths` is the norm — note the
plural slot on this same method is correctly `assetPaths`, so the singular
`landscapePath` is doubly inconsistent (singular landscape vs plural asset).

## Why this is friction (clean-outcome PROCESS finding)

The task succeeded first try with no retries (judge filed nothing,
`filed_id` empty). The friction is purely discoverability/clarity: the
caller had to use a parameter named for the wrong asset type. The auditee's
own friction note flagged it: *"the single-mesh param being named landscapePath
for a StaticMesh is a documented-but-odd misnomer."* It only worked here
because the wiki documented the misnomer; an agent relying on naming
convention (the `assetPath`/`assetPaths` pattern everywhere else in `asset.*`)
would mis-guess `assetPath` and eat a `landscapePath or assetPaths required`
error (`AssetWorkflowHandler.cpp:730`).

## What it should do

Mirror the established alias machinery (`E-material-editor-param-name-drift #4`,
the FParamSpec/`RequireAssetPath` alias set): add `assetPath` (and ideally
`meshPath`) as an alias for the singular slot so `asset.generate_lods
{assetPath: …}` works, keeping `landscapePath` for backward compat. Then
make `assetPath` the **canonical** name in the registered param spec and the
method description so discovery surfaces the convention-conforming name first;
demote `landscapePath` to a documented legacy alias. Docs angle: update
`docs/wiki-src/asset.md` so the `asset.generate_lods` entry leads with
`assetPath`/`assetPaths` and notes `landscapePath` only as a deprecated alias.

## Friction evidence (this task — asset.generate_lods DemoRoom LOD pass, 13 calls, outcome clean)

Story: set up LOD chains on DemoRoom static meshes. The single-mesh
generation call was `asset.generate_lods {landscapePath=Display_Main_B
lodCount=4}` — a StaticMesh addressed through a param named `landscapePath`.
The batch call used the convention-conforming `assetPaths=[Straight,Bend]`.
All 13 calls succeeded; no retries, no `python.execute` fallback. The
friction note: *"none — wiki clearly documented asset.generate_lods …
Only minor note: … the single-mesh param being named landscapePath for a
StaticMesh is a documented-but-odd misnomer."*

Distinct from `F-geometry-lod-settings-batch` (OPEN), which is about the
*call-count* asymmetry of `geometry.set_lod_settings` having no batch form —
a different method and a different friction. This ticket is solely the
`landscapePath`-on-a-StaticMesh naming misnomer on `asset.generate_lods`.

## History
- `#2-alias-fix` `IN-REVIEW` developer — Made `assetPath` the canonical single-mesh slot for `asset.generate_lods` and demoted the `landscapePath` misnomer to a backward-compat alias (added `meshPath` too, matching `asset.nanite_rebuild_mesh`), reusing the established FParamSpec alias machinery rather than inventing a new path. In `AssetWorkflowHandler.cpp`: added a canonical-first key list `GenerateLodsSingleMeshKeys()` = `{assetPath, meshPath, landscapePath}`, registered the spec via `ParamAliasUtils::MakeAliasParamSpec("assetPath", …, GenerateLodsSingleMeshKeys())` (so the dispatcher honors all three as known params and surfaces `assetPath` as canonical), read the single-mesh value via `Ctx.GetStringFirstOf(GenerateLodsSingleMeshKeys())` instead of the raw `landscapePath` field read, updated the handler summary + empty-slot error to lead with `assetPath`/`assetPaths` (`"assetPath (single) or assetPaths (batch) required"`), and added the `Handlers/ParamAliasUtils.h` include. No `docs/wiki-src` overlay edit was needed — `asset.generate_lods` has no hand-authored H3 section; its discovery content is auto-generated from the now-corrected ParamSpec. Regression test: `Source/PinWright/Private/Tests/Assets/TestGenerateLodsPathParamAlias.cpp` — (1) static check that the registered single-mesh spec is canonical `assetPath`, optional, with `meshPath` + `landscapePath` aliases and that `landscapePath` is no longer a standalone canonical param; (2) end-to-end dispatch of `{assetPath: <missing>}` through the real `FRpcDispatcher` asserting it is NOT rejected as `UNKNOWN_PARAMS` (alias honored) nor as the empty-slot `INVALID_ARGUMENT` (body resolved the value). Reverting the rename makes `assetPath` an unknown param, failing both layers. Files: `Source/PinWright/Private/Handlers/Asset/AssetWorkflowHandler.cpp`, `Source/PinWright/Private/Tests/Assets/TestGenerateLodsPathParamAlias.cpp`.
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `asset.generate_lods` DemoRoom LOD-pass task (13 calls, outcome clean, judge filed nothing, `filed_id` empty). PROCESS finding: `asset.generate_lods`'s single-mesh input slot is named `landscapePath` (`AssetWorkflowHandler.cpp:686`) even though it loads a `UStaticMesh` (`:743-745`); the registered param description self-admits *"the param name is a misnomer kept for backward compatibility."* The auditee used `landscapePath=Display_Main_B` to LOD a static mesh, then `assetPaths=[…]` for the batch — the singular slot violates the `assetPath`/`assetPaths` convention this `asset.*` namespace and the rest of the codebase use (sibling DONE sweeps: `E-material-editor-param-name-drift`, `E-blueprint-param-name-path-vs-assetpath`, `E-widget-asset-path-alias-drift`, `E-asset-list-path-ignored`). No errors/retries — pure naming-discoverability overhead. Proposed: add an `assetPath` alias (reuse the FParamSpec alias machinery), make `assetPath` canonical in the spec + description, demote `landscapePath` to a legacy alias, and lead with `assetPath`/`assetPaths` in `docs/wiki-src/asset.md`. Dedup: no existing `asset.generate_lods` / `landscapePath`-misnomer ticket; `F-geometry-lod-settings-batch` (OPEN) is a different method + a call-count (not naming) friction.
