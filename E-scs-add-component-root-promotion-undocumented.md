---
id: E-scs-add-component-root-promotion-undocumented
title: "scs.add_component 'attach to root' doc doesn't warn that UE promotes the first scene component to RootComponent and reparents later root-level components under it"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [scs, blueprint, add_component, reparent, docs, discovery, root-validation]
---

# `blueprint.scs.add_component` hides UE's scene-root promotion / auto-reparent rule

`blueprint.scs.add_component`'s `parentComponentName` doc reads: *"omit (or
empty) to attach as a child of the root."* For a Blueprint with **no
native/inherited root** (a from-scratch `Actor` parent), that framing is
misleading: there is no pre-existing root to be "a child of." UE's
`USimpleConstructionScript::ValidateSceneRootNodes` instead makes the **first**
scene component you add the actor's `RootComponent`, and every *subsequent*
root-level scene component is silently re-parented **under that first one**. So
"add BeaconBase → root, then DetectionZone → root" does not produce two
siblings under a root — it produces `DetectionZone` nested under `BeaconBase`.

Nothing in the discovery surface tells an agent this will happen:
- `blueprint.scs.add_component` method page: only "omit … to attach as a child
  of the root" — no mention of root promotion or reparenting.
- `blueprint.scs` overlay (`docs/wiki-src/blueprint.scs.md`): covers
  template-vs-instance and inherited-override rules, but says nothing about the
  no-native-root case.
- `blueprint.scs.reparent_component` page: `newParentName` "empty for root" —
  same ambiguous "root" framing, no note that emptying it on a scene component
  may be a no-op because UE re-promotes the existing root.

This is a **process** gap distinct from the readback bug
`B-scs-get-local-child-parent-link-missing` (which is about `scs.get` not
emitting a `parent` field). Even if that readback bug is fixed so the tree
*reads back* correctly, an agent who asked for "DetectionZone attached to the
root" and sees it nested under BeaconBase still has no doc explaining that this
is intended, expected UE behavior rather than a tool error. The two findings
are complementary: one fixes what the readback *says*, this one fixes what the
docs *promised* the write would do.

**What it should do:** Add a short note to the `blueprint.scs` overlay (and
ideally an `### blueprint.scs.add_component` H3 / the `parentComponentName`
description) stating: with no inherited/native root, the first scene component
added becomes the actor `RootComponent`; additional scene components passed
with an empty `parentComponentName` are attached under that root component by
UE's scene-root validation (`ValidateSceneRootNodes`), not kept as independent
roots — so "attach to root" == "child of the first scene component." Note that
`reparent_component` to an empty parent on such a node is therefore typically a
no-op (UE re-promotes the existing root). Naming this explicitly removes the
need to read UE engine source to confirm correctness.

**Wiki page to improve:** `docs/wiki-src/blueprint.scs.md` (overlay; add a
`##`-level note and/or the `### blueprint.scs.add_component` /
`### blueprint.scs.reparent_component` H3 sections). The actual edit is the
downstream wiki process, not this ticket.

**Evidence (this task — `blueprint.scs.set_property` seed, all calls `ok:true`,
no `is_error`):** Friction note: *"blueprint.scs.get reports BeaconBase
child_count=2 … I worried DetectionZone was wrongly nested, did an unnecessary
reparent_component (which UE put right back), then read UE's
SimpleConstructionScript::ValidateSceneRootNodes to confirm this is correct,
intended UE behavior … the readback ambiguity … cost several diagnostic
calls."* Concretely the agent spent, after a clean build: 1 wasted
`blueprint.scs.reparent_component` (DetectionZone → root) that UE immediately
reverted, plus the follow-on re-readback/re-dump/re-compile chain
(`scs.get`, `asset.dump`, `compile`, `scs.get`) and a UE-source read — friction
that a one-paragraph docs note would have pre-empted.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-auditor finding on the `blueprint.scs.set_property` seed task (build BP_WarningBeacon from scratch). Every call returned `ok:true`/no error, yet the agent burned an unnecessary `reparent_component` (UE reverted it) and a UE-source dive because `scs.add_component`'s "attach to root" doc never warns that, with no native root, UE promotes the first scene component to `RootComponent` and reparents later root-level scene components under it (`ValidateSceneRootNodes`). Distinct from `B-scs-get-local-child-parent-link-missing` (readback bug): this is the docs/discovery gap that persists even after that fix. Proposed: note the root-promotion/auto-reparent rule in `docs/wiki-src/blueprint.scs.md` (overlay + add_component/reparent_component H3s).
- `#3-corroborating-evidence-clean-run` `OPEN` reporter — Cross-task evidence that the `#2` overlay note works as intended (still IN-REVIEW, not yet verified). A later `blueprint.scs.add_component` seed task built `/Game/Vehicles/BP_PatrolDrone` end-to-end: Body (StaticMesh, root, Cube) with **both** Sensor (Sphere) and Headlamp (SpotLight) parented under Body, transforms + CastShadow=false + Intensity=8000, compile, read-back. Outcome `clean`, friction `none`. The agent's friction note credits this overlay directly: *"wiki 'No native root' note pre-warned that Body would be promoted to RootComponent, so I attached Sensor/Headlamp explicitly to Body and no reparent fix was needed; every call succeeded first try, no python.execute or plugin-source fallback."* All 9 read-the-wiki calls included `blueprint.scs.reparent_component.md` (pre-loaded in case it was needed) but **zero** reparent execute calls fired — the exact wasted `reparent_component` + UE-source dive the originating `#1` task burned was pre-empted. Strengthens the case to verify and close this docs ticket. No new ticket filed.
- `#2-overlay-root-promotion-note` `IN-REVIEW` developer — Documented UE's no-native-root scene-root promotion/auto-reparent rule in the `blueprint.scs` overlay. Verified the gap against current source first: `SCSHandler.cpp:85` parentComponentName still reads "omit (or empty) to attach as a child of the root." and `:141` newParentName "empty for root" — neither mentions promotion; `AddSCSComponent` calls `SCS->AddNode` for the no-parent case and always reports `parent:"(root)"` (`EditorAutomationRpcGateway_SCSHandlers.cpp:714,780-782`); the reparent no-op message is at `:1014-1023` — so the defect is present, not already fixed. Sibling `B-scs-get-local-child-parent-link-missing` (IN-REVIEW) is a distinct `scs.get` readback bug, not a duplicate. Changes: `Docs/wiki-src/blueprint.scs.md` — added a `## No native root: UE promotes the first scene component and reparents the rest` namespace-page section, plus `### blueprint.scs.add_component` and `### blueprint.scs.reparent_component` H3 method sections (naming `ValidateSceneRootNodes`, the first-scene-component→`RootComponent` rule, the fixed `parent:"(root)"` label, and the reparent no-op). Regression test: appended two cases to `Private/Tests/Infra/TestWikiHandler.cpp` — `FWikiHandlerScsAddComponentDocumentsRootPromotionTest` (`…MethodPage.ScsAddComponentDocumentsRootPromotion`) and `FWikiHandlerScsReparentDocumentsNoOpTest` (`…MethodPage.ScsReparentDocumentsNoOp`), each rendering the real method page via `WikiHandler::RenderPage` and asserting on overlay-exclusive markers (`ValidateSceneRootNodes` / `RootComponent` / the verbatim "no changes made" no-op string) that appear nowhere in the auto summary or param descriptions, so they fail iff the H3 overlay sections are reverted. Not yet compiled/tested (later phase).
