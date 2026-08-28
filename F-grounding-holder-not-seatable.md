---
id: F-grounding-holder-not-seatable
title: "Both grounding verbs measure an ISM/HISM holder actor as if it were a single prop, and neither refuses"
status: IN-REVIEW
severity: Medium
category: feature
tags: [spatial, ground_actors, verify_grounding, ism, hism, instanced-static-mesh, refusal, symmetry]
encounters: 1
lastSeen: 2026-08-27
---

# A scatter holder has no meaningful underside, and both verbs measure one anyway

An `InstancedStaticMeshComponent` / `HierarchicalInstancedStaticMeshComponent` holder actor's bounds
cover **every instance**, so its "footprint" is the union of a whole scatter and its "underside" is
meaningless. `spatial.verify_grounding` will pass or fail it on a number that describes nothing, and
`spatial.ground_actors` will happily move the holder -- relocating every instance at once.

Neither verb refuses. Both should, with a `HOLDER_NOT_SEATABLE`-style typed error naming the component
and pointing at whatever the right per-instance path is.

**The refusal has to be symmetric.** If only the seat verb refuses, the verify verb still returns a
verdict nobody should trust; if only verify refuses, the seat verb will move something verify declines
to judge. That symmetry requirement is why neither of the two agents working these verbs took it --
each owned one side.

## History
- `#1-raised-independently-by-both-grounding-fixes` `OPEN` reporter -- Named by the agent fixing
  `B-ground-actors-prefix-captures-foreign-actors` and again, independently, by the agent fixing
  `B-verify-grounding-maxgap-false-fail`. Originally recorded as encounter 2 of the ground_actors
  ticket.
- `#2-symmetric-holder-refusal` `IN-REVIEW` developer — "Added a shared holder detector
  (`GroundPlacement::FindInstancedHolder` / `DescribeInstancedHolder`) in GroundPlacementUtils.cpp
  doing one `GetComponents<UInstancedStaticMeshComponent>` sweep, keeping the component with the
  most instances and refusing only at 2+ instances; registered `HOLDER_NOT_SEATABLE` in
  ErrorCodes.h; wired it symmetrically so `EvaluateContact` refuses ahead of every other criterion
  (verify side) and `SeatActor` refuses before the pre-move probe with its own
  `EGroundSeatStatus::HolderNotSeatable` (seat side), both emitting the same code and the same
  component-naming text; level.audit still gets its raw numbers because MeasureContact keeps
  sampling. Regression tests
  `PinWright.spatial.verify_grounding.InstancedHolderRefused` and
  `PinWright.spatial.ground_actors.InstancedHolderRefused` in TestGroundPlacement.cpp put a 3-instance
  HISM holder over a wide floor, so before the fix verify returned ACTOR_NOT_GROUNDED and
  ground_actors moved the whole scatter. NOT COMPILED OR RUN."
