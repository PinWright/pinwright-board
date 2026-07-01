---
id: B-asset-dump-bp-meta-degrades-when-genclass-null
title: "asset.dump meta.json reports className='Blueprint' and parentClass='' when GeneratedClass is null"
status: DONE
severity: High
category: bug
tags: [asset-dump, blueprint, meta]
---

# asset.dump meta.json reports className='Blueprint' and parentClass='' when GeneratedClass is null

For Blueprints whose GeneratedClass fails to load at dump time, `meta.json` reports `className: "Blueprint"` (raw type, not the generated `_C` class) and `parentClass: ""`. Despite the meta degradation, `bpir.txt` may still be fully populated — the partial-load path walked the BP graphs but didn't resolve GeneratedClass.

The properties.json placeholder fix from `B-asset-dump-properties-skipped-on-stub-class` ensures the file is emitted, but does not fix the meta.json fields.

**Repro:**
1. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/App/LevelBlueprints/B_SearchMode_Mine/meta.json`.
2. Observe: `className: "Blueprint"`, `parentClass: ""`.
3. Inspect the matching `bpir.txt` — fully populated (React event with SpawnSystemAtLocation, etc.).

**Fix:** In `BuildMetaJson` (`AssetDumpBuilder.cpp:49,55` — note: ticket originally said `BuildAssetMeta`; actual function name is `BuildMetaJson`), the `ClassName` null-fallback returned the literal `Asset->GetClass()->GetName()` (`"Blueprint"` / `"WidgetBlueprint"`) when `GeneratedClass` was null. Synthesize the canonical `{Name}_C` from the BP's own name (`BP->GetName() + TEXT("_C")`) on the failure path. Leave the existing `ParentClass`-null guard alone — empty is the honest answer when the parent class itself failed to resolve. Note: the ticket's earlier proposal (`Blueprint->ParentClass->GetPathName()`) would null-deref on the cited repro asset (`B_SearchMode_Mine`), where both pointers are null.

## History
- `#1-initial-repro` `OPEN` reporter — Blueprints with null/unloaded GeneratedClass produce `meta.json` with `className: "Blueprint"`, `parentClass: ""` while `bpir.txt` is fully populated. Sample path: `App/App/LevelBlueprints/B_SearchMode_Mine/meta.json`. Sole repro in 30,335-asset sweep.
- `#2-synthesize-bp-name-c-on-stub-class` `IN-REVIEW` developer — Synthesized `{Name}_C` from BP/WBP own name in `BuildMetaJson` (`AssetDumpBuilder.cpp:49,55`) when GeneratedClass is null; original proposal would have null-deref'd on the cited repro asset because ParentClass is also null. Added `FAssetDumpBuilderMetaJsonBpStubClassTest` covering BP and WBP cases.
- `#3-verify-fix` `DONE` tester — Verified: live `asset.dump` on `/App/App/LevelBlueprints/B_SearchMode_Mine` now writes `meta.json` with `className: "B_SearchMode_Mine_C"` (was `"Blueprint"`); `parentClass` correctly stays empty since ParentClass is also null on this asset, matching the IN-REVIEW spec.
