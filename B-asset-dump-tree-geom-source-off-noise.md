---
id: B-asset-dump-tree-geom-source-off-noise
title: "asset.dump tree.xml emits Geom.source='off' on every node — redundant noise on every element"
status: DONE
severity: Low
category: bug
tags: [asset-dump, widget, tree-xml, noise]
---

# asset.dump tree.xml emits Geom.source='off' on every node — redundant noise on every element

All 545 `tree.xml` files have `Geom.source="off"` on every element (only sentinel value emitted). Asset-dump path never resolves geometry by design, so the attribute is redundant; emitting once on the root or as a header comment would suffice.

**Repro:**
1. Inspect any `tree.xml` under `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/`.
2. Observe: every element carries `Geom.source="off"`.

**Fix:** Drop the root-sentinel emit at `BuildWidgetTreeXmlWithDiagnostic` line ~399 (`RootGeomAttrs.Add("Geom.source", "off")`). The per-element gate on `Geom.bEnabled` in `BuildXmlString` already correctly suppresses geometry attributes on descendants — the only leak was the root sentinel. Live mode's source-value domain (`designer|live|offscreen|auto|ambiguous`) never overlaps `"off"`, so absence-means-disabled is unambiguous. `WidgetXmlImportHandler` already silently skips `Geom.*` attributes, so round-trip is unaffected.

## History
- `#1-initial-repro` `OPEN` reporter — Every element in every dumped `tree.xml` (545 files) carries the sentinel `Geom.source="off"`. Asset-dump path never resolves geometry by design; the attribute is pure noise on every node.
- `#2-drop-root-geom-sentinel` `IN-REVIEW` developer — Deleted root-sentinel `Geom.source="off"` emit at `WidgetXmlExporter.cpp` (`BuildWidgetTreeXmlWithDiagnostic`). Per-element gating already suppressed geom attrs on descendants; the leak was the unconditional seed of `RootGeomAttrs`. Added regression test `FXmlAssetDumpExportNoGeomSourceOffTest` asserting the asset-dump XML contains no `Geom.source=` attribute.
- `#3-verify-fix` `DONE` tester — Verified: re-dumped `/App/Blueprints/UI/W_ActionHotkey.W_ActionHotkey` via `asset.dump`; fresh `tree.xml` (root `<CanvasPanel name="CanvasPanel_0" IsVariable="true">`) contains zero `Geom.source` matches, whereas the prior stale cache file had `Geom.source="off"` on the root element at line 3.
