---
id: B-nir-undefined-parameter-name-leak
title: "NIR emits 'Undefined parameter name' placeholder for unresolved pins instead of erroring"
status: DONE
severity: High
category: bug
tags: [niagara, nir]
---

# NIR emits 'Undefined parameter name' placeholder for unresolved pins

When the NIR decompiler can't resolve a module pin (custom modules with unusual pin shapes, dynamic-input-bound pins, or missing graph context), it falls back to emitting:

```
static `Undefined parameter name` = 0.0
```

instead of either (a) resolving the pin via graph connectivity, or (b) erroring loudly so the consumer knows the dump is partial.

Result: dump output looks valid but conveys no semantic information for the affected modules.

## Affected

~20+ NIR files contain this pattern. Notable examples:
- `Game/Effects/Particles/Item/NS_GunPad_Loading/nir.txt` line 107 (ScaleColor module)
- `Game/Characters/.../NS_CharacterPortalDissolve/nir.txt` lines 123-137 (SampleSkeletalMesh module — 15 instances in one emitter)

## Fix sketch

Two-pronged:
1. Improve pin resolution in `NIRDecompiler.cpp` — traverse graph links and module input metadata to recover the actual parameter name when the direct lookup fails.
2. When resolution still fails: emit a structured comment like `/* unresolved pin: index=3, type=NiagaraFloat */` AND record an issue in `niagara_compile.json` with code `NIR_UNRESOLVED_PIN`. Do not silently emit `Undefined parameter name`.

## History
- `#2-nir-parity-wave-plan` `IN-REVIEW` implementer — NIR static-switch emission suppresses the Unreal `Undefined parameter name` sentinel and emits `NIR_UNRESOLVED_PIN` diagnostics instead of a misleading static assignment.
- `#1-undefined-pin-leak` `OPEN` reporter — placeholder string silently leaks into NIR output. High severity because it looks like real data but isn't. Especially common on SampleSkeletalMesh, ScaleColor modules.
- `#3-verify-fix` `DONE` tester — Verified: fresh asset.dump of /Game/Effects/Particles/Item/NS_GunPad_Loading and /Game/Effects/Particles/Environmental/NS_CharacterPortalDissolve. Zero occurrences of `Undefined parameter name` in either nir.txt; `NIR_UNRESOLVED_PIN` warning comments present (10+ in GunPad_Loading, 300 in CharacterPortalDissolve).
