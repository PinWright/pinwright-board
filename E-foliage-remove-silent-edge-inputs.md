---
id: E-foliage-remove-silent-edge-inputs
title: "foliage.remove returns an undifferentiated success:true/instancesRemoved:0 on a nonexistent (typo'd) foliageTypePath and on an omit-both call, and silently lets removeAll:true override a co-supplied foliageTypePath into a wholesale wipe — no not-found/invalid-arg error and no documented precedence"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [foliage, remove, honesty, silent-noop, precedence, removeAll, docs, verify-after-mutate]
encounters: 1
lastSeen: 2026-07-01T18:18:12.3531155+03:00
---

# `foliage.remove` never distinguishes "removed exactly what you named" from "matched nothing" or "wiped everything"

`foliage.remove` (params: optional `foliageTypePath`, optional `removeAll`) works
correctly on its two happy paths — a valid `foliageTypePath` removes ONLY that
type's instances (replay: 4 rocks removed, 3 trees left intact), and
`removeAll:true` clears every type. But across three ambiguous/edge inputs it
returns the *same* `{"success":true,"instancesRemoved":N}` shape with no honest
error and no documented precedence, so the caller cannot tell an honest scoped
removal apart from a typo, an under-specified call, or an accidental over-broad
wipe. All three are replay-confirmed live at HEAD.

## 1. Nonexistent / typo'd `foliageTypePath` -> silent false-confirmation

A `foliageTypePath` that does not resolve to any asset returns success with
`instancesRemoved:0` and **no "type not found" error** — indistinguishable from a
valid type that happened to have zero instances. On a mutation verb this is a
false-confirmation: an agent that fat-fingers a foliage-type path (common when
driving by name) is told the removal "succeeded" while its real foliage is
untouched.

Replay (7 instances present: 4 `Oracle_Rocks` + 3 `Oracle_Trees`):
- `foliage.remove {"foliageTypePath":"/Game/Foliage/Nonexistent_Garbage_ZZZ"}`
  -> `{"success":true,"instancesRemoved":0,"foliageActorPath":"...InstancedFoliageActor_0","existsAfter":true}`
- Follow-up `foliage.get_instances {}` still `count:7` — nothing removed, no error raised.

## 2. Neither param supplied (`{}`) -> silent no-op success

Calling with neither `foliageTypePath` nor `removeAll` returns success with
`instancesRemoved:0` rather than an `INVALID_ARGUMENT` prompting for one of the
two. An agent that intended a wholesale clear but forgot `removeAll:true` is told
"success" while nothing happened.

Replay: `foliage.remove {}`
-> `{"success":true,"instancesRemoved":0,...}` (7 instances still present afterward).

## 3. `removeAll:true` + a specific `foliageTypePath` -> silent over-broad wipe

Co-supplying both, `removeAll` silently wins and every type is wiped — the named
scope is ignored with no precedence noted anywhere in the docs. An agent that
names a scope (signalling scoped intent) yet also passes `removeAll:true` (a
templated/merged arg, or a misread of the two params) silently loses *all* other
types, not just the one it named.

Replay (7 instances present): `foliage.remove {"removeAll":true,"foliageTypePath":"/Game/Foliage/Oracle_Rocks"}`
-> `{"success":true,"instancesRemoved":7,...}` — all 7 (both types) gone, not the 4
`Oracle_Rocks` the path named. Follow-up `foliage.get_instances {}` returns `count:0`.

## Guilty source (`Plugins/PinWright/Source/PinWright/Private/Handlers/Environment/FoliageHandler.cpp`)

The remove body (`:303-329`) is one if/else-if with an unconditional
`success:true` tail — there is no not-found branch, no missing-args branch, and
`removeAll` is tested first so it dominates any co-supplied path:

```cpp
int32 RemovedCount = 0;

if (bRemoveAll) {                                              // :305  removeAll checked FIRST -> wins over any foliageTypePath (case 3)
  IFA->ForEachFoliageInfo([&](UFoliageType *Type, FFoliageInfo &Info) {
    RemovedCount += Info.Instances.Num();
    Info.Instances.Empty();
    return true;
  });
  IFA->Modify();
} else if (!FoliageTypePath.IsEmpty()) {                      // :312
  if (UEditorAssetLibrary::DoesAssetExist(FoliageTypePath)) { // :313  nonexistent path -> no else, RemovedCount stays 0, NO error (case 1)
    UFoliageType *FoliageType =
        LoadObject<UFoliageType>(nullptr, *FoliageTypePath);
    if (FoliageType) {
      FFoliageInfo *Info = IFA->FindInfo(FoliageType);
      if (Info) {
        RemovedCount = Info->Instances.Num();
        Info->Instances.Empty();
        IFA->Modify();
      }
    }
  }
}
// neither branch taken when !bRemoveAll && FoliageTypePath.IsEmpty() -> RemovedCount stays 0 (case 2)

TSharedPtr<FJsonObject> Resp = MakeShared<FJsonObject>();
Resp->SetBoolField(TEXT("success"), true);                    // :328  ALWAYS success, regardless of the above
Resp->SetNumberField(TEXT("instancesRemoved"), RemovedCount); // :329
```

The registry doc (`:254-258`) only says `foliageTypePath` "(omit with removeAll
for all)" and `removeAll` "Remove all foliage instances of all types" — it never
states that (a) a non-resolving path is a silent 0, (b) omitting both is a silent
0, or (c) `removeAll` overrides a co-supplied `foliageTypePath`.

## What it should do / how to fix

Make outcome distinguishable from the four inputs (all cheap, at the source):
- **Nonexistent path:** when `foliageTypePath` is set and
  `!UEditorAssetLibrary::DoesAssetExist(...)` (or the type isn't found on the
  IFA), return `ASSET_NOT_FOUND` / `TYPE_NOT_FOUND` (or at minimum add a
  `typeFound:false` field) instead of `success:true, instancesRemoved:0`, so a
  typo is not confirmed as a removal.
- **Neither param:** when `!bRemoveAll && FoliageTypePath.IsEmpty()`, return
  `INVALID_ARGUMENT` ("specify foliageTypePath or removeAll:true") instead of a
  silent no-op.
- **removeAll + path conflict:** either reject the contradictory pair, or
  document that `removeAll` wins and echo the interpretation taken (e.g.
  `mode:"all"`) so the wholesale wipe is not silent.
- **Docs:** state the precedence and the three edge behaviors on the
  `foliage.remove` overlay in `docs/wiki-src/foliage.md` (currently there is no
  per-method authoring section for `remove`).

**Workaround today:** never trust `foliage.remove`'s `instancesRemoved:0` as
proof of a scoped removal — read back with `foliage.get_instances` before/after
to confirm the target existed and only the intended type changed; never co-supply
`removeAll:true` with a `foliageTypePath` (the path is ignored).

severity rationale: impact=silent false-confirmation on a typo'd scoped remove + silent over-broad wipe when a scope is co-supplied (caller trusts a lie / loses other types) × reach=rare/edge inputs on a moderate-frequency method (happy paths are clean) -> Medium

## History
- `#1-initial-repro` `OPEN` reporter — Seed-`foliage.remove` adversarial audit
  (selective vs. wholesale removal, plus the ambiguous edge inputs the doc leaves
  open). Happy paths confirmed clean: a valid `foliageTypePath` removed only that
  type (4 `Oracle_Rocks` dropped, 3 `Oracle_Trees` intact), `removeAll:true`
  cleared all. Three edge inputs replay-confirmed live at HEAD to each return an
  undifferentiated `success:true` with no honest error/precedence:
  (1) `foliageTypePath:"/Game/Foliage/Nonexistent_Garbage_ZZZ"` ->
  `{"success":true,"instancesRemoved":0}` (7 instances still present after) — no
  "type not found"; (2) `{}` (neither param) -> `{"success":true,"instancesRemoved":0}`
  (silent no-op); (3) `{"removeAll":true,"foliageTypePath":"/Game/Foliage/Oracle_Rocks"}`
  -> `{"success":true,"instancesRemoved":7}` — wiped both types (removeAll wins,
  named scope silently ignored), `get_instances` then `count:0`. Guilty structure
  `FoliageHandler.cpp:303-329`: single if/`removeAll`-first / else-if-path with
  an `DoesAssetExist` gate that has no else, and an unconditional `success:true`
  tail (`:328-329`) — no not-found branch, no missing-args branch, no precedence.
  Dedup: ripgrep across OPEN/closed found no `foliage.remove` ticket; the three
  existing foliage tickets are add_type auto-save, get_instances scale read-back,
  and nested input schemas (different methods/direction), and the `*-silent-noop`
  bug tickets (configure-sense-config, bt-set-node-properties,
  configure-world-partition, volume-set-properties) are valid-input no-ops on
  other methods, not this invalid/edge-input honesty gap.
- `#2-fix-edge-input-honesty` `IN-REVIEW` developer — Made `foliage.remove`'s
  three ambiguous/edge outcomes distinguishable at the source
  (`Plugins/PinWright/Source/PinWright/Private/Handlers/Environment/FoliageHandler.cpp`,
  remove handler). Added two caller-input gates BEFORE the world/IFA lookup (matching
  foliage.paint / add_instances, which validate the type path before touching the
  world): case 2 (neither `foliageTypePath` nor `removeAll`) -> `INVALID_ARGUMENT`
  "specify foliageTypePath or set removeAll:true"; case 1 (a `foliageTypePath` that
  `!DoesAssetExist`) -> `ASSET_NOT_FOUND` (both codes already in ErrorCodes.h — no new
  spellings; the `removeAll`-wins path skips the case-1 gate so a bad co-supplied path
  can't turn the wipe into an error). Case 3 (precedence): the success response now
  echoes `mode` (`"all"` for a wholesale wipe, `"type"` for a scoped removal) so a
  `removeAll:true` co-supplied with a path is no longer a silent over-broad wipe. Docs:
  added a `### foliage.remove` overlay section to `Plugins/PinWright/Docs/wiki-src/foliage.md`
  stating the precedence + the three edge behaviors (previously no per-method section).
  Regression test `Plugins/PinWright/Source/PinWright/Private/Tests/Environment/TestFoliageRemoveEdgeInputHonesty.cpp`
  routes both edge payloads through the real dispatcher (FRpcDispatcher::ProcessRequest ->
  registered `foliage.remove`) and asserts exact codes `ASSET_NOT_FOUND` / `INVALID_ARGUMENT`
  (tests `PinWright.foliage.remove.NonexistentPathErrors` and `.MissingScopeErrors`); both
  run before the world/IFA lookup so they are deterministic regardless of host content and
  fail on the reverted code (which returns success:true or FOLIAGE_ACTOR_NOT_FOUND).
