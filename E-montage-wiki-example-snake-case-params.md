---
id: E-montage-wiki-example-snake-case-params
title: "animation.authoring montage wiki example uses snake_case params (asset_path/slot_name/start_time/blend_time) while the real schema is camelCase — copy-paste teaches the wrong spelling"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [animation, anim-montage, docs, wiki, snake-case, camelcase, param-naming]
---

# create_montage wiki example uses snake_case; the schema is camelCase

The montage build-out example on the `animation.authoring` wiki overlay shows
every parameter in **snake_case**, but the actual handler schemas use
**camelCase**. An agent who copy-pastes the documented sequence learns the wrong
param spelling for the whole montage chain.

## Evidence (source-confirmed)

`docs/wiki-src/animation.authoring.md` (the `create_montage` section, ~lines
41-46) documents:

```
call("animation.authoring.add_montage_slot",   { asset_path: "...", slot_name: "DefaultSlot" })
call("animation.authoring.add_montage_section", { asset_path: "...", section_name: "Default", time: 0.0 })
call("animation.authoring.set_section_timing",  { asset_path: "...", section_name: "Default", start_time: 0.0, end_time: 1.5 })
call("animation.authoring.set_blend_in",        { asset_path: "...", blend_time: 0.25 })
call("animation.authoring.set_blend_out",       { asset_path: "...", blend_time: 0.25 })
```

The real `RPC_PARAM_*` specs in `AnimationAuthoringHandler_Sequence.cpp` are all
camelCase:
- `add_montage_slot` (:1199): `assetPath`, `animationPath`, `slotName`, `startTime`, `save` — **not** `asset_path`/`slot_name`.
- `add_montage_section` (:1157): `assetPath`, `sectionName`, `startTime`, `save` — note `startTime`, **not** the documented `time`.
- `set_section_timing` (:1267): `assetPath`, `sectionName`, `startTime`, `save` — there is **no `end_time`/`endTime`** param at all; the handler only sets a section's start time, so the documented `end_time: 1.5` is doubly wrong (wrong case *and* nonexistent param).

So the documented example mixes three problems: snake_case spellings the schema
rejects, a `time` param that should be `startTime`, and an `end_time` param that
does not exist on `set_section_timing`.

## Friction (from the audited task)

Struggle audit of the `MTG_HitReact` hit-reaction montage task (namespace
animation.authoring, outcome tool_bug — the two correctness bugs filed as
`B-add-montage-notify-time-dropped` (judge) and
`B-montage-slot-no-sequence-length-recalc` (this audit)). Friction note
(verbatim): *"Minor: create_montage wiki example uses snake_case params while the
real schema is camelCase."* The agent did not get stuck on it (the runtime
`Ctx.Get*` reads tolerate the case via the camelCase/snake_case alias rule for
the params that exist), but the docs example actively teaches the wrong spelling
and a phantom `end_time` param, which is a discoverability trap for the next
caller.

The montage chain currently parses these because the dispatcher's
camelCase↔snake_case alias rule papers over the *casing* for params that exist —
but `end_time` has no counterpart at all, so a literal copy-paste of the example
would silently drop the section end time (or, post any param-strictness change,
hard-fail). The wiki should show the canonical camelCase spelling and only
params that actually exist.

## What it should do

Downstream wiki edit (not this audit's job) on the overlay page
`docs/wiki-src/animation.authoring.md`, `create_montage` section:
- Rewrite the example to camelCase: `assetPath`, `slotName`, `animationPath`,
  `sectionName`, `startTime`, `blendTime` (verify each against the actual
  `RPC_PARAM_*` names), matching the rest of the page (the `create_composite` /
  `add_composite_segment` examples a few lines below already use camelCase
  `assetPath`/`animationPath`, so the montage block is the odd one out).
- Drop the nonexistent `end_time` from the `set_section_timing` example, or
  replace it with the real `startTime`-only contract (a section's end is the
  next section's start, so document that explicitly).
- Rename the `add_montage_section` example's `time:` to `startTime:`.

Pairs with `B-montage-slot-no-sequence-length-recalc`: that example sequence also
implies a slot add yields a usable montage, while in fact the length stays 0
until the length-recalc bug is fixed — once both land, the wiki example should
show a build that actually produces a non-zero-length, correctly-spelled montage.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `MTG_HitReact`
  montage authoring task (outcome tool_bug). Confirmed against source:
  `docs/wiki-src/animation.authoring.md` create_montage example (~:41-46) shows
  `asset_path`/`slot_name`/`section_name`/`start_time`/`end_time`/`blend_time`/`time`
  (snake_case), while the handler schemas in
  `AnimationAuthoringHandler_Sequence.cpp` are camelCase
  (`assetPath`/`slotName`/`sectionName`/`startTime`/`save`); `set_section_timing`
  (:1267) has no `end_time`/`endTime` param at all, and `add_montage_section`
  (:1157) uses `startTime` not the documented `time`. Sibling camelCase
  `create_composite`/`add_composite_segment` examples on the same page make the
  montage block the inconsistent one. Friction note (verbatim): "Minor:
  create_montage wiki example uses snake_case params while the real schema is
  camelCase." Tagged docs; downstream wiki overlay edit names the page above.
- `#2-second-task-repro` `OPEN` reporter — Independent reproduction in a second
  montage task (`AM_DinoDragon_IdleToWalk`, namespace animation, focus
  `animation.authoring.link_sections`). Friction note (verbatim): *"a minor doc
  inconsistency: create_montage's wiki example uses snake_case (asset_path/slot_name)
  while the actual param schema and all sibling montage methods use camelCase
  (assetPath/slotName)."* Same overlay page (`docs/wiki-src/animation.authoring.md`,
  create_montage section), same casing trap, confirmed on a different skeleton
  (SK_DinoDragon_Skeleton) and a different agent run — the example continues to
  teach the wrong spelling to copy-pasters. Aggregated here rather than re-filed.
- `#3-additional-runtime-replay` `OPEN` reporter — Third independent task repro
  (`AM_DinoDragon_RoarBite`, namespace animation.authoring, seed
  `animation.authoring.create_montage`, same SK_DinoDragon_Skeleton). First entry
  with a **live runtime-error replay** of the documented example shape (prior
  entries #1/#2 were source-confirmed only). Replayed the wiki Notes block
  verbatim: `create_montage {asset_path, skeletonPath, slot_name}` (the
  documented snake_case keys, no `name`) →
  `[MISSING_REQUIRED_PARAM] Missing required parameter 'name' (type: string)`
  (the example omits `name` entirely and uses `asset_path`, neither of which the
  handler accepts). And `set_section_timing {assetPath, sectionName, start_time:0,
  end_time:1.5}` (the documented `start_time`/`end_time` form) →
  `[UNKNOWN_PARAMS] Unknown parameter(s) for 'animation.authoring.set_section_timing':
  [start_time, end_time]. Valid parameters: [assetPath, sectionName, startTime, save].`
  — confirming both the snake_case rejection and that `end_time` is a phantom
  param the handler hard-rejects. So a literal copy-paste of the create_montage
  Notes example does NOT silently parse via the alias rule (as #1 hypothesized
  for the casing) for `set_section_timing` — it hard-fails `[UNKNOWN_PARAMS]`,
  and the create_montage call itself hard-fails for the missing `name`. The
  attempt agent succeeded only by ignoring the documented example and using the
  authoritative camelCase Parameters list instead. No new ticket; aggregated here.
- `#4-additional-notes-self-contradiction` `OPEN` reporter — Realism audit of an
  Echo sword-combo montage task (namespace animation.authoring, realism mode / no
  seed; culprit `animation.authoring.create_montage`). Additional evidence in the
  SAME `create_montage` wiki Notes block: beyond the snake_case params, the Notes
  paragraph **directly contradicts the page's own summary and the real behavior**.
  The page summary + Parameters say the montage is created *"with one slot and an
  empty 'Default' section"*, but the Notes say (verbatim): *"The created montage
  starts empty — no slots, no sections, zero length."* Replay-confirmed live via
  `mcp__editor-automation__call`: `create_montage {name:"M_Echo_OracleReplay",
  path:"/Game/Characters/Echo/Animations",
  skeletonPath:"/Game/Characters/Echo/Meshes/Echo_Skeleton"}` → `success:true`,
  then `get_animation_info {assetPath:".../M_Echo_OracleReplay"}` →
  `{"numSections":1,"numSlots":1,...,"sections":[{"sectionName":"Default",...}],
  "slots":[{"slotName":"DefaultSlot",...}]}` — i.e. one slot + one 'Default'
  section, NOT empty. So the Notes "starts empty — no slots, no sections, zero
  length" is flatly false (the summary is the correct one). This misleads a caller
  into thinking they must `add_montage_slot`/`add_montage_section "Default"` to
  bootstrap, when both already exist (compounding the snake_case trap in the same
  block, and overlapping the documented-vs-duplicate slot behavior tracked in
  `B-create-montage-duplicate-default-slot`). Same overlay page / same Notes block;
  the downstream wiki fix should also delete/correct the "starts empty" sentence so
  it matches the summary and reality. Aggregated here rather than re-filed.
- `#5-retriage` `OPEN` triage — Low→Medium: wiki example teaches snake_case params + a phantom end_time that hard-reject on copy-paste, niche montage path.
- `#6-fix` `IN-REVIEW` developer — All three validity lenses (correctness/adversarial/historian) confirmed against current source and voted valid; defect is present, not already-fixed. Verified the canonical camelCase specs in `Source/PinWright/Private/Handlers/Animation/AnimationAuthoringHandler_Sequence.cpp`: `create_montage` (:1091, requires `name`+`skeletonPath`, takes `path`/`slotName`/`save`; summary at :1092 says "with one slot and an empty 'Default' section"), `add_montage_slot` (:1207 → `assetPath`/`animationPath`/`slotName`/`startTime`/`save`), `add_montage_section` (:1165 → `assetPath`/`sectionName`/`startTime`/`save`), `set_section_timing` (:1284 → `assetPath`/`sectionName`/`startTime`/`save`; body only reads `startTime` at :1319-1323, no `endTime` param exists), `set_blend_in`/`set_blend_out` (:1405/:1454 → `assetPath`/`blendTime`/`blendOption`/`save`). Ripgrep of the handler file for `asset_path|slot_name|section_name|start_time|end_time|blend_time` returns ZERO hits — no snake_case aliases declared, so every snake_case key in the old example hard-rejects (the dispatcher's strict UNKNOWN_PARAMS gate). Fix (docs-only overlay edit, no code change): rewrote the `### animation.authoring.create_montage` "Typical sequence" example in `Docs/wiki-src/animation.authoring.md` to canonical camelCase (`assetPath`/`animationPath`/`slotName`/`sectionName`/`startTime`/`blendTime`), renamed `time:`→`startTime:`, dropped the phantom `end_time:1.5` (added prose noting a section runs until the next section's start, so set only `startTime`), and corrected the contradictory Notes sentence "starts empty — no slots, no sections, zero length" to match the summary/handler reality (one slot + empty `Default` section). Regression test: added `PinWright.infra.wiki_handler.MethodPage.CreateMontageExampleParams` (`FWikiHandlerCreateMontageExampleParamsTest`) in `Source/PinWright/Private/Tests/Infra/TestWikiHandler.cpp` — drives the real `WikiHandler::RenderPage("animation.authoring.create_montage")` overlay render path and asserts the example uses `assetPath`/`animationPath`/`sectionName`/`startTime`/`blendTime` and does NOT use the snake_case keys (`asset_path:`/`slot_name:`/`section_name:`/`start_time:`/`blend_time:`) or the phantom `end_time:`/`endTime:` (case-sensitive `<key>:` matches, mirroring the `CreateMetasoundExampleParams`/`AddMappingExampleParam` precedent for this bug class). It fails if the example is reverted to snake_case. Did not compile/run (later phase verifies).
