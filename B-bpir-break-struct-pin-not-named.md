---
id: B-bpir-break-struct-pin-not-named
title: "blueprint.decompile renders a Break Hit Result node's near-synonym output pins (BoneName vs HitBoneName) indistinguishably, so an IR-only review cannot tell which one feeds a consumer — this hid a real High-severity headshot defect in BP_WeaponBase"
status: OPEN
severity: High
category: bug
tags: [bpir, decompile, break-struct, hit-result, pin-names, near-synonym, review-blind-spot, get_node_connections, blueprint]
encounters: 1
lastSeen: 2026-09-03T00:00:00Z
---

# A struct-break read in decompiled BPIR does not name the pin it reads

## What was called

```
blueprint.decompile {assetPath: "/Game/FPS/Weapons/BP_WeaponBase"}
```

## What happened

The `ProcessHit` graph contains a `Break Hit Result` node. `FHitResult` carries **two**
bone-name fields whose Blueprint pin display names are near-synonyms:

| pin | backing field | value for a line trace against a skeletal mesh |
|---|---|---|
| `Hit Bone Name` | `FHitResult::BoneName` | the bone actually hit (e.g. `head`) |
| `Bone Name` | `FHitResult::MyBoneName` | the *tracing* component's bone — `None` for a line trace |

The mapping was read in engine source:
`C:/UE_5.8/Engine/Source/Runtime/Engine/Private/GameplayStatics.cpp:1409-1410`
(the break function's out-parameter assignment), and the semantics comment distinguishing the
two is at `HitResult.h:143`.

In the decompiled BPIR the two outputs of that break node render **indistinguishably** — the
IR text does not carry the source pin on a struct-break read, so a reviewer reading the
decompile has no way to tell which of the two fields feeds the downstream comparison.

## Why this is High, not cosmetic

The game code under review is wired from **`BoneName`** (`MyBoneName`, always `None` for the
line trace this weapon fires), not from **`HitBoneName`**. Every headshot test in `ProcessHit`
therefore evaluates false — a real gameplay defect. The BPIR read is the surface a reviewer
uses to audit exactly this kind of wiring, and it was the surface that let the defect through:
the decompile *looked* correct.

Establishing the truth cost a second, targeted round-trip on a node id that had to be
recovered first:

```
blueprint.get_node_connections {assetPath: "/Game/FPS/Weapons/BP_WeaponBase",
                                nodeId: "DF4FE15F4946DC757DE3E8A81129E006"}
```

Only that call named the pin at the source end of the wire. A reviewer who trusts the
decompile — the documented, human-readable review surface — will not make that call, because
the decompile gives no signal that anything is ambiguous.

## What was expected

The IR should name the source pin explicitly on a struct-break read, so the two forms are
textually distinct in the decompiled body:

```
%hit_bone = %n2.HitBoneName      # FHitResult::BoneName    — the bone that was hit
%my_bone  = %n2.BoneName         # FHitResult::MyBoneName  — None for a line trace
```

That is enough to make the defect visible on a plain read, with no follow-up RPC.

## Root cause — GUESS, not source-read

No PinWright source was read for this ticket. The engine-side field mapping above **was**
read and is fact; the plugin-side mechanism is inference: the decompiler appears to render a
struct-break consumer by the *struct member* it resolves to rather than by the *pin* it is
wired from, or to collapse both pins onto one register name. Whoever picks this up should
confirm against the break-struct emit path in the decompiler before fixing.

**Workaround:** never trust a struct-break read in decompiled BPIR when the struct has
near-synonym fields (`FHitResult` `BoneName`/`MyBoneName`, `Component`/`MyComponent`,
`Actor`/`MyItem`); recover the node id and issue `blueprint.get_node_connections` per break
node to read the pin at the source end.

**Fix:** emit the source pin name on every struct-break read in the decompiled body. If the
register form cannot carry it, a trailing comment naming the pin (and, where the display name
diverges from the backing field, the field) is enough to make the review honest.

## Related

- `B-bpir-break-struct-emits-generic-node` (OPEN) — the *authoring* side of `break<T>` on
  native-break structs; `FHitResult` is named there as likely affected. Different direction
  (compile, not decompile), same struct family.
- `B-decompile-struct-subfield-dropped` (IN-REVIEW) — reflected struct **property** emit
  dropping sub-fields; that is IrCore's `ExportText` default-suppression, not pin naming.
- `E-bpir-dollar-chained-struct-field` (DONE) — chained `$name.a.b` access on the compile path.
- `E-decompile-bpir-text-not-verbatim-roundtrip` — the general "decompile is not the graph"
  gap this is a concrete, costly instance of.

## History
- `#1-filed` `OPEN` WEAPONS-critic — Found during a WEAPONS critic review. `blueprint.decompile {assetPath:"/Game/FPS/Weapons/BP_WeaponBase"}` renders the `Break Hit Result` node in `ProcessHit` such that its `BoneName` and `HitBoneName` outputs are indistinguishable in the IR text, so an IR-only read cannot tell which pin feeds the downstream headshot comparison. The graph is in fact wired from `BoneName` — `FHitResult::MyBoneName`, which is `None` for a line trace — instead of `HitBoneName` (`FHitResult::BoneName`), so every headshot test evaluates false; that is a real High game defect the decompile hid. Engine mapping read directly at `C:/UE_5.8/Engine/Source/Runtime/Engine/Private/GameplayStatics.cpp:1409-1410`, pin semantics comment at `HitResult.h:143` — those two citations are measured, not inferred. Disambiguating the wire required a second targeted `blueprint.get_node_connections` round-trip on node id `DF4FE15F4946DC757DE3E8A81129E006`, which a reviewer trusting the decompile has no reason to make. Plugin-side root cause is an explicit GUESS (no PinWright source was read): the decompiler appears to render a struct-break consumer by resolved struct member rather than by the pin it is wired from. Ask: name the source pin explicitly on a struct-break read so `%n2.HitBoneName` and `%n2.BoneName` are textually distinct in the body.
