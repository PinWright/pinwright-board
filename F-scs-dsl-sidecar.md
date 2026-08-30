---
id: F-scs-dsl-sidecar
title: "Add SCS DSL sidecar for blueprint component hierarchies"
status: DONE
severity: Medium
category: feature
tags: [scs, dump-format, dsl, size-reduction]
---

# Add SCS DSL sidecar for blueprint component hierarchies

`scs.json` (Simple Construction Script — blueprint component hierarchy dump) currently totals **17 MB across 905 files** (avg 19 KB; worst 489 KB on `BP_Medieval_Building_House_VLarge`). Output is pretty JSON; per-component blocks repeat structural boilerplate (`type`, `properties`, `child_components`).

Component trees are a clean shape for a section-based DSL — same pattern that has worked for bpir.txt, mgir.txt, nir.txt, etc. Estimated compression: 5–10× (17 MB → ~2–4 MB) based on existing IR ratios.

## Proposed format

```scs
# ==== Simple Construction Script ====
# /Game/Blueprints/BP_Medieval_House.BP_Medieval_House

component(RootComponent) {
    type: CapsuleComponent
    inherits: Character.CapsuleComponent
    properties {
        CapsuleHalfHeight: 96.0
        CapsuleRadius: 40.0
        RelativeLocation: (0, 0, 0)
    }

    children {
        component(CharacterMesh0) {
            type: SkeletalMeshComponent
            properties {
                SkeletalMesh: /Game/Characters/Mannequin/Meshes/SK_Mannequin
                RelativeLocation: (0, 0, -96)
            }
        }

        component(WeaponSocket) {
            type: SceneComponent
            properties {
                SocketName: Hand_R
            }
            children {
                component(Weapon) {
                    type: StaticMeshComponent
                    properties {
                        StaticMesh: /Game/Weapons/Rifle.Rifle
                    }
                }
            }
        }
    }
}
```

Conventions match the existing IR family:
- Section headers with `==== Name ====` banners
- `key: value` property lines, indent-based nesting
- UE types written natively (`(0, 0, 0)` for vectors, `/Game/...` paths inline)
- `# comments` allowed
- Use `inherits:` to surface parent-class components (resolves the existing inherited-parent gap noted in `B-asset-dump-scs-omits-inherited-parent`).

## Implementation outline

1. New file: `Source/PinWright/Private/Handlers/Blueprint/SCSTextEmitter.{h,cpp}` (parallel to existing `NIRTextEmitter.cpp`).
2. Reuse `IrTextUtils` tokenizer infrastructure.
3. Wire `scs.txt` (or `.scs`) into `DumpFileNames`, `GetAspectVersion` (start at v1, no bump needed), and `IrSidecarRegistry`.
4. Keep `scs.json` emission gated behind a flag for one release cycle as fallback; remove once consumers (plugin tests, agents) cut over.
5. Add fixture tests under `Source/PinWright/Private/Tests/`.

Effort estimate: 3 days for emitter + tests + wire-up; +1 day if a round-trip parser is needed (probably not, since the plugin is the only writer).

**Fix:** Implement `scs.txt` emitter, add aspect versioning, and keep `scs.json` as the live/dump JSON surface. Dropping `scs.json` is not part of this ticket's implementation pass.

## History
- `#2-scs-txt-sidecar` `IN-REVIEW` implementer — added `Handlers/Blueprint/SCSTextEmitter`, emits `scs.txt` alongside existing `scs.json`, reconstructs nested `children {}` blocks from flat parent links, and covers both synthetic nested text and asset-dump smoke behavior.
- `#1-initial-proposal` `OPEN` reporter — scs.json is 17 MB across 905 files with stable schema (component → properties → children tree). Strong fit for DSL emission; 5–10× compression expected. See related B-asset-dump-scs-omits-inherited-parent.
- `#3-verify-scs-txt` `DONE` tester — Verified: ran `asset.dump` on 3 assets (B_RaceTrack_Stabilized_Trainig_06_5, B_MenuDroneSpawner, BP_Medieval_Building_DoorExtension). Each `writtenPaths` includes `scs.txt` next to `scs.json`. Content matches proposed DSL: `component(Name) { type: ..., source: scs|native, transform {...}, properties {...} }`, indent-nested blocks, native UE vector syntax, inline `/App/...` and `/Game/...` paths.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 2 body citations repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. Path(s) here that move by more than the prefix in this ticket, taken from the plugin's rename history rather than the prefix rule: `Source/EditorAutomationRpcGateway/Private/Handlers/UI/SCSTextEmitter.h` → `Source/PinWright/Private/Handlers/Blueprint/SCSTextEmitter.h`; `Source/EditorAutomationRpcGateway/Private/Handlers/UI/SCSTextEmitter.cpp` → `Source/PinWright/Private/Handlers/Blueprint/SCSTextEmitter.cpp`. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
