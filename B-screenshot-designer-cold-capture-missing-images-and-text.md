---
id: B-screenshot-designer-cold-capture-missing-images-and-text
title: "widget.screenshot_designer target:preview on a cold widget returns success with images and text missing that a second capture of the same widget draws"
status: DONE
severity: Medium
category: bug
tags: [widget, screenshot-designer, umg, cold-frame, texture-streaming, false-evidence]
encounters: 1
lastSeen: 2026-10-01T11:43:00Z
---

# A cold Designer preview capture is incomplete but reports the same success shape as a complete one

UE 5.8, Linux, wt2 checkout, offscreen editor. `widget.screenshot_designer {widgetPath:
/App/App/UI/LobbyAndMenu/W_LyraFrontEnd, max_size: 1400}` as the first capture of that widget in the
session returned success, 137510 bytes, `alphaZeroFraction` 0.671. The same call about 40 s later
(Designer closed in between, nothing edited) returned 251059 bytes, `alphaZeroFraction` 0.591. The
first PNG is missing the top-left logo image (PDS / GEOSCAN), the weekly-tournament panel's
background image, its title text, its "tournament is being prepared" text and the countdown digits.
The second has all of them. A third capture with the Designer already open matched the second
(251037 bytes).

Both responses have the same fields, so the caller cannot tell an incomplete frame from a complete
one. It looks like the cold-frame class closed for thumbnails in `B-thumbnail-cold-first-frame-no-stats`:
textures and font faces that are still streaming or lazily loading when the single `FWidgetRenderer`
draw runs. Not yet root-caused.

**Workaround:** capture twice and keep the second, or `editor.open_asset` and let the Designer tick
before capturing.

## History
- `#1-found-during-hang-repro` `OPEN` developer — Found while trying to reproduce
  `B-screenshot-designer-hangs-game-thread` (`#5` there). Files:
  `Saved/Screenshots/WidgetDesigner/repro_frontend_cold.png` (incomplete) and `repro_par_c.png`
  (complete) in the wt2 host checkout.
- `#2-wait-for-render-assets` `IN-REVIEW` developer — Root-caused from the two repro PNGs and the asset dumps: the cold frame drew the trophy icon as the engine's grey checker (texture still compiling) and dropped the logo texture, and the missing tournament texts are exactly the `CommonTextBlock`s whose `FontMaterial` is `M_UI_PDSTextGradient` / `MI_UI_PDSTextGradient_Red` (no game-thread shader map yet; the texts without a font material drew). Fonts themselves load synchronously in the editor (`Slate.Font.AsyncLazyLoad` is disallowed in Editor). Fix in `WidgetDesignerCaptureUtil::CapturePreviewToPng`: before the single `FWidgetRenderer` draw it collects every `UTexture` / `UMaterialInterface` the preview tree references (new exported `CollectRenderAssets`: `FReferenceFinder` over the preview and every nested object — brushes, font materials, atlas sprites — plus `GetUsedTextures` of each material), submits incomplete shader maps at high priority, and pumps `PinWright::AssetCompile::AdvanceOnGameThread` until all textures stopped compiling and all shader maps are complete, bounded by `WidgetDesignerCaptureInternal::ReadinessBudgetSeconds()` (20 s); ready textures then `WaitForStreaming`. New response fields on `target:preview`: `previewComplete` and `readiness {textures, materials, waitedSeconds, notReady[]}`; `notReady` names what was still compiling when the budget ran out, so an incomplete frame is never a silent success. `asset.dump`'s `preview.png` uses the same path and now lists an incomplete preview under `skipped` (file still written, cache baseline not recorded). Not covered: resources reached only through a style class CDO or loaded async through a soft pointer (e.g. `CommonLazyImage`). Files: `Source/PinWright/Private/Handlers/UI/WidgetDesignerCaptureUtil.{h,cpp}`, `WidgetDesignerCaptureInternal.h`, `WidgetDesignerScreenshotHandler.cpp`, `Handlers/Asset/AssetDumpHandler.cpp`, `docs/wiki-src/widget.md`, `CHANGELOG.md`. Tests (`Tests/Widget/TestWidgetDesignerColdCapture.cpp`): `PinWright.widget.screenshot_designer.ColdTextureIsDrawnFinished`, `PinWright.widget.screenshot_designer.ExpiredReadinessBudgetReportsIncomplete`, `PinWright.widget.screenshot_designer.ReadinessCollectsBrushTexturesAndFontMaterials`.
- `#3-verified-linux` `DONE` tester — PinWright `10212ee4` (on origin/master `6283b63b`), UE 5.8 Linux Vulkan. Runs: w23-final = offscreen full suite, no DISPLAY, 5568/5568 passed; w23-xfinal = DISPLAY=:0 offscreen, drive.os_input+click_occlusion+os_gesture+input, 31/31; w23-vis = DISPLAY=:0 windowed drive.input.ModifierChord, 2/2; Python = Content/Python/tests, 427 OK / 5 skipped (all skips Windows-only or an absent local script). Passed in w23-final: `PinWright.widget.screenshot_designer.ColdTextureIsDrawnFinished` (a still-compiling texture is waited for and drawn finished), `.ExpiredReadinessBudgetReportsIncomplete` (budget expiry -> `previewComplete:false` with `readiness.notReady[]`) and `.ReadinessCollectsBrushTexturesAndFontMaterials`; all 26 `widget.screenshot_designer.*` tests passed. The title defect (an incomplete cold capture had the same success shape as a complete one) is fixed: the response carries `previewComplete` and `readiness`. Limits: the #1 `W_LyraFrontEnd` cold capture was not re-run; resources reached only through a style-class CDO or an async soft pointer (e.g. `CommonLazyImage`) are not collected, so such a frame can still be incomplete without `notReady` naming it (documented).
