---
id: B-screenshot-designer-double-srgb
title: "`widget.screenshot_designer` preview PNGs are sRGB-encoded twice; every colour reads wrong"
status: DONE
severity: High
category: bug
tags: [render, capture, gamma, wrong-data-that-looks-correct]
encounters: 1
lastSeen: 2026-08-28
---

# `widget.screenshot_designer` preview PNGs are sRGB-encoded twice

Reproduced live, UE 5.8, plugin `d195a55d`. A scratch `Border` tinted with the linear value that
renders as `#254D67` — `(0.018501, 0.074214, 0.135727)` — captured with `target:"preview"`:

```
(128,72)  -> (106, 149, 170)
(64,40)   -> (106, 149, 170)
(200,110) -> (106, 149, 170)
```

Expected `(37, 77, 103)`. `(106,149,170)` is exactly `sRGB_encode((37,77,103)/255)` — the transfer
function applied one extra time. Decoding once returns `(36.75, 76.64, 102.50)`. Pure gamma-2.2 does
not fit (it gives `(106,148,169)`), so the extra transform is specifically the piecewise sRGB curve.

Consequence: any colour sampled from a designer preview is wrong, and a correct material or brush
looks washed out. Callers who compare a sampled pixel against a design value will chase a phantom.

## Mechanism — two encodes, two flags, one line pair

`Handlers/UI/WidgetDesignerCaptureUtil.cpp:186-194`:

```cpp
TSharedPtr<FWidgetRenderer> WidgetRenderer = MakeShared<FWidgetRenderer>(/*bUseGammaCorrection=*/true);
TStrongObjectPtr<UTextureRenderTarget2D> RenderTarget(FWidgetRenderer::CreateTargetFor(
    FVector2D(PreviewDrawSize.X, PreviewDrawSize.Y), TF_Bilinear, /*bUseGammaCorrection=*/true));
```

1. **Shader encode.** `bUseGammaCorrection=true` reaches `Slate3DRenderer.cpp:182`
   (`bAllowGammaCorrection`) -> `SlateRHIRenderingPolicy.cpp:1508-1509` -> `SlateElementPixelShader.usf:87-97`
   -> `GammaCorrectionCommon.ush:65-78` `LinearToSrgb`. Encode #1.
2. **Hardware encode.** `CreateTargetFor(..., true)` sets `bIsLinearSpace = false`
   (`WidgetRenderer.cpp:81-94`), so `bForceLinearGamma` is false and `UTextureRenderTarget2D::IsSRGB()`
   returns true (`TextureRenderTarget2D.cpp:72-87`, `OverrideFormat == PF_B8G8R8A8` takes the `else`
   branch). That sets `TexCreate_SRGB` (`:492-495`) and the RTV format becomes
   `B8G8R8A8_UNORM_SRGB` (`D3D12Texture.cpp:1677-1681,1722`). The ROP encodes again.

Nothing undoes it: `ReadPixels` on an 8-bit BGRA target is a byte copy, and
`FImageUtils::PNGCompressImageArray` wraps the bytes as `ERawImageFormat::BGRA8` and compresses
verbatim. There is no `ToFColor` anywhere in the plugin — the bug is entirely these two flags.

## Only this path is wrong

| Path | Encodes |
|---|---|
| `screenshot_designer` **preview** (`WidgetDesignerCaptureUtil.cpp:189,194`) | **2 — bug** |
| `screenshot_designer` **window** (`WidgetDesignerScreenshotHandler.cpp:307`) | 1 |
| `editor.screenshot_window` (`EditorWindowHandlers.cpp:910`) | 1 |
| viewport captures (`PreviewViewportCaptureUtils.cpp:2023`) | 1 |
| ortho tiles (`OrthoTileCaptureUtils.cpp:564-569`) | 1 |

The engine's own widget-to-texture consumer uses the opposite pairing —
`Runtime/UMG/Private/Components/WidgetComponent.cpp:631` `bApplyGammaCorrection(false)` with hardware
sRGB — which is the shape the fix should take. Note `CreateTargetFor(..., false)` would flip
`bForceLinearGamma` to true and yield raw linear bytes instead, so the fix needs a hand-built RT.

Why the engine gets away with `FWidgetRenderer(true)` generally: the double-encoded target is normally
*sampled* through an sRGB SRV, which decodes one of the two encodes back out. A raw `ReadPixels` -> PNG
bypasses that compensation.

## Blast radius

`asset.dump` uses the same helper for `preview.png` (`Handlers/Asset/AssetDumpHandler.cpp:739`), so
every committed widget preview in an asset-dump mirror is double-encoded too.

## Notes

The in-source comment at `WidgetDesignerCaptureUtil.cpp:186-188` claims the config "matches what
`FSlateApplication::TakeScreenshot` returns". It does not — that path encodes once. That false claim is
likely why the double flag has read as deliberate.

Secondary: batches flagged `ESlateBatchDrawFlag::NoGamma` skip encode #1
(`SlateRHIRenderingPolicy.cpp:1111`), so a widget mixing normal brushes with NoGamma content comes out
with two different gamma treatments in one PNG.

## Coverage gap

No test in the capture suite asserts a rendered pixel value. `Tests/Widget/TestWidgetDesignerCaptureAlpha.cpp`
covers alpha only, and its closest test (`FWidgetDesignerStampLeavesRgbTest`) asserts RGB is unchanged by
the alpha stamp over a *synthesized* `TArray<FColor>`, never touching `FWidgetRenderer` or the render
target. A test that renders a known solid brush colour and asserts the decoded PNG pixel is what should
land with the fix.

Nothing in the plugin docs mentions gamma or sRGB — zero hits for either across the whole `Docs/` tree,
including the `widget.screenshot_designer` wiki section (`wiki-src/widget.md:506-524`).

## History

- `#1-reported-with-live-repro` `OPEN` reporter — Confirmed at runtime on `d195a55d` / UE 5.8 with an independently computed linear tint; sampled `(106,149,170)` against an expected `(37,77,103)`, matching `sRGB_encode` applied twice. Both flags and the full engine encode chain verified in source.
- `#2-single-srgb-encode-on-preview-path` `IN-REVIEW` developer — Removed the shader-side encode and kept the hardware one: the renderer/render-target pair moved out of `CapturePreviewToPng` into a new `WidgetDesignerCaptureUtil::RenderSlateWidgetToSrgbColors` (the only widget→pixels route in that TU), which now builds `FWidgetRenderer(/*bUseGammaCorrection=*/false)` over the unchanged `CreateTargetFor(..., true)` hardware-sRGB target — the same pairing `UWidgetComponent` and `OrthoTileCaptureUtils` use. No inverse curve, no linear-bytes flip; `NoGamma` batches now land in the same output space as the rest of the frame. Files: `Handlers/UI/WidgetDesignerCaptureUtil.h/.cpp`; `Handlers/Asset/AssetDumpCache.cpp` (`preview.png` aspect 2→3, every dumped preview's bytes changed) and its pinned literal in `Tests/Utility/TestAssetDumpPreviewAspectVersion.cpp`. Tests added in `Tests/Widget/TestWidgetDesignerCaptureGamma.cpp`: `PinWright.widget.screenshot_designer.PreviewEncodesSrgbExactlyOnce` renders three known linear tints through the shipped helper and asserts the read-back bytes match `ToFColor(bSRGB=true)` within 3 counts and that zero pixels carry the double-encoded value (the two references are asserted separable first, and coverage asserted, so neither can pass vacuously); `PinWright.widget.screenshot_designer.RenderRejectsZeroDrawSize` pins the degenerate-size refusal that `InitCustomFormat`'s `check()` would otherwise turn into a suite-host crash. Not compiled or run — orchestrator owns the build.

- `#3-measured-single-encode-on-live-editor` `DONE` verifier — 2026-08-28. Ran `#1`'s quantitative repro against the live editor on plugin `b79ba53e`, UE 5.8, and decoded the PNG with a stdlib zlib/struct decoder (no PIL). Scratch widget `/Game/PinWrightScratch/WBP_PwVerifyGamma`: two `UBorder`s split 50/50 by anchors — `ColA` `BrushColor` = the ticket's linear `(0.018501, 0.074214, 0.135727)`, `ColB` = an independent second colour `(0.5, 0.25, 0.75)` so the result proves a mapping and not one lucky sample. `widget.screenshot_designer {target:"preview", max_size:256}` → 256×144, `renderer:"widgetRenderer"`, `opaqueStamped:true`. **Measured, exact, no tolerance needed:** ColA = **(37, 77, 103)** on 48.5% of pixels (`#1` measured `(106,149,170)`); ColB = **(188, 137, 225)**. Computed references: encode-once `(37,77,103)` / `(188,137,225)`; encode-twice `(106,149,170)` / `(223,194,241)`. Every measured channel lands on encode-once to the byte, and a full histogram of the image (10 distinct colours) contains **zero** pixels of either double-encoded value — the two references are far apart (35–69 counts per channel), so the match is not coincidence. Looked at the image, not only the numbers: left half renders as the dark slate blue `#254D67` the ticket names, right half a mid purple; neither is washed out. **Blast radius checked:** `asset.dump {includeWidgetScreenshot:true}` on the same widget wrote a 1024×576 `preview.png` whose histogram is the same two colours — `(37,77,103)` and `(188,137,225)` at 49.6% each — so the shared helper is single-encoded on that path too. Automation suite deliberately not run; all evidence is decoded pixels off the live editor.
