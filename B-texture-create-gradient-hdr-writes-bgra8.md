---
id: B-texture-create-gradient-hdr-writes-bgra8
title: "texture.create_gradient_texture has the identical HDR format mismatch just fixed on create_noise_texture"
status: OPEN
severity: High
category: bug
tags: [texture, create_gradient_texture, hdr, TSF_RGBA16F, wrong-pixels, format-mismatch]
encounters: 1
lastSeen: 2026-08-28
---

# Same `CreateEmptyTexture(..., bHDR)` call, same 4-byte fill loop

`B-noise-texture-hdr-writes-bgra8` is fixed: `TSF_RGBA16F` is 8 bytes per pixel, the fill loop indexed
at `*4` and wrote 4 bytes, so `hdr: true` filled only the first half of the source buffer and every
written byte pair was reinterpreted as one half-float.

`texture.create_gradient_texture` makes the same `CreateEmptyTexture(..., bHDR)` call
(`TextureHandler.cpp:586` after that fix) and feeds it the same 4-byte BGRA fill loop. Same defect,
different verb.

**Fix:** the same shape — branch the fill on `Source.GetFormat() == TSF_RGBA16F` (the allocation, not
the request flag, so fill and allocation cannot disagree) and write four `FFloat16` channels.

## History
- `#1-found-fixing-the-sibling` `OPEN` reporter — Found by the agent fixing the `create_noise_texture`
  HDR defect, which swept the file's other `CreateEmptyTexture` callers. Source reading, not reproduced.
