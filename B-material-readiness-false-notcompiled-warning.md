---
id: B-material-readiness-false-notcompiled-warning
title: "materialReadiness reports notCompiled + fallbackPossible:true + a Default-Material warning on frames that demonstrably rendered the real materials — and contradicts subjectMaterialsRendered:true in the same block"
status: OPEN
severity: Low
category: bug
tags: [render, capture_asset_preview, generate_thumbnail, materialReadiness, notCompiled, fallbackPossible, false-positive, readiness, visual-verification, weapons]
encounters: 1
lastSeen: 2026-09-05T00:00:00Z
---

# The readiness block warns about a fallback that the same block says did not happen

`materialReadiness` exists to stop a caller trusting a capture of the engine Default Material. On
every capture taken this round it fired the warning over frames that rendered the real materials,
while carrying the fields that prove they did.

## What came back — measured

Fired on **all 19 asset-preview captures** of the round, including frames that visually show
triplanar mottle and four distinct material instances:

```jsonc
"materialReadiness": {
  "status": "notCompiled",
  "fallbackPossible": true,
  "warning": "... may have used the engine Default Material ...",
  "subjectMaterialsRendered": true,
  "fallbackOccurred": false
}
```

`subjectMaterialsRendered: true` and `fallbackOccurred: false` sit in the **same block** as the
warning. The block contradicts itself, and the half that is wrong is the loud half.

## Why the frames are known good

Not asserted from the block's own fields — read off the images. The captures show triplanar
mottling and **four visually distinct material instances** on the subject. A Default-Material
fallback renders a flat grey with a world-space checker grid; nothing in these frames resembles
that.

## Why this is not the silent-fallback ticket

`B-capture-verbs-silent-default-material-fallback` (IN-REVIEW, High) is the **opposite direction**:
a capture of a broken material returning `success: true` with nothing saying the material fell back.
Its fix added exactly this readiness probing, and its own description sets `fallbackPossible: true`
with `possibleReason: "shaderMapIncomplete"` whenever a used material is still `notCompiled`,
`outstanding` or `timedOut`. So this ticket is that fix's false-positive side: the fail-closed rule
is firing on a state that is not actually a fallback risk, on the normal path, every time.

Filed separately rather than appended because closing the silent-fallback ticket should not be
blocked on this, and because the ask here is a refinement of a shipped behaviour rather than a
failure of it. A tester verifying that ticket should know this ticket exists.

## What is asked for

1. **Let the outcome override the prediction.** When `subjectMaterialsRendered: true` and
   `fallbackOccurred: false` are both established for the frame, `fallbackPossible` and the warning
   text are stale predictions of a risk that has since resolved — suppress the warning, or restate it
   as `fallbackPossible: false, resolvedBy: "observedRender"`. A block must not carry a warning its
   own siblings disprove.
2. **Say which material is `notCompiled`.** `status: "notCompiled"` with no name gives a read-only
   caller nothing to act on. If it is a material that is on the asset but not on-screen (an unused
   slot, a LOD-only material), naming it converts an alarm into a note.
3. **Give a read-only caller a way to clear it.** Today there is none: the work was read-only, no
   material was authored, and no verb in the read path compiles a shader map — so the warning fires
   on every capture and can never be resolved by anything the caller is doing. Either the readiness
   probe should wait/compile on request, or the docs should say plainly that this warning is expected
   on assets whose shader maps are not warm and is not a defect in the capture.

## Severity

**Low**, and deliberately so despite the 19/19 reach. Nothing is lost or corrupted, the correct
answer is in the same block, and the frames are good. It is pure friction — but it is friction on a
reflex the plugin is trying to build (check readiness before trusting a capture), and a warning that
is always wrong is how that reflex gets unlearned. That is the argument for fixing it rather than
its impact class.

## Related

- `B-capture-verbs-silent-default-material-fallback` (IN-REVIEW, High) — **the ticket whose fix this
  is the false-positive side of.** Read together; the fail-closed behaviour it added is correct and
  should stay, this asks only that an observed-good render suppress the prediction.
- `E-material-verbs-have-no-shader-compile-signal` — the missing signal a read-only caller would need
  to satisfy ask (3).
- `B-compile-material-blocks-and-mislabels` — the other side, where compile state is reported wrongly
  on the authoring path.
- `E-capture-preview-pose-ignores-bounds-shape`, `B-preview-key-azimuth-moves-backdrop` — the other
  capture-verb tickets this review round touched.

## History
- `#1-filed` `OPEN` WEAPONS-critic — Measured during a WEAPONS critic review round 3. `materialReadiness` returned `status: "notCompiled"`, `fallbackPossible: true` and a warning that the frame "may have used the engine Default Material" on **all 19 asset-preview captures** taken this round — including frames that visibly show triplanar mottle and four distinct material instances, none of which resembles the flat grey + world-space checker grid a Default-Material fallback produces. The frames are called good from the **images**, not from the response. The same block simultaneously carries `subjectMaterialsRendered: true` and `fallbackOccurred: false`, so it contradicts itself and the wrong half is the loud one. This is the false-positive side of `B-capture-verbs-silent-default-material-fallback` (IN-REVIEW, High), whose fix added this probe and whose description sets `fallbackPossible:true` with `possibleReason:"shaderMapIncomplete"` for any used material still `notCompiled`/`outstanding`/`timedOut` — filed separately so closing that ticket is not blocked on this, and because the ask is a refinement of shipped behaviour rather than a failure of it; a tester verifying that ticket should know this exists. Ask: let an observed-good outcome override the prediction (suppress, or restate as `fallbackPossible:false, resolvedBy:"observedRender"`) — a block must not carry a warning its own siblings disprove; name which material is `notCompiled`, since an unnamed status gives a read-only caller nothing to act on and may well be an off-screen or LOD-only slot; and give read-only work a way to clear it, since no verb in the read path compiles a shader map, so today the warning fires on every capture and is unresolvable by anything the caller is doing. Severity **Low** despite 19/19 reach: nothing is lost, the correct answer is in the same block, the frames are good — pure friction. Recorded reason to fix it anyway: it lands on the reflex the plugin is trying to build (check readiness before trusting a capture), and a warning that is always wrong is how that reflex gets unlearned. No plugin source was opened for this ticket and no `file:line` is claimed.
