---
id: B-create-empty-texture-mip-size-wrong-for-float-rgba
title: "CreateEmptyTexture sizes its hand-built platform mip at 16 bytes/pixel for PF_FloatRGBA, which is 8"
status: OPEN
severity: Medium
category: bug
tags: [texture, CreateEmptyTexture, PF_FloatRGBA, platform-data, latent, shared-helper]
encounters: 1
lastSeen: 2026-08-28
---

# A wrong size that is currently harmless for one reason only

`CreateEmptyTexture` hand-builds a platform mip and sizes it at 16 bytes per pixel for `PF_FloatRGBA`.
The format is 8.

It does not currently break anything, because `UpdateResource()` rebuilds platform data from `Source`
in the editor, so the hand-built mip is discarded before anything reads it. That is the only thing
standing between this and a real defect — and it is a property of the editor path, not of the helper.

Shared helper: five verbs call it.

**Fix:** size it correctly, or stop hand-building the mip at all if `UpdateResource()` always
supersedes it — the second is likely the better change, since a buffer nothing reads is a trap for the
next reader.

## History
- `#1-found-fixing-the-hdr-mismatch` `OPEN` reporter — Found by the agent fixing
  `B-noise-texture-hdr-writes-bgra8` while establishing the correct bytes-per-pixel for
  `TSF_RGBA16F`. Deliberately left alone: shared helper, five callers, and no live symptom.
