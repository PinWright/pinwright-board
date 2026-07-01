---
id: B-nir-duplicate-static-input-lines
title: "NIR emits duplicate consecutive 'static X = Y' lines for the same static switch input"
status: DONE
severity: Medium
category: bug
tags: [niagara, nir]
---

# NIR emits duplicate consecutive 'static X = Y' lines

In `nir.txt` output, many modules show the same `static <name> = <value>` line repeated 2-5 times consecutively. The state is identical between consecutive lines — pure textual duplication, not different versions.

Example from `Game/Effects/Particles/Item/NS_GunPad_Loading/nir.txt`:

```
module SystemState @0 enabled
    static `Inactive Response` = 1.0
    static `Inactive Response` = 1.0
    static `Loop Behavior` = 1.0
    static `Loop Behavior` = 1.0
    static `Loop Behavior` = 1.0
    static UseLoopDelay = false
    static UseLoopDelay = false
    static UseLoopDelay = false
```

The repetition appears to mirror version-history entries from `niagara_stack.json`'s `staticSwitchInputs[]` (which lists every overridden version of the input), but NIR doesn't dedupe before emitting.

## Affected files

~20+ NIR files containing `SystemState`, `EmitterState`, or similar modules with versioned static switches.

## Fix sketch

In the NIR stack-emission path: track `(moduleScope, inputName)` pairs already emitted per module and skip the duplicate. Alternatively, deduplicate `staticSwitchInputs[]` at the source by keeping only the highest-version override.

## History
- `#2-nir-parity-wave-plan` `IN-REVIEW` implementer — NIR module-row emission deduplicates identical static-switch lines without changing `niagara_stack.json` helper output.
- `#3-verify-fix` `DONE` tester — Verified: re-ran `asset.dump` on NS_GunPad_Loading and NS_WallPortal. SystemState module in NS_GunPad_Loading now shows `Inactive Response`, `Loop Behavior`, `UseLoopDelay` each exactly once (was 2x/3x/3x respectively per ticket repro); NS_WallPortal SystemState and emitter EmitterState/SpriteRendererProperties stacks also show no consecutive duplicate `static <name>` lines.
- `#1-duplicate-static-lines` `OPEN` reporter — confirmed in NS_GunPad_Loading, NS_WallPortal nir.txt at lines 9-13 (SystemState module). Sequential duplication, not semantic difference.
