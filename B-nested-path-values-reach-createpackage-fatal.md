---
id: B-nested-path-values-reach-createpackage-fatal
title: "Nested path values reach `CreatePackage`'s Fatal — the dispatch type gate only sees top-level params"
status: OPEN
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
