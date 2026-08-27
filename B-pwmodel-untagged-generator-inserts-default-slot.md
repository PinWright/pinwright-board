---
id: B-pwmodel-untagged-generator-inserts-default-slot
title: "An untagged generator inserts an implicit 'Default' slot IN FIRST-USE ORDER, renumbering the author's declared slots, and no diagnostic fires even when a materials block names every slot the author wanted"
status: IN-REVIEW
severity: High
category: bug
tags: [pwmodel, materials, slot, material-id, default-slot, diagnostics, silent-noop, no-diagnostic, slot-renumbering]
encounters: 1
lastSeen: 2026-08-27T14:03:19Z
---

# A `materials { }` block that names two slots can compile to three, with the declared slot 1 pushed to index 2 and nothing said

A `.pwmodel` whose `materials { }` block declares exactly two slots, and every one of
whose generators is meant to be tagged, silently produces a **three**-slot asset when one
sibling generator is left untagged. The implicit `Default` slot is allocated in model-wide
first-use order like any other, so it lands *between* the two declared slots and pushes the
second one from index 1 to index 2.

Referencers bind by **index**, not by name — `model.compile`'s own wiki says so
("referencers address sections by **index** (`OverrideMaterials`, section material
assignments, LOD section settings)") — so every placed instance that binds a material to
"slot 1" then binds it to the phantom slot instead of the slot the author named.

Nothing reports it. `success: true`, every `health` field clean, `diagnosticSummary.total`
accounts for zero material diagnostics, and the only place the fact appears is
`materialSlotList`, which the caller has to read and diff against their own source by eye.

## Differential repro (replayed live via `mcp__pinwright__call`, `diagnosticLimit: 0`)

Two `model.validate` calls. The documents differ by **one `material="Stone"`** on the
second `box`, and produce **byte-identical geometry**: 36 triangles, 24 vertices,
`signedVolume` 3,000,000, identical `bounds` in both.

```
pwmodel 0

materials {
    Stone      = "/Game/Atlantis/Materials/MI_Stone_Temple"
    StoneAlgae = "/Game/Atlantis/Materials/MI_Stone_Algae"
}

part a {
    box size=(100, 100, 100) at=(0, 0, 50) material="Stone"
    box size=(100, 100, 100) at=(300, 0, 50)          # <- untagged
}

part b {
    box size=(100, 100, 100) at=(600, 0, 50) material="StoneAlgae"
}
```

| | untagged sibling | `material="Stone"` added |
|---|---|---|
| `materialSlots` | **3** | 2 |
| index 0 | `Stone` -> `MI_Stone_Temple` | `Stone` -> `MI_Stone_Temple` |
| index 1 | **`Default` -> `""`** | `StoneAlgae` -> `MI_Stone_Algae` |
| index 2 | `StoneAlgae` -> `MI_Stone_Algae` | — |
| `diagnosticSummary.total` | **1** | 1 |
| material diagnostics | **0** | 0 |

The single diagnostic in both runs is `PWMODEL_FLOATING_COMPONENT`, raised by the three
separated probe boxes and unrelated to materials. With `diagnosticLimit: 0` every entry is
printed, so nothing was folded or suppressed.

## Why no diagnostic fires (verified in source, not inferred)

`ValidateMaterialSlots` in `PwModelParser.cpp` walks part-level ops and, on an untagged
generator, sets a bool instead of adding the slot to the set it later checks —
`PwModelParser.cpp:2817-2860`:

```cpp
    TMap<FString, FSlotReference> ReferencedSlots;
    ...
    bool bUsesImplicitDefaultSlot = false;
```

with the reasoning at `:2820-2826`:

> But it is never TAGGED by a `material=` the author wrote, so feeding it into
> `ReferencedSlots` would fire `PWMODEL_UNBOUND_MATERIAL` on every ordinary document that
> simply has no materials block, which is the normal case and not a mistake.

That reasoning is correct for the case it names. The gap is that the bool is then consulted
in only **one** of the two loops that follow — the `PWMODEL_UNUSED_MATERIAL` loop at
`:2892-2896`, to stop a deliberate `materials { Default = "…" }` binding being reported as
dropped. The `PWMODEL_UNBOUND_MATERIAL` loop at `:2873-2889` iterates `ReferencedSlots`
only, so the implicit `Default` — which really is referenced, really is unbound, and really
does occupy an index — can never reach it.

The **placement** of the slot is deliberate and defended in source, and this ticket is not
asking for it to change. `PwModelCompiler.cpp:50-54`:

```cpp
// Allocated lazily, so a model whose geometry is fully tagged never grows one; it is
// material ID 0 whenever the first geometry in the model is untagged, which is the
// ordinary case. Forcing it to 0 in the other case would renumber every explicitly named
// slot after it, which is worse than a Default that sits wherever it was first needed.
```

Allocation happens through the ordinary first-use path, `PwModelCompiler.cpp:1368-1388`,
where an op with no `material=` falls back to `DefaultSlotName` and goes through the same
`ResolveSlot` (`:734`) that appends on first use.

## What it should do

Fire a diagnostic for the implicit `Default` slot **only when the document has a non-empty
`materials` block that does not bind `Default`**. That is exactly the population the current
suppression over-serves: an author who wrote a `materials` block has stated the slot table
they intend, so a phantom slot inserted into the middle of it is a mistake by definition,
while a document with no `materials` block at all stays silent as it does today. The bool is
already computed and `Doc.Materials` is already in hand at `:2891`, so the condition is
available where the check would live.

Worth naming the consequence in the message rather than just the fact — "slot 'Default' was
created for untagged geometry and occupies index N; slots after it are renumbered" — because
the index shift, not the extra slot, is what breaks the caller.

## Measured cost on a real build

Found on `Content/Atlantis/Meshes/SM_Arch_Ruin.pwmodel` (Atlantis level, ruined-architecture
pass). Three parts had their first generator tagged and their siblings not: six arch
voussoirs, two break-stub stones and one silt-band box. Result was `materialSlots: 3` with
`StoneAlgae` at index 2. Three placement agents were queued to bind `MI_Stone_Algae` to
**slot 1** via `actor.spawn`'s `materialPaths`, which would have put the algae instance on
the arch ring and left the silt band on plain stone — inverted, on every placed copy, with
`success: true` and a fully green `health` block. Caught only by reading `materialSlotList`
against the source by eye.

## Distinct from `B-pwmodel-modifier-output-takes-slot-zero`

That ticket (OPEN, High) is about **modifier** output — `sweep`, `extrude_along_spline` and
friends — keeping `MaterialID == 0` because `DispatchModifier` never calls `ResolveSlot`, so
the geometry lands on whatever slot the first part happened to tag, and the author cannot
correct it because `MakeModifier` omits `AcceptMaterialSlot`.

| | that ticket | this |
|---|---|---|
| Op kind | modifiers (`RunOp` -> `DispatchModifier`, `PwModelCompiler.cpp:2066`) | **generators** (`RunGenerator`, `:1368-1388`) |
| Slot table | unchanged; no slot is allocated | **a slot is allocated**, and the count changes |
| Symptom | geometry takes another part's material | a phantom slot **renumbers** the declared ones |
| Author's remedy | none — `material=` is rejected on those ops | `material=` on every generator; entirely available |
| Missing diagnostic | untagged modifier output | unbound implicit `Default` alongside an explicit `materials` block |

The two share a root theme — the slot table is first-use-ordered and nothing narrates
changes to it — and a single new "geometry landed in a slot your `materials` block never
named" diagnostic would cover both. They are separate because the remedies differ: this one
is a parser-side condition on an existing check, that one needs `ResolveSlot` wiring plus a
new parser parameter.

severity rationale: impact=silent-wrong-material-index on every placed instance, with success/health/diagnostics all clean and only a hand-diff of `materialSlotList` able to catch it x reach=any multi-slot document that leaves one generator untagged, which is the ordinary shape of a multi-primitive part -> High

## Not verified

`model.compile`'s `unboundSlots` field is documented as naming slots that fall back to the
default material, and would plausibly list `Default` here. It was **not** exercised for this
case: `unboundSlots` appears only when an asset is written, and probing it would have meant
writing a scratch asset into a `/Game` folder shared live with other agents. If it does fire,
this ticket narrows to validate-only — the pre-asset surface the authoring loop actually runs
on, where the whole point is to catch a mistake before writing.

## History
- `#1-initial-repro` `OPEN` reporter — Found authoring `SM_Arch_Ruin.pwmodel` for the Atlantis level. A document declaring `materials { Stone, StoneAlgae }` compiled to `materialSlots: 3` with an implicit `Default` at index 1 and `StoneAlgae` displaced to index 2, because three parts tagged only their FIRST sibling generator. Zero material diagnostics on `model.validate` at `diagnosticLimit: 0`; `success: true`, `health` fully green. Reduced to a 3-box / 2-part differential repro (above) whose two variants differ by one `material="Stone"` and produce byte-identical geometry (36 tris, 24 verts, signedVolume 3,000,000) while differing in slot count 3 vs 2. Mechanism confirmed in source: `ValidateMaterialSlots` records untagged generators in `bUsesImplicitDefaultSlot` (`PwModelParser.cpp:2827`) rather than in `ReferencedSlots`, and consults that bool only in the `PWMODEL_UNUSED_MATERIAL` loop (`:2892-2896`), never in the `PWMODEL_UNBOUND_MATERIAL` loop (`:2873-2889`) — so the implicit slot is structurally unreachable by the unbound check. Slot PLACEMENT is deliberate and defended at `PwModelCompiler.cpp:50-54` and is explicitly not what this asks to change; the fix is to fire the unbound diagnostic for `Default` when `Doc.Materials` is non-empty and does not bind it, leaving documents with no `materials` block silent as today. Real cost: three placement agents were about to bind `MI_Stone_Algae` to slot 1, which would have inverted the stone and algae materials on every placed copy of the mesh. Classified TOOL BUG (silent wrong output on a normal authoring path).
- `#2-implicit-default-slot-diagnostic` `IN-REVIEW` developer — Added `PWMODEL_IMPLICIT_DEFAULT_SLOT` (`PwModelDiagnostic.h`) and emitted it from the COMPILER rather than the parser: `FCompiler::WarnOnImplicitDefaultSlot` runs once after `MergeParts`, when the slot table is final, so the message can name the index `Default` took and each declared slot that moved with its new index — facts the parser cannot know. `RunGenerator` records the FIRST untagged part-level generator (op name, line, column, part) at the allocation site. Fires only when the document has a `materials` block that does not bind `Default` AND at least one declared slot reached the table; documents with no block, documents binding `Default`, and documents whose geometry is untagged throughout (already covered per-binding by `PWMODEL_UNUSED_MATERIAL`) stay silent. Slot PLACEMENT deliberately unchanged, per this ticket: appending `Default` last would renumber every document that already compiled with it in the middle — this defect inflicted on assets that are already placed. Files: `Source/PinWrightGeometry/Private/Model/PwModelCompiler.cpp`, `.../PwModelDiagnostic.h`, `.../PwModelAst.h` (corrected a comment claiming declaration order is the asset's slot order), `docs/pwmodel-format.md` (diagnostics-table row required by `core.pwmodel_diagnostics.DocumentedCodesMatchEmittedCodes`; corrected the "Slot `Default` (ID 0)" claim in the Materials section). `PwModelParser.cpp` untouched. Tests: `PinWright.Model.MaterialSlots.UntaggedGeneratorRenumberingDeclaredSlotsIsReported` and `PinWright.Model.MaterialSlots.OrdinaryUntaggedDocumentsAreNotReported` in the new `Source/PinWrightGeometry/Private/Tests/Model/TestPwModelMaterialSlotOrder.cpp` — the ticket's own differential repro, asserting the slot table is Stone/Default/Algae, that exactly one warning fires anchored on the untagged generator's line naming index 1 and 'Algae' at index 2, and that the one-`material=`-later variant is two slots and silent.
