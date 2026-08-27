---
id: B-noise-texture-hdr-writes-bgra8
title: "texture.create_noise_texture hdr:true initialises a TSF_RGBA16F source but the pixel loop still writes BGRA8 bytes"
status: IN-REVIEW
severity: High
category: bug
tags: [texture, create_noise_texture, hdr, TSF_RGBA16F, wrong-pixels, format-mismatch]
encounters: 1
lastSeen: 2026-08-27
---

# The HDR path allocates one format and fills it with another

`texture.create_noise_texture` with `hdr: true` initialises the texture source as `TSF_RGBA16F`, but
the pixel loop that fills it still writes 8-bit BGRA bytes. The buffer is therefore interpreted as
half-float data that was never written as half-float.

Found while fixing the two noise defects in the same function
(`B-noise-texture-noisetype-ignored`, `B-noise-texture-seamless-lattice`) and deliberately left
alone as out of scope.

**Not reproduced** -- the claim is a source reading of the format passed to the source init against
the writes in the fill loop. Someone should confirm what the resulting texture actually looks like
before deciding whether the fix is to write half-floats or to stop advertising `hdr`.

## History
- `#1-found-in-the-same-function` `OPEN` reporter -- Recorded by the agent fixing the two
  `create_noise_texture` defects. Source-level claim; out of that ticket's scope.
- `#2-fill-loop-writes-half-floats` `IN-REVIEW` developer -- Confirmed from source: `TSF_RGBA16F` is
  8 bytes/pixel (`ERawImageFormat::GetBytesPerPixel`), so the 4-byte BGRA fill wrote only the first
  half of the buffer and every written pair of bytes was reinterpreted as one half-float -- garbage
  colour over a half-unwritten image, with alpha built from a neighbouring pixel's bytes. Fixed by
  writing half-floats rather than dropping `hdr`: the fill loop in
  `Handlers/Material/TextureHandler.cpp` (`create_noise_texture`) now branches on the format the
  source actually carries (`Source.GetFormat() == TSF_RGBA16F`) and writes four `FFloat16` channels
  in R,G,B,A order (linear, `SRGB` is off on that path), byte path unchanged. The response echoes
  `hdr` read off the source, and the `hdr` param description now says RGBA16F/half-float. New test
  `PinWright.texture.create_noise_texture.HdrWritesHalfFloatPixels` in
  `Tests/Material/TestNoiseTextureFormatAndOctaves.cpp` reads the source mip and asserts every pixel
  is finite, in 0-1, grayscale and opaque, that the field varies, and that it matches the BGRA8 field
  from identical parameters within one quantisation step. Not fixed here, same shared helper:
  `create_gradient_texture` has the identical byte-fill-into-RGBA16F mismatch (its
  `CreateEmptyTexture(..., bHDR)` call is TextureHandler.cpp:586 post-fix), and `CreateEmptyTexture`
  sizes its hand-built platform mip at 16 bytes/pixel for
  `PF_FloatRGBA`, which is 8 -- harmless only because `UpdateResource()` rebuilds it from Source.
