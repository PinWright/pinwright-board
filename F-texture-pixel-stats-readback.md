---
id: F-texture-pixel-stats-readback
title: "No live read returns texture pixel content/stats — desaturate / invert / adjust_levels / channel_pack success is unverifiable except by inferring from re-encoded sizeBytes; needs a pixel-stats (channel mean/min/max, grayscale) readback"
status: IN-REVIEW
severity: Medium
category: feature
tags: [texture, desaturate, invert, adjust_levels, channel_pack, combine_textures, readback, pixel-stats, grayscale, verify, describe, get_texture_info]
encounters: 1
lastSeen: 2026-06-25T07:14:35Z
---

# The pixel-mutating texture verbs have no content readback — you cannot confirm a desaturate actually produced grayscale

The `texture.*` family includes several verbs that **rewrite the actual image
pixels** — `desaturate`, `invert`, `adjust_levels`, `channel_pack`,
`combine_textures` — yet every live texture read RPC returns only *metadata*:

- `texture.describe` / the `texture.json` sidecar → `kind`, `textureClass`,
  `size`, `pixelFormat`, `compressionSettings`, `lodGroup`, `srgb`,
  `mipGenSettings`, `neverStream`, `source` (the field set enumerated in
  `B-asset-dump-texture2d-duplicate-sidecars`).
- `texture.get_texture_info` → `width`, `height`, `format`, `mipCount`, `sRGB`,
  `lodBias`, `compression`.

None of these expose anything about the **pixel data**. So after calling
`desaturate` (intended: turn an RGB diffuse into a pure grayscale mask),
`adjust_levels` (intended: push contrast), or `invert` (intended: flip to a
cavity mask), the caller has **no way to read back whether the pixels actually
changed in the intended direction** — whether the result is in fact grayscale,
whether contrast moved, whether the image is the photographic negative of the
source. The dimensions/format readback (already messy — see
`B-texture-describe-size-resident-mip`) confirms only that the *envelope* round-
tripped, never the *content*.

The only available signal is **indirect and unreliable**: the verb returns a
bare success string, and after a force-save the re-encoded package's
`sizeBytes` shifts slightly (the audited task saw 5.86 / 6.06 / 6.07 MB across
the three saves). A caller is reduced to *inferring* "the desaturate worked"
from (a) the call not erroring and (b) the file size moving — neither of which
proves the image is grayscale, let alone correct. A no-op handler that
re-encoded the unchanged image would produce the same success string and a
similarly-perturbed `sizeBytes`, so the inference cannot distinguish a real
transform from a silent no-op (cf. the silent-no-op class:
`B-texture-create-placeholder-fake-success`, `B-widget-apply-style-silent-noop`).

## Why this is friction (not just normal granularity)

"Derive a reusable grayscale mask from a diffuse, then an inverted cavity
variant" is a canonical texture-authoring intent, and step 7 of the user's
story was *explicitly* "read back both ... and confirm." The metadata reads
answer the dimensions question but are structurally incapable of answering the
content question the verbs exist to satisfy. Every pixel-mutating verb in the
family inherits the gap, so it is not a one-off.

This is distinct from the already-filed texture readback tickets:
- `B-texture-describe-size-resident-mip` (judge-filed) — `size`/`pixelFormat`
  report the resident mip; a *metadata-correctness* bug.
- `E-texture-describe-omits-lodbias-wrap` — `describe` omits the *sampling-
  settings* fields (`lodBias`/wrap/`filter`/VT) the `set_*` setters write; a
  *settings* readback gap.
- `F-texture-sampling-settings-batch` — batch *write* of those settings.

All three concern metadata/settings. **None** provides a read of the *image
content*, which is the only thing that verifies the pixel-mutating verbs.

## What it should do

Add a live pixel-statistics readback so the set/verify round-trip closes for the
image-editing verbs, e.g. `texture.get_pixel_stats(assetPath, mip?)` (or fold a
`stats` block into `texture.describe`) returning cheap, content-revealing
aggregates computed from the source mip:

- per-channel `mean` / `min` / `max` (R,G,B,A),
- a `grayscale` boolean (or a max per-pixel channel spread) so a desaturate is
  verifiable as "R==G==B everywhere",
- optionally a content hash of the source mip so two reads can be compared for
  "did the pixels change at all" and an invert can be checked against the
  pre-invert hash.

These are computable from `Texture->Source.LockMip(0)` (the source is already
read for the `source{x,y}` block today) without decoding platform data, so no
streaming dependency. With a `grayscale` flag plus channel means, `desaturate`,
`invert` (means flip toward `255-mean`), and `adjust_levels` (min/max spread
widens) all become directly verifiable instead of inferred from `sizeBytes`.

**Workaround:** none that proves content. The audited task fell back to
inferring grayscale from each verb's success string plus the per-save re-encoded
`sizeBytes` deltas (5.86 / 6.06 / 6.07 MB) — an indirect signal that cannot
distinguish a correct transform from a silent no-op.

## Friction evidence (this task — `texture.desaturate` leather-mask round trip, outcome tool_bug/clean)

Story: duplicate `T_Leather` → `T_Leather_Mask`, `desaturate` + `adjust_levels`
it to a grayscale mask, duplicate again → `T_Leather_Mask_Inv` and `invert` it,
then "read back both ... and confirm ... intact." Self-report: "Built
T_Leather_Mask (desaturate+adjust_levels) and T_Leather_Mask_Inv (invert) ...
all three 2048x2048 PF_DXT1 sRGB; 23 calls all ok." Friction note, verbatim:
**"no pixel/hash verb so grayscale inferred from success + re-encoded sizeBytes
per force-save (5.86/6.06/6.07MB)."** Every call succeeded (no retries, no
errors, no `python.execute` fallback) — the gap is the *unverifiability* of the
pixel transforms, not any failure. The metadata readbacks
(`describe`+`get_texture_info`) confirmed only resolution/format, never that the
desaturate/invert/adjust_levels did what the user asked.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `texture.desaturate`
  leather-mask task (namespace `texture`, outcome `tool_bug`; judge filed the
  resident-mip metadata bug `B-texture-describe-size-resident-mip`). Distinct
  PROCESS angle: the pixel-mutating texture verbs (`desaturate`, `invert`,
  `adjust_levels`, `channel_pack`, `combine_textures`) have **no live content
  readback** — `texture.describe` and `get_texture_info` return only metadata, so
  the agent could not confirm the desaturate produced grayscale and fell back to
  inferring success from each verb's bare success string plus the re-encoded
  `sizeBytes` per force-save (5.86/6.06/6.07 MB) — an indirect signal that cannot
  distinguish a real transform from a silent no-op. 23 calls, all ok, no retries/
  errors/fallbacks; pure unverifiability, not a failure. Proposed: add a live
  pixel-stats readback (`texture.get_pixel_stats` or a `stats` block on
  `describe`) returning per-channel mean/min/max, a `grayscale` flag, and an
  optional source-mip hash, computed from `Source.LockMip(0)` (no streaming dep)
  so desaturate/invert/adjust_levels become directly verifiable. Dedup: ripgrep
  across OPEN+closed — no texture pixel/stats/histogram/content-readback ticket
  exists; `B-texture-describe-size-resident-mip` (metadata correctness),
  `E-texture-describe-omits-lodbias-wrap` (settings readback), and
  `F-texture-sampling-settings-batch` (settings batch write) all concern
  metadata/settings, none the image content this ticket covers.
- `#2-add-get-pixel-stats` `IN-REVIEW` developer — Added a live source-mip
  pixel-content readback so the pixel-mutating verbs are verifiable instead of
  inferred from `sizeBytes`. New `texture.get_pixel_stats(assetPath, mip?)` RPC
  (registered in `Source/PinWright/Private/Handlers/Material/TextureHandler.cpp`,
  next to `texture.describe`) loads the `UTexture` and delegates to a new
  exported helper `TexturePixelStats::BuildPixelStatsJson` in
  `Source/PinWright/Private/Handlers/Asset/TexturePixelStats.{h,cpp}`. The helper
  reads the editable source mip via `FTextureSource::LockMipReadOnly` (the same
  access the mutating verbs already use — no streaming/platform-decode dep) and
  returns per-channel `mean`/`min`/`max` (R,G,B,A), a `grayscale` boolean plus
  `maxChannelSpread`, and a CRC32 `hash` of the source mip (for pre/post "did
  the pixels change" comparison). It branches on `Source.GetFormat()`: `TSF_BGRA8`
  (B,G,R,A byte order, the desaturate/invert output) and single-channel `TSF_G8`
  are supported; any other source format fails loud with
  `PIXEL_STATS_UNAVAILABLE` rather than misreading the layout. With `grayscale` +
  channel means, `desaturate` (R==G==B), `invert` (means flip toward 255-mean),
  and `adjust_levels` (min/max spread widens) all become directly verifiable.
  Scope trim per the adversarial lens: only the verbs' own source formats are
  read; wider formats (RGBA16/16F/G16) are explicitly rejected, not faked.
  Regression test:
  `Source/PinWright/Private/Tests/Assets/TestTexturePixelStats.cpp`
  (`PinWright.texture.get_pixel_stats.SourceMipStats`) builds transient
  `UTexture2D`s with known BGRA8 source bytes and asserts the BGRA decode order,
  channel mean/min/max, `grayscale` true/false, `maxChannelSpread`, distinct
  hashes for distinct images, and that a `TSF_RGBA16F` source is rejected;
  reverting the fix removes the helper so the test fails to compile/link.
