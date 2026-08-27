---
id: B-noise-texture-hdr-writes-bgra8
title: "texture.create_noise_texture hdr:true initialises a TSF_RGBA16F source but the pixel loop still writes BGRA8 bytes"
status: OPEN
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
