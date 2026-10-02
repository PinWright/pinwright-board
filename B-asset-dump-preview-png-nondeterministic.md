---
id: B-asset-dump-preview-png-nondeterministic
title: "preview.png capture size is timing-sensitive; no byte-compare skip, no aspect version"
status: WONTFIX
severity: Low
category: bug
tags: [asset-dump, widget-screenshot, determinism]
---

# preview.png capture size is timing-sensitive; no byte-compare skip, no aspect version

The opt-in widget screenshot sidecar (`includeWidgetScreenshot=true` on `asset.dump` / `asset.dump_folder`) is the one dump aspect with no determinism guarantee:

- `WidgetDesignerCaptureUtil.cpp` derives the capture output size from the Slate geometry settle loop in `WidgetDesignerCaptureInternal.h`. The settle loop is timing-sensitive, so the resolved geometry - and therefore the PNG dimensions/content - can differ run-to-run on the same unchanged asset.
- The write path goes through `WriteAssetDumpBinaryFile` with no byte-compare skip, so even an identical capture rewrites the file, and a merely-jittered capture produces a spurious content diff in a versioned dump root.
- `preview.png` has no entry in the `GetAspectVersion` table (`AssetDumpCache.cpp`), so a future capture-format change cannot invalidate stale cached previews the way every other aspect can.

Currently latent: `includeWidgetScreenshot` defaults to `false`, and folder sweeps bypass cache hits entirely when it is enabled, so nobody sees the churn today. It becomes real the moment screenshots are re-enabled for routine sweeps.

**Fix:** when screenshots are re-enabled - pin the capture size to the authored design-time size instead of the settled geometry, and/or add a byte-compare skip before `WriteAssetDumpBinaryFile`; add a `preview.png` row to the `GetAspectVersion` table in the same change.

## History
- `#1-filed-from-determinism-audit` `OPEN` reporter - Found during the dump-determinism audit: capture size comes from a timing-sensitive settle loop, the binary write path has no byte-compare skip, and preview.png is absent from the aspect-version table. Latent while includeWidgetScreenshot defaults off; fix when screenshots are re-enabled.
- `#2-stale-sweep-mostly-fixed` `WONTFIX` developer — Re-checked at plugin HEAD `10212ee4`: two of the three items are already in source. `AssetDumpWriter::WriteAssetDumpBinaryFile` (`Utils/AssetDumpWriter.cpp:704-716`) now loads the existing file and returns without writing when the bytes are identical, and `preview.png` has a row in `GetAspectVersion` (`Handlers/Asset/AssetDumpCache.cpp:865`, now at version 4) pinned by `TestAssetDumpPreviewAspectVersion.cpp`. The remaining item, capture size taken from the settled preview geometry (`Handlers/UI/WidgetDesignerCaptureUtil.cpp:437-447`, long axis normalised to `MaxSize`), is latent by the ticket's own account: `includeWidgetScreenshot` defaults off and widget-screenshot dumps bypass the cache at both ends (`AssetDumpCache.cpp:848-857`). No run-to-run churn has been observed; refile with a measured diff if screenshots are enabled for routine sweeps.
