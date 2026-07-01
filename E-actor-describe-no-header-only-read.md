---
id: E-actor-describe-no-header-only-read
title: "actor.describe has no field-projection mode — a single-field confirm read (label / transform / attachParent / DataLayerAssets) returns the full component tree, overflows 10k, and spills to file"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [actor, actor-describe, response-size, fields-select, spill]
---

# actor.describe has no field-projection / header-only read mode

**Fix:** Add an optional `fields` allow-list to `actor.describe` — `fields:["label","transform"]`,
`fields:["properties"]`, `fields:["components"]` — that projects the top-level
describe object down to the requested keys (mirroring `property.list`'s
`propertyNames` allow-list, the shared vocabulary `E-component-read-filter` already
cites). The valid top-level keys are `name`/`label`/`path`/`class`/`level`/`folder`/
`guid`/`tags`/`transform`/`properties`/`components`. This covers the evidence intents
below: transform/label (#1/#3) project to `fields:["label","transform"]`; the
`DataLayerAssets` membership (#2) lives in the sparse modified-property block, read
via `fields:["properties"]`; the `attachParent` (#4) and smart-link config are
*per-component* data nested inside `components`, read via `fields:["components"]`
paired with the existing `nameMatch`/`componentClass` component filter to narrow to
the one component — in every case dropping the bulky unwanted blocks that caused the
spill. Also add the complementary
`includeComponents:false` (alias `componentsMode:"none"`) boolean for the common
"identity + transform, drop the whole component array" read. Optional + additive,
default output byte-identical, no aspect-version bump — the same constraint
`E-component-read-filter` honored. Docs: steer plain spawn-confirm (label + location)
reads to the lighter `actor.get` (+ `actor.get_transform` for rotation) and warn that
unprojected `actor.describe` spills on multi-component actors (`docs/wiki-src/actor.md`).

The single most common actor read intent — *"confirm the actor landed and read
back its world transform and the label the editor actually assigned"* — has no
right-sized verb. `actor.describe` is the natural method for "read this actor's
transform + label," but it always returns the full compact actor-description
shape: identity, world transform, sparse modified actor properties, **every
component**, scene attachment data, per-component relative transforms, and
sparse modified component properties (`actor.md` overlay, `### actor.describe`).
On a single, ordinary actor that carries even a handful of default components,
that payload exceeds the 10000-character display threshold, so the working
large-output guardrail (`E-http-response-spill` DONE;
`B-get-components-renders-empty-to-caller` IN-REVIEW) spills it to
`Saved/EditorAutomation/HttpResponses/...` and the caller has to `Read` the
file off disk to extract a single location vector and a label string.

The two sibling reads don't fill the gap for this intent:

- `actor.list` narrows by label/name substring but returns only label/name
  (and itself spills on big worlds — `B-get-components-renders-empty-to-caller`
  `#5` measured an `actor.list` response at 18310 chars). It confirms presence
  but not the world transform.
- `actor.get_components` is the lean *component* read (the wiki even steers
  transform-only callers here), but it returns **component** relative transforms,
  not the **actor's** world transform + label — the wrong axis for "did the
  spawn land at [0,0,200] and what label did it get."

So the "read back transform/label" intent has no path that fits in 10k: one verb
returns too little (`actor.list` = label only), the other returns too much
(`actor.describe` = full component tree → spill).

This is **distinct from `E-component-read-filter`** (DONE), which added
component-scoping filters (`nameMatch` / `componentClass`) to `actor.describe`,
`actor.get_components`, and `blueprint.scs.get`. Those filters narrow *which
components* are dumped — they cannot express "I want zero components, just the
actor header." A caller who wants the actor's own transform has no
`nameMatch`/`componentClass` value that drops the component array entirely while
keeping identity + world transform.

## What it should do

All options are optional + additive, default output byte-identical, no
aspect-version bump — the same constraint `E-component-read-filter` honored:

1. **A `fields` allow-list** (e.g. `fields:["label","transform"]`,
   `fields:["attachParent"]`, `fields:["properties"]`) on `actor.describe`,
   projecting the top-level describe object down to the requested keys —
   mirroring `property.list`'s `propertyNames` allow-list that
   `E-component-read-filter` already cites as the shared vocabulary. *(Recommended
   — this is the general fix: all four evidence intents below wanted exactly one
   top-level key, including the single-actor-property cases (`DataLayerAssets`,
   `attachParent`) that an `includeComponents:false` boolean would NOT isolate
   since the sparse modified-actor-property block is always emitted and can itself
   spill.)*
2. **`includeComponents: false` (or `componentsMode: "none"`) on
   `actor.describe`** — return identity, transform, and sparse modified actor
   properties, omit the component array entirely. A convenient shorthand for the
   common "identity + transform, no component tree" confirm read (equivalent to
   `fields` minus `components`, but one boolean and self-documenting).
3. **Docs steer (do regardless)** — the `actor.md` overlay's `### actor.describe`
   section should tell callers that a plain spawn-confirm (label + location) is
   cheaper via the lighter `actor.get` (which returns location + scale; pair with
   `actor.get_transform` for full rotation — `actor.get` does NOT return the full
   world transform) and that an unprojected `actor.describe` will spill on
   multi-component actors. (Tag `docs`, edit `docs/wiki-src/actor.md`.)

## Evidence

From the `effect.activate_niagara` struggle audit (namespace `effect`, outcome
`clean`). Story step 2, verbatim: *"Confirm the actor actually landed in the
world and read back its transform/label so I know the spawn took (use actor.list
or a similar actor read so we have the real label the editor assigned)."* The
agent ran `actor.list {filter:"PreviewNiagara_"}` (count=1 — presence confirmed,
but no transform) then escalated to `actor.describe {PreviewNiagara_Island}` for
the world transform. Friction note, verbatim: *"actor.describe overflowed the
10k display limit and spilled to a HttpResponses file, so I had to Read that file
for the transform."* The target was a single spawned `NS_SpawnFromIslandNDC`
Niagara actor (a NiagaraComponent + default scene/billboard components) at
`[0,0,200]` — not a component-heavy actor, yet a trivial 2-field read
(world location + assigned label) still tripped the spill path and cost a manual
off-disk `Read`. Net: 2 actor-read calls + 1 file Read to answer "did the spawn
land and what's its label."

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the effect.activate_niagara struggle audit (outcome clean). `actor.describe` is the natural "read back transform/label" verb but always returns the full component tree, overflowing 10k and spilling to HttpResponses even for a single ordinary Niagara actor; the agent had to `Read` the spill file for one location vector. Siblings don't fit: `actor.list` returns label-only (and itself spills on big worlds), `actor.get_components` returns component (not actor) transforms. Distinct from `E-component-read-filter` (DONE) — that scopes *which components* are dumped, not "zero components, header only." Proposed: an `includeComponents:false`/`componentsMode:"none"` (or a `fields` allow-list) on `actor.describe`, optional + additive like the existing filter params; docs fallback in `docs/wiki-src/actor.md` pointing spawn-confirm reads at `actor.get`.
- `#2-evidence-datalayer-readback` `OPEN` reporter — Same spill, different read intent, from the world_partition struggle audit (outcome tool_bug for an unrelated finding; this is the PROCESS angle). After assigning 3 cube StaticMeshActors to data layers, the agent verified membership with `actor.describe` on each — `actor.describe {A}`, `{C}`, `{B}` — to read back the single `DataLayerAssets` sparse modified actor property. Friction note, verbatim: *"actor.describe payloads exceeded the 10000-char display threshold and spilled to JSON files on disk, so I had to Read/Grep those files to confirm DataLayerAssets membership rather than seeing it inline."* Net: 3 describe calls, each spilled, each forcing an off-disk Read/Grep to confirm one property. This data point favors proposed **option 2 (`fields` allow-list)** over option 1 (`includeComponents:false`): the caller wanted exactly one *actor property* (`DataLayerAssets`), not the transform/label header — dropping the component array alone (option 1) would still dump every modified actor property and could still spill, and wouldn't isolate the one field; `fields:["DataLayerAssets"]` would. Reinforces that the missing right-sized read is per-field, not just per-component. Confirms the spill-on-verify friction recurs across distinct namespaces (effect → world_partition) and verify intents (transform/label → datalayer membership).
- `#3-evidence-navlinkproxy-readback` `OPEN` reporter — Third independent namespace (navigation), from the `navigation.set_nav_link_type` struggle audit (outcome tool_bug for an unrelated PointLinks finding `B-nav-link-proxy-appends-default-link`; this is the PROCESS angle). Story step 6 was "read the actor back to confirm it exists and inspect its properties so I can see the smart-link settings took (actor.get … actor.describe / actor.get_components if useful)." The agent ran `actor.get {LedgeJumpLink}` and `actor.get_components {LedgeJumpLink}` first (those fit inline), then escalated to `actor.describe {LedgeJumpLink}` to see the smart-link component config — and that one spilled. Friction note, verbatim: *"actor.describe output exceeded the 10k display limit and spilled to a file (had to Read it)."* The target was a single NavLinkProxy actor (its smart-link `UNavLinkCustomComponent` plus default scene/billboard components) — again not a component-heavy actor, yet the full-tree describe still tripped the spill path and cost a manual off-disk Read. Note this task confirms `actor.get` and `actor.get_components` *do* fit inline here, so the gap is specifically `actor.describe`'s all-or-nothing full-tree shape; a caller who only needs to confirm one component's instanced properties (here the SmartLinkComp settings) still pays the spill. Reinforces both proposed options (an `includeComponents`/`componentsMode` knob would not have helped this case since the wanted data *was* a component, but a component-scoping `nameMatch:"SmartLink"` on `actor.describe` — already shipped via `E-component-read-filter` — likely would have; worth a docs note steering smart-link-config readback to a filtered `actor.describe` rather than the unfiltered call). Net across the three audits: spill-on-verify recurs across effect → world_partition → navigation and across transform/label, datalayer-membership, and component-config readback intents.
- `#4-evidence-attachparent-readback` `OPEN` reporter — Fourth independent recurrence, namespace `actor` (the patrol-route block-out struggle audit, outcome clean — every call `ok`/non-error, all 28 round-tripped). Story step: attach the first waypoint to the anchor with `actor.attach`, then confirm the attachment took. The agent ran `actor.attach {A1 child of Anchor}` then `actor.describe {Waypoint_A1}` purely to read back the **single** `attachParent` field — and that one describe spilled. Friction note, verbatim: *"the only minor wrinkle was actor.describe exceeding the 10k display threshold and being written to a file, which I grepped for the attachParent field as intended."* The target was a single cube `StaticMeshActor` (the spawned-cube-mesh marker) with only its default StaticMeshComponent + scene/billboard components — again not a component-heavy actor, yet the full-tree describe still tripped the spill path and cost a manual off-disk Grep for one string field. This is the strongest data point yet for proposed **option 2 (`fields` allow-list)** over option 1: the caller wanted exactly one field (`attachParent`, a scene-attachment datum), so neither `includeComponents:false` (it's not a component-array problem) nor the shipped `nameMatch`/`componentClass` filters (it's not a *which-components* problem) would isolate it — `fields:["attachParent"]` (or `fields:["attachParent","worldTransform"]`) would. Net across four audits: spill-on-verify recurs effect → world_partition → navigation → actor and across transform/label, datalayer-membership, component-config, and attach-parent readback intents — the through-line is that `actor.describe` is the only verb exposing each of these data (attachParent, DataLayerAssets, smart-link config) yet has no way to return just the wanted field, so the most trivial confirm-the-edit-took read keeps tripping the spill and paying a Read/Grep tax.
- `#5-reword-and-fix` `IN-REVIEW` developer — Reworded to center the **`fields` allow-list** as the primary fix (the original frontmatter led with `includeComponents:false`, but the adversarial review correctly flagged that option as the weaker, partly-`actor.get`-duplicating choice; three of four evidence entries — `DataLayerAssets`, `attachParent`, smart-link config — want a single top-level key that `includeComponents:false` would NOT isolate). Also corrected the body: `B-get-components-renders-empty-to-caller` is IN-REVIEW not DONE (only `E-http-response-spill` is the DONE spill), and `actor.get` returns location+scale (no rotation) so the docs steer now pairs it with `actor.get_transform`. **Implemented** both params on `actor.describe`: (1) `fields` (alias `field`) — case-insensitive top-level-key allow-list projecting the describe object to the requested keys (`name`/`label`/`path`/`class`/`level`/`folder`/`guid`/`tags`/`transform`/`properties`/`components`; `schema`+`storage` always retained as envelope); (2) `includeComponents:false` (alias `componentsMode:"none"`/`include_components`) dropping the component array. Both optional + additive; with neither param the output is byte-identical (no aspect-version bump). Files: `Source/.../Private/Utils/ActorDescribeBuilder.h` + `ActorDescribeBuilder.cpp` (new `FActorDescribeOptions` struct + projection in `BuildActorJson`), `Source/.../Private/Handlers/Actor/DescribeHandler.cpp` (parse `fields`/`includeComponents`, register params), and `Docs/wiki-src/actor.md` (`### actor.describe` spill warning + `fields`/`includeComponents` doc + `actor.get`/`actor.get_transform` steer). Test: extended `Source/.../Private/Tests/World/TestActorDescribeBuilder.cpp` with `FActorDescribeBuilderFieldProjectionTest` exercising `BuildActorJson` with a `fields` allow-list (projects to exactly `{schema,storage,label,transform}`, drops `components`/`properties`/`guid`) and `includeComponents:false` (keeps the header, drops only `components`) — fails if the projection is reverted.
