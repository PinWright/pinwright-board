---
id: B-nested-path-values-reach-createpackage-fatal
title: "Nested path values reach `CreatePackage`'s Fatal — the dispatch type gate only sees top-level params"
status: IN-REVIEW
severity: High
category: bug
tags: [dispatch, param-validation, nested-params, createpackage, editor-crash, path-safety]
blockedBy: [B-createpackage-unvalidated-paths-plugin-wide]
---

# Nested path values bypass the dispatch-boundary `//` gate

`B-createpackage-unvalidated-paths-plugin-wide` closes the `CreatePackage` Fatal
by declaring path-shaped parameters as `path` / `classref` / `filepath` and
refusing `//` once, at the dispatch boundary. That gate reads **top-level wire
params only**. Paths that arrive inside an array element or as the value half of
a map are invisible to it, and still reach a load on raw caller text.

`CreatePackage` logs at **Fatal** on a package name containing `//`
(`UObjectGlobals.cpp:1094-1096`). Fatal is not compiled out in any
configuration — it ends the process and every unsaved package in it. The load
door is `StaticLoadObjectInternal` → `ResolveName2(..., Create=true)` (`:1427`)
→ `CreatePackage` (`:1310`).

## Confirmed reachers

Each found by reading the call site during the plugin-wide wave; none is
sanitized, and each takes its string from a nested value.

| Verb / surface | Nested slot | Load site |
|---|---|---|
| `gas.add_effect_modifier`, `gas.set_modifier_attribute` | `captures[].attribute` | `GASHandler.cpp:2977` → `LoadObject<UClass>` |
| `audio.music.*` | `stems[].assetPath`, `targetVoices[]`, `classAdjusters[].soundClass` | `AudioMusicHandler.cpp` ~`:1744` |
| `material.graph.create_nodes` | `nodes[].texturePath` | `MaterialGraphHandler.cpp:587` → `LoadObject<UTexture>` |
| `material.create_material_instance`, `material.set_material_instance_parameters` | `texture: {ParamName: AssetPath}` | `MaterialInstanceOverrides.h:116` → `LoadObject<UTexture>` |
| `foliage.create_procedural` | `foliageTypes[].meshPath` | `FoliageHandler.cpp:2419` → `LoadObject<UStaticMesh>`, zero sanitization |

## Why the existing `NestedKeys` mechanism cannot close this

Three independent blockers, each found by a different cluster agent:

1. **`FParamSpec::NestedKeys` is an untyped allow-list.** It can say a key is
   permitted; it cannot say a key is a *path*. There is nowhere to hang the
   `//` rule.
2. **The paths are often values, not keys.** `texture: {ParamName: AssetPath}`
   carries the path in the value half of a map, and `nodes[].texturePath` /
   `foliageTypes[].meshPath` carry it inside array elements. A key-shaped gate
   sees neither. `create_material_instance` *already* declares `NestedKeys` and
   is still exposed.
3. **Adopting the allow-list is a contract narrowing on at least one verb.**
   `foliage.create_procedural` deliberately tolerates unread per-type keys and
   echoes them (`UnreadPerTypeFields`, `FoliageHandler.cpp:2382-2384`); its
   description ends "Unread keys you pass are echoed in `ignoredFields`".
   Turning `NestedKeys` into a closed set there breaks documented behaviour.

There is also **no required-param nested macro** — `RPC_PARAM_OPT_NESTED`
hardcodes `bRequired=false`, while the real case (`foliageTypes`) is required.

## What a fix needs

- **Typed nested keys** — a nested key must be able to declare `path` /
  `classref` / `filepath`, so the same one-line `//` rule applies to it.
- **Element-level checking for arrays** — the top-level gate unions over param
  values; it must be able to descend into array elements. Note that a
  `path|array` union on an array slot buys **zero** protection and only widens
  the slot to accept a scalar the handler then drops.
- **Map-value checking** — for `{ParamName: AssetPath}` shapes, where the path
  is neither the key nor a top-level value.
- **A required-nested macro**, or `bRequired` threaded through the existing one.
- **Open-key-set semantics preserved** — declaring which nested keys are paths
  must not imply that undeclared keys are refused, or `foliage.create_procedural`
  regresses.

## Not a workaround

Guarding the five load sites individually was considered and rejected for the
same reason the parent ticket rejected it at 385 sites: it does not prevent the
sixth, and it re-grows the validation layer the parent wave shrank. The
mechanism gap is the defect.

## History

- `#1-filed-from-plugin-wide-wave` **OPEN** (Reporter) — Filed during the
  `B-createpackage-unvalidated-paths-plugin-wide` implementation wave. Three
  cluster agents independently reported the nested layer could not be adopted;
  nested-key adoption was descoped from that wave rather than forcing a
  mechanism that does not fit. Blocked on the parent ticket landing first, since
  the type tokens and the `//` rule are prerequisites.
- `#2-guarded-at-the-point-of-use` **IN-REVIEW** (Developer) — Verified TRUE from
  source; four of the five reachers were still raw loads (the two audio rows in the table are
  wrong, see the Fix section). Closed at the point of use rather than by typing nested keys:
  `PinWrightGuardedLoad::LoadObjectChecked` (`Utils/GuardedLoad.h`) guards where the load
  happens, so it is blind to input shape and needs no new declarative mechanism, no required-
  nested macro, and no narrowing of `foliage.create_procedural`'s open key set. Same helper
  closes `B-ir-source-class-refs-reach-createpackage-fatal`. Not compiled or run by the
  implementing agent; a separate compile pass follows.

## Fix

Confirmed TRUE by reading source on 2026-09-03; four of the five reachers were still raw.
Closed at the **point of use** rather than by typing nested keys, because a guard keyed on where
the load happens cannot be bypassed by input shape - which is the property the ticket's
"typed nested keys + array descent + map-value rule + required-nested macro" design was buying at
the cost of four new declarative mechanisms, and it also answers the IR-text sibling ticket and
internally-composed strings for free. `foliage.create_procedural`'s open-key-set contract is
untouched, because nothing was declared.

New shared guard: `Source/PinWright/Private/Utils/GuardedLoad.h` -
`PinWrightGuardedLoad::LoadObjectChecked<T>(Path, OutRefusal?, LoadFlags?)`, a drop-in for
`LoadObject<T>(nullptr, *Path)` built on `CanReachCreatePackageFatal`. Refuses `//`, logs at
Warning (never Error - `bElevateLogWarningsToErrors`), fills an optional caller-facing reason.

Reachers converted (4 of 5 - see "correction" below):
- `Handlers/Systems/GASHandler.cpp` `ResolveGameplayAttributeFromSpec` (`captures[].attribute`,
  also the shared resolver behind `gas.add_effect_modifier` / `gas.set_modifier_attribute`);
  refusal now reports `INVALID_ARGUMENT` instead of `ATTRIBUTE_SET_NOT_FOUND`.
- `Handlers/Audio/AudioMusicHandler.cpp` (`stems[].assetPath`), keeping `LOAD_NoWarn|LOAD_Quiet`.
- `Handlers/Material/MaterialGraphHandler.cpp` (`nodes[].texturePath`).
- `Handlers/Material/MaterialInstanceOverrides.h` (`texture: {ParamName: AssetPath}` - the map-VALUE
  shape); the refusal text now rides the existing per-parameter `failed[]` entry.
- `Handlers/Environment/FoliageHandler.cpp` (`foliageTypes[].meshPath`); refusal rides
  `NoteSkippedType`.

**Correction to the ticket's table:** the audio `targetVoices[]` and `classAdjusters[].soundClass`
rows are NOT reachers. Both resolve through `NormalizeContentAssetPath`
(`AudioAuthoringHandler.cpp:261-265`, `:2439`), which runs `FPaths::RemoveDuplicateSlashes` before
the load. They are silent-collapse cases, not editor kills. `targetVoices[]` is additionally an
array of PLAIN strings, so it is expressible at the boundary today as `path|array` - no mechanism
change needed for that shape at all.

Docs: `Docs/rpc-design.md` gains section 24 (and section 23's "what this does not cover" paragraph
now points at it); `Handlers/ParamTypeCheck.h` and `Utils/PathUtils.h` header contracts updated to
name the second layer.

### Verification

No editor run is needed for the code review; the tests are automation tests.

1. Read `Source/PinWright/Private/Utils/GuardedLoad.h` - the rule must be exactly
   `CanReachCreatePackageFatal`, nothing wider (a class slot legally holds a bare short name and a
   dotted object path).
2. Run `PinWright.Core.Path.GuardedLoad.*` (3 tests,
   `Private/Tests/Core/TestGuardedLoadPathSafety.cpp`) - refusal contract, the six legal shapes that
   must still be accepted, and that a refused path is never handed to a loader.
3. Run `PinWright.infra.contract.LoadGuard.NestedValueFilesDoNotRegrow`
   (`Private/Tests/Infra/TestIrAndNestedLoadGuard.cpp`) - a counted source-scan ratchet over the
   five files, so a NEW nested raw load in any of them goes red. Counts were measured post-fix:
   AudioMusicHandler 0, FoliageHandler 12, MaterialGraphHandler 2, MaterialInstanceOverrides 0,
   GASHandler 23. The non-zero remainders are TOP-LEVEL `path` params already covered at the
   dispatch boundary.
4. Do NOT try to drive a `//` through these verbs on a build without the fix: `CreatePackage`'s
   Fatal ends the suite host rather than reporting a red.

Not committed (board and plugin repos both left dirty, per the task).
