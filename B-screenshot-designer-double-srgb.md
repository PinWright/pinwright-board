---
id: B-screenshot-designer-double-srgb
title: "`widget.screenshot_designer` preview PNGs are sRGB-encoded twice; every colour reads wrong"
status: OPEN
severity: High
category: bug
tags: [render, capture, gamma, wrong-data-that-looks-correct]
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
