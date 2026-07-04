---
id: F-animation-set-curve-key-no-batch
title: "animation.authoring.set_curve_key is one-key-per-RPC — authoring an N-key driver curve costs N round-trips while the sibling niagara.set_curve_keys already batches"
status: OPEN
severity: Low
category: feature
tags: [no-batch-authoring, animation, curve, set_curve_key, batch, ergonomic]
encounters: 1
lastSeen: 2026-07-04T15:25:37.0000000Z
---

# `animation.authoring.set_curve_key` has no batch form — one RPC per key

`animation.authoring.set_curve_key` writes exactly **one** `{frame, value}` onto
one Float curve per call (registered params: `assetPath`, `curveName`, `frame`,
`value`, `createIfMissing`, `save` — see
`Source/PinWright/Private/Handlers/Animation/AnimationAuthoringHandler_Sequence.cpp:487`,
which does a single `Controller.SetCurveKey(CurveId, FRichCurveKey(...))` per
invocation). Authoring an N-key float driver curve therefore costs N separate
round-trips that differ only in `frame`/`value`, repeating `assetPath` +
`curveName` verbatim each time. A longer eased curve scales linearly.

## What it should do

Add a batch curve-key setter for `animation.authoring`, e.g.
`animation.authoring.set_curve_keys {assetPath, curveName, keys:[{frame, value, interp?}...], createIfMissing?, save?}`
(or accept an optional `keys` array on the existing `set_curve_key`), applying a
whole curve's keys in ONE call. Keep the existing single-`frame`/`value` form
working for the trivial case.

## Evidence of precedent

The plugin already exposes exactly this shape one namespace over:
`niagara.set_curve_keys {assetPath, scope, parameterName, keys:[{time, value, interp?, arriveTangent?, leaveTangent?, ...}]}`
replaces a whole `FRichCurve`'s keys in a single RPC
(`Source/PinWright/Private/Handlers/Niagara/NiagaraCurveHandler.cpp:228`, shipped
via `F-niagara-curve-authoring`). The animation-curve path having only the
singular write form — even though `animation.authoring.list_curves` already reads
every curve's keys back at once — is an ergonomic asymmetry, not a functional bug.

## Why it's process friction (clean per-call outcome)

Every call in the evidence task succeeded first-try; this is a pure round-trip /
batch-convenience gap, not a failure. The "workaround" is simply to issue the N
calls (they all work), so no source-dive or error recovery is needed — hence Low.

**Evidence (this audit):** clean-outcome process audit of a curve-authoring task
(focus `animation.authoring.set_curve_key`, namespace `animation.authoring`;
story: create a 60f@30fps `AS_DinoDragon_Glow` sequence bound to the DinoDragon
skeleton and author a `GlowIntensity` curve of 4 keys + a `GlowSpeed` curve of 3
keys). The Attempt spent **7 consecutive `set_curve_key` RPCs** — 4 (GlowIntensity
f0=0, f18=0.6, f30=1.0, f60=0) + 3 (GlowSpeed f0=0, f30=0.5, f60=1.0) — all
`ok:true`, differing only in frame/value, to lay down two trivial driver curves.
A batch form collapses this to 2 calls. Outcome `clean`, friction note "none"
(the agent even reported "single-call helpers existed for each step"); zero
`is_error` in the 11-call log. The `create_animation_sequence` wiki page also
points post-create authors only at `add_bone_track`/`set_bone_key` and never
mentions curve authoring, so a batch curve API would ideally be cross-linked there.

severity rationale: impact=pure-friction (N extra round-trips, no failure and no
source-dive workaround needed — the N calls just work) × reach=rare (animation
float-curve authoring, not an every-session path) -> Low

## Cross-ref

- `niagara.set_curve_keys` (`F-niagara-curve-authoring`, DONE) — the batch
  curve-key precedent this proposal mirrors.
- `animation.authoring.list_curves` (`F-rpc-animation-list-curves`, DONE) — the
  read side already returns all keys at once; only the write side is per-key.
- `F-add-mapping-batch-keys` (OPEN) — same `no-batch-authoring` family (one-item-
  per-call authoring verb, no batch form) but a DISTINCT method/subsystem
  (`input.add_mapping` / IMC key binding); filed separately, not merged.

## History
- `#1-initial-audit` `OPEN` reporter — Clean-outcome process audit of an animation curve-authoring task (focus `animation.authoring.set_curve_key`). The Attempt issued 7 consecutive single-key `set_curve_key` RPCs (4 GlowIntensity + 3 GlowSpeed), all success, differing only in frame/value, to author two float driver curves on `AS_DinoDragon_Glow` — a batch form would make that 2 calls. Confirmed on the on-disk plugin source that `niagara.set_curve_keys` (batch) exists but `animation.authoring` registers only the singular `set_curve_key` (one `Controller.SetCurveKey` per call) with no plural. Pure round-trip/ergonomic gap: every call landed first-try, zero is_error, no workaround needed. Proposes `animation.authoring.set_curve_keys {assetPath, curveName, keys:[...]}` (or a `keys` array on `set_curve_key`) mirroring the niagara precedent. No prior animation curve-key batch ticket on the board (ripgrep clean for `set_curve_key`/batch under OPEN+closed); the nearest sibling `F-add-mapping-batch-keys` is a distinct method/subsystem, kept separate.
