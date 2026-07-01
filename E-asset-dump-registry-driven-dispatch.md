---
id: E-asset-dump-registry-driven-dispatch
title: "Move simple JSON-builder branches in BuildAllFilesForAsset into a registry"
status: DONE
severity: Medium
category: ergonomic
tags: [asset-dump, refactor, extensibility]
---

# Move simple JSON-builder branches in BuildAllFilesForAsset into a registry

`Handlers/Asset/AssetDumpHandler.cpp::BuildAllFilesForAsset` (lines
414–817) routes every supported asset class through an inline
`if (Cast<T>) … else if (Cast<U>) …` chain. Each new asset kind adds a
fresh branch and contributes to a single ~400-line function. The existing
`Utils/IrSidecarRegistry.{h,cpp}` only covers *text IR* sidecars
(`{ Text, Warnings, bSuccess }` shape — MGIR/AGIR/SCIR/BTIR); JSON
builders still all sit in the if/else chain.

A sibling registry for JSON-builder sidecars would let most new asset
types ship as a single registration record rather than an edit to the
hot dispatch function, while leaving the genuinely irregular branches
(Niagara, Widget, World, Blueprint family) explicit where they need to
be.

## Claim verification — branch count and shape

`BuildAllFilesForAsset` body spans lines 414–817 (404 LoC including
helpers and the meta attachment tail). Counting actual class branches
(not the redirector early return):

- **Top-level chain (`if`/`else if`, before the default tail):** 17
  branches — NiagaraSystem, NiagaraEmitter, NiagaraScript,
  ParticleSystem, MIC, UMaterial, UMaterialFunction, MetaSound
  (Patch||Source under `MCP_DISPATCH_HAS_METASOUND`), UWidgetBlueprint,
  UAnimBlueprint, UBlueprint, UDataTable, UBehaviorTree, UBlackboardData,
  UUserDefinedStruct, UWorld. Plus UObjectRedirector as a separate
  early-return guard above the chain.
- **Default `else` tail:** 13 branches — StaticMesh, SkeletalMesh,
  Texture, SoundWave, SoundCue, LevelSequence, AnimSequence, AnimMontage,
  BlendSpace, PhysicsAsset, Skeleton, LandscapeGrassType,
  SubsurfaceProfile. Default tail shares one Properties emission via
  the asset's CDO at the top of the `else` block (lines 729–730).

Total: 30 class-discriminated branches. The reporter's "~27" is in the
right ballpark; actual count is 30.

## Branch classification (registry candidates vs stay inline)

**Stay-inline (irregular, non-uniform sidecar shape) — 9 branches:**

- `UObjectRedirector` — emits redirector-only `properties.json` and
  returns early, bypassing the rest of the function. Cannot be a
  registry leaf without inverting control flow.
- `UNiagaraSystem`, `UNiagaraEmitter`, `UNiagaraScript` — each calls
  3–4 distinct `NiagaraDumpBuilder` helpers (Parameters, Stack, Graphs,
  Compile) plus the compile-state-into-meta attachment runs after the
  dispatch tail (lines 805–812). Not a single-builder branch.
- `UMetaSoundPatch || UMetaSoundSource` — needs the
  `MCP_DISPATCH_HAS_METASOUND` guard, plus `Props->RemoveField` to strip
  `RootMetasoundDocument` before properties emission. Two-class
  predicate, custom property-massaging.
- `UWidgetBlueprint` — calls the blueprint-properties aspect, the widget
  tree aspect, BPIR text, widget animations exporter (with its own
  diagnostic path), and the optional screenshot aspect gated on
  `bIncludeWidgetScreenshot` + `OutBinaryFiles`. Many side outputs.
- `UAnimBlueprint`, `UBlueprint` — `BuildBlueprintPropertiesAspect` plus
  optional `ShouldEmitBpirText`-gated BPIR plus SCS files plus
  (AnimBlueprint only) AnimGraph. Blueprint-properties path is
  fundamentally not a per-class-default Properties dump and feeds
  `BlueprintPropertiesStatus`.
- `UWorld` — builds Properties against CDO+ParentCDO (the only branch
  that uses ParentCDO), plus WorldSettings JSON, Level BP text,
  sublevels JSON, and `BuildLevelActorDumpFiles`. Multi-aspect.

**Registry candidates (Cast<T> + Properties + 1–2 JSON-builder calls,
optional text-emitter twin, optional diagnostic on null) — 21 branches:**

- *Top-level chain:* ParticleSystem (Cascade), MIC (MaterialInstance +
  diagnostic), UMaterial (Properties-only), UMaterialFunction
  (Properties-only), UDataTable (DataTable + diagnostic), UBehaviorTree
  (Properties-only), UBlackboardData (Properties-only),
  UUserDefinedStruct (UserDefinedStruct + diagnostic).
- *Default tail:* StaticMesh (JSON + text emitter), SkeletalMesh,
  Texture (JSON + text emitter), SoundWave, SoundCue, LevelSequence,
  AnimSequence, AnimMontage, BlendSpace, PhysicsAsset, Skeleton,
  LandscapeGrassType, SubsurfaceProfile.

All 21 follow one of three shapes:

- **A.** Class default Properties only (UMaterial, UMaterialFunction,
  UBehaviorTree, UBlackboardData).
- **B.** Class default Properties + one JSON sidecar from a single
  builder, optional null-diagnostic (MIC, UDataTable, UUserDefinedStruct,
  most of the default tail).
- **C.** Like B, plus a text-emitter twin written next to the JSON
  (StaticMesh + StaticMeshTxt, Texture + TextureTxt).

The default-tail branches additionally rely on the shared Properties
emission at the top of the `else` (line 730). A JSON-builder registry
must either keep that hoist or move it into the registry walker so the
top-level chain candidates also reuse it (since today they each emit
their own Properties line explicitly).

## Proposed registry shape

A sibling to `Utils/IrSidecarRegistry`, e.g.
`Utils/JsonSidecarRegistry.{h,cpp}`:

```
struct FJsonSidecarSpec {
    const TCHAR* Name = nullptr;          // human label, e.g. "static_mesh"
    const TCHAR* FileName = nullptr;      // DumpFileNames::* string
    UClass* (*ClassFn)() = nullptr;       // class discriminator thunk
    TSharedPtr<FJsonObject> (*BuildFn)(UObject*) = nullptr;
    const TCHAR* (*TextEmitterFileName)() = nullptr;   // optional pair
    FString (*TextEmitterFn)(TSharedPtr<FJsonObject>) = nullptr;
    const TCHAR* NullDiagnostic = nullptr; // if non-null, emit on null Build
    int32 Priority = 100;
};

REGISTER_DUMP_JSON_SIDECAR(...)  // one record per asset kind
```

The dispatcher walks the registry in priority order and emits at most
one (or paired JSON+text) sidecar per matching spec. The shared
class-default Properties emission happens unconditionally before the
walk for any asset that is not in the irregular allow-list (Niagara,
MetaSound, Widget/Anim/Blueprint, World, Redirector). Properties-only
classes (UMaterial, UMaterialFunction, UBehaviorTree, UBlackboardData)
become *zero registrations* — they fall through to the shared
Properties emit, full stop.

## Migration plan

Sequenced so each step is testable on its own and the asset-dump cache
stays warm where possible:

1. Add `Utils/JsonSidecarRegistry.{h,cpp}` and the
   `REGISTER_DUMP_JSON_SIDECAR` macro. Wire `BuildAllFilesForAsset` to
   call a `RunRegisteredJsonSidecars(Asset, Files, OutFileErrors)` at
   the spot the default tail does today, *before* deleting any inline
   branch. The walker is a no-op until registrations exist.
2. Hoist the shared "emit Properties against CDO" out of the default
   `else` into a small helper that runs for any asset not in the
   irregular allow-list. This subsumes branches that today exist purely
   to call that one helper.
3. Migrate the default-tail branches one type at a time
   (StaticMesh → SkeletalMesh → … → SubsurfaceProfile). Each migration
   deletes its `else if` and adds a `REGISTER_DUMP_JSON_SIDECAR` near
   the corresponding builder file. Bumps the aspect version if and only
   if the output bytes change (it should not for a pure refactor; see
   below).
4. Migrate the top-level Properties-and-one-sidecar branches
   (ParticleSystem, MIC, UDataTable, UUserDefinedStruct).
5. Delete the now-empty Properties-only branches (UMaterial,
   UMaterialFunction, UBehaviorTree, UBlackboardData) once step 2's
   hoist is in place.

After step 5, `BuildAllFilesForAsset` should hold only the irregular
branches (Niagara family, MetaSound, Widget/Anim/Blueprint, World) plus
the redirector early return — a ~150 LoC function instead of ~400.

## Aspect-version compatibility

`Handlers/Asset/AssetDumpCache.cpp::GetAspectVersion` and the `Versions`
table decide per-aspect freshness. Each candidate sidecar
(`static_mesh.json`, `texture.json`, etc.) has an entry there. Per the
plugin `CLAUDE.md` rule, an aspect-version bump is required only when
the *serialized bytes* change.

- A clean refactor that routes the same `BuildXJson(Asset)` call
  through the registry instead of an inline `if (Cast<X>)` should emit
  byte-identical output and therefore needs **no** version bump.
- The walker MUST preserve the existing emit order (Properties first,
  then the type-specific sidecar, then the meta + sidecarsEmitted tail
  on `MetaJson`). Order doesn't affect a single sidecar's bytes, but
  `meta.json::sidecarsEmitted` reflects the file list, so changing the
  list order would shift bytes there. Easiest guarantee: append in the
  same per-class order the inline chain uses today.
- If the refactor incidentally changes the diagnostic *message* a
  builder returns on null (e.g. centralized through `NullDiagnostic`),
  that does change recorded diagnostics, and the consumer is `OutFileErrors`
  (transient — not on disk in the dump), so still no bump needed.
- Any future *behavior* change (different builder, different field
  set, schema rev) bumps in the same commit, same rule as today. The
  registry doesn't change the bump rule, only the place the call lives.

Concretely: this refactor should ship with the `Versions` table
unchanged and a parity check that runs `asset.dump_folder` on a small
fixture set before and after, diffing the resulting `.editor-automation/asset-dumps/`
tree to byte-equal.

## History
- `#1-initial-scope` `OPEN` reporter — Filed from architecture audit
  against `Handlers/Asset/AssetDumpHandler.cpp::BuildAllFilesForAsset`
  (lines 414–817). Confirmed 30 class branches in the dispatch chain
  (not 27): 17 in the top-level chain plus the redirector early
  return, and 13 in the default `else` tail. 21 of the 30 are
  registry candidates (Cast + Properties + one JSON builder, with
  optional text-emitter twin and null-diagnostic); the 9 remaining
  (UObjectRedirector, Niagara System/Emitter/Script, MetaSound,
  UWidgetBlueprint, UAnimBlueprint, UBlueprint, UWorld) must stay
  inline due to multi-builder or non-uniform sidecar shapes. Existing
  `Utils/IrSidecarRegistry` covers text-IR sidecars only and is not
  reusable for JSON builders (different result shape); a sibling
  `Utils/JsonSidecarRegistry` is the proposed extension point.
  Aspect-version bumping rule from
  `Plugins/EditorAutomationRpcGateway/CLAUDE.md` applies: no bump
  needed for a byte-identical refactor; the migration plan calls for a
  before/after parity check on a fixture dump tree to enforce that.
  Non-blocking; lands incrementally.
- `#2-registry-implemented` `IN-REVIEW` developer — Added
  `Utils/JsonSidecarRegistry.{h,cpp}` (sibling to `IrSidecarRegistry`)
  with `FJsonSidecarSpec`, `FAutoRegisterJsonSidecar`,
  `GetRegisteredJsonSidecars()`, and the `REGISTER_DUMP_JSON_SIDECAR`
  macro. In `AssetDumpHandler.cpp::BuildAllFilesForAsset`, deleted all 21
  registry-candidate branches (ParticleSystem, MIC, UMaterial,
  UMaterialFunction, UDataTable, UBehaviorTree, UBlackboardData,
  UUserDefinedStruct, plus the 13 default-tail types) and replaced the
  terminal `else` with a `RunRegisteredJsonSidecars` helper that emits the
  shared class-default Properties sidecar then walks the registry
  (JSON + optional text twin + optional null diagnostic, deduped by file
  name, sorted by priority). The Properties-only kinds now register
  nothing and fall through to the shared emit. Niagara family, MetaSound,
  Widget/Anim/Blueprint, World, and the Redirector early return stay
  inline. Emitted 17 `REGISTER_DUMP_JSON_SIDECAR` records next to their
  builders (StaticMesh+txt, SkeletalMesh, Texture+txt, SoundWave,
  SoundCue, LevelSequence, AnimSequence, AnimMontage, BlendSpace,
  PhysicsAsset, Skeleton, LandscapeGrassType, SubsurfaceProfile,
  MaterialInstance, DataTable, UserDefinedStruct, Cascade). Pruned
  orphaned builder/type includes from `AssetDumpHandler.cpp`. No
  aspect-version bump (output is byte-identical). Added regression test
  `Tests/Assets/TestJsonSidecarRegistry.cpp` asserting the migrated
  registrations are present (class discriminator, text twin, null
  diagnostic) and that a registered build function produces a valid
  sidecar on a transient `UStaticMesh`.
- `#3-review-fixups` `IN-REVIEW` developer — Addressed review issue #1:
  reverted the three out-of-spec `Versions`-table entries
  (`niagara_parameters.json`, `niagara_stack.json`, `niagara_graphs.json`)
  added to `AssetDumpCache.cpp::GetAspectVersion`. The spec (line 189)
  requires the `Versions` table to ship unchanged for this byte-identical
  refactor, and those three sidecars are unrelated to the JSON-registry
  migration (`niagara_parameters`/`niagara_stack` are never emitted;
  `niagara_graphs` belongs to the Niagara handler). The remaining
  `AssetDumpCache.cpp`/`.h` deltas (SortedJsonWriter swap, cache-version
  bump to 2) are from the separate `B-sortedjson-not-enforced-on-disk-writes`
  ticket and were left intact. Review issue #2 (62+ unrelated files in the
  working tree — BPIR/animation/blueprint-graph/error-code/etc. subsystems)
  is a real branch-hygiene problem but is NOT this ticket's diff to revert:
  those are uncommitted, in-progress changes for other open board tickets
  (each has its own modified `Docs/board/*.md`). Discarding them would
  destroy unrelated work. The correct remedy is to scope the *commit* to
  this ticket's paths at staging time, not to `git checkout` foreign files
  out of the tree. This ticket's footprint is isolated and verified:
  `Utils/JsonSidecarRegistry.{h,cpp}`, `Handlers/Asset/AssetDumpHandler.cpp`,
  `Handlers/Asset/AssetDumpCache.cpp` (Versions table now unchanged), the 17
  `REGISTER_DUMP_JSON_SIDECAR` builder `.cpp` files, and
  `Tests/Assets/TestJsonSidecarRegistry.cpp`.
- `#4-verify-fix` `DONE` tester — Verified live via `asset.dump` on two
  migrated registry-candidate types. StaticMesh
  (`/Engine/EditorMeshes/Camera/SM_CraneRig_Base`, shape C) emitted
  `static_mesh.json` (populated bounds/materials/LOD/tri counts) + its
  `static_mesh.txt` text twin; DataTable
  (`/Game/ContextEffects/DT_AnimEffectTags`, shape B) emitted
  `data_table.json` (4 rows, correct `rowStruct`, alpha-sorted) plus the
  shared class-default `properties.json`. Both `meta.json::sidecarsEmitted`
  lists are correct and alpha-sorted, confirming the registry walker
  preserves the shared-Properties-then-sidecar emit through the registry
  path.
