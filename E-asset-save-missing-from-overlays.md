---
id: E-asset-save-missing-from-overlays
title: "asset.save absent from asset / safe-mutation-save overlays; agents fall back to blunt editor.save_all"
status: OPEN
severity: Medium
category: ergonomic
tags: [asset, save, wiki, overlay, discoverability]
---

# asset.save absent from asset / safe-mutation-save overlays

`asset.save {assetPath, force?}` is the correct narrow single-package writer
(registered `AssetSaveHandler.cpp:34`), but the two overlay pages an agent reads
before persisting never mention it:

- `docs/wiki-src/safe-mutation-save.md` — recipe step 5 (lines 29-32) and the
  "Choosing The Writer" table (lines 44-45) name only `editor.save_all` /
  `level.save` / `level.save_as`. There is no single-asset row.
- `docs/wiki-src/asset.md` — no save section at all; the only saves it documents
  are the rename/move fixup re-saves (`asset.fixup_redirectors`, lines 191-197).

Only `blueprint.md:384` names `asset.save`, and it links `[asset.save](asset.md)`
— a dangling cross-ref, since `asset.md` has no such anchor or content.

Consequence: an agent reading those pages concludes `editor.save_all` is the only
persist option and uses it, which flushes EVERY dirty package — persisting
unrelated incidentally-dirtied assets alongside the intended one. `asset.save`
would have persisted only the one package.

**Fix:** Add `asset.save` to the safe-mutation-save "Choosing The Writer" table
and recipe step 5 as the targeted alternative to `editor.save_all`, and give
`asset.md` an `asset.save` mention so the `blueprint.md` cross-ref resolves.
Impl of the RPC itself is `F-asset-save` (IN-REVIEW) — this ticket is the
overlay-docs follow-up only.

## History
- `#1-initial-repro` `OPEN` reporter — Confirmed: grep for `asset.save` across `docs/wiki-src/` matched only `blueprint.md:384` (a `[asset.save](asset.md)` link to a page with no such anchor); `safe-mutation-save.md` (step 5 lines 29-32, table lines 44-45) and `asset.md` (no save section) omit it entirely. Handler exists at `AssetSaveHandler.cpp:34`. Session evidence: lacking any single-asset save on those pages, used `editor.save_all {}` → `{savedCount:2, totalDirty:2}`; git showed the intended `PM_Drone.uasset` AND unrelated `W_DroneSelect_EditDrone.uasset` modified. `asset.save {assetPath}` would have persisted only `PM_Drone`. Separate from `F-asset-save` (impl ticket, IN-REVIEW) whose Fix/History cover the handler + test only, not the overlays.
