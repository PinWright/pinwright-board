---
id: B-asset-dump-tier3-determinism-backlog
title: "Tier-3 dump-determinism backlog: DDC-derived mesh stats, CPF_Transient, auto-suffixed identities, TSet order"
status: OPEN
severity: Low
category: bug
tags: [asset-dump, determinism, backlog]
---

# Tier-3 dump-determinism backlog: DDC-derived mesh stats, CPF_Transient, auto-suffixed identities, TSet order

Residual low-impact nondeterminism sources left open by the July 2026 dump-determinism audit. The Tier-1/2 items (`pluginVersion` / `dumpSchemaVersion` removal from `meta.json`, Niagara `changeId` removal, `anim_graph.json` `pages[]` ordering) are fixed separately; the items below do not flip on routine re-dumps of an unchanged asset within one engine build - they surface only on engine rebuilds, transient editor state, or unlucky allocation order.

1. **StaticMesh stats read from built DDC render data** - `static_mesh.json` `trianglesByLod` / `verticesByLod` / `bounds` come from the built render data, which can shift after an engine rebuild (DDC rebuild) with no source-asset edit. SkeletalMesh is fine - its builder reads the imported model, not render data.
2. **Full-reflection builders don't skip `CPF_Transient`** - DataTable (`DataTableDumpBuilder.cpp:17`), StateTree (`StateTreeDumpBuilder.cpp:51`), and Cascade (`CascadeDumpBuilder.cpp:76`) serialize via full reflection with no transient-flag skip, so editor-transient property state can leak into the sidecars.
3. **Auto-suffixed instance names used as identity** - PhysicsAsset `bodies[].name` (`PhysicsAssetDumpBuilder.cpp:57`) and SoundCue node names / edge endpoints (`SoundCueDumpBuilder.cpp:28,159`) key on object instance names like `Foo_2` that the engine auto-suffixes at creation, so a semantically identical re-authored or duplicated asset gets different identities.
4. **TSet-typed UPROPERTY values serialize in native iteration order** - `PropertyExport.cpp:1036-1086`; the only remaining unsorted container path found by the audit.

**Fix:** low severity, fix opportunistically. Every individual fix changes serialized bytes for its aspect and MUST bump that aspect's entry in the `GetAspectVersion` table (`AssetDumpCache.cpp`) in the same commit.

## History
- `#1-filed-from-determinism-audit` `OPEN` reporter - Filed as the consolidated Tier-3 backlog from the dump-determinism audit; all four items carry file:line references from the audit pass. None churn routine same-build re-dumps, hence one Low ticket instead of four.
- `#2-set-ordering-fixed` `OPEN` developer - Partial implementation: generic `FSetProperty` values now sort by canonical serialized JSON before emission, and SoundCue concurrency paths are sorted. The ticket remains OPEN because its other Tier-3 items are unchanged.
