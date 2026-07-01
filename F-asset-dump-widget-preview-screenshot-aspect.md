---
id: F-asset-dump-widget-preview-screenshot-aspect
title: "asset.dump should optionally emit Widget Blueprint preview screenshots"
status: DONE
severity: Medium
category: feature
tags: [asset, dump, widget, screenshot]
---

# asset.dump should optionally emit Widget Blueprint preview screenshots

Widget Blueprint dumps are currently static text artifacts. They include `tree.xml`, `properties.json`, BPIR, and animation JSON when available, but no rendered preview image. That keeps the dump cheap, but it means visual review still needs a separate live designer screenshot step.

**Workaround:** Use the existing widget designer screenshot RPC for targeted visual checks.

**Fix:** Add an opt-in widget screenshot aspect for `asset.dump` / `asset.dump_folder` that reuses the safe designer screenshot path and records screenshot failures as diagnostics instead of failing the whole dump.

## History
- `#1-dump-audit-no-rendered-artifacts` `OPEN` reporter — Read-only audit found no `.png/.jpg/.jpeg/.bmp/.webp` files under `.editor-automation/asset-dumps`, even for dumped Widget Blueprints. Existing designer screenshot support writes to `Saved/Screenshots/WidgetDesigner`, not to each dump folder. If asset dumps are intended to be self-contained visual review artifacts, add an opt-in widget screenshot aspect that reuses `widget.screenshot_designer` safely and records failures as diagnostics instead of blocking the dump. Do not couple this to geometry resolution or the known offscreen crash path.
- `#2-opt-in-preview-png-aspect` `IN-REVIEW` developer — Implemented opt-in preview.png aspect for asset.dump and asset.dump_folder via new includeWidgetScreenshot param. Extracted safe Designer-tier capture path from WidgetDesignerScreenshotHandler.cpp into Handlers/UI/WidgetDesignerCaptureUtil; existing handler refactored to delegate (no behavior change). Added WriteAssetDumpBinaryFile helper to AssetDumpWriter for binary outputs. BuildAllFilesForAsset extended with binary-files out-param; DumpSingleAsset/StartAsyncFolderDump/FAsyncFolderDumpState/TickFolderDump all thread the new opt-in. Failures record per-asset diagnostics via RecordAspectDiagnostic; never block other aspects. Regression test TestAssetDumpWidgetScreenshot.cpp covers opt-in success, default-omits, and failure-records-diagnostic.
- `#3-verify-opt-in-and-default-omit` `DONE` tester — Verified: asset.dump on /GameLogsSystem/Common/WBP_GLS_EditableTextBox with includeWidgetScreenshot=true wrote preview.png (9341 bytes) alongside meta/properties/tree/bpir; asset.dump on /GameLogsSystem/Common/WBP_GLS_ButtonToggle_IconText without the flag emitted only 4 files (no preview.png). Wiki page for asset.dump documents the new includeWidgetScreenshot param.
