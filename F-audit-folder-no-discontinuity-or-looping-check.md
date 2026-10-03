---
id: F-audit-folder-no-discontinuity-or-looping-check
title: "audio.analysis.audit_folder has no edge-discontinuity, loop-seam or bLooping check, so a folder with per-shot clicks and non-looping beds sweeps clean"
status: OPEN
severity: Medium
category: feature
tags: [audio, analysis, audit_folder, discontinuity, looping, bLooping, hygiene-gate]
encounters: 2
costly: 2
lastSeen: 2026-10-02T00:00:00Z
rice: [2, 2, 1, 2]
priority: 17
---

# audit_folder cannot see the click defects the per-asset analyzer measures

Split out of `E-synth-cookbook-no-ambience-loops` (its History `#2` and `#3`), which was closed out
on the cookbook / loop-render side. The remaining ask is on a different verb, and it is not covered
by that change.

`audio.analysis.audit_folder` has seven finding types: non-finite, digital silence, clipping, DC
offset, loudness outlier, sample-rate outlier and channel-count outlier. None of them looks at a
wave's edges or its wrap:

1. **Edge discontinuity (one-shots).** A reporter's sweep over 67 waves returned `0 flagged` while
   two weapon mechanical layers carried `startDiscontinuity` `0.0035` and `0.0013`, i.e. a click on
   every shot. Every other wave in the folder was at exactly `0`.
2. **Loop seam (looping waves).** `analysis.technical.loopSeamRatio` now exists (wrap step divided
   by the RMS adjacent-sample step, below about 3 when seamless). A `bLooping: true` wave whose
   ratio is far above that clicks once per loop, and the audit does not report it.
3. **`bLooping` itself.** Three seamless beds named `*_Loop` were exported with `bLooping: false`
   (the `audio.synth.export` default). They play once and stop, and the sweep is clean.

Note: a correctly built seamless loop has **nonzero** start/end discontinuities by construction
(`master.loopCrossfadeMs` loops are not faded). So an edge check must not run unconditionally on a
looping wave. One shape that fits: on a `bLooping` wave check `loopSeamRatio`, on a non-looping
wave check `max(startDiscontinuity, endDiscontinuity)`. A non-looping wave with large nonzero
edges is then exactly the "seamless bed shipped as a one-shot" case from `#2`. Publish both
thresholds under `thresholds`, beside `dcOffsetLinear`. Mind the existing
`PinWright.audio.analysis.audit_folder.FlagsDefectsAndFitsCeiling` fixtures: its `SW_Clean` must
still sweep clean.

## History
- `#1-split-from-ambience-ticket` `OPEN` developer — Filed while implementing `E-synth-cookbook-no-ambience-loops`. That change adds `master.loopCrossfadeMs`, `analysis.technical.loopSeamRatio` and the cookbook's ambience family, and documents start/end discontinuity. It does not touch `audit_folder` (`Source/PinWright/Private/Handlers/Audio/AudioAnalysisHandler.cpp`, `AuditOneAsset` / `AuditFindingVocabulary`). The finding work lands on a separate verb with its own fixtures, so it is tracked here instead of widening that ticket.
