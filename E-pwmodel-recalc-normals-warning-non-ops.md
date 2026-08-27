---
id: E-pwmodel-recalc-normals-warning-non-ops
title: "`recalculate_normals`' own warning tells a `.pwmodel` author to write `Set Mesh To Per Vertex Normals` or `Compute Split Normals` — two engine verb names that are not ops in this format, so following the remedy costs a compile cycle to `PWSRC_UNKNOWN_OP`"
status: OPEN
severity: Medium
category: ergonomic
tags: [pwmodel, diagnostics, recalculate_normals, append_buffers, PWMODEL_STAGE_WARNING, PWSRC_UNKNOWN_OP, warning-text, vocabulary-leak]
encounters: 1
lastSeen: 2026-08-27T18:57:03+05:00
---

# A warning whose remedy cannot compile

Running `recalculate_normals` on geometry from an `append_buffers` op that supplied no `normals=`
buffer returns:

```
PWMODEL_STAGE_WARNING: 'recalculate_normals': RecomputeNormals: TargetMesh did not have
normals to recompute; falling back to per-vertex normals. Consider using 'Set Mesh To Per
Vertex Normals' or 'Compute Split Normals' instead.
```

Neither `Set Mesh To Per Vertex Normals` nor `Compute Split Normals` is an op in this format.
`model.describe_ops` lists neither, and writing either one gets `PWSRC_UNKNOWN_OP`. The op in the
table that wraps the second is **`split_normals`**; the first has **no `.pwmodel` spelling at all**.

The fallback behaviour itself is correct — per-vertex normals are exactly what an author wants for a
buffer that shipped no normal layer. This ticket is about the diagnostic text, not the behaviour.

## Root cause (guilty source lines)

The engine message is drained verbatim into the op result at
`Plugins/PinWright/Source/PinWrightGeometry/Private/Handlers/Geometry/GeometryOps_Modeling.cpp:641-643`:

```cpp
    FGeometryScriptDebugSink Debug;
    UGeometryScriptLibrary_MeshNormalsFunctions::RecomputeNormals(Mesh, NormalOptions, false, Debug.Get());
    Debug.DrainWarningsInto(Result);
```

`DrainWarningsInto` (`Handlers/Geometry/GeometryScriptDebugSink.h:235-249`) copies
`Message.Message.ToString()` into `Result.Warnings` with no rewriting.

The route to the author does have a translation layer, and it is **by construction unable to touch
this message**. `PwModelCompiler.cpp:2520-2532` routes op warnings to `PWMODEL_STAGE_WARNING` through
`PwModelWarningNames::Translate`, and the comment there states the intent exactly — *"TRANSLATED,
not forwarded … the rewrite happens here, at the one place a warning crosses into the document's
vocabulary."* But `Translate` (`Model/PwModelParser.cpp:1573-1596`) only rewrites the **leading
parameter-name token** of a `Clamp*Warn` message; its header contract
(`Model/PwModelParser.h:276-280`) says so: *"Warnings that do not open with a mapped label come back
byte-identical — every prose warning."* The `RecomputeNormals` text is prose and its offending words
are engine **op** names, not parameter names, so the one layer that exists to prevent exactly this
leak passes it through untouched. The name table (`PwModelParser.cpp:1509+`) has no rows for op
names; a separate RPC-verb-to-op table does exist (`PwModelParser.cpp:1346+`,
`geometry.create_sphere` -> `sphere`), which is the shape a fix could reuse.

**Secondary finding, verified:** the comment at `GeometryScriptDebugSink.h:231-232` asserts *"No
GeometryScript function this module calls emits a WarningMessage on UE 5.8, so this changes no
response shape today."* That is false as of this measurement — `RecomputeNormals` emits one and it
reached a `.pwmodel` author. Correct that comment in the same edit; it is the sentence that would
otherwise persuade the next reader no warning can arrive on this route.

## Verbatim repro

Any `part` whose first op is `append_buffers` without `uvs=` / `normals=`, followed by
`recalculate_normals`. Minimal shape:

```
part p {
    append_buffers positions=[...] triangles=[...]     # no normals=, no uvs=
    recalculate_normals
}
```

Measured 2026-08-27, UE 5.8, this checkout. Reproduced on `SM_Kelp_Blade.pwmodel` and on both fish
models in the same session — three separate documents, same warning.

Then, following the remedy as written:

```
    split_normals            # the real op, and the only one of the two that has a spelling
    compute_split_normals    # -> PWSRC_UNKNOWN_OP
```

## What it should do

Either:

- **Rewrite the remedy in `.pwmodel` terms** at the drain site or in the translation layer: *"this is
  the expected result for buffers with no `normals=`; use `split_normals` if you want hard edges."*
- **Or suppress the sentence** when the mesh genuinely had no normal layer to begin with. In that
  case the fallback is not a mistake and there is nothing for the author to do, so the remedy clause
  is noise as well as wrong.

`model.authoring` states the general rule that a message is prose and `code` is the contract — but
this message does not merely fail to help, it **instructs the reader to write something that cannot
compile**.

## Workaround

Ignore the remedy; the fallback is correct. Cost is one wasted compile per author who trusts the
message.

## Distinct from related tickets

- `B-animation-provenance-misclassified` (IN-REVIEW, Medium) is the **same genus one surface over** —
  a diagnostic whose text misdirects the reader — but on `.pwanim`, and its fault is a *code*
  collision (`PWSRC_BAD_VALUE` used for both ownership refusals and malformed values), not message
  text. **This one is worse in kind than that one:** an ambiguous code leaves the reader unsure which
  of two recoveries to attempt; this message is unambiguous and affirmatively wrong, and the reader
  who acts on it is guaranteed to fail. Different format, different mechanism, no dedup.
- `B-pwmodel-overlap-check-skips-modifiers` is about a diagnostic that cannot fire; this is about one
  that fires correctly and then misinstructs. Orthogonal.

severity rationale: impact=pure friction — the fallback behaviour is correct and nothing is silently wrong, so this is a Low-impact diagnostic-text defect × reach bumped one band: it fires on every `append_buffers` part that does not hand-author normals, which is the normal case for machine-generated geometry (hit on three separate documents in one session), and the rubric explicitly ranks a Low-impact gap on an every-session path above a High-impact gap on a rare one. Its remedy is affirmatively uncompilable rather than merely unhelpful, so the cost is a real compile cycle to `PWSRC_UNKNOWN_OP` -> Medium.

## History
- `#1-initial-repro` `OPEN` reporter — Found building the Atlantis level (map as forcing function; see host `CLAUDE.md` § "What this project is for"), 2026-08-27, UE 5.8, PinWright at this checkout's HEAD. `recalculate_normals` after an `append_buffers` op with no `normals=` returns `PWMODEL_STAGE_WARNING: 'recalculate_normals': RecomputeNormals: TargetMesh did not have normals to recompute; falling back to per-vertex normals. Consider using 'Set Mesh To Per Vertex Normals' or 'Compute Split Normals' instead.` Neither named verb is a `.pwmodel` op: `model.describe_ops` lists neither, `PWSRC_UNKNOWN_OP` is what writing either one gets, the op wrapping the second is `split_normals`, and the first has no spelling in this format at all. Reproduced on `SM_Kelp_Blade.pwmodel` and both fish. Source-confirmed at HEAD: the engine text is drained verbatim at `GeometryOps_Modeling.cpp:641-643` via `DrainWarningsInto` (`GeometryScriptDebugSink.h:235-249`), and the one translation layer on the route to the author — `PwModelCompiler.cpp:2520-2532`, whose comment says warnings are "TRANSLATED, not forwarded … the one place a warning crosses into the document's vocabulary" — cannot reach it: `PwModelWarningNames::Translate` (`PwModelParser.cpp:1573-1596`) rewrites only the LEADING parameter-name token of a clamp message and its contract (`PwModelParser.h:276-280`) states that prose warnings "come back byte-identical". The offending words are engine OP names, and the name table has rows for parameters only; a separate RPC-verb-to-op table already exists at `PwModelParser.cpp:1346+` and is the shape a fix could reuse. Secondary, verified: the comment at `GeometryScriptDebugSink.h:231-232` claims "No GeometryScript function this module calls emits a WarningMessage on UE 5.8, so this changes no response shape today" — falsified by this measurement; correct it in the same edit. Fix: rewrite the remedy in `.pwmodel` terms ("expected for buffers with no `normals=`; use `split_normals` for hard edges"), or suppress the remedy sentence when the mesh had no normal layer at all, which is not a mistake. Filed as `E-` because the fallback behaviour is correct and only the diagnostic text is at fault. Deduped: NO-MATCH board-wide (1316 tickets); nearest is `B-animation-provenance-misclassified`, the same genus (a misdirecting diagnostic) one surface over on `.pwanim` and a code collision rather than message text — and worse in kind here, since an ambiguous code leaves the reader guessing while this one guarantees a failed compile.
