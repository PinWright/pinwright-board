---
id: B-landscape-nullrhi-refuses-non-edit-layer
title: "Under NullRHI landscape.sculpt and landscape.edit refuse every landscape, including non-edit-layer landscapes on UE 5.3-5.6 whose height writes need no renderer"
status: WONTFIX
severity: Low
category: bug
tags: [landscape, nullrhi, headless, rendering-guard, engine-versions, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T13:03:05Z
---

# Landscape renderer guard is broader than its reason

`B-headless-nullrhi-renderer-dependence` `#2` made `PinWrightRendering::RequireRenderer(Ctx)` the first
statement of `landscape.sculpt` (`Handlers/Environment/LandscapeHandler.cpp:964-968`) and
`landscape.edit` (`:1849-1853`). The stated reason in both comments is edit layers: "Edit-layer
landscapes merge heights on the GPU (ALandscape::CanUpdateLayersContent is false without a renderer),
so a write never reaches the heightmap this verb verifies." The guard does not check for edit layers,
so under `mode: headless` both verbs refuse every landscape.

That matches the engine only from UE 5.7: `ULandscapeInfo::CanHaveLayersContent()` returns `true`
unconditionally in 5.7 and 5.8 (`C:\UE_5.8\Engine\Source\Runtime\Landscape\Private\LandscapeEdit.cpp:5378-5381`).
On 5.3-5.6 it returns the landscape's `bCanHaveLayersContent` property (`C:\UE_5.6\...\LandscapeEdit.cpp:5684-5691`,
`LandscapeEditLayers.cpp:12188-12200`; property at `Landscape.h:668`, default `false`), and a landscape
without edit layers writes heights straight to the heightmap on the CPU, which is readable without a
renderer. On those engines the refusal is unnecessary for non-edit-layer landscapes.

Verified by engine source read on 5.3, 5.5, 5.6, 5.7 and 5.8; not run headless on a 5.3-5.6 host.

**Impact:** a headless agent on UE 5.3-5.6 cannot sculpt a legacy (non-edit-layer) landscape and is
told to switch to offscreen mode; a documented workaround exists (the refusal names the rendering
modes). Rare combination of old engine, headless mode and legacy landscape.
**Fix:** guard only when the target landscape can have layer content: resolve the landscape first,
then call `RequireRenderer` when `ULandscapeInfo::CanHaveLayersContent()` is true (always on 5.7+).
Update the landscape "No renderer (headless mode)" wiki section to say which landscapes are refused.

## History
- `#1-guard-broader-than-reason` `OPEN` reporter — Found in today's review of the NullRHI hardening. The candidate said 5.3-5.5; engine source shows non-edit-layer landscapes still exist on 5.6 (`bCanHaveLayersContent` present, `ULandscapeInfo::CanHaveLayersContent` reads it), and 5.7+ always report edit layers, so the affected range is 5.3-5.6.
- `#2-stale-sweep-yagni` `WONTFIX` developer — Still accurate at plugin HEAD `10212ee4`: `RequireRenderer` is the first statement of `landscape.sculpt` and `landscape.edit` (`Handlers/Environment/LandscapeHandler.cpp:966`, `:1851`). The case where that refusal is unnecessary needs three things at once: UE 5.3-5.6, headless mode and a non-edit-layer landscape. It was found by reading engine source, not by a caller hitting it. The refusal is clean and names the rendering modes to switch to, so the workaround is one restart in offscreen mode. Nothing is silently wrong.
