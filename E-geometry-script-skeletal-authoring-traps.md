---
id: E-geometry-script-skeletal-authoring-traps
title: "Two undocumented Geometry Script skeletal traps: mesh_create_bone_weights must run AFTER geometry is appended, and the Python bindings are unreal.GeometryScript_*, not GeometryScriptLibrary_*"
status: OPEN
severity: Medium
category: ergonomic
tags: [geometry-script, bone-weights, skeletal-mesh, python-execute, silent-failure, ordering-trap, docs]
---

# Geometry Script skeletal authoring: an ordering trap and a binding-name trap

Two live-verified traps from this host project's skeletal pipeline. Both cost real debugging time,
neither is written down anywhere in the plugin, and the first one is a **hard constraint on any
future typed skeletal verb** — a verb that does not enforce it reproduces the same silent failure
with a friendlier name.

## Trap 1 — `mesh_create_bone_weights` must run AFTER all geometry is appended

`mesh_create_bone_weights` allocates the bone-weight attribute for the vertices that exist **at the
moment it is called**. Call it on an empty or partially built mesh and every vertex appended
afterwards has no weight entry.

The failure is silent at the point of the mistake and opaque at the point of detection:

1. `mesh_create_bone_weights` succeeds.
2. Geometry is appended.
3. `set_vertex_bone_weights` returns `is_valid=False` for the later vertices — **no exception, no
   log line**; the return value is the only signal, and it is easy to discard.
4. Asset creation then fails with *"LOD 0 has no skin weight attributes"* — an error that names
   neither the ordering nor the call that caused it.

Reproduced independently in this project. The correct order is: append **all** geometry →
`mesh_create_bone_weights` → `set_vertex_bone_weights` / `compute_smooth_bone_weights`.

**Constraint on future work:** a typed `geometry.*` bone-weight verb must either enforce this
ordering internally (a composite `bind_skin_weights` that appends-then-creates-then-sets is
strictly better than three separate verbs) or detect and error on it explicitly. Exposing
`mesh_create_bone_weights` as a standalone verb without that guard just relocates the trap. See
`F-geometry-skeletal-mesh-roundtrip-verbs`.

## Trap 2 — the Python bindings are `unreal.GeometryScript_*`, not `GeometryScriptLibrary_*`

This build exposes Geometry Script to Python as `unreal.GeometryScript_AssetUtils`,
`unreal.GeometryScript_BoneWeights`, `unreal.GeometryScript_Primitives`, `unreal.GeometryScript_MeshEdits`,
`unreal.GeometryScript_MeshQueries`, `unreal.GeometryScript_MeshSpatial`, `unreal.GeometryScript_Materials`,
`unreal.GeometryScript_NewAssetUtils` — **not** the `unreal.GeometryScriptLibrary_*` names that most
documentation (and the C++ class names) use. Cause is the engine's own
`UCLASS(meta=(ScriptName="GeometryScript_AssetUtils"))` binding, so the C++ symbol and the Python
symbol legitimately differ. Live-verified in this project, not a doc error.

Every `python.execute` caller hits this on their first line and has no way to know it in advance.
Two related name hazards observed in the same corpus:

- `get_num_triangle_i_ds` and `get_num_triangle_ids` **both** occur across API drift; working
  scripts defensively `hasattr()`-probe for the right spelling.
- `set_vertex_position` lives on `GeometryScript_MeshEdits` — probing four plausible class names
  was required to find it.

## Fix (docs)

Record both in the plugin docs, where a caller reaching for skeletal work will see them:

- `docs/wiki-src/geometry.md` — the binding-name mapping (`unreal.GeometryScript_*`, with the
  `ScriptName` meta as the reason) and the `hasattr()` caveat for the `_ids`/`_i_ds` drift, so
  `python.execute` callers do not burn a round-trip on `AttributeError`.
- `docs/wiki-src/skeleton.md` and/or `Docs/lessons.md` — the bone-weight ordering trap as a
  one-liner in the standing UE-API-trap list, phrased as the invariant
  (*geometry first, then `mesh_create_bone_weights`, then weights*) plus the exact downstream error
  string *"LOD 0 has no skin weight attributes"* so a search for the symptom finds the cause.

`Docs/lessons.md` is the right home for trap 1 specifically: it is the plugin's existing
one-line-UE-API-trap file, and this is exactly that shape.

severity rationale: impact=soft blocker — both are surmountable, but only via a source dive or trial
and error, and trap 1 fails silently and reports a downstream error that points at the wrong place ×
reach=rare (skeletal authoring through `python.execute`), bumped up because trap 1 is also a
correctness constraint on the future typed verbs -> Medium

## Relationship to other tickets

- `F-geometry-skeletal-mesh-roundtrip-verbs` — trap 1 is an acceptance constraint on that ticket's
  bone-weight verbs, not merely a doc note.
- `E-geometry-namespace-skeletal-scope-undocumented` — the reason callers are in `python.execute`
  for skeletal work at all.

## History
- `#1-triage-ordering-and-binding-traps` `OPEN` reporter — Filed from a mesh/skeletal authoring triage against this host project's shipped skeletal pipeline (plugin HEAD `9da255f6d0ef5017cfddee03cd9458266c99d764`). Trap 1, independently reproduced: `mesh_create_bone_weights` allocates bone-weight attributes only for vertices that exist when it is called, so calling it before geometry is appended makes `set_vertex_bone_weights` silently return `is_valid=False` for every later vertex and asset creation then fails with "LOD 0 has no skin weight attributes" — an error naming neither the ordering nor the offending call. Correct order: append all geometry → `mesh_create_bone_weights` → set/compute weights. Any future typed bone-weight verb must enforce or detect this, ideally as one composite `bind_skin_weights`, or it reproduces the silent failure. Trap 2, live-verified: this build binds Geometry Script to Python as `unreal.GeometryScript_*` (`GeometryScript_AssetUtils`, `GeometryScript_BoneWeights`, `GeometryScript_Primitives`, ...), not the documented `unreal.GeometryScriptLibrary_*`, because of the engine's `UCLASS(meta=(ScriptName=...))` binding — so the C++ and Python symbols legitimately differ and every `python.execute` caller hits it on line one. Same corpus also shows both `get_num_triangle_ids` and `get_num_triangle_i_ds` spellings in play (scripts `hasattr()`-probe) and `set_vertex_position` living on `GeometryScript_MeshEdits`. Fix is docs: binding names + `_ids` drift into `docs/wiki-src/geometry.md`, the ordering invariant plus the verbatim downstream error string into `Docs/lessons.md` / `docs/wiki-src/skeleton.md`. Neither trap was previously on the board.
