---
id: E-wiki-bpir-example-uncompilable-and-skel-qualifier-undocumented
title: "The BPIR wiki ships a struct-break example that cannot compile and mis-annotates its own type, and the interface page tells you to write BPI_X_C without saying the decompiler emits SKEL_BPI_X_C"
status: OPEN
severity: Medium
category: ergonomic
tags: [docs, wiki, wiki-src, bpir, break-struct, interface, message, skel-class, round-trip, copy-paste-fails]
encounters: 1
lastSeen: 2026-09-05T18:07:35Z
---

# Two BPIR pages teach things the build does not do

## 1. `bpir.examples.struct-make-break.md:14` cannot compile as written

```
%result: struct<Vector> = break<HitResult>(%hit.OutHit)
call PrintString(InString: %result.BoneName)
```

Two faults in two lines:

- **`break<HitResult>` builds a `K2Node_BreakStruct` with zero output pins.** Measured on this
  build: `get_node_details` on the produced node returns `pins:[{pinName:"HitResult",
  direction:"Input"}]` and nothing else, so every `%result.<Member>` read fails with
  `[COMPILE_FAILED] Could not resolve value '%result.BoneName' for pin ''`. The example is not
  merely stylistically off — pasted verbatim it errors. (The node-construction half of that is
  `B-bpir-break-struct-emits-generic-node`'s territory; what belongs here is that the wiki ships
  it as a worked example.)
- **The annotation is wrong independently of that.** A `HitResult` break is annotated
  `struct<Vector>`. Even had the op worked, the example teaches an incorrect type for its own
  result.

`bpir.instructions` § 2.7 presents the same construct, so the error is in two places.

The working spelling is `call BreakHitResult(Hit: )` with dotted member access
(`%b.HitBoneName`, `%b.BoneName`), which is what every real authoring path in this project ended
up using. The example should be that, or the op should be fixed and the example kept.

Aggravating detail: line 19 of that page lists a **Round-trip: `BreakStruct`** test as covering it.
A reader takes the citation as evidence the snippet above it works.

## 2. `bpir.instructions.md:47,52` promises a round trip that does not hold for Blueprint interfaces

The page instructs (`:52`) that you must write `BPI_Damageable_C`, `_C` included, and shows
(`:47`) `message BPI_Damageable_C::ApplyDamage(Target: %owner, Amount: 10.0)`. What it never says
is that the decompiler does not give that string back: it emits **`SKEL_BPI_Damageable_C::`**.

Measured on three assets, including two no agent in this project authored:

```
authored  message BPI_PWProbe_Msg_C::ProbePing(...)   ->  message SKEL_BPI_PWProbe_Msg_C::ProbePing(...)
BP_Button_Interface                                   ->  message SKEL_BPI_Player_Interactions_C::
BP_KioskButton                                        ->  message SKEL_BPInterface_Button_C::
```

It still recompiles in-session, because class resolution matches the loaded skeleton class by exact
short name and `_C` is preserved, and it does **not** reach the asset — the saved `.uasset` carries
the real interface name and zero `SKEL_` strings for it. So this is a text-fidelity gap rather than
corruption, and that is precisely why the page needs to say so: a reader diffing decompiled output
against what they wrote sees an unexplained mismatch and cannot tell which of the two is wrong.

Why no test catches it: `TestBpirInterfaceMessageRoundTrip.cpp:49` pins its fixture to the native
`/Script/UMG.UserListEntry`, which has no skeleton twin. Every Blueprint interface — which is all
of them in this project — takes the untested path.

## Fix

- Replace the `break<HitResult>` example in `bpir.examples.struct-make-break` (and § 2.7 of
  `bpir.instructions`) with the `call BreakHitResult` form, or fix the op and correct the
  `struct<Vector>` annotation. Re-check the test citation on line 19 actually covers whatever
  survives.
- State on `bpir.instructions` § "Interface calls" that the decompiler emits the `SKEL_`-qualified
  class for Blueprint interfaces, that it round-trips in-session, and that it is not what was
  written. Better still, normalise the emitter — `ShouldQualifyFunctionName` returns true for
  `UK2Node_Message` ahead of the `NormalizeToGeneratedFunction` helper that
  `B-bpir-decompile-skel-qualified-cross-class-calls` added for the sibling case, so the fix has a
  precedent in the same file.
- Add a Blueprint-interface fixture to the message round-trip test.

## History
- `#1-uncompilable-example-and-undocumented-skel-qualifier` `OPEN` reporter - Found on the FPS build after the wave 4-9 plugin rebuild, 2026-09-05, while authoring BPIR probes. `bpir.examples.struct-make-break.md:14` ships `%result: struct<Vector> = break<HitResult>(%hit.OutHit)` followed by `%result.BoneName`; on this build `break<HitResult>` produces a `K2Node_BreakStruct` with only its input pin, so the member read fails `COMPILE_FAILED`, and the `struct<Vector>` annotation is wrong for a `HitResult` regardless. `bpir.instructions` § 2.7 repeats the construct, and line 19 of the examples page cites a round-trip test as covering it. Separately, `bpir.instructions.md:47,52` instructs writing `BPI_Damageable_C` and does not disclose that the decompiler emits `SKEL_BPI_Damageable_C::`; reproduced on an authored interface and on two pre-existing assets (`BP_Button_Interface`, `BP_KioskButton`). That text recompiles in-session and does not reach the `.uasset`, so it is a fidelity gap, not corruption — but an undisclosed one, which leaves a reader diffing their own source against the decompile unable to tell which side is wrong. `TestBpirInterfaceMessageRoundTrip.cpp:49` uses the native `/Script/UMG.UserListEntry`, which has no skeleton twin, so no Blueprint interface is exercised. Same class as `E-wiki-describe-ops-boolean-material-stale`: generated pages teaching behaviour the build does not have.
