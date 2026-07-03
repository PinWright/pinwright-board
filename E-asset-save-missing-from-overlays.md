---
id: E-asset-save-missing-from-overlays
title: "safe-mutation-save recipe omits asset.save; agents fall back to blunt editor.save_all"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [asset, save, wiki, overlay, discoverability]
---

# safe-mutation-save recipe omits asset.save

`asset.save {assetPath, force?}` is the correct narrow single-package writer
(registered `AssetSaveHandler.cpp:34`), but the canonical save-recipe overlay does
not offer it as an option:

- `docs/wiki-src/safe-mutation-save.md` — recipe step 5 and the "Choosing The
  Writer" table name only `editor.save_all` / `level.save` / `level.save_as`, with
  no single-asset row. This contradicts step 2's own "pick the narrowest writer"
  rule: the save step jumps straight to the broadest writers.

Scope correction (the original report over-stated the invisibility of `asset.save`):
`asset.save` is NOT absent from the served asset surface. The `asset` namespace page
auto-emits a `## Methods` index from the registry (`WikiHandler::RenderMethodList`,
`WikiHandler.cpp:353`), and because `asset.save` is registered under category
`asset` it appears there with its macro summary; `call("asset.save")` likewise
renders a full method page. It is also named on `property.md:42` ("call
`editor.save_all` / `asset.save`...") and `blueprint.md:384` (`[asset.save](asset.md)`
— a whole-page link, no anchor fragment, that lands on the asset page which lists
it). So the original "only `blueprint.md` names it" and "dangling cross-ref" claims
were wrong.

The genuine, narrower gap: the `safe-mutation-save` TOPIC page (a topic page carries
no auto `## Methods` index) never mentioned `asset.save`, and `asset.md` had no
hand-authored `### asset.save` H3 to enrich the method page beyond the one-line
macro summary.

Consequence: an agent reading the save recipe sees no single-asset option and uses
`editor.save_all`, which flushes EVERY dirty package — persisting unrelated
incidentally-dirtied assets alongside the intended one. `asset.save` would have
persisted only the one package. Soft friction with a working fallback (hence Low —
docs/discoverability per the board rubric).

**Fix:** Add `asset.save` to the safe-mutation-save recipe step 5 and the "Choosing
The Writer" table as the targeted single-package alternative to `editor.save_all`,
and add a concise `### asset.save` H3 to `asset.md` (recipe cross-link + the
`saved`/`pendingFlush`/`integrityGate` readback contract) so the `call("asset.save")`
method page carries a Notes section. This is method-page enrichment, not a
broken-link fix — the cross-refs already resolve via the auto Methods index. Impl of
the RPC itself is `F-asset-save` (IN-REVIEW); this ticket is the overlay-docs
follow-up only.

## History
- `#1-initial-repro` `OPEN` reporter — Confirmed: grep for `asset.save` across `docs/wiki-src/` matched only `blueprint.md:384` (a `[asset.save](asset.md)` link to a page with no such anchor); `safe-mutation-save.md` (step 5 lines 29-32, table lines 44-45) and `asset.md` (no save section) omit it entirely. Handler exists at `AssetSaveHandler.cpp:34`. Session evidence: lacking any single-asset save on those pages, used `editor.save_all {}` → `{savedCount:2, totalDirty:2}`; git showed the intended `PM_Drone.uasset` AND unrelated `W_DroneSelect_EditDrone.uasset` modified. `asset.save {assetPath}` would have persisted only `PM_Drone`. Separate from `F-asset-save` (impl ticket, IN-REVIEW) whose Fix/History cover the handler + test only, not the overlays.
- `#2-reworded-and-implemented` `IN-REVIEW` developer — Reworded: corrected two false claims (verified against source) — `property.md:42` also names `asset.save`, and the `blueprint.md:384` link is a whole-page ref (no anchor) that resolves because the `asset` namespace page auto-lists `asset.save` via `WikiHandler::RenderMethodList` (`WikiHandler.cpp:353`), so it is not "dangling"; lowered severity Medium→Low (docs/discoverability per board rubric). Implemented the real gap: added `asset.save` to `safe-mutation-save.md` recipe step 5 + the "Choosing The Writer" table as the narrow single-package writer, and a `### asset.save` H3 to `asset.md` (recipe cross-link + `saved`/`pendingFlush`/`integrityGate` readback) enriching the `call("asset.save")` method page. Files: `Plugins/PinWright/Docs/wiki-src/safe-mutation-save.md`, `Plugins/PinWright/Docs/wiki-src/asset.md`. Regression test `PinWright.infra.wiki_handler.TopicPage.SafeMutationSaveAssetSave` (`Tests/Infra/TestSafeMutationSaveAssetSaveDocs.cpp`) renders the live safe-mutation-save topic page and asset.save method page via `WikiHandler::RenderPage` and asserts `asset.save` + the narrow-alternative framing + the method-page recipe cross-link; reverting the overlay edits makes it fail.
