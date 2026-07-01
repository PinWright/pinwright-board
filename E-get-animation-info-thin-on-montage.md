---
id: E-get-animation-info-thin-on-montage
title: "`animation.authoring.get_animation_info` on an AnimMontage emits only {numSections, numSlots, duration} — no section names / startTimes / nextSectionName links, so the link_sections round-trip is unverifiable without asset.dump"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [animation, anim-montage, asset-dump, parity, get-animation-info, link-sections, docs]
---

# `get_animation_info` on a montage is count-only — section names / links live only in the asset.dump sidecar

`get_animation_info` is documented as the introspection dual used to confirm an
asset's shape after authoring (the *Inspect-after-mutate* loop on the
`animation.authoring` wiki overlay). For a `UAnimSequence` it returns a rich
shape (`duration`, `numFrames`, `frameRate`, `frameRateRational`, `additiveType`,
`rawTrackCount`, `skeletonAssetPath`, …). But the `UAnimMontage` branch
(`AnimationAuthoringHandler_Sequence.cpp:1817-1826`) emits only **counts**:

```
{"animationInfo":{"assetType":"AnimMontage","duration":0,"numSections":3,"numSlots":2,"numNotifies":0},"success":true,"message":"Animation info retrieved"}
```

`numSections`/`numSlots` confirm *how many* sections and slots exist but never
*which* — no per-section `sectionName`, no `startTime`, and critically no
`nextSectionName`. So after the canonical montage build-out
(`add_montage_section` ×N → `set_section_timing` → `link_sections`) there is no
way to confirm the **section names, their start times, or the section→section
links** from `get_animation_info` — the very chaining the `link_sections` verb
exists to author. The only way to verify *which* section links to *which* is to
fall back to the read-only `asset.dump` sidecar `anim_montage.json`, which
`AnimMontageDumpBuilder::…` (`Private/Handlers/Asset/AnimMontageDumpBuilder.cpp`)
emits with full fidelity:

```
"sections":[
  {"sectionName":"Idle","nextSectionName":"Walk","startTime":0},
  {"sectionName":"Walk","nextSectionName":"","startTime":0.96666663885116577}, …
]
```

So the section→section link map (`nextSectionName`), the section names, and the
per-section start times are **exclusive to the dump** — unreachable through the
live introspection RPC. This is the same no-exclusive-dump-fields parity class
that `E-rpc-animation-extend-get-animation-info` (DONE) fixed for the
`UAnimSequence` branch and `E-get-animation-info-thin-on-blend-space` (OPEN)
files for the `UBlendSpace` branch — here repeated for the `UAnimMontage` branch.
The call is not wrong (the counts are accurate, no crash, valid JSON) — it is too
thin to verify the very structure `link_sections`/`set_section_timing`/
`add_montage_section` author.

Two compounding discoverability traps observed in the same task:
- The fallback sidecar `anim_montage.json` is itself **undocumented** — a grep
  over the whole `docs/wiki-src/` tree finds zero mentions of `anim_montage`
  (the `asset.dump-sidecars.md` page references `UAnimMontage` only in a
  cast-order gotcha, not as a documented sidecar with its field shape). So the
  agent had to discover the sidecar's existence and section schema empirically.
- The task's prescribed SUCCESS CHECK was `get_animation_info` itself, which
  *structurally cannot* confirm the `link_sections` round-trip — the success
  criterion points at a method that lacks the field it needs to check.

This is distinct from `B-montage-slot-no-sequence-length-recalc` (the `duration:0`
length **correctness** bug — a different field, a write-side defect): even once
length is fixed, `get_animation_info` would still report only counts for sections,
so the section/link **readback parity** gap is independent and would remain.

**Workaround:** read the `asset.dump` `anim_montage.json` sidecar (`sections[]`
with `sectionName`/`nextSectionName`/`startTime`, and per-slot `segments[].startPos`)
for the section names, start times, and link map.

**Fix:** Additively extend the `UAnimMontage` branch of `get_animation_info` to
emit a `sections[]` array (`sectionName`, `nextSectionName`, `startTime`) and a
`slots[]` summary (slot name + segment animation paths + `startPos`), reusing the
same public accessors `AnimMontageDumpBuilder` already uses
(`Montage->CompositeSections[i].SectionName/NextSectionName/GetTime()`,
`Montage->SlotAnimTracks[i]`) — ideally by delegating to the dump builder so the
live shape matches the sidecar. Keep `numSections`/`numSlots`/`duration` for
backward compatibility. Separately (downstream wiki process, not this audit's
job), document the `anim_montage.json` sidecar and its `sections[]` field shape
on `docs/wiki-src/asset.dump-sidecars.md`.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `AM_DinoDragon_IdleToWalk` montage task (namespace animation, focus `animation.authoring.link_sections`, outcome tool_bug; the length half is the judge-filed `B-montage-slot-no-sequence-length-recalc`). Friction note (verbatim): *"the task's prescribed SUCCESS CHECK method (animation.authoring.get_animation_info) is too shallow for a montage — it returns only numSections/numSlots and duration:0, never section names, start times, or next-section links, so it structurally cannot confirm the link_sections round-trip; I had to fall back to asset.dump's anim_montage.json sidecar (which itself is undocumented on the asset.dump-sidecars wiki page) to verify Idle->Walk."* Source-confirmed: `AnimationAuthoringHandler_Sequence.cpp:1817-1826` UAnimMontage branch sets only `assetType`/`duration`/`numSections`(`CompositeSections.Num()`)/`numSlots`(`SlotAnimTracks.Num()`)/`numNotifies` — no section names/links; `AnimMontageDumpBuilder.cpp:30-32` emits per-section `sectionName`/`nextSectionName`/`startTime`. Grep over `docs/wiki-src/` finds zero `anim_montage` mentions (sidecar undocumented). Same no-exclusive-dump-fields parity class as DONE `E-rpc-animation-extend-get-animation-info` (AnimSequence) and OPEN `E-get-animation-info-thin-on-blend-space` (BlendSpace); here for the AnimMontage branch. The seed `link_sections` round-trips correctly (dump shows Idle.nextSectionName="Walk"); culprit is `get_animation_info` thinness, not the seed. Distinct from `B-montage-slot-no-sequence-length-recalc` (that is the duration=0 write-side length bug; this is the section/link readback parity gap). Tagged docs for the downstream `asset.dump-sidecars.md` sidecar-documentation edit. Call-log: 15 calls, `get_animation_info` ×3 (two source-clip inspects + one montage read-back that returned counts-only), forcing an `asset.dump` to obtain the section/link verification.
- `#2-fix` `IN-REVIEW` developer — Extended the `UAnimMontage` branch of `animation.authoring.get_animation_info` to additively merge the asset.dump anim_montage.json sidecar shape, mirroring the already-shipped `UBlendSpace` delegation pattern in the same handler. The branch now delegates to `AnimMontageDumpBuilder::BuildAnimMontageJson(Montage)` and copies every builder field the handler hasn't already set (`sections[]` with `sectionName`/`nextSectionName`/`startTime`/`linkValue`, `slots[]` with per-segment `startPos`, `notifies[]`, `blendIn`/`blendOut`/`bEnableAutoBlendOut`), so the section→section link map that `link_sections`/`set_section_timing`/`add_montage_section` author is now verifiable through the live RPC without falling back to `asset.dump`. Back-compat `assetType`/`skeletonPath`/`duration`/`numSections`/`numSlots`/`numNotifies` are set first and take precedence. Files: `Source/PinWright/Private/Handlers/Animation/AnimationAuthoringHandler_Sequence.cpp` (added `#include "Handlers/Asset/AnimMontageDumpBuilder.h"` + the delegation merge in the montage branch). Test: `FAuthoringGetAnimationInfoMontageParityFieldsTest` (`PinWright.animation.authoring.get_animation_info.MontageParityFields`) in `Source/PinWright/Private/Tests/Gameplay/TestAnimationHandlers.cpp` — builds a transient UAnimMontage with two linked composite sections (Idle→Walk), invokes the real handler, and asserts `sections[0].sectionName=="Idle"`, `sections[0].nextSectionName=="Walk"`, `startTime` present, `slots[]` present, and back-compat `assetType`/`numSections` preserved; it fails if the delegation is reverted to the counts-only shape. The wiki-doc half (documenting the `anim_montage.json` sidecar shape on `asset.dump-sidecars.md`) remains the downstream process noted in **Fix:**, out of scope for this readback-parity change.
