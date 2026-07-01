---
id: B-material-graph-mutators-bypass-editor-open-guard
title: "material.graph.* mutators bypass the open-editor guard (silent clobber when Material Editor is open)"
status: DONE
severity: High
category: bug
tags: [material, handler, editor, silent-clobber, mcp]
---

# material.graph.* mutators bypass the open-editor guard (silent clobber when Material Editor is open)

**Problem:** The fix for `B-material-graph-edit-clobbered-by-open-editor` added an
open-editor guard — `UE::EditorAutomation::Material::IsMaterialEditorOpen(UMaterial*)`
in `MaterialFinders.h`, wired through the `LOAD_MATERIAL_OR_RETURN` macro — to the
`material.authoring.*` family and the MPC handler. But `MaterialGraphHandler.cpp`'s
`material.graph.*` mutators load the `UMaterial` via their own bare
`LoadObject<UMaterial>` and never call the guard. So they still SILENTLY CLOBBER when
the asset is open in the Material Editor: `FMaterialEditor` edits a working copy that
overwrites external edits on its next save/sync, and the RPC returns success.

**Exposed RPCs** (all in `Source/EditorAutomationRpcGateway/Private/Handlers/Material/MaterialGraphHandler.cpp`,
7 mutators): `material.graph.add_node` (~line 44), `remove_node` (~106),
`connect_nodes` (~156), `break_connections` (~242), `add_texture_sample` (~410),
`add_expression` (~481), `create_nodes` (~528). `material.graph.get_node_details` is
read-only — no guard needed.

**Fix sketch:** Prefer promoting `LOAD_MATERIAL_OR_RETURN` (currently a file-local
macro in `MaterialAuthoringHandler.cpp`) into a shared header (e.g. `MaterialFinders.h`)
so both authoring and graph handlers use one guarded loader (DRY). Alternative: add an
explicit `IsMaterialEditorOpen(Material)` → `EDITOR_OPEN` error check immediately after
each `LoadObject` in the 7 mutators, matching the authoring pattern. Regression test:
drive `material.graph.add_node` on a material whose editor is open and assert
`EDITOR_OPEN`; mirror `Tests/Material/TestMaterialGraphEditWithEditorOpen.cpp`.

**Related — possible systemic exposure (UNVERIFIED, investigate separately before
ticketing):** Niagara mutators (`NiagaraEditHandler`, `NiagaraAdvancedEditHandler`,
`NiagaraCurveHandler`, `NiagaraGraphHandler`) and Blueprint graph handlers also
`LoadObject` + mutate assets backed by working-copy editors (`FNiagaraSystemToolkit`,
`FBlueprintEditor`) with no analogous guard. Confirm which editors clobber-on-save
before filing; if confirmed, add type-specific `IsXEditorOpen` helpers per family (via
`UAssetEditorSubsystem::FindEditorForAsset`), not a single generic helper.

## History
- `#1-graph-family-unguarded` `OPEN` reporter — Found during a wiki-sync one-hop audit: the `B-material-graph-edit-clobbered-by-open-editor` fix guarded only `material.authoring.*` (+ MPC); the 7 `material.graph.*` mutators in `MaterialGraphHandler.cpp` load via bare `LoadObject<UMaterial>` with no `IsMaterialEditorOpen` check, so they remain exposed to silent clobber when the Material Editor is open. Sibling ticket: `B-material-graph-edit-clobbered-by-open-editor`.
- `#2-guard-graph-mutators` `IN-REVIEW` developer — Added a shared guarded material loader in MaterialFinders.h and routed all material.graph mutators in MaterialGraphHandler.cpp through it, so open Material Editor assets now return EDITOR_OPEN instead of mutating the LoadObject copy. Refactored existing authoring/MPC guarded loaders to the shared helper and added TestMaterialGraphMutatorsWithEditorOpen.cpp covering the graph mutator family.
- `#3-verify-fix` `DONE` tester — Verified live: opened `/Game/UI/Hud/M_MSDF_Icon` via `editor.open_asset`, then `material.graph.add_node` and `material.graph.create_nodes` both returned `[EDITOR_OPEN] Material is currently open in the Material Editor` instead of mutating; closed the editor with `editor.close_asset` after. Guard confirmed wired across two distinct graph mutators.
