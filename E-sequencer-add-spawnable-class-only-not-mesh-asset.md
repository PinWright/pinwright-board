---
id: E-sequencer-add-spawnable-class-only-not-mesh-asset
title: "sequencer.add_spawnable_from_class docs say 'Class name or asset path' but reject a SkeletalMesh/StaticMesh asset path with CLASS_NOT_FOUND"
status: OPEN
severity: Low
category: ergonomic
tags: [sequencer, add_spawnable_from_class, docs, className, class-vs-asset, discoverability, wiki]
encounters: 1
lastSeen: 2026-07-11T08:14:20+03:00
---

# add_spawnable_from_class className: docs imply any asset path, but a mesh asset path is rejected

The generated wiki page for `sequencer.add_spawnable_from_class` documents its
one required param as:

> `className` (`string`, required): Class name or asset path to spawn

"or asset path" reads as if any content asset path works — including a
SkeletalMesh/StaticMesh asset — but only a `UClass` name or a *class-bearing*
asset path (a Blueprint / generated-class asset) is accepted. Passing a mesh
asset path returns `[CLASS_NOT_FOUND]`. This is doubly confusing because the
exact same mesh asset path is accepted by `actor.spawn {meshPath:...}`, so the
caller reasonably assumes "asset path" means the same thing here.

**Repro:**
`sequencer.add_spawnable_from_class {className:"/Game/Characters/Mannequins/Meshes/SKM_Manny.SKM_Manny"}`
-> `[CLASS_NOT_FOUND] Class not found`.
`sequencer.add_spawnable_from_class {className:"BP_PhysicsControlCharacter"}`
-> success (a class-bearing asset). Same run had already used
`actor.spawn {meshPath:"/Game/Characters/Mannequins/Meshes/SKM_Manny.SKM_Manny"}`
successfully, sharpening the inconsistency.

**Fix direction (docs):** on the sequencer overlay page
`docs/wiki-src/sequencer.md` (the `add_spawnable_from_class` entry), clarify that
`className` accepts a `UClass` name or a **class-bearing** asset (Blueprint /
generated class) only — NOT a raw StaticMesh/SkeletalMesh asset path — and point
callers who have a bare mesh to `actor.spawn {meshPath}` + `sequencer.add_actor`
(or the FK control-rig path) instead. Optionally, `add_spawnable_from_class`
could improve the `[CLASS_NOT_FOUND]` message to say "expected a UClass or
class-bearing (Blueprint) asset; got a <AssetClass> asset" when it resolves the
path to a non-class asset.

severity rationale: impact=docs/discoverability (misleading param description
forces a guess; self-recoverable) x reach=rare (spawnable authoring verb) -> Low

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of a `sequencer.key_controls`
  focus task (block out an FK control-rig pose beat). Trying to bind a skeletal
  mesh into a level sequence, the agent passed the `SKM_Manny` mesh ASSET path to
  `sequencer.add_spawnable_from_class {className:...}` and got `[CLASS_NOT_FOUND]`,
  after that same mesh path had worked with `actor.spawn {meshPath}`. A later call
  with a class-bearing asset (`BP_PhysicsControlCharacter`) succeeded. CallAnalyzer
  flagged it `surprising` on `sequencer.add_spawnable_from_class`; the served wiki
  page reads `className (string, required): Class name or asset path to spawn`,
  which overstates what "asset path" accepts. Page to fix: sequencer overlay
  `docs/wiki-src/sequencer.md`. Companion process finding to the same task's tool
  bugs (`B-sequencer-add-actor-unbound-possessable`,
  `B-sequencer-create-save-no-disk-write`, filed by the per-finding judge); this
  is the distinct docs angle the judge explicitly left out of the add-actor family
  ticket ("NOT in this family: add_spawnable_from_class uses a different mechanism").
