---
id: E-niagara-inspect-stack-no-module-enabled-flag
title: "niagara.inspect's stack aspect omits the module enabled flag that niagara.set_stack_enabled writes, so the only read path for it is a whole-system NIR decompile"
status: OPEN
severity: Low
category: ergonomic
tags: [niagara, niagara-inspect, set-stack-enabled, stack, readback, response-size, decompile-nir]
encounters: 1
lastSeen: 2026-09-05T22:30:00+03:00
---

# There is no cheap read for "is this stack module enabled?"

`niagara.set_stack_enabled {assetPath, entryId, enabled}` writes a per-module
enabled flag, but no read verb in the `niagara` namespace publishes it cheaply.

`niagara.inspect {includeStack:true}` emits each module as

```
ownerKind, ownerName, scriptUsage, index, present, entryId, entryKey,
name, nodeName, title, functionScript, selectedScriptVersion, posY,
staticSwitchInputs, moduleInputs
```

— fifteen fields, none of which is the enabled flag. `present` is not it: a
disabled module is still `present: true`. So a caller that disabled a module in
an earlier session, or that inherits an asset another agent authored, cannot
tell from the verb the wiki points at for stack work whether the module runs.

## The state IS readable, but only from the expensive surface

`niagara.decompile_nir` publishes it, per module, in the stack block:

```
module ParticleState@v1.2 @5 enabled
module ScaleColor@v1.0   @6 disabled
module FloatFromCurve001 @7 disabled
```

That is the whole workaround, and it costs a full NIR decompile of the asset.
Measured on `/Game/FPS/VFX/NS_Muzzle_AR` (6 emitters): **913,751 characters**,
spilled to `Saved/PinWright/HttpResponses/*.json` and parsed off disk, to answer
one boolean about one module. The equivalent `niagara.inspect` stack call is
already spilling too, so the field would be free where the caller is looking.

## Why it matters here

A one-frame additive element (a muzzle bore streak) is authored with its
`ScaleColor` fade deliberately disabled so it holds full HDR for its whole life.
Whether that disable is still in effect changes the correct lifetime by a factor
of two — and the value cannot be recovered from `moduleInputs`, because a
disabled `ScaleColor` still carries its linked `Scale Alpha` pin and its curve
data interface exactly like an enabled one.

## Ask

Add `enabled` (and, if they differ, the inherited/overridden distinction) to
`stack.modules[]` in `niagara.inspect` and in the `niagara_stack.json` sidecar,
so `set_stack_enabled` has a read path on the same surface as its write.
