---
id: B-lighting-create-level-false-success-no-umap
title: "lighting.create_lighting_enabled_level reports success:true/existsAfter:true but writes no .umap — the create/save disk-presence hardening (VerifyLevelSavedToDisk) was never applied to this handler"
status: IN-REVIEW
severity: High
category: bug
tags: [lighting, create-level, save, false-success, silent-noop, no-umap, mcp-safe-level-save]
encounters: 1
lastSeen: 2026-07-04T16:17:14.0325862+03:00
---

# `lighting.create_lighting_enabled_level` falsely reports `success:true` for a level it never wrote to disk

`lighting.create_lighting_enabled_level` builds a fresh in-memory map
(`GEditor->NewMap()`), spawns a directional light + sky light, calls
`McpSafeLevelSave`, and on that boolean's `true` returns
`{success:true, existsAfter:true, message:"Level created with lighting",
levelPath:<path>}`. But **no `.umap` file is written to disk** — the world exists
only in memory + the asset registry. The caller gets a confident
`success:true`/`existsAfter:true` that contradicts on-disk reality: a classic
silent success-with-no-effect.

This is the same lenient `McpSafeLevelSave` OR-policy masking that
`B-create-level-saved-true-no-umap` and `B-level-save-saved-true-in-memory-no-umap`
document — but on a **different handler that the disk-presence hardening was never
applied to**. The `level.*` save/create verbs were fixed to re-gate the
reported-success boolean through the shared `VerifyLevelSavedToDisk` helper and now
honestly return `SAVE_VERIFICATION_FAILED` when no `.umap` lands. This lighting
handler still trusts the bare boolean, so it is strictly **worse** than its now-honest
siblings: it reports a FALSE `success:true` rather than an honest failure.

## Root cause (handler-confirmed)

`Source/PinWright/Private/Handlers/Environment/LightingHandler.cpp:900-914`:

```cpp
bool bSaved = McpSafeLevelSave(EditorWorld->PersistentLevel, *Path, 5);
if (bSaved)
{
    TSharedPtr<FJsonObject> Resp = MakeShared<FJsonObject>();
    Resp->SetBoolField(TEXT("success"), true);
    Resp->SetStringField(TEXT("path"), Path);
    Resp->SetStringField(TEXT("message"), TEXT("Level created with lighting"));
    Resp->SetBoolField(TEXT("existsAfter"), true);
    Resp->SetStringField(TEXT("levelPath"), Path);
    Ctx.SendSuccess(Resp);
}
```

`bSaved` comes straight from `McpSafeLevelSave` (`Utils/AssetUtils.cpp:468`), whose
`ShouldTreatLevelSaveAsSuccess` OR-policy (`Utils/AssetUtils.cpp:370-383`) returns
`true` when any of `bFileExistsOnDisk || bPackageExists || bAssetExists ||
bPackageClean` holds:

```cpp
if (!bSaveReportedSuccess) { return false; }
return bFileExistsOnDisk || bPackageExists || bAssetExists || bPackageClean;
```

For a freshly `NewMap()`'d world whose package was `CreatePackage`'d but never
written, the clean-package / registry signals satisfy the OR even though
`bFileExistsOnDisk == false`, so `bSaved` is `true` and `success:true` is a lie.
There is no JSON-level signal of the discrepancy.

## What it should do

Apply the exact create/save hardening the sibling verbs already carry: after
`McpSafeLevelSave`, re-gate the reported-success boolean through the shared
`VerifyLevelSavedToDisk(Path, bSaved, OutFilename, OutErrorCode)` helper
(`Utils/AssetUtils.cpp:392-414`) — resolve the package to a `.umap` filename, probe
`IFileManager::Get().FileExists`, and only report `success:true`/`existsAfter:true`
when the file actually landed. When it did not, drive the existing `SAVE_FAILED`
branch with the honest `SAVE_VERIFICATION_FAILED` code (naming the in-memory-only
cause) instead of a false `success:true`. This is the same fix already applied at
`LevelHandler.cpp:271` (`level.save`), `LevelHandler.cpp:358` (`level.save_as`), and
`LevelStructureHandler.cpp:297` (`level.structure.create_level`); the lighting
handler was simply missed.

## Distinct from the already-filed siblings

- **`B-create-level-saved-true-no-umap`** (IN-REVIEW) — `level.structure.create_level`
  in `LevelStructureHandler.cpp`; hardened to return `SAVE_VERIFICATION_FAILED`. This
  is a **different handler/namespace** (`lighting.create_lighting_enabled_level`,
  `LightingHandler.cpp`) that its fix does not touch, and it still emits a false
  `success:true` rather than the honest error.
- **`B-level-save-saved-true-in-memory-no-umap`** (IN-REVIEW) — `level.save`/`level.save_as`;
  same shared root cause, already re-gated through `VerifyLevelSavedToDisk`. Exactly
  the precedent for filing this as its own per-verb ticket: same OR-policy, different
  verb the fix does not reach.
- **`B-level-create-makes-wp-map`** (IN-REVIEW) — `level.create` `NewMap(true)` produces
  a WP map. Distinct: this lighting handler calls `NewMap()` (default
  `bIsPartitionedWorld=false`, a non-WP map with `.umap` = the honest success signal),
  so the missing `.umap` here is the false-`success` OR-policy, not WP scaffolding.

severity rationale: impact=silent-false-success (caller trusts a lie — `success:true`
on a level that does not exist on disk) × reach=normal-path (create-a-lighting-level
is the very first step of a lighting-sandbox setup; not rare) -> High

## Repro (replayed via mcp__pinwright__call)

1. `lighting.create_lighting_enabled_level {path:"/Game/Maps/OracleLightReplay_LS1"}`
   → `{success:true, path:"/Game/Maps/OracleLightReplay_LS1", message:"Level created
   with lighting", existsAfter:true, levelPath:"/Game/Maps/OracleLightReplay_LS1"}`.
2. On-disk check: `Content/Maps/` contains only `ExampleProjectWelcome.umap` — **no
   `OracleLightReplay_LS1.umap`** was written despite `success:true`/`existsAfter:true`.
3. `level.get_info {levelPath:"/Game/Maps/OracleLightReplay_LS1"}` →
   `{levelName:"PersistentLevel", actorCount:13}` (the world is present in memory as the
   active world; it is not persisted to disk).

## History
- `#1-initial-repro` `OPEN` reporter — Found during a lighting-sandbox setup task
  where every persistence route failed; `lighting.create_lighting_enabled_level` was the
  most misleading because it alone returns `success:true`/`existsAfter:true` on the no-op
  (the `level.*` save/create verbs now honestly return SAVE_VERIFICATION_FAILED). Replayed
  against the live editor: `lighting.create_lighting_enabled_level
  {path:"/Game/Maps/OracleLightReplay_LS1"}` → `{success:true, existsAfter:true}`, but
  `Content/Maps/` held only `ExampleProjectWelcome.umap` (no `.umap` written).
  Code-confirmed: `LightingHandler.cpp:900` calls bare `McpSafeLevelSave` and reports
  `success:true` from its lenient `ShouldTreatLevelSaveAsSuccess` OR-policy boolean
  (`AssetUtils.cpp:370-383`) with no disk re-gate — the `VerifyLevelSavedToDisk` hardening
  (`AssetUtils.cpp:392`) that `level.save`/`level.save_as`/`level.structure.create_level`
  already apply (`LevelHandler.cpp:271`/`:358`, `LevelStructureHandler.cpp:297`) was never
  extended here. Dedup: ripgrep over OPEN + closed for `create_lighting_enabled_level` /
  `lighting.create` found no prior ticket; distinct from `B-create-level-saved-true-no-umap`
  / `B-level-save-saved-true-in-memory-no-umap` / `B-level-create-makes-wp-map` (different
  handler/namespace, and this one still emits a false `success:true` rather than an honest
  error). Seeds the shared `mcp-safe-level-save` / `false-success` symptom family.
- `#2-fix-regate-verifylevelsavedtodisk` `IN-REVIEW` developer — Applied the same disk-presence
  hardening the sibling level.* verbs carry. `LightingHandler.cpp` (`lighting.create_lighting_enabled_level`)
  now re-gates the bare `McpSafeLevelSave` boolean through the shared
  `VerifyLevelSavedToDisk(Path, bSaveReported, OutFilename, OutErrorCode)` helper
  (`AssetUtils.cpp:392`): it only reports `success:true`/`existsAfter:true` when the `.umap`
  actually lands on disk, and otherwise drives `Ctx.SendError` with the honest
  `SAVE_VERIFICATION_FAILED` (reported-success-but-no-file) / `SAVE_FAILED` code instead of a
  false success — mirroring `level.save` (`LevelHandler.cpp:271`), `level.save_as`
  (`LevelHandler.cpp:358`), and `level.structure.create_level` (`LevelStructureHandler.cpp:296-321`).
  Files: `Source/PinWright/Private/Handlers/Environment/LightingHandler.cpp`. Regression test:
  `PinWright.lighting.create_lighting_enabled_level.ReportsHonestPersistence` (added to
  `Source/PinWright/Private/Tests/World/TestLevelHandlers.cpp`, reusing that file's safe
  `DiscardProbeMapPackage` + `FScopedEditorWorldMapGuard` scaffolding) — drives the real handler
  end-to-end and asserts the persistence-honesty invariant (reported `success` must agree with
  `.umap`-on-disk reality; no file -> `success:false` + honest `SAVE_(VERIFICATION_)FAILED`). It
  fails if the handler is reverted to trust the bare `McpSafeLevelSave` boolean. Plugin compiled
  clean.
