---
id: B-nir-renderer-omits-default-valued-properties
title: "nir.txt renderer blocks omit properties sitting at the class default, so an absent bool reads as false when it is actually true — bSubImageBlend defaults to true and inverts"
status: OPEN
severity: Medium
category: bug
tags: [nir, asset-dump, niagara, renderer, defaults, silent-omission, boolean-inversion, sprite-renderer, subuv]
encounters: 1
lastSeen: 2026-09-03T00:45:00+05:00
---

# `nir.txt` omits default-valued renderer properties, and the reader cannot tell "absent" from "false"

## Symptom

A `renderer NiagaraSpriteRendererProperties @0 enabled { ... }` block in `nir.txt` lists
only the properties whose value **differs from the class default**. Properties at their
default are omitted entirely, with no marker saying so.

That is fine for a human skimming, and wrong for anything that parses it, because the
omission is indistinguishable from "false" for a bool. It bites hardest on
`UNiagaraSpriteRendererProperties::bSubImageBlend`, whose class default in UE 5.8 is
**`true`**:

| what `nir.txt` shows | actual value |
|---|---|
| `bSubImageBlend: false` present | `false` (explicitly overridden) |
| line absent entirely | **`true`** (class default) |

So the naive reading — "the line isn't there, so the flag isn't set" — returns the exact
opposite of the truth on the one renderer flag most likely to be audited across a package.

## Measurement

Two controls on emitters neither of which had this property written by the caller, both
read back with `property.get` on the renderer subobject:

```
E_FPS_MuzzleAR_Petals   nir: no bSubImageBlend line
  property.get bSubImageBlend -> true      # absent == true

E_ImpMetal_Sparks       nir: "bSubImageBlend: false"
  property.get bSubImageBlend -> false     # printed == explicitly false
```

`SubImageSize` behaves the same way: absent means the default `(1,1)`, and only a
non-default size is printed. That case happens to be harmless because the default is the
"unset" value a reader would guess, which is exactly why the `bSubImageBlend` inversion
goes unnoticed — one property in the same block confirms the reader's assumption while the
other silently contradicts it.

## Why it matters

This is a read-surface defect on the sidecar the docs steer bulk auditing toward.
`asset.dump_folder` + parse `nir.txt` is the recommended way to answer "which assets in
this subtree have property X", precisely because it avoids an RPC per asset. A boolean that
inverts on omission makes that answer wrong in the confident direction: the audit reports a
clean list of assets needing a fix, and the assets are already correct.

Concretely, in this session: an audit of 57 emitters under `/Game/FPS/VFX/Emitters`
reported 11 sprite renderers as `bSubImageBlend = false` and needing repair. All 11 were
already `true`. The wrong list was reported to the requesting agent as a work estimate
before `property.get` on two controls showed the parse was inverted. The cost was bounded
here only because the number looked implausible and got checked; a smaller discrepancy
would have produced eleven pointless remove/add/compile/save cycles on a shared editor, or
a false "already correct" verdict in the other direction.

## Expected

Any of these closes it; the first is the smallest:

- Print every reflected property in the renderer block, defaults included. The block is
  already ~10 lines; completeness costs little and removes the ambiguity entirely.
- Or keep the omission and emit a marker the parser can key on — a `defaults omitted:` list
  of names, or a `@default` suffix on printed values mirroring the `static ... @source
  override @default N` convention the **module** blocks already use. Module static-switch
  lines get `@source override @default 0.0`; renderer property lines get nothing, so the two
  halves of the same file disagree about how much a reader is told.
- Or document the rule in [`niagara.dump-files`](niagara.dump-files.md), which currently
  describes `nir.txt` as carrying "renderers" without saying the listing is filtered.

The third alone is weak: it leaves every existing parser wrong until its author re-reads the
page.

## Workaround

Do not infer a renderer property's value from its absence in `nir.txt`. For a bool, treat
absence as "unknown" and confirm with
`property.get {objectPath: "<emitter>.<emitter>:NiagaraSpriteRendererProperties_0",
propertyName: "..."}`, which returns the resolved value. When auditing in bulk, establish
the polarity first with a known-good and a known-bad control asset — one where the line is
printed and one where it is not — rather than assuming which way round it goes.

severity rationale: impact=silently inverted boolean on the recommended bulk-read path,
producing confidently wrong audit results in either direction x reach=every `nir.txt`
renderer block, i.e. every Niagara emitter in any dumped subtree, but only for properties
whose class default is not the "unset-looking" value -> Medium.

## History
- `#1-filed` `OPEN` reporter — Found auditing `SubImageSize` / `bSubImageBlend` across 57 emitters under `/Game/FPS/VFX/Emitters` on EAContentExamples58 (UE 5.8, shared editor, port 27145) while repairing flipbook renderers left at 1x1. The parse was a plain `'bSubImageBlend: true' in body` test over the `renderer ... { }` block, which never matched because the string never appears — the true case is expressed by absence. Corrected polarity is measured, not inferred: the two `property.get` controls quoted above are on emitters whose renderer I never wrote to, one printing the line and one omitting it, returning `false` and `true` respectively. The class-default reading (`bSubImageBlend` defaults to `true`) follows from that pair plus the omission rule; I did **not** read `NiagaraSpriteRendererProperties.h` to confirm the UPROPERTY initialiser, so a verifier should, and should also check whether other renderer bools (`bEnableCameraDistanceCulling`, `bGpuLowLatencyTranslucency`, sort flags) have non-false defaults and therefore invert the same way. `SubImageSize` was verified to follow the identical omit-at-default rule, defaulting to `(1,1)`.
