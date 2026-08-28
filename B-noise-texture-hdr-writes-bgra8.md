---
id: B-noise-texture-hdr-writes-bgra8
title: "texture.create_noise_texture hdr:true initialises a TSF_RGBA16F source but the pixel loop still writes BGRA8 bytes"
status: DONE
severity: High
category: bug
tags: [texture, create_noise_texture, hdr, TSF_RGBA16F, wrong-pixels, format-mismatch]
encounters: 1
lastSeen: 2026-08-28
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

- `#3-verified-behaviourally-on-the-built-binary` `DONE` verifier — 2026-08-28, against the
  `b79ba53e` build. Two 256x256 textures from identical parameters (`scale:5`, `octaves:3`,
  `seed:42`), one `hdr:true` and one `hdr:false`.

  Format is real, not echoed from the request: `texture.get_texture_info` on the HDR asset reports
  `format: "FloatRGBA"`, `sRGB: false`, `compression: "TC_HDR"`, 9 mips, and the create response
  echoes `hdr: true`. The LDR control reports `sourceFormat: "TSF_BGRA8"`.

  **The pixels are correct, which is the half a format check cannot answer.** The predicted pre-fix
  symptom was garbage colour over a half-unwritten buffer with alpha built from a neighbouring
  pixel's bytes — a 4-byte fill into an 8-byte/pixel surface. Both textures were rendered to PNG and
  read as images: the HDR frame is a complete, smooth, fully-covered greyscale cloud field with no
  black region, no colour fringing and no alpha artefact, and it is visibly the SAME field as the
  BGRA8 control — same cloud shapes in the same places, differing only in overall brightness and
  contrast, which is what `SRGB` being off on the linear path should do. Measured on the two decoded
  frames: **Pearson r = 0.99251, Spearman rho = 0.99785** over all 65,536 pixels, with means 187.89
  (HDR) against 130.97 (BGRA8) — one field under two transfer curves. `imageStats` on the HDR frame:
  `litPixelFraction 1`, min 0.447, max 0.933, 115 tone levels, `blank: false`.

  **A gap found while verifying, not a defect in this fix:** `texture.get_pixel_stats` refuses the
  format it now has to read — `[PIXEL_STATS_UNAVAILABLE] Unsupported source format TSF_RGBA16F for
  pixel stats (only TSF_BGRA8 and TSF_G8 are read)`. So the shipped test reads the source mip from
  C++, and a **caller** has no way to read HDR pixels back at all; the only route left is rendering a
  thumbnail, which is what this verification had to do. That is worth its own ticket and is not a
  reason to hold this one open. `create_gradient_texture`'s identical byte-fill-into-RGBA16F
  mismatch, which `#2` explicitly left unfixed, was not exercised here. Closing.
