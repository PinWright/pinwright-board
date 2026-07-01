---
id: B-widget-export-xml-no-inherited-root
title: "`widget.export_xml` returns TREE_EMPTY for child WBPs that inherit the root from a parent WBP"
status: DONE
severity: Medium
category: bug
tags: [widget-xml, inheritance, asset-dump-divergence]
---

# `widget.export_xml` returns TREE_EMPTY for child WBPs that inherit the root from a parent WBP

The live `widget.export_xml` RPC dereferences `WidgetBP->WidgetTree->RootWidget` directly (`WidgetXmlExportHandler.cpp:137`). When that field is `nullptr` — the normal representation of a child WBP whose tree is inherited from its parent — the handler bails with `TREE_EMPTY` ("Widget tree has no root widget"). This is a divergence from the asset-dump path: `AssetDumpHandler.cpp:1597` calls `WidgetXmlExporter::BuildWidgetTreeXmlWithDiagnostic`, which walks up the BPGC chain via `FindWidgetTreeOwningClass()` (`WidgetXmlExporter.cpp:362-389`), starts the export from the inherited root, and tags it with `inherited_from="..."` plus an HTML comment. So the dump emits a usable `tree.xml` while the live RPC errors out on the same asset.

This violates the project policy that asset dumps must not have exclusive functionality — every dump output should be reachable via a live RPC.

**Workaround:** read the cached `tree.xml` from `.editor-automation/asset-dumps/...` if it exists.

**Fix:** share the inherited-root resolution logic from `WidgetXmlExporter` through a small helper/result, then have both the dump exporter and the live `widget.export_xml` handler use that resolver for full-tree export. Preserve the live handler's geometry-aware XML generation and `widget_count` response shape; map `bEmptyByDesign=true` with no start widget to a clean empty result, and report `Reason` on real failures. No new RPC, no schema change. Subtree-root export (`widget_name` param) keeps its existing `FindWidget` path since that already searches the BP's own tree only.

## History
- `#1-initial-repro` `OPEN` reporter — Live `widget.export_xml` returns `TREE_EMPTY` on child WBPs whose `WidgetTree->RootWidget` is null because the tree is inherited from a parent WBP. The asset-dump path (AssetDumpHandler.cpp:1597 → BuildWidgetTreeXmlWithDiagnostic) handles this case by walking the BPGC chain and emits a `tree.xml` annotated with `inherited_from=`. Wire WidgetXmlExportHandler through the same exporter helper to close the gap.
- `#2-shared-inherited-root` `IN-REVIEW` developer — Changed WidgetXmlExporter.h, WidgetXmlExporter.cpp, WidgetXmlExportHandler.cpp, and TestWidgetXmlHandlers.cpp to expose a shared inherited-root resolver, preserve the live handler's geometry-aware XML generation and widget_count response, emit inherited_from/comment metadata for inherited roots, and add the FXmlExportInheritedRootHandlerTest regression.
- `#3-verify-inherited-root` `DONE` tester — Verified: `widget.export_xml` on `/Game/UI/Foundation/Dialogs/W_ConfirmationError` (child WBP with null `WidgetTree->RootWidget` inheriting from `W_ConfirmationDialog`) returned full XML with leading `<!-- inherited from /Game/UI/Foundation/Dialogs/W_ConfirmationDialog.W_ConfirmationDialog -->` comment, root `<Overlay name="Overlay_1" inherited_from="..."` attribute, and `widget_count: 15`. No TREE_EMPTY error.
