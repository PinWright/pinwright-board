---
id: F-capture-asset-preview-no-material-subject-kind
title: "render.capture_asset_preview serves no material subject kind, so the one verb that reviews an asset in a preview viewport cannot review the asset type most often reviewed that way"
status: OPEN
severity: Low
category: feature
tags: [render, capture-asset-preview, generate-thumbnail, material, material-instance, subject-kind, preview]
encounters: 1
lastSeen: 2026-09-05T19:50:00Z
---

# No material subject kind on the preview-capture verb

## What happened

```
render.capture_asset_preview {assetPath: "/Game/FPS/Player/MI_FPSArms", primitive intent: sphere}
-> [UNSUPPORTED_ASSET_EDITOR] render.capture_asset_preview captures the asset editor preview of a
   Static Mesh, Skeletal Mesh, animation asset or Niagara system ('/Game/FPS/Player/MI_FPSArms' is a
   MaterialInstanceConstant, which no capture subject kind serves). For a material or a texture,
   asset.generate_thumbnail renders one offscreen to a file and opens no asset editor.
```

**The refusal is good behaviour and the error message is excellent** — it names the asset's class,
says plainly that no subject kind serves it, and points at the verb that does. Nothing here is a
silent failure. This is a capability request, not a defect report.

## Why it is worth asking for

`subject.kind` covers `staticMesh | skeletalMesh | animation | niagara`. A material is the asset
type people *most* often review in a preview viewport, and the verb's whole value proposition — the
controllable rig — is exactly what a material review needs: `viewMode`, `previewScene`
(key azimuth/elevation/intensity, sky, floor), `count` / `views: 'sides'` orbits, `exposure`
pinning, and `measureCoverage`. `asset.generate_thumbnail` is the right answer today and it is a
good one, but it offers `primitive`, `azimuth`, `elevation` and `zoom` only: no view modes, no
explicit light rig, no orbit set, no coverage differential.

Concretely, the thing I could not do: compare a material across two lighting rigs at a pinned
exposure to decide whether a surface reads as fabric or as plastic. `generate_thumbnail` renders
under whatever rig the thumbnail system uses, and `previewScene` — the field that exists precisely
because "a preview scene belongs to the editor showing it, so the same call on two machines returns
different pixels" — is on the other verb.

## Asked for

A `material` subject kind on `render.capture_asset_preview`, taking the same `primitive` /
`primitiveMesh` shape selector `generate_thumbnail` already implements, so the shared rig controls
apply to materials too. Failing that, lift `viewMode` and `previewScene` onto
`asset.generate_thumbnail`.

## Note for the project side, not the plugin

`Docs/fps/PLAN.md` rule 10's capture exception is written as "allowed for **static meshes and
materials only**". Materials are the half that verb does not serve, so anyone following that rule
literally hits this refusal. The project rule wants rewording toward `asset.generate_thumbnail`,
which is strictly safer anyway: it opens no asset editor at all, so rule 10's whole
close-the-window hazard does not apply to it.

severity rationale: impact=an alternative verb exists and is well signposted, so nothing is blocked
x reach=any caller reviewing material appearance with a controlled rig -> Low

## History
- `#1-filed` `OPEN` reporter — Hit on the FPS PLAYER stream while checking whether a viewmodel arms material renders dark or pale independently of the mesh carrying it. `asset.generate_thumbnail {primitive: "sphere"}` answered the question outright — its `materialReadiness` block reported `usingDefaultMaterial: false, fallbackOccurred: false, compiled: true, errorCount: 0` alongside `meanLuminance 0.131`, which settled the question in one call. Recording the gap rather than a complaint: the fallback path worked, and the `allowFallback` default of false is exactly the right gate for this kind of check.
