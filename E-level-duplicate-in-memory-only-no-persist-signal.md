---
id: E-level-duplicate-in-memory-only-no-persist-signal
title: "level.duplicate returns duplicated:true for an in-memory-only copy (no .umap on disk) with no persistence signal, so an immediate level.load fails [LEVEL_NOT_PERSISTED]"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [level, duplicate, persistence, in-memory, save, error-message, level-not-persisted, wiki]
---

# `level.duplicate` reports `duplicated:true` but the copy is only in memory

`level.duplicate` calls `UEditorAssetLibrary::DuplicateAsset(SourcePath,
DestinationPath)` (`LevelHandler.cpp:1019`) and, on success, returns
`{sourcePath, destinationPath, duplicated:true}` (`:1022-1026`). `DuplicateAsset`
creates the duplicate **in memory only** (the package is marked dirty but never
written) — no `.umap` lands on disk. The `duplicated:true` result carries **no
persistence signal** (no `saved`, no `persisted`, no "in-memory only" note), and
the wiki page documents none either ("Duplicate an existing level package to a
new location without modifying the source." — no mention that the copy is
unsaved). So a caller reasonably reads `duplicated:true` as "an addressable,
loadable duplicate now exists" and the very next natural step — load the copy as
the active world — fails:

```
level.duplicate {sourcePath:/Game/Maps/Lighting/Lighting_Realtime,
                 destinationPath:/Game/Maps/Lighting/Sandbox/OracleReplay_DupTrap}
  -> {"sourcePath":"/Game/Maps/Lighting/Lighting_Realtime",
      "destinationPath":"/Game/Maps/Lighting/Sandbox/OracleReplay_DupTrap",
      "duplicated":true}
level.load {levelPath:/Game/Maps/Lighting/Sandbox/OracleReplay_DupTrap}
  -> [LEVEL_NOT_PERSISTED] Level '/Game/Maps/Lighting/Sandbox/OracleReplay_DupTrap'
     is registered/loaded in memory but was never saved to disk, so it cannot be
     loaded from disk. Save it (level.save / level.save_as) or discard the
     orphaned world; do not retry path forms.
```

On-disk confirmation: after `duplicated:true`, `Content/Maps/Lighting/Sandbox/`
contains **no** `OracleReplay_DupTrap.umap` — the duplicate exists only in
memory + the asset registry.

## Why this is ergonomic, not a bug

This is intentionally NOT filed as a tool bug / false-success. The duplicate
genuinely exists, is recoverable, and the state is honest — unlike
`B-create-level-saved-true-no-umap` (`create_level`), which bogusly *clears* the
dirty flag and strands an **unflushable** orphan that `editor.save_all` reports
`totalDirty:0` for. Here the package stays dirty, so `editor.save_all`
(or `level.save_as`) DOES persist it, and the downstream `LEVEL_NOT_PERSISTED`
error (shipped by IN-REVIEW `E-level-load-file-not-found-vs-in-memory-orphan`)
already names the recovery. The friction is purely that `level.duplicate`'s own
success contract is misleading about completeness: `duplicated:true` with no
persistence field implies a load-ready on-disk copy, when a separate save is
required first. The audited attempt hit exactly this — `duplicated:true`, then a
first `level.load` failed `LEVEL_NOT_PERSISTED`, then `editor.save_all`, then a
second `level.load` succeeded — extra calls that an honest result/doc would
collapse.

## Quotable demonstration (verbatim, replayed via mcp__editor-automation__call)

- `level.duplicate` result: `{"sourcePath":"/Game/Maps/Lighting/Lighting_Realtime","destinationPath":"/Game/Maps/Lighting/Sandbox/OracleReplay_DupTrap","duplicated":true}`
- immediately-following `level.load` on that destination:
  `[LEVEL_NOT_PERSISTED] Level '/Game/Maps/Lighting/Sandbox/OracleReplay_DupTrap' is registered/loaded in memory but was never saved to disk, so it cannot be loaded from disk. Save it (level.save / level.save_as) or discard the orphaned world; do not retry path forms.`
- on-disk: no `OracleReplay_DupTrap.umap` under `Content/Maps/Lighting/Sandbox/`.

## Fix (scoped to message/doc clarity — the in-memory copy is honest, just under-signalled)

Two cheap, additive, contract-preserving changes — NO behavior change to the
duplicate itself:

1. **Echo a persistence signal in the result.** Add `saved:false` plus a short
   `note:"Duplicate exists in memory only (not yet on disk). Save it
   (level.save_as / editor.save_all) before level.load can open it; an immediate
   load fails LEVEL_NOT_PERSISTED."` so the caller learns the copy is not yet on
   disk without having to discover it via a failed load. These are additive
   fields; `duplicated:true` keeps its meaning (the in-memory copy genuinely
   exists). Deliberately NOT adding an opt-in `save` param: that would drag
   `level.duplicate` into the `McpSafeLevelSave` disk-verify path that
   `create_level` is currently losing to (`B-create-level-saved-true-no-umap`),
   trading real save-path risk for a cosmetic gain the `saved:false`/`note`
   signal + `LEVEL_NOT_PERSISTED` recovery already cover.
2. **Document it in the wiki.** The `level.duplicate` overlay (under
   `docs/wiki-src/level.md`) gets a `### level.duplicate` section stating the
   duplicate is created in memory only and must be saved (`level.save_as` /
   `editor.save_all`) before `level.load` can open it from disk — cross-referencing
   `LEVEL_NOT_PERSISTED`.

## Workaround

After `level.duplicate`, call `editor.save_all` (or `level.save_as` to write the
copy) before `level.load` on the destination — exactly the recovery the
`LEVEL_NOT_PERSISTED` error names.

## History
- `#1-initial-repro` `OPEN` reporter — SEED-mode oracle replay of `level.duplicate`
  (duplicate a lighting map to a sandbox copy, then load + light it). Replayed
  `level.duplicate {/Game/Maps/Lighting/Lighting_Realtime →
  /Game/Maps/Lighting/Sandbox/OracleReplay_DupTrap}` → `{...,"duplicated":true}`;
  an immediate `level.load` on the destination → `[LEVEL_NOT_PERSISTED]` (verbatim
  above); on-disk check confirmed no `OracleReplay_DupTrap.umap` was written
  (duplicate is in-memory only). Root cause: handler uses
  `UEditorAssetLibrary::DuplicateAsset` (`LevelHandler.cpp:1019`), which makes an
  unsaved in-memory copy, and the `duplicated:true` result (`:1022-1026`) plus the
  wiki page carry no persistence caveat. Classed ergonomic, not bug: the dirty
  package is recoverable (`editor.save_all`/`level.save_as` persist it) and the
  `LEVEL_NOT_PERSISTED` diagnostic already names the recovery — distinct from the
  unflushable false-`saved:true` orphan of `B-create-level-saved-true-no-umap`
  (`create_level`, IN-REVIEW) and from the load-error-wording fix of
  `E-level-load-file-not-found-vs-in-memory-orphan` (IN-REVIEW, which SHIPPED the
  `LEVEL_NOT_PERSISTED` message this repro relies on). Dedup: ripgrep over OPEN +
  closed found no ticket on `level.duplicate`'s in-memory-only result / missing
  persistence signal (the only `DuplicateAsset`-adjacent persistence tickets are
  about `create_level` and `world_partition.create_datalayer`, different methods).
- `#2-reword-and-fix` `IN-REVIEW` developer — Reworded: corrected the stale line
  citations (`:995`→`:1019` DuplicateAsset call, `:998-1002`→`:1022-1026` result
  block — the file had shifted ~24 lines) and narrowed the Fix to the
  additive-result-signal + wiki-doc scope, dropping the opt-in `save` param (it
  would pull `level.duplicate` into the `McpSafeLevelSave` disk-verify path that
  `create_level` is still losing to in `B-create-level-saved-true-no-umap` — real
  save-path risk for a cosmetic gain the new signal + `LEVEL_NOT_PERSISTED`
  recovery already cover). Implemented: `level.duplicate`'s success result now
  echoes `saved:false` plus a `note` naming `level.save_as`/`editor.save_all` as
  the save step before `level.load` (additive — `duplicated:true` unchanged), and
  the wiki overlay gains a `### level.duplicate` section documenting the
  in-memory-only copy + cross-referencing `LEVEL_NOT_PERSISTED`. Files:
  `Source/PinWright/Private/Handlers/Level/LevelHandler.cpp` (handler result),
  `docs/wiki-src/level.md` (overlay). Regression test:
  `FLevelDuplicateReportsInMemoryNotSavedTest`
  (`PinWright.level.duplicate.ReportsInMemoryNotSaved` in
  `Source/PinWright/Private/Tests/World/TestLevelHandlers.cpp`) duplicates
  `/Engine/Maps/Entry` through the production handler and asserts the success
  result carries `duplicated:true`, `saved:false`, and a non-empty `note`;
  reverting the fix drops `saved`/`note` and fails it. The persistence assertions
  fire only on the success branch (the duplicate is environment-dependent in CI)
  and the probe map is discarded via `DiscardProbeMapPackage` on every exit path.
