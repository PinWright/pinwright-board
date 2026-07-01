---
id: B-get-dependencies-object-path-empty
title: "asset.get_dependencies / asset.get_dependencies_classified silently return empty for the object-path form (/Game/.../SM_Gear.SM_Gear) that sibling asset.get / asset.get_metadata accept"
status: IN-REVIEW
severity: Medium
category: bug
tags: [asset, get-dependencies, get-dependencies-classified, object-path, package-name, assetregistry, fname, silent-empty]
---

# `asset.get_dependencies` returns `{"dependencies":[]}` for an object-path `assetPath` that its own sibling verbs resolve fine

`asset.get_dependencies` reads `assetPath` and passes the raw caller string
straight to `AssetRegistry::GetDependencies(FName(*AssetPath), ...)`
(`AssetQueryHandler.cpp:53`, and the recursive branch seeds the queue the same
way at `:36`). `GetDependencies` is keyed on the **package** FName. When the
caller supplies the standard **object-path** form — `/Game/.../SM_Gear.SM_Gear`
(package + `.AssetName` suffix) — the FName has no matching package node in the
registry, so the lookup returns nothing and the handler reports
`{"dependencies":[]}` with `isError:false`. There is no normalization of the
`.Object` suffix to a package name before the lookup.

This is a **silent wrong result**, not a clean error: the asset genuinely has
hard dependencies (the same call with the package-name form returns them), so
an agent that passed the object path is handed a plausible-looking empty list
and no signal that it gave the wrong path shape. `asset.get_dependencies_classified`
(registered at `AssetMetadataHandler.cpp:285`, buggy lookup at
`AssetMetadataHandler.cpp:361` — same `asset.*` namespace, different file) shares
the identical package-FName lookup and the identical silent-empty behavior on the
object path.

The trap is sharp because **the sibling read verbs in the same `asset.*`
namespace accept the object-path form**:
- `asset.get {assetPath:"/Game/.../SM_Gear.SM_Gear"}` → resolves, returns the
  StaticMesh summary (its own wiki even documents the package form, but the
  handler accepts both).
- `asset.get_metadata {assetPath:"/Game/.../SM_Gear.SM_Gear"}` → resolves,
  returns full registry tags + custom metadata; its `StaticMaterials` tag even
  lists `M_ReflectionDemo_Metallic_3` — i.e. the asset DOES have a material
  hard-dep, proving the `get_dependencies` empty answer for the same path is
  wrong, not legitimately empty.

So an agent that just did `asset.get` / `asset.get_metadata` on the object path
(the natural "read it before I act on it" step) and then reuses that exact same
path string for `asset.get_dependencies` gets a silently empty, wrong answer.
The wiki page does not warn that this one verb requires the suffix stripped.

This is distinct from the two existing OPEN reference-direction tickets:
- `E-asset-dependencies-references-inverted` — the *direction* of
  `asset.dependencies` / `asset.references` names vs behavior, and a wrong
  `asset.get_dependencies` cross-ref. That ticket is about which way the arrow
  points; this one is about the **input path form being silently mishandled**.
- `E-asset-path-vs-assetpath-list-drift` — the param **name** drift (`path` vs
  `assetPath`) across asset verbs. This ticket is the same param name
  (`assetPath`), a different axis: the **value form** (object-path vs
  package-name) accepted within that one param.

## Repro (verbatim, replay-confirmed against mcp__editor-automation__call)

Asset: `/Game/ExampleContent/Blueprints/Meshes/SM_Gear` (a StaticMesh in the
Content Examples project).

1. `asset.get_dependencies {assetPath:"/Game/ExampleContent/Blueprints/Meshes/SM_Gear.SM_Gear"}`
   → `{"dependencies":[]}`   (object-path form — silently empty, isError:false)
2. `asset.get_dependencies {assetPath:"/Game/ExampleContent/Blueprints/Meshes/SM_Gear"}`
   → `{"dependencies":["/Script/NavigationSystem","/Game/ExampleContent/Blueprints/Materials/M_ReflectionDemo_Metallic_3"]}`
   (package-name form — real hard deps)
3. `asset.get_dependencies_classified {assetPath:"/Game/ExampleContent/Blueprints/Meshes/SM_Gear.SM_Gear", mode:"all"}`
   → `{"success":true,"dependencies":[],"classifiedDependencies":[],...,"dependencyCount":0,...}`
   (same object-path form — same silent-empty)
4. Sibling verbs accept the SAME object path:
   - `asset.get {assetPath:"/Game/ExampleContent/Blueprints/Meshes/SM_Gear.SM_Gear"}`
     → `{"success":true,"result":{"name":"SM_Gear","class":"/Script/Engine.StaticMesh",...}}`
   - `asset.get_metadata {assetPath:"/Game/ExampleContent/Blueprints/Meshes/SM_Gear.SM_Gear"}`
     → `{"success":true,...,"tags":{...,"StaticMaterials":"M_ReflectionDemo_Metallic_3,'/Game/.../M_ReflectionDemo_Metallic_3.M_ReflectionDemo_Metallic_3'>"},...}`
     (proves the material hard-dep exists, so step 1's `[]` is wrong)

## What it should do

Normalize `assetPath` to a package name before the registry lookup in both
`asset.get_dependencies` (non-recursive at `AssetQueryHandler.cpp:53`, recursive
seed at `:36`) and `asset.get_dependencies_classified` (registered at
`AssetMetadataHandler.cpp:285`, lookup at `AssetMetadataHandler.cpp:361`): strip
the `.AssetName` object suffix (the part after the last `.` when a
package-relative path carries one), e.g. via the existing `NormalizeAssetPath`
helper (`AssetUtils.cpp:44`, which strips the suffix through
`FPackageName::ObjectPathToPackageName` and validates with
`IsValidLongPackageName`) or `FSoftObjectPath(...).GetLongPackageName()`, so
callers can pass either form, exactly like the sibling `asset.get` /
`asset.get_metadata` verbs already do. (Note: `Ctx.RequireAssetPath` does NOT
strip the object suffix — it only sanitizes slashes and rejects traversal — so it
cannot be the normalizer here.) Either form
should produce identical dependency lists. (Belt-and-suspenders alternative if
ambiguous-input policy forbids silent normalization: return a clean
`ASSET_NOT_FOUND` / `INVALID_PATH_FORM` error when the package node is missing,
instead of a success with an empty array — never a silent wrong-empty.)

## History
- `#1-initial-repro` `OPEN` reporter — Found during a content-organization
  tagging task (seed `asset.get`; this finding is about the neighbor verb
  `asset.get_dependencies`, not the seed). Replay-confirmed against
  `mcp__editor-automation__call`: object-path `/Game/.../SM_Gear.SM_Gear` →
  `{"dependencies":[]}`; package-name `/Game/.../SM_Gear` → the real hard deps
  `["/Script/NavigationSystem","/Game/.../M_ReflectionDemo_Metallic_3"]`.
  `asset.get_dependencies_classified` exhibits the identical silent-empty on the
  object path. Confirmed sibling verbs `asset.get` and `asset.get_metadata`
  resolve the SAME object path fine (the latter's `StaticMaterials` tag lists the
  material dep, proving the empty list is wrong, not legitimately empty). Root
  cause in source: `AssetQueryHandler.cpp:53` (and recursive seed `:36`) pass
  `FName(*AssetPath)` — the raw caller string, object suffix and all — straight to
  `AssetRegistry::GetDependencies`, which is keyed on the package FName; the
  `.Object` suffix yields no node. No suffix normalization anywhere in the handler.
  Distinct from the two OPEN reference tickets (`E-asset-dependencies-references-inverted`
  = arrow direction; `E-asset-path-vs-assetpath-list-drift` = param-name drift);
  this is the path-VALUE-form being silently mishandled within the `assetPath`
  param, producing a wrong-empty success rather than an error. Fix: normalize the
  object-path suffix to a package name before the registry lookup in both
  `asset.get_dependencies` and `asset.get_dependencies_classified` (or error
  instead of returning empty).
- `#2-additional-blueprint-actors` `OPEN` reporter — Additional evidence
  (independent replay, different assets — Blueprint actors instead of the
  `SM_Gear` StaticMesh). During a content-health audit of
  `/Game/Global/Interactable`, the object-path form silently returned empty for
  both spot-checked actor BPs while the package-name form returned the real hard
  deps, confirming the same `.AssetName`-suffix-not-normalized root cause on
  Blueprints:
  - `asset.get_dependencies {assetPath:"/Game/Global/Interactable/BP_DemoTrigger.BP_DemoTrigger"}`
    → `{"dependencies":[]}`   (object-path — silent wrong-empty, isError:false)
  - `asset.get_dependencies {assetPath:"/Game/Global/Interactable/BP_DemoTrigger"}`
    → `{"dependencies":["/Script/NavigationSystem","/Engine/EditorBlueprintResources/StandardMacros","/Game/Global/Interactable/BPInterface_Button","/Game/Global/Interactable/BPI_Interface","/Game/Global/Materials/M_Button_Inst","/Game/Global/DemoRoom/Meshes/Display_Button"]}`   (package-name — 6 real hard deps)
  - `asset.get_dependencies {assetPath:"/Game/Global/Interactable/BP_AnimDemoTrigger.BP_AnimDemoTrigger"}`
    → `{"dependencies":[]}`; package-name form →
    `["/Script/NavigationSystem","/Game/Global/Interactable/BPInterface_Button","/Game/Global/DemoRoom/Meshes/Button","/Game/Global/Interactable/BPI_Interface","/Game/Global/Materials/M_Button_AnimDemo"]}`   (5 real hard deps)
  - `asset.get_dependencies_classified {assetPath:"/Game/Global/Interactable/BP_DemoTrigger.BP_DemoTrigger"}`
    → `{"success":true,"dependencies":[],"classifiedDependencies":[],"dependencyCount":0,...}`   (same object-path silent-empty)
  Cross-checks proving the empty answer is wrong, not legitimately empty: the
  asset's own dump sidecars from `asset.dump_folder` show the refs
  (`scs.json` StaticMesh `/Game/Global/DemoRoom/Meshes/Display_Button` +
  `bpir.txt` ConstructionScript `CreateDynamicMaterialInstance(... /Game/Global/Materials/M_Button_Inst ...)` and `SKEL_BPInterface_Button::CallTriggerActor`),
  and `asset.references {assetPath:"/Game/Global/Interactable/BP_DemoTrigger.BP_DemoTrigger"}`
  (object-path form) returns all 6 outbound deps — i.e. the sibling reference verb
  tolerates the object path while `get_dependencies` does not. Real-harm note: the
  audited attempt agent passed the object-path form (the natural reuse of the path
  it had just run `asset.get` on) and self-reported the `[]` as "matching the hard
  refs in the dump sidecars" — i.e. it was actively misled into certifying a
  silently-wrong empty dependency list. Replay-confirmed against
  `mcp__editor-automation__call` (seed `asset.dump_folder`; culprit
  `asset.get_dependencies`).
- `#4-third-verb-get-asset-graph-uncovered` `OPEN` reporter — Additional evidence,
  same root cause on a **THIRD `asset.*` verb the `#3` fix does NOT touch**:
  `asset.get_asset_graph`. The `#3` normalization landed only in
  `asset.get_dependencies` (`AssetQueryHandler.cpp`) and
  `asset.get_dependencies_classified` (`AssetMetadataHandler.cpp`), but
  `asset.get_asset_graph` lives in `AssetMetadataHandler.cpp:508` and has its own
  un-normalized BFS: it seeds the queue with the raw caller `AssetPath` (`:538`)
  and at each step calls `AssetRegistry.GetDependencies(FName(*Current), ...)`
  (`:553`) — the same package-FName-keyed lookup, no `NormalizeAssetPath`, no
  object-suffix strip. So the object-path form yields a root node whose adjacency
  is empty and the BFS terminates immediately with `success:true` and a
  one-entry graph `{root:[]}` — the identical silent-wrong-empty, but here it
  certifies the asset as having NO dependency graph at all. Replay-confirmed against
  `mcp__editor-automation__call` on a DemoRoom material instance
  (`/Game/Global/DemoRoom/Materials/MI_Metal_Light`), the path form being the exact
  canonical `Package.Object` string `asset.get`/`asset.list` return:
  - `asset.get {assetPath:"/Game/Global/DemoRoom/Materials/MI_Metal_Light.MI_Metal_Light"}`
    → resolves; `result.path` is `"...MI_Metal_Light.MI_Metal_Light"` (object-path is
    the canonical form the sibling read verbs hand back).
  - `asset.get_asset_graph {assetPath:"/Game/Global/DemoRoom/Materials/MI_Metal_Light.MI_Metal_Light", maxDepth:3}`
    → `{"success":true,"graph":{"/Game/Global/DemoRoom/Materials/MI_Metal_Light.MI_Metal_Light":[]}}`
    (object-path form — silent wrong-empty, the whole graph is just an empty root)
  - `asset.get_asset_graph {assetPath:"/Game/Global/DemoRoom/Materials/MI_Metal_Light", maxDepth:3}`
    → `{"success":true,"graph":{"/Game/Global/DemoRoom/Materials/MI_Metal_Light":["/Game/Global/DemoRoom/Materials/M_Metal"],"/Game/Global/DemoRoom/Materials/M_Metal":["/Game/Global/DemoRoom/Materials/Textures/T_GrungySurface_slnneipc_4K_MR"],"/Game/Global/DemoRoom/Materials/Textures/T_GrungySurface_slnneipc_4K_MR":[]}}`
    (package-name form — the real 3-node BFS graph)
  Cross-check proving the object-path empty is wrong, not legitimately empty: the
  fixed-sibling `asset.get_dependencies {assetPath:"...MI_Metal_Light.MI_Metal_Light"}`
  → `{"dependencies":["/Game/Global/DemoRoom/Materials/M_Metal"]}` (the `#3` fix
  already tolerates the object path there) — so within one namespace the SAME
  object path now returns real deps from `get_dependencies` but an empty graph from
  `get_asset_graph`, an asymmetry the `#3` fix introduced by not covering this third
  verb. Same fix needed: run the seed `AssetPath` (and ideally each queued node)
  through `NormalizeAssetPath` before the package-FName-keyed
  `GetDependencies`/`GetReferencers` lookup at `AssetMetadataHandler.cpp:553` so the
  object-path and package-name forms yield identical graphs (or error rather than
  return a silent empty root). Note the BFS frontier nodes come from
  `Dep.ToString()` (package names) so the recursion is unaffected — only the
  caller-supplied ROOT needs normalizing — but normalizing every node is harmless
  and future-proof. Seed `asset.get_dependencies` (SEED mode); culprit
  `asset.get_asset_graph`.
- `#3-fix-normalize-object-path` `IN-REVIEW` developer — Reworded the ticket to
  correct two citation slips (the classified verb lives at
  `AssetMetadataHandler.cpp:285` registration / `:361` lookup, NOT
  `AssetQueryHandler.cpp:303`; and `Ctx.RequireAssetPath` does NOT strip the
  `.AssetName` suffix so it was dropped from the fix-helper list — `NormalizeAssetPath`
  at `AssetUtils.cpp:44` is the correct existing helper). Implemented the
  root-cause fix: both verbs now run the input `assetPath` through
  `NormalizeAssetPath` (which strips the object suffix via
  `FPackageName::ObjectPathToPackageName` and validates the package name) before
  the package-FName-keyed `AssetRegistry::GetDependencies` lookup, so the
  object-path and package-name forms resolve to the identical dependency list —
  matching the tolerant sibling read verbs. On an invalid/unnormalizable path the
  original string is preserved (no new error; behavior unchanged for genuinely bad
  input). Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Asset/AssetQueryHandler.cpp`
  (`asset.get_dependencies`, both the non-recursive `:53` lookup and the recursive
  `:36` seed now run against the normalized path),
  `Source/EditorAutomationRpcGateway/Private/Handlers/Asset/AssetMetadataHandler.cpp`
  (`asset.get_dependencies_classified`, normalize after the `IsValidAssetPath`
  guard and before the `:361` lookup). Regression test:
  `Source/EditorAutomationRpcGateway/Private/Tests/Assets/TestAssetDependenciesObjectPath.cpp`
  — `EditorAutomationRpcGateway.asset.get_dependencies.ObjectPathEquivalence` and
  `…get_dependencies_classified.ObjectPathEquivalence` scan the live asset registry
  for any `/Game` asset with a hard package dependency, then assert the object-path
  form (`package.AssetName`) returns the identical dependency list as the
  package-name form via the real handlers; they would fail (object form `[]` ≠
  package form's non-empty list) if the normalization were reverted, and
  warn-and-pass on an empty headless registry (same pattern as the existing
  `asset.search.NativeSubclass` live test). Not yet compiled/tested (later phase).
