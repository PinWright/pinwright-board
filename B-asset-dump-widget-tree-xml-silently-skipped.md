---
id: B-asset-dump-widget-tree-xml-silently-skipped
title: "asset.dump silently skips tree.xml on ~5% of WidgetBlueprints (writes meta+properties+bpir but no tree.xml)"
status: DONE
severity: High
category: bug
tags: [asset-dump, widget, tree-xml]
---

# `asset.dump` silently skips `tree.xml` on some widgets

For 25 of 541 `WidgetBlueprint` dumps in a single sweep
(`pluginVersion: 0.2.0`, dumped 2026-05-01), the dump folder is
missing `tree.xml`. All 25 have valid `meta.json` (`assetType:
WidgetBlueprint`), substantial `properties.json` (11–41 KB), and a
`bpir.txt` — only the widget tree XML is missing. The dump did not
return an error and no `aspect_diagnostic.json` was written; the
file is just absent.

Examples:
- `App\App\UI\LobbyAndMenu\Minimap\W_Minimap\` (props 14 KB)
- `App\App\UI\LobbyAndMenu\TrackMenu\W_SingleRaceTrackEnd\` (props 11 KB)
- `Game\UI\Foundation\Dialogs\W_ConfirmationDefault\`

No correlation with widget complexity, parent class, or location.

Likely cause: the WidgetBlueprint branch in `AssetDumpHandler` calls
into the `widget.export_xml` writer, which throws or early-exits
silently on certain widget subclasses or named-slot configurations.
The handler swallows the exception and skips writing the file rather
than emitting a diagnostic aspect.

**Repro:** Run `asset.dump_folder("/App", { "recursive": true })` and
spot-check the listed examples — `tree.xml` is missing while every
other expected sibling file is present.

**Workaround:** Call `widget.export_xml` directly on the asset and
inspect the response; if it succeeds, the dump-side wrapper is the
problem.

**Fix (proposed):** Wrap the `tree.xml` writer in the same
diagnostic-fallback used by the MGIR aspect: on failure, write
`aspect_diagnostic.json` with the error rather than dropping the
file silently. Long-term, identify and fix the widget pattern
that triggers the silent skip.

## History
- `#1-initial-audit` `OPEN` reporter — 25 of 541 WidgetBlueprint dumps in a single sweep are missing `tree.xml` despite passing meta+properties+bpir. No diagnostic written. Same plugin version and dump session as the working ones, ruling out a versioning gap.
- `#2-tree-xml-empty-diagnostic` `IN-REVIEW` developer — Added `FWidgetTreeXmlResult` struct + `BuildWidgetTreeXmlWithDiagnostic` in WidgetXmlExporter.h/.cpp distinguishing null-WBP/null-WidgetTree/null-RootWidget cases. AssetDumpHandler.cpp WBP branch writes a marker `tree.xml` for `bEmptyByDesign=true` (no RootWidget) and calls `RecordAspectDiagnostic` for genuine anomalies (null WidgetTree). `BuildWidgetTreeXml` overload retained for back-compat. Tests in TestAssetDumpBuilder.cpp.
- `#3-verified-fix` `DONE` tester — Verified W_Minimap and W_ConfirmationDefault (both previously in the missing-25 list) now produce `tree.xml` on `asset.dump`. Both files contain the marker `<!-- empty: WidgetTree has no RootWidget on this WBP -->`, confirming the empty-by-design path writes a placeholder rather than silently skipping. File counts went from 3 (meta+properties+bpir) to 4 (+tree.xml).
