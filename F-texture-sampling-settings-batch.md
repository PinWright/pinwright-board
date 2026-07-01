---
id: F-texture-sampling-settings-batch
title: "No batch setter for a texture's sampling settings — configuring wrap + filter + group + LOD bias on one texture costs 4 separate set_* RPCs (each its own write+save round trip)"
status: OPEN
severity: Low
category: feature
tags: [texture, batch, set_texture_wrap, set_texture_filter, set_texture_group, set_lod_bias, sampling, save, ergonomic]
encounters: 1
lastSeen: 2026-06-24T09:27:13Z
---

# Configuring a texture's sampling has no batch form; the canonical "tune this texture for tiling" intent costs 4 single-field setters

Setting up one texture's sampling behaviour is a single conceptual authoring
step — "make this floor texture tile cleanly and stay crisp" — but it decomposes
into **four separate `texture.*` setter RPCs**, each of which loads the asset,
writes one reflection field, and (with the default `save:true`) re-saves the
package:

- `texture.set_texture_wrap {wrapMode}`
- `texture.set_texture_filter {filter}`
- `texture.set_texture_group {textureGroup}`
- `texture.set_lod_bias {lodBias}`

There is no `texture.set_sampling_settings` (or equivalent) that accepts the
whole bag in one call with a single save. Each of the four is its own
write+package-save round trip on the same asset, so the canonical "tune a
texture for use in a material" motion pays 4× the package-save cost where one
would do.

## Why this is friction (not just normal granularity)

"Find a Texture2D and configure it for tiling — Wrap addressing, Trilinear
filtering, World group, a touch of negative LOD bias" is a recurring, canonical
authoring intent (it directly precedes wiring the texture into a material). The
user expressed it as one step ("set its wrap mode to Wrap, set its filter to
Trilinear, put it in the World texture group, and nudge its LOD bias to -1"),
but it became four RPCs that differ only in which field they touch — the same
shape the board has already accepted as batch-worthy for
`F-geometry-lod-settings-batch` (per-LOD reduction → `lods:[{...}]`) and
`F-vehicle-wheel-asset-batch-properties` (suspension fields →
`set_wheel_asset_properties`). Both of those cite the established
`F-batch-pin-defaults` (DONE) plural-handler precedent. Texture sampling is the
same case: N single-field setters on one asset, where the user's intent was one.

This is a clean-outcome PROCESS finding — every call in the task succeeded first
try with no retries, workarounds, or `python.execute` fallback (see evidence).
The friction is the call *count* and the repeated write+save, not any failure.
It is distinct from the judge's `E-texture-describe-omits-lodbias-wrap`, which
covers the missing *read-back* of these fields (`texture.describe` omitting
`lodBias`/wrap/`filter`); this ticket is about the *write* surface having no
batch form. The two compound: lacking a batch setter, the agent also interleaved
a `describe` readback after every setter, so the single intent cost 4 setters +
several readbacks.

## What it should do

Add a batched setter so the whole sampling bag is one call + one save, e.g.:

```
texture.set_sampling_settings(assetPath,
    wrapMode? /* or addressX?/addressY? */,
    filter?,
    textureGroup?,
    lodBias?,
    save? /* one save after the whole batch */)
```

applying only the provided fields under one load and a single package save.
Keep the four singular setters as documented single-field fast paths (mirrors
how `F-batch-pin-defaults` added the plural form next to the singular). This
also aligns the texture surface with `texture.create_pattern_texture` /
`texture.create_*`, which already take multiple creation params in one call —
the asymmetry is that creation is one-shot but post-create sampling tuning is
one-call-per-field.

**Workaround:** Issue the four `texture.set_*` setters in sequence (the agent
did exactly this: `set_texture_wrap` Wrap, `set_texture_filter` Trilinear,
`set_texture_group` TEXTUREGROUP_World, `set_lod_bias` -1), each `save:true`.

## Friction evidence (this task — texture.set_texture_wrap "tiling stone floor" setup, outcome ergo/clean)

Story: configure `/Game/Global/Textures/T_FloorMarble_D` for a tiling
stone-floor material — Wrap addressing, Trilinear filter, World texture group,
LOD bias -1, persisted, with a readback after each edit. Call log shows the four
sampling edits split across four consecutive single-field setters, each
`save=true`:

- `texture.set_texture_wrap {T_FloorMarble_D wrapMode=Wrap save=true}` → ok
- `texture.set_texture_filter {T_FloorMarble_D filter=Trilinear save=true}` → ok
- `texture.set_texture_group {T_FloorMarble_D textureGroup=TEXTUREGROUP_World save=true}` → ok
- `texture.set_lod_bias {T_FloorMarble_D lodBias=-1 save=true}` → ok

All four succeeded first try (`ok:true`, no `is_error`, no retries). A single
`texture.set_sampling_settings(assetPath, {wrapMode, filter, textureGroup,
lodBias}, save)` would have collapsed the four write+save round trips into one.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the
  `texture.set_texture_wrap` "tiling stone floor" task (namespace `texture`,
  outcome `ergo`, judge filed the read-back gap `E-texture-describe-omits-lodbias-wrap`).
  Distinct PROCESS angle: the *write* surface has no batch form, so the single
  authoring intent "configure this texture's sampling" cost **4 single-field
  setters** — `set_texture_wrap` (Wrap), `set_texture_filter` (Trilinear),
  `set_texture_group` (TEXTUREGROUP_World), `set_lod_bias` (-1) — each its own
  load + reflection write + package save (`save=true`). No retries/errors/
  fallbacks; pure call-count + repeated-save overhead. Proposed: add a batched
  `texture.set_sampling_settings(assetPath, {wrapMode/filter/textureGroup/lodBias},
  save)` (apply provided fields, single save), keeping the singular setters as
  fast paths — mirrors implemented `F-batch-pin-defaults` and the sibling
  audits `F-geometry-lod-settings-batch` / `F-vehicle-wheel-asset-batch-properties`.
  Dedup: ripgrep across OPEN+closed — no `F-texture*` ticket exists and no
  texture batch/sampling-settings ticket; `E-texture-describe-omits-lodbias-wrap`
  is the read-back layer (distinct write-vs-read facet);
  `E-texture-action-handler-param-docs` is the param-docs layer (distinct).
