---
id: B-bpir-break-struct-pin-not-named
title: "blueprint.decompile renders a Break Hit Result node's near-synonym output pins (BoneName vs HitBoneName) indistinguishably, so an IR-only review cannot tell which one feeds a consumer — this hid a real High-severity headshot defect in BP_WeaponBase"
status: IN-REVIEW
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

## Fix

**The filed mechanism was wrong; the symptom is real for one shape.** An explicit
`Break Hit Result` node has always named its source pin: `ResolveInputValue` renders every
non-`ReturnValue` output as `%reg.<PinName>` using the raw `PinName`
(`Decompiler/BpirDecompiler.cpp:2211-2217, 2237-2243`, via `FormatOutputPinNameToken`), and
nothing in the BPIR emit path reads a pin display name — the only `Pin->GetDisplayName()`
call in the decompiler side is `Compiler/NodeLayoutEngine.cpp:85`, used for layout width.
A committed dump confirms the shipped spelling: `asset-dumps/Game/Effects/Blueprints/B_FootStep/bpir.txt:56-58`
emits `%n14 = call BreakHitResult(...)` then `%n14.Location` / `%n14.Normal`.

The real hole is the **split struct pin** — the graph's *inline* break, which the decompiler
did not model at all. `UEdGraphSchema_K2::SplitPin` (`EdGraphSchema_K2.cpp:7422-7461`) hides
the struct pin and moves every wire onto sub-pins named `<ParentPinName>_<MemberName>`, with
`ParentPin` set. The decompiler resolved from the sub-pin as if it were the node's value:

- a split **variable read** collapsed *every* member onto the bare `$Var` — two members of
  one struct rendered identically **and the member was dropped**, so a recompile wired the
  whole struct. This is exactly the reported symptom, and worse than reported;
- elsewhere it printed the raw `Parent_Member` sub-pin name (`%n1.ReturnValue_HitBoneName`),
  which no freshly created node carries, so the recompile could not resolve it either;
- a split output also leaked into `EmitEntrySignature`, which listed an event's `FHitResult`
  parameter as a flat member list instead of `struct<HitResult> HitInfo`, and into the
  `%name: <type>` annotation, which named the first member's type instead of the struct.

Fix: one shared pair of helpers in the decompiler's anonymous namespace — `GetSplitPinRoot`
(walk `ParentPin` to the pin that carries the value) and `FormatSplitMemberSuffix` (dotted
member path, each segment `FormatNameToken`-formatted, prefix-stripped from the sub-pin
name). Every value-reference site resolves against the root and re-attaches the suffix, so a
split read decompiles to the dotted access the compiler already re-resolves through
`ResolveChainFromPin` / the struct-`ReturnValue` fallback: `$Hit.HitBoneName`,
`%n0.OutHit.BoneName`, `%n0.BoneName`. **No compiler change was needed** — that spelling was
already accepted (`Compiler/BpirValueResolver.cpp:345-395, 883-970, 1176-1290`), which is
why this shape was chosen over inventing a split-pin syntax: BPIR gains no new grammar, and
the recompiled graph is an explicit break node, semantically identical.

Files changed:

- `Plugins/PinWright/Source/PinWright/Private/Decompiler/BpirDecompiler.cpp` — new
  `GetSplitPinRoot` / `FormatSplitMemberSuffix`; applied in `TryFormatEntryParam`, the
  `UK2Node_VariableGet` and `UK2Node_VariableSet` branches, and both `%reg` reference sites.
- `Plugins/PinWright/Source/PinWright/Private/Decompiler/BpirTextEmitter.cpp` —
  `IsPrimaryOutputDataPin` admits a hidden pin that has `SubPins` and rejects sub-pins, so
  the split parent is the primary output for entry signatures and type annotations.
- `Plugins/PinWright/Source/PinWright/Private/Tests/Bpir/TestBpirStructBreakPinNaming.cpp` —
  new; three decompile+recompile tests, each wiring **both** bone pins to distinct consumers:
  `PinWright.bpir.round_trip.BreakHitResultBonePinsNamed` (explicit break node — the ticket's
  literal scenario), `.SplitVariableStructPinNamed`, `.SplitReturnValueStructPinNamed`. Each
  asserts the two reads are textually distinct *and* that the recompiled graph's
  `set StoredHitBone` / `set StoredTraceBone` value pins trace back to `HitBoneName` /
  `BoneName` respectively — text distinctness alone cannot prove that.
- `Plugins/PinWright/docs/wiki-src/bpir.md`, `bpir.instructions.md` — document the dotted
  member spelling for split pins and the pin-name-not-display-name rule.

**Not compiled and not run** — a separate compile pass follows.

**Reviewer verification.** Run the three new tests plus the existing decompiler/round-trip
groups (`PinWright.bpir.decompiler.*`, `PinWright.bpir.round_trip.*`) in one editor. Then
re-dump one asset that contains a split struct pin and confirm the churn is only
`Parent_Member` → `Parent.Member` (and, where a split pin was the primary output, a corrected
`: struct<T>` annotation or entry-parameter list). `asset-dumps/Game/GameplayCueNotifies/GCNL_Character_DamageTaken/bpir.txt:22-23`
is a live instance: `%n1.ReturnValue_HitBoneName` there should become `%n1.HitBoneName`.

**Known adjacent gap, deliberately out of scope:** split *input* pins still emit one argument
per sub-pin (`FormatArgs` skips the hidden parent and names each sub-pin `Hit_Location`), which
a fresh node cannot resolve on recompile. That is the write side of the same blindness and
wants its own ticket.

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
- `#2-fixed-split-pin-blindness` `IN-REVIEW` developer — Verified against plugin source, no editor. The filed mechanism is FALSE: the decompiler already renders `%reg.<PinName>` from the raw pin name (`Decompiler/BpirDecompiler.cpp:2211-2217, 2237-2243`) and reads no pin display name anywhere in the BPIR emit path, so an explicit `Break Hit Result` node did name its pin — `asset-dumps/Game/Effects/Blueprints/B_FootStep/bpir.txt:56-58` shows the shipped `%n14.Location` / `%n14.Normal` form. The symptom is nonetheless real for the graph's *inline* break, a split struct pin, which the decompiler did not model at all: a split variable read collapsed every member onto the bare `$Var` (dropping the member entirely), and elsewhere it printed the raw `<Parent>_<Member>` sub-pin name, which no freshly created node carries. Fixed with one shared root/member-suffix helper pair applied at every value-reference site, emitting the dotted access the compiler already accepts (`$Hit.HitBoneName`, `%n0.OutHit.BoneName`, `%n0.BoneName`); `IsPrimaryOutputDataPin` now treats a split parent, not its sub-pins, as the node's primary output so entry signatures and `%name: <type>` annotations name the struct. Three decompile+recompile tests added in `Tests/Bpir/TestBpirStructBreakPinNaming.cpp`. Not compiled, not run — see the Fix section for reviewer verification and the out-of-scope split-*input*-pin gap.
- `#3-verified-in-fps-build` `IN-REVIEW` FPS-tester — Re-verified `#2`'s fix in a live editor after the 2026-09-05 plugin pull. **Both halves hold.** (a) *The ticket's literal scenario.* Probe `blueprint.create {name:"BP_PWProbe_Break", savePath:"/Game/FPS/Weapons/Test", parentClass:"Actor"}` plus two `name` variables, then `blueprint.compile_bpir {assetPath:"/Game/FPS/Weapons/Test/BP_PWProbe_Break", mode:"append", code:"entry function ProbeBreak(struct<HitResult> Hit) { %b = call BreakHitResult(Hit: $Hit); set StoredHitBone = %b.HitBoneName; set StoredTraceBone = %b.BoneName }"}` → `nodeCount:3, errors:[], warnings:[]`. Graph truth from `blueprint.graph.get_execution_flow {graphName:"ProbeBreak", includeDataInputs:true}`: `Set StoredHitBone` has `dataInputs:[{pin:"StoredHitBone", source:"678B08324A4780018D01D4B8A2F65C3B", sourcePin:"HitBoneName"}]` and `Set StoredTraceBone` has `sourcePin:"BoneName"` off the **same** break node — the two near-synonym pins are wired to distinct consumers exactly as authored. `blueprint.decompile_function {functionName:"ProbeBreak"}` renders them textually distinct — `set StoredHitBone = %n0.HitBoneName` / `set StoredTraceBone = %n0.BoneName` — confirming `#2`'s reading that an explicit break node always named its source pin and that `#1`'s filed mechanism was false. Round-tripped: recompiling that decompiled text verbatim as `ProbeBreakRT` rebuilt the mapping on a fresh node (`AA2355954C891D7D606487BBC85656EA`) with `sourcePin:"HitBoneName"` / `sourcePin:"BoneName"` intact, so the text distinctness is load-bearing and not cosmetic. `blueprint.graph.find_orphaned_nodes` → `orphanedCount:0` over `["EventGraph","UserConstructionScript","ProbeBreak","ProbeBreakRT"]`, `totalNodes:12`. (b) *The split struct pin — the real defect `#2` fixed.* No PinWright verb authors a split pin (`blueprint.graph.create_node` has no split operation and BPIR emits explicit breaks), so this was verified against a pre-existing asset with its committed pre-fix dump as the control: `blueprint.decompile {assetPath:"/Game/ExampleContent/PhysicsControl/Blueprints/BP_PhysicsBox", graphName:"EventGraph"}`. `asset-dumps/Game/ExampleContent/PhysicsControl/Blueprints/BP_PhysicsBox/bpir.txt:8-10` (committed, pre-fix) reads `%n0: struct<Vector> = call K2_GetComponentToWorld(Target: $Cube)` / `set InitialPosition = %n0.ReturnValue_Location` / `set InitialOrientation = %n0.ReturnValue_Rotation`. The same three lines now decompile as `%n0: struct<Transform> = call K2_GetComponentToWorld(Target: $Cube)` / `set InitialPosition = %n0.Location` / `set InitialOrientation = %n0.Rotation` — the prefix-stripped dotted member access from `GetSplitPinRoot` / `FormatSplitMemberSuffix`, and the corrected `: struct<Transform>` annotation from `IsPrimaryOutputDataPin` treating the split parent rather than its first sub-pin as the node's primary output. Both changes land in one asset, and the churn is exactly the `Parent_Member` → `Parent.Member` shape the Fix section predicted. Note the instance the Fix section cites, `asset-dumps/Game/GameplayCueNotifies/GCNL_Character_DamageTaken/bpir.txt:22-23`, **does not exist in this checkout** — that citation is from another host's tree; `BP_PhysicsBox` is the local equivalent. (c) *Split **input** pins, listed in the Fix section as "known adjacent gap, deliberately out of scope", are fixed too.* The same decompile emits `%split_4ABA15E54FE58C9E83B510B0CC4795C3_0 = make<PhysicsControlModifierData>(MovementType: EPhysicsMovementType::Kinematic, CollisionType: ECollisionEnabled::QueryAndPhysics, …)` followed by `call CreateBodyModifier(…, BodyModifierData: %split_4ABA15E54FE58C9E83B510B0CC4795C3_0)`, where the pre-fix dump line 7 had one argument per sub-pin (`BodyModifierData_MovementType: …`, `BodyModifierData_CollisionType: …`) that no freshly created node could resolve. `bpir.instructions` §2.7 documents the generated-`make` behaviour, so the ticket text is behind the shipped build here — the adjacent-gap ticket the Fix section asks for is not needed. **Adjacent defect hit while building the probe, belonging to `B-bpir-break-struct-emits-generic-node`, not here:** `break<HitResult>($Hit)` compiles clean (`nodeCount:2, errors:[], warnings:[]`) and builds a `K2Node_BreakStruct` titled "Break Hit Result" (`417ACA1F441D1B2087F0F9B574E214EA`) with **zero output pins** — `blueprint.graph.get_node_details` reports `pins:[{pinName:"HitResult", direction:"Input", pinType:"struct"}]` and nothing more — so every member read against it dies with `[COMPILE_FAILED] Line 3: Could not resolve value '%b.HitBoneName' for pin ''`. FHitResult has a native break (`UGameplayStatics::BreakHitResult`), which is what the editor and the shipped `B_FootStep` dump both use; it is recorded here only because it is what forces this probe onto `call BreakHitResult(...)`. Minor cosmetic observation, no ticket filed: that call decompiles with the annotation `%n0: bool = call BreakHitResult(...)` — the first out-parameter's type rather than the struct being broken. **On disk:** `Content/FPS/Weapons/Test/BP_PWProbe_Break.uasset` 26465 bytes / mtime 2026-09-05 20:48:31 → 66850 bytes / 20:51:04 across `asset.save {force:true}` reporting `saveState:"written"`; `grep -a` counts were 0 for `ProbeBreak`, `ProbeBreakRT` and `BreakHitResult` before the save and 8 / 4 / 19 after, with `HitBoneName` 2 and `BoneName` 4 (two of those four being the tail of `HitBoneName`) — both pin names durable in the file. Probe asset is throwaway. Status left `IN-REVIEW`; no encounter recorded, this is a pass not a failure.
