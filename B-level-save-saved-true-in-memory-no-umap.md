---
id: B-level-save-saved-true-in-memory-no-umap
title: "level.save reports result:{saved:true} for an unsaved in-memory-only active world while writing no .umap — the create_level disk-presence hardening (against the lenient McpSafeLevelSave OR-policy) was never applied to level.save / level.save_as"
status: IN-REVIEW
severity: High
category: bug
tags: [level, level-save, save, false-success, in-memory, no-umap, mcp-safe-level-save, job-status, silent-failure]
---

# `level.save` falsely reports `saved:true` for an in-memory-only world

`level.save` (`LevelHandler.cpp:217-261`) wraps `McpSafeLevelSave(Live, PackageName, 5)`
in an async job and sets the job result's `saved` field directly from that boolean
(`R->SetBoolField(TEXT("saved"), bOk)` at `:255`, `bOk = McpSafeLevelSave(...)` at
`:253`). When the **active editor world is an unsaved in-memory-only world**
(e.g. one produced by `create_level {save:false}`, which post-`E-create-level-save-false-world-not-active`
is now made the active world), `McpSafeLevelSave` returns `true` — its
`ShouldTreatLevelSaveAsSuccess` OR-policy treats a *clean package* / *registry
asset* as success even when **no `.umap` lands on disk** (the same lenient policy
documented in `B-create-level-saved-true-no-umap`). So:

- `level.save {}` → async job → `system.job_status` → `result:{saved:true}` —
  but **no `.umap` is written to disk**.

The caller gets a confident `saved:true` (through the now-correct completion
signal that `B-level-save-no-completion-signal` added) that **contradicts the
on-disk reality**. `level.save` is the only verb a caller would reach for to
persist the active world, and it reports a false success exactly when the world
most needs honest persistence (a freshly-created, never-on-disk world).

## Distinct from the already-filed siblings

- **`B-create-level-saved-true-no-umap`** (IN-REVIEW) fixed the **`create_level`
  handler only**. Its `#2` added a *create-specific* stricter predicate
  (`ShouldTreatCreateLevelSaveAsSuccess(bSaveReportedSuccess, bFileExistsOnDisk)`)
  and an `IFileManager::FileExists` probe **inside the `create_level` handler**,
  deliberately leaving the shared `McpSafeLevelSave` / `ShouldTreatLevelSaveAsSuccess`
  OR-policy intact (its package/mount-workflow tests lock it). `level.save`
  (`LevelHandler.cpp:253`) still calls bare `McpSafeLevelSave` with **no** disk
  probe, so it inherits the exact lenient masking the create handler was hardened
  against — on a *different verb that the create handler's fix does not touch*.
  In other words: it is the create-handler *hardening* (the `FileExists` probe +
  `ShouldTreatCreateLevelSaveAsSuccess` re-gate) that was never extended to
  `level.save` / `level.save_as`, leaving the shared OR-policy load-bearing there.
- **`B-level-save-no-completion-signal`** (DONE) added the async completion signal
  and `result:{saved:true/false}` — but its `#3` verification ran against a **real
  on-disk level** (`/Game/System/FrontEnd/Maps/L_Core`), where the `.umap` already
  exists, so it never exercised the in-memory-only case. It made the signal
  *arrive*; it did not make the signal *honest* for an unsaved world.
- **`B-level-save-as-no-completion-signal`** (DONE) — same shape on `level.save_as`
  (`LevelHandler.cpp:314` also calls bare `McpSafeLevelSave`); carries the same
  latent defect, so this fix hardens `level.save_as` too for parity.
- **`E-level-duplicate-in-memory-only-no-persist-signal`** (OPEN) is the *honest*
  case — `level.duplicate` leaves the package **dirty** and reports only
  `duplicated:true` (no false `saved`); it explicitly contrasts itself against the
  false-`saved:true` of `create_level`. `level.save` here is the false-`saved:true`
  case on a save verb, not a missing-signal case.

## Root cause (handler-confirmed)

`Source/PinWright/Private/Handlers/Level/LevelHandler.cpp:253-256`:

```cpp
const bool bOk = McpSafeLevelSave(Live, PackageName, 5);
auto R = MakeShared<FJsonObject>();
R->SetBoolField(TEXT("saved"), bOk);
OnComplete(bOk, R, bOk ? FString() : TEXT("SAVE_FAILED"));
```

`McpSafeLevelSave` → `ShouldTreatLevelSaveAsSuccess` returns true on
`bPackageClean || bAssetExists || …` even with `bFileExistsOnDisk == false`
(`Utils/AssetUtils.cpp`, per `B-create-level-saved-true-no-umap` root-cause). For a
brand-new in-memory world whose package was `CreatePackage`'d but never written,
the clean/registry signals are satisfied, so `bOk` is `true` and `saved:true` is a
lie.

## Fix

Apply the **same create-to-new-path disk-presence verification** that
`B-create-level-saved-true-no-umap` `#2` added to the create handler, but to
`level.save` (and, for parity, `level.save_as`): after `McpSafeLevelSave`, convert
`PackageName` to a `.umap` filename and probe `IFileManager::Get().FileExists`;
report `saved:true` only when the file actually exists on disk, otherwise
`saved:false` with an honest error (e.g. `SAVE_VERIFICATION_FAILED` /
`LEVEL_NOT_PERSISTED` naming the in-memory-only cause). Reuse
`ShouldTreatCreateLevelSaveAsSuccess` (already added in `Utils/AssetUtils`) rather
than re-deriving the probe. This keeps the shared OR-policy (and its package/mount
tests) intact while making disk presence load-bearing for the save verbs too.

## Process impact (why this is filed as a PROCESS finding, not just a re-judge)

In the audited task the agent had *already* taken the `create_level {save:false}`
workaround and built the whole WP world, then tried to persist it. **Three**
distinct save verbs each reported confident success while no `.umap` landed:
`level.save` → `system.job_status` → `result:{saved:true}`; and `editor.save_all`
→ `savedCount:2/totalDirty:2` (saved only the 2 DataLayer assets). With *no* honest
persistence signal anywhere, the agent was driven to **read the plugin's C++ and
board tickets** (`B-create-level-saved-true-no-umap`, `E-create-level-save-false-world-not-active`)
to understand why the save "succeeded" but produced no file — a discoverability
dead-end the friction note calls out verbatim: *"level.save reported saved:true but
on-disk checks proved no .umap exists (false success) … I had to read the plugin's
C++ board tickets … needing that source dive is itself a discoverability gap."*
The false `level.save{saved:true}` is the specific signal that contradicted reality
and forced the dive; fixing it removes one of the three misleading green lights.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of the
  `level.structure.create_level` "OW_Prototype" open-world-WP-setup task (19 calls).
  After the `create_level {save:true}` in-spec call hard-failed
  (`[SAVE_VERIFICATION_FAILED]`, world torn down — covered by
  `B-create-level-saved-true-no-umap` `#3` for this same seed) and the
  `{save:false}` workaround built the WP world in memory, the persist step exposed a
  **distinct** false-success: `level.save` → `system.job_status` returned
  `result:{saved:true}` for the in-memory-only active world with **no `OW_Prototype.umap`
  on disk**, and `editor.save_all` reported `savedCount:2/totalDirty:2` (only the 2
  DataLayer assets; the `.umap` still never written). Root-caused to
  `LevelHandler.cpp:259-261` calling bare `McpSafeLevelSave` (lenient
  `ShouldTreatLevelSaveAsSuccess` OR-policy) with no disk probe — the exact masking
  that `B-create-level-saved-true-no-umap` `#2` hardened **only in the `create_level`
  handler**, never in `level.save`. Distinct from `B-create-level-saved-true-no-umap`
  (different verb/handler; create-handler fix does not touch `level.save`), from
  `B-level-save-no-completion-signal` (DONE — added the signal; verified against a
  real on-disk `L_Core`, never the in-memory case), and from
  `E-level-duplicate-in-memory-only-no-persist-signal` (the *honest* dirty-package
  case, no false `saved`). The judge filed `B-level-structure-info-datalayers-stub`
  for this task's data-layer-readback bug; this is the orthogonal PROCESS angle — the
  false `level.save{saved:true}` that, alongside the false `editor.save_all` accounting,
  left the agent with no honest persistence signal and forced a C++/board source dive
  (friction note: *"needing that source dive is itself a discoverability gap"*).
  Dedup: ripgrep over OPEN + closed (`level.save`, `saved:true`, `no.*umap`,
  `McpSafeLevelSave`, `in-memory`) found the false-`saved:true` defect filed only
  against `create_level`, never against the `level.save` verb.
- `#2-reword-and-fix` `IN-REVIEW` developer — Reworded: the title/body said the
  lenient OR-policy "was never applied to level.save", which is backwards — it is
  the **create-handler disk-presence HARDENING** (the `FileExists` probe +
  `ShouldTreatCreateLevelSaveAsSuccess` re-gate from `B-create-level-saved-true-no-umap`
  `#2`) that was never extended to the save verbs; corrected the title and the
  framing, and refreshed the stale line citations (`:223-267`/`:259-262` → actual
  `:217-261`/`:253-256`; `save_as` bare call `:320` → `:314`). Fix: extended the
  exact create-level hardening to **both** save verbs in
  `Source/PinWright/Private/Handlers/Level/LevelHandler.cpp` — `level.save`
  (handler at `:217`) and `level.save_as` (`:264`) now, after `McpSafeLevelSave`,
  re-gate the reported-success boolean through the shared `VerifyLevelSavedToDisk`
  helper (resolve `PackageName`/`savePath` to a `.umap` filename, probe
  `IFileManager::Get().FileExists`, then apply
  `ShouldTreatCreateLevelSaveAsSuccess(bSaveReported, bFileOnDisk)`).
  `saved:true` is now reported only when the `.umap` is actually on disk; a save
  that reported success but wrote no file (in-memory-only active world, write
  blocked, source-control lock) now returns `saved:false` with
  `SAVE_VERIFICATION_FAILED` plus the `levelPath` / `persistenceNote` fields that
  steer the caller to `level.save_as`. `save_as` also now only re-scans the asset
  registry when the file genuinely landed. The shared
  `ShouldTreatLevelSaveAsSuccess` OR-policy (and its package/mount tests) is left
  untouched — disk presence is just made load-bearing at the verb boundary, same
  as create_level. Two complementary regression tests in
  `Source/PinWright/Private/Tests/Core/TestLevelSaveLoadUtils.cpp` (merged from the
  two parallel fix hosts, both retained):
  `FLevelSaveInMemoryWorldReportsNotPersistedTest`
  (`PinWright.core.level_save_load.level_save_disk_probe.InMemoryWorldNotPersisted`)
  locks the underlying predicate contract — `CreatePackage`s a real
  never-saved-to-disk world, reproduces the package→.umap→`FileExists` probe,
  confirms the lenient OR-policy still masks it via the clean-package fallback, and
  asserts `ShouldTreatCreateLevelSaveAsSuccess(true, false)` reports honest `false`
  (and `(true, true)` reports `true`); and
  `FLevelSaveVerbRejectsInMemoryNoDiskFileTest`
  (`PinWright.core.level_save_load.level_save_verb.RejectsInMemoryNoDiskFile`) locks
  the bundled `VerifyLevelSavedToDisk` helper + its error-code classification —
  asserting a reported-success save with no `.umap` resolves to `saved:false` /
  `SAVE_VERIFICATION_FAILED`, and a non-reported save to `SAVE_FAILED`. Both fail if
  either handler is reverted to a bare `saved = McpSafeLevelSave(...)`.
